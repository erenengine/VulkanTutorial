## 소개

지금 우리가 사용하는 정점 버퍼는 올바르게 작동하지만, CPU에서 접근할 수 있도록 하는 메모리 타입이 그래픽 카드 자체에서 읽기에 가장 최적의 메모리 타입은 아닐 수 있습니다. 가장 최적화된 메모리는 `vk::MemoryPropertyFlags::DEVICE_LOCAL` 플래그를 가지며, 보통 외장 그래픽 카드에서는 CPU가 접근할 수 없습니다. 이번 장에서는 두 개의 정점 버퍼를 만들 것입니다. 하나는 정점 배열의 데이터를 업로드하기 위한 CPU 접근 가능 메모리의 *스테이징 버퍼(staging buffer)*이고, 다른 하나는 디바이스 로컬 메모리에 있는 최종 정점 버퍼입니다. 그런 다음 버퍼 복사 명령을 사용해 스테이징 버퍼의 데이터를 실제 정점 버퍼로 이동시킬 것입니다.

## 전송 큐 (Transfer queue)

버퍼 복사 명령은 `vk::QueueFlags::TRANSFER`를 지원하는 큐 패밀리(queue family)를 필요로 합니다. 좋은 소식은 `vk::QueueFlags::GRAPHICS`나 `vk::QueueFlags::COMPUTE` 기능을 가진 모든 큐 패밀리는 이미 암시적으로 `vk::QueueFlags::TRANSFER` 연산을 지원한다는 것입니다. 이런 경우 구현체는 `queue_flags`에 이 비트를 명시적으로 표시하지 않아도 됩니다.

만약 도전해보고 싶다면, 전송 연산만을 위한 별도의 큐 패밀리를 사용해볼 수도 있습니다. 이를 위해서는 프로그램에 다음과 같은 수정이 필요합니다.

*   `QueueFamilyIndices` 구조체와 `find_queue_families` 함수를 수정하여, `GRAPHICS` 플래그는 없지만 `TRANSFER` 플래그를 가진 큐 패밀리를 명시적으로 찾도록 합니다.
*   `create_logical_device`를 수정하여 전송 큐에 대한 핸들을 요청합니다.
*   전송 큐 패밀리에서 제출될 커맨드 버퍼를 위한 두 번째 커맨드 풀(command pool)을 생성합니다.
*   리소스의 `sharing_mode`를 `vk::SharingMode::CONCURRENT`로 변경하고, 큐 패밀리 인덱스 슬라이스(`&[u32]`)에 그래픽 큐와 전송 큐 패밀리를 모두 지정합니다.
*   `cmd_copy_buffer`와 같은 모든 전송 명령을 그래픽 큐가 아닌 전송 큐에 제출합니다.

약간의 작업이 필요하지만, 이를 통해 큐 패밀리 간에 리소스를 어떻게 공유하는지에 대해 많은 것을 배울 수 있을 것입니다.

## 버퍼 생성 추상화

이번 장에서는 여러 버퍼를 생성할 것이므로, 버퍼 생성을 헬퍼 함수로 옮기는 것이 좋습니다. `create_buffer`라는 새 함수를 만들고, `create_vertex_buffer`에 있던 코드(매핑 제외)를 이 함수로 옮기세요. Rust에서는 출력 매개변수 대신 튜플을 반환하는 것이 일반적입니다.

```rust
fn create_buffer(
    &self,
    size: vk::DeviceSize,
    usage: vk::BufferUsageFlags,
    properties: vk::MemoryPropertyFlags,
) -> (vk::Buffer, vk::DeviceMemory) {
    let buffer_info = vk::BufferCreateInfo::builder()
        .size(size)
        .usage(usage)
        .sharing_mode(vk::SharingMode::EXCLUSIVE);

    let buffer = unsafe {
        self.device
            .create_buffer(&buffer_info, None)
            .expect("failed to create buffer!")
    };

    let mem_requirements = unsafe { self.device.get_buffer_memory_requirements(buffer) };

    let alloc_info = vk::MemoryAllocateInfo::builder()
        .allocation_size(mem_requirements.size)
        .memory_type_index(self.find_memory_type(mem_requirements.memory_type_bits, properties));

    let buffer_memory = unsafe {
        self.device
            .allocate_memory(&alloc_info, None)
            .expect("failed to allocate buffer memory!")
    };

    unsafe {
        self.device
            .bind_buffer_memory(buffer, buffer_memory, 0)
            .expect("failed to bind buffer memory!");
    }

    (buffer, buffer_memory)
}
```

