이번 장에서는 모든 것을 하나로 합칠 시간입니다. 메인 루프에서 호출되어 삼각형을 화면에 그리는 `draw_frame` 함수를 작성할 것입니다. 먼저 함수를 만들고 `main_loop`에서 호출해 봅시다.

```rust
fn main_loop(&mut self) {
    self.event_loop
        .run(move |event, _, control_flow| {
            // ... 이벤트 처리 ...
            match event {
                // ...
                Event::MainEventsCleared => {
                    self.window.request_redraw();
                }
                Event::RedrawRequested(_) => {
                    self.draw_frame();
                }
                // ...
            }
        })
}

// ...

impl HelloTriangleApplication {
    fn draw_frame(&mut self) {
        // 여기에 렌더링 로직을 작성합니다.
    }
}
```
*참고: Rust에서는 이벤트 루프 기반으로 동작하므로, C++의 `while` 루프 대신 `winit`의 `RedrawRequested` 이벤트 핸들러 내에서 `draw_frame`을 호출하는 것이 일반적입니다.*

## 프레임의 개요

높은 수준에서 Vulkan으로 프레임을 렌더링하는 것은 다음과 같은 공통된 단계로 구성됩니다.

*   이전 프레임이 끝나기를 기다립니다.
*   스왑 체인에서 이미지를 가져옵니다.
*   가져온 이미지에 장면을 그리는 커맨드 버퍼를 기록합니다.
*   기록된 커맨드 버퍼를 제출합니다.
*   스왑 체인 이미지를 제시(present)합니다.

이후 장에서 드로잉 함수를 더 확장하겠지만, 지금으로서는 이것이 우리 렌더링 루프의 핵심입니다.

<!-- 프레임 개요를 보여주는 이미지 추가 -->

## 동기화

<!-- 동기화를 보여주는 이미지 추가 -->

Vulkan의 핵심 설계 철학 중 하나는 GPU에서의 실행 동기화가 명시적이라는 것입니다. (이하 동기화에 대한 개념 설명은 C++ 버전과 동일하므로 생략하고, Rust/Ash 구현에 초점을 맞춥니다.)

... (세마포어와 펜스에 대한 개념 설명) ...

### 무엇을 선택해야 할까?

우리에게는 두 가지 동기화 프리미티브가 있고, 마침 동기화를 적용할 두 곳이 있습니다. 스왑 체인 작업과 이전 프레임이 끝나기를 기다리는 것입니다. 스왑 체인 작업은 GPU에서 발생하므로 세마포어를 사용하고, 이전 프레임이 끝나기를 기다리는 작업은 CPU(호스트)가 기다려야 하므로 펜스를 사용합니다. 이는 GPU가 커맨드 버퍼를 사용하는 동안 CPU가 해당 커맨드 버퍼를 덮어쓰지 않도록 보장하기 위함입니다.

## 동기화 객체 생성하기

단일 프레임만 처리하는 대신, 여러 프레임이 동시에 처리 중(in-flight)일 수 있는 보다 일반적인 접근 방식을 사용하겠습니다. 이를 통해 GPU가 하나의 프레임을 렌더링하는 동안 CPU는 다음 프레임을 준비할 수 있어 성능이 향상됩니다. `MAX_FRAMES_IN_FLIGHT` 상수를 정의하고, 각 프레임에 대한 동기화 객체 세트를 생성합니다.

```rust
const MAX_FRAMES_IN_FLIGHT: usize = 2;

// AppData 구조체에 추가
struct AppData {
    // ...
    image_available_semaphores: Vec<vk::Semaphore>,
    render_finished_semaphores: Vec<vk::Semaphore>,
    in_flight_fences: Vec<vk::Fence>,
    // ...
}

// HelloTriangleApplication 구조체에 추가
struct HelloTriangleApplication {
    // ...
    current_frame: usize,
    // ...
}
```

이제 동기화 객체를 생성하는 `create_sync_objects` 함수를 만들어 봅시다.

