## 소개

이 튜토리얼에서는 [Vulkan](https://www.khronos.org/vulkan/) 그래픽 및 컴퓨팅 API의 기초를 배웁니다. Vulkan은 [Khronos group](https://www.khronos.org/)(OpenGL으로 유명한)에서 만든 새로운 API로, 최신 그래픽 카드를 훨씬 더 잘 추상화합니다. 이 새로운 인터페이스를 통해 애플리케이션이 무엇을 하려는지 더 잘 기술할 수 있으며, 이는 [OpenGL](https://en.wikipedia.org/wiki/OpenGL)이나 [Direct3D](https://en.wikipedia.org/wiki/Direct3D)와 같은 기존 API에 비해 더 나은 성능과 예측하기 쉬운 드라이버 동작으로 이어질 수 있습니다. Vulkan의 기본 개념은 [Direct3D 12](https://en.wikipedia.org/wiki/Direct3D#Direct3D_12)나 [Metal](https://en.wikipedia.org/wiki/Metal_(API))과 유사하지만, Vulkan은 완전한 크로스플랫폼이라는 장점이 있어 Windows, Linux, Android용 개발을 동시에 할 수 있습니다.

하지만 이러한 이점을 얻기 위해 치러야 할 대가는 상당히 장황한 API를 다뤄야 한다는 것입니다. 초기 프레임 버퍼 생성, 버퍼 및 텍스처 이미지와 같은 객체에 대한 메모리 관리 등 그래픽 API와 관련된 모든 세부 사항을 애플리케이션에서 처음부터 설정해야 합니다. 그래픽 드라이버가 해주는 것들이 훨씬 적어지는데, 이는 정확한 동작을 보장하기 위해 애플리케이션에서 더 많은 작업을 해야 한다는 것을 의미합니다.

핵심은 Vulkan이 모두를 위한 것은 아니라는 점입니다. Vulkan은 고성능 컴퓨터 그래픽에 열정적이고, 기꺼이 노력을 투자할 의향이 있는 프로그래머를 대상으로 합니다. 컴퓨터 그래픽보다는 게임 개발에 더 관심이 있다면, 가까운 시일 내에 Vulkan 때문에 지원이 중단되지는 않을 OpenGL이나 Direct3D를 계속 사용하는 것이 좋습니다. 또 다른 대안은 [Unreal Engine](https://en.wikipedia.org/wiki/Unreal_Engine#Unreal_Engine_4)이나 [Unity](https://en.wikipedia.org/wiki/Unity_(game_engine))와 같은 엔진을 사용하는 것입니다. 이 엔진들은 내부적으로 Vulkan을 사용하면서도 여러분에게 훨씬 더 높은 수준의 API를 제공할 수 있습니다.

이제 이 점을 명확히 했으니, 이 튜토리얼을 따라가기 위한 몇 가지 선수 조건을 살펴보겠습니다:

*   Vulkan과 호환되는 그래픽 카드 및 드라이버 ([NVIDIA](https://developer.nvidia.com/vulkan-driver), [AMD](http://www.amd.com/en-us/innovations/software-technologies/technologies-gaming/vulkan), [Intel](https://software.intel.com/en-us/blogs/2016/03/14/new-intel-vulkan-beta-1540204404-graphics-driver-for-windows-78110-1540), [Apple Silicon (또는 Apple M1)](https://www.phoronix.com/scan.php?page=news_item&px=Apple-Silicon-Vulkan-MoltenVK))
*   C++ 경험 (RAII, 초기화 목록(initializer list)에 대한 친숙함)
*   C++17 기능을 충분히 지원하는 컴파일러 (Visual Studio 2017+, GCC 7+, 또는 Clang 5+)
*   3D 컴퓨터 그래픽에 대한 약간의 경험

이 튜토리얼은 OpenGL이나 Direct3D 개념에 대한 지식을 가정하지는 않지만, 3D 컴퓨터 그래픽의 기초는 알고 있어야 합니다. 예를 들어, 원근 투영(perspective projection)의 기저에 있는 수학은 설명하지 않을 것입니다. 컴퓨터 그래픽 개념에 대한 훌륭한 입문서로는 [이 온라인 책](https://paroj.github.io/gltut/)을 참고하세요. 그 외 다른 훌륭한 컴퓨터 그래픽 자료는 다음과 같습니다:

*   [주말 동안 레이 트레이싱 (Ray tracing in one weekend)](https://github.com/RayTracing/raytracing.github.io)
*   [물리 기반 렌더링(PBR) 책 (Physically Based Rendering book)](http://www.pbr-book.org/)
*   실제 엔진에서 Vulkan이 사용된 예시: 오픈소스 [Quake](https://github.com/Novum/vkQuake)와 [DOOM 3](https://github.com/DustinHLand/vkDOOM3)

원한다면 C++ 대신 C를 사용할 수도 있지만, 그럴 경우 다른 선형대수 라이브러리를 사용해야 하며 코드 구조화는 스스로 해결해야 합니다. 우리는 C++의 클래스나 RAII 같은 기능을 사용해 로직과 리소스의 생명 주기를 관리할 것입니다. 또한 Rust 개발자를 위한 두 가지 대체 튜토리얼 버전도 있습니다: [Vulkano 기반](https://github.com/bwasty/vulkan-tutorial-rs), [Vulkanalia 기반](https://kylemayes.github.io/vulkanalia).

다른 프로그래밍 언어를 사용하는 개발자들이 쉽게 따라올 수 있도록, 그리고 기본 API에 익숙해지기 위해 우리는 원본 C API를 사용하여 Vulkan을 다룰 것입니다. 하지만 C++를 사용한다면, 일부 궂은일들을 추상화하고 특정 유형의 오류를 방지하는 데 도움을 주는 최신 [Vulkan-Hpp](https://github.com/KhronosGroup/Vulkan-Hpp) 바인딩을 사용하는 것을 선호할 수도 있습니다.

## 전자책

이 튜토리얼을 전자책으로 읽고 싶다면, 아래 링크에서 EPUB 또는 PDF 버전을 다운로드할 수 있습니다:

*   [EPUB](https://vulkan-tutorial.com/resources/vulkan_tutorial_en.epub)
*   [PDF](https://vulkan-tutorial.com/resources/vulkan_tutorial_en.pdf)

## 튜토리얼 구조

우리는 Vulkan이 어떻게 작동하는지에 대한 개요와 화면에 첫 번째 삼각형을 띄우기 위해 해야 할 작업들을 살펴보는 것으로 시작할 것입니다. 전체 그림 속에서 각 작은 단계들의 기본적인 역할을 이해하고 나면 그 목적이 더 명확해질 것입니다. 다음으로, [Vulkan SDK](https://lunarg.com/vulkan-sdk/), 선형대수 연산을 위한 [GLM 라이브러리](http://glm.g-truc.net/), 창 생성을 위한 [GLFW](http://www.glfw.org/)로 개발 환경을 설정할 것입니다. 이 튜토리얼은 Windows의 Visual Studio와 Ubuntu Linux의 GCC에서 설정하는 방법을 다룰 것입니다.

그 후, 첫 번째 삼각형을 렌더링하는 데 필요한 Vulkan 프로그램의 모든 기본 구성 요소를 구현할 것입니다. 각 챕터는 대략 다음과 같은 구조를 따릅니다:

*   새로운 개념과 그 목적을 소개합니다
*   관련된 모든 API 호출을 사용하여 프로그램에 통합합니다
*   일부를 헬퍼 함수로 추상화합니다

각 챕터는 이전 챕터에 이어지도록 작성되었지만, 특정 Vulkan 기능을 소개하는 독립적인 문서로도 읽을 수 있습니다. 이는 이 사이트가 참고 자료로도 유용하다는 것을 의미합니다. 모든 Vulkan 함수와 타입은 사양(specification)에 링크되어 있으므로, 클릭하여 더 자세히 알아볼 수 있습니다. Vulkan은 매우 새로운 API이므로 사양 자체에 일부 미흡한 점이 있을 수 있습니다. [이 Khronos 리포지토리](https://github.com/KhronosGroup/Vulkan-Docs)에 피드백을 제출하는 것을 권장합니다.

앞서 언급했듯이, Vulkan API는 그래픽 하드웨어를 최대한 제어할 수 있도록 많은 파라미터를 가진 장황한 API를 가지고 있습니다. 이로 인해 텍스처 생성과 같은 기본 작업도 매번 반복해야 하는 많은 단계를 거치게 됩니다. 따라서 우리는 튜토리얼 전반에 걸쳐 우리만의 헬퍼 함수 모음을 만들어 나갈 것입니다.

또한 각 챕터는 해당 지점까지의 전체 코드 목록 링크로 마무리됩니다. 코드 구조에 대해 의문이 있거나, 버그를 처리하며 비교하고 싶을 때 참조할 수 있습니다. 모든 코드 파일은 여러 벤더의 그래픽 카드에서 테스트하여 정확성을 검증했습니다. 각 챕터 끝에는 댓글 섹션도 있어 특정 주제와 관련된 질문을 할 수 있습니다. 저희가 여러분을 돕기 쉽도록 플랫폼, 드라이버 버전, 소스 코드, 예상 동작 및 실제 동작을 명시해 주세요.

이 튜토리얼은 커뮤니티의 노력으로 만들어지는 것을 목표로 합니다. Vulkan은 아직 매우 새로운 API이며 모범 사례(best practice)가 완전히 정립되지 않았습니다. 튜토리얼과 사이트 자체에 대한 어떤 종류의 피드백이든 있다면, 주저하지 말고 [GitHub 리포지토리](https://github.com/Overv/VulkanTutorial)에 이슈를 제출하거나 풀 리퀘스트(pull request)를 보내주세요. 리포지토리를 'watch'하면 튜토리얼 업데이트 알림을 받을 수 있습니다.

Vulkan으로 여러분의 첫 번째 삼각형을 화면에 그리는 의식을 치른 후에는, 선형 변환, 텍스처, 3D 모델을 포함하도록 프로그램을 확장해 나갈 것입니다.

이전에 그래픽 API를 다뤄본 적이 있다면, 첫 도형이 화면에 나타나기까지 많은 단계가 있을 수 있다는 것을 알 것입니다. Vulkan에는 이러한 초기 단계가 많지만, 각각의 개별 단계는 이해하기 쉽고 불필요하게 느껴지지 않을 것입니다. 또한, 그 지루해 보이는 삼각형을 일단 그리고 나면, 완전한 텍스처를 입힌 3D 모델을 그리는 데는 그리 많은 추가 작업이 필요하지 않으며, 그 지점을 넘어선 각 단계는 훨씬 더 보람찰 것이라는 점을 명심하는 것이 중요합니다.

튜토리얼을 따라가다가 문제가 발생하면, 먼저 FAQ를 확인하여 문제와 해결책이 이미 있는지 확인해 보세요. 그래도 문제가 해결되지 않으면, 가장 관련 있는 챕터의 댓글 섹션에서 자유롭게 도움을 요청하세요.

고성능 그래픽 API의 미래로 뛰어들 준비가 되셨나요? [시작합시다!](!ko/Overview)

## 라이선스

Copyright (C) 2015-2023, Alexander Overvoorde

콘텐츠는 별도로 명시되지 않는 한 [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)에 따라 라이선스가 부여됩니다. 기여함으로써 귀하는 귀하의 기여물을 동일한 라이선스 하에 대중에게 라이선스하는 데 동의하는 것입니다.

소스 리포지토리의 `code` 디렉토리에 있는 코드 목록은 [CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/)에 따라 라이선스가 부여됩니다. 해당 디렉토리에 기여함으로써 귀하는 귀하의 기여물을 동일한 퍼블릭 도메인과 유사한 라이선스 하에 대중에게 라이선스하는 데 동의하는 것입니다.

이 프로그램은 유용할 것이라는 희망으로 배포되지만, 어떠한 보증도 없이 배포됩니다. 상품성이나 특정 목적에의 적합성에 대한 묵시적인 보증조차 없습니다.