Vulkan에서 그리기 연산이나 메모리 전송과 같은 커맨드(command)는 함수 호출을 통해 직접 실행되지 않습니다. 대신, 수행하려는 모든 작업을 커맨드 버퍼(command buffer) 객체에 기록(record)해야 합니다. 이 방식의 장점은 Vulkan에게 무엇을 할지 알려줄 준비가 되었을 때 모든 커맨드가 함께 제출된다는 것입니다. 그러면 Vulkan은 모든 커맨드를 한 번에 사용할 수 있으므로 더 효율적으로 처리할 수 있습니다. 또한, 원한다면 여러 스레드에서 커맨드 기록을 수행할 수도 있습니다.

## 커맨드 풀 (Command pools)

커맨드 버퍼를 생성하기 전에 먼저 커맨드 풀(command pool)을 생성해야 합니다. 커맨드 풀은 버퍼를 저장하는 데 사용되는 메모리를 관리하며, 커맨드 버퍼는 이 풀에서 할당됩니다. 메인 애플리케이션 `struct`에 `vk::CommandPool`을 저장할 새 필드를 추가합니다.

```rust
struct VulkanApp {
    // ...
    command_pool: vk::CommandPool,
    // ...
}
```

그런 다음 `create_command_pool`이라는 새 메서드를 만들고, `init_vulkan`에서 프레임버퍼가 생성된 후에 호출합니다.

```rust
impl VulkanApp {
    fn init_vulkan(&mut self) {
        self.create_instance();
        self.setup_debug_messenger();
        self.create_surface();
        self.pick_physical_device();
        self.create_logical_device();
        self.create_swapchain();
        self.create_image_views();
        self.create_render_pass();
        self.create_graphics_pipeline();
        self.create_framebuffers();
        self.create_command_pool();
    }

    // ...

    fn create_command_pool(&mut self) {
        // ...
    }
}
```

커맨드 풀 생성에는 `ash`의 빌더(builder) 패턴을 사용하는 것이 편리하고 안전합니다.

```rust
let queue_family_indices = self.find_queue_families(self.physical_device);

let pool_info = vk::CommandPoolCreateInfo::builder()
    .flags(vk::CommandPoolCreateFlags::RESET_COMMAND_BUFFER)
    .queue_family_index(queue_family_indices.graphics_family.unwrap());
```

`ash`의 빌더를 사용하면 `sType` 필드가 자동으로 채워져 코드가 더 깔끔해집니다. 플래그는 타입-세이프(type-safe) 열거형으로 제공됩니다.

*   `vk::CommandPoolCreateFlags::TRANSIENT`: 커맨드 버퍼가 새로운 커맨드로 매우 자주 다시 기록될 것임을 암시합니다 (메모리 할당 동작이 변경될 수 있음).
*   `vk::CommandPoolCreateFlags::RESET_COMMAND_BUFFER`: 커맨드 버퍼를 개별적으로 다시 기록할 수 있도록 허용합니다. 이 플래그가 없으면 모든 커맨드 버퍼를 함께 리셋해야 합니다.

우리는 매 프레임마다 커맨드 버퍼를 기록할 것이므로, 이를 리셋하고 다시 기록할 수 있어야 합니다. 따라서 커맨드 풀에 `RESET_COMMAND_BUFFER` 플래그를 설정해야 합니다.

커맨드 버퍼는 우리가 가져온 그래픽스 및 프레젠테이션 큐와 같은 장치 큐 중 하나에 제출하여 실행됩니다. 각 커맨드 풀은 단일 유형의 큐에 제출되는 커맨드 버퍼만 할당할 수 있습니다. 우리는 그리기를 위한 커맨드를 기록할 것이므로 그래픽스 큐 패밀리를 선택했습니다. `Option<u32>` 타입의 큐 패밀리 인덱스는 `unwrap()`을 통해 값을 가져옵니다.

```rust
self.command_pool = unsafe {
    self.device
        .create_command_pool(&pool_info, None)
        .expect("Failed to create command pool!")
};
```

`ash`에서 생성 함수는 `Device`나 `Instance`의 메서드로 제공됩니다. C++의 `nullptr`는 Rust의 `None`에 해당합니다. Vulkan API 호출은 드라이버와의 상호작용과 유효한 상태 유지를 프로그래머에게 위임하므로 `unsafe` 블록 안에서 호출해야 합니다. `ash` 함수는 `Result`를 반환하므로 `?` 연산자나 `expect`를 사용해 에러를 처리할 수 있습니다.