다양한 종류의 버퍼를 생성하는 데 이 함수를 사용할 수 있도록 버퍼 크기, 메모리 속성, 사용 목적을 매개변수로 받습니다. 함수는 생성된 버퍼와 메모리 핸들을 튜플로 반환합니다.

이제 `create_vertex_buffer`에서 버퍼 생성 및 메모리 할당 코드를 제거하고, 대신 `create_buffer`를 호출할 수 있습니다.

```rust
fn create_vertex_buffer(&mut self) {
    let buffer_size = (std::mem::size_of::<Vertex>() * self.vertices.len()) as vk::DeviceSize;

    let (vertex_buffer, vertex_buffer_memory) = self.create_buffer(
        buffer_size,
        vk::BufferUsageFlags::VERTEX_BUFFER,
        vk::MemoryPropertyFlags::HOST_VISIBLE | vk::MemoryPropertyFlags::HOST_COHERENT,
    );
    self.vertex_buffer = vertex_buffer;
    self.vertex_buffer_memory = vertex_buffer_memory;

    let data_ptr = unsafe {
        self.device
            .map_memory(
                self.vertex_buffer_memory,
                0,
                buffer_size,
                vk::MemoryMapFlags::empty(),
            )
            .expect("failed to map vertex buffer memory!")
    };

    unsafe {
        let mut align = ash::util::Align::new(
            data_ptr,
            std::mem::align_of::<Vertex>() as u64,
            buffer_size,
        );
        align.copy_from_slice(&self.vertices);
        self.device.unmap_memory(self.vertex_buffer_memory);
    }
}
```

프로그램을 실행하여 정점 버퍼가 여전히 제대로 작동하는지 확인하세요.

## 스테이징 버퍼 사용하기

이제 `create_vertex_buffer` 함수를 수정하여, 호스트 가시성 버퍼는 임시 버퍼로만 사용하고, 디바이스 로컬 버퍼를 실제 정점 버퍼로 사용하도록 변경하겠습니다.

```rust
fn create_vertex_buffer(&mut self) {
    let buffer_size = (std::mem::size_of::<Vertex>() * self.vertices.len()) as vk::DeviceSize;

    let (staging_buffer, staging_buffer_memory) = self.create_buffer(
        buffer_size,
        vk::BufferUsageFlags::TRANSFER_SRC,
        vk::MemoryPropertyFlags::HOST_VISIBLE | vk::MemoryPropertyFlags::HOST_COHERENT,
    );

    unsafe {
        let data_ptr = self.device.map_memory(
            staging_buffer_memory,
            0,
            buffer_size,
            vk::MemoryMapFlags::empty(),
        ).expect("failed to map staging buffer memory!");

        let mut align = ash::util::Align::new(
            data_ptr,
            std::mem::align_of::<Vertex>() as u64,
            buffer_size,
        );
        align.copy_from_slice(&self.vertices);

        self.device.unmap_memory(staging_buffer_memory);
    }

    let (vertex_buffer, vertex_buffer_memory) = self.create_buffer(
        buffer_size,
        vk::BufferUsageFlags::TRANSFER_DST | vk::BufferUsageFlags::VERTEX_BUFFER,
        vk::MemoryPropertyFlags::DEVICE_LOCAL,
    );
    self.vertex_buffer = vertex_buffer;
    self.vertex_buffer_memory = vertex_buffer_memory;

    // ... 다음 단계에서 복사 로직 추가 ...
}
```

이제 정점 데이터를 매핑하고 복사하기 위해 `staging_buffer`와 `staging_buffer_memory`를 사용합니다. 이번 장에서는 두 개의 새로운 버퍼 사용 플래그를 사용합니다:

