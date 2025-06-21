## 밸리데이션 레이어란 무엇인가?

Vulkan API는 최소한의 드라이버 오버헤드를 목표로 설계되었으며, 이 목표가 드러나는 부분 중 하나는 API에 기본적으로 내장된 오류 검사가 매우 제한적이라는 점입니다. 열거형(enum) 값을 잘못 설정하거나 필수 파라미터에 null 포인터를 전달하는 것과 같은 간단한 실수조차도 일반적으로 명시적으로 처리되지 않으며, 크래시나 정의되지 않은 동작(undefined behavior)으로 이어질 뿐입니다. Vulkan은 개발자가 수행하는 모든 작업을 매우 명시적으로 지정해야 하므로, 논리 장치(logical device)를 생성할 때 새로운 GPU 기능을 사용하면서 해당 기능 사용을 요청하는 것을 잊는 등 많은 사소한 실수를 하기 쉽습니다.

하지만 이러한 검사를 API에 추가할 수 없다는 의미는 아닙니다. Vulkan은 이를 위해 *밸리데이션 레이어*라는 멋진 시스템을 도입했습니다. 밸리데이션 레이어는 Vulkan 함수 호출에 끼어들어(hook into) 추가적인 작업을 적용하는 선택적 컴포넌트입니다. 밸리데이션 레이어의 일반적인 작업은 다음과 같습니다.

*   사양에 명시된 값과 파라미터 값을 비교하여 오용을 감지
*   객체의 생성 및 소멸을 추적하여 리소스 누수(resource leak)를 발견
*   호출이 발생한 스레드를 추적하여 스레드 안전성(thread safety)을 검사
*   모든 호출과 그 파라미터를 표준 출력으로 로깅
*   프로파일링 및 재현(replaying)을 위해 Vulkan 호출을 추적

이러한 밸리데이션 레이어들은 원하는 모든 디버깅 기능을 포함하도록 자유롭게 쌓아서(stacked) 사용할 수 있습니다. 디버그 빌드에서는 밸리데이션 레이어를 활성화하고 릴리즈 빌드에서는 완전히 비활성화하면, 두 가지 장점을 모두 누릴 수 있습니다!

