## 소개

지금까지 우리가 다루었던 지오메트리는 3D로 투영되었지만, 실제로는 완전히 평면이었습니다. 이번 장에서는 3D 메시를 준비하기 위해 위치(position)에 Z 좌표를 추가할 것입니다. 이 세 번째 좌표를 사용하여 현재 사각형 위에 또 다른 사각형을 배치함으로써, 지오메트리가 깊이 순으로 정렬되지 않았을 때 발생하는 문제를 직접 확인해 보겠습니다.

## 3D 지오메트리

먼저 `Vertex` 구조체를 변경하여 위치에 3D 벡터를 사용하고, 그에 맞춰 `get_attribute_descriptions`의 `format`을 업데이트합니다. Rust에서는 `glam` 크레이트의 `Vec3`를 사용합니다.

```rust
#[repr(C)]
#[derive(Clone, Debug, Copy)]
struct Vertex {
    pos: glam::Vec3,
    color: glam::Vec3,
    tex_coord: glam::Vec2,
}

impl Vertex {
    // ... get_binding_description ...

    fn get_attribute_descriptions() -> [vk::VertexInputAttributeDescription; 3] {
        [
            vk::VertexInputAttributeDescription {
                binding: 0,
                location: 0,
                format: vk::Format::R32G32B32_SFLOAT,
                offset: memoffset::offset_of!(Vertex, pos) as u32,
            },
            // ... color and tex_coord attributes ...
        ]
    }
}
```

다음으로, 정점 셰이더가 3D 좌표를 입력으로 받아 변환하도록 수정합니다. 이 코드는 C++ 버전과 동일합니다. 수정 후에는 반드시 셰이더를 다시 컴파일해야 합니다!

```glsl
layout(location = 0) in vec3 inPosition;

...

void main() {
    gl_Position = ubo.proj * ubo.view * ubo.model * vec4(inPosition, 1.0);
    fragColor = inColor;
    fragTexCoord = inTexCoord;
}
```

마지막으로, `VERTICES` 상수를 Z 좌표를 포함하도록 업데이트합니다.

```rust
const VERTICES: [Vertex; 4] = [
    Vertex { pos: glam::vec3(-0.5, -0.5, 0.0), color: glam::vec3(1.0, 0.0, 0.0), tex_coord: glam::vec2(0.0, 0.0) },
    Vertex { pos: glam::vec3(0.5, -0.5, 0.0), color: glam::vec3(0.0, 1.0, 0.0), tex_coord: glam::vec2(1.0, 0.0) },
    Vertex { pos: glam::vec3(0.5, 0.5, 0.0), color: glam::vec3(0.0, 0.0, 1.0), tex_coord: glam::vec2(1.0, 1.0) },
    Vertex { pos: glam::vec3(-0.5, 0.5, 0.0), color: glam::vec3(1.0, 1.0, 1.0), tex_coord: glam::vec2(0.0, 1.0) },
];
```

지금 애플리케이션을 실행하면 이전과 완전히 동일한 결과를 볼 수 있습니다. 이제 장면을 더 흥미롭게 만들고 이번 장에서 다룰 문제를 보여주기 위해 지오메트리를 추가할 시간입니다. 현재 사각형 바로 아래에 위치할 사각형을 정의하기 위해 정점들을 복제합니다.

![](/images/extra_square.svg)

새 사각형의 Z 좌표는 `-0.5f`로 설정하고, 추가된 사각형에 대한 인덱스도 추가합니다.

```rust
const VERTICES: [Vertex; 8] = [
    Vertex { pos: glam::vec3(-0.5, -0.5, 0.0), color: glam::vec3(1.0, 0.0, 0.0), tex_coord: glam::vec2(0.0, 0.0) },
    Vertex { pos: glam::vec3(0.5, -0.5, 0.0), color: glam::vec3(0.0, 1.0, 0.0), tex_coord: glam::vec2(1.0, 0.0) },
    Vertex { pos: glam::vec3(0.5, 0.5, 0.0), color: glam::vec3(0.0, 0.0, 1.0), tex_coord: glam::vec2(1.0, 1.0) },
    Vertex { pos: glam::vec3(-0.5, 0.5, 0.0), color: glam::vec3(1.0, 1.0, 1.0), tex_coord: glam::vec2(0.0, 1.0) },

    Vertex { pos: glam::vec3(-0.5, -0.5, -0.5), color: glam::vec3(1.0, 0.0, 0.0), tex_coord: glam::vec2(0.0, 0.0) },
    Vertex { pos: glam::vec3(0.5, -0.5, -0.5), color: glam::vec3(0.0, 1.0, 0.0), tex_coord: glam::vec2(1.0, 0.0) },
    Vertex { pos: glam::vec3(0.5, 0.5, -0.5), color: glam::vec3(0.0, 0.0, 1.0), tex_coord: glam::vec2(1.0, 1.0) },
    Vertex { pos: glam::vec3(-0.5, 0.5, -0.5), color: glam::vec3(1.0, 1.0, 1.0), tex_coord: glam::vec2(0.0, 1.0) },
];

const INDICES: [u16; 12] = [
    0, 1, 2, 2, 3, 0,
    4, 5, 6, 6, 7, 4,
];
```

