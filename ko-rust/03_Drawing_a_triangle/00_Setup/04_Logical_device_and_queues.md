## 서론

사용할 물리 장치를 선택한 후에는, 이와 상호작용하기 위한 *논리 장치*를 설정해야 합니다. 논리 장치 생성 과정은 인스턴스 생성 과정과 유사하며, 우리가 사용하고자 하는 기능들을 기술합니다. 또한, 어떤 큐 패밀리를 사용할 수 있는지 질의했으므로 이제 어떤 큐를 생성할지 명시해야 합니다. 요구 사항이 다양하다면 동일한 물리 장치에서 여러 개의 논리 장치를 생성할 수도 있습니다.

먼저 애플리케이션 구조체에 논리 장치를 저장할 새 필드를 추가합니다. `ash::Device`는 논리 장치를 나타내는 타입입니다.

```rust
use ash::{Device, vk};

struct HelloTriangleApplication {
    // ... 기존 필드들 ...
    physical_device: vk::PhysicalDevice,
    device: Device,
}
```

다음으로, `init_vulkan` 내에서 `create_logical_device` 함수를 호출하도록 구성합니다.

```rust
impl HelloTriangleApplication {
    pub fn new(window: &Window) -> Self {
        let mut app = // ... 초기화 ...
        app.init_vulkan();
        app
    }

    fn init_vulkan(&mut self) {
        self.create_instance();
        self.setup_debug_messenger();
        self.pick_physical_device();
        self.create_logical_device();
    }

    fn create_logical_device(&mut self) {
        // ... 구현 ...
    }
}
```

## 생성할 큐 명시하기

논리 장치 생성은 여러 구조체를 채우는 것으로 시작합니다. 그 첫 번째는 `vk::DeviceQueueCreateInfo`입니다. 이 구조체는 단일 큐 패밀리에 대해 우리가 원하는 큐의 개수를 기술합니다. 지금은 그래픽스 기능이 있는 큐에만 관심이 있습니다.

```rust
// create_logical_device 메서드 내부
let indices = self.find_queue_families(self.physical_device);

let queue_create_info = vk::DeviceQueueCreateInfo::builder()
    .queue_family_index(indices.graphics_family.unwrap())
    .queue_priorities(&[1.0]) // 큐가 하나뿐이라도 우선순위는 필수
    .build();
```

Rust와 `ash`에서는 빌더 패턴을 사용하여 구조체를 더 안전하고 명확하게 생성할 수 있습니다. C++의 `std::optional::value()`는 Rust의 `Option::unwrap()`에 해당합니다. 여기서는 큐 패밀리를 이미 찾았다고 확신할 수 있으므로 `unwrap()`을 사용합니다.

사용 가능한 최신 드라이버들은 각 큐 패밀리마다 소수의 큐만 생성하도록 허용하며, 실제로 하나보다 더 많이 필요한 경우는 드뭅니다. 여러 스레드에서 모든 커맨드 버퍼를 생성한 뒤, 메인 스레드에서 단 한 번의 저비용 호출로 모두 제출할 수 있기 때문입니다.

Vulkan은 `0.0`에서 `1.0` 사이의 부동 소수점 값을 사용하여 큐에 우선순위를 할당하고 커맨드 버퍼 실행 스케줄링에 영향을 줄 수 있습니다. 큐가 하나만 있어도 이 설정은 필수입니다. 위 코드에서는 `&[1.0]` 슬라이스를 전달하여 이를 설정했습니다.

## 사용할 장치 기능 명시하기

다음 정보는 우리가 사용할 장치 기능의 집합입니다. 이는 이전 장에서 `vkGetPhysicalDeviceFeatures`로 지원 여부를 확인했던 지오메트리 셰이더 같은 기능들입니다. 지금 당장은 특별한 기능이 필요 없으므로, 모든 필드가 기본값(`false`)으로 설정된 구조체를 생성합니다.

```rust
let device_features = vk::PhysicalDeviceFeatures::builder().build();
```

더 흥미로운 Vulkan 기능을 사용하기 시작할 때 이 구조체로 다시 돌아올 것입니다.

## 논리 장치 생성하기

이제 `vk::DeviceCreateInfo` 구조체를 채울 준비가 되었습니다.

