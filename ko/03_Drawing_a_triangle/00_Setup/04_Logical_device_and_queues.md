## 도입

사용할 물리 장치를 선택한 후에는, 이와 상호작용하기 위한 *논리 장치*를 설정해야 합니다. 논리 장치 생성 과정은 인스턴스 생성 과정과 유사하며, 우리가 사용하고자 하는 기능들을 기술합니다. 또한, 어떤 큐 패밀리를 사용할 수 있는지 질의했으므로 이제 어떤 큐를 생성할지 명시해야 합니다. 요구 사항이 다양하다면 동일한 물리 장치에서 여러 개의 논리 장치를 생성할 수도 있습니다.

먼저 클래스 멤버를 새로 추가하여 논리 장치 핸들을 저장하도록 합시다.

```c++
VkDevice device;
```

다음으로, `initVulkan`에서 호출될 `createLogicalDevice` 함수를 추가합니다.

```c++
void initVulkan() {
    createInstance();
    setupDebugMessenger();
    pickPhysicalDevice();
    createLogicalDevice();
}

void createLogicalDevice() {

}
```

## 생성할 큐 명시하기

논리 장치 생성 과정에는 여러 구조체에 상세 정보를 지정하는 작업이 포함됩니다. 그중 첫 번째는 `VkDeviceQueueCreateInfo`입니다. 이 구조체는 단일 큐 패밀리에 대해 우리가 원하는 큐의 개수를 기술합니다. 지금 당장은 그래픽스 기능이 있는 큐에만 관심이 있습니다.

```c++
QueueFamilyIndices indices = findQueueFamilies(physicalDevice);

VkDeviceQueueCreateInfo queueCreateInfo{};
queueCreateInfo.sType = VK_STRUCTURE_TYPE_DEVICE_QUEUE_CREATE_INFO;
queueCreateInfo.queueFamilyIndex = indices.graphicsFamily.value();
queueCreateInfo.queueCount = 1;
```

현재 사용 가능한 드라이버들은 각 큐 패밀리마다 소수의 큐만 생성하도록 허용하며, 실제로 하나보다 더 많이 필요한 경우는 거의 없습니다. 그 이유는 여러 스레드에서 모든 커맨드 버퍼를 생성한 다음, 메인 스레드에서 단 한 번의 오버헤드가 적은 호출로 모두 제출할 수 있기 때문입니다.

Vulkan에서는 `0.0`에서 `1.0` 사이의 부동 소수점 숫자를 사용하여 큐에 우선순위를 할당하고 커맨드 버퍼 실행 스케줄링에 영향을 줄 수 있습니다. 큐가 하나만 있는 경우에도 이 설정은 필수입니다.

```c++
float queuePriority = 1.0f;
queueCreateInfo.pQueuePriorities = &queuePriority;
```

## 사용할 장치 기능 명시하기

다음으로 명시할 정보는 우리가 사용할 장치 기능의 집합입니다. 이는 이전 장에서 `vkGetPhysicalDeviceFeatures`로 지원 여부를 질의했던 지오메트리 셰이더와 같은 기능들입니다. 지금 당장은 특별한 기능이 필요 없으므로, 구조체를 정의하고 모든 값을 `VK_FALSE`로 두면 됩니다. Vulkan으로 더 흥미로운 작업을 시작할 때 이 구조체로 다시 돌아올 것입니다.

```c++
VkPhysicalDeviceFeatures deviceFeatures{};
```

## 논리 장치 생성하기

앞선 두 구조체가 준비되었으니, 이제 메인 `VkDeviceCreateInfo` 구조체를 채워나갈 수 있습니다.

```c++
VkDeviceCreateInfo createInfo{};
createInfo.sType = VK_STRUCTURE_TYPE_DEVICE_CREATE_INFO;
```

먼저 큐 생성 정보와 장치 기능 구조체를 가리키는 포인터를 추가합니다.

```c++
createInfo.pQueueCreateInfos = &queueCreateInfo;
createInfo.queueCreateInfoCount = 1;

createInfo.pEnabledFeatures = &deviceFeatures;
```

