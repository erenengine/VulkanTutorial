Vulkan은 플랫폼에 종속되지 않는 API이므로, 그 자체만으로는 윈도우 시스템과 직접 상호작용할 수 없습니다. Vulkan과 윈도우 시스템을 연결하여 렌더링 결과를 화면에 표시하려면 WSI(Window System Integration) 확장을 사용해야 합니다. 이번 장에서는 그 첫 번째 확장인 `VK_KHR_surface`에 대해 논의하겠습니다. 이 확장은 렌더링된 이미지를 표시할 추상적인 유형의 표면을 나타내는 `VkSurfaceKHR` 객체를 제공합니다. 우리 프로그램의 서피스는 이미 GLFW로 열어 둔 창을 기반으로 생성될 것입니다.

`VK_KHR_surface` 확장은 인스턴스 수준 확장(instance level extension)이며, `glfwGetRequiredInstanceExtensions`가 반환하는 목록에 포함되어 있으므로 우리는 이미 이 확장을 활성화했습니다. 이 목록에는 다음 몇 장에서 사용할 다른 WSI 확장들도 포함되어 있습니다.

윈도우 서피스는 인스턴스 생성 직후에 만들어야 합니다. 왜냐하면 서피스는 물리 장치 선택에 영향을 줄 수 있기 때문입니다. 이 작업을 뒤로 미룬 이유는 윈도우 서피스가 렌더 타겟 및 프레젠테이션이라는 더 큰 주제의 일부이며, 이를 기본 설정 과정에서 설명하면 내용이 복잡해지기 때문입니다. 또한, 오프스크린 렌더링(off-screen rendering)만 필요한 경우 윈도우 서피스는 전적으로 선택적인 구성 요소라는 점에 유의해야 합니다. Vulkan을 사용하면 OpenGL에서 필요했던 보이지 않는 창을 만드는 것과 같은 꼼수 없이도 오프스크린 렌더링이 가능합니다.

## 윈도우 서피스 생성하기

먼저 디버그 콜백 바로 아래에 `surface` 클래스 멤버를 추가합니다.

```c++
VkSurfaceKHR surface;
```

`VkSurfaceKHR` 객체와 그 사용법은 플랫폼에 독립적이지만, 그 생성 과정은 윈도우 시스템의 세부 사항에 의존하기 때문에 플랫폼 종속적입니다. 예를 들어, Windows에서는 `HWND`와 `HMODULE` 핸들이 필요합니다. 따라서 플랫폼별 추가 확장이 있으며, Windows에서는 이를 `VK_KHR_win32_surface`라고 부릅니다. 이 확장 역시 `glfwGetRequiredInstanceExtensions`가 반환하는 목록에 자동으로 포함됩니다.

이 플랫폼별 확장을 사용하여 Windows에서 서피스를 만드는 방법을 보여드리겠지만, 이 튜토리얼에서 실제로 사용하지는 않을 것입니다. GLFW와 같은 라이브러리를 사용하면서 플랫폼 종속적인 코드를 사용하는 것은 이치에 맞지 않기 때문입니다. 사실 GLFW에는 `glfwCreateWindowSurface`라는 함수가 있어 플랫폼 간의 차이점을 알아서 처리해줍니다. 그럼에도 불구하고, GLFW에 의존하기 전에 내부적으로 어떤 작업이 이루어지는지 살펴보는 것은 좋은 경험이 될 것입니다.

네이티브 플랫폼 함수에 접근하려면 상단의 include 구문을 다음과 같이 수정해야 합니다.

```c++
#define VK_USE_PLATFORM_WIN32_KHR
#define GLFW_INCLUDE_VULKAN
#include <GLFW/glfw3.h>
#define GLFW_EXPOSE_NATIVE_WIN32
#include <GLFW/glfw3native.h>
```

윈도우 서피스는 Vulkan 객체이므로, `VkWin32SurfaceCreateInfoKHR` 구조체를 채워야 합니다. 여기에는 `hwnd`와 `hinstance`라는 두 가지 중요한 매개변수가 있습니다. 이들은 각각 창과 프로세스에 대한 핸들입니다.

