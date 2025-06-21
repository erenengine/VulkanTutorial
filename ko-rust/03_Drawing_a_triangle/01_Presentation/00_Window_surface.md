Vulkan은 플랫폼에 구애받지 않는 API이므로, 자체적으로 윈도우 시스템과 직접 통신할 수 없습니다. Vulkan과 윈도우 시스템을 연결하여 렌더링 결과를 화면에 표시하려면, WSI(Window System Integration) 확장을 사용해야 합니다. 이 장에서는 그 첫 번째 확장인 `VK_KHR_surface`에 대해 논의합니다. 이 확장은 렌더링된 이미지를 표시하기 위한 추상적인 표면 타입을 나타내는 `vk::SurfaceKHR` 객체를 제공합니다. 우리 프로그램의 서피스는 `winit`으로 이미 열어 둔 창을 기반으로 합니다.

`VK_KHR_surface` 확장은 인스턴스 수준 확장(instance-level extension)이며, `ash_window::enumerate_required_extensions`가 반환하는 목록에 포함되어 있으므로 이미 활성화했습니다. 이 목록에는 다음 장들에서 사용할 다른 WSI 확장들도 포함됩니다.

윈도우 서피스는 물리 장치 선택에 영향을 줄 수 있으므로, 인스턴스 생성 직후에 만들어야 합니다. 이 과정을 뒤로 미룬 이유는 윈도우 서피스가 렌더 타겟 및 프레젠테이션이라는 더 큰 주제의 일부이며, 이를 기본 설정 과정에서 다루면 내용이 복잡해지기 때문입니다. 또한, 오프스크린 렌더링(off-screen rendering)만 필요하다면 윈도우 서피스는 전적으로 선택 사항입니다. Vulkan은 OpenGL처럼 보이지 않는 창을 만드는 꼼수 없이도 오프스크린 렌더링을 허용합니다.

## 윈도우 서피스 생성

먼저 메인 애플리케이션 구조체에 `surface_loader`와 `surface` 필드를 추가합니다. `ash`에서는 확장 기능 함수들을 사용하기 위해 해당 확장 로더(loader)가 필요합니다.

```rust
use ash::extensions::khr;

struct VulkanApp {
    // ... 기존 필드들
    surface_loader: khr::Surface,
    surface: vk::SurfaceKHR,
    // ...
}
```

`vk::SurfaceKHR` 객체와 그 사용법은 플랫폼에 독립적이지만, 생성 과정은 윈도우 시스템 세부 사항에 따라 달라지므로 플랫폼 종속적입니다. 예를 들어, Windows에서는 `HWND`와 `HMODULE` 핸들이 필요합니다. 따라서 플랫폼별 추가 확장이 있으며, Windows용으로는 `VK_KHR_win32_surface`가 있습니다.

이 플랫폼별 확장을 사용하여 Windows에서 서피스를 만드는 방법을 보여드리겠지만, 이 튜토리얼에서 실제로 사용하지는 않을 것입니다. `winit`과 같은 라이브러리를 사용하면서 플랫폼 종속 코드를 직접 사용하는 것은 바람직하지 않습니다. 다행히 `ash-window` 크레이트가 플랫폼 간 차이를 추상화해줍니다. 그럼에도 불구하고, 라이브러리에 의존하기 전에 내부적으로 어떤 일이 일어나는지 살펴보는 것은 좋습니다.

네이티브 플랫폼 함수에 접근하려면 `raw-window-handle` 크레이트가 필요합니다.

```rust
use raw_window_handle::{HasRawWindowHandle, RawWindowHandle};
use ash::extensions::khr::Win32Surface;

// 이 코드는 시연용이며 실제로는 사용하지 않습니다.
fn create_surface_natively(
    entry: &ash::Entry,
    instance: &ash::Instance,
    window: &winit::window::Window,
) -> vk::SurfaceKHR {
    let handle = window.raw_window_handle();
    if let RawWindowHandle::Win32(win_handle) = handle {
        let surface_info = vk::Win32SurfaceCreateInfoKHR::builder()
            .hinstance(win_handle.hinstance)
            .hwnd(win_handle.hwnd);

        let win32_surface_loader = Win32Surface::new(entry, instance);
        unsafe {
            win32_surface_loader
                .create_win32_surface(&surface_info, None)
                .expect("Failed to create Win32 surface.")
        }
    } else {
        panic!("Unsupported window handle type");
    }
}
```

`ash-window` 크레이트의 `create_surface` 함수는 각 플랫폼에 맞춰 정확히 이 작업을 수행합니다. 이제 이 함수를 우리 프로그램에 통합해 보겠습니다. `init_vulkan` 함수에서 인스턴스 생성과 디버그 메신저 설정 직후에 호출될 `create_surface` 메서드를 추가합니다.

```rust
impl VulkanApp {
    pub fn init_vulkan(&mut self, window: &winit::window::Window) {
        self.create_instance(window);
        self.setup_debug_messenger();
        self.create_surface(window);
        self.pick_physical_device();
        self.create_logical_device();
    }

    fn create_surface(&mut self, window: &winit::window::Window) {
        // 서피스 확장 로더 생성
        self.surface_loader = khr::Surface::new(&self.entry, &self.instance);

        // 서피스 생성
        self.surface = unsafe {
            ash_window::create_surface(&self.entry, &self.instance, window, None)
                .expect("Failed to create window surface")
        };
    }
}
```

