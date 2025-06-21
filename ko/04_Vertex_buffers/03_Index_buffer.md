## 소개

실제 애플리케이션에서 렌더링할 3D 메시는 여러 삼각형 간에 정점을 공유하는 경우가 많습니다. 이는 사각형을 그리는 것처럼 간단한 작업에서도 이미 발생합니다.

![](/images/vertex_vs_index.svg)

사각형을 그리려면 두 개의 삼각형이 필요하며, 이는 6개의 정점으로 구성된 정점 버퍼가 필요하다는 것을 의미합니다. 문제는 두 정점의 데이터가 중복되어 50%의 중복이 발생한다는 것입니다. 더 복잡한 메시에서는 정점이 평균 3개의 삼각형에서 재사용되므로 이 문제는 더욱 심각해집니다. 이 문제에 대한 해결책은 *인덱스 버퍼(index buffer)*를 사용하는 것입니다.

인덱스 버퍼는 본질적으로 정점 버퍼에 대한 포인터 배열입니다. 인덱스 버퍼를 사용하면 정점 데이터의 순서를 바꾸고, 여러 정점에 대해 기존 데이터를 재사용할 수 있습니다. 위 그림은 4개의 고유한 정점을 포함하는 정점 버퍼가 있을 때, 사각형을 위한 인덱스 버퍼가 어떻게 보일지를 보여줍니다. 처음 세 개의 인덱스는 오른쪽 위 삼각형을 정의하고, 마지막 세 개의 인덱스는 왼쪽 아래 삼각형의 정점을 정의합니다.

## 인덱스 버퍼 생성

이번 장에서는 정점 데이터를 수정하고 인덱스 데이터를 추가하여 그림과 같은 사각형을 그려보겠습니다. 네 개의 모서리를 나타내도록 정점 데이터를 수정합니다.

```c++
const std::vector<Vertex> vertices = {
    {{-0.5f, -0.5f}, {1.0f, 0.0f, 0.0f}},
    {{0.5f, -0.5f}, {0.0f, 1.0f, 0.0f}},
    {{0.5f, 0.5f}, {0.0f, 0.0f, 1.0f}},
    {{-0.5f, 0.5f}, {1.0f, 1.0f, 1.0f}}
};
```

왼쪽 아래 모서리는 빨간색, 오른쪽 아래는 녹색, 오른쪽 위는 파란색, 왼쪽 위는 흰색입니다. `indices`라는 새 배열을 추가하여 인덱스 버퍼의 내용을 나타냅니다. 이 배열은 그림의 인덱스와 일치시켜 오른쪽 위 삼각형과 왼쪽 아래 삼각형을 그려야 합니다.

```c++
const std::vector<uint16_t> indices = {
    0, 1, 2, 2, 3, 0
};
```

`vertices`의 항목 수에 따라 인덱스 버퍼에 `uint16_t` 또는 `uint32_t`를 사용할 수 있습니다. 65535개 미만의 고유 정점을 사용하므로 지금은 `uint16_t`를 사용하겠습니다.

정점 데이터와 마찬가지로, 인덱스도 GPU가 접근할 수 있도록 `VkBuffer`에 업로드해야 합니다. 인덱스 버퍼의 리소스를 저장하기 위해 두 개의 새로운 클래스 멤버를 정의합니다.

```c++
VkBuffer vertexBuffer;
VkDeviceMemory vertexBufferMemory;
VkBuffer indexBuffer;
VkDeviceMemory indexBufferMemory;
```

이제 추가할 `createIndexBuffer` 함수는 `createVertexBuffer`와 거의 동일합니다.

```c++
void initVulkan() {
    ...
    createVertexBuffer();
    createIndexBuffer();
    ...
}

void createIndexBuffer() {
    VkDeviceSize bufferSize = sizeof(indices[0]) * indices.size();

    VkBuffer stagingBuffer;
    VkDeviceMemory stagingBufferMemory;
    createBuffer(bufferSize, VK_BUFFER_USAGE_TRANSFER_SRC_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, stagingBuffer, stagingBufferMemory);

    void* data;
    vkMapMemory(device, stagingBufferMemory, 0, bufferSize, 0, &data);
    memcpy(data, indices.data(), (size_t) bufferSize);
    vkUnmapMemory(device, stagingBufferMemory);

    createBuffer(bufferSize, VK_BUFFER_USAGE_TRANSFER_DST_BIT | VK_BUFFER_USAGE_INDEX_BUFFER_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, indexBuffer, indexBufferMemory);

    copyBuffer(stagingBuffer, indexBuffer, bufferSize);

    vkDestroyBuffer(device, stagingBuffer, nullptr);
    vkFreeMemory(device, stagingBufferMemory, nullptr);
}
```