이제 프로그램을 실행하면 마치 에셔(Escher)의 그림과 같은 이상한 결과물을 보게 될 것입니다.

![](/images/depth_issues.png)

문제의 원인과 해결책(깊이 정렬 또는 깊이 버퍼 사용)은 C++ 버전과 동일합니다. 우리는 **깊이 버퍼**를 사용하여 이 문제를 해결할 것입니다.

C++에서 `GLM_FORCE_DEPTH_ZERO_TO_ONE` 매크로를 정의했던 것과 달리, `glam`에서는 `perspective_rh_zo` (Right-Handed, Zero-to-One) 함수를 사용하여 Vulkan의 `0.0`에서 `1.0` 깊이 범위를 사용하는 원근 투영 행렬을 직접 생성할 수 있습니다. `update_uniform_buffer`에서 이 함수를 사용하도록 수정해야 합니다.

```rust
// in update_uniform_buffer
let proj = glam::Mat4::perspective_rh_zo(
    45.0f32.to_radians(),
    self.swapchain_extent.width as f32 / self.swapchain_extent.height as f32,
    0.1,
    10.0,
);
```

## 깊이 이미지와 이미지 뷰

깊이 첨부는 이미지, 메모리, 이미지 뷰 세 가지 리소스가 필요합니다. 애플리케이션 구조체에 관련 필드를 추가합니다.

```rust
struct HelloTriangleApplication {
    // ...
    depth_image: vk::Image,
    depth_image_memory: vk::DeviceMemory,
    depth_image_view: vk::ImageView,
}
```

이 리소스들을 설정하기 위해 `create_depth_resources`라는 새 메서드를 만듭니다.

```rust
// in init_vulkan
// ...
self.create_command_pool();
self.create_depth_resources(); // 텍스처 이미지 생성 전에 호출
self.create_texture_image();
// ...

// ...

impl HelloTriangleApplication {
    // ...
    fn create_depth_resources(&mut self) {
        // ...
    }
}
```

깊이 이미지 포맷을 찾기 위해, C++ 버전과 유사한 `find_supported_format` 헬퍼 함수를 만듭니다.

```rust
fn find_supported_format(
    instance: &ash::Instance,
    physical_device: vk::PhysicalDevice,
    candidates: &[vk::Format],
    tiling: vk::ImageTiling,
    features: vk::FormatFeatureFlags,
) -> vk::Format {
    candidates.iter().cloned().find(|format| {
        let props = unsafe {
            instance.get_physical_device_format_properties(physical_device, *format)
        };
        if tiling == vk::ImageTiling::LINEAR {
            props.linear_tiling_features.contains(features)
        } else { // tiling == vk::ImageTiling::OPTIMAL
            props.optimal_tiling_features.contains(features)
        }
    }).expect("failed to find supported format!")
}
```

이 함수를 사용하여 깊이 포맷을 찾는 `find_depth_format` 함수를 만듭니다.

```rust
fn find_depth_format(
    instance: &ash::Instance,
    physical_device: vk::PhysicalDevice,
) -> vk::Format {
    find_supported_format(
        instance,
        physical_device,
        &[
            vk::Format::D32_SFLOAT,
            vk::Format::D32_SFLOAT_S8_UINT,
            vk::Format::D24_UNORM_S8_UINT,
        ],
        vk::ImageTiling::OPTIMAL,
        vk::FormatFeatureFlags::DEPTH_STENCIL_ATTACHMENT,
    )
}

fn has_stencil_component(format: vk::Format) -> bool {
    format == vk::Format::D32_SFLOAT_S8_UINT || format == vk::Format::D24_UNORM_S8_UINT
}
```

이제 `create_depth_resources` 메서드 내부를 채웁니다.

```rust
fn create_depth_resources(&mut self) {
    let depth_format = find_depth_format(&self.instance, self.physical_device);

    let (depth_image, depth_image_memory) = self.create_image(
        self.swapchain_extent.width,
        self.swapchain_extent.height,
        depth_format,
        vk::ImageTiling::OPTIMAL,
        vk::ImageUsageFlags::DEPTH_STENCIL_ATTACHMENT,
        vk::MemoryPropertyFlags::DEVICE_LOCAL,
    );
    self.depth_image = depth_image;
    self.depth_image_memory = depth_image_memory;

    self.depth_image_view = self.create_image_view(
        self.depth_image,
        depth_format,
        vk::ImageAspectFlags::DEPTH,
    );

    // 레이아웃 전환은 선택사항이며 렌더 패스에서 처리됩니다.
    // 명시적으로 전환하려면 아래 코드를 사용합니다.
    // self.transition_image_layout(
    //     self.depth_image,
    //     depth_format,
    //     vk::ImageLayout::UNDEFINED,
    //     vk::ImageLayout::DEPTH_STENCIL_ATTACHMENT_OPTIMAL,
    // );
}
```