```rust
// C-호환 문자열을 위한 준비
use std::ffi::{CStr, CString};
use std::os::raw::c_char;

// 필요한 장치 확장 (스왑체인)
let device_extensions = [
    ash::extensions::khr::Swapchain::name().as_ptr(),
];

// 유효성 검사 레이어 설정 (C++ 버전과 동일한 로직)
let validation_layer_names: Vec<CString> = VALIDATION_LAYERS
    .iter()
    .map(|&s| CString::new(s).unwrap())
    .collect();
let validation_layer_name_ptrs: Vec<*const c_char> = validation_layer_names
    .iter()
    .map(|s| s.as_ptr())
    .collect();

let mut create_info = vk::DeviceCreateInfo::builder()
    .queue_create_infos(std::slice::from_ref(&queue_create_info))
    .enabled_features(&device_features)
    .enabled_extension_names(&device_extensions);

if ENABLE_VALIDATION_LAYERS {
    create_info = create_info
        .enabled_layer_names(&validation_layer_name_ptrs);
}
```

큐 생성 정보와 장치 기능 구조체에 대한 참조를 빌더에 전달합니다. `queue_create_infos`는 슬라이스를 받으므로, `std::slice::from_ref`를 사용하여 단일 항목으로부터 슬라이스를 만듭니다.

나머지 정보는 `vk::InstanceCreateInfo`와 유사하며 확장과 유효성 검사 레이어를 지정합니다. 차이점은 이번에는 장치에 한정된다는 점입니다.

장치별 확장의 예로 `VK_KHR_swapchain`이 있습니다. 이 확장은 렌더링된 이미지를 창에 표시하는 데 필수적입니다. 이 튜토리얼에서는 나중에 다루지만, 지금 활성화해 두는 것이 일반적입니다.

이전 Vulkan 구현은 인스턴스와 장치별 유효성 검사 레이어를 구분했지만 [더 이상은 그렇지 않습니다](https://www.khronos.org/registry/vulkan/specs/1.3-extensions/html/chap40.html#extendingvulkan-layers-devicelayerdeprecation). 즉, 최신 구현에서는 `vk::DeviceCreateInfo`의 `enabled_layer_names` 필드가 무시됩니다. 하지만 구버전 구현과의 호환성을 위해 설정해 두는 것이 좋습니다.

이제 `create_device` 함수를 호출하여 논리 장치를 인스턴스화할 수 있습니다. 이 함수는 FFI(Foreign Function Interface) 호출이므로 `unsafe` 블록 안에서 호출해야 합니다.

```rust
let device = unsafe {
    self.instance
        .create_device(self.physical_device, &create_info, None)
        .expect("Failed to create logical device!")
};
self.device = Some(device); // Option<Device> 에 저장하거나 직접 할당
```

`ash`에서는 `create_device`가 `Instance`의 메서드입니다. 파라미터는 물리 장치, 생성 정보, 그리고 선택적인 할당 콜백입니다. `expect`를 사용하여 오류 발생 시 프로그램을 패닉시킵니다.

장치는 애플리케이션이 종료될 때 소멸되어야 합니다. Rust에서는 `Drop` 트레잇을 구현하여 이 작업을 자동화하는 것이 관례입니다.

```rust
impl Drop for HelloTriangleApplication {
    fn drop(&mut self) {
        unsafe {
            // ... 다른 리소스 정리 ...
            self.device.destroy_device(None);
            // ... 인스턴스 정리 ...
        }
    }
}
```
논리 장치는 인스턴스와 직접 상호작용하지 않으므로 생성 시 인스턴스가 파라미터로 필요하지 않습니다.

## 큐 핸들 가져오기

큐는 논리 장치와 함께 자동으로 생성되지만, 상호작용할 핸들이 아직 없습니다. 구조체에 그래픽스 큐 핸들을 저장할 필드를 추가합니다.

```rust
struct HelloTriangleApplication {
    // ...
    device: Device,
    graphics_queue: vk::Queue,
}
```

장치 큐는 장치가 파괴될 때 암시적으로 정리되므로, `Drop` 트레잇에서 별도로 정리할 필요가 없습니다.

`get_device_queue` 함수를 사용하여 큐 핸들을 가져올 수 있습니다. 파라미터는 큐 패밀리 인덱스와 큐 인덱스입니다. 우리는 이 패밀리에서 큐를 하나만 생성했으므로 인덱스 `0`을 사용합니다.

```rust
// create_logical_device 메서드 마지막 부분
let graphics_queue = unsafe { self.device.get_device_queue(indices.graphics_family.unwrap(), 0) };
self.graphics_queue = graphics_queue;
```

이제 논리 장치와 큐 핸들을 사용해 그래픽 카드로 작업을 시작할 준비가 되었습니다! 다음 장들에서는 결과를 창 시스템에 표시하기 위한 리소스를 설정할 것입니다.

[Rust 코드 예시](https://github.com/bwasty/vulkan-tutorial-rs/blob/master/src/04_logical_device.rs)