```rust
// main.rs
impl HelloTriangleApplication {
    pub fn new(event_loop: &EventLoop<()>) -> Self {
        // ...
        let command_buffers =
            create_command_buffers(&device, &command_pool, &graphics_pipeline, &framebuffers, &render_pass, &app_data);

        let (
            image_available_semaphores,
            render_finished_semaphores,
            in_flight_fences,
        ) = create_sync_objects(&device);

        let mut app_data = AppData {
            // ...
            image_available_semaphores,
            render_finished_semaphores,
            in_flight_fences,
        };

        Self {
            // ...
            app_data,
            current_frame: 0,
        }
    }
}

// create.rs 또는 적절한 모듈
fn create_sync_objects(
    device: &ash::Device,
) -> (
    Vec<vk::Semaphore>,
    Vec<vk::Semaphore>,
    Vec<vk::Fence>,
) {
    let mut image_available_semaphores = Vec::with_capacity(MAX_FRAMES_IN_FLIGHT);
    let mut render_finished_semaphores = Vec::with_capacity(MAX_FRAMES_IN_FLIGHT);
    let mut in_flight_fences = Vec::with_capacity(MAX_FRAMES_IN_FLIGHT);

    let semaphore_create_info = vk::SemaphoreCreateInfo::builder();

    let fence_create_info = vk::FenceCreateInfo::builder()
        .flags(vk::FenceCreateFlags::SIGNALED); // 첫 프레임에서 바로 통과하도록 신호된 상태로 생성

    for _ in 0..MAX_FRAMES_IN_FLIGHT {
        unsafe {
            let image_available_semaphore = device
                .create_semaphore(&semaphore_create_info, None)
                .expect("Failed to create Semaphore Object!");
            let render_finished_semaphore = device
                .create_semaphore(&semaphore_create_info, None)
                .expect("Failed to create Semaphore Object!");
            let in_flight_fence = device
                .create_fence(&fence_create_info, None)
                .expect("Failed to create Fence Object!");

            image_available_semaphores.push(image_available_semaphore);
            render_finished_semaphores.push(render_finished_semaphore);
            in_flight_fences.push(in_flight_fence);
        }
    }

    (
        image_available_semaphores,
        render_finished_semaphores,
        in_flight_fences,
    )
}
```
`ash`에서는 `vk::...CreateInfo::builder()` 패턴을 사용하여 구조체를 생성합니다. `vkCreate...` 함수 호출은 `unsafe` 블록 안에서 이루어지며, Rust의 `Result` 타입을 반환하므로 `expect`를 사용해 오류를 처리합니다.

생성된 객체들은 `Drop` 트레잇 구현에서 정리해야 합니다.

```rust
// main.rs
impl Drop for HelloTriangleApplication {
    fn drop(&mut self) {
        unsafe {
            self.device.device_wait_idle().unwrap();

            for i in 0..MAX_FRAMES_IN_FLIGHT {
                self.device.destroy_semaphore(self.app_data.image_available_semaphores[i], None);
                self.device.destroy_semaphore(self.app_data.render_finished_semaphores[i], None);
                self.device.destroy_fence(self.app_data.in_flight_fences[i], None);
            }
            // ... 다른 리소스 정리 ...
        }
    }
}
```

## `draw_frame` 구현하기

### 이전 프레임 기다리기

프레임 시작 시, 현재 프레임에 할당된 펜스를 기다려 이전 작업이 완료되었는지 확인합니다.

```rust
fn draw_frame(&mut self) {
    let fence = self.app_data.in_flight_fences[self.current_frame];
    
    unsafe {
        self.device
            .wait_for_fences(&[fence], true, u64::MAX)
            .expect("Failed to wait for Fence!");
    }
}
```
`wait_for_fences`는 펜스 슬라이스(`&[fence]`)를 인자로 받습니다. `true`는 모든 펜스를 기다리겠다는 의미이며, 타임아웃은 `u64::MAX`로 설정하여 비활성화합니다.

*참고: C++ 버전의 첫 프레임 교착 상태 문제는 펜스를 `vk::FenceCreateFlags::SIGNALED` 플래그와 함께 생성하여 해결했습니다. `wait_for_fences`가 첫 호출에서 즉시 반환될 것입니다.*

### 스왑 체인에서 이미지 가져오기

다음으로 스왑 체인에서 렌더링할 이미지를 가져옵니다. 이 작업이 완료되면 `image_available_semaphores`가 신호를 받습니다.

```rust
// draw_frame 함수 내
let image_available_semaphore = self.app_data.image_available_semaphores[self.current_frame];

let (image_index, _is_suboptimal) = unsafe {
    self.swapchain_loader
        .acquire_next_image(
            self.app_data.swapchain,
            u64::MAX,
            image_available_semaphore,
            vk::Fence::null(),
        )
        .expect("Failed to acquire next image.")
};

// 펜스를 기다린 후에는 재사용하기 전에 반드시 리셋해야 합니다.
unsafe {
    self.device
        .reset_fences(&[fence])
        .expect("Failed to reset Fence!");
}
```
`acquire_next_image`는 이미지 인덱스와 스왑체인이 최적 상태가 아님을 나타내는 bool 값을 튜플로 반환합니다. 지금은 `_is_suboptimal` 값을 무시하지만, 창 크기 조절 등을 처리할 때 중요해집니다. 펜스는 이미지 획득 *후*에 리셋하여 CPU-GPU 병렬 실행을 극대화할 수 있습니다.

### 커맨드 버퍼 기록 및 제출

이제 이미지를 사용할 수 있으므로, 해당 이미지에 그리는 커맨드 버퍼를 다시 기록하고 제출합니다.

