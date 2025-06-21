## 동시에 여러 프레임 렌더링하기 (Frames in flight)

현재 우리의 렌더 루프에는 한 가지 명백한 결함이 있습니다. 이전 프레임의 렌더링이 끝나기를 기다려야만 다음 프레임의 렌더링을 시작할 수 있다는 점인데, 이는 호스트(CPU)의 불필요한 유휴 상태를 유발합니다.

<!-- insert diagram showing our current render loop and the 'multi frame in flight' render loop -->

이 문제를 해결하는 방법은 여러 프레임을 동시에 *작업 중(in-flight)* 상태로 두는 것입니다. 즉, 한 프레임의 렌더링이 다음 프레임의 기록을 방해하지 않도록 하는 것입니다. 어떻게 이렇게 할 수 있을까요? 렌더링 중에 접근하고 수정하는 모든 리소스를 복제해야 합니다. 따라서 여러 개의 커맨드 버퍼, 세마포어, 펜스가 필요합니다. 이후 챕터에서는 다른 리소스들의 여러 인스턴스도 추가할 것이므로, 이 개념은 다시 등장하게 될 것입니다.

먼저 프로그램 상단에 동시에 처리할 프레임 수를 정의하는 상수를 추가합니다.

```c++
const int MAX_FRAMES_IN_FLIGHT = 2;
```

2를 선택한 이유는 CPU가 GPU보다 *너무* 앞서 나가는 것을 원치 않기 때문입니다. 2개의 프레임이 동시 실행되면, CPU와 GPU가 동시에 각자의 작업을 처리할 수 있습니다. 만약 CPU가 먼저 작업을 마치면, GPU가 렌더링을 마칠 때까지 기다렸다가 다음 작업을 제출합니다. 3개 이상의 프레임을 사용하면 CPU가 GPU를 앞질러 지연 시간(latency)을 추가할 수 있습니다. 일반적으로 추가적인 지연 시간은 바람직하지 않습니다. 하지만 애플리케이션에 동시 실행 프레임 수를 제어할 수 있는 권한을 주는 것은 Vulkan의 명시적인(explicit) 특성을 보여주는 또 다른 예시입니다.

각 프레임은 자체적인 커맨드 버퍼, 세마포어 집합, 펜스를 가져야 합니다. 기존 객체들의 이름을 바꾸고 `std::vector`로 변경합니다.

```c++
std::vector<VkCommandBuffer> commandBuffers;

...

std::vector<VkSemaphore> imageAvailableSemaphores;
std::vector<VkSemaphore> renderFinishedSemaphores;
std::vector<VkFence> inFlightFences;
```

다음으로 여러 개의 커맨드 버퍼를 생성해야 합니다. `createCommandBuffer`를 `createCommandBuffers`로 이름을 바꿉니다. 그리고 커맨드 버퍼 벡터의 크기를 `MAX_FRAMES_IN_FLIGHT`로 조절하고, `VkCommandBufferAllocateInfo`가 해당 개수만큼의 커맨드 버퍼를 담도록 수정하며, 할당 대상을 우리의 커맨드 버퍼 벡터로 변경해야 합니다.

```c++
void createCommandBuffers() {
    commandBuffers.resize(MAX_FRAMES_IN_FLIGHT);
    ...
    allocInfo.commandBufferCount = (uint32_t) commandBuffers.size();

    if (vkAllocateCommandBuffers(device, &allocInfo, commandBuffers.data()) != VK_SUCCESS) {
        throw std::runtime_error("failed to allocate command buffers!");
    }
}
```

`createSyncObjects` 함수는 모든 동기화 객체들을 생성하도록 변경해야 합니다.

```c++
void createSyncObjects() {
    imageAvailableSemaphores.resize(MAX_FRAMES_IN_FLIGHT);
    renderFinishedSemaphores.resize(MAX_FRAMES_IN_FLIGHT);
    inFlightFences.resize(MAX_FRAMES_IN_FLIGHT);

    VkSemaphoreCreateInfo semaphoreInfo{};
    semaphoreInfo.sType = VK_STRUCTURE_TYPE_SEMAPHORE_CREATE_INFO;

    VkFenceCreateInfo fenceInfo{};
    fenceInfo.sType = VK_STRUCTURE_TYPE_FENCE_CREATE_INFO;
    fenceInfo.flags = VK_FENCE_CREATE_SIGNALED_BIT;

    for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHT; i++) {
        if (vkCreateSemaphore(device, &semaphoreInfo, nullptr, &imageAvailableSemaphores[i]) != VK_SUCCESS ||
            vkCreateSemaphore(device, &semaphoreInfo, nullptr, &renderFinishedSemaphores[i]) != VK_SUCCESS ||
            vkCreateFence(device, &fenceInfo, nullptr, &inFlightFences[i]) != VK_SUCCESS) {

            throw std::runtime_error("failed to create synchronization objects for a frame!");
        }
    }
}
```

