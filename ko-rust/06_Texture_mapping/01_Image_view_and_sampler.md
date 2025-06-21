이번 장에서는 그래픽스 파이프라인이 이미지를 샘플링하는 데 필요한 두 가지 리소스를 더 만들 것입니다. 첫 번째 리소스는 스왑 체인 이미지에서 이미 다루었던 것이지만, 두 번째 리소스는 새로운 것으로 셰이더가 이미지에서 텍셀(texel)을 어떻게 읽을지와 관련이 있습니다.

## 텍스처 이미지 뷰

우리는 이전에 스왑 체인 이미지와 프레임버퍼에서 이미지가 직접 접근되는 대신 이미지 뷰를 통해 접근된다는 것을 보았습니다. 텍스처 이미지에 대해서도 이러한 이미지 뷰를 만들어야 합니다.

텍스처 이미지의 `ash::vk::ImageView`를 저장할 구조체 필드를 추가하고, 이를 생성할 `create_texture_image_view` 메서드를 새로 만듭니다.

```rust
struct HelloTriangleApplication {
    // ...
    texture_image: vk::Image,
    texture_image_memory: vk::DeviceMemory,
    texture_image_view: vk::ImageView,
    // ...
}

impl HelloTriangleApplication {
    pub fn new(window: &Window) -> Self {
        // ...
    }

    fn init_vulkan(&mut self, window: &Window) {
        // ...
        self.create_texture_image();
        self.create_texture_image_view();
        self.create_vertex_buffer();
        // ...
    }

    // ...

    fn create_texture_image_view(&mut self) {
        // ...
    }
}
```

이 메서드의 코드는 `create_image_views` 메서드를 거의 그대로 가져와서 만들 수 있습니다. 변경해야 할 부분은 `format`과 `image` 단 두 가지뿐입니다.

```rust
let view_info = vk::ImageViewCreateInfo::builder()
    .image(self.texture_image)
    .view_type(vk::ImageViewType::TYPE_2D)
    .format(vk::Format::R8G8B8A8_SRGB)
    .subresource_range(
        vk::ImageSubresourceRange::builder()
            .aspect_mask(vk::ImageAspectFlags::COLOR)
            .base_mip_level(0)
            .level_count(1)
            .base_array_layer(0)
            .layer_count(1)
            .build(),
    );

self.texture_image_view = unsafe {
    self.device
        .create_image_view(&view_info, None)
        .expect("Failed to create texture image view!")
};
```
C++ 버전에서 `viewInfo.components` 초기화를 생략한 것처럼, Rust의 빌더 패턴에서도 기본값은 `IDENTITY`이므로 명시적으로 설정할 필요가 없습니다.

`create_image_views`와 많은 로직이 중복되므로, 이를 새로운 `create_image_view` 헬퍼 메서드로 추상화할 수 있습니다.

```rust
impl HelloTriangleApplication {
    // ...
    fn create_image_view(
        &self,
        image: vk::Image,
        format: vk::Format,
    ) -> vk::ImageView {
        let view_info = vk::ImageViewCreateInfo::builder()
            .image(image)
            .view_type(vk::ImageViewType::TYPE_2D)
            .format(format)
            .subresource_range(
                vk::ImageSubresourceRange::builder()
                    .aspect_mask(vk::ImageAspectFlags::COLOR)
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
    // ...
}
```

이제 `create_texture_image_view` 메서드는 다음과 같이 단순화할 수 있습니다.

```rust
fn create_texture_image_view(&mut self) {
    self.texture_image_view =
        self.create_image_view(self.texture_image, vk::Format::R8G8B8A8_SRGB);
}
```

그리고 `create_image_views`도 다음과 같이 단순화됩니다.

```rust
fn create_image_views(&mut self) {
    self.swapchain_image_views = self
        .swapchain_images
        .iter()
        .map(|&image| {
            self.create_image_view(image, self.swapchain_image_format)
        })
        .collect();
}
```

프로그램이 끝날 때, 이미지 자체를 파괴하기 직전에 이미지 뷰를 파괴하도록 `cleanup` 메서드를 수정해야 합니다.

