## 소개

Vulkan에서 버퍼(Buffer)는 그래픽 카드가 읽을 수 있는 임의의 데이터를 저장하는 데 사용되는 메모리 영역입니다. 이번 장에서 다룰 정점 데이터(vertex data)를 저장하는 데 사용할 수도 있지만, 앞으로의 장에서 살펴볼 다른 많은 목적으로도 사용될 수 있습니다. 지금까지 다뤄온 다른 Vulkan 객체들과는 달리, 버퍼는 스스로 메모리를 할당하지 않습니다. 이전 장들에서 보았듯이 Vulkan API는 프로그래머가 거의 모든 것을 직접 제어하도록 하며, 메모리 관리도 그중 하나입니다.

## 버퍼 생성

`createVertexBuffer`라는 새 함수를 만들고, `initVulkan` 함수에서 `createCommandBuffers` 바로 전에 호출하도록 합시다.

```c++
void initVulkan() {
    createInstance();
    setupDebugMessenger();
    createSurface();
    pickPhysicalDevice();
    createLogicalDevice();
    createSwapChain();
    createImageViews();
    createRenderPass();
    createGraphicsPipeline();
    createFramebuffers();
    createCommandPool();
    createVertexBuffer();
    createCommandBuffers();
    createSyncObjects();
}

...

void createVertexBuffer() {

}
```

버퍼를 생성하려면 `VkBufferCreateInfo` 구조체를 채워야 합니다.

```c++
VkBufferCreateInfo bufferInfo{};
bufferInfo.sType = VK_STRUCTURE_TYPE_BUFFER_CREATE_INFO;
bufferInfo.size = sizeof(vertices[0]) * vertices.size();
```

구조체의 첫 번째 필드는 `size`로, 버퍼의 크기를 바이트 단위로 지정합니다. 정점 데이터의 바이트 크기는 `sizeof`를 사용해 간단하게 계산할 수 있습니다.

```c++
bufferInfo.usage = VK_BUFFER_USAGE_VERTEX_BUFFER_BIT;
```

두 번째 필드는 `usage`로, 버퍼에 있는 데이터가 어떤 목적으로 사용될지를 나타냅니다. 비트 OR 연산을 사용하여 여러 목적을 동시에 지정할 수도 있습니다. 우리의 사용 사례는 정점 버퍼이므로, 이 플래그를 사용합니다. 다른 사용 유형은 향후 장에서 살펴보겠습니다.

```c++
bufferInfo.sharingMode = VK_SHARING_MODE_EXCLUSIVE;
```

스왑 체인의 이미지처럼, 버퍼도 특정 큐 패밀리가 소유하거나 여러 큐 패밀리 간에 공유될 수 있습니다. 이 버퍼는 그래픽스 큐에서만 사용할 것이므로, 배타적(exclusive) 접근 방식이면 충분합니다.

`flags` 매개변수는 희소 버퍼 메모리(sparse buffer memory)를 설정하는 데 사용되며, 지금은 중요하지 않습니다. 기본값인 `0`으로 두겠습니다.

이제 `vkCreateBuffer`를 사용해 버퍼를 생성할 수 있습니다. 버퍼 핸들을 저장할 클래스 멤버 `vertexBuffer`를 정의하고 함수를 호출합니다.

```c++
VkBuffer vertexBuffer;

...

void createVertexBuffer() {
    VkBufferCreateInfo bufferInfo{};
    bufferInfo.sType = VK_STRUCTURE_TYPE_BUFFER_CREATE_INFO;
    bufferInfo.size = sizeof(vertices[0]) * vertices.size();
    bufferInfo.usage = VK_BUFFER_USAGE_VERTEX_BUFFER_BIT;
    bufferInfo.sharingMode = VK_SHARING_MODE_EXCLUSIVE;

    if (vkCreateBuffer(device, &bufferInfo, nullptr, &vertexBuffer) != VK_SUCCESS) {
        throw std::runtime_error("failed to create vertex buffer!");
    }
}
```

버퍼는 프로그램이 끝날 때까지 렌더링 명령어에서 사용할 수 있어야 하며, 스왑 체인에 종속되지 않으므로 원래의 `cleanup` 함수에서 정리합니다.

