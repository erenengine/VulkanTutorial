## 일반적인 구조

이전 장에서 여러분은 모든 설정을 마친 Vulkan 프로젝트를 만들고 예제 코드로 테스트했습니다. 이번 장에서는 다음 코드를 가지고 처음부터 시작합니다.

```rust
use ash::vk;

use std::error::Error;

struct HelloTriangleApplication {

}

impl HelloTriangleApplication {
    pub fn new() -> Self {
        HelloTriangleApplication {}
    }

    pub fn run(&mut self) -> Result<(), Box<dyn Error>> {
        self.init_vulkan()?;
        self.main_loop();
        self.cleanup();
        Ok(())
    }

    fn init_vulkan(&mut self) -> Result<(), Box<dyn Error>> {
        Ok(())
    }

    fn main_loop(&mut self) {

    }

    fn cleanup(&mut self) {

    }
}

fn main() {
    let mut app = HelloTriangleApplication::new();

    if let Err(e) = app.run() {
        eprintln!("오류 발생: {}", e);
        std::process::exit(1);
    }
}
```

먼저 `ash` 크레이트에서 Vulkan 타입, 함수, 열거형을 가져옵니다. `std::error::Error` 트레이트는 Rust의 관용적인 오류 처리 방식에 사용됩니다.

프로그램 자체는 `struct`로 감싸져 있습니다. Vulkan 객체들을 구조체 필드로 저장하고, 각 객체를 초기화하는 메서드들을 추가하여 `run` 메서드에서 순서대로 호출할 것입니다. 모든 준비가 끝나면 메인 루프에 진입하여 프레임 렌더링을 시작합니다. 잠시 후에 창이 닫힐 때까지 이벤트를 처리하는 `main_loop`를 채워 넣을 것입니다. 루프가 종료되면, `cleanup` 메서드에서 사용했던 리소스들을 반드시 해제할 것입니다.

실행 중 치명적인 오류가 발생하면, `Result<T, E>` 열거형을 반환합니다. 이 예제에서는 `Box<dyn std::error::Error>`를 사용하여 다양한 타입의 오류를 처리합니다. 오류가 발생하면 `main` 함수로 전파되어 터미널에 출력됩니다. 곧 다룰 오류의 한 예는 특정 필수 익스텐션이 지원되지 않는다는 것을 발견하는 경우입니다.

이 장 이후의 거의 모든 장에서는 `init_vulkan`에서 호출될 새로운 초기화 로직과, `cleanup`에서 마지막에 해제해야 할 하나 이상의 새로운 Vulkan 객체를 구조체 필드에 추가할 것입니다.

## 리소스 관리

C/C++에서 `malloc`으로 할당된 모든 메모리에 `free` 호출이 필요한 것처럼, 우리가 생성하는 모든 Vulkan 객체는 더 이상 필요하지 않을 때 명시적으로 파괴되어야 합니다. 하지만 Rust는 RAII(Resource Acquisition Is Initialization) 패턴을 언어 차원에서 강력하게 지원합니다. 객체의 수명은 소유권(ownership) 시스템에 의해 관리되며, 객체가 스코프를 벗어날 때 `Drop` 트레이트가 자동으로 호출됩니다.

Vulkan 핸들을 감싸는 구조체를 만들고 그 구조체에 `Drop`을 구현하면 리소스 관리를 자동화할 수 있습니다. 이는 더 큰 Rust 프로그램에서 권장되는 모델입니다.

하지만 이 튜토리얼에서는 C++ 원본의 교육적 목표를 따라, Vulkan 객체의 할당과 해제를 명시적으로 다루겠습니다. `cleanup` 메서드에서 `ash`가 제공하는 `destroy_...` 함수들을 직접 호출할 것입니다. 이를 통해 Vulkan API가 내부적으로 어떻게 동작하는지 명확하게 배울 수 있습니다.

Vulkan 객체는 `create_...` 함수로 직접 생성되거나, `allocate_...` 함수를 통해 다른 객체에서 할당됩니다. 사용이 끝난 객체는 그에 상응하는 `destroy_...` 및 `free_...` 함수로 파괴해야 합니다. 이 함수들의 매개변수는 일반적으로 객체 유형에 따라 다르지만, 마지막 매개변수로 `pAllocator`에 해당하는 인자를 받는 경우가 많습니다. 이는 사용자 정의 메모리 할당자를 위한 콜백을 지정하는 선택적 매개변수입니다. 이 튜토리얼에서는 이 매개변수를 무시하고 항상 `None`을 전달할 것입니다.

## Winit 통합

Vulkan은 오프스크린 렌더링을 위해 창 없이도 완벽하게 작동하지만, 화면에 무언가를 보여주는 것이 훨씬 더 흥미롭습니다! Rust 생태계에서는 창 생성을 위해 `winit` 라이브러리를 주로 사용합니다.

먼저 `Cargo.toml` 파일에 `ash`와 `winit` 의존성을 추가해야 합니다.

```toml
[dependencies]
ash = "0.37"
winit = "0.28"
```