```rust
impl Drop for HelloTriangleApplication {
    fn drop(&mut self) {
        unsafe {
            self.cleanup_swapchain();

            self.device.destroy_image_view(self.texture_image_view, None);

            self.device.destroy_image(self.texture_image, None);
            self.device.free_memory(self.texture_image_memory, None);
            // ...
        }
    }
}
```

## 샘플러

셰이더가 이미지에서 직접 텍셀을 읽는 것도 가능하지만, 이미지가 텍스처로 사용될 때는 흔한 방식이 아닙니다. 텍스처는 보통 샘플러를 통해 접근되며, 샘플러는 최종적으로 검색될 색상을 계산하기 위해 필터링과 변환을 적용합니다.

이러한 필터들은 오버샘플링(oversampling) 같은 문제를 해결하는 데 유용합니다. 텍셀보다 더 많은 프래그먼트가 있는 지오메트리에 텍스처가 매핑되는 경우를 생각해보세요. 만약 각 프래그먼트의 텍스처 좌표에 가장 가까운 텍셀을 단순히 가져온다면, 아래 첫 번째 이미지와 같은 결과를 얻게 될 것입니다.

![](/images/texture_filtering.png)

만약 가장 가까운 4개의 텍셀을 선형 보간(linear interpolation)으로 혼합한다면, 오른쪽 이미지처럼 더 부드러운 결과를 얻을 수 있습니다. 물론 애플리케이션의 아트 스타일에 따라 왼쪽 스타일(마인크래프트처럼)이 더 적합할 수도 있지만, 일반적인 그래픽스 애플리케이션에서는 오른쪽 방식이 선호됩니다. 샘플러 객체는 텍스처에서 색상을 읽을 때 이 필터링을 자동으로 적용해줍니다.

언더샘플링(undersampling)은 그 반대의 문제로, 프래그먼트보다 텍셀이 더 많은 경우입니다. 이는 체커보드 텍스처처럼 고주파 패턴을 예리한 각도에서 샘플링할 때 아티팩트를 유발합니다.

![](/images/anisotropic_filtering.png)

