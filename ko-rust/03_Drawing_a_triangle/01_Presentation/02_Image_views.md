### Rust와 Ash 라이브러리를 사용한 이미지 뷰 생성

스왑 체인에 있는 이미지를 포함한 모든 `VkImage`를 렌더 파이프라인에서 사용하려면 `VkImageView` 객체를 생성해야 합니다. 이미지 뷰는 말 그대로 이미지로의 뷰(view)입니다. 이는 이미지에 접근하는 방법과 접근할 이미지의 부분을 기술합니다. 예를 들어, 밉매핑 레벨이 없는 2D 텍스처나 깊이 텍스처로 취급해야 하는지 등을 명시합니다.

이번 장에서는 스왑 체인의 모든 이미지에 대한 기본적인 이미지 뷰를 만드는 `create_image_views` 함수를 작성할 것입니다. 이렇게 하면 나중에 이미지 뷰들을 컬러 타겟으로 사용할 수 있습니다.

먼저, 이미지 뷰를 저장할 구조체 필드를 추가합니다.

```rust
struct VulkanApp {
    // ... 다른 필드들
    swapchain_images: Vec<vk::Image>,
    swapchain_format: vk::Format,
    swapchain_extent: vk::Extent2D,
    swapchain_image_views: Vec<vk::ImageView>, // 이 필드를 추가합니다.
}
```

`create_image_views` 함수를 만들고 스왑 체인 생성 직후에 호출하도록 합니다. Rust에서는 보통 `new` 생성자나 초기화 함수 내에서 순서대로 호출합니다.

```rust
impl VulkanApp {
    pub fn new(window: &winit::window::Window) -> Self {
        // ...
        app.create_logical_device();
        app.create_swapchain();
        app.create_image_views(); // 스왑체인 생성 직후 호출
        // ...
    }

    fn create_image_views(&mut self) {
        // 이 함수를 구현합니다.
    }
}
```

이제 `create_image_views` 함수를 구현해 보겠습니다. C++ 버전처럼 벡터의 크기를 미리 조정하는 대신, 각 스왑 체인 이미지를 순회하면서 생성된 이미지 뷰를 새 벡터에 추가하는 방식을 사용하겠습니다.

```rust
fn create_image_views(&mut self) {
    self.swapchain_image_views = self.swapchain_images
        .iter()
        .map(|&image| {
            // 여기에 각 이미지에 대한 이미지 뷰 생성 로직이 들어갑니다.
        })
        .collect();
}
```

`map` 클로저 내부에서 각 이미지에 대한 이미지 뷰를 생성합니다. 이미지 뷰 생성을 위한 파라미터는 `vk::ImageViewCreateInfo` 구조체에 명시됩니다. `ash` 라이브러리는 빌더(builder) 패턴을 제공하여 이 구조체를 더 안전하고 편리하게 생성할 수 있습니다.

```rust
// map 클로저 내부
let components = vk::ComponentMapping {
    r: vk::ComponentSwizzle::IDENTITY,
    g: vk::ComponentSwizzle::IDENTITY,
    b: vk::ComponentSwizzle::IDENTITY,
    a: vk::ComponentSwizzle::IDENTITY,
};
```

`components` 필드는 컬러 채널을 스위즐(swizzle)할 수 있게 해줍니다. 예를 들어, 모든 채널을 빨간색 채널에 매핑하여 단색 텍스처를 만들 수 있습니다. 여기서는 기본 매핑을 사용합니다.

```rust
// map 클로저 내부, components 정의 다음
let subresource_range = vk::ImageSubresourceRange {
    aspect_mask: vk::ImageAspectFlags::COLOR,
    base_mip_level: 0,
    level_count: 1,
    base_array_layer: 0,
    layer_count: 1,
};
```

