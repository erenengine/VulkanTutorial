이 챕터에서는 Rust와 `ash`를 사용한 Vulkan 애플리케이션 개발 환경을 설정하고 몇 가지 유용한 라이브러리를 설치합니다. Rust의 빌드 시스템인 Cargo 덕분에 대부분의 설정은 플랫폼에 상관없이 동일하지만, Vulkan SDK 자체의 설치는 운영체제별로 다르기 때문에 여기서는 각각 따로 설명합니다.

## 공통 설정: Rust 및 Cargo 프로젝트

운영체제별 설정을 진행하기 전에, 모든 플랫폼에서 공통으로 필요한 Rust 개발 환경을 먼저 구성합니다.

### Rust 설치

아직 Rust가 설치되지 않았다면, [공식 웹사이트](https://rustup.rs/)의 안내에 따라 `rustup`을 설치하세요. `rustup`은 Rust 컴파일러(`rustc`)와 패키지 매니저 겸 빌드 시스템인 `cargo`를 관리해주는 도구입니다.

### Cargo 프로젝트 생성 및 의존성 추가

먼저, 프로젝트를 위한 새 디렉터리를 만들고 그 안에서 Cargo 프로젝트를 시작합니다.

```bash
mkdir vulkan-test-rs
cd vulkan-test-rs
cargo new .
```

이제 `Cargo.toml` 파일을 열어 필요한 라이브러리(크레이트, crate)들을 `[dependencies]` 섹션에 추가합니다.

*   **ash**: Vulkan API에 대한 저수준의 안전하지 않은 Rust 바인딩을 제공합니다.
*   **winit**: 창 생성 및 이벤트 처리를 위한 크로스플랫폼 라이브러리입니다. C++의 GLFW 역할을 합니다.
*   **glam**: Rust를 위한 간단하고 빠른 선형대수 라이브러리입니다. C++의 GLM 역할을 합니다.
*   **raw-window-handle**: `winit` 창과 Vulkan을 연결하기 위해 필요한 플랫폼별 창 핸들 정보를 제공합니다.

`Cargo.toml` 파일은 다음과 같이 보일 것입니다:

```toml
[package]
name = "vulkan-test-rs"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
ash = "0.37"
winit = "0.28"
glam = "0.24"
raw-window-handle = "0.5"
```
*(참고: 위 버전 번호는 작성 시점의 최신 버전일 수 있습니다. 실제 프로젝트에서는 최신 버전을 확인하고 사용하세요.)*

이제 각 운영체제에 맞는 Vulkan SDK 설치를 진행합니다.

## Windows

Windows에서 개발하신다면, 먼저 Vulkan SDK를 설치해야 합니다.

### Vulkan SDK

Vulkan 애플리케이션 개발에 필요한 가장 중요한 구성 요소는 SDK입니다. SDK에는 헤더, 표준 유효성 검사 레이어, 디버깅 도구, 그리고 Vulkan 함수를 위한 로더(`vulkan-1.dll`)가 포함되어 있습니다.

SDK는 [LunarG 웹사이트](https://vulkan.lunarg.com/) 페이지 하단의 버튼을 통해 다운로드할 수 있습니다.

![](/images/vulkan_sdk_download_buttons.png)

설치를 진행하고 SDK가 설치된 위치를 잘 기억해두세요. 설치가 완료되면 그래픽 카드와 드라이버가 Vulkan을 제대로 지원하는지 확인합니다. SDK 설치 경로의 `Bin` 디렉터리로 이동하여 `vkcube.exe` 데모를 실행하세요. 다음과 같은 화면이 나타나야 합니다:

![](/images/cube_demo.png)

만약 오류 메시지가 표시된다면 드라이버가 최신 버전인지, Vulkan 런타임을 포함하고 있는지, 그리고 그래픽 카드가 지원되는 모델인지 확인하세요.

### 프로젝트 테스트

이제 모든 준비가 끝났습니다. `src/main.rs` 파일을 열고 모든 내용을 아래 코드로 교체하세요. 이 코드는 `winit`으로 창을 만들고, `ash`를 통해 사용 가능한 Vulkan 확장 기능의 개수를 출력합니다.

```rust
use ash::vk;
use winit::{
    event::{Event, WindowEvent},
    event_loop::{ControlFlow, EventLoop},
    window::WindowBuilder,
};
use glam::{Mat4, Vec4};

fn main() {
    // winit으로 이벤트 루프와 창 생성
    let event_loop = EventLoop::new();
    let window = WindowBuilder::new()
        .with_title("Vulkan Window (Rust)")
        .with_inner_size(winit::dpi::LogicalSize::new(800, 600))
        .build(&event_loop)
        .expect("Failed to create window.");

    // ash로 Vulkan 로더 진입점 확보
    // unsafe: Vulkan 라이브러리를 로드하는 것은 외부 C 라이브러리와 상호작용하므로 unsafe합니다.
    let entry = unsafe { ash::Entry::load() }.expect("Failed to load Vulkan loader");

    // 사용 가능한 인스턴스 확장 기능 열거 및 개수 출력
    let extension_properties = entry
        .enumerate_instance_extension_properties()
        .expect("Failed to enumerate instance extension properties.");
    println!("{} extensions supported", extension_properties.len());

    // glam 라이브러리 테스트
    let matrix = Mat4::IDENTITY;
    let vec = Vec4::ONE;
    let _test = matrix * vec;

    // 이벤트 루프 실행
    event_loop.run(move |event, _, control_flow| {
        *control_flow = ControlFlow::Wait;

        match event {
            Event::WindowEvent {
                event: WindowEvent::CloseRequested,
                ..
            } => {
                *control_flow = ControlFlow::Exit;
            }
            _ => (),
        }
    });
}
```

이제 터미널에서 `cargo run` 명령을 실행하여 프로젝트를 컴파일하고 실행합니다.

```bash
cargo run
```

다음과 같이 터미널에 확장 기능 개수가 출력되고 빈 창이 나타나면 성공입니다.

![](/images/vs_test_window.png)

축하합니다, 이제 Rust로 Vulkan을 탐험할 모든 준비가 끝났습니다!

## Linux

이 설명은 Ubuntu, Fedora, Arch Linux 사용자를 대상으로 하지만, 자신의 배포판에 맞는 패키지 매니저 명령어로 변경하여 따라 할 수 있습니다.

### Vulkan 패키지

Linux에서 Vulkan 개발에 필요한 주요 구성 요소는 Vulkan 로더, 유효성 검사 레이어, 그리고 Vulkan 지원 여부를 테스트할 커맨드 라인 유틸리티입니다.

*   `sudo apt install vulkan-tools libvulkan-dev` 또는 `sudo dnf install vulkan-tools vulkan-loader-devel`: `vkcube`와 같은 유틸리티와 Vulkan 로더 및 개발 파일을 설치합니다.
*   `sudo apt install vulkan-validationlayers-dev spirv-tools` 또는 `sudo dnf install vulkan-validation-layers-devel`: 표준 유효성 검사 레이어와 SPIR-V 도구를 설치합니다.
*   Arch Linux: `sudo pacman -S vulkan-devel`로 필요한 모든 도구를 설치할 수 있습니다.

`winit`이 창을 생성하기 위해 필요한 시스템 라이브러리도 설치해야 합니다. Ubuntu/Debian 기준으로는 다음과 같습니다.

```bash
sudo apt install libx11-dev libxcb-randr0-dev libxcb-shape0-dev libxcb-xfixes0-dev
```

설치가 성공적으로 완료되었다면, `vkcube`를 실행하여 다음과 같은 창이 나타나는지 확인하세요:

![](/images/cube_demo_nowindow.png)

### 프로젝트 테스트

이제 공통 설정 섹션에서 작성한 Rust 프로젝트를 테스트할 차례입니다. `src/main.rs` 파일을 위 Windows 섹션의 예제 코드로 채우세요.

그리고 터미널에서 `cargo run`을 실행합니다.

```bash
cargo run
```

터미널에 확장 기능 개수가 출력되고 빈 창이 나타나면 성공입니다. 이제 Rust로 Vulkan을 탐험할 모든 준비가 끝났습니다!

## MacOS

이 설명은 Homebrew 패키지 매니저를 사용한다고 가정합니다. 또한, 최소 MacOS 버전 10.11이 필요하며, 기기가 [Metal API](https://en.wikipedia.org/wiki/Metal_(API)#Supported_GPUs)를 지원해야 합니다.

### Vulkan SDK

MacOS는 Vulkan을 네이티브로 지원하지 않으므로, LunarG의 SDK는 [MoltenVK](https://moltengl.com/)를 사용하여 Vulkan API 호출을 Apple의 Metal API 호출로 변환합니다.

[LunarG 웹사이트](https://vulkan.lunarg.com/)에서 SDK를 다운로드하여 원하는 위치에 압축을 해제하세요.

![](/images/vulkan_sdk_download_buttons.png)

압축 해제한 폴더의 `Applications` 디렉터리에서 `vkcube`를 실행하여 다음과 같은 화면이 나타나는지 확인하세요:

![](/images/cube_demo_mac.png)

### 환경 변수 설정

MoltenVK가 올바르게 동작하려면, Vulkan 로더가 필요한 파일을 찾을 수 있도록 몇 가지 환경 변수를 설정해야 합니다. 터미널에서 `cargo run`을 실행하기 전에 다음 변수들을 설정하거나, 셸 설정 파일(`.zshrc`, `.bash_profile` 등)에 추가할 수 있습니다. `path/to/your/vulkansdk` 부분은 실제 SDK 압축을 해제한 경로로 변경해야 합니다.

```bash
export VK_ICD_FILENAMES=path/to/your/vulkansdk/macOS/share/vulkan/icd.d/MoltenVK_icd.json
export VK_LAYER_PATH=path/to/your/vulkansdk/macOS/share/vulkan/explicit_layer.d
```

### 프로젝트 테스트

이제 공통 설정 섹션에서 작성한 Rust 프로젝트를 테스트할 차례입니다. `src/main.rs` 파일을 위 Windows 섹션의 예제 코드로 채우세요.

그리고 위에서 설명한 환경 변수를 설정한 터미널에서 `cargo run`을 실행합니다.

```bash
# 환경 변수를 현재 세션에만 적용하여 실행하는 예시
VK_ICD_FILENAMES=path/to/your/vulkansdk/macOS/share/vulkan/icd.d/MoltenVK_icd.json \
VK_LAYER_PATH=path/to/your/vulkansdk/macOS/share/vulkan/explicit_layer.d \
cargo run
```

터미널에 확장 기능 개수가 출력되고 빈 창이 나타나면 성공입니다. `ash`와 MoltenVK가 성공적으로 연동된 것입니다.

축하합니다, 이제 Rust로 Vulkan을 탐험할 모든 준비가 끝났습니다