나머지 정보는 `VkInstanceCreateInfo` 구조체와 유사하며, 확장과 유효성 검사 레이어를 명시해야 합니다. 차이점은 이번에는 이것들이 장치에 한정된다는 점입니다.

장치별 확장의 한 예로 `VK_KHR_swapchain`이 있습니다. 이 확장은 해당 장치에서 렌더링된 이미지를 창에 표시할 수 있게 해줍니다. 시스템에 있는 Vulkan 장치 중에는 이 기능이 없는 경우도 있을 수 있는데, 예를 들어 연산 작업만 지원하는 장치가 그렇습니다. 이 확장에 대해서는 스왑 체인 장에서 다시 다룰 것입니다.

이전 Vulkan 구현에서는 인스턴스와 장치별 유효성 검사 레이어를 구분했지만, [이제는 그렇지 않습니다](https://www.khronos.org/registry/vulkan/specs/1.3-extensions/html/chap40.html#extendingvulkan-layers-devicelayerdeprecation). 이는 최신 구현에서 `VkDeviceCreateInfo`의 `enabledLayerCount`와 `ppEnabledLayerNames` 필드가 무시된다는 의미입니다. 하지만 구버전 구현과의 호환성을 위해 여전히 설정해두는 것이 좋습니다.

```c++
createInfo.enabledExtensionCount = 0;

if (enableValidationLayers) {
    createInfo.enabledLayerCount = static_cast<uint32_t>(validationLayers.size());
    createInfo.ppEnabledLayerNames = validationLayers.data();
} else {
    createInfo.enabledLayerCount = 0;
}
```

지금은 장치별 확장이 필요하지 않습니다.

이제 모든 준비가 끝났습니다. 이름에 걸맞게 `vkCreateDevice` 함수를 호출하여 논리 장치를 인스턴스화할 준비가 되었습니다.

```c++
if (vkCreateDevice(physicalDevice, &createInfo, nullptr, &device) != VK_SUCCESS) {
    throw std::runtime_error("failed to create logical device!");
}
```

파라미터는 각각 상호작용할 물리 장치, 방금 명시한 큐 및 사용 정보, 선택적인 할당 콜백 포인터, 그리고 논리 장치 핸들을 저장할 변수를 가리키는 포인터입니다. 인스턴스 생성 함수와 유사하게, 이 호출은 존재하지 않는 확장을 활성화하거나 지원되지 않는 기능의 사용을 명시하는 경우 오류를 반환할 수 있습니다.

장치는 `cleanup`에서 `vkDestroyDevice` 함수로 소멸시켜야 합니다.

```c++
void cleanup() {
    vkDestroyDevice(device, nullptr);
    ...
}
```

논리 장치는 인스턴스와 직접적으로 상호작용하지 않기 때문에, 파라미터로 포함되지 않습니다.

## 큐 핸들 가져오기

큐는 논리 장치와 함께 자동으로 생성되지만, 아직 큐와 상호작용할 핸들을 가지고 있지는 않습니다. 먼저 클래스 멤버를 추가하여 그래픽스 큐의 핸들을 저장하도록 합시다.

```c++
VkQueue graphicsQueue;
```

장치 큐는 장치가 소멸될 때 암시적으로 정리되므로, `cleanup`에서 따로 처리할 필요가 없습니다.

`vkGetDeviceQueue` 함수를 사용하여 각 큐 패밀리에 대한 큐 핸들을 가져올 수 있습니다. 파라미터는 논리 장치, 큐 패밀리, 큐 인덱스, 그리고 큐 핸들을 저장할 변수를 가리키는 포인터입니다. 이 패밀리에서는 큐를 하나만 생성하므로, 인덱스는 간단히 `0`을 사용합니다.

```c++
vkGetDeviceQueue(device, indices.graphicsFamily.value(), 0, &graphicsQueue);
```

이제 논리 장치와 큐 핸들을 사용해 그래픽 카드로 무언가를 실제로 시작할 수 있습니다! 다음 몇 개의 장에 걸쳐, 결과를 창 시스템에 표시하기 위한 리소스를 설정해 보겠습니다.

[C++ 코드](/code/04_logical_device.cpp)