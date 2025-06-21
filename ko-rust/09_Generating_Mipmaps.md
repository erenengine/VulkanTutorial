## 서론
이제 우리 프로그램은 3D 모델을 로드하고 렌더링할 수 있습니다. 이번 장에서는 밉맵 생성이라는 기능을 하나 더 추가할 것입니다. 밉맵은 게임과 렌더링 소프트웨어에서 널리 사용되며, Vulkan은 밉맵 생성 방법을 완벽하게 제어할 수 있도록 해줍니다.

밉맵은 미리 계산된, 축소된 버전의 이미지입니다. 각각의 새 이미지는 이전 이미지의 너비와 높이가 절반입니다. 밉맵은 *디테일 수준(Level of Detail, LOD)*의 한 형태로 사용됩니다. 카메라에서 멀리 떨어진 객체는 더 작은 밉 이미지에서 텍스처를 샘플링합니다. 더 작은 이미지를 사용하면 렌더링 속도가 향상되고 [모아레 패턴](https://ko.wikipedia.org/wiki/%EB%AC%B4%EC%95%84%EB%A0%88_%EB%AC%B4%EB%8A%AC)과 같은 아티팩트를 방지할 수 있습니다. 밉맵이 어떻게 생겼는지 보여주는 예시는 다음과 같습니다:

![](/images/mipmaps_example.jpg)

## 이미지 생성

Vulkan에서 각 밉 이미지는 `vk::Image`의 서로 다른 *밉 레벨(mip level)*에 저장됩니다. 밉 레벨 0은 원본 이미지이며, 레벨 0 이후의 밉 레벨들은 흔히 *밉 체인(mip chain)*이라고 불립니다.

밉 레벨의 수는 `vk::Image`를 생성할 때 지정됩니다. 지금까지 우리는 항상 이 값을 1로 설정했습니다. 이제 이미지의 크기로부터 밉 레벨의 수를 계산해야 합니다. 먼저, 이 수를 저장할 구조체 필드를 추가합니다:

```rust
struct HelloTriangleApplication {
    ...
    mip_levels: u32,
    texture_image: vk::Image,
    ...
}
```

`mip_levels`의 값은 `create_texture_image`에서 텍스처를 로드한 후에 계산할 수 있습니다:

```rust
let image = image::load_from_memory(include_bytes!(TEXTURE_PATH))
    .expect("Failed to load texture image")
    .to_rgba8();
let (tex_width, tex_height) = image.dimensions();
let image_data = image.into_raw();
...
self.mip_levels = ((tex_width.max(tex_height) as f32).log2().floor() + 1.0) as u32;
```

이 코드는 밉 체인의 레벨 수를 계산합니다. `max` 메서드는 가장 큰 차원(너비 또는 높이)을 선택합니다. `log2` 메서드는 해당 차원을 2로 몇 번 나눌 수 있는지 계산합니다. `floor` 메서드는 가장 큰 차원이 2의 거듭제곱이 아닌 경우를 처리합니다. `1`을 더해서 원본 이미지 자체도 밉 레벨을 갖도록 합니다.

이 값을 사용하려면 `create_image`, `create_image_view`, `transition_image_layout` 함수를 수정하여 밉 레벨 수를 지정할 수 있도록 해야 합니다. 함수들에 `mip_levels` 매개변수를 추가하세요:

```rust
unsafe fn create_image(
    &self,
    width: u32,
    height: u32,
    mip_levels: u32,
    format: vk::Format,
    tiling: vk::ImageTiling,
    usage: vk::ImageUsageFlags,
    properties: vk::MemoryPropertyFlags,
) -> (vk::Image, vk::DeviceMemory) {
    ...
    let image_info = vk::ImageCreateInfo::builder()
        ...
        .mip_levels(mip_levels)
        ...;
    ...
}
```
```rust
unsafe fn create_image_view(
    &self,
    image: vk::Image,
    format: vk::Format,
    aspect_flags: vk::ImageAspectFlags,
    mip_levels: u32,
) -> vk::ImageView {
    ...
    let subresource_range = vk::ImageSubresourceRange::builder()
        ...
        .level_count(mip_levels);

    let view_info = vk::ImageViewCreateInfo::builder()
        .subresource_range(*subresource_range);
    ...
}
```
```rust
unsafe fn transition_image_layout(
    &self,
    image: vk::Image,
    format: vk::Format,
    old_layout: vk::ImageLayout,
    new_layout: vk::ImageLayout,
    mip_levels: u32,
) {
    ...
    let barrier = vk::ImageMemoryBarrier::builder()
        ...
        .subresource_range(vk::ImageSubresourceRange {
            aspect_mask: vk::ImageAspectFlags::COLOR,
            base_mip_level: 0,
            level_count: mip_levels,
            base_array_layer: 0,
            layer_count: 1,
        });
    ...
}
```

이 함수들에 대한 모든 호출을 올바른 값을 사용하도록 업데이트합니다 (`unsafe` 블록 안에서 호출해야 합니다):

```rust
let (depth_image, depth_image_memory) = self.create_image(
    self.swapchain_extent.width,
    self.swapchain_extent.height,
    1,
    depth_format,
    vk::ImageTiling::OPTIMAL,
    vk::ImageUsageFlags::DEPTH_STENCIL_ATTACHMENT,
    vk::MemoryPropertyFlags::DEVICE_LOCAL,
);
...
let (texture_image, texture_image_memory) = self.create_image(
    tex_width,
    tex_height,
    self.mip_levels,
    vk::Format::R8G8B8A8_SRGB,
    vk::ImageTiling::OPTIMAL,
    vk::ImageUsageFlags::TRANSFER_DST | vk::ImageUsageFlags::SAMPLED,
    vk::MemoryPropertyFlags::DEVICE_LOCAL,
);
```
```rust
self.swapchain_image_views = self
    .swapchain_images
    .iter()
    .map(|&image| {
        self.create_image_view(
            image,
            self.swapchain_image_format,
            vk::ImageAspectFlags::COLOR,
            1,
        )
    })
    .collect();
...
self.depth_image_view = self.create_image_view(
    self.depth_image,
    depth_format,
    vk::ImageAspectFlags::DEPTH,
    1,
);
...
self.texture_image_view = self.create_image_view(
    self.texture_image,
    vk::Format::R8G8B8A8_SRGB,
    vk::ImageAspectFlags::COLOR,
    self.mip_levels,
);
```
```rust
self.transition_image_layout(
    self.depth_image,
    depth_format,
    vk::ImageLayout::UNDEFINED,
    vk::ImageLayout::DEPTH_STENCIL_ATTACHMENT_OPTIMAL,
    1,
);
...
self.transition_image_layout(
    self.texture_image,
    vk::Format::R8G8B8A8_SRGB,
    vk::ImageLayout::UNDEFINED,
    vk::ImageLayout::TRANSFER_DST_OPTIMAL,
    self.mip_levels,
);
```

## 밉맵 생성하기

이제 우리의 텍스처 이미지는 여러 밉 레벨을 가지지만, 스테이징 버퍼는 밉 레벨 0을 채우는 데만 사용될 수 있습니다. 다른 레벨들은 여전히 정의되지 않은 상태입니다. 이 레벨들을 채우려면 우리가 가진 단일 레벨로부터 데이터를 생성해야 합니다. 이를 위해 `vkCmdBlitImage` 명령을 사용할 것입니다. 이 명령은 복사, 스케일링, 필터링 연산을 수행합니다. 우리는 이 명령을 여러 번 호출하여 우리 텍스처 이미지의 각 레벨로 데이터를 *블릿(blit)*할 것입니다.

`vkCmdBlitImage`는 전송 작업으로 간주되므로, 텍스처 이미지를 전송의 소스(source)와 대상(destination)으로 모두 사용할 것임을 Vulkan에 알려야 합니다. `create_texture_image`에서 텍스처 이미지의 사용 플래그에 `vk::ImageUsageFlags::TRANSFER_SRC`를 추가합니다:

```rust
let (texture_image, texture_image_memory) = self.create_image(
    tex_width,
    tex_height,
    self.mip_levels,
    vk::Format::R8G8B8A8_SRGB,
    vk::ImageTiling::OPTIMAL,
    vk::ImageUsageFlags::TRANSFER_SRC | vk::ImageUsageFlags::TRANSFER_DST | vk::ImageUsageFlags::SAMPLED,
    vk::MemoryPropertyFlags::DEVICE_LOCAL,
);
```

다른 이미지 작업과 마찬가지로, `vkCmdBlitImage`는 작동하는 이미지의 레이아웃에 의존합니다. `transition_image_layout`은 전체 이미지에 대해서만 레이아웃 전환을 수행하므로, 밉맵을 생성하기 위해 파이프라인 배리어 명령을 직접 기록해야 합니다. `create_texture_image`에서 `vk::ImageLayout::SHADER_READ_ONLY_OPTIMAL`로의 기존 전환을 제거합니다. 밉맵 생성 함수가 이 전환을 처리할 것입니다.

```rust
// In create_texture_image...
self.transition_image_layout(
    self.texture_image,
    vk::Format::R8G8B8A8_SRGB,
    vk::ImageLayout::UNDEFINED,
    vk::ImageLayout::TRANSFER_DST_OPTIMAL,
    self.mip_levels,
);
self.copy_buffer_to_image(
    staging_buffer,
    self.texture_image,
    tex_width,
    tex_height,
);
// 밉맵을 생성하는 동안 SHADER_READ_ONLY_OPTIMAL로 전환됨
```

이렇게 하면 텍스처 이미지의 각 레벨이 `vk::ImageLayout::TRANSFER_DST_OPTIMAL` 상태로 남게 됩니다. 이제 밉맵을 생성하는 함수를 작성해 보겠습니다:

```rust
unsafe fn generate_mipmaps(
    &self,
    image: vk::Image,
    tex_width: u32,
    tex_height: u32,
    mip_levels: u32,
) {
    let command_buffer = self.begin_single_time_commands();

    let mut barrier = vk::ImageMemoryBarrier::builder()
        .image(image)
        .src_queue_family_index(vk::QUEUE_FAMILY_IGNORED)
        .dst_queue_family_index(vk::QUEUE_FAMILY_IGNORED)
        .subresource_range(
            vk::ImageSubresourceRange::builder()
                .aspect_mask(vk::ImageAspectFlags::COLOR)
                .base_array_layer(0)
                .layer_count(1)
                .level_count(1)
                .build(),
        )
        .build();

    let mut mip_width = tex_width as i32;
    let mut mip_height = tex_height as i32;

    for i in 1..mip_levels {
        // ... 루프 내용
    }

    // ... 루프 후 배리어
    
    self.end_single_time_commands(command_buffer);
}
```

여러 번의 전환을 수행할 것이므로 `barrier` 변수를 재사용할 것입니다. `subresource_range.base_mip_level`, `old_layout`, `new_layout`, `src_access_mask`, `dst_access_mask`가 각 전환마다 변경됩니다.

루프는 각 `vkCmdBlitImage` 명령을 기록합니다. 루프 변수가 0이 아닌 1에서 시작하는 점에 유의하세요.

```rust
// 루프 내부
barrier.subresource_range.base_mip_level = i - 1;
barrier.old_layout = vk::ImageLayout::TRANSFER_DST_OPTIMAL;
barrier.new_layout = vk::ImageLayout::TRANSFER_SRC_OPTIMAL;
barrier.src_access_mask = vk::AccessFlags::TRANSFER_WRITE;
barrier.dst_access_mask = vk::AccessFlags::TRANSFER_READ;

self.device.cmd_pipeline_barrier(
    command_buffer,
    vk::PipelineStageFlags::TRANSFER,
    vk::PipelineStageFlags::TRANSFER,
    vk::DependencyFlags::empty(),
    &[],
    &[],
    &[barrier],
);
```
먼저, `i - 1` 레벨을 `vk::ImageLayout::TRANSFER_SRC_OPTIMAL`로 전환합니다. 이 전환은 이전 블릿 명령이나 `copy_buffer_to_image`로부터 `i - 1` 레벨이 채워질 때까지 기다립니다. 현재 블릿 명령은 이 전환을 기다립니다.

```rust
// 루프 내부, 첫 번째 배리어 다음
let blit = vk::ImageBlit::builder()
    .src_offsets([
        vk::Offset3D { x: 0, y: 0, z: 0 },
        vk::Offset3D { x: mip_width, y: mip_height, z: 1 },
    ])
    .src_subresource(
        vk::ImageSubresourceLayers::builder()
            .aspect_mask(vk::ImageAspectFlags::COLOR)
            .mip_level(i - 1)
            .base_array_layer(0)
            .layer_count(1)
            .build(),
    )
    .dst_offsets([
        vk::Offset3D { x: 0, y: 0, z: 0 },
        vk::Offset3D {
            x: if mip_width > 1 { mip_width / 2 } else { 1 },
            y: if mip_height > 1 { mip_height / 2 } else { 1 },
            z: 1,
        },
    ])
    .dst_subresource(
        vk::ImageSubresourceLayers::builder()
            .aspect_mask(vk::ImageAspectFlags::COLOR)
            .mip_level(i)
            .base_array_layer(0)
            .layer_count(1)
            .build(),
    )
    .build();

self.device.cmd_blit_image(
    command_buffer,
    image,
    vk::ImageLayout::TRANSFER_SRC_OPTIMAL,
    image,
    vk::ImageLayout::TRANSFER_DST_OPTIMAL,
    &[blit],
    vk::Filter::LINEAR,
);
```
이제 블릿 명령을 기록합니다. `src_image`와 `dst_image` 매개변수 모두에 `image`가 사용되는 점에 유의하세요. 소스 밉 레벨은 방금 `vk::ImageLayout::TRANSFER_SRC_OPTIMAL`로 전환되었고, 대상 레벨은 아직 `vk::ImageLayout::TRANSFER_DST_OPTIMAL` 상태입니다. 보간을 위해 필터로 `vk::Filter::LINEAR`를 사용합니다.

```rust
// 루프 내부, cmd_blit_image 다음
barrier.old_layout = vk::ImageLayout::TRANSFER_SRC_OPTIMAL;
barrier.new_layout = vk::ImageLayout::SHADER_READ_ONLY_OPTIMAL;
barrier.src_access_mask = vk::AccessFlags::TRANSFER_READ;
barrier.dst_access_mask = vk::AccessFlags::SHADER_READ;

self.device.cmd_pipeline_barrier(
    command_buffer,
    vk::PipelineStageFlags::TRANSFER,
    vk::PipelineStageFlags::FRAGMENT_SHADER,
    vk::DependencyFlags::empty(),
    &[],
    &[],
    &[barrier],
);

if mip_width > 1 { mip_width /= 2; }
if mip_height > 1 { mip_height /= 2; }
```
이 배리어는 밉 레벨 `i - 1`을 `vk::ImageLayout::SHADER_READ_ONLY_OPTIMAL`로 전환합니다. 이 전환은 현재 블릿 명령이 완료되기를 기다립니다. 그 후, 다음 이터레이션을 위해 밉 차원을 절반으로 줄입니다.

```rust
// 루프 이후
barrier.subresource_range.base_mip_level = mip_levels - 1;
barrier.old_layout = vk::ImageLayout::TRANSFER_DST_OPTIMAL;
barrier.new_layout = vk::ImageLayout::SHADER_READ_ONLY_OPTIMAL;
barrier.src_access_mask = vk::AccessFlags::TRANSFER_WRITE;
barrier.dst_access_mask = vk::AccessFlags::SHADER_READ;

self.device.cmd_pipeline_barrier(
    command_buffer,
    vk::PipelineStageFlags::TRANSFER,
    vk::PipelineStageFlags::FRAGMENT_SHADER,
    vk::DependencyFlags::empty(),
    &[],
    &[],
    &[barrier],
);

self.end_single_time_commands(command_buffer);
```
커맨드 버퍼를 종료하기 전에, 마지막 밉 레벨을 `vk::ImageLayout::SHADER_READ_ONLY_OPTIMAL`로 전환하는 배리어를 하나 더 삽입합니다. 마지막 밉 레벨은 블릿의 소스로 사용되지 않았기 때문에 루프에서 처리되지 않았습니다.

마지막으로, `create_texture_image`에서 `generate_mipmaps`를 호출합니다:

```rust
// in create_texture_image, after copy_buffer_to_image
self.generate_mipmaps(
    self.texture_image,
    tex_width,
    tex_height,
    self.mip_levels
);
```

## 선형 필터링 지원

`vkCmdBlitImage`는 편리하지만, 모든 하드웨어에서 선형 필터링을 지원하지는 않을 수 있습니다. `vkGetPhysicalDeviceFormatProperties` 함수로 이를 확인할 수 있습니다.

먼저 `generate_mipmaps` 함수에 `image_format` 매개변수를 추가하고, 호출하는 곳에서도 전달해줍니다.

```rust
// in create_texture_image
self.generate_mipmaps(
    self.texture_image,
    vk::Format::R8G8B8A8_SRGB,
    tex_width,
    tex_height,
    self.mip_levels,
);

// function signature
unsafe fn generate_mipmaps(
    &self,
    image: vk::Image,
    image_format: vk::Format,
    tex_width: u32,
    tex_height: u32,
    mip_levels: u32,
) {
    // ...
}
```

`generate_mipmaps` 함수 시작 부분에서, `get_physical_device_format_properties`를 사용하여 형식 속성을 확인합니다:

```rust
// in generate_mipmaps
let format_properties = self
    .instance
    .get_physical_device_format_properties(self.physical_device, image_format);

if !format_properties
    .optimal_tiling_features
    .contains(vk::FormatFeatureFlags::SAMPLED_IMAGE_FILTER_LINEAR)
{
    panic!("Texture image format does not support linear blitting!");
}
```

최적 타일링 이미지를 생성하므로 `optimal_tiling_features`를 확인합니다. 선형 필터링 기능 지원은 `vk::FormatFeatureFlags::SAMPLED_IMAGE_FILTER_LINEAR` 플래그로 확인할 수 있습니다. 지원되지 않는 경우, 다른 이미지 형식을 찾거나 [stb_image_resize](https://github.com/nothings/stb/blob/master/stb_image_resize.h)와 같은 라이브러리를 사용하여 소프트웨어에서 밉맵을 구현할 수 있습니다.

## 샘플러

`vk::Image`가 밉맵 데이터를 보유하는 동안, `vk::Sampler`는 렌더링 중에 해당 데이터를 읽는 방법을 제어합니다. Vulkan은 `min_lod`, `max_lod`, `mip_lod_bias`, `mipmap_mode`를 지정할 수 있게 해줍니다.

이 장의 결과를 보려면, `texture_sampler`의 설정을 업데이트해야 합니다. 이미 `min_filter`와 `mag_filter`는 `vk::Filter::LINEAR`로 설정했습니다. 이제 밉맵 관련 설정을 추가합니다.

```rust
// in create_texture_sampler
let sampler_info = vk::SamplerCreateInfo::builder()
    .mag_filter(vk::Filter::LINEAR)
    .min_filter(vk::Filter::LINEAR)
    .address_mode_u(vk::SamplerAddressMode::REPEAT)
    .address_mode_v(vk::SamplerAddressMode::REPEAT)
    .address_mode_w(vk::SamplerAddressMode::REPEAT)
    .anisotropy_enable(true)
    .max_anisotropy(properties.limits.max_sampler_anisotropy)
    .border_color(vk::BorderColor::INT_OPAQUE_BLACK)
    .unnormalized_coordinates(false)
    .compare_enable(false)
    .compare_op(vk::CompareOp::ALWAYS)
    .mipmap_mode(vk::SamplerMipmapMode::LINEAR)
    .min_lod(0.0)
    .max_lod(self.mip_levels as f32) // 모든 밉 레벨 사용
    .mip_lod_bias(0.0); // 선택 사항
```
모든 밉 레벨을 사용하려면 `min_lod`를 0.0으로, `max_lod`를 밉 레벨의 수로 설정합니다. `mip_lod_bias`는 lod 계산에 대한 편향을 추가하는 데 사용되며, 여기서는 0.0으로 둡니다.

이제 프로그램을 실행하면 다음과 같은 화면을 볼 수 있습니다:

![](/images/mipmaps.png)

장면이 단순해서 차이가 극적이지는 않지만, 종이에 적힌 글씨를 자세히 보면 차이점을 발견할 수 있습니다.

![](/images/mipmaps_comparison.png)

밉맵을 사용하면 글씨가 부드럽게 처리되지만, 밉맵이 없으면 모아레 아티팩트로 인해 거친 가장자리와 끊김이 보입니다.

`min_lod`와 같은 샘플러 설정을 변경하여 밉맵 효과를 시험해 볼 수 있습니다. 예를 들어, `min_lod`를 높이면 강제로 더 흐릿한(더 높은 레벨의) 밉맵을 사용하게 됩니다.

```rust
// in create_texture_sampler
...
.min_lod((self.mip_levels / 2) as f32)
.max_lod(self.mip_levels as f32)
...
```

이 설정은 객체가 카메라에서 더 멀리 있을 때 렌더링되는 모습과 유사한 이미지를 생성합니다:

![](/images/highmipmaps.png)