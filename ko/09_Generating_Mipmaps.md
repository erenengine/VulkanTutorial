## 서론
이제 우리 프로그램은 3D 모델을 로드하고 렌더링할 수 있습니다. 이번 장에서는 밉맵 생성이라는 기능을 하나 더 추가할 것입니다. 밉맵은 게임과 렌더링 소프트웨어에서 널리 사용되며, Vulkan은 밉맵 생성 방법을 완벽하게 제어할 수 있도록 해줍니다.

밉맵은 미리 계산된, 축소된 버전의 이미지입니다. 각각의 새 이미지는 이전 이미지의 너비와 높이가 절반입니다. 밉맵은 *디테일 수준(Level of Detail, LOD)*의 한 형태로 사용됩니다. 카메라에서 멀리 떨어진 객체는 더 작은 밉 이미지에서 텍스처를 샘플링합니다. 더 작은 이미지를 사용하면 렌더링 속도가 향상되고 [모아레 패턴](https://ko.wikipedia.org/wiki/%EB%AC%B4%EC%95%84%EB%A0%88_%EB%AC%B4%EB%8A%AC)과 같은 아티팩트를 방지할 수 있습니다. 밉맵이 어떻게 생겼는지 보여주는 예시는 다음과 같습니다:

![](/images/mipmaps_example.jpg)

## 이미지 생성

Vulkan에서 각 밉 이미지는 `VkImage`의 서로 다른 *밉 레벨(mip level)*에 저장됩니다. 밉 레벨 0은 원본 이미지이며, 레벨 0 이후의 밉 레벨들은 흔히 *밉 체인(mip chain)*이라고 불립니다.

밉 레벨의 수는 `VkImage`를 생성할 때 지정됩니다. 지금까지 우리는 항상 이 값을 1로 설정했습니다. 이제 이미지의 크기로부터 밉 레벨의 수를 계산해야 합니다. 먼저, 이 수를 저장할 클래스 멤버를 추가합니다:

```c++
...
uint32_t mipLevels;
VkImage textureImage;
...
```

`mipLevels`의 값은 `createTextureImage`에서 텍스처를 로드한 후에 찾을 수 있습니다:

```c++
int texWidth, texHeight, texChannels;
stbi_uc* pixels = stbi_load(TEXTURE_PATH.c_str(), &texWidth, &texHeight, &texChannels, STBI_rgb_alpha);
...
mipLevels = static_cast<uint32_t>(std::floor(std::log2(std::max(texWidth, texHeight)))) + 1;
```

이 코드는 밉 체인의 레벨 수를 계산합니다. `max` 함수는 가장 큰 차원(너비 또는 높이)을 선택합니다. `log2` 함수는 해당 차원을 2로 몇 번 나눌 수 있는지 계산합니다. `floor` 함수는 가장 큰 차원이 2의 거듭제곱이 아닌 경우를 처리합니다. `1`을 더해서 원본 이미지 자체도 밉 레벨을 갖도록 합니다.

이 값을 사용하려면 `createImage`, `createImageView`, `transitionImageLayout` 함수를 수정하여 밉 레벨 수를 지정할 수 있도록 해야 합니다. 함수들에 `mipLevels` 매개변수를 추가하세요:

```c++
void createImage(uint32_t width, uint32_t height, uint32_t mipLevels, VkFormat format, VkImageTiling tiling, VkImageUsageFlags usage, VkMemoryPropertyFlags properties, VkImage& image, VkDeviceMemory& imageMemory) {
    ...
    imageInfo.mipLevels = mipLevels;
    ...
}
```
```c++
VkImageView createImageView(VkImage image, VkFormat format, VkImageAspectFlags aspectFlags, uint32_t mipLevels) {
    ...
    viewInfo.subresourceRange.levelCount = mipLevels;
    ...
}
```
```c++
void transitionImageLayout(VkImage image, VkFormat format, VkImageLayout oldLayout, VkImageLayout newLayout, uint32_t mipLevels) {
    ...
    barrier.subresourceRange.levelCount = mipLevels;
    ...
}
```

이 함수들에 대한 모든 호출을 올바른 값을 사용하도록 업데이트합니다:

```c++
createImage(swapChainExtent.width, swapChainExtent.height, 1, depthFormat, VK_IMAGE_TILING_OPTIMAL, VK_IMAGE_USAGE_DEPTH_STENCIL_ATTACHMENT_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, depthImage, depthImageMemory);
...
createImage(texWidth, texHeight, mipLevels, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_TILING_OPTIMAL, VK_IMAGE_USAGE_TRANSFER_DST_BIT | VK_IMAGE_USAGE_SAMPLED_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, textureImage, textureImageMemory);
```
```c++
swapChainImageViews[i] = createImageView(swapChainImages[i], swapChainImageFormat, VK_IMAGE_ASPECT_COLOR_BIT, 1);
...
depthImageView = createImageView(depthImage, depthFormat, VK_IMAGE_ASPECT_DEPTH_BIT, 1);
...
textureImageView = createImageView(textureImage, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_ASPECT_COLOR_BIT, mipLevels);
```
```c++
transitionImageLayout(depthImage, depthFormat, VK_IMAGE_LAYOUT_UNDEFINED, VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL, 1);
...
transitionImageLayout(textureImage, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_LAYOUT_UNDEFINED, VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL, mipLevels);
```

## 밉맵 생성하기

이제 우리의 텍스처 이미지는 여러 밉 레벨을 가지지만, 스테이징 버퍼는 밉 레벨 0을 채우는 데만 사용될 수 있습니다. 다른 레벨들은 여전히 정의되지 않은 상태입니다. 이 레벨들을 채우려면 우리가 가진 단일 레벨로부터 데이터를 생성해야 합니다. 이를 위해 `vkCmdBlitImage` 명령을 사용할 것입니다. 이 명령은 복사, 스케일링, 필터링 연산을 수행합니다. 우리는 이 명령을 여러 번 호출하여 우리 텍스처 이미지의 각 레벨로 데이터를 *블릿(blit)*할 것입니다.

`vkCmdBlitImage`는 전송 작업으로 간주되므로, 텍스처 이미지를 전송의 소스(source)와 대상(destination)으로 모두 사용할 것임을 Vulkan에 알려야 합니다. `createTextureImage`에서 텍스처 이미지의 사용 플래그에 `VK_IMAGE_USAGE_TRANSFER_SRC_BIT`를 추가합니다:

```c++
...
createImage(texWidth, texHeight, mipLevels, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_TILING_OPTIMAL, VK_IMAGE_USAGE_TRANSFER_SRC_BIT | VK_IMAGE_USAGE_TRANSFER_DST_BIT | VK_IMAGE_USAGE_SAMPLED_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, textureImage, textureImageMemory);
...
```

다른 이미지 작업과 마찬가지로, `vkCmdBlitImage`는 작동하는 이미지의 레이아웃에 의존합니다. 전체 이미지를 `VK_IMAGE_LAYOUT_GENERAL`로 전환할 수도 있지만, 이는 매우 느릴 가능성이 높습니다. 최적의 성능을 위해서는 소스 이미지는 `VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL`에, 대상 이미지는 `VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL`에 있어야 합니다. Vulkan은 이미지의 각 밉 레벨을 독립적으로 전환할 수 있도록 허용합니다. 각 블릿은 한 번에 두 개의 밉 레벨만 다루므로, 블릿 명령 사이에 각 레벨을 최적의 레이아웃으로 전환할 수 있습니다.

`transitionImageLayout`은 전체 이미지에 대해서만 레이아웃 전환을 수행하므로, 파이프라인 배리어 명령을 몇 개 더 작성해야 합니다. `createTextureImage`에서 `VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL`로의 기존 전환을 제거합니다:

```c++
...
transitionImageLayout(textureImage, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_LAYOUT_UNDEFINED, VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL, mipLevels);
    copyBufferToImage(stagingBuffer, textureImage, static_cast<uint32_t>(texWidth), static_cast<uint32_t>(texHeight));
// 밉맵을 생성하는 동안 VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL로 전환됨
...
```

이렇게 하면 텍스처 이미지의 각 레벨이 `VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL` 상태로 남게 됩니다. 각 레벨은 해당 레벨에서 읽는 블릿 명령이 완료된 후 `VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL`로 전환될 것입니다.

이제 밉맵을 생성하는 함수를 작성해 보겠습니다:

```c++
void generateMipmaps(VkImage image, int32_t texWidth, int32_t texHeight, uint32_t mipLevels) {
    VkCommandBuffer commandBuffer = beginSingleTimeCommands();

    VkImageMemoryBarrier barrier{};
    barrier.sType = VK_STRUCTURE_TYPE_IMAGE_MEMORY_BARRIER;
    barrier.image = image;
    barrier.srcQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED;
    barrier.dstQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED;
    barrier.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
    barrier.subresourceRange.baseArrayLayer = 0;
    barrier.subresourceRange.layerCount = 1;
    barrier.subresourceRange.levelCount = 1;

    endSingleTimeCommands(commandBuffer);
}
```

여러 번의 전환을 수행할 것이므로 이 `VkImageMemoryBarrier`를 재사용할 것입니다. 위에서 설정된 필드들은 모든 배리어에 대해 동일하게 유지됩니다. `subresourceRange.miplevel`, `oldLayout`, `newLayout`, `srcAccessMask`, `dstAccessMask`는 각 전환마다 변경될 것입니다.

```c++
int32_t mipWidth = texWidth;
int32_t mipHeight = texHeight;

for (uint32_t i = 1; i < mipLevels; i++) {

}
```

이 루프는 각 `VkCmdBlitImage` 명령을 기록합니다. 루프 변수가 0이 아닌 1에서 시작하는 점에 유의하세요.

```c++
barrier.subresourceRange.baseMipLevel = i - 1;
barrier.oldLayout = VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL;
barrier.newLayout = VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL;
barrier.srcAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT;
barrier.dstAccessMask = VK_ACCESS_TRANSFER_READ_BIT;

vkCmdPipelineBarrier(commandBuffer,
    VK_PIPELINE_STAGE_TRANSFER_BIT, VK_PIPELINE_STAGE_TRANSFER_BIT, 0,
    0, nullptr,
    0, nullptr,
    1, &barrier);
```

먼저, `i - 1` 레벨을 `VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL`로 전환합니다. 이 전환은 이전 블릿 명령이나 `vkCmdCopyBufferToImage`로부터 `i - 1` 레벨이 채워질 때까지 기다립니다. 현재 블릿 명령은 이 전환을 기다리게 됩니다.

```c++
VkImageBlit blit{};
blit.srcOffsets[0] = { 0, 0, 0 };
blit.srcOffsets[1] = { mipWidth, mipHeight, 1 };
blit.srcSubresource.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
blit.srcSubresource.mipLevel = i - 1;
blit.srcSubresource.baseArrayLayer = 0;
blit.srcSubresource.layerCount = 1;
blit.dstOffsets[0] = { 0, 0, 0 };
blit.dstOffsets[1] = { mipWidth > 1 ? mipWidth / 2 : 1, mipHeight > 1 ? mipHeight / 2 : 1, 1 };
blit.dstSubresource.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
blit.dstSubresource.mipLevel = i;
blit.dstSubresource.baseArrayLayer = 0;
blit.dstSubresource.layerCount = 1;
```

다음으로, 블릿 작업에 사용될 영역을 지정합니다. 소스 밉 레벨은 `i - 1`이고 대상 밉 레벨은 `i`입니다. `srcOffsets` 배열의 두 요소는 데이터가 블릿될 3D 영역을 결정합니다. `dstOffsets`는 데이터가 블릿될 영역을 결정합니다. 각 밉 레벨은 이전 레벨 크기의 절반이므로 `dstOffsets[1]`의 X와 Y 차원은 2로 나눕니다. 2D 이미지는 깊이가 1이므로 `srcOffsets[1]`과 `dstOffsets[1]`의 Z 차원은 1이어야 합니다.

```c++
vkCmdBlitImage(commandBuffer,
    image, VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL,
    image, VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL,
    1, &blit,
    VK_FILTER_LINEAR);
```

이제 블릿 명령을 기록합니다. `srcImage`와 `dstImage` 매개변수 모두에 `textureImage`가 사용되는 점에 유의하세요. 이는 동일한 이미지의 서로 다른 레벨 간에 블리팅을 수행하기 때문입니다. 소스 밉 레벨은 방금 `VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL`로 전환되었고, 대상 레벨은 `createTextureImage`에서부터 `VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL` 상태로 남아있습니다.

([정점 버퍼](!kr/Vertex_buffers/Staging_buffer)에서 제안된 것처럼) 전용 전송 큐를 사용하고 있다면 주의하세요: `vkCmdBlitImage`는 그래픽스 기능이 있는 큐에 제출되어야 합니다.

마지막 매개변수는 블릿에 사용할 `VkFilter`를 지정할 수 있게 해줍니다. 여기서는 `VkSampler`를 만들 때와 동일한 필터링 옵션을 가집니다. 우리는 보간을 활성화하기 위해 `VK_FILTER_LINEAR`를 사용합니다.

```c++
barrier.oldLayout = VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL;
barrier.newLayout = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL;
barrier.srcAccessMask = VK_ACCESS_TRANSFER_READ_BIT;
barrier.dstAccessMask = VK_ACCESS_SHADER_READ_BIT;

vkCmdPipelineBarrier(commandBuffer,
    VK_PIPELINE_STAGE_TRANSFER_BIT, VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT, 0,
    0, nullptr,
    0, nullptr,
    1, &barrier);
```

이 배리어는 밉 레벨 `i - 1`을 `VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL`로 전환합니다. 이 전환은 현재 블릿 명령이 완료되기를 기다립니다. 모든 샘플링 작업은 이 전환이 완료되기를 기다릴 것입니다.

```c++
    ...
    if (mipWidth > 1) mipWidth /= 2;
    if (mipHeight > 1) mipHeight /= 2;
}
```

루프의 끝에서 현재 밉 차원을 2로 나눕니다. 각 차원이 0이 되지 않도록 나누기 전에 확인합니다. 이는 이미지가 정사각형이 아닐 경우를 처리하는데, 한쪽 밉 차원이 다른 쪽보다 먼저 1에 도달하기 때문입니다. 이런 경우, 해당 차원은 나머지 모든 레벨에 대해 1로 유지되어야 합니다.

```c++
    barrier.subresourceRange.baseMipLevel = mipLevels - 1;
    barrier.oldLayout = VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL;
    barrier.newLayout = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL;
    barrier.srcAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT;
    barrier.dstAccessMask = VK_ACCESS_SHADER_READ_BIT;

    vkCmdPipelineBarrier(commandBuffer,
        VK_PIPELINE_STAGE_TRANSFER_BIT, VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT, 0,
        0, nullptr,
        0, nullptr,
        1, &barrier);

    endSingleTimeCommands(commandBuffer);
}
```

커맨드 버퍼를 종료하기 전에, 파이프라인 배리어를 하나 더 삽입합니다. 이 배리어는 마지막 밉 레벨을 `VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL`에서 `VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL`로 전환합니다. 마지막 밉 레벨은 블릿의 소스로 사용되지 않기 때문에 루프에서 처리되지 않았습니다.

마지막으로, `createTextureImage`에서 `generateMipmaps`를 호출합니다:

```c++
transitionImageLayout(textureImage, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_LAYOUT_UNDEFINED, VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL, mipLevels);
    copyBufferToImage(stagingBuffer, textureImage, static_cast<uint32_t>(texWidth), static_cast<uint32_t>(texHeight));
// 밉맵을 생성하는 동안 VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL로 전환됨
...
generateMipmaps(textureImage, texWidth, texHeight, mipLevels);
```

이제 우리 텍스처 이미지의 밉맵이 완전히 채워졌습니다.

## 선형 필터링 지원

`vkCmdBlitImage`와 같은 내장 함수를 사용하여 모든 밉 레벨을 생성하는 것은 매우 편리하지만, 불행히도 모든 플랫폼에서 지원된다고 보장되지는 않습니다. 이를 위해서는 우리가 사용하는 텍스처 이미지 형식이 선형 필터링을 지원해야 하며, 이는 `vkGetPhysicalDeviceFormatProperties` 함수로 확인할 수 있습니다. 이를 위해 `generateMipmaps` 함수에 확인 코드를 추가할 것입니다.

먼저 이미지 형식을 지정하는 추가 매개변수를 추가합니다:

```c++
void createTextureImage() {
    ...

    generateMipmaps(textureImage, VK_FORMAT_R8G8B8A8_SRGB, texWidth, texHeight, mipLevels);
}

void generateMipmaps(VkImage image, VkFormat imageFormat, int32_t texWidth, int32_t texHeight, uint32_t mipLevels) {

    ...
}
```

`generateMipmaps` 함수에서 `vkGetPhysicalDeviceFormatProperties`를 사용하여 텍스처 이미지 형식의 속성을 요청합니다:

```c++
void generateMipmaps(VkImage image, VkFormat imageFormat, int32_t texWidth, int32_t texHeight, uint32_t mipLevels) {

    // 이미지 형식이 선형 블리팅을 지원하는지 확인
    VkFormatProperties formatProperties;
    vkGetPhysicalDeviceFormatProperties(physicalDevice, imageFormat, &formatProperties);

    ...
```

`VkFormatProperties` 구조체에는 `linearTilingFeatures`, `optimalTilingFeatures`, `bufferFeatures`라는 세 개의 필드가 있으며, 각각 형식이 사용되는 방식에 따라 어떻게 사용될 수 있는지를 설명합니다. 우리는 최적(optimal) 타일링 형식으로 텍스처 이미지를 생성하므로, `optimalTilingFeatures`를 확인해야 합니다. 선형 필터링 기능 지원은 `VK_FORMAT_FEATURE_SAMPLED_IMAGE_FILTER_LINEAR_BIT`로 확인할 수 있습니다:

```c++
if (!(formatProperties.optimalTilingFeatures & VK_FORMAT_FEATURE_SAMPLED_IMAGE_FILTER_LINEAR_BIT)) {
    throw std::runtime_error("texture image format does not support linear blitting!");
}
```

이 경우 두 가지 대안이 있습니다. 선형 블리팅을 *지원하는* 일반적인 텍스처 이미지 형식을 검색하는 함수를 구현하거나, [stb_image_resize](https://github.com/nothings/stb/blob/master/stb_image_resize.h)와 같은 라이브러리를 사용하여 소프트웨어에서 밉맵 생성을 구현할 수 있습니다. 그런 다음 각 밉 레벨을 원본 이미지를 로드했던 것과 같은 방식으로 이미지에 로드할 수 있습니다.

실무에서는 런타임에 밉맵 레벨을 생성하는 것이 일반적이지 않다는 점을 알아두는 것이 좋습니다. 보통은 로딩 속도를 향상시키기 위해 미리 생성하여 기본 레벨과 함께 텍스처 파일에 저장합니다. 소프트웨어에서 리사이징을 구현하고 파일에서 여러 레벨을 로드하는 것은 독자의 연습 과제로 남겨두겠습니다.

## 샘플러

`VkImage`가 밉맵 데이터를 보유하는 동안, `VkSampler`는 렌더링 중에 해당 데이터를 읽는 방법을 제어합니다. Vulkan은 `minLod`, `maxLod`, `mipLodBias`, `mipmapMode`를 지정할 수 있게 해줍니다("Lod"는 "Level of Detail"을 의미합니다). 텍스처가 샘플링될 때, 샘플러는 다음 의사 코드에 따라 밉 레벨을 선택합니다:

```c++
lod = getLodLevelFromScreenSize(); // 객체가 가까우면 작아지고, 음수일 수 있음
lod = clamp(lod + mipLodBias, minLod, maxLod);

level = clamp(floor(lod), 0, texture.mipLevels - 1);  // 텍스처의 밉 레벨 수로 클램핑됨

if (mipmapMode == VK_SAMPLER_MIPMAP_MODE_NEAREST) {
    color = sample(level);
} else {
    color = blend(sample(level), sample(level + 1));
}
```

`samplerInfo.mipmapMode`가 `VK_SAMPLER_MIPMAP_MODE_NEAREST`이면 `lod`는 샘플링할 밉 레벨을 선택합니다. 밉맵 모드가 `VK_SAMPLER_MIPMAP_MODE_LINEAR`이면, `lod`는 샘플링할 두 개의 밉 레벨을 선택하는 데 사용됩니다. 해당 레벨들이 샘플링되고 결과는 선형으로 블렌딩됩니다.

샘플 작업은 `lod`의 영향도 받습니다:

```c++
if (lod <= 0) {
    color = readTexture(uv, magFilter);
} else {
    color = readTexture(uv, minFilter);
}
```

객체가 카메라에 가까우면 `magFilter`가 필터로 사용됩니다. 객체가 카메라에서 더 멀리 있으면 `minFilter`가 사용됩니다. 보통 `lod`는 음수가 아니며, 카메라에 가까울 때만 0입니다. `mipLodBias`를 사용하면 Vulkan이 평소보다 낮은 `lod`와 `level`을 사용하도록 강제할 수 있습니다.

이 장의 결과를 보려면 `textureSampler`에 대한 값을 선택해야 합니다. 우리는 이미 `minFilter`와 `magFilter`를 `VK_FILTER_LINEAR`를 사용하도록 설정했습니다. 이제 `minLod`, `maxLod`, `mipLodBias`, `mipmapMode`에 대한 값을 선택하기만 하면 됩니다.

```c++
void createTextureSampler() {
    ...
    samplerInfo.mipmapMode = VK_SAMPLER_MIPMAP_MODE_LINEAR;
    samplerInfo.minLod = 0.0f; // 선택 사항
    samplerInfo.maxLod = VK_LOD_CLAMP_NONE;
    samplerInfo.mipLodBias = 0.0f; // 선택 사항
    ...
}
```

전체 범위의 밉 레벨을 사용할 수 있도록 `minLod`를 0.0f로 설정하고, `maxLod`는 `VK_LOD_CLAMP_NONE`으로 설정합니다. 이 상수는 `1000.0f`와 같으며, 이는 텍스처에서 사용 가능한 모든 밉맵 레벨을 샘플링한다는 것을 의미합니다. `lod` 값을 변경할 이유가 없으므로 `mipLodBias`를 0.0f로 설정합니다.

이제 프로그램을 실행하면 다음과 같은 화면을 볼 수 있습니다:

![](/images/mipmaps.png)

장면이 매우 단순하기 때문에 극적인 차이는 없습니다. 자세히 보면 미묘한 차이가 있습니다.

![](/images/mipmaps_comparison.png)

가장 눈에 띄는 차이점은 종이에 적힌 글씨입니다. 밉맵을 사용하면 글씨가 부드럽게 처리되었습니다. 밉맵이 없으면 글씨는 모아레 아티팩트로 인해 거친 가장자리와 끊김이 있습니다.

샘플러 설정을 바꿔보면서 밉매핑에 어떤 영향을 미치는지 시험해 볼 수 있습니다. 예를 들어 `minLod`를 변경하여 샘플러가 가장 낮은 밉 레벨을 사용하지 않도록 강제할 수 있습니다:

```c++
samplerInfo.minLod = static_cast<float>(mipLevels / 2);
```

이 설정은 다음과 같은 이미지를 생성합니다:


![](/images/highmipmaps.png)

이것이 객체가 카메라에서 더 멀리 있을 때 더 높은 밉 레벨이 사용되는 방식입니다.

[C++ 코드](/code/29_mipmapping.cpp) /
[정점 셰이더](/code/27_shader_depth.vert) /
[프래그먼트 셰이더](/code/27_shader_depth.frag)