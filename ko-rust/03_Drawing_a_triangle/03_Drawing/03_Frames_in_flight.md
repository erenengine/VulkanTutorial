## 동시에 여러 프레임 렌더링하기 (Frames in flight)

현재 우리의 렌더 루프에는 한 가지 명백한 결함이 있습니다. 이전 프레임의 렌더링이 끝나기를 기다려야만 다음 프레임의 렌더링을 시작할 수 있다는 점인데, 이는 호스트(CPU)의 불필요한 유휴 상태를 유발합니다.

<!-- insert diagram showing our current render loop and the 'multi frame in flight' render loop -->

이 문제를 해결하는 방법은 여러 프레임을 동시에 *작업 중(in-flight)* 상태로 두는 것입니다. 즉, 한 프레임의 렌더링이 다음 프레임의 기록을 방해하지 않도록 하는 것입니다. 어떻게 이렇게 할 수 있을까요? 렌더링 중에 접근하고 수정하는 모든 리소스를 복제해야 합니다. 따라서 여러 개의 커맨드 버퍼, 세마포어, 펜스가 필요합니다. 이후 챕터에서는 다른 리소스들의 여러 인스턴스도 추가할 것이므로, 이 개념은 다시 등장하게 될 것입니다.

먼저 프로그램 상단에 동시에 처리할 프레임 수를 정의하는 상수를 추가합니다. Rust에서는 `usize` 타입을 사용하는 것이 일반적입니다.

```rust
const MAX_FRAMES_IN_FLIGHT: usize = 2;
```

2를 선택한 이유는 CPU가 GPU보다 *너무* 앞서 나가는 것을 원치 않기 때문입니다. 2개의 프레임이 동시 실행되면, CPU와 GPU가 동시에 각자의 작업을 처리할 수 있습니다. 만약 CPU가 먼저 작업을 마치면, GPU가 렌더링을 마칠 때까지 기다렸다가 다음 작업을 제출합니다. 3개 이상의 프레임을 사용하면 CPU가 GPU를 앞질러 지연 시간(latency)을 추가할 수 있습니다. 일반적으로 추가적인 지연 시간은 바람직하지 않습니다. 하지만 애플리케이션에 동시 실행 프레임 수를 제어할 수 있는 권한을 주는 것은 Vulkan의 명시적인(explicit) 특성을 보여주는 또 다른 예시입니다.

각 프레임은 자체적인 커맨드 버퍼, 세마포어 집합, 펜스를 가져야 합니다. 기존 객체들을 `Vec`으로 변경합니다. `App` 구조체 내의 필드들을 다음과 같이 수정합니다.

```rust
struct App {
    // ...
    command_buffers: Vec<vk::CommandBuffer>,
    image_available_semaphores: Vec<vk::Semaphore>,
    render_finished_semaphores: Vec<vk::Semaphore>,
    in_flight_fences: Vec<vk::Fence>,
    // ...
}
```

다음으로 여러 개의 커맨드 버퍼를 생성해야 합니다. `create_command_buffers` 함수를 수정합니다. `ash`의 `allocate_command_buffers` 함수는 `Vec<vk::CommandBuffer>`를 반환하므로, 반환된 벡터를 바로 필드에 할당하면 됩니다.

```rust
fn create_command_buffers(&mut self) {
    let command_buffer_allocate_info = vk::CommandBufferAllocateInfo::builder()
        .command_pool(self.command_pool)
        .level(vk::CommandBufferLevel::PRIMARY)
        .command_buffer_count(MAX_FRAMES_IN_FLIGHT as u32);

    self.command_buffers = unsafe {
        self.device
            .allocate_command_buffers(&command_buffer_allocate_info)
            .expect("Failed to allocate Command Buffers!")
    };
}
```

`create_sync_objects` 함수는 모든 동기화 객체들을 생성하도록 변경해야 합니다. Ash의 빌더 패턴을 사용하여 생성 정보를 만들고, 루프 내에서 객체들을 생성합니다.

```rust
fn create_sync_objects(&mut self) {
    self.image_available_semaphores = Vec::with_capacity(MAX_FRAMES_IN_FLIGHT);
    self.render_finished_semaphores = Vec::with_capacity(MAX_FRAMES_IN_FLIGHT);
    self.in_flight_fences = Vec::with_capacity(MAX_FRAMES_IN_FLIGHT);

    let semaphore_info = vk::SemaphoreCreateInfo::builder();
    let fence_info = vk::FenceCreateInfo::builder()
        .flags(vk::FenceCreateFlags::SIGNALED);

    for _ in 0..MAX_FRAMES_IN_FLIGHT {
        unsafe {
            let image_available_semaphore = self
                .device
                .create_semaphore(&semaphore_info, None)
                .expect("Failed to create Semaphore Object!");
            let render_finished_semaphore = self
                .device
                .create_semaphore(&semaphore_info, None)
                .expect("Failed to create Semaphore Object!");
            let in_flight_fence = self
                .device
                .create_fence(&fence_info, None)
                .expect("Failed to create Fence Object!");

            self.image_available_semaphores.push(image_available_semaphore);
            self.render_finished_semaphores.push(render_finished_semaphore);
            self.in_flight_fences.push(in_flight_fence);
        }
    }
}
```

