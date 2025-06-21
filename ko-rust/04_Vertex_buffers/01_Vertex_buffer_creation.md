## 소개

Vulkan에서 버퍼(Buffer)는 그래픽 카드가 읽을 수 있는 임의의 데이터를 저장하는 데 사용되는 메모리 영역입니다. 이번 장에서 다룰 정점 데이터(vertex data)를 저장하는 데 사용할 수도 있지만, 앞으로의 장에서 살펴볼 다른 많은 목적으로도 사용될 수 있습니다. 지금까지 다뤄온 다른 Vulkan 객체들과는 달리, 버퍼는 스스로 메모리를 할당하지 않습니다. 이전 장들에서 보았듯이 Vulkan API는 프로그래머가 거의 모든 것을 직접 제어하도록 하며, 메모리 관리도 그중 하나입니다. Rust에서도 `ash`와 같은 라이브러리를 사용하더라도 이 원칙은 동일하게 적용됩니다.

## 버퍼 생성

`create_vertex_buffer`라는 새 함수를 만들고, `init_vulkan` 함수에서 `create_command_buffers` 바로 전에 호출하도록 합시다.

```rust
// In `impl HelloTriangleApplication`
fn init_vulkan(&mut self) -> Result<(), Box<dyn std::error::Error>> {
    self.create_instance()?;
    self.setup_debug_messenger()?;
    self.create_surface()?;
    self.pick_physical_device()?;
    self.create_logical_device()?;
    self.create_swapchain()?;
    self.create_image_views()?;
    self.create_render_pass()?;
    self.create_graphics_pipeline()?;
    self.create_framebuffers()?;
    self.create_command_pool()?;
    self.create_vertex_buffer()?; // <-- 새로운 호출
    self.create_command_buffers()?;
    self.create_sync_objects()?;
    Ok(())
}

// ...

fn create_vertex_buffer(&mut self) -> Result<(), Box<dyn std::error::Error>> {
    // 여기에 구현
    Ok(())
}
```

버퍼를 생성하려면 `vk::BufferCreateInfo` 구조체를 채워야 합니다. Rust와 `ash`에서는 빌더(builder) 패턴을 사용하는 것이 일반적이며 훨씬 깔끔합니다.

```rust
// in create_vertex_buffer
let buffer_size = (std::mem::size_of::<Vertex>() * self.vertices.len()) as vk::DeviceSize;

let buffer_info = vk::BufferCreateInfo::builder()
    .size(buffer_size)
    .usage(vk::BufferUsageFlags::VERTEX_BUFFER)
    .sharing_mode(vk::SharingMode::EXCLUSIVE);
```

`size` 필드는 버퍼의 크기를 바이트 단위로 지정합니다. Rust에서는 `std::mem::size_of`를 사용해 정점 구조체의 크기를 얻고, 이를 벡터의 길이와 곱하여 전체 크기를 계산할 수 있습니다.

`usage` 필드는 버퍼의 데이터가 어떤 목적으로 사용될지를 나타냅니다. 우리의 사용 사례는 정점 버퍼이므로 `vk::BufferUsageFlags::VERTEX_BUFFER`를 사용합니다.

`sharing_mode`는 스왑 체인의 이미지처럼 버퍼가 특정 큐 패밀리에 의해 배타적으로 소유될지, 아니면 여러 큐 간에 공유될지를 결정합니다. 우리는 그래픽스 큐에서만 사용할 것이므로 `vk::SharingMode::EXCLUSIVE`으로 설정합니다.

이제 `device.create_buffer`를 사용해 버퍼를 생성할 수 있습니다. 버퍼 핸들과 할당된 메모리를 저장할 구조체 필드를 정의합니다.

```rust
// in `HelloTriangleApplication` struct
struct HelloTriangleApplication {
    // ...
    vertex_buffer: vk::Buffer,
    vertex_buffer_memory: vk::DeviceMemory,
    // ...
}

// in create_vertex_buffer
let vertex_buffer = unsafe {
    self.device.create_buffer(&buffer_info, None)?
};
self.vertex_buffer = vertex_buffer;
```

`ash`의 생성 및 파괴 함수들은 대부분 `unsafe`로 표시되어 있습니다. 이는 개발자가 Vulkan 객체의 생명주기를 올바르게 관리할 책임이 있다는 것을 명시하기 위함입니다. 예를 들어, `device`가 파괴되기 전에 이 버퍼를 반드시 파괴해야 합니다.

버퍼는 프로그램이 끝날 때까지 사용되므로, Rust의 `Drop` 트레잇을 구현하여 정리하는 것이 가장 관용적입니다.