`create_image_view` 함수가 `aspect_flags`를 인자로 받도록 수정해야 합니다.

```rust
fn create_image_view(
    &self,
    image: vk::Image,
    format: vk::Format,
    aspect_flags: vk::ImageAspectFlags,
) -> vk::ImageView {
    let view_info = vk::ImageViewCreateInfo::builder()
        .image(image)
        .view_type(vk::ImageViewType::TYPE_2D)
        .format(format)
        .subresource_range(
            vk::ImageSubresourceRange::builder()
                .aspect_mask(aspect_flags)
                .base_mip_level(0)
                .level_count(1)
                .base_array_layer(0)
                .layer_count(1)
                .build(),
        );

    unsafe {
        self.device
            .create_image_view(&view_info, None)
            .expect("Failed to create image view!")
    }
}
```

모든 `create_image_view` 호출을 새로운 시그니처에 맞게 업데이트해야 합니다.

```rust
// in create_image_views
self.swapchain_image_views[i] = self.create_image_view(
    self.swapchain_images[i],
    self.swapchain_image_format,
    vk::ImageAspectFlags::COLOR,
);

// in create_texture_image
self.texture_image_view = self.create_image_view(
    self.texture_image,
    vk::Format::R8G8B8A8_SRGB,
    vk::ImageAspectFlags::COLOR,
);
```

## 렌더 패스

`create_render_pass` 메서드를 수정하여 깊이 첨부를 포함하도록 합니다. 먼저 `VkAttachmentDescription`을 추가합니다. `ash`의 빌더 패턴을 사용합니다.

```rust
// in create_render_pass
let depth_format = find_depth_format(&self.instance, self.physical_device);
let depth_attachment = vk::AttachmentDescription::builder()
    .format(depth_format)
    .samples(vk::SampleCountFlags::TYPE_1)
    .load_op(vk::AttachmentLoadOp::CLEAR)
    .store_op(vk::AttachmentStoreOp::DONT_CARE)
    .stencil_load_op(vk::AttachmentLoadOp::DONT_CARE)
    .stencil_store_op(vk::AttachmentStoreOp::DONT_CARE)
    .initial_layout(vk::ImageLayout::UNDEFINED)
    .final_layout(vk::ImageLayout::DEPTH_STENCIL_ATTACHMENT_OPTIMAL);
```

깊이 첨부에 대한 참조를 추가합니다.

```rust
let depth_attachment_ref = vk::AttachmentReference::builder()
    .attachment(1)
    .layout(vk::ImageLayout::DEPTH_STENCIL_ATTACHMENT_OPTIMAL);
```

서브패스 설명(`SubpassDescription`)을 업데이트하여 깊이-스텐실 첨부를 참조하도록 합니다.

```rust
let subpass = vk::SubpassDescription::builder()
    .pipeline_bind_point(vk::PipelineBindPoint::GRAPHICS)
    .color_attachments(std::slice::from_ref(&color_attachment_ref))
    .depth_stencil_attachment(&depth_attachment_ref);
```

렌더 패스 생성 정보에 두 첨부를 모두 포함합니다.

```rust
let attachments = [color_attachment.build(), depth_attachment.build()];

let dependency = vk::SubpassDependency::builder()
    .src_subpass(vk::SUBPASS_EXTERNAL)
    .dst_subpass(0)
    .src_stage_mask(
        vk::PipelineStageFlags::COLOR_ATTACHMENT_OUTPUT
            | vk::PipelineStageFlags::EARLY_FRAGMENT_TESTS,
    )
    .src_access_mask(vk::AccessFlags::empty())
    .dst_stage_mask(
        vk::PipelineStageFlags::COLOR_ATTACHMENT_OUTPUT
            | vk::PipelineStageFlags::EARLY_FRAGMENT_TESTS,
    )
    .dst_access_mask(
        vk::AccessFlags::COLOR_ATTACHMENT_WRITE
            | vk::AccessFlags::DEPTH_STENCIL_ATTACHMENT_WRITE,
    );

let render_pass_info = vk::RenderPassCreateInfo::builder()
    .attachments(&attachments)
    .subpasses(std::slice::from_ref(&subpass))
    .dependencies(std::slice::from_ref(&dependency));
```
`SubpassDependency`는 렌더 패스가 시작되기 전에 이미지(색상 및 깊이)가 준비되도록 보장합니다. 깊이 버퍼는 `LOAD_OP_CLEAR`로 인해 `EARLY_FRAGMENT_TESTS` 단계에서 쓰여지므로, `dstStageMask`와 `dstAccessMask`에 관련 플래그를 추가하여 이 쓰기 작업을 허용해야 합니다.

