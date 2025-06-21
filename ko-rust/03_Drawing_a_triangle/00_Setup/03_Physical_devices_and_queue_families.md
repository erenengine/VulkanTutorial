## 물리 디바이스 선택하기

`Instance`를 통해 벌칸 라이브러리를 초기화한 후에는, 시스템에서 우리가 필요로 하는 기능을 지원하는 그래픽 카드를 찾아 선택해야 합니다. 사실 여러 개의 그래픽 카드를 선택하여 동시에 사용할 수도 있지만, 이 튜토리얼에서는 우리에게 필요한 첫 번째 그래픽 카드만 사용하겠습니다.

`pick_physical_device` 함수를 추가하고 `init_vulkan` 함수에서 호출하도록 하겠습니다. Rust에서는 메서드를 `impl` 블록 안에 정의합니다.

```rust
impl HelloTriangleApplication {
    pub fn init_vulkan(&mut self) {
        self.create_instance();
        self.setup_debug_messenger();
        self.pick_physical_device();
    }

    fn pick_physical_device(&mut self) {
        // ...
    }
}
```

최종적으로 선택할 그래픽 카드는 `vk::PhysicalDevice` 핸들에 저장되며, 이 핸들을 구조체의 새 필드로 추가합니다. 이 객체는 `Instance`가 소멸될 때 암시적으로 함께 소멸되므로, `drop` 트레이트에서 따로 처리할 필요는 없습니다.

`vk::PhysicalDevice::null()`은 C++의 `VK_NULL_HANDLE`에 해당하는 값입니다.

```rust
struct HelloTriangleApplication {
    // ...
    physical_device: vk::PhysicalDevice,
}

impl HelloTriangleApplication {
    pub fn new() -> Self {
        // ...
        Self {
            // ...
            physical_device: vk::PhysicalDevice::null(),
        }
    }
}
```

그래픽 카드를 나열하는 것은 `ash`를 사용하면 매우 간단합니다. `Instance`의 `enumerate_physical_devices` 메서드는 사용 가능한 모든 물리 디바이스의 `Vec`을 반환합니다.

```rust
fn pick_physical_device(&mut self) {
    let devices = unsafe {
        self.instance
            .enumerate_physical_devices()
            .expect("물리 디바이스를 찾는데 실패했습니다!")
    };
}
```

만약 벌칸을 지원하는 디바이스가 하나도 없다면 벡터는 비어있을 것입니다.

```rust
if devices.is_empty() {
    panic!("벌칸을 지원하는 GPU를 찾지 못했습니다!");
}
```

이제 각 디바이스를 평가하여 우리가 수행하려는 작업에 적합한지 확인해야 합니다. 모든 그래픽 카드가 동일하게 만들어지지는 않기 때문입니다. 이를 위해 새로운 헬퍼(helper) 메서드를 도입하겠습니다.

```rust
fn is_device_suitable(&self, device: vk::PhysicalDevice) -> bool {
    true
}
```

그리고 물리 디바이스 중 어느 것이든 이 메서드의 요구사항을 충족하는지 확인할 것입니다.

```rust
for &device in &devices {
    if self.is_device_suitable(device) {
        self.physical_device = device;
        break;
    }
}

if self.physical_device == vk::PhysicalDevice::null() {
    panic!("적합한 GPU를 찾지 못했습니다!");
}
```

Rust의 이터레이터(iterator)를 사용하면 더 관용적으로 작성할 수 있습니다.

```rust
let device = devices.into_iter().find(|&p_device| self.is_device_suitable(p_device));

match device {
    Some(p_device) => self.physical_device = p_device,
    None => panic!("적합한 GPU를 찾지 못했습니다!"),
}
```

이 튜토리얼에서는 이해하기 쉽도록 간단한 `for` 루프를 사용하겠습니다.

## 기본적인 디바이스 적합성 검사

디바이스의 적합성을 평가하기 위해 몇 가지 세부 정보를 쿼리하는 것으로 시작할 수 있습니다. `ash`에서는 디바이스 속성을 가져오는 것이 더 간단합니다.