```rust
// in `impl Drop for HelloTriangleApplication`
impl Drop for HelloTriangleApplication {
    fn drop(&mut self) {
        unsafe {
            // ... 다른 객체들 정리
            self.device.destroy_buffer(self.vertex_buffer, None);
            // ...
        }
    }
}
```

## 메모리 요구사항

버퍼는 생성되었지만 아직 실제 메모리가 할당되지 않았습니다. 버퍼에 메모리를 할당하는 첫 단계는 `get_buffer_memory_requirements` 함수를 사용하여 메모리 요구사항을 쿼리하는 것입니다.

```rust
// in create_vertex_buffer, after create_buffer
let mem_requirements = unsafe {
    self.device.get_buffer_memory_requirements(self.vertex_buffer)
};
```

`ash`에서는 이 함수가 `vk::MemoryRequirements` 구조체를 직접 반환합니다. 이 구조체의 필드는 다음과 같습니다.

*   `size`: 필요한 메모리의 크기(바이트 단위). `buffer_info.size`와 다를 수 있습니다.
*   `alignment`: 할당된 메모리 영역 내에서 버퍼가 시작되는 오프셋(바이트 단위).
*   `memory_type_bits`: 버퍼에 적합한 메모리 타입들의 비트 필드.

그래픽 카드는 다양한 종류의 메모리를 제공하며, 각각의 특성이 다릅니다. 버퍼의 요구사항과 애플리케이션의 요구사항을 결합하여 올바른 메모리 타입을 찾아야 합니다. 이를 위해 `find_memory_type` 함수를 만들어 봅시다.

```rust
// In `impl HelloTriangleApplication`
fn find_memory_type(&self, type_filter: u32, properties: vk::MemoryPropertyFlags) -> u32 {
    let mem_properties = unsafe {
        self.instance.get_physical_device_memory_properties(self.physical_device)
    };

    for i in 0..mem_properties.memory_type_count {
        if (type_filter & (1 << i)) != 0
            && mem_properties.memory_types[i as usize]
                .property_flags
                .contains(properties)
        {
            return i;
        }
    }

    panic!("failed to find suitable memory type!");
}
```

이 함수는 C++ 버전과 매우 유사하게 동작합니다. 먼저 `instance.get_physical_device_memory_properties`를 통해 물리 디바이스의 메모리 속성을 가져옵니다.

루프 안에서는 `type_filter`를 통해 버퍼가 요구하는 메모리 타입인지 확인하고, `ash`의 비트플래그가 제공하는 `contains` 메서드를 사용해 우리가 필요로 하는 `properties`(예: CPU에서 접근 가능하고 일관성을 유지하는 속성)를 모두 포함하는지 검사합니다.

## 메모리 할당

이제 올바른 메모리 타입을 결정할 수 있으므로, `vk::MemoryAllocateInfo` 구조체를 채워 메모리를 할당할 수 있습니다.

```rust
// in create_vertex_buffer
let memory_type_index = self.find_memory_type(
    mem_requirements.memory_type_bits,
    vk::MemoryPropertyFlags::HOST_VISIBLE | vk::MemoryPropertyFlags::HOST_COHERENT,
);

let alloc_info = vk::MemoryAllocateInfo::builder()
    .allocation_size(mem_requirements.size)
    .memory_type_index(memory_type_index);

let vertex_buffer_memory = unsafe {
    self.device.allocate_memory(&alloc_info, None)?
};
self.vertex_buffer_memory = vertex_buffer_memory;
```

`find_memory_type`을 호출하여 적절한 메모리 타입의 인덱스를 찾고, `allocation_size`와 함께 `MemoryAllocateInfo`를 설정합니다. `HOST_VISIBLE` 속성은 CPU가 이 메모리에 접근(map)할 수 있게 하고, `HOST_COHERENT` 속성은 CPU의 쓰기가 별도의 명시적 플러시(flush) 작업 없이 GPU에 보이도록 보장합니다.

메모리 할당에 성공했다면, `bind_buffer_memory`를 호출하여 이 메모리를 버퍼와 연결합니다.

```rust
// in create_vertex_buffer, after allocate_memory
unsafe {
    self.device.bind_buffer_memory(self.vertex_buffer, self.vertex_buffer_memory, 0)?;
}
```

오프셋은 `0`으로 설정합니다. 이 메모리는 이 버퍼만을 위해 할당되었기 때문입니다.

할당된 메모리 역시 `Drop` 트레잇 내에서 해제해야 합니다. 버퍼가 파괴된 후에 메모리를 해제하는 것이 좋습니다.

