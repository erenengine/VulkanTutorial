## 소개

지금까지 우리가 다루었던 지오메트리는 3D로 투영되었지만, 실제로는 완전히 평면이었습니다. 이번 장에서는 3D 메시를 준비하기 위해 위치(position)에 Z 좌표를 추가할 것입니다. 이 세 번째 좌표를 사용하여 현재 사각형 위에 또 다른 사각형을 배치함으로써, 지오메트리가 깊이 순으로 정렬되지 않았을 때 발생하는 문제를 직접 확인해 보겠습니다.

## 3D 지오메트리

먼저 `Vertex` 구조체를 변경하여 위치에 3D 벡터를 사용하고, 그에 맞춰 `VkVertexInputAttributeDescription`의 `format`을 업데이트합니다.

```c++
struct Vertex {
    glm::vec3 pos;
    glm::vec3 color;
    glm::vec2 texCoord;

    ...

    static std::array<VkVertexInputAttributeDescription, 3> getAttributeDescriptions() {
        std::array<VkVertexInputAttributeDescription, 3> attributeDescriptions{};

        attributeDescriptions[0].binding = 0;
        attributeDescriptions[0].location = 0;
        attributeDescriptions[0].format = VK_FORMAT_R32G32B32_SFLOAT;
        attributeDescriptions[0].offset = offsetof(Vertex, pos);

        ...
    }
};
```

다음으로, 정점 셰이더가 3D 좌표를 입력으로 받아 변환하도록 수정합니다. 수정 후에는 반드시 셰이더를 다시 컴파일해야 합니다!

```glsl
layout(location = 0) in vec3 inPosition;

...

void main() {
    gl_Position = ubo.proj * ubo.view * ubo.model * vec4(inPosition, 1.0);
    fragColor = inColor;
    fragTexCoord = inTexCoord;
}
```

마지막으로, `vertices` 컨테이너를 Z 좌표를 포함하도록 업데이트합니다.

```c++
const std::vector<Vertex> vertices = {
    {{-0.5f, -0.5f, 0.0f}, {1.0f, 0.0f, 0.0f}, {0.0f, 0.0f}},
    {{0.5f, -0.5f, 0.0f}, {0.0f, 1.0f, 0.0f}, {1.0f, 0.0f}},
    {{0.5f, 0.5f, 0.0f}, {0.0f, 0.0f, 1.0f}, {1.0f, 1.0f}},
    {{-0.5f, 0.5f, 0.0f}, {1.0f, 1.0f, 1.0f}, {0.0f, 1.0f}}
};
```

지금 애플리케이션을 실행하면 이전과 완전히 동일한 결과를 볼 수 있습니다. 이제 장면을 더 흥미롭게 만들고 이번 장에서 다룰 문제를 보여주기 위해 지오메트리를 추가할 시간입니다. 현재 사각형 바로 아래에 위치할 사각형을 정의하기 위해 정점들을 복제합니다.

![](/images/extra_square.svg)

새 사각형의 Z 좌표는 `-0.5f`로 설정하고, 추가된 사각형에 대한 인덱스도 추가합니다.

```c++
const std::vector<Vertex> vertices = {
    {{-0.5f, -0.5f, 0.0f}, {1.0f, 0.0f, 0.0f}, {0.0f, 0.0f}},
    {{0.5f, -0.5f, 0.0f}, {0.0f, 1.0f, 0.0f}, {1.0f, 0.0f}},
    {{0.5f, 0.5f, 0.0f}, {0.0f, 0.0f, 1.0f}, {1.0f, 1.0f}},
    {{-0.5f, 0.5f, 0.0f}, {1.0f, 1.0f, 1.0f}, {0.0f, 1.0f}},

    {{-0.5f, -0.5f, -0.5f}, {1.0f, 0.0f, 0.0f}, {0.0f, 0.0f}},
    {{0.5f, -0.5f, -0.5f}, {0.0f, 1.0f, 0.0f}, {1.0f, 0.0f}},
    {{0.5f, 0.5f, -0.5f}, {0.0f, 0.0f, 1.0f}, {1.0f, 1.0f}},
    {{-0.5f, 0.5f, -0.5f}, {1.0f, 1.0f, 1.0f}, {0.0f, 1.0f}}
};

const std::vector<uint16_t> indices = {
    0, 1, 2, 2, 3, 0,
    4, 5, 6, 6, 7, 4
};
```