주목할 만한 차이점은 두 가지뿐입니다. `bufferSize`는 이제 인덱스 수에 인덱스 타입(`uint16_t` 또는 `uint32_t`)의 크기를 곱한 값과 같습니다. `indexBuffer`의 usage는 `VK_BUFFER_USAGE_VERTEX_BUFFER_BIT` 대신 `VK_BUFFER_USAGE_INDEX_BUFFER_BIT`이어야 하는데, 이는 타당한 설정입니다. 그 외의 과정은 정확히 동일합니다. `indices`의 내용을 복사하기 위한 스테이징 버퍼를 만들고, 그 내용을 최종 장치 로컬 인덱스 버퍼로 복사합니다.

인덱스 버퍼는 정점 버퍼와 마찬가지로 프로그램이 끝날 때 정리해야 합니다.

```c++
void cleanup() {
    cleanupSwapChain();

    vkDestroyBuffer(device, indexBuffer, nullptr);
    vkFreeMemory(device, indexBufferMemory, nullptr);

    vkDestroyBuffer(device, vertexBuffer, nullptr);
    vkFreeMemory(device, vertexBufferMemory, nullptr);

    ...
}
```

## 인덱스 버퍼 사용하기

그리기에 인덱스 버퍼를 사용하는 것은 `recordCommandBuffer`에 두 가지 변경 사항을 수반합니다. 먼저 정점 버퍼에 했던 것처럼 인덱스 버퍼를 바인딩해야 합니다. 차이점은 인덱스 버퍼는 하나만 가질 수 있다는 것입니다. 아쉽게도 각 정점 속성에 대해 서로 다른 인덱스를 사용하는 것은 불가능하므로, 속성 하나만 다르더라도 정점 데이터를 완전히 복제해야 합니다.

```c++
vkCmdBindVertexBuffers(commandBuffer, 0, 1, vertexBuffers, offsets);

vkCmdBindIndexBuffer(commandBuffer, indexBuffer, 0, VK_INDEX_TYPE_UINT16);
```

`vkCmdBindIndexBuffer`를 사용하여 인덱스 버퍼를 바인딩하며, 이 함수는 인덱스 버퍼, 버퍼 내의 바이트 오프셋, 그리고 인덱스 데이터의 타입을 매개변수로 받습니다. 앞서 언급했듯이, 가능한 타입은 `VK_INDEX_TYPE_UINT16`과 `VK_INDEX_TYPE_UINT32`입니다.

인덱스 버퍼를 바인딩하는 것만으로는 아직 아무것도 바뀌지 않으며, Vulkan에 인덱스 버퍼를 사용하도록 지시하기 위해 그리기 명령도 변경해야 합니다. `vkCmdDraw` 라인을 제거하고 `vkCmdDrawIndexed`로 교체합니다.

```c++
vkCmdDrawIndexed(commandBuffer, static_cast<uint32_t>(indices.size()), 1, 0, 0, 0);
```

이 함수 호출은 `vkCmdDraw`와 매우 유사합니다. 처음 두 매개변수는 인덱스의 수와 인스턴스의 수를 지정합니다. 인스턴싱은 사용하지 않으므로 인스턴스는 `1`로 지정합니다. 인덱스의 수는 정점 셰이더로 전달될 정점의 수를 나타냅니다. 다음 매개변수는 인덱스 버퍼로의 오프셋을 지정하며, 값으로 `1`을 사용하면 그래픽 카드가 두 번째 인덱스부터 읽기 시작합니다. 끝에서 두 번째 매개변수는 인덱스 버퍼의 인덱스에 더할 오프셋을 지정합니다. 마지막 매개변수는 인스턴싱을 위한 오프셋을 지정하는데, 우리는 사용하지 않습니다.

이제 프로그램을 실행하면 다음과 같은 결과가 나타나야 합니다.

![](/images/indexed_rectangle.png)

이제 인덱스 버퍼를 사용하여 정점을 재사용함으로써 메모리를 절약하는 방법을 알게 되었습니다. 이는 나중에 복잡한 3D 모델을 로드할 장에서 특히 중요해질 것입니다.

이전 장에서 이미 여러 리소스(예: 버퍼)를 단일 메모리 할당에서 할당해야 한다고 언급했지만, 사실은 한 단계 더 나아가야 합니다. [드라이버 개발자들은](https://developer.nvidia.com/vulkan-memory-management) 정점 버퍼와 인덱스 버퍼 같은 여러 버퍼를 단일 `VkBuffer`에 저장하고 `vkCmdBindVertexBuffers`와 같은 명령어에서 오프셋을 사용할 것을 권장합니다. 이렇게 하면 데이터가 더 가깝게 모여있기 때문에 캐시 친화적(cache friendly)이라는 장점이 있습니다. 동일한 렌더링 작업 중에 사용되지 않는 여러 리소스에 대해 동일한 메모리 청크를 재사용하는 것도 가능합니다. 물론 데이터는 새로고침되어야 합니다. 이를 *에일리어싱(aliasing)*이라고 하며, 일부 Vulkan 함수에는 이를 원한다고 명시적으로 지정하는 플래그가 있습니다.

[C++ 코드](/code/21_index_buffer.cpp) /
[정점 셰이더](/code/18_shader_vertexbuffer.vert) /
[프래그먼트 셰이더](/code/18_shader_vertexbuffer.frag)