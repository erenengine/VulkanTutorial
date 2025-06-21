이 챕터에서는 Vulkan 애플리케이션 개발을 위한 환경을 설정하고 몇 가지 유용한 라이브러리를 설치합니다. 컴파일러를 제외한 우리가 사용할 모든 도구는 Windows, Linux, MacOS와 호환되지만, 설치 단계가 조금씩 다르기 때문에 여기서는 각각 따로 설명합니다.

## Windows

Windows에서 개발하신다면, 코드를 컴파일하기 위해 Visual Studio를 사용한다고 가정하겠습니다. C++17을 완벽하게 지원하려면 Visual Studio 2017 또는 2019를 사용해야 합니다. 아래 설명된 단계는 VS 2017을 기준으로 작성되었습니다.

### Vulkan SDK

Vulkan 애플리케이션 개발에 필요한 가장 중요한 구성 요소는 SDK입니다. SDK에는 헤더, 표준 유효성 검사 레이어, 디버깅 도구, 그리고 Vulkan 함수를 위한 로더가 포함되어 있습니다. 로더는 런타임에 드라이버에서 함수를 찾아주는 역할을 하며, OpenGL의 GLEW와 유사하다고 생각하시면 됩니다.

SDK는 [LunarG 웹사이트](https://vulkan.lunarg.com/) 페이지 하단의 버튼을 통해 다운로드할 수 있습니다. 계정을 만들 필요는 없지만, 계정을 만들면 몇 가지 추가적인 문서에 접근할 수 있어 유용할 수 있습니다.

![](/images/vulkan_sdk_download_buttons.png)

설치를 진행하고 SDK가 설치된 위치를 잘 기억해두세요. 가장 먼저 할 일은 그래픽 카드와 드라이버가 Vulkan을 제대로 지원하는지 확인하는 것입니다. SDK를 설치한 디렉터리로 이동하여 `Bin` 디렉터리를 열고 `vkcube.exe` 데모를 실행하세요. 다음과 같은 화면이 나타나야 합니다:

![](/images/cube_demo.png)

만약 오류 메시지가 표시된다면 드라이버가 최신 버전인지, Vulkan 런타임을 포함하고 있는지, 그리고 그래픽 카드가 지원되는 모델인지 확인하세요. 주요 제조사별 드라이버 링크는 [소개 챕터](!en/Introduction)에서 확인할 수 있습니다.

이 디렉터리에는 개발에 유용한 또 다른 프로그램이 있습니다. `glslangValidator.exe`와 `glslc.exe` 프로그램은 사람이 읽을 수 있는 [GLSL](https://en.wikipedia.org/wiki/OpenGL_Shading_Language) 셰이더를 바이트코드로 컴파일하는 데 사용됩니다. 이 부분은 [셰이더 모듈](!en/Drawing_a_triangle/Graphics_pipeline_basics/Shader_modules) 챕터에서 자세히 다룰 것입니다. `Bin` 디렉터리에는 Vulkan 로더와 유효성 검사 레이어의 바이너리가 포함되어 있으며, `Lib` 디렉터리에는 라이브러리가 들어있습니다.

마지막으로 `Include` 디렉터리에는 Vulkan 헤더가 있습니다. 다른 파일들도 자유롭게 둘러보셔도 좋지만, 이 튜토리얼에서는 필요하지 않습니다.

### GLFW

앞서 언급했듯이 Vulkan 자체는 플랫폼에 독립적인 API이며 렌더링 결과를 표시할 창을 만드는 도구는 포함하지 않습니다. Vulkan의 크로스플랫폼 이점을 활용하고 Win32의 끔찍함을 피하기 위해, 우리는 [GLFW 라이브러리](http://www.glfw.org/)를 사용하여 창을 만들 것입니다. GLFW는 Windows, Linux, MacOS를 지원합니다. [SDL](https://www.libsdl.org/)과 같은 다른 라이브러리도 있지만, GLFW의 장점은 단순히 창 생성뿐만 아니라 다른 플랫폼별 Vulkan 관련 사항들도 추상화해준다는 점입니다.

[공식 웹사이트](http://www.glfw.org/download.html)에서 최신 버전의 GLFW를 찾을 수 있습니다. 이 튜토리얼에서는 64비트 바이너리를 사용하지만, 물론 32비트 모드로 빌드할 수도 있습니다. 그럴 경우 Vulkan SDK의 `Lib` 대신 `Lib32` 디렉터리에 있는 바이너리와 링크해야 합니다. 다운로드 후, 압축 파일을 적절한 위치에 푸세요. 저는 문서 폴더 아래의 Visual Studio 디렉터리에 `Libraries`라는 디렉터리를 만들었습니다.

![](/images/glfw_directory.png)

### GLM

DirectX 12와 달리 Vulkan은 선형대수 연산을 위한 라이브러리를 포함하지 않으므로, 직접 다운로드해야 합니다. [GLM](http://glm.g-truc.net/)은 그래픽 API와 함께 사용하도록 설계된 멋진 라이브러리로, OpenGL에서도 흔히 사용됩니다.

GLM은 헤더 전용 라이브러리이므로, [최신 버전](https://github.com/g-truc/glm/releases)을 다운로드하여 적절한 위치에 저장하기만 하면 됩니다. 이제 다음과 비슷한 디렉터리 구조를 갖게 될 것입니다:

![](/images/library_directory.png)

### Visual Studio 설정하기

이제 모든 종속 요소를 설치했으니, Vulkan을 위한 기본 Visual Studio 프로젝트를 설정하고 모든 것이 제대로 작동하는지 확인하기 위해 간단한 코드를 작성해 보겠습니다.

Visual Studio를 시작하고 `Windows 데스크톱 마법사` 프로젝트를 새로 만드세요. 이름을 입력하고 `확인`을 누릅니다.

![](/images/vs_new_cpp_project.png)

디버그 메시지를 출력할 공간이 있도록 애플리케이션 종류로 `콘솔 애플리케이션(.exe)`을 선택하고, Visual Studio가 상용구 코드를 추가하지 않도록 `빈 프로젝트`를 체크하세요.

![](/images/vs_application_settings.png)

`확인`을 눌러 프로젝트를 만들고 C++ 소스 파일을 추가합니다. 이미 이 과정은 알고 계시겠지만, 완전성을 위해 단계를 포함했습니다.

![](/images/vs_new_item.png)

![](/images/vs_new_source_file.png)

이제 파일에 다음 코드를 추가하세요. 지금 당장 이 코드를 이해하려고 애쓰지 마세요. 우리는 단지 Vulkan 애플리케이션을 컴파일하고 실행할 수 있는지 확인하는 중입니다. 다음 챕터부터 처음부터 다시 시작할 것입니다.

```c++
#define GLFW_INCLUDE_VULKAN
#include <GLFW/glfw3.h>

#define GLM_FORCE_RADIANS
#define GLM_FORCE_DEPTH_ZERO_TO_ONE
#include <glm/vec4.hpp>
#include <glm/mat4x4.hpp>

#include <iostream>

int main() {
    glfwInit();

    glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
    GLFWwindow* window = glfwCreateWindow(800, 600, "Vulkan window", nullptr, nullptr);

    uint32_t extensionCount = 0;
    vkEnumerateInstanceExtensionProperties(nullptr, &extensionCount, nullptr);

    std::cout << extensionCount << " extensions supported\n";

    glm::mat4 matrix;
    glm::vec4 vec;
    auto test = matrix * vec;

    while(!glfwWindowShouldClose(window)) {
        glfwPollEvents();
    }

    glfwDestroyWindow(window);

    glfwTerminate();

    return 0;
}
```

이제 오류를 없애기 위해 프로젝트를 구성해 보겠습니다. 프로젝트 속성 대화상자를 열고, 대부분의 설정이 `Debug`와 `Release` 모드 모두에 적용되므로 `모든 구성`이 선택되었는지 확인하세요.

![](/images/vs_open_project_properties.png)

![](/images/vs_all_configs.png)

`C/C++ -> 일반 -> 추가 포함 디렉터리`로 이동하여 드롭다운 상자에서 `<편집...>`을 누르세요.

![](/images/vs_cpp_general.png)

Vulkan, GLFW, GLM의 헤더 디렉터리를 추가합니다:

![](/images/vs_include_dirs.png)

다음으로, `링커 -> 일반` 아래의 라이브러리 디렉터리 편집기를 엽니다:

![](/images/vs_link_settings.png)

그리고 Vulkan과 GLFW의 오브젝트 파일 위치를 추가합니다:

![](/images/vs_link_dirs.png)

`링커 -> 입력`으로 이동하여 `추가 종속성` 드롭다운 상자에서 `<편집...>`을 누르세요.

![](/images/vs_link_input.png)

Vulkan과 GLFW 오브젝트 파일의 이름을 입력합니다:

![](/images/vs_dependencies.png)

그리고 마지막으로 컴파일러가 C++17 기능을 지원하도록 변경합니다:

![](/images/vs_cpp17.png)

이제 프로젝트 속성 대화상자를 닫아도 됩니다. 모든 것을 올바르게 설정했다면 코드에서 더 이상 오류가 강조 표시되지 않을 것입니다.

마지막으로, 실제로 64비트 모드로 컴파일하고 있는지 확인하세요:

![](/images/vs_build_mode.png)

`F5`를 눌러 프로젝트를 컴파일하고 실행하면 다음과 같이 명령 프롬프트와 창이 나타날 것입니다:

![](/images/vs_test_window.png)

확장 기능(extension)의 수가 0이 아니어야 합니다. 축하합니다, 이제 [Vulkan과 함께할](!en/Drawing_a_triangle/Setup/Base_code) 모든 준비가 끝났습니다!

## Linux

이 설명은 Ubuntu, Fedora, Arch Linux 사용자를 대상으로 하지만, 자신의 배포판에 맞는 패키지 매니저 명령어로 변경하여 따라 할 수 있습니다. C++17을 지원하는 컴파일러(GCC 7+ 또는 Clang 5+)가 필요하며, `make`도 필요합니다.

### Vulkan 패키지

Linux에서 Vulkan 애플리케이션을 개발하는 데 필요한 가장 중요한 구성 요소는 Vulkan 로더, 유효성 검사 레이어, 그리고 여러분의 컴퓨터가 Vulkan을 지원하는지 테스트할 몇 가지 커맨드 라인 유틸리티입니다:

* `sudo apt install vulkan-tools` 또는 `sudo dnf install vulkan-tools`: 커맨드 라인 유틸리티, 특히 `vulkaninfo`와 `vkcube`를 설치합니다. 이들을 실행하여 컴퓨터가 Vulkan을 지원하는지 확인하세요.
* `sudo apt install libvulkan-dev` 또는 `sudo dnf install vulkan-loader-devel`: Vulkan 로더를 설치합니다. 로더는 런타임에 드라이버에서 함수를 찾아주는 역할을 하며, OpenGL의 GLEW와 유사하다고 생각하시면 됩니다.
* `sudo apt install vulkan-validationlayers spirv-tools` 또는 `sudo dnf install mesa-vulkan-drivers vulkan-validation-layers-devel`: 표준 유효성 검사 레이어와 필요한 SPIR-V 도구를 설치합니다. 이는 Vulkan 애플리케이션을 디버깅할 때 매우 중요하며, 다음 챕터에서 다룰 것입니다.

Arch Linux에서는 `sudo pacman -S vulkan-devel`을 실행하여 위의 모든 도구를 설치할 수 있습니다.

설치가 성공적으로 완료되었다면 Vulkan 관련 부분은 모두 준비된 것입니다. `vkcube`를 실행하여 다음과 같은 창이 나타나는지 확인하는 것을 잊지 마세요:

![](/images/cube_demo_nowindow.png)

만약 오류 메시지가 표시된다면 드라이버가 최신 버전인지, Vulkan 런타임을 포함하고 있는지, 그리고 그래픽 카드가 지원되는 모델인지 확인하세요. 주요 제조사별 드라이버 링크는 [소개 챕터](!en/Introduction)에서 확인할 수 있습니다.

### X Window System and XFree86-VidModeExtension
이 라이브러리들이 시스템에 없을 수 있습니다. 없다면 다음 명령어를 사용해 설치할 수 있습니다:
* `sudo apt install libxxf86vm-dev` 또는 `dnf install libXxf86vm-devel`: XFree86-VidModeExtension에 대한 인터페이스를 제공합니다.
* `sudo apt install libxi-dev` 또는 `dnf install libXi-devel`: XINPUT 확장에 대한 X Window System 클라이언트 인터페이스를 제공합니다.

### GLFW

앞서 언급했듯이 Vulkan 자체는 플랫폼에 독립적인 API이며 렌더링 결과를 표시할 창을 만드는 도구는 포함하지 않습니다. Vulkan의 크로스플랫폼 이점을 활용하고 X11의 끔찍함을 피하기 위해, 우리는 [GLFW 라이브러리](http://www.glfw.org/)를 사용하여 창을 만들 것입니다. GLFW는 Windows, Linux, MacOS를 지원합니다. [SDL](https://www.libsdl.org/)과 같은 다른 라이브러리도 있지만, GLFW의 장점은 단순히 창 생성뿐만 아니라 다른 플랫폼별 Vulkan 관련 사항들도 추상화해준다는 점입니다.

다음 명령어를 통해 GLFW를 설치할 것입니다:

```bash
sudo apt install libglfw3-dev
```
또는
```bash
sudo dnf install glfw-devel
```
또는
```bash
sudo pacman -S glfw
```

### GLM

DirectX 12와 달리 Vulkan은 선형대수 연산을 위한 라이브러리를 포함하지 않으므로, 직접 다운로드해야 합니다. [GLM](http://glm.g-truc.net/)은 그래픽 API와 함께 사용하도록 설계된 멋진 라이브러리로, OpenGL에서도 흔히 사용됩니다.

이것은 `libglm-dev` 또는 `glm-devel` 패키지로부터 설치할 수 있는 헤더 전용 라이브러리입니다:

```bash
sudo apt install libglm-dev
```
또는
```bash
sudo dnf install glm-devel
```
또는
```bash
sudo pacman -S glm
```

### 셰이더 컴파일러

이제 거의 모든 것이 준비되었지만, 사람이 읽을 수 있는 [GLSL](https://en.wikipedia.org/wiki/OpenGL_Shading_Language)을 바이트코드로 컴파일할 프로그램이 필요합니다.

널리 사용되는 두 셰이더 컴파일러는 Khronos Group의 `glslangValidator`와 Google의 `glslc`입니다. 후자는 GCC 및 Clang과 유사한 사용법을 가지고 있으므로, 이것을 사용하겠습니다: Ubuntu에서는 Google의 [비공식 바이너리](https://github.com/google/shaderc/blob/main/downloads.md)를 다운로드하고 `glslc`를 `/usr/local/bin`에 복사하세요. 권한에 따라 `sudo`가 필요할 수 있습니다. Fedora에서는 `sudo dnf install glslc`를, Arch Linux에서는 `sudo pacman -S shaderc`를 실행하세요. 테스트하려면 `glslc`를 실행해보세요. 컴파일할 셰이더를 전달하지 않았다고 올바르게 불평할 것입니다:

`glslc: error: no input files`

`glslc`에 대해서는 [셰이더 모듈](!en/Drawing_a_triangle/Graphics_pipeline_basics/Shader_modules) 챕터에서 자세히 다룰 것입니다.

### Makefile 프로젝트 설정하기

이제 모든 종속 요소를 설치했으니, Vulkan을 위한 기본 Makefile 프로젝트를 설정하고 모든 것이 제대로 작동하는지 확인하기 위해 간단한 코드를 작성해 보겠습니다.

`VulkanTest`와 같은 이름으로 적절한 위치에 새 디렉터리를 만드세요. `main.cpp`라는 소스 파일을 만들고 다음 코드를 삽입하세요. 지금 당장 이 코드를 이해하려고 애쓰지 마세요. 우리는 단지 Vulkan 애플리케이션을 컴파일하고 실행할 수 있는지 확인하는 중입니다. 다음 챕터부터 처음부터 다시 시작할 것입니다.

```c++
#define GLFW_INCLUDE_VULKAN
#include <GLFW/glfw3.h>

#define GLM_FORCE_RADIANS
#define GLM_FORCE_DEPTH_ZERO_TO_ONE
#include <glm/vec4.hpp>
#include <glm/mat4x4.hpp>

#include <iostream>

int main() {
    glfwInit();

    glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
    GLFWwindow* window = glfwCreateWindow(800, 600, "Vulkan window", nullptr, nullptr);

    uint32_t extensionCount = 0;
    vkEnumerateInstanceExtensionProperties(nullptr, &extensionCount, nullptr);

    std::cout << extensionCount << " extensions supported\n";

    glm::mat4 matrix;
    glm::vec4 vec;
    auto test = matrix * vec;

    while(!glfwWindowShouldClose(window)) {
        glfwPollEvents();
    }

    glfwDestroyWindow(window);

    glfwTerminate();

    return 0;
}
```

다음으로, 이 기본 Vulkan 코드를 컴파일하고 실행하기 위한 Makefile을 작성하겠습니다. `Makefile`이라는 이름의 빈 파일을 만드세요. 변수나 규칙과 같은 Makefile의 기본 개념에 대해서는 이미 어느 정도 경험이 있다고 가정하겠습니다. 만약 아니라면, [이 튜토리얼](https://makefiletutorial.com/)을 통해 빠르게 익힐 수 있습니다.

먼저 파일의 나머지 부분을 단순화하기 위해 몇 가지 변수를 정의하겠습니다. 기본 컴파일러 플래그를 지정할 `CFLAGS` 변수를 정의합니다:

```make
CFLAGS = -std=c++17 -O2
```

최신 C++(`-std=c++17`)을 사용할 것이고, 최적화 수준은 O2로 설정할 것입니다. `-O2`를 제거하면 프로그램을 더 빨리 컴파일할 수 있지만, 릴리스 빌드에서는 다시 추가하는 것을 기억해야 합니다.

비슷하게, `LDFLAGS` 변수에 링커 플래그를 정의합니다:

```make
LDFLAGS = -lglfw -lvulkan -ldl -lpthread -lX11 -lXxf86vm -lXrandr -lXi
```

`-lglfw` 플래그는 GLFW를 위한 것이고, `-lvulkan`은 Vulkan 함수 로더와 링크하며, 나머지 플래그는 GLFW가 필요로 하는 저수준 시스템 라이브러리들입니다. 나머지 플래그는 GLFW 자체의 종속성인 스레딩 및 창 관리 라이브러리입니다.

`Xxf86vm`과 `Xi` 라이브러리가 아직 시스템에 설치되어 있지 않을 수 있습니다. 다음 패키지에서 찾을 수 있습니다:

```bash
sudo apt install libxxf86vm-dev libxi-dev
```
또는
```bash
sudo dnf install libXi-devel libXxf86vm-devel
```
또는
```bash
sudo pacman -S libxi libxxf86vm
```

이제 `VulkanTest`를 컴파일하는 규칙을 지정하는 것은 간단합니다. 들여쓰기는 공백 대신 탭을 사용해야 합니다.

```make
VulkanTest: main.cpp
	g++ $(CFLAGS) -o VulkanTest main.cpp $(LDFLAGS)
```

Makefile을 저장하고 `main.cpp`와 `Makefile`이 있는 디렉터리에서 `make`를 실행하여 이 규칙이 작동하는지 확인하세요. `VulkanTest` 실행 파일이 생성되어야 합니다.

이제 `test`와 `clean`이라는 두 개의 규칙을 더 정의할 것입니다. 전자는 실행 파일을 실행하고, 후자는 빌드된 실행 파일을 제거합니다:

```make
.PHONY: test clean

test: VulkanTest
	./VulkanTest

clean:
	rm -f VulkanTest
```

`make test`를 실행하면 프로그램이 성공적으로 실행되고 Vulkan 확장 기능의 수가 표시될 것입니다. 빈 창을 닫으면 애플리케이션이 성공 반환 코드(`0`)로 종료되어야 합니다. 이제 다음과 같은 완전한 Makefile을 갖게 되었을 것입니다:

```make
CFLAGS = -std=c++17 -O2
LDFLAGS = -lglfw -lvulkan -ldl -lpthread -lX11 -lXxf86vm -lXrandr -lXi

VulkanTest: main.cpp
	g++ $(CFLAGS) -o VulkanTest main.cpp $(LDFLAGS)

.PHONY: test clean

test: VulkanTest
	./VulkanTest

clean:
	rm -f VulkanTest
```

이제 이 디렉터리를 Vulkan 프로젝트의 템플릿으로 사용할 수 있습니다. 복사해서 `HelloTriangle` 같은 이름으로 바꾸고 `main.cpp`의 모든 코드를 지우세요.

이제 [진정한 모험](!en/Drawing_a_triangle/Setup/Base_code)을 떠날 준비가 모두 끝났습니다.

## MacOS

이 설명은 Xcode와 [Homebrew 패키지 매니저](https://brew.sh/)를 사용한다고 가정합니다. 또한, 최소 MacOS 버전 10.11이 필요하며, 사용하시는 기기가 [Metal API](https://en.wikipedia.org/wiki/Metal_(API)#Supported_GPUs)를 지원해야 한다는 점을 명심하세요.

### Vulkan SDK

Vulkan 애플리케이션 개발에 필요한 가장 중요한 구성 요소는 SDK입니다. SDK에는 헤더, 표준 유효성 검사 레이어, 디버깅 도구, 그리고 Vulkan 함수를 위한 로더가 포함되어 있습니다. 로더는 런타임에 드라이버에서 함수를 찾아주는 역할을 하며, OpenGL의 GLEW와 유사하다고 생각하시면 됩니다.

SDK는 [LunarG 웹사이트](https://vulkan.lunarg.com/) 페이지 하단의 버튼을 통해 다운로드할 수 있습니다. 계정을 만들 필요는 없지만, 계정을 만들면 몇 가지 추가적인 문서에 접근할 수 있어 유용할 수 있습니다.

![](/images/vulkan_sdk_download_buttons.png)

MacOS용 SDK 버전은 내부적으로 [MoltenVK](https://moltengl.com/)를 사용합니다. MacOS는 Vulkan을 네이티브로 지원하지 않으므로, MoltenVK는 Vulkan API 호출을 Apple의 Metal 그래픽 프레임워크로 변환하는 레이어 역할을 합니다. 이를 통해 Apple의 Metal 프레임워크가 제공하는 디버깅 및 성능 이점을 활용할 수 있습니다.

다운로드 후, 내용물을 원하는 폴더에 압축 해제하세요 (Xcode에서 프로젝트를 만들 때 이 경로를 참조해야 하므로 잘 기억해두세요). 압축 해제한 폴더 안의 `Applications` 폴더에 SDK를 사용하는 몇 가지 데모를 실행할 수 있는 실행 파일들이 있습니다. `vkcube` 실행 파일을 실행하면 다음과 같은 화면이 나타날 것입니다:

![](/images/cube_demo_mac.png)

### GLFW

앞서 언급했듯이 Vulkan 자체는 플랫폼에 독립적인 API이며 렌더링 결과를 표시할 창을 만드는 도구는 포함하지 않습니다. 우리는 [GLFW 라이브러리](http://www.glfw.org/)를 사용하여 창을 만들 것입니다. GLFW는 Windows, Linux, MacOS를 지원합니다. [SDL](https://www.libsdl.org/)과 같은 다른 라이브러리도 있지만, GLFW의 장점은 단순히 창 생성뿐만 아니라 다른 플랫폼별 Vulkan 관련 사항들도 추상화해준다는 점입니다.

MacOS에 GLFW를 설치하기 위해 Homebrew 패키지 매니저를 사용하여 `glfw` 패키지를 설치합니다:

```bash
brew install glfw
```

### GLM

Vulkan은 선형대수 연산을 위한 라이브러리를 포함하지 않으므로, 직접 다운로드해야 합니다. [GLM](http://glm.g-truc.net/)은 그래픽 API와 함께 사용하도록 설계된 멋진 라이브러리로, OpenGL에서도 흔히 사용됩니다.

이것은 `glm` 패키지로부터 설치할 수 있는 헤더 전용 라이브러리입니다:

```bash
brew install glm
```

### Xcode 설정하기

이제 모든 종속 요소를 설치했으니, Vulkan을 위한 기본 Xcode 프로젝트를 설정해 보겠습니다. 여기 설명의 대부분은 모든 종속성을 프로젝트에 연결하기 위한 '배관' 작업과 같습니다. 또한, 다음 설명에서 `vulkansdk` 폴더를 언급할 때는 Vulkan SDK를 압축 해제한 폴더를 가리킨다는 점을 기억하세요.

Xcode를 시작하고 새 Xcode 프로젝트를 만듭니다. 열리는 창에서 Application > Command Line Tool을 선택하세요.

![](/images/xcode_new_project.png)

`Next`를 선택하고, 프로젝트 이름을 작성한 후 `Language`로 `C++`를 선택하세요.

![](/images/xcode_new_project_2.png)

`Next`를 누르면 프로젝트가 생성됩니다. 이제 생성된 `main.cpp` 파일의 코드를 다음 코드로 변경합시다:

```c++
#define GLFW_INCLUDE_VULKAN
#include <GLFW/glfw3.h>

#define GLM_FORCE_RADIANS
#define GLM_FORCE_DEPTH_ZERO_TO_ONE
#include <glm/vec4.hpp>
#include <glm/mat4x4.hpp>

#include <iostream>

int main() {
    glfwInit();

    glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
    GLFWwindow* window = glfwCreateWindow(800, 600, "Vulkan window", nullptr, nullptr);

    uint32_t extensionCount = 0;
    vkEnumerateInstanceExtensionProperties(nullptr, &extensionCount, nullptr);

    std::cout << extensionCount << " extensions supported\n";

    glm::mat4 matrix;
    glm::vec4 vec;
    auto test = matrix * vec;

    while(!glfwWindowShouldClose(window)) {
        glfwPollEvents();
    }

    glfwDestroyWindow(window);

    glfwTerminate();

    return 0;
}
```

아직 이 코드가 무엇을 하는지 전부 이해할 필요는 없습니다. 우리는 단지 모든 것이 작동하는지 확인하기 위해 몇 가지 API 호출을 설정하는 중입니다.

Xcode는 이미 찾을 수 없는 라이브러리 같은 오류들을 보여주고 있을 것입니다. 이제 그 오류들을 없애기 위해 프로젝트 설정을 시작하겠습니다. *프로젝트 탐색기(Project Navigator)* 패널에서 프로젝트를 선택하세요. *Build Settings* 탭을 연 다음:

* **Header Search Paths** 필드를 찾아 `/usr/local/include` (Homebrew가 헤더를 설치하는 위치이므로 glm과 glfw3 헤더 파일이 있어야 합니다)와 Vulkan 헤더를 위한 `vulkansdk/macOS/include` 링크를 추가하세요.
* **Library Search Paths** 필드를 찾아 `/usr/local/lib` (마찬가지로 Homebrew가 라이브러리를 설치하는 위치이므로 glm과 glfw3 라이브러리 파일이 있어야 합니다)와 `vulkansdk/macOS/lib` 링크를 추가하세요.

다음과 같이 보일 것입니다 (물론, 파일을 어디에 두었느냐에 따라 경로는 달라집니다):

![](/images/xcode_paths.png)

이제 *Build Phases* 탭의 **Link Binary With Libraries**에 `glfw3`와 `vulkan` 프레임워크를 모두 추가할 것입니다. 작업을 쉽게 하기 위해 동적 라이브러리를 프로젝트에 추가할 것입니다 (정적 프레임워크를 사용하고 싶다면 해당 라이브러리의 문서를 확인하세요).

* glfw의 경우 `/usr/local/lib` 폴더를 열면 `libglfw.3.x.dylib`과 같은 이름의 파일이 있을 것입니다 ("x"는 라이브러리의 버전 번호이며, Homebrew에서 패키지를 다운로드한 시점에 따라 다를 수 있습니다). 이 파일을 Xcode의 Linked Frameworks and Libraries 탭으로 드래그 앤 드롭하세요.
* vulkan의 경우 `vulkansdk/macOS/lib`로 이동하세요. `libvulkan.1.dylib`와 `libvulkan.1.x.xx.dylib` 두 파일에 대해 동일한 작업을 수행하세요 ("x"는 다운로드한 SDK의 버전 번호입니다).

이 라이브러리들을 추가한 후, 같은 탭의 **Copy Files**에서 `Destination`을 "Frameworks"로 변경하고, 하위 경로는 비우고 "Copy only when installing"을 선택 해제하세요. "+" 기호를 클릭하고 이 세 프레임워크를 여기에 모두 추가하세요.

Xcode 설정은 다음과 같을 것입니다:

![](/images/xcode_frameworks.png)

마지막으로 설정해야 할 것은 몇 가지 환경 변수입니다. Xcode 툴바에서 `Product` > `Scheme` > `Edit Scheme...`으로 이동하여, `Arguments` 탭에 다음 두 환경 변수를 추가하세요:

* VK_ICD_FILENAMES = `vulkansdk/macOS/share/vulkan/icd.d/MoltenVK_icd.json`
* VK_LAYER_PATH = `vulkansdk/macOS/share/vulkan/explicit_layer.d`

다음과 같이 보일 것입니다:

![](/images/xcode_variables.png)

드디어 모든 준비가 끝났습니다! 이제 프로젝트를 실행하면 (선택한 구성에 따라 빌드 구성을 Debug 또는 Release로 설정하는 것을 잊지 마세요) 다음과 같은 화면이 나타날 것입니다:

![](/images/xcode_output.png)

확장 기능(extension)의 수가 0이 아니어야 합니다. 다른 로그들은 라이브러리에서 나온 것이며, 설정에 따라 다른 메시지를 받을 수도 있습니다.

이제 [진짜배기](!en/Drawing_a_triangle/Setup/Base_code)를 위한 모든 준비가 끝났습니다.