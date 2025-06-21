Vulkan에는 '기본 프레임버퍼(default framebuffer)'라는 개념이 없습니다. 따라서 우리가 렌더링할 버퍼를 화면에 시각화하기 전에 소유할 인프라가 필요합니다. 이 인프라를 **스왑 체인(swap chain)** 이라고 하며, Vulkan에서는 명시적으로 생성해야 합니다.

스왑 체인은 본질적으로 화면에 표시되기를 기다리는 이미지들의 큐(queue)입니다. 우리 애플리케이션은 렌더링할 이미지를 이 큐에서 가져와(acquire) 렌더링한 다음, 다시 큐에 반환합니다. 큐가 정확히 어떻게 작동하고 큐에서 이미지를 표시하는 조건은 스왑 체인 설정 방식에 따라 다르지만, 스왑 체인의 일반적인 목적은 이미지 표시를 화면의 주사율(refresh rate)과 동기화하는 것입니다.

## 스왑 체인 지원 확인

모든 그래픽 카드가 이미지를 화면에 직접 표시할 수 있는 것은 아닙니다. 예를 들어 서버용으로 설계되어 디스플레이 출력이 없는 경우가 그렇습니다. 둘째로, 이미지 표시는 창 시스템(window system) 및 창과 관련된 표면(surface)과 밀접하게 연관되어 있으므로 실제 Vulkan 코어의 일부가 아닙니다. 따라서 `VK_KHR_swapchain` 장치 확장 기능의 지원 여부를 쿼리한 후 활성화해야 합니다.

이를 위해 먼저 `is_device_suitable` 함수를 확장하여 이 확장이 지원되는지 확인합니다. `ash` 라이브러리는 `ash::extensions::khr::Swapchain::name()`과 같이 확장 기능의 이름을 안전하게 가져올 수 있는 상수를 제공하여 오타를 방지합니다.

먼저, 필요한 장치 확장 기능 목록을 상수로 정의합니다. `CStr` 타입을 사용하여 C 문자열과의 호환성을 보장합니다.

```rust
use std::ffi::CStr;
// ... 다른 use 구문들

const DEVICE_EXTENSIONS: [&'static CStr; 1] = [ash::extensions::khr::Swapchain::name()];
```

다음으로, `is_device_suitable`에서 추가 검사로 호출될 새로운 함수 `check_device_extension_support`를 만듭니다.

```rust
// is_device_suitable 함수 내부에서...
let extensions_supported =
    check_device_extension_support(&self.instance, physical_device);

// ...

indices.is_complete() && extensions_supported
```

```rust
use std::collections::HashSet;
use ash::Instance;
use ash::vk;

fn check_device_extension_support(
    instance: &Instance,
    physical_device: vk::PhysicalDevice,
) -> bool {
    // 필요한 확장 기능들을 HashSet으로 변환합니다.
    let mut required_extensions = HashSet::from_iter(DEVICE_EXTENSIONS.iter().map(|s| *s));

    // 장치가 지원하는 확장 기능들을 가져옵니다.
    // ash는 C++의 2중 호출 패턴 대신 Result<Vec<...>>를 바로 반환해 편리합니다.
    let available_extensions = unsafe {
        instance
            .enumerate_device_extension_properties(physical_device)
            .expect("Failed to enumerate device extension properties.")
    };

    // 사용 가능한 확장 기능들을 순회하며 필요한 확장 기능 목록에서 제거합니다.
    for extension in available_extensions.iter() {
        // C 스타일의 char 배열을 CStr로 변환합니다.
        let extension_name = unsafe { CStr::from_ptr(extension.extension_name.as_ptr()) };
        required_extensions.remove(extension_name);
    }

    // 모든 필요한 확장 기능이 제거되었다면 지원하는 것입니다.
    required_extensions.is_empty()
}
```

Rust에서는 C++의 `std::set` 대신 `std::collections::HashSet`을 사용했습니다. 로직은 동일합니다. 사용 가능한 확장을 순회하면서 필요한 확장 목록에서 하나씩 제거하고, 최종적으로 목록이 비었는지 확인합니다. 이제 코드를 실행하여 그래픽 카드가 실제로 스왑 체인을 생성할 수 있는지 확인하십시오. 이전 장에서 확인했던 표현 큐(presentation queue)의 가용성은 스왑 체인 확장이 지원되어야 함을 의미하지만, 명시적으로 확인하고 활성화하는 것이 좋습니다.