커맨드 풀은 프로그램이 끝날 때 파괴되어야 합니다. Rust에서는 `Drop` 트레잇을 구현하여 리소스 정리를 자동화하는 것이 일반적입니다.

```rust
impl Drop for VulkanApp {
    fn drop(&mut self) {
        unsafe {
            self.device.destroy_command_pool(self.command_pool, None);
            // ...
        }
    }
}
```

## 커맨드 버퍼 할당

이제 커맨드 버퍼 할당을 시작할 수 있습니다. `vk::CommandBuffer` 객체를 `struct`의 필드로 추가합니다. 커맨드 버퍼는 커맨드 풀이 파괴될 때 자동으로 해제되므로, 명시적인 정리 코드가 필요하지 않습니다.

```rust
struct VulkanApp {
    // ...
    command_pool: vk::CommandPool,
    command_buffer: vk::CommandBuffer,
    // ...
}
```

이제 커맨드 풀에서 단일 커맨드 버퍼를 할당하는 `create_command_buffer` 메서드 작업을 시작하겠습니다.

```rust
impl VulkanApp {
    fn init_vulkan(&mut self) {
        // ...
        self.create_command_pool();
        self.create_command_buffer();
    }

    // ...

    fn create_command_buffer(&mut self) {
        let alloc_info = vk::CommandBufferAllocateInfo::builder()
            .command_pool(self.command_pool)
            .level(vk::CommandBufferLevel::PRIMARY)
            .command_buffer_count(1);

        self.command_buffer = unsafe {
            self.device
                .allocate_command_buffers(&alloc_info)
                .expect("Failed to allocate command buffers!")[0]
        };
    }
}
```

커맨드 버퍼는 `allocate_command_buffers` 메서드로 할당됩니다. 이 메서드는 `Vec<vk::CommandBuffer>`를 반환하므로, 하나만 할당했더라도 첫 번째 요소(`[0]`)를 가져와야 합니다.

`level` 매개변수는 할당된 커맨드 버퍼가 주(primary) 커맨드 버퍼인지 보조(secondary) 커맨드 버퍼인지를 지정합니다.

*   `vk::CommandBufferLevel::PRIMARY`: 큐에 제출하여 실행할 수 있지만, 다른 커맨드 버퍼에서 호출될 수는 없습니다.
*   `vk::CommandBufferLevel::SECONDARY`: 직접 제출할 수는 없지만, 주 커맨드 버퍼에서 호출될 수 있습니다.

여기서는 보조 커맨드 버퍼 기능을 사용하지 않겠지만, 주 커맨드 버퍼에서 공통 작업을 재사용하는 데 유용하다는 것을 상상할 수 있습니다.

## 커맨드 버퍼 기록

이제 실행하려는 커맨드를 커맨드 버퍼에 작성하는 `record_command_buffer` 메서드 작업을 시작하겠습니다. 현재 스왑체인 이미지의 인덱스를 매개변수로 받습니다.

```rust
impl VulkanApp {
    fn record_command_buffer(&self, image_index: u32) {
        // ...
    }
}
```

커맨드 버퍼 기록은 항상 `begin_command_buffer`를 호출하는 것으로 시작합니다.

```rust
let begin_info = vk::CommandBufferBeginInfo::builder();

unsafe {
    self.device
        .begin_command_buffer(self.command_buffer, &begin_info)
        .expect("Failed to begin recording command buffer!");
}
```

`flags` 매개변수는 커맨드 버퍼 사용 방식을 지정합니다. `ash`의 빌더는 기본적으로 플래그를 0으로 설정합니다.

*   `vk::CommandBufferUsageFlags::ONE_TIME_SUBMIT`: 커맨드 버퍼는 한 번 실행된 직후 다시 기록될 것입니다.
*   `vk::CommandBufferUsageFlags::RENDER_PASS_CONTINUE`: 이것은 단일 렌더 패스 내에서만 사용될 보조 커맨드 버퍼입니다.
*   `vk::CommandBufferUsageFlags::SIMULTANEOUS_USE`: 커맨드 버퍼가 이미 실행 대기 중인 상태에서도 다시 제출될 수 있습니다.

지금 우리에게는 이 플래그들 중 어느 것도 해당되지 않습니다. 커맨드 버퍼가 이미 기록되었다면 `begin_command_buffer` 호출은 암시적으로 리셋합니다.

## 렌더 패스 시작하기

그리기는 `cmd_begin_render_pass`로 렌더 패스를 시작하는 것으로 시작됩니다.