*   `vk::BufferUsageFlags::TRANSFER_SRC`: 버퍼가 메모리 전송 연산의 원본(source)으로 사용될 수 있습니다.
*   `vk::BufferUsageFlags::TRANSFER_DST`: 버퍼가 메모리 전송 연산의 대상(destination)으로 사용될 수 있습니다.

`vertex_buffer`는 이제 디바이스 로컬 메모리 타입으로 할당됩니다. 이는 일반적으로 우리가 `map_memory`를 사용할 수 없다는 것을 의미합니다. 하지만 `staging_buffer`에서 `vertex_buffer`로 데이터를 복사할 수는 있습니다. 이를 위해 `staging_buffer`에는 전송 원본 플래그를, `vertex_buffer`에는 정점 버퍼 사용 플래그와 함께 전송 대상 플래그를 지정해야 합니다.

이제 한 버퍼의 내용을 다른 버퍼로 복사하는 `copy_buffer` 함수를 작성하겠습니다.

```rust
fn copy_buffer(&self, src_buffer: vk::Buffer, dst_buffer: vk::Buffer, size: vk::DeviceSize) {
    // ...
}
```

메모리 전송 연산은 그리기 명령과 마찬가지로 커맨드 버퍼를 사용하여 실행됩니다. 따라서 먼저 임시 커맨드 버퍼를 할당해야 합니다. 이런 종류의 단기 버퍼를 위해 `vk::CommandPoolCreateFlags::TRANSIENT` 플래그를 사용하여 별도의 커맨드 풀을 만드는 것을 고려할 수 있습니다.

```rust
fn copy_buffer(&self, src_buffer: vk::Buffer, dst_buffer: vk::Buffer, size: vk::DeviceSize) {
    let alloc_info = vk::CommandBufferAllocateInfo::builder()
        .level(vk::CommandBufferLevel::PRIMARY)
        .command_pool(self.command_pool)
        .command_buffer_count(1);

    let command_buffer = unsafe {
        self.device
            .allocate_command_buffers(&alloc_info)
            .expect("failed to allocate command buffers!")[0]
    };
}
```

그리고 즉시 커맨드 버퍼 기록을 시작합니다. `ash`에서는 빌더 패턴을 사용합니다.

```rust
let begin_info = vk::CommandBufferBeginInfo::builder()
    .flags(vk::CommandBufferUsageFlags::ONE_TIME_SUBMIT);

unsafe {
    self.device
        .begin_command_buffer(command_buffer, &begin_info)
        .expect("failed to begin recording command buffer!");
}
```

우리는 이 커맨드 버퍼를 한 번만 사용할 것이므로 `vk::CommandBufferUsageFlags::ONE_TIME_SUBMIT` 플래그를 사용하여 드라이버에 최적화 힌트를 줍니다.

```rust
let copy_region = vk::BufferCopy::builder()
    .src_offset(0) // Optional
    .dst_offset(0) // Optional
    .size(size)
    .build();

unsafe {
    self.device
        .cmd_copy_buffer(command_buffer, src_buffer, dst_buffer, &[copy_region]);
}
```

버퍼 내용은 `cmd_copy_buffer` 명령을 통해 전송됩니다. 이 함수는 원본과 대상 버퍼, 그리고 복사할 영역들을 담은 슬라이스(`&[vk::BufferCopy]`)를 인자로 받습니다.

```rust
unsafe {
    self.device
        .end_command_buffer(command_buffer)
        .expect("failed to record command buffer!");
}
```

이 커맨드 버퍼는 복사 명령만 포함하므로, 바로 기록을 중단합니다. 이제 커맨드 버퍼를 실행하여 전송을 완료합니다.

```rust
let submit_info = vk::SubmitInfo::builder()
    .command_buffers(&[command_buffer])
    .build();

unsafe {
    self.device
        .queue_submit(self.graphics_queue, &[submit_info], vk::Fence::null())
        .expect("failed to submit draw command buffer!");
    self.device
        .queue_wait_idle(self.graphics_queue)
        .expect("queue wait idle failed!");
}
```