## 장치 확장 기능 활성화

스왑 체인을 사용하려면 먼저 `VK_KHR_swapchain` 확장을 활성화해야 합니다. 논리 장치를 생성할 때 `ash`의 빌더 패턴을 사용하여 활성화할 수 있습니다.

```rust
// CStr 슬라이스에서 C-호환 포인터 슬라이스를 만듭니다.
let extension_names_raw: Vec<*const i8> = DEVICE_EXTENSIONS
    .iter()
    .map(|s| s.as_ptr())
    .collect();

let device_create_info = vk::DeviceCreateInfo::builder()
    .queue_create_infos(&queue_create_infos)
    .enabled_features(&device_features)
    .enabled_extension_names(&extension_names_raw); // 여기서 확장 기능을 지정합니다.
```

## 스왑 체인 지원 상세 정보 쿼리

스왑 체인이 사용 가능한지 확인하는 것만으로는 충분하지 않습니다. 스왑 체인이 우리 창 표면(window surface)과 호환되지 않을 수 있기 때문입니다. 이제 스왑 체인 생성에 필요한 세부 정보들을 쿼리해야 합니다.

*   기본 표면 기능 (스왑 체인의 최소/최대 이미지 수, 이미지의 최소/최대 너비 및 높이)
*   표면 형식 (픽셀 형식, 색 공간)
*   사용 가능한 표현 모드

이 정보들을 담을 구조체를 정의합니다. C++ 버전과 유사하지만 Rust 스타일을 따릅니다.

```rust
struct SwapChainSupportDetails {
    capabilities: vk::SurfaceCapabilitiesKHR,
    formats: Vec<vk::SurfaceFormatKHR>,
    present_modes: Vec<vk::PresentModeKHR>,
}
```

이 구조체를 채울 `query_swapchain_support` 함수를 만듭니다. `ash`에서는 `Surface` 로더를 사용해야 합니다.

```rust
// 주 애플리케이션 구조체에 surface_loader 필드가 있어야 합니다.
// self.surface_loader: ash::extensions::khr::Surface

fn query_swapchain_support(
    physical_device: vk::PhysicalDevice,
    surface_loader: &ash::extensions::khr::Surface,
    surface: vk::SurfaceKHR,
) -> SwapChainSupportDetails {
    unsafe {
        // capabilities 쿼리
        let capabilities = surface_loader
            .get_physical_device_surface_capabilities(physical_device, surface)
            .expect("Failed to query for surface capabilities.");

        // formats 쿼리
        let formats = surface_loader
            .get_physical_device_surface_formats(physical_device, surface)
            .expect("Failed to query for surface formats.");

        // present_modes 쿼리
        let present_modes = surface_loader
            .get_physical_device_surface_present_modes(physical_device, surface)
            .expect("Failed to query for surface present modes.");

        SwapChainSupportDetails {
            capabilities,
            formats,
            present_modes,
        }
    }
}
```

이제 `is_device_suitable` 함수를 다시 수정하여, 확장 기능 지원이 확인된 후에 스왑 체인 지원이 충분한지 확인합니다. 형식이 하나 이상, 표현 모드가 하나 이상이면 충분하다고 간주합니다.

```rust
// is_device_suitable 함수 내부에서...
if extensions_supported {
    let swapchain_support = query_swapchain_support(physical_device, &self.surface_loader, self.surface);
    let swapchain_adequate = !swapchain_support.formats.is_empty()
        && !swapchain_support.present_modes.is_empty();
    
    indices.is_complete() && swapchain_adequate
} else {
    false
}
```

## 스왑 체인에 적합한 설정 선택하기

`swapchain_adequate` 조건이 충족되었다면, 이제 사용 가능한 옵션 중에서 최적의 설정을 선택해야 합니다.

### 표면 형식

가장 이상적인 형식은 `B8G8R8A8_SRGB` 형식과 `SRGB_NONLINEAR` 색 공간의 조합입니다. 이 조합을 우선적으로 찾고, 없다면 첫 번째 사용 가능한 형식을 선택합니다.

