## 설정

파이프라인 생성을 완료하기 전에, Vulkan에게 렌더링 중에 사용될 프레임버퍼 어태치먼트(attachment)에 대해 알려주어야 합니다. 우리는 몇 개의 색상 및 깊이 버퍼가 있을지, 각각에 몇 개의 샘플을 사용할지, 그리고 렌더링 작업 전반에 걸쳐 해당 콘텐츠를 어떻게 처리해야 하는지 지정해야 합니다. 이 모든 정보는 *렌더 패스(render pass)* 객체에 담기게 되며, 이를 위해 새로운 `create_render_pass` 함수를 만들 것입니다. 이 함수를 주 애플리케이션 초기화 로직에서 `create_graphics_pipeline` 앞에 호출하세요.

```rust
// 애플리케이션 초기화 함수 내에서...
self.create_swapchain();
self.create_image_views();
self.create_render_pass();
self.create_graphics_pipeline();

...

// 애플리케이션 구현(impl) 블록 내에 함수 추가
fn create_render_pass(&mut self) -> Result<(), Box<dyn std::error::Error>> {
    // ... 구현 ...
    Ok(())
}
```

## 어태치먼트 명세 (Attachment description)

우리의 경우, 스왑체인의 이미지 중 하나로 표현되는 단일 색상 버퍼 어태치먼트만 갖게 될 것입니다. Ash에서는 빌더(builder) 패턴을 사용하여 구조체를 명확하고 안전하게 생성하는 것이 일반적입니다.

```rust
// create_render_pass 함수 내에서
let color_attachment = vk::AttachmentDescription::builder()
    .format(self.swapchain_format)
    .samples(vk::SampleCountFlags::TYPE_1)
    .load_op(vk::AttachmentLoadOp::CLEAR)
    .store_op(vk::AttachmentStoreOp::STORE)
    .stencil_load_op(vk::AttachmentLoadOp::DONT_CARE)
    .stencil_store_op(vk::AttachmentStoreOp::DONT_CARE)
    .initial_layout(vk::ImageLayout::UNDEFINED)
    .final_layout(vk::ImageLayout::PRESENT_SRC_KHR)
    .build();
```

색상 어태치먼트의 `format`은 스왑체인 이미지의 포맷과 일치해야 하며, 아직 멀티샘플링은 다루지 않으므로 `vk::SampleCountFlags::TYPE_1` 샘플을 사용하겠습니다.

`load_op`와 `store_op`는 렌더링 전후에 어태치먼트의 데이터를 어떻게 처리할지를 결정합니다. `load_op`에 사용할 수 있는 선택지는 다음과 같습니다:

*   `vk::AttachmentLoadOp::LOAD`: 어태치먼트의 기존 내용을 보존합니다.
*   `vk::AttachmentLoadOp::CLEAR`: 시작 시 값을 특정 상수로 지웁니다.
*   `vk::AttachmentLoadOp::DONT_CARE`: 기존 내용이 정의되지 않음(undefined); 신경 쓰지 않습니다.

우리의 경우, 새 프레임을 그리기 전에 프레임버퍼를 검은색으로 지우기 위해 clear 작업을 사용할 것입니다. `store_op`에는 두 가지 가능성만 있습니다:

*   `vk::AttachmentStoreOp::STORE`: 렌더링된 내용은 메모리에 저장되어 나중에 읽을 수 있습니다.
*   `vk::AttachmentStoreOp::DONT_CARE`: 렌더링 작업 후 프레임버퍼의 내용은 정의되지 않습니다.

우리는 렌더링된 삼각형을 화면에서 보고 싶으므로, store 작업을 사용할 것입니다.

`load_op`와 `store_op`는 색상 및 깊이 데이터에 적용되며, `stencil_load_op` / `stencil_store_op`는 스텐실 데이터에 적용됩니다. 우리 애플리케이션은 스텐실 버퍼를 사용하지 않으므로, 로딩과 저장 결과는 중요하지 않습니다.

Vulkan에서 텍스처와 프레임버퍼는 특정 픽셀 포맷을 가진 `VkImage` 객체로 표현됩니다. 하지만 이미지로 무엇을 하려는지에 따라 메모리 내 픽셀의 레이아웃이 변경될 수 있습니다.

가장 일반적인 레이아웃 중 일부는 다음과 같습니다:

*   `vk::ImageLayout::COLOR_ATTACHMENT_OPTIMAL`: 색상 어태치먼트로 사용되는 이미지
*   `vk::ImageLayout::PRESENT_SRC_KHR`: 스왑체인에 표시(present)될 이미지
*   `vk::ImageLayout::TRANSFER_DST_OPTIMAL`: 메모리 복사 작업의 대상으로 사용될 이미지