Vulkan은 내장된 밸리데이션 레이어를 제공하지 않지만, LunarG Vulkan SDK는 일반적인 오류를 검사하는 훌륭한 레이어 세트를 제공합니다. 이 레이어들은 완전히 [오픈 소스](https://github.com/KhronosGroup/Vulkan-ValidationLayers)이므로, 어떤 종류의 실수를 검사하는지 확인하고 기여할 수도 있습니다. 밸리데이션 레이어를 사용하는 것은 실수로 정의되지 않은 동작에 의존하여 애플리케이션이 다른 드라이버에서 깨지는 것을 방지하는 가장 좋은 방법입니다.

밸리데이션 레이어는 시스템에 설치된 경우에만 사용할 수 있습니다. 예를 들어, LunarG 밸리데이션 레이어는 Vulkan SDK가 설치된 PC에서만 사용할 수 있습니다.

## 밸리데이션 레이어 사용하기 (Rust & Ash)

이 섹션에서는 Vulkan SDK가 제공하는 표준 진단 레이어를 활성화하는 방법을 살펴보겠습니다. 확장(extension)과 마찬가지로, 밸리데이션 레이어도 이름을 지정하여 활성화해야 합니다. 모든 유용한 표준 밸리데이션은 SDK에 포함된 `VK_LAYER_KHRONOS_validation`이라는 레이어에 번들로 제공됩니다.

먼저 활성화할 레이어와 그 활성화 여부를 결정하는 상수를 정의합시다. Rust에서는 C++의 `#ifdef NDEBUG`와 유사한 역할을 하는 조건부 컴파일 속성 `#[cfg(debug_assertions)]`를 사용합니다. `debug_assertions`는 디버그 빌드에서 활성화됩니다.

Vulkan C API는 C 스타일 문자열(null로 끝나는 바이트 배열)을 요구하므로, `std::ffi::CString`을 사용해 Rust 문자열을 변환해야 합니다.

```rust
use std::ffi::{c_char, CStr, CString};

const WIDTH: u32 = 800;
const HEIGHT: u32 = 600;

// Vulkan C API와 통신하기 위해 C 문자열 포인터 목록을 만듭니다.
const VALIDATION_LAYERS: [*const c_char; 1] =
    [b"VK_LAYER_KHRONOS_validation\0".as_ptr() as *const c_char];

// cfg! 매크로를 사용하여 컴파일 타임에 값을 결정합니다.
const ENABLE_VALIDATION_LAYERS: bool = cfg!(debug_assertions);
```
**참고:** `b"..."` 구문은 바이트 슬라이스를 만듭니다. 여기에 `\0`을 추가하여 C 문자열 형식에 맞춥니다.

다음으로, 요청된 모든 레이어가 사용 가능한지 확인하는 `check_validation_layer_support` 함수를 추가합니다. `ash`의 `Entry::enumerate_instance_layer_properties` 함수를 사용합니다.

```rust
// 이 함수는 C 문자열 포인터를 다루므로 unsafe 블록이 필요합니다.
unsafe fn check_validation_layer_support(entry: &ash::Entry) -> bool {
    let available_layers = entry
        .enumerate_instance_layer_properties()
        .expect("인스턴스 레이어 속성을 가져오지 못했습니다.");

    for &layer_name_ptr in VALIDATION_LAYERS.iter() {
        let layer_name = CStr::from_ptr(layer_name_ptr);
        let mut layer_found = false;

        for layer_properties in available_layers.iter() {
            let available_layer_name = CStr::from_ptr(layer_properties.layer_name.as_ptr());
            if available_layer_name == layer_name {
                layer_found = true;
                break;
            }
        }

        if !layer_found {
            return false;
        }
    }

    true
}
```

이제 이 함수를 인스턴스 생성 로직에 통합합니다.

```rust
// in create_instance()
if ENABLE_VALIDATION_LAYERS && !unsafe { check_validation_layer_support(&self.entry) } {
    panic!("요청한 밸리데이션 레이어를 사용할 수 없습니다!");
}
```

마지막으로 `ash`의 빌더 패턴을 사용하여 `InstanceCreateInfo` 구조체를 수정하고, 밸리데이션 레이어를 활성화합니다.

```rust
// in create_instance()
let mut create_info = vk::InstanceCreateInfo::builder()
    .application_info(&app_info)
    .enabled_extension_names(&extensions);

if ENABLE_VALIDATION_LAYERS {
    create_info = create_info.enabled_layer_names(&VALIDATION_LAYERS);
}
// ...
```
`ash`의 빌더는 슬라이스(`&VALIDATION_LAYERS`)를 받아 자동으로 `enabledLayerCount`와 `ppEnabledLayerNames`를 설정해 주므로 매우 편리합니다.

## 메시지 콜백 (Message Callback)

밸리데이션 레이어는 기본적으로 디버그 메시지를 표준 출력으로 인쇄하지만, Rust에서 직접 콜백을 제공하여 처리할 수 있습니다. 이를 위해 `VK_EXT_debug_utils` 확장이 필요합니다.

먼저 필요한 확장 목록을 반환하는 함수를 수정합니다. `ash`는 `ash::extensions::ext::DebugUtils::name()`을 통해 확장 이름을 `&'static CStr`로 제공하여 편리하게 사용할 수 있습니다.

```rust
fn get_required_extensions(window: &winit::window::Window) -> Vec<*const c_char> {
    let mut extensions = ash_window::enumerate_required_extensions(window)
        .expect("필요한 확장 목록을 가져오지 못했습니다.")
        .to_vec();

    if ENABLE_VALIDATION_LAYERS {
        extensions.push(ash::extensions::ext::DebugUtils::name().as_ptr());
    }

    extensions
}
```

이제 디버그 콜백 함수를 정의합니다. 이 함수는 C ABI를 따라야 하므로 `extern "system"`으로 선언합니다. `ash`는 `PFN_vkDebugUtilsMessengerCallbackEXT` 타입 별칭을 제공합니다.

```rust
use ash::vk; // vk 네임스페이스를 가져옵니다.
use std::os::raw::c_void;

// Vulkan이 호출할 콜백 함수
unsafe extern "system" fn vulkan_debug_callback(
    message_severity: vk::DebugUtilsMessageSeverityFlagsEXT,
    _message_type: vk::DebugUtilsMessageTypeFlagsEXT,
    p_callback_data: *const vk::DebugUtilsMessengerCallbackDataEXT,
    _p_user_data: *mut c_void,
) -> vk::Bool32 {
    let message = CStr::from_ptr((*p_callback_data).p_message);
    let severity = format!("{:?}", message_severity).to_lowercase();
    println!("[Vulkan Validation] [{}]: {:?}", severity, message);

    vk::FALSE
}
```
**안전성(Safety):** 이 함수는 `unsafe`입니다. C 코드로부터 호출되며, `p_callback_data` 같은 원시 포인터를 역참조하기 때문입니다. `CStr::from_ptr`을 사용하여 메모리를 안전하게 읽습니다.

이제 이 콜백을 등록하는 `setup_debug_messenger` 함수를 만듭니다. `ash`에서는 확장 기능 로드가 매우 간단합니다. `DebugUtils` 구조체를 생성하기만 하면 됩니다. 이 구조체와 메신저 핸들은 애플리케이션이 살아있는 동안 유지되어야 하므로, 주 애플리케이션 구조체에 멤버로 저장합니다.

```rust
// 애플리케이션 구조체에 추가
struct HelloTriangleApplication {
    // ...
    debug_utils: Option<ash::extensions::ext::DebugUtils>,
    debug_messenger: Option<vk::DebugUtilsMessengerEXT>,
}

impl HelloTriangleApplication {
    fn setup_debug_messenger(&mut self) {
        if !ENABLE_VALIDATION_LAYERS {
            return;
        }

        let debug_utils = ash::extensions::ext::DebugUtils::new(&self.entry, &self.instance);

        let create_info = vk::DebugUtilsMessengerCreateInfoEXT::builder()
            .message_severity(
                vk::DebugUtilsMessageSeverityFlagsEXT::ERROR
                    | vk::DebugUtilsMessageSeverityFlagsEXT::WARNING
                    // | vk::DebugUtilsMessageSeverityFlagsEXT::INFO
                    // | vk::DebugUtilsMessageSeverityFlagsEXT::VERBOSE,
            )
            .message_type(
                vk::DebugUtilsMessageTypeFlagsEXT::GENERAL
                    | vk::DebugUtilsMessageTypeFlagsEXT::VALIDATION
                    | vk::DebugUtilsMessageTypeFlagsEXT::PERFORMANCE,
            )
            .pfn_user_callback(Some(vulkan_debug_callback))
            .build();

        let debug_messenger = unsafe {
            debug_utils
                .create_debug_utils_messenger(&create_info, None)
                .expect("디버그 메신저 설정에 실패했습니다!")
        };
        
        self.debug_utils = Some(debug_utils);
        self.debug_messenger = Some(debug_messenger);
    }
}
```
`ash`의 `DebugUtils::new`는 필요한 함수 포인터(`vkCreateDebugUtilsMessengerEXT`, `vkDestroyDebugUtilsMessengerEXT` 등)를 자동으로 로드합니다. 수동으로 `vkGetInstanceProcAddr`를 호출할 필요가 없습니다.

메신저는 리소스이므로, 애플리케이션이 종료될 때 반드시 정리해야 합니다. Rust에서는 `Drop` 트레잇을 구현하여 이를 자동화하는 것이 가장 이상적입니다(RAII 패턴).

```rust
impl Drop for HelloTriangleApplication {
    fn drop(&mut self) {
        unsafe {
            if let (Some(debug_utils), Some(debug_messenger)) = (self.debug_utils.as_ref(), self.debug_messenger) {
                debug_utils.destroy_debug_utils_messenger(*debug_messenger, None);
            }
            // ... 다른 리소스 정리 ...
            self.instance.destroy_instance(None);
        }
    }
}
```

## 인스턴스 생성 및 소멸 디버깅하기

`vkCreateInstance`와 `vkDestroyInstance` 호출 자체의 오류를 디버깅하려면, 인스턴스 생성 정보에 디버그 메신저 생성 정보를 연결해야 합니다. `ash` 빌더의 `.p_next()` 메서드를 사용합니다.

```rust
// in create_instance()
let mut debug_create_info = vk::DebugUtilsMessengerCreateInfoEXT::builder()
    .message_severity(
        vk::DebugUtilsMessageSeverityFlagsEXT::ERROR
            | vk::DebugUtilsMessageSeverityFlagsEXT::WARNING,
    )
    .message_type(
        vk::DebugUtilsMessageTypeFlagsEXT::GENERAL
            | vk::DebugUtilsMessageTypeFlagsEXT::VALIDATION
            | vk::DebugUtilsMessageTypeFlagsEXT::PERFORMANCE,
    )
    .pfn_user_callback(Some(vulkan_debug_callback));

let mut create_info = vk::InstanceCreateInfo::builder()
    .application_info(&app_info)
    .enabled_extension_names(&extensions);

if ENABLE_VALIDATION_LAYERS {
    create_info = create_info
        .enabled_layer_names(&VALIDATION_LAYERS)
        .p_next(&mut debug_create_info as *mut _ as *const c_void);
}
```
**안전성(Safety):** `.p_next()`는 원시 포인터를 받기 때문에 `unsafe` 코드 블록이 필요합니다. `&mut debug_create_info as *mut _ as *const c_void` 캐스팅은 Rust의 가변 참조를 C API가 요구하는 `const void*` 타입으로 변환합니다. `debug_create_info` 변수는 `vkCreateInstance` 호출 이후까지 살아있어야 하므로, `create_info`보다 먼저 선언되어야 합니다.

이 메신저는 인스턴스 생성 및 소멸 시에만 사용되고 자동으로 정리되므로, 별도로 `destroy`를 호출할 필요가 없습니다.

## 테스트하기

의도적으로 실수를 만들어 밸리데이션 레이어가 작동하는지 확인해 봅시다. `Drop` 구현에서 `destroy_debug_utils_messenger` 호출을 일시적으로 주석 처리하고 디버그 모드로 프로그램을 실행하세요. 프로그램이 종료될 때 다음과 유사한 오류 메시지가 터미널에 출력될 것입니다.

```
[Vulkan Validation] [error]: Validation Error: [ UNASSIGNED-ObjectTracker-ObjectLeak ] Object 0x... (type: DEBUG_UTILS_MESSENGER_EXT) was not destroyed.
```
이 메시지는 디버그 메신저가 제대로 정리되지 않았음을 명확히 알려줍니다.

## 설정

`vk_layer_settings.txt` 파일을 이용한 밸리데이션 레이어 설정은 C++과 동일하게 적용됩니다. 이 파일은 언어에 구애받지 않고 Vulkan SDK가 읽어 들이기 때문입니다.

`ash`와 Rust를 사용하면 빌더 패턴, 타입 안전성, `Drop`을 통한 자동 리소스 관리(RAII) 등 Rust의 강력한 기능들을 활용하여 C++보다 더 안전하고 간결하게 밸리데이션 레이어를 설정할 수 있습니다. 이제 시스템의 [Vulkan 장치](!ko/Drawing_a_triangle/Setup/Physical_devices_and_queue_families)에 대해 알아볼 시간입니다.

[Rust 코드](/code/02_validation_layers.rs)