```rust
// draw_frame 함수 내
let command_buffer = self.command_buffers[image_index as usize];

unsafe {
    self.device
        .reset_command_buffer(command_buffer, vk::CommandBufferResetFlags::empty())
        .expect("Failed to reset Command Buffer!");
}

// 이전 장에서 만든 함수를 호출
record_command_buffer(
    &self.device,
    command_buffer,
    self.app_data.render_pass,
    self.app_data.framebuffers[image_index as usize],
    self.app_data.graphics_pipeline,
    self.app_data.swapchain_extent,
);

let wait_semaphores = [image_available_semaphore];
let wait_stages = [vk::PipelineStageFlags::COLOR_ATTACHMENT_OUTPUT];
let signal_semaphores = [self.app_data.render_finished_semaphores[self.current_frame]];

let submit_infos = [vk::SubmitInfo::builder()
    .wait_semaphores(&wait_semaphores)
    .wait_dst_stage_mask(&wait_stages)
    .command_buffers(&[command_buffer])
    .signal_semaphores(&signal_semaphores)
    .build()];

unsafe {
    self.device
        .queue_submit(self.graphics_queue, &submit_infos, fence)
        .expect("Failed to submit draw command buffer!");
}
```
`ash`의 빌더 패턴을 사용하여 `VkSubmitInfo`를 간결하게 생성합니다. `queue_submit`의 마지막 인자로 `fence`를 전달하여, 이 작업이 끝나면 해당 펜스에 신호를 보내도록 합니다.

### 서브패스 종속성

이미지 레이아웃 전환을 올바른 시점에 수행하기 위해 서브패스 종속성을 설정해야 합니다. `acquire_next_image`가 완료된 후, 그리고 우리가 이미지에 색상을 쓰기 시작하기 전에 전환이 일어나야 합니다. `create_render_pass` 함수를 수정합니다.

```rust
// create.rs 또는 적절한 모듈
let dependency = vk::SubpassDependency::builder()
    .src_subpass(vk::SUBPASS_EXTERNAL)
    .dst_subpass(0)
    .src_stage_mask(vk::PipelineStageFlags::COLOR_ATTACHMENT_OUTPUT)
    .src_access_mask(vk::AccessFlags::empty())
    .dst_stage_mask(vk::PipelineStageFlags::COLOR_ATTACHMENT_OUTPUT)
    .dst_access_mask(vk::AccessFlags::COLOR_ATTACHMENT_WRITE)
    .build();

let dependencies = [dependency];

let render_pass_info = vk::RenderPassCreateInfo::builder()
    .attachments(&attachments)
    .subpasses(&subpasses)
    .dependencies(&dependencies); // 종속성 추가
```
이 종속성은 외부(스왑체인 이미지 획득)에서 우리의 첫 번째 서브패스(인덱스 0)로의 전환을 제어합니다. `COLOR_ATTACHMENT_OUTPUT` 단계에서 쓰기 작업(`COLOR_ATTACHMENT_WRITE`)을 시작하기 직전에 전환이 발생하도록 보장합니다.

### 프레젠테이션

마지막으로 렌더링된 이미지를 화면에 표시하기 위해 프레젠테이션 큐에 제출합니다.

```rust
// draw_frame 함수 내
let swapchains = [self.app_data.swapchain];
let image_indices = [image_index];

let present_info = vk::PresentInfoKHR::builder()
    .wait_semaphores(&signal_semaphores) // 렌더링이 끝나면 신호를 받는 세마포어를 기다림
    .swapchains(&swapchains)
    .image_indices(&image_indices);

unsafe {
    self.swapchain_loader
        .queue_present(self.present_queue, &present_info)
        .expect("Failed to present queue.");
}

// 다음 프레임을 위해 프레임 인덱스를 업데이트
self.current_frame = (self.current_frame + 1) % MAX_FRAMES_IN_FLIGHT;
```
`queue_present`는 렌더링이 완료되었음을 알리는 `render_finished_semaphore`를 기다린 후 실행됩니다. 마지막으로 `current_frame` 인덱스를 순환시켜 다음 `draw_frame` 호출에서 다음 프레임의 동기화 객체 세트를 사용하도록 합니다.

### 유휴 상태 대기

프로그램이 종료될 때, 모든 비동기 작업이 완료될 때까지 기다려야 리소스를 안전하게 해제할 수 있습니다. `main_loop`가 끝난 후와 `Drop` 구현의 시작 부분에서 `device_wait_idle`을 호출합니다.

```rust
// main_loop에서 이벤트 루프가 끝난 후 (실제로는 winit의 클로저 밖)
// 또는 Drop 구현에서
unsafe {
    self.device.device_wait_idle().unwrap();
}
```
이제 프로그램을 실행하면 화면에 삼각형이 나타나고, 창을 닫아도 유효성 검사 오류 없이 깔끔하게 종료될 것입니다.

## 결론

상당한 양의 코드를 통해 드디어 화면에 무언가를 띄웠습니다! Vulkan의 부트스트래핑 과정은 복잡하지만, 그만큼 명시성을 통해 강력한 제어권을 얻을 수 있습니다. 지금까지 작성한 코드를 다시 살펴보며 각 Vulkan 객체의 역할과 상호 관계를 이해하는 시간을 갖는 것이 좋습니다.

다음 장에서는 여러 프레임을 동시에 처리하는 렌더링 루프를 더욱 견고하게 만들고 창 크기 변경과 같은 예외 상황을 처리하는 방법을 알아보겠습니다.