이제 프로그램을 실행하면 마치 에셔(Escher)의 그림과 같은 이상한 결과물을 보게 될 것입니다.

![](/images/depth_issues.png)

이 문제의 원인은 아래쪽 사각형의 프래그먼트가 단순히 인덱스 배열의 뒤에 온다는 이유만으로 위쪽 사각형의 프래그먼트 위에 그려지기 때문입니다. 이 문제를 해결하는 방법은 두 가지가 있습니다.

*   모든 그리기 호출(draw call)을 뒤쪽에서 앞쪽 순서로 깊이에 따라 정렬하기
*   깊이 버퍼(depth buffer)를 이용한 깊이 테스팅(depth testing) 사용하기

첫 번째 접근 방식은 보통 투명한 객체를 그릴 때 사용됩니다. 순서에 상관없는 투명도 처리는 해결하기 어려운 문제이기 때문입니다. 하지만 프래그먼트를 깊이 순으로 정렬하는 문제는 보통 **깊이 버퍼**를 사용하여 해결합니다. 깊이 버퍼는 색상 첨부(color attachment)가 모든 위치의 색상을 저장하는 것처럼, 모든 위치의 깊이(depth) 값을 저장하는 추가적인 첨부입니다. 래스터라이저가 프래그먼트를 생성할 때마다 깊이 테스트는 새 프래그먼트가 이전 프래그먼트보다 더 가까운지 확인합니다. 그렇지 않다면 새 프래그먼트는 폐기됩니다. 깊이 테스트를 통과한 프래그먼트는 자신의 깊이 값을 깊이 버퍼에 기록합니다. 프래그먼트 셰이더에서 색상 출력을 조작할 수 있듯이 이 깊이 값도 조작할 수 있습니다.

```c++
#define GLM_FORCE_RADIANS
#define GLM_FORCE_DEPTH_ZERO_TO_ONE
#include <glm/glm.hpp>
#include <glm/gtc/matrix_transform.hpp>
```

GLM이 생성하는 원근 투영 행렬은 기본적으로 OpenGL의 깊이 범위인 `-1.0`에서 `1.0`을 사용합니다. 우리는 `GLM_FORCE_DEPTH_ZERO_TO_ONE` 정의를 사용하여 Vulkan의 깊이 범위인 `0.0`에서 `1.0`을 사용하도록 설정해야 합니다.

## 깊이 이미지와 이미지 뷰

깊이 첨부는 색상 첨부와 마찬가지로 이미지를 기반으로 합니다. 차이점은 스왑 체인이 우리를 위해 깊이 이미지를 자동으로 생성해주지 않는다는 것입니다. 우리는 단 하나의 깊이 이미지만 필요합니다. 한 번에 하나의 그리기 작업만 실행되기 때문입니다. 깊이 이미지는 다시 이미지, 메모리, 이미지 뷰라는 세 가지 리소스가 필요합니다.

```c++
VkImage depthImage;
VkDeviceMemory depthImageMemory;
VkImageView depthImageView;
```

이러한 리소스들을 설정하기 위해 `createDepthResources`라는 새 함수를 만듭니다.

```c++
void initVulkan() {
    ...
    createCommandPool();
    createDepthResources();
    createTextureImage();
    ...
}

...

void createDepthResources() {

}
```

깊이 이미지를 만드는 것은 꽤 간단합니다. 스왑 체인 extent로 정의된 색상 첨부와 동일한 해상도를 가져야 하며, 깊이 첨부에 적합한 이미지 사용법, 최적 타일링(optimal tiling), 그리고 디바이스 로컬 메모리(device local memory)를 사용해야 합니다. 남은 유일한 질문은 "깊이 이미지에 적합한 포맷은 무엇인가?"입니다. 포맷은 깊이 구성 요소(depth component)를 포함해야 하며, 이는 `VK_FORMAT_` 이름에 `_D??_`로 표시됩니다.

