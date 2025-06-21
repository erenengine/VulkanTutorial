## 소개

지금 우리가 사용하는 정점 버퍼는 올바르게 작동하지만, CPU에서 접근할 수 있도록 하는 메모리 타입이 그래픽 카드 자체에서 읽기에 가장 최적의 메모리 타입은 아닐 수 있습니다. 가장 최적화된 메모리는 `VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT` 플래그를 가지며, 보통 외장 그래픽 카드에서는 CPU가 접근할 수 없습니다. 이번 장에서는 두 개의 정점 버퍼를 만들 것입니다. 하나는 정점 배열의 데이터를 업로드하기 위한 CPU 접근 가능 메모리의 *스테이징 버퍼(staging buffer)*이고, 다른 하나는 디바이스 로컬 메모리에 있는 최종 정점 버퍼입니다. 그런 다음 버퍼 복사 명령을 사용해 스테이징 버퍼의 데이터를 실제 정점 버퍼로 이동시킬 것입니다.

## 전송 큐 (Transfer queue)

버퍼 복사 명령은 `VK_QUEUE_TRANSFER_BIT`로 표시되는 전송(transfer) 연산을 지원하는 큐 패밀리(queue family)를 필요로 합니다. 좋은 소식은 `VK_QUEUE_GRAPHICS_BIT`나 `VK_QUEUE_COMPUTE_BIT` 기능을 가진 모든 큐 패밀리는 이미 암시적으로 `VK_QUEUE_TRANSFER_BIT` 연산을 지원한다는 것입니다. 이런 경우 구현체는 `queueFlags`에 이 비트를 명시적으로 표시하지 않아도 됩니다.

만약 도전해보고 싶다면, 전송 연산만을 위한 별도의 큐 패밀리를 사용해볼 수도 있습니다. 이를 위해서는 프로그램에 다음과 같은 수정이 필요합니다.

*   `QueueFamilyIndices`와 `findQueueFamilies`를 수정하여 `VK_QUEUE_GRAPHICS_BIT`는 없지만 `VK_QUEUE_TRANSFER_BIT` 비트를 가진 큐 패밀리를 명시적으로 찾도록 합니다.
*   `createLogicalDevice`를 수정하여 전송 큐에 대한 핸들을 요청합니다.
*   전송 큐 패밀리에서 제출될 커맨드 버퍼를 위한 두 번째 커맨드 풀(command pool)을 생성합니다.
*   리소스의 `sharingMode`를 `VK_SHARING_MODE_CONCURRENT`로 변경하고 그래픽 큐와 전송 큐 패밀리를 모두 지정합니다.
*   `vkCmdCopyBuffer`와 같은 모든 전송 명령을 그래픽 큐가 아닌 전송 큐에 제출합니다.

약간의 작업이 필요하지만, 이를 통해 큐 패밀리 간에 리소스를 어떻게 공유하는지에 대해 많은 것을 배울 수 있을 것입니다.

## 버퍼 생성 추상화

이번 장에서는 여러 버퍼를 생성할 것이므로, 버퍼 생성을 헬퍼(helper) 함수로 옮기는 것이 좋습니다. `createBuffer`라는 새 함수를 만들고, `createVertexBuffer`에 있던 코드(매핑 제외)를 이 함수로 옮기세요.

```c++
void createBuffer(VkDeviceSize size, VkBufferUsageFlags usage, VkMemoryPropertyFlags properties, VkBuffer& buffer, VkDeviceMemory& bufferMemory) {
    VkBufferCreateInfo bufferInfo{};
    bufferInfo.sType = VK_STRUCTURE_TYPE_BUFFER_CREATE_INFO;
    bufferInfo.size = size;
    bufferInfo.usage = usage;
    bufferInfo.sharingMode = VK_SHARING_MODE_EXCLUSIVE;

    if (vkCreateBuffer(device, &bufferInfo, nullptr, &buffer) != VK_SUCCESS) {
        throw std::runtime_error("failed to create buffer!");
    }

    VkMemoryRequirements memRequirements;
    vkGetBufferMemoryRequirements(device, buffer, &memRequirements);

    VkMemoryAllocateInfo allocInfo{};
    allocInfo.sType = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO;
    allocInfo.allocationSize = memRequirements.size;
    allocInfo.memoryTypeIndex = findMemoryType(memRequirements.memoryTypeBits, properties);

    if (vkAllocateMemory(device, &allocInfo, nullptr, &bufferMemory) != VK_SUCCESS) {
        throw std::runtime_error("failed to allocate buffer memory!");
    }

    vkBindBufferMemory(device, buffer, bufferMemory, 0);
}
```

다양한 종류의 버퍼를 생성하는 데 이 함수를 사용할 수 있도록 버퍼 크기, 메모리 속성, 사용 목적을 매개변수로 추가해야 합니다. 마지막 두 매개변수는 핸들을 기록하기 위한 출력 변수입니다.