마찬가지로, 이 객체들도 `cleanup`에서 모두 정리되어야 합니다. Rust의 `unsafe` 블록 안에서 파괴 함수를 호출합니다.

```rust
unsafe fn cleanup(&mut self) {
    for i in 0..MAX_FRAMES_IN_FLIGHT {
        self.device.destroy_semaphore(self.render_finished_semaphores[i], None);
        self.device.destroy_semaphore(self.image_available_semaphores[i], None);
        self.device.destroy_fence(self.in_flight_fences[i], None);
    }
    // ...
}
```

커맨드 버퍼는 커맨드 풀이 해제될 때 자동으로 해제되므로, 커맨드 버퍼 정리를 위해 추가로 할 일은 없습니다.

매 프레임마다 올바른 객체를 사용하기 위해, 현재 프레임을 추적해야 합니다. 이를 위해 `App` 구조체에 프레임 인덱스를 추가합니다.

```rust
struct App {
    // ...
    current_frame: usize,
    // ...
}
// App::new()에서 초기화...
current_frame: 0,
```

이제 `draw_frame` 함수를 올바른 객체들을 사용하도록 수정할 수 있습니다. `self.current_frame`을 사용하여 각 프레임에 맞는 동기화 객체와 커맨드 버퍼에 접근합니다.

```rust
fn draw_frame(&mut self) {
    let in_flight_fence = self.in_flight_fences[self.current_frame];

    unsafe {
        self.device
            .wait_for_fences(&[in_flight_fence], true, u64::MAX)
            .expect("Failed to wait for Fence!");

        self.device
            .reset_fences(&[in_flight_fence])
            .expect("Failed to reset Fence!");
    }

    let (image_index, _is_suboptimal) = unsafe {
        self.swapchain_loader
            .acquire_next_image(
                self.swapchain,
                u64::MAX,
                self.image_available_semaphores[self.current_frame],
                vk::Fence::null(),
            )
            .expect("Failed to acquire next image.")
    };
    
    let command_buffer = self.command_buffers[self.current_frame];
    unsafe {
        self.device
            .reset_command_buffer(command_buffer, vk::CommandBufferResetFlags::empty())
            .expect("Failed to reset Command Buffer!");
    }

    self.record_command_buffer(command_buffer, image_index);
    
    let wait_semaphores = [self.image_available_semaphores[self.current_frame]];
    let wait_stages = [vk::PipelineStageFlags::COLOR_ATTACHMENT_OUTPUT];
    let signal_semaphores = [self.render_finished_semaphores[self.current_frame]];
    
    let submit_infos = [vk::SubmitInfo::builder()
        .wait_semaphores(&wait_semaphores)
        .wait_dst_stage_mask(&wait_stages)
        .command_buffers(&[command_buffer])
        .signal_semaphores(&signal_semaphores)
        .build()];

    unsafe {
        self.device
            .queue_submit(self.graphics_queue, &submit_infos, in_flight_fence)
            .expect("Failed to submit draw command buffer!");
    }
    
    // ... Present KHR 로직 ...
}
```

물론, 매번 다음 프레임으로 넘어가는 것을 잊지 말아야 합니다. `draw_frame` 함수의 마지막 부분에 추가합니다.

```rust
fn draw_frame(&mut self) {
    // ...
    
    self.current_frame = (self.current_frame + 1) % MAX_FRAMES_IN_FLIGHT;
}
```

나머지(%) 연산자를 사용함으로써, 프레임 인덱스는 `MAX_FRAMES_IN_FLIGHT` 만큼의 프레임이 큐에 쌓인 후 다시 순환하게 됩니다.

이제 우리는 최대 `MAX_FRAMES_IN_FLIGHT`개의 프레임만 작업 큐에 쌓이도록 하고, 이 프레임들이 서로를 침범하지 않도록 하는 데 필요한 모든 동기화를 구현했습니다. 최종 정리(cleanup)와 같은 코드의 다른 부분에서는 `device.device_wait_idle()`처럼 더 단순한 동기화에 의존해도 괜찮다는 점에 유의하세요. 성능 요구사항에 따라 어떤 접근 방식을 사용할지 결정해야 합니다.

동기화에 대해 예제를 통해 더 배우고 싶다면, Khronos의 [이 광범위한 개요](https://github.com/KhronosGroup/Vulkan-Docs/wiki/Synchronization-Examples#swapchain-image-acquire-and-present)를 살펴보세요.

다음 챕터에서는 잘 동작하는 Vulkan 프로그램을 위해 필요한 또 다른 작은 사항을 다룰 것입니다.