```c++
void cleanup() {
    cleanupSwapChain();

    vkDestroyBuffer(device, vertexBuffer, nullptr);

    ...
}
```

## 메모리 요구사항

버퍼는 생성되었지만 아직 메모리가 할당되지 않았습니다. 버퍼에 메모리를 할당하는 첫 단계는 이름 그대로인 `vkGetBufferMemoryRequirements` 함수를 사용하여 메모리 요구사항을 쿼리하는 것입니다.

```c++
VkMemoryRequirements memRequirements;
vkGetBufferMemoryRequirements(device, vertexBuffer, &memRequirements);
```

`VkMemoryRequirements` 구조체에는 세 가지 필드가 있습니다.

*   `size`: 필요한 메모리의 크기(바이트 단위)이며, `bufferInfo.size`와 다를 수 있습니다.
*   `alignment`: 할당된 메모리 영역 내에서 버퍼가 시작되는 오프셋(offset)을 바이트 단위로 나타냅니다. `bufferInfo.usage`와 `bufferInfo.flags`에 따라 달라집니다.
*   `memoryTypeBits`: 버퍼에 적합한 메모리 타입들의 비트 필드입니다.

그래픽 카드는 할당할 수 있는 여러 다른 종류의 메모리를 제공합니다. 각 메모리 타입은 허용되는 연산과 성능 특성 면에서 다릅니다. 우리는 버퍼의 요구사항과 우리 애플리케이션의 요구사항을 결합하여 사용할 올바른 메모리 타입을 찾아야 합니다. 이를 위해 `findMemoryType`이라는 새 함수를 만들어 봅시다.

```c++
uint32_t findMemoryType(uint32_t typeFilter, VkMemoryPropertyFlags properties) {

}
```

먼저 `vkGetPhysicalDeviceMemoryProperties`를 사용하여 사용 가능한 메모리 타입에 대한 정보를 쿼리해야 합니다.

```c++
VkPhysicalDeviceMemoryProperties memProperties;
vkGetPhysicalDeviceMemoryProperties(physicalDevice, &memProperties);
```

`VkPhysicalDeviceMemoryProperties` 구조체에는 `memoryTypes`와 `memoryHeaps` 두 개의 배열이 있습니다. 메모리 힙(Memory heap)은 전용 VRAM이나 VRAM이 부족할 때 사용되는 RAM의 스왑 공간과 같은 개별적인 메모리 자원입니다. 다양한 종류의 메모리 타입이 이 힙들 안에 존재합니다. 지금은 메모리 타입 자체에만 신경 쓰고 힙은 신경 쓰지 않겠지만, 이 선택이 성능에 영향을 미칠 수 있다는 점은 상상할 수 있을 것입니다.

먼저 버퍼 자체에 적합한 메모리 타입을 찾아봅시다.

```c++
for (uint32_t i = 0; i < memProperties.memoryTypeCount; i++) {
    if (typeFilter & (1 << i)) {
        return i;
    }
}

throw std::runtime_error("failed to find suitable memory type!");
```

`typeFilter` 매개변수는 적합한 메모리 타입들의 비트 필드를 지정하는 데 사용됩니다. 즉, 단순히 모든 메모리 타입을 순회하며 해당하는 비트가 `1`로 설정되어 있는지 확인하면 적합한 메모리 타입의 인덱스를 찾을 수 있습니다.

하지만 우리는 정점 버퍼에 적합한 메모리 타입에만 관심 있는 것이 아닙니다. 또한 그 메모리에 정점 데이터를 쓸 수 있어야 합니다. `memoryTypes` 배열은 각 메모리 타입의 힙과 속성을 지정하는 `VkMemoryType` 구조체로 구성됩니다. 이 속성들은 메모리의 특별한 기능들을 정의하는데, 예를 들어 CPU에서 메모리에 쓰기 위해 맵핑(map)할 수 있는 기능이 있습니다. 이 속성은 `VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT`으로 표시되며, `VK_MEMORY_PROPERTY_HOST_COHERENT_BIT` 속성도 필요합니다. 메모리를 맵핑할 때 그 이유를 알게 될 것입니다.

이제 이 속성들의 지원 여부도 확인하도록 루프를 수정할 수 있습니다.