```rust
fn choose_swap_surface_format(
    available_formats: &[vk::SurfaceFormatKHR],
) -> vk::SurfaceFormatKHR {
    available_formats
        .iter()
        .find(|format| {
            format.format == vk::Format::B8G8R8A8_SRGB
                && format.color_space == vk::ColorSpaceKHR::SRGB_NONLINEAR
        })
        .map(|format| *format) // find는 &T를 반환하므로 복사
        .unwrap_or(available_formats[0])
}
```
Rust의 이터레이터 메서드(`find`, `map`)를 사용하면 더 간결하고 표현력 있는 코드를 작성할 수 있습니다.

### 표현 모드

표현 모드는 스왑 체인의 동작을 결정하는 가장 중요한 설정입니다. 네 가지 모드가 있습니다:

*   `IMMEDIATE`: 즉시 표시 (티어링 가능성 있음)
*   `FIFO`: 수직 동기화와 유사한 큐 방식 (보장된 가용성)
*   `FIFO_RELAXED`: `FIFO`의 변형으로, 큐가 비었을 때 지연 없이 표시 (티어링 가능성 있음)
*   `MAILBOX`: 삼중 버퍼링과 유사. 큐가 가득 차면 새 이미지로 교체 (낮은 지연 시간, 티어링 없음)

`MAILBOX` 모드가 성능과 품질 면에서 훌륭한 절충안이므로 우선적으로 선택하고, 없다면 보장된 `FIFO` 모드를 사용합니다.

```rust
fn choose_swap_present_mode(
    available_present_modes: &[vk::PresentModeKHR],
) -> vk::PresentModeKHR {
    available_present_modes
        .iter()
        .find(|&&mode| mode == vk::PresentModeKHR::MAILBOX)
        .map(|mode| *mode)
        .unwrap_or(vk::PresentModeKHR::FIFO)
}
```

### 스왑 범위

스왑 범위는 스왑 체인 이미지의 해상도이며, 보통 창의 픽셀 단위 해상도와 같습니다.

창 관리자가 해상도를 정해주는 경우(`current_extent`의 너비가 `u32::MAX`가 아닌 경우), 그 값을 그대로 사용합니다. 그렇지 않으면, `winit` (또는 사용하는 창 라이브러리)에서 프레임버퍼의 픽셀 크기를 직접 쿼리하여 사용해야 합니다. 고해상도(HiDPI) 디스플레이에서는 창의 논리적 크기와 픽셀 크기가 다를 수 있기 때문입니다.

```rust
use winit::window::Window;

fn choose_swap_extent(
    capabilities: &vk::SurfaceCapabilitiesKHR,
    window: &Window,
) -> vk::Extent2D {
    if capabilities.current_extent.width != u32::MAX {
        capabilities.current_extent
    } else {
        let framebuffer_size = window.inner_size(); // winit 0.28+ 에서는 inner_size()가 픽셀 단위

        let mut actual_extent = vk::Extent2D {
            width: framebuffer_size.width,
            height: framebuffer_size.height,
        };

        // 해상도를 Vulkan 구현체가 지원하는 최소/최대 범위 내로 클램핑합니다.
        actual_extent.width = actual_extent.width.clamp(
            capabilities.min_image_extent.width,
            capabilities.max_image_extent.width,
        );
        actual_extent.height = actual_extent.height.clamp(
            capabilities.min_image_extent.height,
            capabilities.max_image_extent.height,
        );

        actual_extent
    }
}
```

## 스왑 체인 생성

이제 모든 준비가 끝났습니다. 지금까지 만든 헬퍼 함수들을 사용하여 스왑 체인을 생성해 봅시다. 주 애플리케이션 구조체에 스왑 체인 관련 필드들을 추가해야 합니다.

```rust
struct VulkanApp {
    // ... 기존 필드들
    swapchain_loader: ash::extensions::khr::Swapchain,
    swapchain: vk::SwapchainKHR,
    swapchain_images: Vec<vk::Image>,
    swapchain_format: vk::Format,
    swapchain_extent: vk::Extent2D,
}
```
`create_swapchain` 함수를 구현합니다.