마찬가지로, 이 객체들도 모두 정리(cleanup)되어야 합니다.

```c++
void cleanup() {
    for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHT; i++) {
        vkDestroySemaphore(device, renderFinishedSemaphores[i], nullptr);
        vkDestroySemaphore(device, imageAvailableSemaphores[i], nullptr);
        vkDestroyFence(device, inFlightFences[i], nullptr);
    }

    ...
}
```

커맨드 버퍼는 커맨드 풀이 해제될 때 자동으로 해제되므로, 커맨드 버퍼 정리를 위해 추가로 할 일은 없다는 점을 기억하세요.

매 프레임마다 올바른 객체를 사용하기 위해, 현재 프레임을 추적해야 합니다. 이를 위해 프레임 인덱스를 사용하겠습니다.

```c++
uint32_t currentFrame = 0;
```

이제 `drawFrame` 함수를 올바른 객체들을 사용하도록 수정할 수 있습니다.

```c++
void drawFrame() {
    vkWaitForFences(device, 1, &inFlightFences[currentFrame], VK_TRUE, UINT64_MAX);
    vkResetFences(device, 1, &inFlightFences[currentFrame]);

    vkAcquireNextImageKHR(device, swapChain, UINT64_MAX, imageAvailableSemaphores[currentFrame], VK_NULL_HANDLE, &imageIndex);

    ...

    vkResetCommandBuffer(commandBuffers[currentFrame],  0);
    recordCommandBuffer(commandBuffers[currentFrame], imageIndex);

    ...

    submitInfo.pCommandBuffers = &commandBuffers[currentFrame];

    ...

    VkSemaphore waitSemaphores[] = {imageAvailableSemaphores[currentFrame]};

    ...

    VkSemaphore signalSemaphores[] = {renderFinishedSemaphores[currentFrame]};

    ...

    if (vkQueueSubmit(graphicsQueue, 1, &submitInfo, inFlightFences[currentFrame]) != VK_SUCCESS) {
}
```

물론, 매번 다음 프레임으로 넘어가는 것을 잊지 말아야 합니다.

```c++
void drawFrame() {
    ...

    currentFrame = (currentFrame + 1) % MAX_FRAMES_IN_FLIGHT;
}
```

나머지(%) 연산자를 사용함으로써, 프레임 인덱스는 `MAX_FRAMES_IN_FLIGHT` 만큼의 프레임이 큐에 쌓인 후 다시 순환하게 됩니다.

<!-- Possibly use swapchain-image-count for renderFinished semaphores, as it can't
be known with a fence whether the semaphore is ready for re-use. -->

이제 우리는 최대 `MAX_FRAMES_IN_FLIGHT`개의 프레임만 작업 큐에 쌓이도록 하고, 이 프레임들이 서로를 침범하지 않도록 하는 데 필요한 모든 동기화를 구현했습니다. 최종 정리(cleanup)와 같은 코드의 다른 부분에서는 `vkDeviceWaitIdle`처럼 더 단순한 동기화에 의존해도 괜찮다는 점에 유의하세요. 성능 요구사항에 따라 어떤 접근 방식을 사용할지 결정해야 합니다.

동기화에 대해 예제를 통해 더 배우고 싶다면, Khronos의 [이 광범위한 개요](https://github.com/KhronosGroup/Vulkan-Docs/wiki/Synchronization-Examples#swapchain-image-acquire-and-present)를 살펴보세요.

다음 챕터에서는 잘 동작하는 Vulkan 프로그램을 위해 필요한 또 다른 작은 사항을 다룰 것입니다.

[C++ 코드](/code/16_frames_in_flight.cpp) /
[정점 셰이더](/code/09_shader_base.vert) /
[프래그먼트 셰이더](/code/09_shader_base.frag)