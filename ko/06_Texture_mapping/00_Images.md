## 서론

지금까지는 정점별(per-vertex) 색상을 사용해 지오메트리를 색칠해왔는데, 이는 다소 제한적인 방법입니다. 이번 장에서는 텍스처 매핑을 구현하여 지오메트리가 더 흥미롭게 보이도록 만들 것입니다. 이를 통해 다음 장에서는 기본적인 3D 모델을 로드하고 그릴 수도 있게 됩니다.

애플리케이션에 텍스처를 추가하는 작업은 다음 단계를 포함합니다.

*   디바이스 메모리를 기반으로 하는 이미지 객체 생성
*   이미지 파일의 픽셀로 채우기
*   이미지 샘플러 생성
*   텍스처에서 색상을 샘플링하기 위한 결합 이미지 샘플러 디스크립터 추가

이전에도 이미지 객체를 다룬 적이 있지만, 그것들은 스왑체인 확장에 의해 자동으로 생성되었습니다. 이번에는 직접 하나를 만들어야 합니다. 이미지를 생성하고 데이터를 채우는 과정은 정점 버퍼 생성과 유사합니다. 먼저 스테이징 리소스를 생성하고 픽셀 데이터로 채운 다음, 렌더링에 사용할 최종 이미지 객체로 복사합니다. 이 목적으로 스테이징 이미지를 생성할 수도 있지만, Vulkan은 `VkBuffer`에서 이미지로 픽셀을 복사하는 것도 허용하며, 이 API는 [일부 하드웨어에서 실제로 더 빠릅니다](https://developer.nvidia.com/vulkan-memory-management). 우리는 먼저 이 버퍼를 생성하고 픽셀 값으로 채운 다음, 픽셀을 복사할 이미지를 생성할 것입니다. 이미지 생성은 버퍼 생성과 크게 다르지 않습니다. 이전에 보았듯이 메모리 요구사항을 쿼리하고, 디바이스 메모리를 할당하고, 바인딩하는 과정이 포함됩니다.

하지만 이미지 작업 시에는 추가적으로 신경 써야 할 것이 있습니다. 이미지는 메모리에서 픽셀이 구성되는 방식에 영향을 미치는 다양한 *레이아웃(layouts)*을 가질 수 있습니다. 그래픽 하드웨어의 작동 방식 때문에, 단순히 픽셀을 행 단위로 저장하는 것이 최상의 성능으로 이어지지 않을 수 있습니다. 이미지에 대한 어떠한 작업을 수행할 때든, 해당 작업에 최적화된 레이아웃을 가지고 있는지 확인해야 합니다. 렌더 패스를 지정할 때 이미 이러한 레이아웃 중 일부를 본 적이 있습니다:

*   `VK_IMAGE_LAYOUT_PRESENT_SRC_KHR`: 화면 표시에 최적화
*   `VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL`: 프래그먼트 셰이더에서 색상을 쓰는 어태치먼트로 최적화
*   `VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL`: `vkCmdCopyImageToBuffer`와 같은 전송 작업의 소스로 최적화
*   `VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL`: `vkCmdCopyBufferToImage`와 같은 전송 작업의 대상으로 최적화
*   `VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL`: 셰이더에서 샘플링하기에 최적화

이미지의 레이아웃을 전환하는 가장 일반적인 방법 중 하나는 *파이프라인 배리어(pipeline barrier)*입니다. 파이프라인 배리어는 주로 리소스 접근을 동기화하는 데 사용됩니다. 예를 들어, 이미지를 읽기 전에 쓰기가 완료되었는지 확인하는 것과 같지만, 레이아웃을 전환하는 데에도 사용할 수 있습니다. 이번 장에서는 파이프라인 배리어가 이 목적으로 어떻게 사용되는지 볼 것입니다. 배리어는 `VK_SHARING_MODE_EXCLUSIVE`를 사용할 때 큐 패밀리 소유권을 이전하는 데에도 추가적으로 사용될 수 있습니다.

## 이미지 라이브러리

이미지를 로드하기 위한 많은 라이브러리가 있으며, BMP나 PPM과 같은 간단한 포맷을 로드하는 코드를 직접 작성할 수도 있습니다. 이 튜토리얼에서는 [stb collection](https://github.com/nothings/stb)의 stb_image 라이브러리를 사용할 것입니다. 이 라이브러리의 장점은 모든 코드가 단일 파일에 있어 빌드 구성을 복잡하게 할 필요가 없다는 것입니다. `stb_image.h`를 다운로드하여 GLFW와 GLM을 저장한 디렉토리와 같이 편리한 위치에 저장하고, 해당 위치를 인클루드 경로에 추가하십시오.

**Visual Studio**

`stb_image.h`가 있는 디렉토리를 `Additional Include Directories` 경로에 추가합니다.

![](/images/include_dirs_stb.png)

**Makefile**

`stb_image.h`가 있는 디렉토리를 GCC의 인클루드 디렉토리에 추가합니다:

```text
VULKAN_SDK_PATH = /home/user/VulkanSDK/x.x.x.x/x86_64
STB_INCLUDE_PATH = /home/user/libraries/stb

...

CFLAGS = -std=c++17 -I$(VULKAN_SDK_PATH)/include -I$(STB_INCLUDE_PATH)
```

## 이미지 로딩하기

다음과 같이 이미지 라이브러리를 포함시킵니다:

```c++
#define STB_IMAGE_IMPLEMENTATION
#include <stb_image.h>
```

헤더는 기본적으로 함수의 프로토타입만 정의합니다. 하나의 코드 파일에서 `STB_IMAGE_IMPLEMENTATION` 정의와 함께 헤더를 포함해야 함수 본문이 포함되어 링킹 오류가 발생하지 않습니다.

```c++
void initVulkan() {
    ...
    createCommandPool();
    createTextureImage();
    createVertexBuffer();
    ...
}

...

void createTextureImage() {

}
```

`createTextureImage`라는 새 함수를 만들어 이미지를 로드하고 Vulkan 이미지 객체에 업로드할 것입니다. 커맨드 버퍼를 사용할 것이므로 `createCommandPool` 이후에 호출되어야 합니다.

`shaders` 디렉토리 옆에 `textures`라는 새 디렉토리를 만들어 텍스처 이미지를 저장합니다. 해당 디렉토리에서 `texture.jpg`라는 이미지를 로드할 것입니다. 저는 512 x 512 픽셀로 리사이즈된 다음 [CC0 라이선스 이미지](https://pixabay.com/en/statue-sculpture-fig-historically-1275469/)를 사용하기로 했지만, 원하는 이미지를 자유롭게 선택해도 좋습니다. 이 라이브러리는 JPEG, PNG, BMP, GIF와 같은 대부분의 일반적인 이미지 파일 형식을 지원합니다.

![](/images/texture.jpg)

이 라이브러리로 이미지를 로드하는 것은 정말 쉽습니다:

```c++
void createTextureImage() {
    int texWidth, texHeight, texChannels;
    stbi_uc* pixels = stbi_load("textures/texture.jpg", &texWidth, &texHeight, &texChannels, STBI_rgb_alpha);
    VkDeviceSize imageSize = texWidth * texHeight * 4;

    if (!pixels) {
        throw std::runtime_error("failed to load texture image!");
    }
}
```

`stbi_load` 함수는 파일 경로와 로드할 채널 수를 인자로 받습니다. `STBI_rgb_alpha` 값은 이미지가 알파 채널을 가지고 있지 않더라도 강제로 알파 채널과 함께 로드하도록 하여, 나중에 다른 텍스처와의 일관성을 유지하는 데 좋습니다. 가운데 세 개의 매개변수는 이미지의 너비, 높이, 실제 채널 수에 대한 출력입니다. 반환되는 포인터는 픽셀 값 배열의 첫 번째 요소입니다. 픽셀은 `STBI_rgb_alpha`의 경우 픽셀당 4바이트로 행 단위로 배치되어 총 `texWidth * texHeight * 4`개의 값을 가집니다.

## 스테이징 버퍼

이제 호스트 가시성(host visible) 메모리에 버퍼를 만들어 `vkMapMemory`를 사용하고 픽셀을 복사할 수 있도록 하겠습니다. 이 임시 버퍼를 위한 변수들을 `createTextureImage` 함수에 추가합니다:

```c++
VkBuffer stagingBuffer;
VkDeviceMemory stagingBufferMemory;
```

버퍼는 매핑할 수 있도록 호스트 가시성 메모리에 있어야 하며, 나중에 이미지로 복사할 수 있도록 전송 소스로 사용될 수 있어야 합니다:

```c++
createBuffer(imageSize, VK_BUFFER_USAGE_TRANSFER_SRC_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, stagingBuffer, stagingBufferMemory);
```

그런 다음 이미지 로딩 라이브러리에서 얻은 픽셀 값을 버퍼에 직접 복사할 수 있습니다:

```c++
void* data;
vkMapMemory(device, stagingBufferMemory, 0, imageSize, 0, &data);
    memcpy(data, pixels, static_cast<size_t>(imageSize));
vkUnmapMemory(device, stagingBufferMemory);
```

이제 원본 픽셀 배열을 정리하는 것을 잊지 마세요:

```c++
stbi_image_free(pixels);
```

## 텍스처 이미지

셰이더가 버퍼의 픽셀 값에 접근하도록 설정할 수도 있지만, Vulkan에서는 이 목적으로 이미지 객체를 사용하는 것이 더 좋습니다. 이미지 객체는 2D 좌표를 사용할 수 있게 하여 색상을 더 쉽고 빠르게 가져올 수 있게 해줍니다. 이미지 객체 내의 픽셀은 텍셀(texel)이라고 하며, 지금부터 이 용어를 사용하겠습니다. 다음의 새 클래스 멤버를 추가합니다:

```c++
VkImage textureImage;
VkDeviceMemory textureImageMemory;
```

이미지의 매개변수는 `VkImageCreateInfo` 구조체에 지정됩니다:

```c++
VkImageCreateInfo imageInfo{};
imageInfo.sType = VK_STRUCTURE_TYPE_IMAGE_CREATE_INFO;
imageInfo.imageType = VK_IMAGE_TYPE_2D;
imageInfo.extent.width = static_cast<uint32_t>(texWidth);
imageInfo.extent.height = static_cast<uint32_t>(texHeight);
imageInfo.extent.depth = 1;
imageInfo.mipLevels = 1;
imageInfo.arrayLayers = 1;
```

`imageType` 필드에 지정된 이미지 타입은 Vulkan에게 이미지의 텍셀이 어떤 종류의 좌표계로 주소 지정될 것인지를 알려줍니다. 1D, 2D, 3D 이미지를 생성할 수 있습니다. 1차원 이미지는 데이터 배열이나 그래디언트를 저장하는 데 사용될 수 있고, 2차원 이미지는 주로 텍스처에 사용되며, 3차원 이미지는 복셀 볼륨 등을 저장하는 데 사용될 수 있습니다. `extent` 필드는 이미지의 차원, 즉 각 축에 얼마나 많은 텍셀이 있는지를 지정합니다. 이것이 `depth`가 `0`이 아닌 `1`이어야 하는 이유입니다. 우리 텍스처는 배열이 아니며, 지금은 밉매핑을 사용하지 않을 것입니다.

```c++
imageInfo.format = VK_FORMAT_R8G8B8A8_SRGB;
```

Vulkan은 다양한 이미지 형식을 지원하지만, 버퍼의 픽셀과 동일한 형식을 텍셀에 사용해야 합니다. 그렇지 않으면 복사 작업이 실패합니다.

```c++
imageInfo.tiling = VK_IMAGE_TILING_OPTIMAL;
```

`tiling` 필드는 다음 두 가지 값 중 하나를 가질 수 있습니다:

*   `VK_IMAGE_TILING_LINEAR`: 텍셀이 우리의 `pixels` 배열처럼 행 우선 순서(row-major order)로 배치됩니다.
*   `VK_IMAGE_TILING_OPTIMAL`: 최적의 접근을 위해 구현에 따라 정의된 순서로 텍셀이 배치됩니다.

이미지의 레이아웃과 달리 타일링 모드는 나중에 변경할 수 없습니다. 이미지 메모리에서 텍셀에 직접 접근하고 싶다면 `VK_IMAGE_TILING_LINEAR`를 사용해야 합니다. 우리는 스테이징 이미지 대신 스테이징 버퍼를 사용할 것이므로 이것은 필요하지 않습니다. 셰이더에서의 효율적인 접근을 위해 `VK_IMAGE_TILING_OPTIMAL`을 사용할 것입니다.

```c++
imageInfo.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
```

이미지의 `initialLayout`에는 두 가지 가능한 값만 있습니다:

*   `VK_IMAGE_LAYOUT_UNDEFINED`: GPU에서 사용할 수 없으며, 첫 번째 전환 시 텍셀이 폐기됩니다.
*   `VK_IMAGE_LAYOUT_PREINITIALIZED`: GPU에서 사용할 수 없지만, 첫 번째 전환 시 텍셀이 보존됩니다.

첫 번째 전환 중에 텍셀이 보존되어야 하는 경우는 거의 없습니다. 그러나 한 가지 예는 `VK_IMAGE_TILING_LINEAR` 레이아웃과 함께 이미지를 스테이징 이미지로 사용하려는 경우입니다. 이 경우 텍셀 데이터를 업로드한 다음 데이터를 잃지 않고 이미지를 전송 소스로 전환하고 싶을 것입니다. 하지만 우리의 경우에는 먼저 이미지를 전송 대상으로 전환한 다음 버퍼 객체에서 텍셀 데이터를 복사할 것이므로 이 속성이 필요 없으며 `VK_IMAGE_LAYOUT_UNDEFINED`를 안전하게 사용할 수 있습니다.

```c++
imageInfo.usage = VK_IMAGE_USAGE_TRANSFER_DST_BIT | VK_IMAGE_USAGE_SAMPLED_BIT;
```

`usage` 필드는 버퍼 생성 시와 동일한 의미를 갖습니다. 이미지는 버퍼 복사의 대상으로 사용될 것이므로 전송 대상으로 설정되어야 합니다. 또한 셰이더에서 이미지에 접근하여 메쉬를 색칠하고 싶으므로 `VK_IMAGE_USAGE_SAMPLED_BIT`를 포함해야 합니다.

```c++
imageInfo.sharingMode = VK_SHARING_MODE_EXCLUSIVE;
```

이미지는 그래픽(따라서 전송도) 작업을 지원하는 단일 큐 패밀리에서만 사용될 것입니다.

```c++
imageInfo.samples = VK_SAMPLE_COUNT_1_BIT;
imageInfo.flags = 0; // Optional
```

`samples` 플래그는 멀티샘플링과 관련이 있습니다. 이것은 어태치먼트로 사용될 이미지에만 관련이 있으므로 하나의 샘플로 고정합니다. 희소 이미지(sparse images)와 관련된 몇 가지 선택적 플래그가 있습니다. 희소 이미지는 특정 영역만 실제로 메모리에 의해 지원되는 이미지입니다. 예를 들어, 복셀 지형에 3D 텍스처를 사용하는 경우, 이를 사용하여 "공기" 값의 큰 볼륨을 저장하기 위한 메모리 할당을 피할 수 있습니다. 이 튜토리얼에서는 사용하지 않을 것이므로 기본값인 `0`으로 둡니다.

```c++
if (vkCreateImage(device, &imageInfo, nullptr, &textureImage) != VK_SUCCESS) {
    throw std::runtime_error("failed to create image!");
}
```

이미지는 `vkCreateImage`를 사용하여 생성되며, 특별히 주목할 만한 매개변수는 없습니다. `VK_FORMAT_R8G8B8A8_SRGB` 형식이 그래픽 하드웨어에서 지원되지 않을 수 있습니다. 수용 가능한 대안 목록을 가지고 지원되는 최상의 것을 선택해야 합니다. 그러나 이 특정 형식에 대한 지원은 매우 널리 퍼져 있으므로 이 단계를 건너뛰겠습니다. 다른 형식을 사용하려면 번거로운 변환이 필요할 수도 있습니다. 깊이 버퍼 장에서 이러한 시스템을 구현할 때 이 문제로 다시 돌아올 것입니다.

```c++
VkMemoryRequirements memRequirements;
vkGetImageMemoryRequirements(device, textureImage, &memRequirements);

VkMemoryAllocateInfo allocInfo{};
allocInfo.sType = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO;
allocInfo.allocationSize = memRequirements.size;
allocInfo.memoryTypeIndex = findMemoryType(memRequirements.memoryTypeBits, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT);

if (vkAllocateMemory(device, &allocInfo, nullptr, &textureImageMemory) != VK_SUCCESS) {
    throw std::runtime_error("failed to allocate image memory!");
}

vkBindImageMemory(device, textureImage, textureImageMemory, 0);
```

이미지용 메모리 할당은 버퍼용 메모리 할당과 정확히 동일한 방식으로 작동합니다. `vkGetBufferMemoryRequirements` 대신 `vkGetImageMemoryRequirements`를 사용하고, `vkBindBufferMemory` 대신 `vkBindImageMemory`를 사용합니다.

이 함수는 이미 상당히 커지고 있으며, 이후 장에서 더 많은 이미지를 생성할 필요가 있으므로, 버퍼에서 했던 것처럼 이미지 생성을 `createImage` 함수로 추상화해야 합니다. 함수를 만들고 이미지 객체 생성 및 메모리 할당을 그곳으로 옮깁니다:

```c++
void createImage(uint32_t width, uint32_t height, VkFormat format, VkImageTiling tiling, VkImageUsageFlags usage, VkMemoryPropertyFlags properties, VkImage& image, VkDeviceMemory& imageMemory) {
    VkImageCreateInfo imageInfo{};
    imageInfo.sType = VK_STRUCTURE_TYPE_IMAGE_CREATE_INFO;
    imageInfo.imageType = VK_IMAGE_TYPE_2D;
    imageInfo.extent.width = width;
    imageInfo.extent.height = height;
    imageInfo.extent.depth = 1;
    imageInfo.mipLevels = 1;
    imageInfo.arrayLayers = 1;
    imageInfo.format = format;
    imageInfo.tiling = tiling;
    imageInfo.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
    imageInfo.usage = usage;
    imageInfo.samples = VK_SAMPLE_COUNT_1_BIT;
    imageInfo.sharingMode = VK_SHARING_MODE_EXCLUSIVE;

    if (vkCreateImage(device, &imageInfo, nullptr, &image) != VK_SUCCESS) {
        throw std::runtime_error("failed to create image!");
    }

    VkMemoryRequirements memRequirements;
    vkGetImageMemoryRequirements(device, image, &memRequirements);

    VkMemoryAllocateInfo allocInfo{};
    allocInfo.sType = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO;
    allocInfo.allocationSize = memRequirements.size;
    allocInfo.memoryTypeIndex = findMemoryType(memRequirements.memoryTypeBits, properties);

    if (vkAllocateMemory(device, &allocInfo, nullptr, &imageMemory) != VK_SUCCESS) {
        throw std::runtime_error("failed to allocate image memory!");
    }

    vkBindImageMemory(device, image, imageMemory, 0);
}
```

너비, 높이, 형식, 타일링 모드, 사용법 및 메모리 속성을 매개변수로 만들었는데, 이는 이 튜토리얼 전체에서 생성할 이미지마다 달라질 것이기 때문입니다.

이제 `createTextureImage` 함수를 다음과 같이 단순화할 수 있습니다:

```c++
void createTextureImage() {
    int texWidth, texHeight, texChannels;
    stbi_uc* pixels = stbi_load("textures/texture.jpg", &texWidth, &texHeight, &texChannels, STBI_rgb_alpha);
    VkDeviceSize imageSize = texWidth * texHeight * 4;

    if (!pixels) {
        throw std::runtime_error("failed to load texture image!");
    }

    VkBuffer stagingBuffer;
    VkDeviceMemory stagingBufferMemory;
    createBuffer(imageSize, VK_BUFFER_USAGE_TRANSFER_SRC_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, stagingBuffer, stagingBufferMemory);

    void* data;
    vkMapMemory(device, stagingBufferMemory, 0, imageSize, 0, &data);
        memcpy(data, pixels, static_cast<size_t>(imageSize));
    vkUnmapMemory(device, stagingBufferMemory);

    stbi_image_free(pixels);

    createImage(texWidth, texHeight, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_TILING_OPTIMAL, VK_IMAGE_USAGE_TRANSFER_DST_BIT | VK_IMAGE_USAGE_SAMPLED_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, textureImage, textureImageMemory);
}
```

## 레이아웃 전환

지금 작성할 함수는 커맨드 버퍼를 다시 기록하고 실행하는 것을 포함하므로, 이 로직을 한두 개의 헬퍼 함수로 옮기기에 좋은 시점입니다:

```c++
VkCommandBuffer beginSingleTimeCommands() {
    VkCommandBufferAllocateInfo allocInfo{};
    allocInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_ALLOCATE_INFO;
    allocInfo.level = VK_COMMAND_BUFFER_LEVEL_PRIMARY;
    allocInfo.commandPool = commandPool;
    allocInfo.commandBufferCount = 1;

    VkCommandBuffer commandBuffer;
    vkAllocateCommandBuffers(device, &allocInfo, &commandBuffer);

    VkCommandBufferBeginInfo beginInfo{};
    beginInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO;
    beginInfo.flags = VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT;

    vkBeginCommandBuffer(commandBuffer, &beginInfo);

    return commandBuffer;
}

void endSingleTimeCommands(VkCommandBuffer commandBuffer) {
    vkEndCommandBuffer(commandBuffer);

    VkSubmitInfo submitInfo{};
    submitInfo.sType = VK_STRUCTURE_TYPE_SUBMIT_INFO;
    submitInfo.commandBufferCount = 1;
    submitInfo.pCommandBuffers = &commandBuffer;

    vkQueueSubmit(graphicsQueue, 1, &submitInfo, VK_NULL_HANDLE);
    vkQueueWaitIdle(graphicsQueue);

    vkFreeCommandBuffers(device, commandPool, 1, &commandBuffer);
}
```

이 함수들의 코드는 `copyBuffer`의 기존 코드를 기반으로 합니다. 이제 해당 함수를 다음과 같이 단순화할 수 있습니다:

```c++
void copyBuffer(VkBuffer srcBuffer, VkBuffer dstBuffer, VkDeviceSize size) {
    VkCommandBuffer commandBuffer = beginSingleTimeCommands();

    VkBufferCopy copyRegion{};
    copyRegion.size = size;
    vkCmdCopyBuffer(commandBuffer, srcBuffer, dstBuffer, 1, &copyRegion);

    endSingleTimeCommands(commandBuffer);
}
```

만약 여전히 버퍼를 사용하고 있었다면, `vkCmdCopyBufferToImage`를 기록하고 실행하는 함수를 작성하여 작업을 마칠 수 있었겠지만, 이 명령어는 이미지가 먼저 올바른 레이아웃에 있어야 합니다. 레이아웃 전환을 처리할 새 함수를 만듭니다:

```c++
void transitionImageLayout(VkImage image, VkFormat format, VkImageLayout oldLayout, VkImageLayout newLayout) {
    VkCommandBuffer commandBuffer = beginSingleTimeCommands();

    endSingleTimeCommands(commandBuffer);
}
```

레이아웃 전환을 수행하는 가장 일반적인 방법 중 하나는 *이미지 메모리 배리어(image memory barrier)*를 사용하는 것입니다. 이러한 파이프라인 배리어는 일반적으로 리소스 접근을 동기화하는 데 사용됩니다(예: 버퍼에서 읽기 전에 쓰기가 완료되었는지 확인). 하지만 `VK_SHARING_MODE_EXCLUSIVE`가 사용될 때 이미지 레이아웃을 전환하고 큐 패밀리 소유권을 이전하는 데에도 사용할 수 있습니다. 버퍼에 대해 이를 수행하기 위한 동등한 *버퍼 메모리 배리어*가 있습니다.

```c++
VkImageMemoryBarrier barrier{};
barrier.sType = VK_STRUCTURE_TYPE_IMAGE_MEMORY_BARRIER;
barrier.oldLayout = oldLayout;
barrier.newLayout = newLayout;
```

첫 두 필드는 레이아웃 전환을 지정합니다. 이미지의 기존 내용에 신경 쓰지 않는다면 `oldLayout`으로 `VK_IMAGE_LAYOUT_UNDEFINED`를 사용할 수 있습니다.

```c++
barrier.srcQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED;
barrier.dstQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED;
```

큐 패밀리 소유권을 이전하기 위해 배리어를 사용하는 경우, 이 두 필드는 큐 패밀리의 인덱스가 되어야 합니다. 이를 원하지 않는 경우 (기본값이 아님!) `VK_QUEUE_FAMILY_IGNORED`로 설정해야 합니다.

```c++
barrier.image = image;
barrier.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
barrier.subresourceRange.baseMipLevel = 0;
barrier.subresourceRange.levelCount = 1;
barrier.subresourceRange.baseArrayLayer = 0;
barrier.subresourceRange.layerCount = 1;
```

`image`와 `subresourceRange`는 영향을 받는 이미지와 이미지의 특정 부분을 지정합니다. 우리 이미지는 배열이 아니고 밉매핑 레벨도 없으므로, 하나의 레벨과 레이어만 지정됩니다.

```c++
barrier.srcAccessMask = 0; // TODO
barrier.dstAccessMask = 0; // TODO
```

배리어는 주로 동기화 목적으로 사용되므로, 배리어 이전에 발생해야 하는 리소스 관련 작업 유형과 배리어를 기다려야 하는 리소스 관련 작업 유형을 지정해야 합니다. 이미 `vkQueueWaitIdle`을 사용하여 수동으로 동기화하고 있음에도 불구하고 이를 수행해야 합니다. 올바른 값은 이전 및 새 레이아웃에 따라 달라지므로, 어떤 전환을 사용할지 파악한 후에 이 부분으로 돌아오겠습니다.

```c++
vkCmdPipelineBarrier(
    commandBuffer,
    0 /* TODO */, 0 /* TODO */,
    0,
    0, nullptr,
    0, nullptr,
    1, &barrier
);
```

모든 유형의 파이프라인 배리어는 동일한 함수를 사용하여 제출됩니다. 커맨드 버퍼 다음의 첫 번째 매개변수는 배리어 이전에 발생해야 하는 작업이 일어나는 파이프라인 스테이지를 지정합니다. 두 번째 매개변수는 배리어를 기다릴 작업이 일어나는 파이프라인 스테이지를 지정합니다. 배리어 전후에 지정할 수 있는 파이프라인 스테이지는 배리어 전후에 리소스를 어떻게 사용하느냐에 따라 다릅니다. 허용되는 값은 사양의 [이 표](https://www.khronos.org/registry/vulkan/specs/1.3-extensions/html/chap7.html#synchronization-access-types-supported)에 나열되어 있습니다. 예를 들어, 배리어 이후에 유니폼에서 읽으려는 경우 `VK_ACCESS_UNIFORM_READ_BIT` 사용법과 유니폼을 읽을 가장 빠른 셰이더(예: `VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT`)를 파이프라인 스테이지로 지정합니다. 이 유형의 사용법에 대해 비-셰이더 파이프라인 스테이지를 지정하는 것은 의미가 없으며, 유효성 검사 계층은 사용 유형과 일치하지 않는 파이프라인 스테이지를 지정할 때 경고합니다.

세 번째 매개변수는 `0` 또는 `VK_DEPENDENCY_BY_REGION_BIT`입니다. 후자는 배리어를 영역별 조건으로 바꿉니다. 이는 구현이 예를 들어, 이미 쓰여진 리소스의 부분부터 읽기 시작할 수 있음을 의미합니다.

마지막 세 쌍의 매개변수는 사용 가능한 세 가지 유형의 파이프라인 배리어(메모리 배리어, 버퍼 메모리 배리어, 그리고 우리가 사용하는 이미지 메모리 배리어)의 배열을 참조합니다. 아직 `VkFormat` 매개변수를 사용하지 않았지만, 깊이 버퍼 장에서 특별한 전환을 위해 사용할 것입니다.

## 버퍼를 이미지로 복사하기

`createTextureImage`로 돌아가기 전에, `copyBufferToImage`라는 헬퍼 함수를 하나 더 작성하겠습니다:

```c++
void copyBufferToImage(VkBuffer buffer, VkImage image, uint32_t width, uint32_t height) {
    VkCommandBuffer commandBuffer = beginSingleTimeCommands();

    endSingleTimeCommands(commandBuffer);
}
```

버퍼 복사와 마찬가지로, 버퍼의 어느 부분을 이미지의 어느 부분으로 복사할지 지정해야 합니다. 이것은 `VkBufferImageCopy` 구조체를 통해 이루어집니다:

```c++
VkBufferImageCopy region{};
region.bufferOffset = 0;
region.bufferRowLength = 0;
region.bufferImageHeight = 0;

region.imageSubresource.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
region.imageSubresource.mipLevel = 0;
region.imageSubresource.baseArrayLayer = 0;
region.imageSubresource.layerCount = 1;

region.imageOffset = {0, 0, 0};
region.imageExtent = {
    width,
    height,
    1
};
```

이 필드들의 대부분은 자명합니다. `bufferOffset`은 픽셀 값이 시작되는 버퍼의 바이트 오프셋을 지정합니다. `bufferRowLength`와 `bufferImageHeight` 필드는 픽셀이 메모리에 어떻게 배치되는지를 지정합니다. 예를 들어, 이미지의 행 사이에 패딩 바이트가 있을 수 있습니다. 두 필드 모두 `0`으로 지정하면 우리 경우처럼 픽셀이 단순히 빽빽하게 채워져 있음을 나타냅니다. `imageSubresource`, `imageOffset`, `imageExtent` 필드는 픽셀을 이미지의 어느 부분으로 복사할지를 나타냅니다.

버퍼에서 이미지로의 복사 작업은 `vkCmdCopyBufferToImage` 함수를 사용하여 큐에 추가됩니다:

```c++
vkCmdCopyBufferToImage(
    commandBuffer,
    buffer,
    image,
    VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL,
    1,
    &region
);
```

네 번째 매개변수는 이미지가 현재 사용하고 있는 레이아웃을 나타냅니다. 여기서는 이미지가 픽셀을 복사하기에 최적화된 레이아웃으로 이미 전환되었다고 가정합니다. 지금은 픽셀 덩어리 하나를 전체 이미지에 복사하고 있지만, `VkBufferImageCopy`의 배열을 지정하여 이 버퍼에서 이미지로 여러 다른 복사를 한 번의 작업으로 수행할 수도 있습니다.

## 텍스처 이미지 준비하기

이제 텍스처 이미지 설정을 마치는 데 필요한 모든 도구를 갖추었으므로 `createTextureImage` 함수로 돌아갑니다. 거기서 마지막으로 한 일은 텍스처 이미지를 생성하는 것이었습니다. 다음 단계는 스테이징 버퍼를 텍스처 이미지로 복사하는 것입니다. 이는 두 단계로 이루어집니다:

*   텍스처 이미지를 `VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL`로 전환
*   버퍼에서 이미지로의 복사 작업 실행

방금 만든 함수들로 이 작업은 쉽게 할 수 있습니다:

```c++
transitionImageLayout(textureImage, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_LAYOUT_UNDEFINED, VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL);
copyBufferToImage(stagingBuffer, textureImage, static_cast<uint32_t>(texWidth), static_cast<uint32_t>(texHeight));
```

이미지는 `VK_IMAGE_LAYOUT_UNDEFINED` 레이아웃으로 생성되었으므로, `textureImage`를 전환할 때 이전 레이아웃으로 지정되어야 합니다. 복사 작업을 수행하기 전에 그 내용에 신경 쓰지 않기 때문에 이렇게 할 수 있다는 것을 기억하세요.

셰이더에서 텍스처 이미지로부터 샘플링을 시작하려면, 셰이더 접근을 위해 준비하는 마지막 전환이 한 번 더 필요합니다:

```c++
transitionImageLayout(textureImage, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL, VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL);
```

## 전환 배리어 마스크

이제 유효성 검사 계층을 활성화한 상태로 애플리케이션을 실행하면, `transitionImageLayout`의 접근 마스크와 파이프라인 스테이지가 유효하지 않다고 불평하는 것을 볼 수 있습니다. 전환의 레이아웃에 따라 이들을 아직 설정해야 합니다.

처리해야 할 두 가지 전환이 있습니다:

*   정의되지 않음 → 전송 대상: 어떤 것도 기다릴 필요 없는 전송 쓰기
*   전송 대상 → 셰이더 읽기: 셰이더 읽기는 전송 쓰기를 기다려야 하며, 특히 프래그먼트 셰이더에서의 셰이더 읽기를 기다려야 합니다. 왜냐하면 우리가 텍스처를 사용할 곳이 거기이기 때문입니다.

이러한 규칙은 다음 접근 마스크와 파이프라인 스테이지를 사용하여 지정됩니다:

```c++
VkPipelineStageFlags sourceStage;
VkPipelineStageFlags destinationStage;

if (oldLayout == VK_IMAGE_LAYOUT_UNDEFINED && newLayout == VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL) {
    barrier.srcAccessMask = 0;
    barrier.dstAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT;

    sourceStage = VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT;
    destinationStage = VK_PIPELINE_STAGE_TRANSFER_BIT;
} else if (oldLayout == VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL && newLayout == VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL) {
    barrier.srcAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT;
    barrier.dstAccessMask = VK_ACCESS_SHADER_READ_BIT;

    sourceStage = VK_PIPELINE_STAGE_TRANSFER_BIT;
    destinationStage = VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT;
} else {
    throw std::invalid_argument("unsupported layout transition!");
}

vkCmdPipelineBarrier(
    commandBuffer,
    sourceStage, destinationStage,
    0,
    0, nullptr,
    0, nullptr,
    1, &barrier
);
```

앞서 언급한 표에서 볼 수 있듯이, 전송 쓰기는 파이프라인 전송 스테이지에서 발생해야 합니다. 쓰기는 어떤 것도 기다릴 필요가 없으므로, 빈 접근 마스크와 배리어 이전 작업을 위한 가장 빠른 파이프라인 스테이지인 `VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT`를 지정할 수 있습니다. `VK_PIPELINE_STAGE_TRANSFER_BIT`는 그래픽 및 컴퓨트 파이프라인 내의 *실제* 스테이지가 아니라는 점에 유의해야 합니다. 이것은 전송이 일어나는 의사-스테이지(pseudo-stage)에 가깝습니다. 더 많은 정보와 다른 의사-스테이지의 예는 [문서](https://www.khronos.org/registry/vulkan/specs/1.3-extensions/html/chap7.html#VkPipelineStageFlagBits)를 참조하십시오.

이미지는 동일한 파이프라인 스테이지에서 쓰여지고, 이후 프래그먼트 셰이더에 의해 읽혀질 것이므로, 프래그먼트 셰이더 파이프라인 스테이지에서 셰이더 읽기 접근을 지정합니다.

만약 앞으로 더 많은 전환이 필요하다면, 이 함수를 확장할 것입니다. 애플리케이션은 이제 성공적으로 실행되어야 하지만, 물론 아직 시각적인 변화는 없습니다.

한 가지 주목할 점은 커맨드 버퍼 제출이 시작 시 암시적인 `VK_ACCESS_HOST_WRITE_BIT` 동기화를 초래한다는 것입니다. `transitionImageLayout` 함수는 단일 명령만 있는 커맨드 버퍼를 실행하므로, 레이아웃 전환에서 `VK_ACCESS_HOST_WRITE_BIT` 의존성이 필요할 경우 이 암시적 동기화를 사용할 수 있습니다. 이를 명시적으로 할지 여부는 여러분에게 달려있지만, 저는 개인적으로 이러한 OpenGL과 같은 "숨겨진" 작업에 의존하는 것을 좋아하지 않습니다.

실제로 모든 작업을 지원하는 특별한 유형의 이미지 레이아웃인 `VK_IMAGE_LAYOUT_GENERAL`이 있습니다. 물론 이것의 문제점은 어떤 작업에 대해서도 반드시 최상의 성능을 제공하지는 않는다는 것입니다. 이미지를 입력과 출력으로 동시에 사용하거나, 사전 초기화된 레이아웃을 벗어난 이미지를 읽는 것과 같은 일부 특수한 경우에 필요합니다.

지금까지 명령을 제출하는 모든 헬퍼 함수는 큐가 유휴 상태가 될 때까지 기다림으로써 동기적으로 실행되도록 설정되었습니다. 실제 애플리케이션에서는 이러한 작업들을 단일 커맨드 버퍼에 결합하고 더 높은 처리량을 위해 비동기적으로 실행하는 것이 권장됩니다. 특히 `createTextureImage` 함수의 전환 및 복사 작업이 그렇습니다. 헬퍼 함수들이 명령을 기록하는 `setupCommandBuffer`를 만들고, 지금까지 기록된 명령을 실행하는 `flushSetupCommands`를 추가하여 이를 실험해 보세요. 텍스처 매핑이 작동한 후에 텍스처 리소스가 여전히 올바르게 설정되었는지 확인하는 것이 가장 좋습니다.

## 정리

`createTextureImage` 함수를 마무리하기 위해, 끝에서 스테이징 버퍼와 그 메모리를 정리합니다:

```c++
    transitionImageLayout(textureImage, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL, VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL);

    vkDestroyBuffer(device, stagingBuffer, nullptr);
    vkFreeMemory(device, stagingBufferMemory, nullptr);
}
```

주 텍스처 이미지는 프로그램이 끝날 때까지 사용됩니다:

```c++
void cleanup() {
    cleanupSwapChain();

    vkDestroyImage(device, textureImage, nullptr);
    vkFreeMemory(device, textureImageMemory, nullptr);

    ...
}
```

이제 이미지는 텍스처를 포함하고 있지만, 그래픽 파이프라인에서 접근할 방법이 아직 필요합니다. 다음 장에서 그 작업을 할 것입니다.

[C++ 코드](/code/24_texture_image.cpp) /
[정점 셰이더](/code/22_shader_ubo.vert) /
[프래그먼트 셰이더](/code/22_shader_ubo.frag)