## 프레임버퍼

`create_framebuffers` 메서드를 수정하여 깊이 이미지 뷰를 두 번째 첨부로 바인딩합니다.

```rust
// in create_framebuffers
let attachments = [self.swapchain_image_views[i], self.depth_image_view];

let framebuffer_info = vk::FramebufferCreateInfo::builder()
    .render_pass(self.render_pass)
    .attachments(&attachments)
    .width(self.swapchain_extent.width)
    .height(self.swapchain_extent.height)
    .layers(1);
```

깊이 이미지 뷰가 생성된 후에 프레임버퍼가 생성되도록 `init_vulkan` 내의 호출 순서를 조정해야 합니다.

```rust
// in init_vulkan
// ...
self.create_depth_resources();
self.create_framebuffers();
// ...
```

## 소거 값 (Clear values)

여러 첨부를 소거하므로, `record_command_buffer`에서 여러 소거 값을 지정해야 합니다.

```rust
// in record_command_buffer
let clear_values = [
    vk::ClearValue {
        color: vk::ClearColorValue {
            float32: [0.0, 0.0, 0.0, 1.0],
        },
    },
    vk::ClearValue {
        depth_stencil: vk::ClearDepthStencilValue {
            depth: 1.0,
            stencil: 0,
        },
    },
];

let render_pass_info = vk::RenderPassBeginInfo::builder()
    .render_pass(self.render_pass)
    .framebuffer(self.swapchain_framebuffers[image_index as usize])
    .render_area(render_area)
    .clear_values(&clear_values); // clear_values 슬라이스 전달
```
`vk::ClearValue`는 Rust에서 union처럼 동작하는 구조체입니다. 색상에는 `color` 필드를, 깊이/스텐실에는 `depth_stencil` 필드를 사용합니다. 깊이의 초기 값은 가장 먼 거리인 `1.0`으로 설정합니다. `clear_values` 배열의 순서는 렌더 패스의 첨부 순서와 일치해야 합니다.

## 깊이 및 스텐실 상태

파이프라인을 생성할 때 `VkPipelineDepthStencilStateCreateInfo`를 통해 깊이 테스팅을 활성화해야 합니다.

```rust
// in create_graphics_pipeline
let depth_stencil_state = vk::PipelineDepthStencilStateCreateInfo::builder()
    .depth_test_enable(true)
    .depth_write_enable(true)
    .depth_compare_op(vk::CompareOp::LESS)
    .depth_bounds_test_enable(false)
    .min_depth_bounds(0.0) // Optional
    .max_depth_bounds(1.0) // Optional
    .stencil_test_enable(false);

let pipeline_info = vk::GraphicsPipelineCreateInfo::builder()
    // ...
    .depth_stencil_state(&depth_stencil_state)
    // ...
```

이제 `GraphicsPipelineCreateInfo` 빌더에 깊이 스텐실 상태를 연결합니다. 렌더 패스가 깊이 첨부를 포함하면 이 상태는 항상 지정되어야 합니다.

이제 프로그램을 실행하면, 지오메트리의 프래그먼트가 올바르게 정렬된 것을 볼 수 있습니다.

![](/images/depth_correct.png)

## 창 크기 조절 처리

창 크기가 조절될 때 깊이 버퍼도 재생성되어야 합니다. `recreate_swapchain` 메서드를 수정합니다.

```rust
fn recreate_swapchain(&mut self) {
    // ...
    unsafe {
        self.device.device_wait_idle().unwrap();
    }
    self.cleanup_swapchain();

    self.create_swapchain();
    self.create_image_views();
    self.create_depth_resources(); // 여기서 깊이 리소스 재생성
    self.create_framebuffers();
}
```

스왑 체인 정리 함수에 깊이 리소스 정리 코드를 추가합니다.

```rust
fn cleanup_swapchain(&mut self) {
    unsafe {
        self.device.destroy_image_view(self.depth_image_view, None);
        self.device.destroy_image(self.depth_image, None);
        self.device.free_memory(self.depth_image_memory, None);

        // ... 다른 리소스 정리
    }
}
```

축하합니다. 이제 여러분의 Rust 애플리케이션은 임의의 3D 지오메트리를 올바르게 렌더링할 준비가 되었습니다. 다음 장에서는 텍스처가 입혀진 모델을 그려보며 이를 시험해 보겠습니다