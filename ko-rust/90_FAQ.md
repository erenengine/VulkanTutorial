이 페이지는 Rust와 `ash` 라이브러리를 사용하여 Vulkan 애플리케이션을 개발하면서 마주칠 수 있는 일반적인 문제들에 대한 해결책을 다룹니다.

## 코어 검증 레이어에서 접근 위반(access violation) 오류가 발생합니다

MSI Afterburner / RivaTuner Statistics Server가 Vulkan과 몇 가지 호환성 문제가 있으므로, 해당 프로그램이 실행 중이지 않은지 확인하십시오.

## 검증 레이어에서 아무런 메시지도 표시되지 않거나, 검증 레이어를 사용할 수 없습니다

먼저, 프로그램이 종료된 후에도 터미널 창을 열어 두어 검증 레이어가 오류를 출력할 시간을 주어야 합니다. 터미널에서 `cargo run`을 사용하여 프로그램을 직접 실행하면 프로그램 종료 후에도 메시지를 확인할 수 있습니다.

그래도 메시지가 표시되지 않고 검증 레이어가 켜져 있는 것이 확실하다면, [이 페이지](https://vulkan.lunarg.com/doc/view/1.2.135.0/windows/getting_started.html)의 '설치 확인(Verify the Installation)' 안내에 따라 Vulkan SDK가 올바르게 설치되었는지 확인해야 합니다. 또한 `VK_LAYER_KHRONOS_validation` 레이어를 지원하려면 SDK 버전이 최소 **1.1.106.0** 이상인지 확인하십시오.

## vkCreateSwapchainKHR 함수 호출 시 SteamOverlayVulkanLayer64.dll에서 오류가 발생합니다

이것은 스팀(Steam) 클라이언트 베타의 호환성 문제로 보입니다. 다음과 같은 몇 가지 해결 방법이 있습니다:
*   스팀 베타 프로그램 참여를 중단합니다.
*   `DISABLE_VK_LAYER_VALVE_steam_overlay_1` 환경 변수를 `1`로 설정합니다.
*   `HKEY_LOCAL_MACHINE\SOFTWARE\Khronos\Vulkan\ImplicitLayers` 경로 아래 레지스트리에서 스팀 오버레이 Vulkan 레이어 항목을 삭제합니다.

## vkCreateInstance 호출이 VK_ERROR_INCOMPATIBLE_DRIVER 오류와 함께 실패합니다

최신 MoltenVK SDK와 함께 macOS를 사용하는 경우, `ash`의 `create_instance` 호출이 `VK_ERROR_INCOMPATIBLE_DRIVER` 오류를 반환할 수 있습니다. 이는 [Vulkan SDK 버전 1.3.216 이상](https://vulkan.lunarg.com/doc/sdk/1.3.216.0/mac/getting_started.html)부터 MoltenVK가 아직 완벽하게 호환되지 않기 때문에, 이를 사용하려면 `VK_KHR_PORTABILITY_subset` 확장을 활성화해야 하기 때문입니다.

`ash::vk::InstanceCreateInfo`를 생성할 때 플래그에 `ash::vk::InstanceCreateFlags::ENUMERATE_PORTABILITY_KHR`를 추가하고, 활성화할 인스턴스 확장 목록에 `ash::extensions::khr::PortabilityEnumeration::name()`을 추가해야 합니다.

코드 예시 (`winit`과 `ash-window`를 사용하는 일반적인 경우):

```rust
use ash::extensions::khr::PortabilityEnumeration;
use ash::vk;

// ... Entry, Window 생성 ...

// winit 같은 윈도우 라이브러리에서 요구하는 확장을 가져옵니다.
let mut extension_names = ash_window::enumerate_required_extensions(window)
    .unwrap()
    .to_vec();

// Portability 확장을 추가합니다.
extension_names.push(PortabilityEnumeration::name().as_ptr());

// create_info의 flags에 ENUMERATE_PORTABILITY_KHR를 추가합니다.
let create_flags = vk::InstanceCreateFlags::ENUMERATE_PORTABILITY_KHR;

let app_info = vk::ApplicationInfo::builder()
    .application_name(CStr::from_bytes_with_nul(b"Vulkan App\0").unwrap())
    // ... 기타 정보 설정
    .build();

let create_info = vk::InstanceCreateInfo::builder()
    .application_info(&app_info)
    .enabled_extension_names(&extension_names)
    // .enabled_layer_names(&layer_names) // 검증 레이어 등
    .flags(create_flags); // 여기에 플래그를 설정합니다.

let instance: ash::Instance = unsafe {
    entry
        .create_instance(&create_info, None)
        .expect("인스턴스 생성에 실패했습니다!");
};
```