`ash_window::create_surface`는 `ash::Entry`, `ash::Instance`, `winit` 윈도우 객체, 그리고 커스텀 할당자를 인자로 받습니다. 이 함수는 내부적으로 플랫폼에 맞는 함수를 호출하고 그 결과를 반환합니다.

서피스를 정리할 때는 원본 Vulkan API를 사용합니다. 서피스는 반드시 인스턴스보다 먼저 파괴되어야 합니다.

```rust
impl Drop for VulkanApp {
    fn drop(&mut self) {
        unsafe {
            // ... 다른 객체들 정리
            self.surface_loader.destroy_surface(self.surface, None);
            self.instance.destroy_instance(None);
        }
    }
}
```

## 프레젠테이션 지원 여부 쿼리

Vulkan 구현이 윈도우 시스템 통합을 지원하더라도, 시스템의 모든 장치가 이를 지원하는 것은 아닙니다. 따라서 `is_device_suitable`을 확장하여 장치가 우리가 생성한 서피스로 이미지를 출력(present)할 수 있는지 확인해야 합니다. 프레젠테이션은 큐에 특화된 기능이므로, 이 문제는 결국 우리가 만든 서피스로의 프레젠테이션을 지원하는 큐 패밀리를 찾는 문제가 됩니다.

드로잉 커맨드를 지원하는 큐 패밀리와 프레젠테이션을 지원하는 큐 패밀리가 다를 수 있습니다. 따라서 별도의 프레젠테이션 큐가 존재할 수 있다는 점을 고려하여 `QueueFamilyIndices` 구조체를 수정합니다.

```rust
#[derive(Default)]
struct QueueFamilyIndices {
    graphics_family: Option<u32>,
    present_family: Option<u32>,
}

impl QueueFamilyIndices {
    fn is_complete(&self) -> bool {
        self.graphics_family.is_some() && self.present_family.is_some()
    }
}
```

다음으로, `find_queue_families` 함수를 수정하여 우리 윈도우 서피스로 프레젠테이션할 수 있는 큐 패밀리를 찾도록 합니다. 이를 확인하는 함수는 `get_physical_device_surface_support`이며, `surface_loader`를 통해 호출합니다.

```rust
// find_queue_families 메서드 내부의 루프
for (i, queue_family) in queue_families.iter().enumerate() {
    let i = i as u32;

    if queue_family.queue_flags.contains(vk::QueueFlags::GRAPHICS) {
        indices.graphics_family = Some(i);
    }

    let present_support = unsafe {
        self.surface_loader
            .get_physical_device_surface_support(device, i, self.surface)
            .unwrap_or(false)
    };

    if present_support {
        indices.present_family = Some(i);
    }

    if indices.is_complete() {
        break;
    }
}
```

두 큐 패밀리가 결국 동일한 큐 패밀리로 결정될 가능성이 높지만, 프로그램 전반에 걸쳐 일관된 접근을 위해 별개의 큐인 것처럼 다룰 것입니다. 물론, 성능 향상을 위해 드로잉과 프레젠테이션을 동일한 큐에서 지원하는 물리 장치를 선호하도록 로직을 추가할 수도 있습니다.

## 프레젠테이션 큐 생성하기

이제 남은 일은 논리 장치 생성 절차를 수정하여 프레젠테이션 큐를 만들고 `vk::Queue` 핸들을 가져오는 것입니다. 구조체에 핸들을 저장할 필드를 추가합니다.

```rust
struct VulkanApp {
    // ...
    present_queue: vk::Queue,
    // ...
}
```

다음으로, 두 큐 패밀리로부터 큐를 생성하기 위해 여러 개의 `vk::DeviceQueueCreateInfo` 구조체가 필요할 수 있습니다. 이를 Rust답게 처리하는 방법은 `HashSet`을 사용하여 필요한 큐 패밀리의 고유한 인덱스 집합을 만드는 것입니다.

```rust
use std::collections::HashSet;

// create_logical_device 메서드 내부
let indices = self.find_queue_families(self.physical_device);

let mut unique_queue_families = HashSet::new();
unique_queue_families.insert(indices.graphics_family.unwrap());
unique_queue_families.insert(indices.present_family.unwrap());

let queue_priority = [1.0];
let queue_create_infos: Vec<_> = unique_queue_families
    .iter()
    .map(|&queue_family_index| {
        vk::DeviceQueueCreateInfo::builder()
            .queue_family_index(queue_family_index)
            .queue_priorities(&queue_priority)
            .build()
    })
    .collect();
```

그리고 `vk::DeviceCreateInfo`가 이 `Vec`을 가리키도록 수정합니다.

```rust
let device_create_info = vk::DeviceCreateInfo::builder()
    .queue_create_infos(&queue_create_infos)
    // ... 다른 빌더 호출들
    .build();
```

만약 큐 패밀리가 같다면, `HashSet` 덕분에 해당 인덱스에 대한 정보는 한 번만 전달됩니다. 마지막으로, 프레젠테이션 큐 핸들을 가져오는 호출을 추가합니다.

```rust
// 논리 장치 생성 후
self.graphics_queue = unsafe { self.device.get_device_queue(indices.graphics_family.unwrap(), 0) };
self.present_queue = unsafe { self.device.get_device_queue(indices.present_family.unwrap(), 0) };
```

만약 두 큐 패밀리가 같다면, `graphics_queue`와 `present_queue` 핸들은 대부분 같은 값을 갖게 됩니다. 다음 장에서는 스왑 체인(swap chain)을 살펴보고, 이를 통해 어떻게 서피스에 이미지를 출력하는지 알아보겠습니다.