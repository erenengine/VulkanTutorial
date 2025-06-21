## 인스턴스 생성 (Rust/Ash)

가장 먼저 해야 할 일은 *인스턴스(instance)*를 생성하여 Vulkan 라이브러리를 초기화하는 것입니다. 인스턴스는 애플리케이션과 Vulkan 라이브러리 간의 연결고리이며, 이 과정에서 애플리케이션에 대한 몇 가지 세부 정보를 드라이버에 지정해야 합니다.

먼저 애플리케이션 구조체에 `create_instance` 함수를 추가하고, 나중에 만들 `init_vulkan` 함수에서 호출하도록 구성합니다.

```rust
fn init_vulkan(&mut self) -> Result<(), Box<dyn std::error::Error>> {
    self.create_instance()?;
    Ok(())
}
```

또한 인스턴스 핸들을 저장할 필드를 구조체에 추가합니다. `ash::Instance` 타입은 Vulkan 인스턴스를 나타냅니다.

```rust
struct HelloTriangleApplication {
    // ... 다른 필드들
    instance: ash::Instance,
}
```

이제 인스턴스를 생성하기 위해, 먼저 애플리케이션에 대한 정보가 담긴 구조체를 채워야 합니다. 이 데이터는 기술적으로 선택 사항이지만, 드라이버가 우리의 특정 애플리케이션을 최적화하는 데 유용한 정보를 제공할 수 있습니다(예: 특정 특수 동작을 하는 잘 알려진 그래픽 엔진을 사용하는 경우). 이 구조체는 `vk::ApplicationInfo`입니다.

Ash에서는 빌더(builder) 패턴을 사용하여 구조체를 안전하고 편리하게 생성합니다.

```rust
// create_instance 함수 내부
use std::ffi::CStr;
use ash::vk;

// ...

let app_name = CStr::from_bytes_with_nul(b"Hello Triangle\0").unwrap();
let engine_name = CStr::from_bytes_with_nul(b"No Engine\0").unwrap();

let app_info = vk::ApplicationInfo::builder()
    .application_name(app_name)
    .application_version(vk::make_api_version(0, 1, 0, 0))
    .engine_name(engine_name)
    .engine_version(vk::make_api_version(0, 1, 0, 0))
    .api_version(vk::API_VERSION_1_0);
```

Vulkan의 많은 구조체와 마찬가지로, `sType` 멤버는 빌더가 자동으로 설정해 줍니다. Ash의 빌더는 `pNext` 멤버를 다루는 확장 기능도 지원하지만, 여기서는 기본값인 null 포인터로 둡니다.

다음으로, 인스턴스 생성을 위한 더 중요한 정보를 담은 구조체를 채워야 합니다. 이 구조체는 필수이며, 우리가 사용할 전역 확장(global extensions)과 유효성 검사 레이어(validation layers)를 Vulkan 드라이버에 알려줍니다.

```rust
let create_info = vk::InstanceCreateInfo::builder()
    .application_info(&app_info);
```

이제 원하는 전역 확장을 지정해야 합니다. Vulkan은 플랫폼에 구애받지 않는 API이므로, 창 시스템과 상호작용하려면 확장이 필요합니다. C++의 GLFW와 마찬가지로, Rust 생태계에서는 `winit` 창 라이브러리와 `ash-window` 크레이트를 함께 사용하여 필요한 확장 목록을 쉽게 얻을 수 있습니다.

```rust
// winit::window::Window 객체가 있다고 가정합니다.
// let window: winit::window::Window = ...;

let required_extensions = ash_window::enumerate_required_extensions(window.display_handle().unwrap().as_raw())
    .unwrap()
    .to_vec();
```

이제 이 확장 목록을 `InstanceCreateInfo` 빌더에 추가합니다.

```rust
let mut create_info = vk::InstanceCreateInfo::builder()
    .application_info(&app_info)
    .enabled_extension_names(&required_extensions);
```

구조체의 마지막 부분은 활성화할 전역 유효성 검사 레이어를 결정합니다. 다음 장에서 자세히 다룰 것이므로 지금은 비워둡니다.

```rust
// create_info 빌더 체인에 추가
// .enabled_layer_count(0)
// .pp_enabled_layer_names(std::ptr::null())
```

이제 Vulkan이 인스턴스를 생성하는 데 필요한 모든 것을 지정했으므로, 마침내 `create_instance`를 호출할 수 있습니다. Ash에서는 Vulkan 라이브러리 로딩을 담당하는 `ash::Entry` 객체를 통해 이 함수를 호출합니다.

```rust
// entry: &ash::Entry 는 함수 인자로 전달받았다고 가정
let instance = unsafe {
    entry
        .create_instance(&create_info, None)
        .expect("Failed to create instance!")
};
self.instance = instance;
```

Rust/Ash에서 객체 생성 함수의 패턴은 다음과 같습니다.