```rust
// create_logical_device 이후에 호출
fn create_swapchain(&mut self, window: &Window) {
    let swapchain_support = query_swapchain_support(self.physical_device, &self.surface_loader, self.surface);

    let surface_format = choose_swap_surface_format(&swapchain_support.formats);
    let present_mode = choose_swap_present_mode(&swapchain_support.present_modes);
    let extent = choose_swap_extent(&swapchain_support.capabilities, window);

    // 스왑 체인에 포함될 이미지 개수를 결정합니다.
    // 최소값보다 하나 더 요청하는 것이 일반적입니다.
    let mut image_count = swapchain_support.capabilities.min_image_count + 1;
    if swapchain_support.capabilities.max_image_count > 0 {
        image_count = image_count.min(swapchain_support.capabilities.max_image_count);
    }

    let indices = find_queue_families(&self.instance, self.physical_device, &self.surface_loader, self.surface);
    let queue_family_indices = [
        indices.graphics_family.unwrap(),
        indices.present_family.unwrap(),
    ];

    let (image_sharing_mode, queue_family_indices_slice) =
        if indices.graphics_family != indices.present_family {
            (vk::SharingMode::CONCURRENT, &queue_family_indices[..])
        } else {
            (vk::SharingMode::EXCLUSIVE, &[] as &[u32])
        };

    let create_info = vk::SwapchainCreateInfoKHR::builder()
        .surface(self.surface)
        .min_image_count(image_count)
        .image_format(surface_format.format)
        .image_color_space(surface_format.color_space)
        .image_extent(extent)
        .image_array_layers(1)
        .image_usage(vk::ImageUsageFlags::COLOR_ATTACHMENT)
        .image_sharing_mode(image_sharing_mode)
        .queue_family_indices(queue_family_indices_slice)
        .pre_transform(swapchain_support.capabilities.current_transform)
        .composite_alpha(vk::CompositeAlphaFlagsKHR::OPAQUE)
        .present_mode(present_mode)
        .clipped(true)
        .old_swapchain(vk::SwapchainKHR::null()); // 창 크기 변경 시 필요

    // ash에서는 Swapchain 확장 기능 로더가 필요합니다.
    let swapchain_loader = ash::extensions::khr::Swapchain::new(&self.instance, &self.device);
    let swapchain = unsafe {
        swapchain_loader
            .create_swapchain(&create_info, None)
            .expect("Failed to create Swapchain!")
    };

    // 스왑 체인 이미지 핸들을 가져옵니다.
    let swapchain_images = unsafe {
        swapchain_loader
            .get_swapchain_images(swapchain)
            .expect("Failed to get Swapchain Images.")
    };
    
    self.swapchain_loader = swapchain_loader;
    self.swapchain = swapchain;
    self.swapchain_images = swapchain_images;
    self.swapchain_format = surface_format.format;
    self.swapchain_extent = extent;
}
```
`ash`에서는 `Swapchain` 확장 함수를 호출하기 위해 `ash::extensions::khr::Swapchain` 로더를 생성해야 합니다. 이 로더는 인스턴스와 논리 장치를 기반으로 만들어지며, 애플리케이션의 상태로 저장되어야 나중에 스왑 체인을 파괴할 때 사용할 수 있습니다.

애플리케이션이 종료될 때 스왑 체인을 정리하기 위해 `Drop` 트레잇을 구현합니다.

```rust
impl Drop for VulkanApp {
    fn drop(&mut self) {
        unsafe {
            self.swapchain_loader.destroy_swapchain(self.swapchain, None);
            self.device.destroy_device(None);
            self.surface_loader.destroy_surface(self.surface, None);
            // ... 나머지 리소스 정리
        }
    }
}
```

## 스왑 체인 이미지 가져오기

`ash`를 사용하면 스왑 체인 이미지를 매우 간단하게 가져올 수 있습니다. `get_swapchain_images` 함수는 이미지 핸들이 담긴 `Vec<vk::Image>`를 바로 반환합니다. 위 `create_swapchain` 함수에서 이미 이 과정을 포함시켰습니다.

```rust
let swapchain_images = unsafe {
    swapchain_loader
        .get_swapchain_images(swapchain)
        .expect("Failed to get Swapchain Images.")
};
```
이미지들은 스왑 체인에 의해 소유되므로, 스왑 체인이 파괴될 때 자동으로 정리됩니다. 별도로 이미지를 파괴할 필요는 없습니다.

이제 우리는 렌더링하고 창에 표시할 수 있는 이미지 집합을 갖게 되었습니다. 다음 장에서는 이 이미지들을 렌더 타겟으로 설정하고, 실제 그래픽 파이프라인과 그리기 명령에 대해 알아보기 시작하겠습니다!

[Rust 코드 예시](https://github.com/ash-rs/ash/blob/master/examples/triangle.rs) (전체적인 구조 참고용)