```c++
VkWin32SurfaceCreateInfoKHR createInfo{};
createInfo.sType = VK_STRUCTURE_TYPE_WIN32_SURFACE_CREATE_INFO_KHR;
createInfo.hwnd = glfwGetWin32Window(window);
createInfo.hinstance = GetModuleHandle(nullptr);
```

`glfwGetWin32Window` 함수는 GLFW 윈도우 객체로부터 원시 `HWND`를 가져오는 데 사용됩니다. `GetModuleHandle` 호출은 현재 프로세스의 `HINSTANCE` 핸들을 반환합니다.

그 후 `vkCreateWin32SurfaceKHR` 함수로 서피스를 생성할 수 있습니다. 이 함수는 인스턴스, 서피스 생성 정보, 사용자 정의 할당자, 그리고 서피스 핸들을 저장할 변수를 매개변수로 받습니다. 기술적으로 이 함수는 WSI 확장 함수이지만 매우 보편적으로 사용되기 때문에 표준 Vulkan 로더에 포함되어 있습니다. 따라서 다른 확장과 달리 명시적으로 함수 포인터를 로드할 필요가 없습니다.

```c++
if (vkCreateWin32SurfaceKHR(instance, &createInfo, nullptr, &surface) != VK_SUCCESS) {
    throw std::runtime_error("failed to create window surface!");
}
```

이 과정은 X11을 사용하는 리눅스와 같은 다른 플랫폼에서도 비슷합니다. 리눅스에서는 `vkCreateXcbSurfaceKHR` 함수가 XCB 연결과 윈도우를 생성 정보로 받습니다.

`glfwCreateWindowSurface` 함수는 각 플랫폼에 맞춰 정확히 이 작업을 수행합니다. 이제 이 함수를 우리 프로그램에 통합해 보겠습니다. `initVulkan` 함수에서 인스턴스 생성과 `setupDebugMessenger` 호출 직후에 호출될 `createSurface` 함수를 추가합니다.

```c++
void initVulkan() {
    createInstance();
    setupDebugMessenger();
    createSurface();
    pickPhysicalDevice();
    createLogicalDevice();
}

void createSurface() {

}
```

GLFW 호출은 구조체 대신 간단한 매개변수를 받기 때문에 함수 구현이 매우 간단합니다.

```c++
void createSurface() {
    if (glfwCreateWindowSurface(instance, window, nullptr, &surface) != VK_SUCCESS) {
        throw std::runtime_error("failed to create window surface!");
    }
}
```

매개변수는 `VkInstance`, GLFW 윈도우 포인터, 사용자 정의 할당자, 그리고 `VkSurfaceKHR` 변수를 가리키는 포인터입니다. 이 함수는 각 플랫폼에 맞는 생성 함수의 `VkResult`를 그대로 반환합니다. GLFW는 서피스 파괴를 위한 특별한 함수를 제공하지 않지만, 원래 Vulkan API를 통해 쉽게 처리할 수 있습니다.

```c++
void cleanup() {
        ...
        vkDestroySurfaceKHR(instance, surface, nullptr);
        vkDestroyInstance(instance, nullptr);
        ...
    }
```

서피스는 반드시 인스턴스보다 먼저 파괴되어야 한다는 점을 명심하십시오.

## 프레젠테이션 지원 여부 쿼리

Vulkan 구현이 윈도우 시스템 통합을 지원하더라도, 시스템의 모든 장치가 이를 지원한다는 의미는 아닙니다. 따라서 `isDeviceSuitable` 함수를 확장하여 장치가 우리가 생성한 서피스로 이미지를 출력(present)할 수 있는지 확인해야 합니다. 프레젠테이션은 큐에 특화된 기능이므로, 이 문제는 결국 우리가 만든 서피스로의 프레젠테이션을 지원하는 큐 패밀리를 찾는 문제가 됩니다.

드로잉 커맨드를 지원하는 큐 패밀리와 프레젠테이션을 지원하는 큐 패밀리가 서로 겹치지 않을 수도 있습니다. 따라서 별도의 프레젠테이션 큐가 존재할 수 있다는 점을 고려하여 `QueueFamilyIndices` 구조체를 수정해야 합니다.