1.  생성 정보가 담긴 구조체의 빌더를 사용합니다.
2.  `ash::Entry` (전역 함수용) 또는 다른 Vulkan 객체(자식 객체용)의 메서드를 호출합니다.
3.  첫 번째 인자로 생성 정보 구조체에 대한 참조를 전달합니다.
4.  두 번째 인자는 사용자 정의 할당자 콜백으로, 이 튜토리얼에서는 `None`을 사용합니다.
5.  이 함수는 `Result`를 반환하므로, Rust의 오류 처리 메커니즘(`?` 연산자, `match`, `expect` 등)을 사용하여 결과를 처리합니다.

`create_instance` 호출은 `unsafe` 블록 안에 있습니다. 이는 Ash가 우리가 제공한 포인터(예: 확장 이름)가 유효하고 올바른 생명주기를 가졌는지 보장할 수 없기 때문입니다. 우리는 이 조건들을 충족함을 보장해야 합니다.

### VK_ERROR_INCOMPATIBLE_DRIVER 오류 발생 시 (macOS)
최신 MoltenVK SDK를 사용하는 macOS에서는 `VK_KHR_portability_subset` 확장이 필수로 요구될 수 있습니다. 이로 인해 `create_instance`가 실패할 수 있습니다.

이 문제를 해결하려면, `VK_INSTANCE_CREATE_ENUMERATE_PORTABILITY_BIT_KHR` 플래그를 추가하고, `VK_KHR_PORTABILITY_ENUMERATION_EXTENSION_NAME` 확장을 활성화해야 합니다.

Rust에서는 `cfg` 속성을 사용하여 플랫폼별 코드를 작성할 수 있습니다.

```rust
let mut required_extensions = ash_window::enumerate_required_extensions(window.display_handle().unwrap().as_raw())
    .unwrap()
    .to_vec();

let mut create_info = vk::InstanceCreateInfo::builder()
    .application_info(&app_info);

if cfg!(target_os = "macos") {
    required_extensions.push(ash::extensions::khr::PortabilityEnumeration::name().as_ptr());
    create_info = create_info.flags(vk::InstanceCreateFlags::ENUMERATE_PORTABILITY_KHR);
}

create_info = create_info.enabled_extension_names(&required_extensions);

// ... create_instance 호출
```

### 확장 지원 여부 확인

선택적 기능에 대한 지원 여부를 확인하고 싶다면, 인스턴스를 생성하기 전에 `enumerate_instance_extension_properties`를 사용하여 지원되는 확장 목록을 가져올 수 있습니다. Ash는 이 과정을 매우 간단하게 만들어 줍니다.

```rust
// entry: &ash::Entry

let available_extensions = entry
    .enumerate_instance_extension_properties(None)
    .expect("Failed to enumerate instance extensions");

println!("Available extensions:");
for extension in available_extensions.iter() {
    let extension_name = unsafe { CStr::from_ptr(extension.extension_name.as_ptr()) };
    println!("\t{}", extension_name.to_str().unwrap());
}
```

Ash는 C++처럼 두 번 호출할 필요 없이 지원되는 모든 확장을 `Vec<vk::ExtensionProperties>`로 편리하게 반환해 줍니다.

도전 과제로, `ash-window`가 요구하는 모든 확장이 `available_extensions` 목록에 포함되어 있는지 확인하는 코드를 작성해 보세요. Rust의 `HashSet`과 이터레이터를 사용하면 효율적으로 구현할 수 있습니다.

### 정리 (Cleaning up)

`ash::Instance`는 프로그램이 종료되기 직전에만 파괴되어야 합니다. Rust에서는 RAII(Resource Acquisition Is Initialization) 패턴을 따르는 것이 가장 일반적입니다. 애플리케이션 구조체에 대해 `Drop` 트레이트를 구현하여 리소스가 범위를 벗어날 때 자동으로 정리되도록 합니다.

```rust
impl Drop for HelloTriangleApplication {
    fn drop(&mut self) {
        unsafe {
            // 다른 모든 Vulkan 리소스가 파괴된 후에 인스턴스를 파괴해야 합니다.
            self.instance.destroy_instance(None);
        }
    }
}
```

`destroy_instance` 호출은 `unsafe`입니다. 왜냐하면 이 인스턴스로부터 생성된 다른 모든 Vulkan 리소스(디바이스, 버퍼 등)가 이미 파괴되었음을 프로그래머가 보장해야 하기 때문입니다. `drop` 메서드 내에서 필드의 소멸 순서를 올바르게 지정하면 이 요구사항을 충족할 수 있습니다.

인스턴스 생성 후의 더 복잡한 단계로 넘어가기 전에, [유효성 검사 레이어](!ko/Drawing_a_triangle/Setup/Validation_layers)를 살펴봄으로써 디버깅 옵션을 평가해 볼 시간입니다.

[Rust 코드](/code/rust/01_instance_creation.rs)