### 프레임버퍼

지난 몇 장에 걸쳐 프레임버퍼에 대해 많이 이야기했고, 스왑 체인 이미지와 동일한 포맷을 가진 단일 프레임버퍼를 사용하도록 렌더 패스를 설정했지만, 아직 실제로 생성하지는 않았습니다.

렌더 패스를 생성할 때 지정한 첨부(attachment)들은 `VkFramebuffer` 객체로 감싸서 바인딩됩니다. 프레임버퍼 객체는 첨부를 나타내는 모든 `VkImageView` 객체를 참조합니다. 우리의 경우에는 단 하나, 바로 색상 첨부(color attachment)입니다. 하지만 첨부에 사용해야 할 이미지는 우리가 프레젠테이션을 위해 스왑 체인에서 이미지를 가져올 때 어떤 이미지를 반환하는지에 따라 달라집니다. 이는 스왑 체인의 모든 이미지에 대해 프레임버퍼를 생성하고, 드로잉 시점에는 가져온 이미지에 해당하는 것을 사용해야 한다는 의미입니다.

이를 위해, 프레임버퍼를 담을 또 다른 `std::vector` 클래스 멤버를 생성합니다:

```c++
std::vector<VkFramebuffer> swapChainFramebuffers;
```

이 배열을 위한 객체들은 `initVulkan`에서 그래픽 파이프라인을 생성한 직후에 호출되는 새로운 함수 `createFramebuffers`에서 생성할 것입니다:

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
}

...

void createFramebuffers() {

}
```

먼저 컨테이너의 크기를 조절하여 모든 프레임버퍼를 담을 수 있도록 합니다:

```c++
void createFramebuffers() {
    swapChainFramebuffers.resize(swapChainImageViews.size());
}
```

그런 다음 이미지 뷰를 순회하며 프레임버퍼를 생성합니다:

```c++
for (size_t i = 0; i < swapChainImageViews.size(); i++) {
    VkImageView attachments[] = {
        swapChainImageViews[i]
    };

    VkFramebufferCreateInfo framebufferInfo{};
    framebufferInfo.sType = VK_STRUCTURE_TYPE_FRAMEBUFFER_CREATE_INFO;
    framebufferInfo.renderPass = renderPass;
    framebufferInfo.attachmentCount = 1;
    framebufferInfo.pAttachments = attachments;
    framebufferInfo.width = swapChainExtent.width;
    framebufferInfo.height = swapChainExtent.height;
    framebufferInfo.layers = 1;

    if (vkCreateFramebuffer(device, &framebufferInfo, nullptr, &swapChainFramebuffers[i]) != VK_SUCCESS) {
        throw std::runtime_error("failed to create framebuffer!");
    }
}
```

보시다시피, 프레임버퍼 생성은 매우 간단합니다. 먼저 프레임버퍼가 어떤 `renderPass`와 호환되어야 하는지 지정해야 합니다. 프레임버퍼는 호환되는 렌더 패스와만 사용할 수 있는데, 이는 대략적으로 말해 동일한 수와 유형의 첨부를 사용한다는 것을 의미합니다.

`attachmentCount`와 `pAttachments` 매개변수는 렌더 패스의 `pAttachment` 배열에 있는 각 첨부 설명에 바인딩될 `VkImageView` 객체를 지정합니다.

`width`와 `height` 매개변수는 이름에서 알 수 있듯이 명확하며, `layers`는 이미지 배열의 레이어 수를 나타냅니다. 우리의 스왑 체인 이미지는 단일 이미지이므로 레이어 수는 `1`입니다.

프레임버퍼는 그것들이 기반으로 하는 이미지 뷰와 렌더 패스보다 먼저 삭제되어야 하지만, 렌더링을 모두 마친 후에만 삭제해야 합니다:

```c++
void cleanup() {
    for (auto framebuffer : swapChainFramebuffers) {
        vkDestroyFramebuffer(device, framebuffer, nullptr);
    }

    ...
}
```

이제 우리는 렌더링에 필요한 모든 객체를 갖추는 중요한 단계에 도달했습니다. 다음 장에서는 첫 실제 드로잉 명령을 작성할 것입니다.

[C++ 코드](/code/13_framebuffers.cpp) /
[정점 셰이더](/code/09_shader_base.vert) /
[프래그먼트 셰이더](/code/09_shader_base.frag)