```c++
for (uint32_t i = 0; i < memProperties.memoryTypeCount; i++) {
    if ((typeFilter & (1 << i)) && (memProperties.memoryTypes[i].propertyFlags & properties) == properties) {
        return i;
    }
}
```

우리가 원하는 속성이 여러 개일 수 있으므로, 비트 AND 연산의 결과가 0이 아닌지 확인하는 것뿐만 아니라, 원하는 속성 비트 필드와 정확히 일치하는지 확인해야 합니다. 버퍼에 적합하면서 우리가 필요로 하는 모든 속성을 가진 메모리 타입이 있다면 그 인덱스를 반환하고, 그렇지 않으면 예외를 던집니다.

## 메모리 할당

이제 올바른 메모리 타입을 결정할 방법을 알았으니, `VkMemoryAllocateInfo` 구조체를 채워 실제로 메모리를 할당할 수 있습니다.

```c++
VkMemoryAllocateInfo allocInfo{};
allocInfo.sType = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO;
allocInfo.allocationSize = memRequirements.size;
allocInfo.memoryTypeIndex = findMemoryType(memRequirements.memoryTypeBits, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT);
```

이제 메모리 할당은 크기와 타입을 지정하는 것만큼 간단해졌습니다. 두 값 모두 정점 버퍼의 메모리 요구사항과 원하는 속성으로부터 파생됩니다. 메모리 핸들을 저장할 클래스 멤버를 만들고 `vkAllocateMemory`로 할당합니다.

```c++
VkBuffer vertexBuffer;
VkDeviceMemory vertexBufferMemory;

...

if (vkAllocateMemory(device, &allocInfo, nullptr, &vertexBufferMemory) != VK_SUCCESS) {
    throw std::runtime_error("failed to allocate vertex buffer memory!");
}
```

메모리 할당에 성공했다면, 이제 `vkBindBufferMemory`를 사용하여 이 메모리를 버퍼와 연결할 수 있습니다.

```c++
vkBindBufferMemory(device, vertexBuffer, vertexBufferMemory, 0);
```

처음 세 매개변수는 이름에서 알 수 있듯이 명확하며, 네 번째 매개변수는 메모리 영역 내의 오프셋입니다. 이 메모리는 이 정점 버퍼를 위해 특별히 할당되었으므로, 오프셋은 간단히 `0`입니다. 만약 오프셋이 0이 아니라면, `memRequirements.alignment`로 나누어떨어져야 합니다.

물론, C++의 동적 메모리 할당처럼, 이 메모리도 언젠가는 해제되어야 합니다. 버퍼 객체에 바인딩된 메모리는 버퍼가 더 이상 사용되지 않을 때 해제할 수 있으므로, 버퍼가 파괴된 후에 해제하도록 합시다.

```c++
void cleanup() {
    cleanupSwapChain();

    vkDestroyBuffer(device, vertexBuffer, nullptr);
    vkFreeMemory(device, vertexBufferMemory, nullptr);
```

## 정점 버퍼 채우기

이제 정점 데이터를 버퍼에 복사할 차례입니다. 이는 `vkMapMemory`를 사용하여 버퍼 메모리를 CPU에서 접근 가능한 메모리로 맵핑(mapping)하여 수행됩니다.

```c++
void* data;
vkMapMemory(device, vertexBufferMemory, 0, bufferInfo.size, 0, &data);
```

이 함수를 사용하면 오프셋과 크기로 정의된 지정된 메모리 자원의 영역에 접근할 수 있습니다. 여기서 오프셋과 크기는 각각 `0`과 `bufferInfo.size`입니다. 모든 메모리를 맵핑하기 위해 특별한 값 `VK_WHOLE_SIZE`를 지정할 수도 있습니다. 뒤에서 두 번째 매개변수는 플래그를 지정하는 데 사용될 수 있지만, 현재 API에는 아직 사용 가능한 플래그가 없습니다. 반드시 `0`으로 설정해야 합니다. 마지막 매개변수는 맵핑된 메모리에 대한 포인터의 출력을 지정합니다.

```c++
void* data;
vkMapMemory(device, vertexBufferMemory, 0, bufferInfo.size, 0, &data);
    memcpy(data, vertices.data(), (size_t) bufferInfo.size);
vkUnmapMemory(device, vertexBufferMemory);
```

