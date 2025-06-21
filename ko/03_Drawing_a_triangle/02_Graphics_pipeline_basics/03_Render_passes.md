## 설정

파이프라인 생성을 완료하기 전에, Vulkan에게 렌더링 중에 사용될 프레임버퍼 어태치먼트(attachment)에 대해 알려주어야 합니다. 우리는 몇 개의 색상 및 깊이 버퍼가 있을지, 각각에 몇 개의 샘플을 사용할지, 그리고 렌더링 작업 전반에 걸쳐 해당 콘텐츠를 어떻게 처리해야 하는지 지정해야 합니다. 이 모든 정보는 *렌더 패스(render pass)* 객체에 담기게 되며, 이를 위해 새로운 `createRenderPass` 함수를 만들 것입니다. 이 함수를 `initVulkan`에서 `createGraphicsPipeline` 앞에 호출하세요.

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
}

...

void createRenderPass() {

}
```

## 어태치먼트 명세 (Attachment description)

우리의 경우, 스왑체인의 이미지 중 하나로 표현되는 단일 색상 버퍼 어태치먼트만 갖게 될 것입니다.

```c++
void createRenderPass() {
    VkAttachmentDescription colorAttachment{};
    colorAttachment.format = swapChainImageFormat;
    colorAttachment.samples = VK_SAMPLE_COUNT_1_BIT;
}
```

색상 어태치먼트의 `format`은 스왑체인 이미지의 포맷과 일치해야 하며, 아직 멀티샘플링은 다루지 않으므로 1개의 샘플을 사용하겠습니다.

```c++
colorAttachment.loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR;
colorAttachment.storeOp = VK_ATTACHMENT_STORE_OP_STORE;
```

`loadOp`와 `storeOp`는 렌더링 전후에 어태치먼트의 데이터를 어떻게 처리할지를 결정합니다. `loadOp`에 사용할 수 있는 선택지는 다음과 같습니다:

*   `VK_ATTACHMENT_LOAD_OP_LOAD`: 어태치먼트의 기존 내용을 보존합니다.
*   `VK_ATTACHMENT_LOAD_OP_CLEAR`: 시작 시 값을 특정 상수로 지웁니다.
*   `VK_ATTACHMENT_LOAD_OP_DONT_CARE`: 기존 내용이 정의되지 않음(undefined); 신경 쓰지 않습니다.

우리의 경우, 새 프레임을 그리기 전에 프레임버퍼를 검은색으로 지우기 위해 clear 작업을 사용할 것입니다. `storeOp`에는 두 가지 가능성만 있습니다:

*   `VK_ATTACHMENT_STORE_OP_STORE`: 렌더링된 내용은 메모리에 저장되어 나중에 읽을 수 있습니다.
*   `VK_ATTACHMENT_STORE_OP_DONT_CARE`: 렌더링 작업 후 프레임버퍼의 내용은 정의되지 않습니다.

우리는 렌더링된 삼각형을 화면에서 보고 싶으므로, store 작업을 사용할 것입니다.

```c++
colorAttachment.stencilLoadOp = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
colorAttachment.stencilStoreOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
```

`loadOp`와 `storeOp`는 색상 및 깊이 데이터에 적용되며, `stencilLoadOp` / `stencilStoreOp`는 스텐실 데이터에 적용됩니다. 우리 애플리케이션은 스텐실 버퍼를 사용하지 않으므로, 로딩과 저장 결과는 중요하지 않습니다.

```c++
colorAttachment.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
colorAttachment.finalLayout = VK_IMAGE_LAYOUT_PRESENT_SRC_KHR;
```

Vulkan에서 텍스처와 프레임버퍼는 특정 픽셀 포맷을 가진 `VkImage` 객체로 표현됩니다. 하지만 이미지로 무엇을 하려는지에 따라 메모리 내 픽셀의 레이아웃이 변경될 수 있습니다.

가장 일반적인 레이아웃 중 일부는 다음과 같습니다:

*   `VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL`: 색상 어태치먼트로 사용되는 이미지
*   `VK_IMAGE_LAYOUT_PRESENT_SRC_KHR`: 스왑체인에 표시(present)될 이미지
*   `VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL`: 메모리 복사 작업의 대상으로 사용될 이미지

이 주제는 텍스처링 장에서 더 깊이 다룰 것이지만, 지금 알아야 할 중요한 점은 이미지는 다음에 수행할 작업에 적합한 특정 레이아웃으로 전환되어야 한다는 점입니다.

`initialLayout`은 렌더 패스가 시작되기 전에 이미지가 어떤 레이아웃을 가질지 지정합니다. `finalLayout`은 렌더 패스가 끝날 때 자동으로 전환될 레이아웃을 지정합니다. `initialLayout`에 `VK_IMAGE_LAYOUT_UNDEFINED`를 사용하면 이미지의 이전 레이아웃이 무엇이었는지 신경 쓰지 않겠다는 의미입니다. 이 특별한 값의 주의할 점은 이미지의 내용이 보존된다는 보장이 없다는 것이지만, 어차피 우리가 내용을 지울 것이기 때문에 문제 되지 않습니다. 우리는 렌더링 후에 이미지가 스왑체인을 통해 화면에 표시될 준비가 되기를 원하므로, `finalLayout`으로 `VK_IMAGE_LAYOUT_PRESENT_SRC_KHR`을 사용합니다.

## 서브패스와 어태치먼트 참조

하나의 렌더 패스는 여러 개의 서브패스(subpass)로 구성될 수 있습니다. 서브패스는 이전 패스의 프레임버퍼 내용에 의존하는 후속 렌더링 작업입니다. 예를 들어, 연달아 적용되는 일련의 후처리 효과 같은 것들입니다. 이러한 렌더링 작업들을 하나의 렌더 패스로 그룹화하면, Vulkan은 연산을 재정렬하고 메모리 대역폭을 절약하여 성능을 향상시킬 수 있습니다. 하지만 우리의 첫 번째 삼각형에서는 단일 서브패스만 사용할 것입니다.

모든 서브패스는 이전 섹션에서 설명한 구조체를 사용하여 명세한 어태치먼트 중 하나 이상을 참조합니다. 이러한 참조 자체는 다음과 같은 `VkAttachmentReference` 구조체입니다.

```c++
VkAttachmentReference colorAttachmentRef{};
colorAttachmentRef.attachment = 0;
colorAttachmentRef.layout = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL;
```

`attachment` 파라미터는 어태치먼트 명세 배열의 인덱스를 통해 참조할 어태치먼트를 지정합니다. 우리 배열은 단일 `VkAttachmentDescription`으로 구성되어 있으므로 인덱스는 `0`입니다. `layout`은 이 참조를 사용하는 서브패스 동안 어태치먼트가 가지길 원하는 레이아웃을 지정합니다. Vulkan은 서브패스가 시작될 때 자동으로 어태치먼트를 이 레이아웃으로 전환합니다. 우리는 이 어태치먼트를 색상 버퍼로 사용할 것이며, 이름에서 알 수 있듯이 `VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL` 레이아웃이 최상의 성능을 제공할 것입니다.

서브패스는 `VkSubpassDescription` 구조체를 사용하여 설명됩니다:

```c++
VkSubpassDescription subpass{};
subpass.pipelineBindPoint = VK_PIPELINE_BIND_POINT_GRAPHICS;
```

Vulkan은 미래에 컴퓨트 서브패스도 지원할 수 있으므로, 이것이 그래픽스 서브패스임을 명시적으로 지정해야 합니다. 다음으로, 색상 어태치먼트에 대한 참조를 지정합니다.

```c++
subpass.colorAttachmentCount = 1;
subpass.pColorAttachments = &colorAttachmentRef;
```

이 배열에서 어태치먼트의 인덱스는 프래그먼트 셰이더에서 `layout(location = 0) out vec4 outColor` 지시문을 통해 직접 참조됩니다!

서브패스에서 참조할 수 있는 다른 유형의 어태치먼트는 다음과 같습니다:

*   `pInputAttachments`: 셰이더에서 읽어오는 어태치먼트
*   `pResolveAttachments`: 멀티샘플링된 색상 어태치먼트를 위해 사용되는 어태치먼트
*   `pDepthStencilAttachment`: 깊이 및 스텐실 데이터를 위한 어태치먼트
*   `pPreserveAttachments`: 이 서브패스에서는 사용되지 않지만 데이터가 보존되어야 하는 어태치먼트

## 렌더 패스

이제 어태치먼트와 이를 참조하는 기본 서브패스가 설명되었으므로, 렌더 패스 자체를 생성할 수 있습니다. `pipelineLayout` 변수 바로 위에 `VkRenderPass` 객체를 담을 새로운 클래스 멤버 변수를 만드세요.

```c++
VkRenderPass renderPass;
VkPipelineLayout pipelineLayout;
```

그런 다음 `VkRenderPassCreateInfo` 구조체를 어태치먼트와 서브패스 배열로 채워서 렌더 패스 객체를 생성할 수 있습니다. `VkAttachmentReference` 객체들은 이 배열의 인덱스를 사용하여 어태치먼트를 참조합니다.

```c++
VkRenderPassCreateInfo renderPassInfo{};
renderPassInfo.sType = VK_STRUCTURE_TYPE_RENDER_PASS_CREATE_INFO;
renderPassInfo.attachmentCount = 1;
renderPassInfo.pAttachments = &colorAttachment;
renderPassInfo.subpassCount = 1;
renderPassInfo.pSubpasses = &subpass;

if (vkCreateRenderPass(device, &renderPassInfo, nullptr, &renderPass) != VK_SUCCESS) {
    throw std::runtime_error("failed to create render pass!");
}
```

파이프라인 레이아웃과 마찬가지로 렌더 패스는 프로그램 전반에서 참조되므로, 프로그램이 끝날 때 정리해야 합니다.

```c++
void cleanup() {
    vkDestroyPipelineLayout(device, pipelineLayout, nullptr);
    vkDestroyRenderPass(device, renderPass, nullptr);
    ...
}
```

상당히 많은 작업이었지만, 다음 장에서는 이 모든 것이 합쳐져 마침내 그래픽스 파이프라인 객체를 생성하게 될 것입니다!

[C++ 코드](/code/11_render_passes.cpp) /
[버텍스 셰이더](/code/09_shader_base.vert) /
[프래그먼트 셰이더](/code/09_shader_base.frag)