```rust
fn is_device_suitable(&self, device: vk::PhysicalDevice) -> bool {
    let device_properties = unsafe { self.instance.get_physical_device_properties(device) };
    let device_features = unsafe { self.instance.get_physical_device_features(device) };
    // ...
}
```

예를 들어, 우리 애플리케이션이 지오메트리 셰이더를 지원하는 외장 그래픽 카드에서만 사용 가능하다고 가정해 봅시다. 그렇다면 `is_device_suitable` 함수는 다음과 같이 보일 것입니다.

```rust
return device_properties.device_type == vk::PhysicalDeviceType::DISCRETE_GPU
    && device_features.geometry_shader == vk::TRUE;
```

디바이스가 적합한지 아닌지만 확인하고 첫 번째 것을 사용하는 대신, 각 디바이스에 점수를 매겨 가장 높은 점수를 받은 디바이스를 선택할 수도 있습니다. 이런 방식을 사용하면 외장 그래픽 카드에 더 높은 점수를 주어 선호하되, 사용 가능한 유일한 GPU가 내장 GPU일 경우 차선책으로 선택할 수 있습니다.

```rust
use std::collections::BTreeMap;

// ...

fn pick_physical_device(&mut self) {
    // ...

    // 후보들을 점수 기준으로 정렬하기 위해 BTreeMap 사용
    let mut candidates = BTreeMap::new();

    for &device in &devices {
        let score = self.rate_device_suitability(device);
        candidates.insert(score, device);
    }

    // 가장 높은 점수를 받은 후보가 적합한지 확인
    if let Some((score, device)) = candidates.iter().rev().next() {
        if *score > 0 {
            self.physical_device = *device;
        } else {
            panic!("적합한 GPU를 찾지 못했습니다!");
        }
    } else {
        panic!("적합한 GPU를 찾지 못했습니다!");
    }
}

fn rate_device_suitability(&self, device: vk::PhysicalDevice) -> u32 {
    let device_properties = unsafe { self.instance.get_physical_device_properties(device) };
    let device_features = unsafe { self.instance.get_physical_device_features(device) };

    let mut score = 0;

    // 외장 GPU는 상당한 성능 이점을 가짐
    if device_properties.device_type == vk::PhysicalDeviceType::DISCRETE_GPU {
        score += 1000;
    }

    // 텍스처의 최대 크기는 그래픽 품질에 영향을 줌
    score += device_properties.limits.max_image_dimension2d;

    // 애플리케이션은 지오메트리 셰이더 없이는 작동할 수 없음
    if device_features.geometry_shader != vk::TRUE {
        return 0;
    }

    score
}
```

이 튜토리얼에서 이 모든 것을 구현할 필요는 없지만, 디바이스 선택 프로세스를 어떻게 설계할 수 있는지에 대한 아이디어를 제공하기 위한 것입니다.

우리는 이제 막 시작하는 단계이므로, 벌칸 지원 여부만이 유일한 요구사항입니다. 따라서 어떤 GPU든 상관없이 사용하겠습니다.

```rust
fn is_device_suitable(&self, device: vk::PhysicalDevice) -> bool {
    true
}
```

다음 섹션에서는 확인해야 할 첫 번째 실질적인 필수 기능에 대해 논의할 것입니다.

## 큐 패밀리 (Queue families)

이전에도 잠시 언급했듯이, 드로잉부터 텍스처 업로드에 이르기까지 벌칸의 거의 모든 작업은 명령(command)을 큐(queue)에 제출해야 합니다. 큐에는 여러 종류가 있으며, 이들은 각기 다른 *큐 패밀리(queue families)*에서 비롯됩니다.

우리는 디바이스가 어떤 큐 패밀리를 지원하는지, 그리고 그중 어떤 큐 패밀리가 우리가 사용하려는 명령을 지원하는지 확인해야 합니다. 이를 위해, 우리가 필요로 하는 모든 큐 패밀리를 찾는 새로운 헬퍼 함수 `find_queue_families`를 추가하겠습니다.