이제 `createVertexBuffer`에서 버퍼 생성 및 메모리 할당 코드를 제거하고, 대신 `createBuffer`를 호출할 수 있습니다.

```c++
void createVertexBuffer() {
    VkDeviceSize bufferSize = sizeof(vertices[0]) * vertices.size();
    createBuffer(bufferSize, VK_BUFFER_USAGE_VERTEX_BUFFER_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, vertexBuffer, vertexBufferMemory);

    void* data;
    vkMapMemory(device, vertexBufferMemory, 0, bufferSize, 0, &data);
        memcpy(data, vertices.data(), (size_t) bufferSize);
    vkUnmapMemory(device, vertexBufferMemory);
}
```

프로그램을 실행하여 정점 버퍼가 여전히 제대로 작동하는지 확인하세요.

## 스테이징 버퍼 사용하기

이제 `createVertexBuffer` 함수를 수정하여, 호스트 가시성(host visible) 버퍼는 임시 버퍼로만 사용하고, 디바이스 로컬(device local) 버퍼를 실제 정점 버퍼로 사용하도록 변경하겠습니다.

```c++
void createVertexBuffer() {
    VkDeviceSize bufferSize = sizeof(vertices[0]) * vertices.size();

    VkBuffer stagingBuffer;
    VkDeviceMemory stagingBufferMemory;
    createBuffer(bufferSize, VK_BUFFER_USAGE_TRANSFER_SRC_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, stagingBuffer, stagingBufferMemory);

    void* data;
    vkMapMemory(device, stagingBufferMemory, 0, bufferSize, 0, &data);
        memcpy(data, vertices.data(), (size_t) bufferSize);
    vkUnmapMemory(device, stagingBufferMemory);

    createBuffer(bufferSize, VK_BUFFER_USAGE_TRANSFER_DST_BIT | VK_BUFFER_USAGE_VERTEX_BUFFER_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, vertexBuffer, vertexBufferMemory);
}
```

이제 정점 데이터를 매핑하고 복사하기 위해 `stagingBuffer`와 `stagingBufferMemory`를 사용합니다. 이번 장에서는 두 개의 새로운 버퍼 사용 플래그를 사용합니다:

*   `VK_BUFFER_USAGE_TRANSFER_SRC_BIT`: 버퍼가 메모리 전송 연산의 원본(source)으로 사용될 수 있습니다.
*   `VK_BUFFER_USAGE_TRANSFER_DST_BIT`: 버퍼가 메모리 전송 연산의 대상(destination)으로 사용될 수 있습니다.

이제 `vertexBuffer`는 디바이스 로컬 메모리 타입으로 할당됩니다. 이는 일반적으로 우리가 `vkMapMemory`를 사용할 수 없다는 것을 의미합니다. 하지만 `stagingBuffer`에서 `vertexBuffer`로 데이터를 복사할 수는 있습니다. 이를 위해 `stagingBuffer`에는 전송 원본 플래그를, `vertexBuffer`에는 정점 버퍼 사용 플래그와 함께 전송 대상 플래그를 지정해야 합니다.

이제 한 버퍼의 내용을 다른 버퍼로 복사하는 `copyBuffer` 함수를 작성하겠습니다.

```c++
void copyBuffer(VkBuffer srcBuffer, VkBuffer dstBuffer, VkDeviceSize size) {

}
```

메모리 전송 연산은 그리기 명령과 마찬가지로 커맨드 버퍼를 사용하여 실행됩니다. 따라서 먼저 임시 커맨드 버퍼를 할당해야 합니다. 이런 종류의 단기(short-lived) 버퍼를 위해 별도의 커맨드 풀을 만드는 것을 고려할 수 있습니다. 왜냐하면 구현체가 메모리 할당 최적화를 적용할 수 있기 때문입니다. 그 경우 커맨드 풀 생성 시 `VK_COMMAND_POOL_CREATE_TRANSIENT_BIT` 플래그를 사용해야 합니다.

```c++
void copyBuffer(VkBuffer srcBuffer, VkBuffer dstBuffer, VkDeviceSize size) {
    VkCommandBufferAllocateInfo allocInfo{};
    allocInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_ALLOCATE_INFO;
    allocInfo.level = VK_COMMAND_BUFFER_LEVEL_PRIMARY;
    allocInfo.commandPool = commandPool;
    allocInfo.commandBufferCount = 1;

    VkCommandBuffer commandBuffer;
    vkAllocateCommandBuffers(device, &allocInfo, &commandBuffer);
}
```

그리고 즉시 커맨드 버퍼 기록을 시작합니다.

```c++
VkCommandBufferBeginInfo beginInfo{};
beginInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO;
beginInfo.flags = VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT;

vkBeginCommandBuffer(commandBuffer, &beginInfo);
```

우리는 이 커맨드 버퍼를 한 번만 사용할 것이며, 복사 작업이 실행 완료될 때까지 함수에서 반환하지 않고 기다릴 것입니다. 드라이버에게 우리의 의도를 `VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT` 플래그를 통해 알려주는 것이 좋은 습관입니다.

