이 페이지는 Vulkan 애플리케이션을 개발하면서 마주칠 수 있는 일반적인 문제들에 대한 해결책을 다룹니다.

## 코어 검증 레이어에서 접근 위반(access violation) 오류가 발생합니다

MSI Afterburner / RivaTuner Statistics Server가 Vulkan과 몇 가지 호환성 문제가 있으므로, 해당 프로그램이 실행 중이지 않은지 확인하십시오.

## 검증 레이어에서 아무런 메시지도 표시되지 않거나, 검증 레이어를 사용할 수 없습니다

먼저, 프로그램이 종료된 후에도 터미널 창을 열어 두어 검증 레이어가 오류를 출력할 시간을 주어야 합니다. Visual Studio에서는 F5 대신 **Ctrl+F5**로 프로그램을 실행하고, Linux에서는 터미널 창에서 직접 프로그램을 실행하여 이를 수행할 수 있습니다.

그래도 메시지가 표시되지 않고 검증 레이어가 켜져 있는 것이 확실하다면, [이 페이지](https://vulkan.lunarg.com/doc/view/1.2.135.0/windows/getting_started.html)의 '설치 확인(Verify the Installation)' 안내에 따라 Vulkan SDK가 올바르게 설치되었는지 확인해야 합니다. 또한 `VK_LAYER_KHRONOS_validation` 레이어를 지원하려면 SDK 버전이 최소 **1.1.106.0** 이상인지 확인하십시오.

## vkCreateSwapchainKHR 함수 호출 시 SteamOverlayVulkanLayer64.dll에서 오류가 발생합니다

이것은 스팀(Steam) 클라이언트 베타의 호환성 문제로 보입니다. 다음과 같은 몇 가지 해결 방법이 있습니다:
*   스팀 베타 프로그램 참여를 중단합니다.
*   `DISABLE_VK_LAYER_VALVE_steam_overlay_1` 환경 변수를 `1`로 설정합니다.
*   `HKEY_LOCAL_MACHINE\SOFTWARE\Khronos\Vulkan\ImplicitLayers` 경로 아래 레지스트리에서 스팀 오버레이 Vulkan 레이어 항목을 삭제합니다.

예시:

![](/images/steam_layers_env.png)

## vkCreateInstance 호출이 VK_ERROR_INCOMPATIBLE_DRIVER 오류와 함께 실패합니다

최신 MoltenVK SDK와 함께 macOS를 사용하는 경우, `vkCreateInstance`가 `VK_ERROR_INCOMPATIBLE_DRIVER` 오류를 반환할 수 있습니다. 이는 [Vulkan SDK 버전 1.3.216 이상](https://vulkan.lunarg.com/doc/sdk/1.3.216.0/mac/getting_started.html)부터 MoltenVK가 아직 완벽하게 호환되지 않기 때문에, 이를 사용하려면 `VK_KHR_PORTABILITY_subset` 확장을 활성화해야 하기 때문입니다.

`VkInstanceCreateInfo`에 `VK_INSTANCE_CREATE_ENUMERATE_PORTABILITY_BIT_KHR` 플래그를 추가하고, 인스턴스 확장 목록에 `VK_KHR_PORTABILITY_ENUMERATION_EXTENSION_NAME`을 추가해야 합니다.

코드 예시:

```c++
...

std::vector<const char*> requiredExtensions;

for(uint32_t i = 0; i < glfwExtensionCount; i++) {
    requiredExtensions.emplace_back(glfwExtensions[i]);
}

requiredExtensions.emplace_back(VK_KHR_PORTABILITY_ENUMERATION_EXTENSION_NAME);

createInfo.flags |= VK_INSTANCE_CREATE_ENUMERATE_PORTABILITY_BIT_KHR;

createInfo.enabledExtensionCount = (uint32_t) requiredExtensions.size();
createInfo.ppEnabledExtensionNames = requiredExtensions.data();

if (vkCreateInstance(&createInfo, nullptr, &instance) != VK_SUCCESS) {
    throw std::runtime_error("failed to create instance!");
}
```