C++ 버전과 달리, Rust에서는 `run` 메서드가 창과 이벤트 루프의 생명주기를 직접 관리하는 것이 더 관용적입니다. `HelloTriangleApplication` 구조체는 Vulkan 관련 객체들을 소유하고, 창(window)은 Vulkan 초기화에 필요한 정보를 제공하는 역할을 합니다.

다음과 같이 `run` 메서드를 수정하고 `main_loop`의 시그니처를 변경합니다. `init_window` 함수는 필요하지 않으며, `run` 메서드 내에서 직접 창을 생성합니다.

```rust
// use 문들을 파일 상단에 추가하세요.
use winit::event::{Event, WindowEvent};
use winit::event_loop::{ControlFlow, EventLoop};
use winit::window::{Window, WindowBuilder};

// ... HelloTriangleApplication 구조체 정의 ...

impl HelloTriangleApplication {
    pub fn run(&mut self) -> Result<(), Box<dyn Error>> {
        // 1. Winit으로 이벤트 루프와 창 생성
        let event_loop = EventLoop::new();
        let window = WindowBuilder::new()
            .with_title("Vulkan")
            .with_inner_size(winit::dpi::LogicalSize::new(WIDTH, HEIGHT))
            .with_resizable(false) // 크기 조절 비활성화
            .build(&event_loop)?;

        // 2. Vulkan 초기화
        self.init_vulkan(&window)?;

        // 3. 메인 루프 실행
        self.main_loop(event_loop, window);

        // 4. 리소스 정리
        self.cleanup();

        Ok(())
    }

    fn init_vulkan(&mut self, window: &Window) -> Result<(), Box<dyn Error>> {
        // 이 메서드는 나중에 채웁니다.
        // window 파라미터는 표면(surface) 생성에 필요합니다.
        Ok(())
    }

    fn main_loop(&mut self, event_loop: EventLoop<()>, window: Window) {
        // ...
    }
    // ...
}
```

`glfwInit()`을 호출하는 대신 `EventLoop::new()`를 호출하여 이벤트 루프를 만듭니다. `glfwWindowHint`를 설정하는 대신, `WindowBuilder`를 사용하여 창의 속성을 설정합니다. `with_resizable(false)`는 창 크기 조절을 비활성화합니다.

창의 크기를 위해 상수를 사용하는 것이 좋습니다. 구조체 정의 위에 상수를 추가합시다.

```rust
const WIDTH: u32 = 800;
const HEIGHT: u32 = 600;

struct HelloTriangleApplication {
    // ...
}
```

오류가 발생하거나 창이 닫힐 때까지 애플리케이션을 실행하려면 `main_loop`를 이벤트 처리 로직으로 채워야 합니다. Winit의 이벤트 루프는 `while` 루프 대신 클로저 기반으로 동작합니다.

```rust
fn main_loop(&mut self, event_loop: EventLoop<()>, window: Window) {
    event_loop.run(move |event, _, control_flow| {
        match event {
            Event::WindowEvent {
                event: WindowEvent::CloseRequested,
                ..
            } => {
                *control_flow = ControlFlow::Exit;
            }
            Event::MainEventsCleared => {
                // 여기서 프레임을 그리는 로직을 호출합니다.
            }
            _ => (),
        }
    });
}
```
이 코드는 `event_loop.run` 메서드를 호출하여 이벤트 처리를 시작합니다. 이 메서드는 애플리케이션이 종료될 때까지 반환되지 않습니다. 클로저 내부에서 발생하는 다양한 이벤트 (`Event`)를 `match` 문으로 처리합니다. 사용자가 창의 닫기 버튼을 누르면 `WindowEvent::CloseRequested` 이벤트가 발생하며, 이때 `control_flow`를 `ControlFlow::Exit`로 설정하여 루프를 종료시킵니다. 나중에는 `Event::MainEventsCleared` 분기에서 매 프레임을 그리는 함수를 호출하게 될 것입니다.

Winit에서는 창과 관련된 리소스가 `window` 객체의 스코프가 끝날 때 자동으로 정리됩니다. `event_loop.run`이 종료되면 `run` 메서드도 종료되고 `window`가 소멸(drop)되므로, `glfwDestroyWindow`나 `glfwTerminate`에 해당하는 명시적인 호출이 필요 없습니다. `cleanup` 메서드는 순수하게 Vulkan 리소스를 정리하는 데 사용될 것입니다. 지금은 비워둡니다.

```rust
fn cleanup(&mut self) {
    // 이 메서드는 나중에 Vulkan 객체들을 파괴하는 데 사용됩니다.
}
```

이제 프로그램을 실행하면 "Vulkan"이라는 제목의 창이 나타나고, 창을 닫아 애플리케이션이 종료될 때까지 유지됩니다. 이제 Vulkan 애플리케이션의 골격을 갖추었으니, [첫 번째 Vulkan 객체 생성하기](!ko/Drawing_a_triangle/Setup/Instance)로 넘어갑시다!

[Rust 코드](/code/rust/00_base_code.rs)