`initial_layout`은 렌더 패스가 시작되기 전에 이미지가 어떤 레이아웃을 가질지 지정합니다. `final_layout`은 렌더 패스가 끝날 때 자동으로 전환될 레이아웃을 지정합니다. `initial_layout`에 `vk::ImageLayout::UNDEFINED`를 사용하면 이미지의 이전 레이아웃이 무엇이었는지 신경 쓰지 않겠다는 의미입니다. 우리는 렌더링 후에 이미지가 스왑체인을 통해 화면에 표시될 준비가 되기를 원하므로, `final_layout`으로 `vk::ImageLayout::PRESENT_SRC_KHR`을 사용합니다.

## 서브패스와 어태치먼트 참조

하나의 렌더 패스는 여러 개의 서브패스(subpass)로 구성될 수 있습니다. 서브패스는 이전 패스의 프레임버퍼 내용에 의존하는 후속 렌더링 작업입니다. 우리의 첫 번째 삼각형에서는 단일 서브패스만 사용할 것입니다.

모든 서브패스는 어태치먼트 명세 중 하나 이상을 참조합니다. 이러한 참조는 `vk::AttachmentReference` 구조체를 통해 이루어집니다.

```rust
let color_attachment_ref = vk::AttachmentReference::builder()
    .attachment(0)
    .layout(vk::ImageLayout::COLOR_ATTACHMENT_OPTIMAL)
    .build();
```

`attachment` 파라미터는 어태치먼트 명세 배열의 인덱스(`0`)를 통해 참조할 어태치먼트를 지정합니다. `layout`은 이 참조를 사용하는 서브패스 동안 어태치먼트가 가지길 원하는 레이아웃을 지정합니다. 우리는 이 어태치먼트를 색상 버퍼로 사용할 것이며, `vk::ImageLayout::COLOR_ATTACHMENT_OPTIMAL` 레이아웃이 최상의 성능을 제공할 것입니다.

서브패스는 `vk::SubpassDescription` 구조체를 사용하여 설명됩니다. Rust에서는 C++의 포인터와 개수 대신 슬라이스(`&[...]`)를 사용합니다.

```rust
let subpass = vk::SubpassDescription::builder()
    .pipeline_bind_point(vk::PipelineBindPoint::GRAPHICS)
    .color_attachments(&[color_attachment_ref]) // 참조를 슬라이스로 전달
    .build();
```

Vulkan은 미래에 컴퓨트 서브패스도 지원할 수 있으므로, `pipeline_bind_point`를 통해 이것이 그래픽스 서브패스임을 명시적으로 지정해야 합니다. `color_attachments` 필드는 색상 어태치먼트 참조의 슬라이스를 받습니다.

이 배열에서 어태치먼트의 인덱스는 프래그먼트 셰이더에서 `layout(location = 0) out vec4 outColor` 지시문을 통해 직접 참조됩니다!

## 렌더 패스

이제 어태치먼트와 이를 참조하는 서브패스가 설명되었으므로, 렌더 패스 자체를 생성할 수 있습니다. 애플리케이션 구조체에 `render_pass` 필드를 추가하세요.

```rust
struct HelloTriangleApplication {
    // ...
    render_pass: vk::RenderPass,
    pipeline_layout: vk::PipelineLayout,
    // ...
}
```

이제 `vk::RenderPassCreateInfo` 구조체를 채워 렌더 패스를 생성합니다. 이 구조체는 어태치먼트와 서브패스의 슬라이스를 받습니다.

```rust
let attachments = [color_attachment];
let subpasses = [subpass];

let render_pass_info = vk::RenderPassCreateInfo::builder()
    .attachments(&attachments)
    .subpasses(&subpasses)
    .build();

self.render_pass = unsafe {
    self.device
        .create_render_pass(&render_pass_info, None)
}?;
```

Ash 라이브러리에서 Vulkan 객체를 생성하거나 파괴하는 대부분의 함수는 `unsafe`로 표시되어 있습니다. 이는 개발자가 유효한 파라미터와 올바른 소멸 순서를 보장해야 함을 상기시켜 줍니다. Rust의 `?` 연산자를 사용하면 `Result`를 반환하는 함수에서 에러를 간결하게 처리할 수 있습니다.

파이프라인 레이아웃과 마찬가지로 렌더 패스는 프로그램 전반에서 참조되므로, 프로그램이 끝날 때 정리해야 합니다. 일반적으로 Rust에서는 `Drop` 트레이트 구현을 통해 이 작업을 수행합니다.

```rust
// Drop 트레이트 구현 내에서
unsafe {
    self.device.destroy_pipeline_layout(self.pipeline_layout, None);
    self.device.destroy_render_pass(self.render_pass, None);
    // ...
}
```

상당히 많은 작업이었지만, 다음 장에서는 이 모든 것이 합쳐져 마침내 그래픽스 파이프라인 객체를 생성하게 될 것입니다