```c++
struct QueueFamilyIndices {
    std::optional<uint32_t> graphicsFamily;
    std::optional<uint32_t> presentFamily;

    bool isComplete() {
        return graphicsFamily.has_value() && presentFamily.has_value();
    }
};
```

다음으로, `findQueueFamilies` 함수를 수정하여 우리 윈도우 서피스로 프레젠테이션할 수 있는 큐 패밀리를 찾도록 합니다. 이를 확인하는 함수는 `vkGetPhysicalDeviceSurfaceSupportKHR`이며, 물리 장치, 큐 패밀리 인덱스, 서피스를 매개변수로 받습니다. `VK_QUEUE_GRAPHICS_BIT`를 확인하는 루프 안에서 이 함수를 호출합니다.

```c++
VkBool32 presentSupport = false;
vkGetPhysicalDeviceSurfaceSupportKHR(device, i, surface, &presentSupport);
```

그런 다음 이 불리언 값을 확인하고 프레젠테이션 큐 패밀리의 인덱스를 저장합니다.

```c++
if (presentSupport) {
    indices.presentFamily = i;
}
```

이 두 큐 패밀리가 결국 동일한 큐 패밀리로 귀결될 가능성이 매우 높지만, 이 프로그램 전반에 걸쳐 일관된 접근 방식을 위해 서로 다른 큐인 것처럼 다룰 것입니다. 그럼에도 불구하고, 성능 향상을 위해 드로잉과 프레젠테이션을 동일한 큐에서 지원하는 물리 장치를 명시적으로 선호하도록 로직을 추가할 수도 있습니다.

## 프레젠테이션 큐 생성하기

이제 남은 작업은 논리 장치 생성 절차를 수정하여 프레젠테이션 큐를 만들고 `VkQueue` 핸들을 가져오는 것입니다. 큐 핸들을 위한 멤버 변수를 추가합니다.

```c++
VkQueue presentQueue;
```

다음으로, 두 큐 패밀리로부터 큐를 생성하기 위해 여러 개의 `VkDeviceQueueCreateInfo` 구조체가 필요할 수 있습니다. 이를 우아하게 처리하는 방법은 필요한 큐에 대해 모든 고유한 큐 패밀리의 집합(set)을 만드는 것입니다.

```c++
#include <set>

...

QueueFamilyIndices indices = findQueueFamilies(physicalDevice);

std::vector<VkDeviceQueueCreateInfo> queueCreateInfos;
std::set<uint32_t> uniqueQueueFamilies = {indices.graphicsFamily.value(), indices.presentFamily.value()};

float queuePriority = 1.0f;
for (uint32_t queueFamily : uniqueQueueFamilies) {
    VkDeviceQueueCreateInfo queueCreateInfo{};
    queueCreateInfo.sType = VK_STRUCTURE_TYPE_DEVICE_QUEUE_CREATE_INFO;
    queueCreateInfo.queueFamilyIndex = queueFamily;
    queueCreateInfo.queueCount = 1;
    queueCreateInfo.pQueuePriorities = &queuePriority;
    queueCreateInfos.push_back(queueCreateInfo);
}
```

그리고 `VkDeviceCreateInfo`가 이 벡터를 가리키도록 수정합니다.

```c++
createInfo.queueCreateInfoCount = static_cast<uint32_t>(queueCreateInfos.size());
createInfo.pQueueCreateInfos = queueCreateInfos.data();
```

만약 큐 패밀리가 동일하다면, 우리는 해당 인덱스를 한 번만 전달하게 됩니다. 마지막으로, 큐 핸들을 가져오는 호출을 추가합니다.

```c++
vkGetDeviceQueue(device, indices.presentFamily.value(), 0, &presentQueue);
```

만약 그래픽스 큐 패밀리와 프레젠테이션 큐 패밀리가 같다면, 두 큐 핸들(`graphicsQueue`와 `presentQueue`)은 이제 동일한 값을 가질 가능성이 높습니다. 다음 장에서는 스왑 체인(swap chain)에 대해 알아보고, 이를 통해 어떻게 서피스에 이미지를 출력할 수 있는지 살펴보겠습니다.

[C++ 코드](/code/05_window_surface.cpp)