```c++
VkBufferCopy copyRegion{};
copyRegion.srcOffset = 0; // Optional
copyRegion.dstOffset = 0; // Optional
copyRegion.size = size;
vkCmdCopyBuffer(commandBuffer, srcBuffer, dstBuffer, 1, &copyRegion);
```

버퍼의 내용은 `vkCmdCopyBuffer` 명령을 통해 전송됩니다. 이 함수는 원본과 대상 버퍼를 인자로 받고, 복사할 영역의 배열을 받습니다. 이 영역들은 `VkBufferCopy` 구조체로 정의되며, 원본 버퍼 오프셋, 대상 버퍼 오프셋, 그리고 크기로 구성됩니다. `vkMapMemory` 명령과 달리 여기서는 `VK_WHOLE_SIZE`를 지정할 수 없습니다.

```c++
vkEndCommandBuffer(commandBuffer);
```

이 커맨드 버퍼는 복사 명령만 포함하므로, 바로 기록을 중단할 수 있습니다. 이제 커맨드 버퍼를 실행하여 전송을 완료합니다.

```c++
VkSubmitInfo submitInfo{};
submitInfo.sType = VK_STRUCTURE_TYPE_SUBMIT_INFO;
submitInfo.commandBufferCount = 1;
submitInfo.pCommandBuffers = &commandBuffer;

vkQueueSubmit(graphicsQueue, 1, &submitInfo, VK_NULL_HANDLE);
vkQueueWaitIdle(graphicsQueue);
```

그리기 명령과 달리 이번에는 기다려야 할 이벤트가 없습니다. 단지 버퍼에 대한 전송을 즉시 실행하기만 하면 됩니다. 이 전송이 완료되기를 기다리는 방법에는 다시 두 가지가 있습니다. 펜스(fence)를 사용하고 `vkWaitForFences`로 기다리거나, 단순히 `vkQueueWaitIdle`로 전송 큐가 유휴(idle) 상태가 될 때까지 기다릴 수 있습니다. 펜스를 사용하면 여러 전송을 동시에 스케줄링하고 모든 작업이 완료될 때까지 기다릴 수 있어, 드라이버가 최적화할 더 많은 기회를 가질 수 있습니다.

```c++
vkFreeCommandBuffers(device, commandPool, 1, &commandBuffer);
```

전송 작업에 사용된 커맨드 버퍼를 정리하는 것을 잊지 마세요.

이제 `createVertexBuffer` 함수에서 `copyBuffer`를 호출하여 정점 데이터를 디바이스 로컬 버퍼로 옮길 수 있습니다.

```c++
createBuffer(bufferSize, VK_BUFFER_USAGE_TRANSFER_DST_BIT | VK_BUFFER_USAGE_VERTEX_BUFFER_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, vertexBuffer, vertexBufferMemory);

copyBuffer(stagingBuffer, vertexBuffer, bufferSize);
```

스테이징 버퍼의 데이터를 디바이스 버퍼로 복사한 후에는 스테이징 버퍼를 정리해야 합니다.

```c++
    ...

    copyBuffer(stagingBuffer, vertexBuffer, bufferSize);

    vkDestroyBuffer(device, stagingBuffer, nullptr);
    vkFreeMemory(device, stagingBufferMemory, nullptr);
}
```

프로그램을 실행하여 익숙한 삼각형이 다시 보이는지 확인하세요. 성능 향상이 지금 당장은 눈에 보이지 않을 수 있지만, 이제 정점 데이터는 고성능 메모리에서 로드되고 있습니다. 이는 앞으로 더 복잡한 지오메트리를 렌더링하기 시작할 때 중요해질 것입니다.

## 결론

실제 애플리케이션에서는 모든 개별 버퍼에 대해 `vkAllocateMemory`를 호출해서는 안 된다는 점에 유의해야 합니다. 동시 메모리 할당의 최대 수는 `maxMemoryAllocationCount` 물리 디바이스 제한에 의해 제한되며, NVIDIA GTX 1080과 같은 고사양 하드웨어에서도 `4096` 정도로 낮을 수 있습니다. 다수의 객체에 대해 동시에 메모리를 할당하는 올바른 방법은, 우리가 많은 함수에서 보았던 `offset` 매개변수를 사용하여 단일 할당을 여러 객체에 나누어 사용하는 커스텀 할당자(custom allocator)를 만드는 것입니다.

이러한 할당자를 직접 구현하거나, GPUOpen 이니셔티브에서 제공하는 [VulkanMemoryAllocator](https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator) 라이브러리를 사용할 수 있습니다. 하지만 이 튜토리얼에서는 지금 당장 이러한 제한에 도달할 일이 없으므로 모든 리소스에 대해 별도의 할당을 사용하는 것이 괜찮습니다.

[C++ 코드](/code/20_staging_buffer.cpp) /
[정점 셰이더](/code/18_shader_vertexbuffer.vert) /
[프래그먼트 셰이더](/code/18_shader_vertexbuffer.frag)