`subresource_range` 필드는 이미지의 용도와 접근할 이미지의 부분을 기술합니다. 우리의 이미지는 밉매핑 레벨이나 여러 레이어 없이 컬러 타겟으로 사용될 것입니다.
*   `aspect_mask`: 컬러 이미지를 다루므로 `vk::ImageAspectFlags::COLOR`을 사용합니다.
*   `base_mip_level`, `level_count`: 밉매핑을 사용하지 않으므로 기본 레벨 0에, 레벨 수는 1로 설정합니다.
*   `base_array_layer`, `layer_count`: 스테레오스코픽 3D 앱이 아니므로 기본 배열 레이어 0에, 레이어 수는 1로 설정합니다.

이제 이 정보들을 사용하여 `ImageViewCreateInfo`를 빌드하고 `create_image_view` 함수를 호출합니다.

```rust
// map 클로저 내부, subresource_range 정의 다음
let create_info = vk::ImageViewCreateInfo::builder()
    .image(image)
    .view_type(vk::ImageViewType::TYPE_2D)
    .format(self.swapchain_format)
    .components(components)
    .subresource_range(subresource_range);

let image_view = unsafe {
    self.device
        .create_image_view(&create_info, None)
        .expect("Failed to create image view!")
};
image_view // map 클로저의 반환 값
```

`ash`의 생성 함수는 `unsafe` 블록 안에서 호출해야 합니다. 왜냐하면 유효하지 않은 파라미터(예: 해제된 `device`나 `image`)를 전달하면 정의되지 않은 동작을 유발할 수 있기 때문입니다. `ash` 함수는 `Result`를 반환하므로, `expect`를 사용하여 에러 발생 시 프로그램을 중단하고 메시지를 출력할 수 있습니다. `None`은 커스텀 할당자를 사용하지 않음을 의미합니다.

전체 `create_image_views` 함수는 다음과 같습니다.

```rust
fn create_image_views(&mut self) {
    self.swapchain_image_views = self
        .swapchain_images
        .iter()
        .map(|&image| {
            let components = vk::ComponentMapping {
                r: vk::ComponentSwizzle::IDENTITY,
                g: vk::ComponentSwizzle::IDENTITY,
                b: vk::ComponentSwizzle::IDENTITY,
                a: vk::ComponentSwizzle::IDENTITY,
            };

            let subresource_range = vk::ImageSubresourceRange {
                aspect_mask: vk::ImageAspectFlags::COLOR,
                base_mip_level: 0,
                level_count: 1,
                base_array_layer: 0,
                layer_count: 1,
            };

            let create_info = vk::ImageViewCreateInfo::builder()
                .image(image)
                .view_type(vk::ImageViewType::TYPE_2D)
                .format(self.swapchain_format)
                .components(components)
                .subresource_range(subresource_range);

            unsafe {
                self.device
                    .create_image_view(&create_info, None)
                    .expect("Failed to create image view!")
            }
        })
        .collect();
}
```

이미지와 달리, 이미지 뷰는 우리가 명시적으로 생성했으므로, 프로그램이 끝날 때 이를 정리하는 코드를 추가해야 합니다. Rust에서는 보통 `Drop` 트레이트 구현을 통해 리소스 해제를 자동화하지만, 이 튜토리얼의 구조를 따라 `cleanup` 함수에 추가하겠습니다.

```rust
impl VulkanApp {
    pub fn cleanup(&mut self) {
        unsafe {
            for &image_view in self.swapchain_image_views.iter() {
                self.device.destroy_image_view(image_view, None);
            }
            // ... 다른 리소스 정리
        }
    }
}
```

`destroy_image_view` 또한 `unsafe` 함수입니다.

이미지 뷰는 이미지를 텍스처로 사용하기 시작하기에는 충분하지만, 렌더 타겟으로 사용되기에는 아직 완전히 준비되지 않았습니다. 이를 위해서는 프레임버퍼(framebuffer)라고 알려진 한 단계의 간접 과정이 더 필요합니다. 하지만 그 전에 그래픽 파이프라인을 먼저 설정해야 합니다.