왼쪽 이미지에서 보듯이, 텍스처가 멀어질수록 흐릿한 덩어리로 변합니다. 이에 대한 해결책은 [비등방성 필터링(anisotropic filtering)](https://ko.wikipedia.org/wiki/%EB%B9%84%EB%93%B1%EB%B0%A9%EC%84%B1_%ED%95%84%ED%84%B0%EB%A7%81)이며, 이 또한 샘플러에 의해 자동으로 적용될 수 있습니다.

이러한 필터 외에도, 샘플러는 변환도 처리할 수 있습니다. 샘플러는 *주소 지정 모드(addressing mode)*를 통해 이미지 외부의 텍셀을 읽으려고 할 때 어떤 일이 일어날지를 결정합니다. 아래 이미지는 몇 가지 가능한 옵션을 보여줍니다.

![](/images/texture_addressing.png)

이제 이러한 샘플러 객체를 설정하기 위해 `create_texture_sampler` 메서드를 만들 것입니다. 나중에 셰이더에서 이 샘플러를 사용해 텍스처로부터 색상을 읽게 됩니다.

```rust
struct HelloTriangleApplication {
    // ...
    texture_image_view: vk::ImageView,
    texture_sampler: vk::Sampler,
    // ...
}

// ...
fn init_vulkan(&mut self, window: &Window) {
    // ...
    self.create_texture_image();
    self.create_texture_image_view();
    self.create_texture_sampler();
    self.create_vertex_buffer();
    // ...
}

// ...
fn create_texture_sampler(&mut self) {
    // ...
}
```

샘플러는 `ash::vk::SamplerCreateInfo` 구조체를 통해 구성되며, 이 구조체는 샘플러가 적용해야 할 모든 필터와 변환을 명시합니다. Ash의 빌더 패턴을 사용하여 생성합니다.

```rust
let sampler_info = vk::SamplerCreateInfo::builder()
    .mag_filter(vk::Filter::LINEAR)
    .min_filter(vk::Filter::LINEAR);
```

`mag_filter`와 `min_filter` 필드는 텍셀이 확대되거나 축소될 때 어떻게 보간할지를 지정합니다. 확대는 위에서 설명한 오버샘플링 문제와 관련이 있고, 축소는 언더샘플링 문제와 관련이 있습니다. 선택지는 `VK_FILTER_NEAREST`와 `VK_FILTER_LINEAR`이며, 이는 위 이미지에서 보여준 모드에 해당합니다.

```rust
let sampler_info = vk::SamplerCreateInfo::builder()
    .mag_filter(vk::Filter::LINEAR)
    .min_filter(vk::Filter::LINEAR)
    .address_mode_u(vk::SamplerAddressMode::REPEAT)
    .address_mode_v(vk::SamplerAddressMode::REPEAT)
    .address_mode_w(vk::SamplerAddressMode::REPEAT);
```

주소 지정 모드는 축별로 지정할 수 있습니다. 사용 가능한 값은 다음과 같습니다. 대부분은 위 이미지에서 시연되었습니다. 축이 X, Y, Z 대신 U, V, W로 불리는 점에 유의하세요. 이는 텍스처 공간 좌표의 관례입니다.

*   `vk::SamplerAddressMode::REPEAT`: 이미지 크기를 벗어날 때 텍스처를 반복합니다.
*   `vk::SamplerAddressMode::MIRRORED_REPEAT`: 반복과 같지만, 크기를 벗어날 때 좌표를 반전시켜 이미지를 거울처럼 반사합니다.
*   `vk::SamplerAddressMode::CLAMP_TO_EDGE`: 이미지 크기를 벗어나는 좌표에 대해 가장 가까운 가장자리의 색상을 사용합니다.
*   `vk::SamplerAddressMode::MIRROR_CLAMP_TO_EDGE`: 가장자리 클램프와 비슷하지만, 가장 가까운 가장자리가 아닌 반대쪽 가장자리를 사용합니다.
*   `vk::SamplerAddressMode::CLAMP_TO_BORDER`: 이미지 크기 밖을 샘플링할 때 지정된 단색을 반환합니다.

이 튜토리얼에서는 이미지 외부를 샘플링하지 않을 것이므로 어떤 주소 지정 모드를 사용하든 큰 차이는 없습니다. 하지만 바닥이나 벽처럼 텍스처를 타일링하는 데 사용될 수 있기 때문에 반복 모드가 아마 가장 일반적일 것입니다.

```rust
// ...
    .anisotropy_enable(true)
    .max_anisotropy(???)
// ...
```

이 두 필드는 비등방성 필터링을 사용할지 여부를 지정합니다. 성능이 우려되는 경우가 아니라면 사용하지 않을 이유가 없습니다. `max_anisotropy` 필드는 최종 색상을 계산하는 데 사용될 수 있는 텍셀 샘플의 양을 제한합니다. 값이 낮을수록 성능은 좋아지지만 결과물의 품질은 떨어집니다. 우리가 사용할 수 있는 값을 알아내려면, 다음과 같이 물리 장치의 속성을 가져와야 합니다.

```rust
let properties = unsafe { self.instance.get_physical_device_properties(self.physical_device) };
```

`ash::vk::PhysicalDeviceProperties` 구조체를 보면 `limits`라는 `ash::vk::PhysicalDeviceLimits` 타입의 필드가 있습니다. 이 구조체는 다시 `max_sampler_anisotropy`라는 필드를 가지고 있으며, 이것이 `max_anisotropy`에 지정할 수 있는 최대값입니다. 최고의 품질을 원한다면 이 값을 직접 사용하면 됩니다.

```rust
// ...
    .max_anisotropy(properties.limits.max_sampler_anisotropy)
// ...
```

`create_texture_sampler` 메서드 내에서 속성을 조회할 수 있습니다.

```rust
// ...
    .border_color(vk::BorderColor::INT_OPAQUE_BLACK)
    .unnormalized_coordinates(false)
// ...
```

`border_color` 필드는 `clamp to border` 주소 지정 모드로 이미지 외부를 샘플링할 때 반환될 색상을 지정합니다. `unnormalized_coordinates` 필드는 텍셀 주소에 정규화된 좌표(`[0, 1)`)를 사용할지 여부를 지정합니다. 실제 애플리케이션에서는 거의 항상 정규화된 좌표를 사용합니다.

```rust
// ...
    .compare_enable(false)
    .compare_op(vk::CompareOp::ALWAYS)
// ...
```
비교 함수는 주로 섀도 맵에 사용되며, 여기서는 비활성화합니다.

```rust
// ...
    .mipmap_mode(vk::SamplerMipmapMode::LINEAR)
    .mip_lod_bias(0.0)
    .min_lod(0.0)
    .max_lod(0.0);
```
이 필드들은 모두 밉매핑에 적용됩니다. 밉매핑은 다음 장에서 다룰 것입니다.

이제 샘플러의 모든 설정이 완료되었습니다. `vkCreateSampler`를 호출하여 샘플러를 생성합니다.

```rust
fn create_texture_sampler(&mut self) {
    let properties = unsafe { self.instance.get_physical_device_properties(self.physical_device) };

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
        .mip_lod_bias(0.0)
        .min_lod(0.0)
        .max_lod(0.0);

    self.texture_sampler = unsafe {
        self.device
            .create_sampler(&sampler_info, None)
            .expect("Failed to create texture sampler!")
    };
}
```

샘플러는 어디에도 `vk::Image`를 참조하지 않는다는 점에 유의하세요. 샘플러는 텍스처에서 색상을 추출하는 인터페이스를 제공하는 별개의 객체입니다.

프로그램이 종료될 때 샘플러를 파괴하도록 `drop` 구현을 수정합니다.

```rust
impl Drop for HelloTriangleApplication {
    fn drop(&mut self) {
        unsafe {
            self.cleanup_swapchain();

            self.device.destroy_sampler(self.texture_sampler, None);
            self.device.destroy_image_view(self.texture_image_view, None);
            self.device.destroy_image(self.texture_image, None);
            self.device.free_memory(self.texture_image_memory, None);

            // ...
        }
    }
}
```

## 비등방성 장치 기능

지금 프로그램을 실행하면 다음과 같은 검증 레이어 메시지를 볼 수 있습니다.

![](/images/validation_layer_anisotropy.png)

이는 비등방성 필터링이 사실 선택적(optional) 장치 기능이기 때문입니다. 이를 요청하도록 `create_logical_device` 메서드를 업데이트해야 합니다.

```rust
// in create_logical_device
let mut features = vk::PhysicalDeviceFeatures::builder();
features.sampler_anisotropy = vk::TRUE;

// ...
let create_info = vk::DeviceCreateInfo::builder()
    .queue_create_infos(&queue_create_infos)
    .enabled_extension_names(&device_extensions_raw)
    .enabled_features(&features);
```

그리고 최신 그래픽 카드가 이를 지원하지 않을 가능성은 매우 낮지만, `is_device_suitable` 함수를 업데이트하여 사용 가능한지 확인해야 합니다.

```rust
// in is_device_suitable
let supported_features = unsafe { instance.get_physical_device_features(device) };

// ...
indices.is_complete()
    && extensions_supported
    && swapchain_adequate
    && supported_features.sampler_anisotropy == vk::TRUE
```

`get_physical_device_features`는 `ash::vk::PhysicalDeviceFeatures` 구조체를 사용하여 지원되는 기능을 나타냅니다.

비등방성 필터링의 사용 가능성을 강제하는 대신, 조건부로 사용하지 않도록 설정할 수도 있습니다.

```rust
// ...
    .anisotropy_enable(false)
    .max_anisotropy(1.0)
// ...
```

다음 장에서는 이미지와 샘플러 객체를 셰이더에 노출하여 사각형에 텍스처를 그릴 것입니다.

(참고: 코드 링크는 원본 C++ 튜토리얼을 가리킵니다.)

[C++ 코드](/code/25_sampler.cpp) /
[정점 셰이더](/code/22_shader_ubo.vert) /
[프래그먼트 셰이더](/code/22_shader_ubo.frag)