```rust
let render_pass_info = {
    let clear_color = vk::ClearValue {
        color: vk::ClearColorValue {
            float32: [0.0, 0.0, 0.0, 1.0],
        },
    };

    let render_area = vk::Rect2D {
        offset: vk::Offset2D { x: 0, y: 0 },
        extent: self.swapchain_extent,
    };

    vk::RenderPassBeginInfo::builder()
        .render_pass(self.render_pass)
        .framebuffer(self.swapchain_framebuffers[image_index as usize])
        .render_area(render_area)
        .clear_values(&[clear_color])
};
```

`ash`에서는 배열을 전달해야 하는 곳에 Rust의 슬라이스(`&[...]`)를 사용합니다. `image_index`를 사용하여 현재 스왑체인 이미지에 맞는 프레임버퍼를 선택합니다. 렌더 영역은 셰이더 로드 및 저장이 일어날 위치를 정의하며, 최상의 성능을 위해 어태치먼트 크기와 일치시키는 것이 좋습니다. 마지막으로, `VK_ATTACHMENT_LOAD_OP_CLEAR`에 사용할 소거 값을 검은색으로 설정했습니다.

```rust
unsafe {
    self.device.cmd_begin_render_pass(
        self.command_buffer,
        &render_pass_info,
        vk::SubpassContents::INLINE,
    );
}
```

커맨드를 기록하는 모든 함수는 `cmd_` 접두사를 가지며 `Device`의 메서드입니다. 반환값이 없으므로 오류 처리가 필요 없습니다. 마지막 매개변수는 다음 두 값 중 하나를 가집니다:

*   `vk::SubpassContents::INLINE`: 렌더 패스 커맨드가 주 커맨드 버퍼 자체에 포함됩니다.
*   `vk::SubpassContents::SECONDARY_COMMAND_BUFFERS`: 렌더 패스 커맨드가 보조 커맨드 버퍼에서 실행됩니다.

우리는 보조 커맨드 버퍼를 사용하지 않으므로 `INLINE`을 선택합니다.

## 기본 드로잉 커맨드

이제 그래픽스 파이프라인을 바인딩합니다.

```rust
unsafe {
    self.device.cmd_bind_pipeline(
        self.command_buffer,
        vk::PipelineBindPoint::GRAPHICS,
        self.graphics_pipeline,
    );
}
```

파이프라인의 뷰포트와 시저 상태를 동적으로 지정했으므로, 드로우 커맨드 전에 이를 설정해야 합니다.

```rust
unsafe {
    let viewport = vk::Viewport {
        x: 0.0,
        y: 0.0,
        width: self.swapchain_extent.width as f32,
        height: self.swapchain_extent.height as f32,
        min_depth: 0.0,
        max_depth: 1.0,
    };
    self.device.cmd_set_viewport(self.command_buffer, 0, &[viewport]);

    let scissor = vk::Rect2D {
        offset: vk::Offset2D { x: 0, y: 0 },
        extent: self.swapchain_extent,
    };
    self.device.cmd_set_scissor(self.command_buffer, 0, &[scissor]);
}
```
`ash`에서는 뷰포트와 시저 같은 단일 항목도 슬라이스(`&[...]`)로 전달해야 합니다.

이제 삼각형을 그리기 위한 드로우 커맨드를 실행합니다.

```rust
unsafe {
    self.device.cmd_draw(self.command_buffer, 3, 1, 0, 0);
}
```

`cmd_draw` 함수는 사전에 많은 정보를 지정했기 때문에 매우 간단합니다.

*   `vertex_count`: 정점 버퍼가 없지만, 3개의 정점을 그립니다.
*   `instance_count`: 인스턴스 렌더링에 사용됩니다. (`1`은 미사용)
*   `first_vertex`: `gl_VertexIndex`의 최솟값을 정의하는 오프셋입니다.
*   `first_instance`: `gl_InstanceIndex`의 최솟값을 정의하는 오프셋입니다.

## 마무리

렌더 패스를 종료하고 커맨드 버퍼 기록을 마칩니다.

```rust
unsafe {
    self.device.cmd_end_render_pass(self.command_buffer);

    self.device
        .end_command_buffer(self.command_buffer)
        .expect("Failed to record command buffer!");
}
```

다음 챕터에서는 메인 루프 코드를 작성할 것입니다. 이 루프는 스왑 체인에서 이미지를 가져오고, 커맨드 버퍼를 기록 및 실행한 다음, 완성된 이미지를 스왑 체인으로 반환하는 작업을 수행합니다.