이제 `memcpy`를 사용하여 정점 데이터를 맵핑된 메모리에 복사하고, `vkUnmapMemory`를 사용하여 다시 언맵핑하면 됩니다. 안타깝게도 드라이버가 데이터를 버퍼 메모리로 즉시 복사하지 않을 수 있습니다 (예: 캐싱 때문). 또한 버퍼에 대한 쓰기가 맵핑된 메모리에서 아직 보이지 않을 수도 있습니다. 이 문제를 해결하는 두 가지 방법이 있습니다.

*   `VK_MEMORY_PROPERTY_HOST_COHERENT_BIT`로 표시된 호스트 일관성(host coherent) 메모리 힙을 사용합니다.
*   맵핑된 메모리에 쓴 후 `vkFlushMappedMemoryRanges`를 호출하고, 맵핑된 메모리에서 읽기 전에 `vkInvalidateMappedMemoryRanges`를 호출합니다.

우리는 첫 번째 접근 방식, 즉 맵핑된 메모리가 항상 할당된 메모리의 내용과 일치하도록 보장하는 방식을 선택했습니다. 이 방식이 명시적인 플러싱(flushing)보다 성능이 약간 저하될 수 있지만, 다음 장에서 왜 이것이 중요하지 않은지 알게 될 것입니다.

메모리 범위를 플러싱하거나 일관성 있는 메모리 힙을 사용하는 것은 드라이버가 우리의 버퍼 쓰기를 인지한다는 것을 의미하지만, 이것이 GPU에서 실제로 보인다는 것을 의미하지는 않습니다. GPU로의 데이터 전송은 백그라운드에서 발생하는 작업이며, 사양에서는 단순히 [다음 `vkQueueSubmit` 호출 시점에 완료되는 것이 보장된다](https://www.khronos.org/registry/vulkan/specs/1.3-extensions/html/chap7.html#synchronization-submission-host-writes)고 명시합니다.

## 정점 버퍼 바인딩

이제 남은 일은 렌더링 작업 중에 정점 버퍼를 바인딩하는 것뿐입니다. 이를 위해 `recordCommandBuffer` 함수를 확장하겠습니다.

```c++
vkCmdBindPipeline(commandBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, graphicsPipeline);

VkBuffer vertexBuffers[] = {vertexBuffer};
VkDeviceSize offsets[] = {0};
vkCmdBindVertexBuffers(commandBuffer, 0, 1, vertexBuffers, offsets);

vkCmdDraw(commandBuffer, static_cast<uint32_t>(vertices.size()), 1, 0, 0);
```

`vkCmdBindVertexBuffers` 함수는 이전 장에서 설정한 것과 같은 바인딩에 정점 버퍼를 바인딩하는 데 사용됩니다. 명령어 버퍼 다음의 첫 두 매개변수는 우리가 정점 버퍼를 지정할 바인딩의 시작 오프셋과 개수를 지정합니다. 마지막 두 매개변수는 바인딩할 정점 버퍼의 배열과 정점 데이터를 읽기 시작할 바이트 오프셋을 지정합니다. 또한 `vkCmdDraw` 호출을 수정하여 하드코딩된 숫자 `3` 대신 버퍼에 있는 정점의 수를 전달하도록 해야 합니다.

이제 프로그램을 실행하면 익숙한 삼각형이 다시 나타날 것입니다.

![](/images/triangle.png)

`vertices` 배열을 수정하여 맨 위 정점의 색상을 흰색으로 변경해 보세요.

```c++
const std::vector<Vertex> vertices = {
    {{0.0f, -0.5f}, {1.0f, 1.0f, 1.0f}},
    {{0.5f, 0.5f}, {0.0f, 1.0f, 0.0f}},
    {{-0.5f, 0.5f}, {0.0f, 0.0f, 1.0f}}
};
```

프로그램을 다시 실행하면 다음과 같이 보일 것입니다.

![](/images/triangle_white.png)

다음 장에서는 더 나은 성능을 제공하지만 약간의 추가 작업이 필요한, 정점 데이터를 정점 버퍼로 복사하는 다른 방법을 살펴보겠습니다.

[C++ 코드](/code/19_vertex_buffer.cpp) /
[정점 셰이더](/code/18_shader_vertexbuffer.vert) /
[프래그먼트 셰이더](/code/18_shader_vertexbuffer.frag)