다음 챕터에서 다른 종류의 큐를 찾게 될 것이므로, 미리 대비하여 인덱스들을 구조체로 묶는 것이 좋습니다.

```rust
struct QueueFamilyIndices {
    graphics_family: Option<u32>,
}
```

큐 패밀리가 존재하지 않는 경우를 처리해야 합니다. C++17의 `std::optional`과 같이, Rust에는 이를 위한 완벽한 타입인 `Option<T>`가 내장되어 있습니다. `Option`은 값이 존재함(`Some(value)`) 또는 존재하지 않음(`None`)을 나타내는 열거형(enum)입니다. `is_some()` 메서드를 사용하여 값이 있는지 확인할 수 있습니다.

이제 `find_queue_families`를 실제로 구현해 보겠습니다.

```rust
fn find_queue_families(&self, device: vk::PhysicalDevice) -> QueueFamilyIndices {
    let mut indices = QueueFamilyIndices {
        graphics_family: None,
    };

    let queue_families = unsafe {
        self.instance
            .get_physical_device_queue_family_properties(device)
    };

    // ...

    indices
}
```

`ash`의 `get_physical_device_queue_family_properties` 메서드는 큐 패밀리 속성 정보가 담긴 `Vec`을 반환합니다. 우리는 `VK_QUEUE_GRAPHICS_BIT`를 지원하는 큐 패밀리를 최소 하나 이상 찾아야 합니다.

`ash` 라이브러리는 비트 플래그를 위해 `bitflags` 크레이트를 사용합니다. 따라서 `&` 연산자 대신 `contains()` 메서드를 사용하여 플래그가 설정되어 있는지 확인하는 것이 더 관용적입니다.

```rust
let mut i = 0;
for queue_family in queue_families.iter() {
    if queue_family.queue_flags.contains(vk::QueueFlags::GRAPHICS) {
        indices.graphics_family = Some(i as u32);
    }

    i += 1;
}
```

Rust의 `enumerate()`를 사용하면 더 깔끔하게 작성할 수 있습니다.

```rust
for (i, queue_family) in queue_families.iter().enumerate() {
    if queue_family.queue_flags.contains(vk::QueueFlags::GRAPHICS) {
        indices.graphics_family = Some(i as u32);
    }
}
```

이제 이 큐 패밀리 조회 함수를 `is_device_suitable` 메서드에서 검사 항목으로 사용하여, 디바이스가 우리가 사용하려는 명령을 처리할 수 있는지 확인할 수 있습니다.

```rust
fn is_device_suitable(&self, device: vk::PhysicalDevice) -> bool {
    let indices = self.find_queue_families(device);

    indices.graphics_family.is_some()
}
```

이를 좀 더 편리하게 만들기 위해, 구조체 자체에 헬퍼 메서드를 추가하겠습니다.

```rust
impl QueueFamilyIndices {
    pub fn is_complete(&self) -> bool {
        self.graphics_family.is_some()
    }
}

// ...

fn is_device_suitable(&self, device: vk::PhysicalDevice) -> bool {
    let indices = self.find_queue_families(device);

    indices.is_complete()
}
```

이 메서드를 `find_queue_families`에서 조기 탈출하는 데에도 사용할 수 있습니다.

```rust
for (i, queue_family) in queue_families.iter().enumerate() {
    if queue_family.queue_flags.contains(vk::QueueFlags::GRAPHICS) {
        indices.graphics_family = Some(i as u32);
    }

    if indices.is_complete() {
        break;
    }
}
```

좋습니다, 이것으로 적절한 물리 디바이스를 찾는 데 필요한 모든 작업이 끝났습니다! 다음 단계는 [논리 디바이스를 생성하여](!ko/Drawing_a_triangle/Setup/Logical_device_and_queues) 물리 디바이스와 상호작용하는 것입니다.

[Rust 코드](/code/03_physical_device_selection.rs)