`queue_wait_idle`을 사용하여 전송이 완료될 때까지 동기적으로 기다립니다. 펜스를 사용하면 여러 전송을 동시에 스케줄링하고 한 번에 기다리는 비동기적인 방식도 가능합니다.

```rust
unsafe {
    self.device
        .free_command_buffers(self.command_pool, &[command_buffer]);
}
```

전송 작업에 사용된 커맨드 버퍼를 정리하는 것을 잊지 마세요.

이제 `create_vertex_buffer` 함수에서 `copy_buffer`를 호출하여 정점 데이터를 디바이스 로컬 버퍼로 옮기고, 스테이징 버퍼를 정리합니다.

```rust
fn create_vertex_buffer(&mut self) {
    let buffer_size = (std::mem::size_of::<Vertex>() * self.vertices.len()) as vk::DeviceSize;

    let (staging_buffer, staging_buffer_memory) = self.create_buffer(
        buffer_size,
        vk::BufferUsageFlags::TRANSFER_SRC,
        vk::MemoryPropertyFlags::HOST_VISIBLE | vk::MemoryPropertyFlags::HOST_COHERENT,
    );

    // ... (Map, copy, unmap logic from before) ...
    unsafe {
        let data_ptr = self.device.map_memory(staging_buffer_memory, 0, buffer_size, vk::MemoryMapFlags::empty()).unwrap();
        let mut align = ash::util::Align::new(data_ptr, std::mem::align_of::<Vertex>() as u64, buffer_size);
        align.copy_from_slice(&self.vertices);
        self.device.unmap_memory(staging_buffer_memory);
    }


    let (vertex_buffer, vertex_buffer_memory) = self.create_buffer(
        buffer_size,
        vk::BufferUsageFlags::TRANSFER_DST | vk::BufferUsageFlags::VERTEX_BUFFER,
        vk::MemoryPropertyFlags::DEVICE_LOCAL,
    );
    self.vertex_buffer = vertex_buffer;
    self.vertex_buffer_memory = vertex_buffer_memory;

    self.copy_buffer(staging_buffer, self.vertex_buffer, buffer_size);

    unsafe {
        self.device.destroy_buffer(staging_buffer, None);
        self.device.free_memory(staging_buffer_memory, None);
    }
}
```

프로그램을 실행하여 익숙한 삼각형이 다시 보이는지 확인하세요. 성능 향상이 지금 당장은 눈에 보이지 않을 수 있지만, 이제 정점 데이터는 고성능 메모리에서 로드되고 있습니다. 이는 앞으로 더 복잡한 지오메트리를 렌더링하기 시작할 때 중요해질 것입니다.

## 결론

실제 애플리케이션에서는 모든 개별 버퍼에 대해 `allocate_memory`를 호출해서는 안 된다는 점에 유의해야 합니다. 동시 메모리 할당의 최대 수는 `max_memory_allocation_count` 물리 디바이스 제한에 의해 제한됩니다. 다수의 객체에 대해 동시에 메모리를 할당하는 올바른 방법은, 단일 할당을 여러 객체에 나누어 사용하는 커스텀 할당자를 만드는 것입니다.

이러한 할당자를 직접 구현하거나, Rust 생태계에서 널리 사용되는 [gpu-allocator](https://github.com/Traverse-Research/gpu-allocator)나 [vk-mem-rs](https://github.com/gwihlidal/vk-mem-rs) 같은 라이브러리를 사용할 수 있습니다. 이들은 C++의 `VulkanMemoryAllocator`에 해당하는 훌륭한 대안입니다. 하지만 이 튜토리얼에서는 지금 당장 이러한 제한에 도달할 일이 없으므로 모든 리소스에 대해 별도의 할당을 사용하는 것이 괜찮습니다.

[Rust 코드](/code/20_staging_buffer) /
[정점 셰이더](/code/18_shader_vertexbuffer.vert) /
[프래그먼트 셰이더](/code/18_shader_vertexbuffer.frag)