```rust
// in `impl Drop for HelloTriangleApplication`
impl Drop for HelloTriangleApplication {
    fn drop(&mut self) {
        unsafe {
            // ...
            self.device.destroy_buffer(self.vertex_buffer, None);
            self.device.free_memory(self.vertex_buffer_memory, None);
            // ...
        }
    }
}
```

## 정점 버퍼 채우기

이제 정점 데이터를 버퍼에 복사할 시간입니다. `device.map_memory`를 사용하여 버퍼 메모리를 CPU가 접근할 수 있는 메모리 공간에 맵핑합니다.

```rust
// in create_vertex_buffer, after bind_buffer_memory
let data_ptr = unsafe {
    self.device.map_memory(
        self.vertex_buffer_memory,
        0,
        buffer_size,
        vk::MemoryMapFlags::empty(),
    )?
};
```

이 함수는 맵핑된 메모리 영역을 가리키는 원시 포인터(`*mut std::ffi::c_void`)를 반환합니다. 이제 이 포인터를 통해 데이터를 복사할 수 있습니다. Rust에서는 `std::ptr::copy_nonoverlapping`을 사용합니다.

```rust
// in create_vertex_buffer, after map_memory
unsafe {
    let mut align = ash::util::Align::new(
        data_ptr,
        std::mem::align_of::<Vertex>() as _,
        buffer_size,
    );
    align.copy_from_slice(&self.vertices);
}
```

**참고**: `ash` 0.37.0부터는 `ash::util::Align`이라는 편리한 유틸리티가 제공됩니다. 이 유틸리티는 맵핑된 메모리의 정렬(alignment)을 처리하고 슬라이스에서 데이터를 안전하게 복사하는 작업을 도와줍니다. `memcpy`와 같은 저수준의 포인터 연산을 직접 사용하는 것보다 안전하고 편리합니다.

데이터 복사가 끝나면 `unmap_memory`를 호출하여 맵핑을 해제합니다.

```rust
// in create_vertex_buffer, after copying data
unsafe {
    self.device.unmap_memory(self.vertex_buffer_memory);
}
```

우리는 `HOST_COHERENT` 메모리 타입을 사용했기 때문에, 드라이버가 CPU의 쓰기를 자동으로 GPU에 반영합니다. 이는 명시적으로 `flush`를 호출하는 것보다 약간의 성능 저하가 있을 수 있지만, 이 예제에서는 더 간단하고 충분합니다.

## 정점 버퍼 바인딩

이제 남은 일은 렌더링 과정에서 정점 버퍼를 바인딩하는 것입니다. `record_command_buffer` 함수를 수정합니다.

```rust
// in record_command_buffer
unsafe {
    self.device.cmd_bind_pipeline(
        command_buffer,
        vk::PipelineBindPoint::GRAPHICS,
        self.graphics_pipeline,
    );

    let vertex_buffers = [self.vertex_buffer];
    let offsets = [0];
    self.device.cmd_bind_vertex_buffers(command_buffer, 0, &vertex_buffers, &offsets);

    self.device.cmd_draw(command_buffer, self.vertices.len() as u32, 1, 0, 0);
}
```

`cmd_bind_vertex_buffers` 함수는 정점 버퍼를 특정 바인딩 위치에 연결합니다. Rust에서는 배열 슬라이스(`&[...]`)를 사용하여 버퍼와 오프셋을 전달합니다. 또한, `cmd_draw` 호출에서 하드코딩된 정점 수 `3` 대신 `self.vertices.len()`을 사용하여 버퍼에 있는 실제 정점 수를 전달하도록 수정합니다.

이제 프로그램을 실행하면 익숙한 삼각형이 다시 나타날 것입니다.

![](/images/triangle.png)

맨 위 정점의 색을 흰색으로 바꾸려면 `vertices` 배열을 수정해 보세요.

```rust
// in main.rs or where Vertex is defined
const VERTICES: [Vertex; 3] = [
    Vertex { pos: [0.0, -0.5], color: [1.0, 1.0, 1.0] },
    Vertex { pos: [0.5, 0.5],  color: [0.0, 1.0, 0.0] },
    Vertex { pos: [-0.5, 0.5], color: [0.0, 0.0, 1.0] },
];
```

프로그램을 다시 실행하면 다음과 같은 결과를 볼 수 있습니다.

![](/images/triangle_white.png)

다음 장에서는 더 나은 성능을 제공하지만 약간의 추가 작업이 필요한, 정점 데이터를 정점 버퍼로 복사하는 다른 방법을 살펴보겠습니다.