텍스처 이미지와 달리, 우리는 프로그램에서 텍셀에 직접 접근하지 않을 것이므로 특정 포맷이 반드시 필요한 것은 아닙니다. 단지 합리적인 정밀도만 가지면 되며, 실제 애플리케이션에서는 최소 24비트가 일반적입니다. 이 요구 사항을 충족하는 몇 가지 포맷이 있습니다.

*   `VK_FORMAT_D32_SFLOAT`: 깊이를 위한 32비트 부동소수점
*   `VK_FORMAT_D32_SFLOAT_S8_UINT`: 깊이를 위한 32비트 부동소수점과 스텐실(stencil)을 위한 8비트 부호 없는 정수
*   `VK_FORMAT_D24_UNORM_S8_UINT`: 깊이를 위한 24비트 정규화 부동소수점과 스텐실을 위한 8비트 부호 없는 정수

스텐실 구성 요소는 [스텐실 테스트](https://en.wikipedia.org/wiki/Stencil_buffer)에 사용되며, 이는 깊이 테스팅과 결합할 수 있는 추가적인 테스트입니다. 이는 이후 튜토리얼에서 다룰 것입니다.

단순히 `VK_FORMAT_D32_SFLOAT` 포맷을 선택할 수도 있습니다. 이 포맷은 매우 보편적으로 지원되기 때문입니다. 하지만 가능하면 애플리케이션에 유연성을 더하는 것이 좋습니다. 우리는 가장 선호하는 포맷부터 순서대로 후보 목록을 받아 지원되는 첫 번째 포맷을 찾는 `findSupportedFormat` 함수를 작성할 것입니다.

```c++
VkFormat findSupportedFormat(const std::vector<VkFormat>& candidates, VkImageTiling tiling, VkFormatFeatureFlags features) {

}
```

포맷 지원 여부는 타일링 모드와 사용법에 따라 달라지므로, 이들을 매개변수로 포함해야 합니다. 포맷 지원 여부는 `vkGetPhysicalDeviceFormatProperties` 함수로 질의할 수 있습니다.

```c++
for (VkFormat format : candidates) {
    VkFormatProperties props;
    vkGetPhysicalDeviceFormatProperties(physicalDevice, format, &props);
}
```

`VkFormatProperties` 구조체는 세 개의 필드를 포함합니다.

*   `linearTilingFeatures`: 선형 타일링에서 지원되는 사용 사례
*   `optimalTilingFeatures`: 최적 타일링에서 지원되는 사용 사례
*   `bufferFeatures`: 버퍼에서 지원되는 사용 사례

여기서는 첫 두 필드만 관련이 있으며, 확인해야 할 필드는 함수의 `tiling` 매개변수에 따라 달라집니다.

```c++
if (tiling == VK_IMAGE_TILING_LINEAR && (props.linearTilingFeatures & features) == features) {
    return format;
} else if (tiling == VK_IMAGE_TILING_OPTIMAL && (props.optimalTilingFeatures & features) == features) {
    return format;
}
```

만약 후보 포맷 중 어느 것도 원하는 사용법을 지원하지 않는다면, 특별한 값을 반환하거나 예외를 던질 수 있습니다.

```c++
VkFormat findSupportedFormat(const std::vector<VkFormat>& candidates, VkImageTiling tiling, VkFormatFeatureFlags features) {
    for (VkFormat format : candidates) {
        VkFormatProperties props;
        vkGetPhysicalDeviceFormatProperties(physicalDevice, format, &props);

        if (tiling == VK_IMAGE_TILING_LINEAR && (props.linearTilingFeatures & features) == features) {
            return format;
        } else if (tiling == VK_IMAGE_TILING_OPTIMAL && (props.optimalTilingFeatures & features) == features) {
            return format;
        }
    }

    throw std::runtime_error("failed to find supported format!");
}
```

이제 이 함수를 사용하여 깊이 첨부로 사용 가능한 깊이 구성 요소를 가진 포맷을 선택하는 `findDepthFormat` 헬퍼 함수를 만들 것입니다.

```c++
VkFormat findDepthFormat() {
    return findSupportedFormat(
        {VK_FORMAT_D32_SFLOAT, VK_FORMAT_D32_SFLOAT_S8_UINT, VK_FORMAT_D24_UNORM_S8_UINT},
        VK_IMAGE_TILING_OPTIMAL,
        VK_FORMAT_FEATURE_DEPTH_STENCIL_ATTACHMENT_BIT
    );
}
```

이 경우에는 `VK_IMAGE_USAGE_` 플래그 대신 `VK_FORMAT_FEATURE_` 플래그를 사용해야 합니다. 이 후보 포맷들은 모두 깊이 구성 요소를 포함하며, 후자의 두 포맷은 스텐실 구성 요소도 포함합니다. 아직 스텐실을 사용하지는 않겠지만, 이 포맷을 가진 이미지의 레이아웃을 전환할 때는 이를 고려해야 합니다. 선택된 깊이 포맷이 스텐실 구성 요소를 포함하는지 알려주는 간단한 헬퍼 함수를 추가합니다.

```c++
bool hasStencilComponent(VkFormat format) {
    return format == VK_FORMAT_D32_SFLOAT_S8_UINT || format == VK_FORMAT_D24_UNORM_S8_UINT;
}
```

`createDepthResources` 함수에서 깊이 포맷을 찾기 위해 이 함수를 호출합니다.

```c++
VkFormat depthFormat = findDepthFormat();
```

이제 `createImage`와 `createImageView` 헬퍼 함수를 호출하는 데 필요한 모든 정보를 갖추었습니다.

```c++
createImage(swapChainExtent.width, swapChainExtent.height, depthFormat, VK_IMAGE_TILING_OPTIMAL, VK_IMAGE_USAGE_DEPTH_STENCIL_ATTACHMENT_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, depthImage, depthImageMemory);
depthImageView = createImageView(depthImage, depthFormat);
```

하지만, `createImageView` 함수는 현재 서브리소스가 항상 `VK_IMAGE_ASPECT_COLOR_BIT`라고 가정하고 있으므로, 해당 필드를 매개변수로 만들어야 합니다.

```c++
VkImageView createImageView(VkImage image, VkFormat format, VkImageAspectFlags aspectFlags) {
    ...
    viewInfo.subresourceRange.aspectMask = aspectFlags;
    ...
}
```

이 함수를 호출하는 모든 곳을 올바른 aspect를 사용하도록 업데이트합니다.

```c++
swapChainImageViews[i] = createImageView(swapChainImages[i], swapChainImageFormat, VK_IMAGE_ASPECT_COLOR_BIT);
...
depthImageView = createImageView(depthImage, depthFormat, VK_IMAGE_ASPECT_DEPTH_BIT);
...
textureImageView = createImageView(textureImage, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_ASPECT_COLOR_BIT);
```

이것으로 깊이 이미지 생성은 끝입니다. 우리는 렌더 패스 시작 시 색상 첨부처럼 깊이 첨부도 소거(clear)할 것이기 때문에, 메모리를 매핑하거나 다른 이미지를 복사할 필요가 없습니다.

### 깊이 이미지 명시적 전환

깊이 첨부로의 이미지 레이아웃 전환은 렌더 패스에서 처리할 것이므로 명시적으로 전환할 필요는 없습니다. 하지만 완전성을 위해 이 섹션에서 그 과정을 설명합니다. 원한다면 이 섹션을 건너뛰어도 좋습니다.

`createDepthResources` 함수 끝에서 `transitionImageLayout`을 호출합니다.

```c++
transitionImageLayout(depthImage, depthFormat, VK_IMAGE_LAYOUT_UNDEFINED, VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL);
```

기존 깊이 이미지 내용이 중요하지 않으므로, `undefined` 레이아웃을 초기 레이아웃으로 사용할 수 있습니다. `transitionImageLayout`의 로직 일부를 수정하여 올바른 서브리소스 aspect를 사용하도록 해야 합니다.

```c++
if (newLayout == VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL) {
    barrier.subresourceRange.aspectMask = VK_IMAGE_ASPECT_DEPTH_BIT;

    if (hasStencilComponent(format)) {
        barrier.subresourceRange.aspectMask |= VK_IMAGE_ASPECT_STENCIL_BIT;
    }
} else {
    barrier.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
}
```

비록 스텐실 구성 요소를 사용하지 않더라도, 깊이 이미지의 레이아웃 전환에는 이를 포함해야 합니다.

마지막으로, 올바른 접근 마스크(access mask)와 파이프라인 단계를 추가합니다.

```c++
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
} else if (oldLayout == VK_IMAGE_LAYOUT_UNDEFINED && newLayout == VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL) {
    barrier.srcAccessMask = 0;
    barrier.dstAccessMask = VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_READ_BIT | VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_WRITE_BIT;

    sourceStage = VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT;
    destinationStage = VK_PIPELINE_STAGE_EARLY_FRAGMENT_TESTS_BIT;
} else {
    throw std::invalid_argument("unsupported layout transition!");
}
```

깊이 버퍼는 프래그먼트가 보이는지 확인하기 위한 깊이 테스트를 위해 읽히고, 새 프래그먼트가 그려질 때 쓰여집니다. 읽기는 `VK_PIPELINE_STAGE_EARLY_FRAGMENT_TESTS_BIT` 단계에서, 쓰기는 `VK_PIPELINE_STAGE_LATE_FRAGMENT_TESTS_BIT` 단계에서 발생합니다. 지정된 작업과 일치하는 가장 이른 파이프라인 단계를 선택하여, 깊이 첨부로 사용될 필요가 있을 때 준비되도록 해야 합니다.

## 렌더 패스

이제 `createRenderPass`를 수정하여 깊이 첨부를 포함하도록 하겠습니다. 먼저 `VkAttachmentDescription`을 지정합니다.

```c++
VkAttachmentDescription depthAttachment{};
depthAttachment.format = findDepthFormat();
depthAttachment.samples = VK_SAMPLE_COUNT_1_BIT;
depthAttachment.loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR;
depthAttachment.storeOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
depthAttachment.stencilLoadOp = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
depthAttachment.stencilStoreOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
depthAttachment.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
depthAttachment.finalLayout = VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL;
```

`format`은 깊이 이미지 자체와 동일해야 합니다. 이번에는 깊이 데이터를 저장하는 데 신경 쓰지 않으므로(`storeOp`), `VK_ATTACHMENT_STORE_OP_DONT_CARE`를 사용합니다. 그리기가 끝난 후에는 사용되지 않을 것이기 때문입니다. 이는 하드웨어가 추가적인 최적화를 수행할 수 있게 해줍니다. 색상 버퍼와 마찬가지로, 이전 깊이 내용에는 신경 쓰지 않으므로 `initialLayout`으로 `VK_IMAGE_LAYOUT_UNDEFINED`를 사용할 수 있습니다.

```c++
VkAttachmentReference depthAttachmentRef{};
depthAttachmentRef.attachment = 1;
depthAttachmentRef.layout = VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL;
```

첫 번째(이자 유일한) 서브패스를 위해 이 첨부에 대한 참조를 추가합니다.

```c++
VkSubpassDescription subpass{};
subpass.pipelineBindPoint = VK_PIPELINE_BIND_POINT_GRAPHICS;
subpass.colorAttachmentCount = 1;
subpass.pColorAttachments = &colorAttachmentRef;
subpass.pDepthStencilAttachment = &depthAttachmentRef;
```

색상 첨부와 달리, 서브패스는 단 하나의 깊이(+스텐실) 첨부만 사용할 수 있습니다. 여러 버퍼에 대해 깊이 테스트를 수행하는 것은 의미가 없습니다.

```c++
std::array<VkAttachmentDescription, 2> attachments = {colorAttachment, depthAttachment};
VkRenderPassCreateInfo renderPassInfo{};
renderPassInfo.sType = VK_STRUCTURE_TYPE_RENDER_PASS_CREATE_INFO;
renderPassInfo.attachmentCount = static_cast<uint32_t>(attachments.size());
renderPassInfo.pAttachments = attachments.data();
renderPassInfo.subpassCount = 1;
renderPassInfo.pSubpasses = &subpass;
...
```

다음으로, 렌더 패스 생성 정보를 업데이트하여 두 첨부를 모두 포함하도록 합니다. `VkRenderPassCreateInfo`를 수정하여 첨부 배열을 가리키도록 합니다.

```c++
VkSubpassDependency dependency{};
dependency.srcSubpass = VK_SUBPASS_EXTERNAL;
dependency.dstSubpass = 0;
dependency.srcStageMask = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT | VK_PIPELINE_STAGE_EARLY_FRAGMENT_TESTS_BIT;
dependency.srcAccessMask = 0;
dependency.dstStageMask = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT | VK_PIPELINE_STAGE_EARLY_FRAGMENT_TESTS_BIT;
dependency.dstAccessMask = VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT | VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_WRITE_BIT;

renderPassInfo.dependencyCount = 1;
renderPassInfo.pDependencies = &dependency;
```

이제 서브패스 종속성을 수정하여 렌더 패스가 시작될 때 깊이 버퍼에 쓰기 작업을 수행할 수 있도록 동기화해야 합니다. 깊이 버퍼는 `loadOp`이 `CLEAR`이므로 `early fragment tests` 단계에서 쓰여집니다. 따라서 `dstStageMask`에 `VK_PIPELINE_STAGE_EARLY_FRAGMENT_TESTS_BIT`를 추가하고, `dstAccessMask`에 `VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_WRITE_BIT`를 추가하여 이 쓰기 작업이 발생할 수 있도록 해야 합니다. 이 종속성은 렌더 패스가 시작되기 전에 이미지 가용성(semaphore)을 기다린 후, 깊이 버퍼 소거 작업이 시작될 수 있도록 보장합니다.

## 프레임버퍼

다음 단계는 프레임버퍼 생성을 수정하여 깊이 이미지를 깊이 첨부에 바인딩하는 것입니다. `createFramebuffers`로 이동하여 깊이 이미지 뷰를 두 번째 첨부로 지정합니다.

```c++
std::array<VkImageView, 2> attachments = {
    swapChainImageViews[i],
    depthImageView
};

VkFramebufferCreateInfo framebufferInfo{};
framebufferInfo.sType = VK_STRUCTURE_TYPE_FRAMEBUFFER_CREATE_INFO;
framebufferInfo.renderPass = renderPass;
framebufferInfo.attachmentCount = static_cast<uint32_t>(attachments.size());
framebufferInfo.pAttachments = attachments.data();
framebufferInfo.width = swapChainExtent.width;
framebufferInfo.height = swapChainExtent.height;
framebufferInfo.layers = 1;
```

색상 첨부는 각 스왑 체인 이미지마다 다르지만, 동일한 깊이 이미지는 모든 프레임버퍼에서 사용될 수 있습니다. 우리의 세마포어 때문에 한 번에 하나의 서브패스만 실행되기 때문입니다.

또한, 깊이 이미지 뷰가 실제로 생성된 후에 프레임버퍼가 생성되도록 `createFramebuffers` 호출을 이동해야 합니다.

```c++
void initVulkan() {
    ...
    createDepthResources();
    createFramebuffers();
    ...
}
```

## 소거 값 (Clear values)

이제 `VK_ATTACHMENT_LOAD_OP_CLEAR`를 사용하는 여러 첨부가 있으므로, 여러 개의 소거 값(clear value)을 지정해야 합니다. `recordCommandBuffer`로 가서 `VkClearValue` 구조체의 배열을 만듭니다.

```c++
std::array<VkClearValue, 2> clearValues{};
clearValues[0].color = {{0.0f, 0.0f, 0.0f, 1.0f}};
clearValues[1].depthStencil = {1.0f, 0};

renderPassInfo.clearValueCount = static_cast<uint32_t>(clearValues.size());
renderPassInfo.pClearValues = clearValues.data();
```

Vulkan에서 깊이 버퍼의 깊이 범위는 `0.0`에서 `1.0`이며, `1.0`은 먼 쪽 뷰 평면(far view plane)에, `0.0`은 가까운 쪽 뷰 평면(near view plane)에 해당합니다. 깊이 버퍼의 각 지점의 초기 값은 가장 먼 깊이인 `1.0`이어야 합니다.

`clearValues`의 순서는 첨부 파일의 순서와 동일해야 함을 유의하세요.

## 깊이 및 스텐실 상태

깊이 첨부는 이제 사용할 준비가 되었지만, 그래픽 파이프라인에서 깊이 테스팅을 활성화해야 합니다. 이는 `VkPipelineDepthStencilStateCreateInfo` 구조체를 통해 구성됩니다.

```c++
VkPipelineDepthStencilStateCreateInfo depthStencil{};
depthStencil.sType = VK_STRUCTURE_TYPE_PIPELINE_DEPTH_STENCIL_STATE_CREATE_INFO;
depthStencil.depthTestEnable = VK_TRUE;
depthStencil.depthWriteEnable = VK_TRUE;
```

`depthTestEnable` 필드는 새 프래그먼트의 깊이를 깊이 버퍼와 비교하여 폐기해야 하는지를 결정합니다. `depthWriteEnable` 필드는 깊이 테스트를 통과한 프래그먼트의 새 깊이가 실제로 깊이 버퍼에 쓰여야 하는지를 지정합니다.

```c++
depthStencil.depthCompareOp = VK_COMPARE_OP_LESS;
```

`depthCompareOp` 필드는 프래그먼트를 유지하거나 폐기하기 위해 수행되는 비교 연산을 지정합니다. 우리는 더 낮은 깊이 = 더 가까움을 의미하는 관례를 따르므로, 새 프래그먼트의 깊이는 이전 값보다 *작아야* 합니다(`LESS`).

```c++
depthStencil.depthBoundsTestEnable = VK_FALSE;
depthStencil.minDepthBounds = 0.0f; // Optional
depthStencil.maxDepthBounds = 1.0f; // Optional
```

`depthBoundsTestEnable`, `minDepthBounds`, `maxDepthBounds` 필드는 선택적인 깊이 경계 테스트에 사용됩니다. 기본적으로 지정된 깊이 범위 내에 있는 프래그먼트만 유지할 수 있게 해줍니다. 우리는 이 기능을 사용하지 않을 것입니다.

```c++
depthStencil.stencilTestEnable = VK_FALSE;
depthStencil.front = {}; // Optional
depthStencil.back = {}; // Optional
```

마지막 세 필드는 스텐실 버퍼 작업을 구성하며, 이 튜토리얼에서는 사용하지 않습니다. 이 작업을 사용하려면 깊이/스텐실 이미지의 포맷이 스텐실 구성 요소를 포함하는지 확인해야 합니다.

```c++
pipelineInfo.pDepthStencilState = &depthStencil;
```

`VkGraphicsPipelineCreateInfo` 구조체를 업데이트하여 방금 채운 깊이 스텐실 상태를 참조하도록 합니다. 렌더 패스가 깊이 스텐실 첨부를 포함하는 경우, 깊이 스텐실 상태는 항상 지정되어야 합니다.

이제 프로그램을 실행하면, 지오메트리의 프래그먼트가 올바르게 정렬된 것을 볼 수 있습니다.

![](/images/depth_correct.png)

## 창 크기 조절 처리

창 크기가 조절될 때, 깊이 버퍼의 해상도도 새 색상 첨부 해상도와 일치하도록 변경되어야 합니다. `recreateSwapChain` 함수를 확장하여 이 경우에 깊이 리소스를 재생성하도록 합니다.

```c++
void recreateSwapChain() {
    int width = 0, height = 0;
    glfwGetFramebufferSize(window, &width, &height);
    while (width == 0 || height == 0) {
        glfwGetFramebufferSize(window, &width, &height);
        glfwWaitEvents();
    }

    vkDeviceWaitIdle(device);

    cleanupSwapChain();

    createSwapChain();
    createImageViews();
    createDepthResources();
    createFramebuffers();
}
```

정리 작업은 스왑 체인 정리 함수에서 이루어져야 합니다.

```c++
void cleanupSwapChain() {
    vkDestroyImageView(device, depthImageView, nullptr);
    vkDestroyImage(device, depthImage, nullptr);
    vkFreeMemory(device, depthImageMemory, nullptr);

    ...
}
```

축하합니다, 이제 여러분의 애플리케이션은 임의의 3D 지오메트리를 렌더링하고 올바르게 보이게 할 준비가 되었습니다. 다음 장에서는 텍스처가 입혀진 모델을 그려보며 이를 시험해 보겠습니다!

[C++ 코드](/code/27_depth_buffering.cpp) /
[정점 셰이더](/code/27_shader_depth.vert) /
[프래그먼트 셰이더](/code/27_shader_depth.frag)