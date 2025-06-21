### 프레임버퍼 (Rust / Ash)

지난 몇 장에 걸쳐 프레임버퍼에 대해 많이 이야기했고, 스왑 체인 이미지와 동일한 포맷을 가진 단일 프레임버퍼를 사용하도록 렌더 패스를 설정했지만, 아직 실제로 생성하지는 않았습니다.

렌더 패스를 생성할 때 지정한 첨부(attachment)들은 `vk::Framebuffer` 객체로 감싸서 바인딩됩니다. 프레임버퍼 객체는 첨부를 나타내는 모든 `vk::ImageView` 객체를 참조합니다. 우리의 경우에는 단 하나, 바로 색상 첨부(color attachment)입니다. 하지만 첨부에 사용해야 할 이미지는 우리가 프레젠테이션을 위해 스왑 체인에서 이미지를 가져올 때 어떤 이미지를 반환하는지에 따라 달라집니다. 이는 스왑 체인의 모든 이미지에 대해 프레임버퍼를 생성하고, 드로잉 시점에는 가져온 이미지에 해당하는 것을 사용해야 한다는 의미입니다.

이를 위해, 프레임버퍼를 담을 `Vec<vk::Framebuffer>` 구조체 필드를 추가합니다:

```rust
struct HelloTriangleApplication {
    // ...
    render_pass: vk::RenderPass,
    pipeline_layout: vk::PipelineLayout,
    graphics_pipeline: vk::Pipeline,
    swapchain_framebuffers: Vec<vk::Framebuffer>,
    // ...
}
```

이 벡터를 채우기 위한 객체들은 애플리케이션 생성자(`new`)에서 그래픽 파이프라인을 생성한 직후에 호출되는 새로운 함수 `create_framebuffers`에서 생성할 것입니다.

```rust
impl HelloTriangleApplication {
    pub fn new(window: &Window) -> Self {
        // ...
        let render_pass = Self::create_render_pass(&device, swapchain_image_format);
        let (graphics_pipeline, pipeline_layout) =
            Self::create_graphics_pipeline(&device, render_pass, swapchain_extent);
        let swapchain_framebuffers =
            Self::create_framebuffers(&device, render_pass, &swapchain_image_views, swapchain_extent);
        // ...
    }

    // ...

    fn create_framebuffers(
        device: &ash::Device,
        render_pass: vk::RenderPass,
        image_views: &[vk::ImageView],
        swapchain_extent: vk::Extent2D,
    ) -> Vec<vk::Framebuffer> {
        // ... 구현 ...
    }
}
```

`create_framebuffers` 함수는 이미지 뷰를 순회하며 각각에 대한 프레임버퍼를 생성합니다.

```rust
fn create_framebuffers(
    device: &ash::Device,
    render_pass: vk::RenderPass,
    image_views: &[vk::ImageView],
    swapchain_extent: vk::Extent2D,
) -> Vec<vk::Framebuffer> {
    let mut framebuffers = Vec::with_capacity(image_views.len());

    for &image_view in image_views.iter() {
        let attachments = [image_view];

        let framebuffer_info = vk::FramebufferCreateInfo::builder()
            .render_pass(render_pass)
            .attachments(&attachments)
            .width(swapchain_extent.width)
            .height(swapchain_extent.height)
            .layers(1);

        let framebuffer = unsafe {
            device
                .create_framebuffer(&framebuffer_info, None)
                .expect("Failed to create Framebuffer!")
        };
        framebuffers.push(framebuffer);
    }

    framebuffers
}
```

보시다시피, Ash의 빌더 패턴 덕분에 프레임버퍼 생성이 매우 간단합니다.
먼저 프레임버퍼가 어떤 `render_pass`와 호환되어야 하는지 지정해야 합니다. 프레임버퍼는 호환되는 렌더 패스와만 사용할 수 있는데, 이는 대략적으로 말해 동일한 수와 유형의 첨부를 사용한다는 것을 의미합니다.

- `render_pass()`: 프레임버퍼가 호환되어야 할 렌더 패스를 지정합니다.
- `attachments()`: 렌더 패스의 `pAttachments` 배열에 있는 각 첨부 설명에 바인딩될 `vk::ImageView` 객체의 슬라이스를 지정합니다.
- `width()`와 `height()`: 이름에서 알 수 있듯이 명확하며, 스왑 체인의 크기(`extent`)에서 가져옵니다.
- `layers()`: 이미지 배열의 레이어 수를 나타냅니다. 우리의 스왑 체인 이미지는 단일 이미지이므로 레이어 수는 `1`입니다.

`device.create_framebuffer` 호출은 `unsafe` 블록 안에 있습니다. 이는 유효하지 않은 핸들이나 매개변수를 전달할 경우 Vulkan 드라이버가 정의되지 않은 동작을 일으킬 수 있기 때문이며, Rust 컴파일러는 이를 보장할 수 없습니다.

생성된 프레임버퍼는 애플리케이션이 종료될 때 정리해야 합니다. Rust에서는 `Drop` 트레잇을 구현하여 이 작업을 자동으로 처리하는 것이 일반적입니다. 프레임버퍼는 그것들이 참조하는 이미지 뷰나 렌더 패스보다 먼저 파괴되어야 합니다. `Drop` 구현에서 파괴 순서를 명시적으로 제어하는 것이 좋습니다.

```rust
impl Drop for HelloTriangleApplication {
    fn drop(&mut self) {
        unsafe {
            // ... 다른 리소스 정리 ...

            for framebuffer in self.swapchain_framebuffers.iter() {
                self.device.destroy_framebuffer(*framebuffer, None);
            }

            self.device.destroy_pipeline(self.graphics_pipeline, None);
            self.device.destroy_pipeline_layout(self.pipeline_layout, None);
            self.device.destroy_render_pass(self.render_pass, None);

            for image_view in self.swapchain_image_views.iter() {
                self.device.destroy_image_view(*image_view, None);
            }

            // ... 나머지 리소스 정리 ...
        }
    }
}
```

이제 우리는 렌더링에 필요한 모든 객체를 갖추는 중요한 단계에 도달했습니다. 다음 장에서는 첫 실제 드로잉 명령을 작성할 것입니다.