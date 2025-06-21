## 소개

이 보너스 챕터에서는 컴퓨트 셰이더(compute shader)에 대해 살펴보겠습니다. 지금까지의 모든 챕터는 Vulkan 파이프라인의 전통적인 그래픽스 부분을 다루었습니다. 하지만 OpenGL과 같은 오래된 API와 달리, Vulkan에서 컴퓨트 셰이더 지원은 필수입니다. 이는 고사양 데스크톱 GPU든 저전력 임베디드 장치든, 사용 가능한 모든 Vulkan 구현에서 컴퓨트 셰이더를 사용할 수 있다는 의미입니다.

이는 여러분의 애플리케이션이 어디서 실행되든 상관없이 GPU(그래픽 처리 장치)를 이용한 범용 컴퓨팅(GPGPU, general purpose computing on graphics processor units)의 세계를 열어줍니다. GPGPU는 전통적으로 CPU의 영역이었던 일반적인 계산을 GPU에서 수행할 수 있음을 의미합니다. GPU가 점점 더 강력해지고 유연해짐에 따라, CPU의 범용적인 능력이 필요했던 많은 작업들을 이제 GPU에서 실시간으로 처리할 수 있게 되었습니다.

GPU의 컴퓨팅 능력이 사용될 수 있는 몇 가지 예로는 이미지 처리, 가시성 테스트, 후처리(post processing), 고급 조명 계산, 애니메이션, 물리(예: 파티클 시스템) 등이 있으며, 이 외에도 훨씬 더 많습니다. 심지어 수치 연산이나 AI 관련 작업처럼 그래픽 출력이 전혀 필요 없는 비시각적 계산 전용 작업에도 컴퓨트를 사용할 수 있습니다. 이를 "헤드리스 컴퓨트(headless compute)"라고 합니다.

## 장점

계산 비용이 많이 드는 작업을 GPU에서 수행하면 몇 가지 장점이 있습니다. 가장 명백한 것은 CPU의 작업을 덜어내는 것입니다. 또 다른 장점은 CPU의 주 메모리와 GPU 메모리 간에 데이터를 옮길 필요가 없다는 점입니다. 모든 데이터는 주 메모리로부터의 느린 전송을 기다릴 필요 없이 GPU에 머무를 수 있습니다.

이 외에도 GPU는 수만 개의 작은 연산 유닛으로 고도로 병렬화되어 있습니다. 이 때문에 몇 개의 큰 연산 유닛을 가진 CPU보다 고도로 병렬화된 워크플로우에 더 적합한 경우가 많습니다.

## Vulkan 파이프라인

컴퓨트는 파이프라인의 그래픽스 부분과 완전히 분리되어 있다는 점을 알아두는 것이 중요합니다. 이는 공식 명세서의 다음 Vulkan 파이프라인 블록 다이어그램에서 확인할 수 있습니다.

![](/images/vulkan_pipeline_block_diagram.png)

이 다이어그램의 왼쪽에는 전통적인 그래픽스 파이프라인 부분이 있고, 오른쪽에는 컴퓨트 셰이더 단계를 포함하여 이 그래픽스 파이프라인에 속하지 않는 여러 단계들이 있습니다. 컴퓨트 셰이더 단계가 그래픽스 파이프라인에서 분리되어 있으므로, 우리는 필요하다고 생각되는 어느 곳에서든 이를 사용할 수 있습니다. 이는 항상 정점 셰이더의 변환된 출력에 적용되는 프래그먼트 셰이더와는 매우 다릅니다.

다이어그램 중앙은 디스크립터 셋(descriptor set)과 같은 요소들이 컴퓨트에서도 사용된다는 것을 보여주므로, 우리가 디스크립터 레이아웃, 디스크립터 셋, 디스크립터에 대해 배운 모든 것이 여기에도 적용됩니다.

## 예제

이 챕터에서 구현할 이해하기 쉬운 예제는 GPU 기반 파티클 시스템입니다. 이러한 시스템은 많은 게임에서 사용되며, 종종 상호작용 가능한 프레임 속도로 업데이트되어야 하는 수천 개의 파티클로 구성됩니다. 이러한 시스템을 렌더링하려면 두 가지 주요 구성 요소가 필요합니다: 정점 버퍼로 전달되는 정점들과, 어떤 방정식에 기반하여 이들을 업데이트하는 방법입니다.

GPU 기반 파티클 시스템을 사용하면 CPU를 거치는 과정이 더 이상 필요하지 않습니다. 정점 데이터는 처음에만 GPU에 업로드되며, 모든 업데이트는 컴퓨트 셰이더를 사용하여 GPU 메모리 내에서 이루어집니다. 이것이 더 빠른 주된 이유 중 하나는 GPU와 로컬 메모리 간의 훨씬 높은 대역폭 때문입니다.

다음은 이 챕터 코드의 스크린샷입니다. 여기에 보이는 파티클은 CPU 상호작용 없이 GPU에서 직접 컴퓨트 셰이더에 의해 업데이트됩니다.

![](/images/compute_shader_particles.png)

## 데이터 조작

### 셰이더 저장 버퍼 객체 (SSBO)

셰이더 저장 버퍼(SSBO, Shader Storage Buffer Object)는 셰이더가 버퍼에서 읽고 쓸 수 있게 해줍니다. Vulkan에서는 버퍼와 이미지에 대해 여러 사용 용도를 지정할 수 있습니다. 따라서 파티클 정점 버퍼를 정점 버퍼(그래픽스 패스)와 저장 버퍼(컴퓨트 패스)로 사용하려면, 해당 사용 플래그로 버퍼를 생성하기만 하면 됩니다.

`ash`에서는 빌더 패턴을 사용하여 생성 정보를 구성합니다. `|` 연산자는 `bitflags` 크레이트에 의해 제공되므로 C++와 유사하게 사용할 수 있습니다.

```rust
use ash::vk;

let buffer_info = vk::BufferCreateInfo::builder()
    .size(buffer_size)
    .usage(
        vk::BufferUsageFlags::VERTEX_BUFFER
            | vk::BufferUsageFlags::STORAGE_BUFFER
            | vk::BufferUsageFlags::TRANSFER_DST,
    )
    .sharing_mode(vk::SharingMode::EXCLUSIVE);

let shader_storage_buffer = unsafe { device.create_buffer(&buffer_info, None) }
    .expect("셰이더 저장 버퍼 생성에 실패했습니다!");
```

GLSL 셰이더의 SSBO 선언은 C++ 예제와 동일합니다. Rust 측에서는 이 구조체를 `#[repr(C)]`로 정의하여 C/GLSL과 메모리 레이아웃을 호환시켜야 합니다.

```glsl
// GLSL
struct Particle {
  vec2 position;
  vec2 velocity;
  vec4 color;
};

layout(std140, binding = 1) readonly buffer ParticleSSBOIn {
   Particle particlesIn[ ];
};

layout(std140, binding = 2) buffer ParticleSSBOOut {
   Particle particlesOut[ ];
};
```

```rust
// Rust
use glam::{Vec2, Vec4}; // 또는 다른 벡터 라이브러리

#[repr(C)]
#[derive(Clone, Copy, Debug)]
struct Particle {
    position: Vec2,
    velocity: Vec2,
    color: Vec4,
}
```

## 컴퓨트 큐 패밀리

컴퓨트 작업을 하려면 `VK_QUEUE_COMPUTE_BIT` 플래그를 지원하는 큐 패밀리를 찾아야 합니다. Rust에서는 이터레이터와 `find`, `enumerate`를 사용하여 이 과정을 더 깔끔하게 작성할 수 있습니다.

```rust
let queue_families = unsafe {
    instance.get_physical_device_queue_family_properties(physical_device)
};

let queue_family_indices = queue_families
    .iter()
    .enumerate()
    .find(|(_index, queue_family)| {
        queue_family.queue_flags.contains(vk::QueueFlags::GRAPHICS)
            && queue_family.queue_flags.contains(vk::QueueFlags::COMPUTE)
    })
    .map(|(index, _queue_family)| index as u32);
```

큐를 가져오는 작업은 `ash`의 `Device` 구조체에 대한 메서드 호출로 이루어집니다.

```rust
let compute_queue = unsafe {
    device.get_device_queue(indices.graphics_and_compute_family.unwrap(), 0)
};
```

## 컴퓨트 셰이더 로드하기

셰이더 로드는 다른 셰이더와 동일하지만, `stage` 필드에 `vk::ShaderStageFlags::COMPUTE`를 사용해야 합니다. Rust에서는 `pName` 필드에 C 문자열을 전달해야 하므로, `CStr`을 사용합니다.

```rust
use std::ffi::CStr;

let compute_shader_code = read_shader_code("shaders/compute.spv"); // 셰이더 파일을 읽는 헬퍼 함수
let compute_shader_module = create_shader_module(&device, &compute_shader_code)?;

let shader_entry_name = CStr::from_bytes_with_nul(b"main\0").unwrap();

let compute_shader_stage_info = vk::PipelineShaderStageCreateInfo::builder()
    .stage(vk::ShaderStageFlags::COMPUTE)
    .module(compute_shader_module)
    .name(shader_entry_name)
    .build();
```

## 셰이더 저장 버퍼 준비하기

`Frames in flight` 개념을 적용하여, 각 프레임마다 SSBO를 생성합니다. Rust에서는 `Vec`을 사용하여 버퍼와 메모리 핸들을 저장합니다.

```rust
let mut shader_storage_buffers: Vec<vk::Buffer> = Vec::with_capacity(MAX_FRAMES_IN_FLIGHT);
let mut shader_storage_buffers_memory: Vec<vk::DeviceMemory> = Vec::with_capacity(MAX_FRAMES_IN_FLIGHT);
```

호스트 측에서 파티클 데이터를 초기화하고, 스테이징 버퍼를 통해 GPU 전용 메모리로 복사합니다.

```rust
// 파티클 초기화 (rand 크레이트 사용 가능)
let mut particles: Vec<Particle> = Vec::with_capacity(PARTICLE_COUNT);
for _ in 0..PARTICLE_COUNT {
    // ... 파티클 생성 로직 ...
}

let buffer_size = (std::mem::size_of::<Particle>() * PARTICLE_COUNT) as vk::DeviceSize;

// 스테이징 버퍼 생성 및 데이터 복사
let (staging_buffer, staging_buffer_memory) = create_buffer(
    &device,
    buffer_size,
    vk::BufferUsageFlags::TRANSFER_SRC,
    vk::MemoryPropertyFlags::HOST_VISIBLE | vk::MemoryPropertyFlags::HOST_COHERENT,
    // ...
)?;

// `ash::util::Align`을 사용한 안전한 메모리 복사
unsafe {
    let data_ptr = device.map_memory(staging_buffer_memory, 0, buffer_size, vk::MemoryMapFlags::empty())?;
    let mut align = ash::util::Align::new(data_ptr, std::mem::align_of::<Particle>() as u64, buffer_size);
    align.copy_from_slice(&particles);
    device.unmap_memory(staging_buffer_memory);
}

// 각 프레임에 대한 SSBO 생성 및 데이터 복사
for i in 0..MAX_FRAMES_IN_FLIGHT {
    let (buffer, memory) = create_buffer(
        &device,
        buffer_size,
        vk::BufferUsageFlags::STORAGE_BUFFER | vk::BufferUsageFlags::VERTEX_BUFFER | vk::BufferUsageFlags::TRANSFER_DST,
        vk::MemoryPropertyFlags::DEVICE_LOCAL,
        // ...
    )?;
    copy_buffer(staging_buffer, buffer, buffer_size, ...)?;
    shader_storage_buffers.push(buffer);
    shader_storage_buffers_memory.push(memory);
}

// 스테이징 버퍼 정리
// ...
```

## 디스크립터

컴퓨트 셰이더를 위한 디스크립터 레이아웃을 설정할 때, `stage_flags`에 `vk::ShaderStageFlags::COMPUTE`를 지정합니다.

```rust
let layout_bindings = [
    vk::DescriptorSetLayoutBinding::builder()
        .binding(0)
        .descriptor_type(vk::DescriptorType::UNIFORM_BUFFER)
        .descriptor_count(1)
        .stage_flags(vk::ShaderStageFlags::COMPUTE)
        .build(),
    vk::DescriptorSetLayoutBinding::builder()
        .binding(1)
        .descriptor_type(vk::DescriptorType::STORAGE_BUFFER)
        .descriptor_count(1)
        .stage_flags(vk::ShaderStageFlags::COMPUTE)
        .build(),
    vk::DescriptorSetLayoutBinding::builder()
        .binding(2)
        .descriptor_type(vk::DescriptorType::STORAGE_BUFFER)
        .descriptor_count(1)
        .stage_flags(vk::ShaderStageFlags::COMPUTE)
        .build(),
];
let layout_info = vk::DescriptorSetLayoutCreateInfo::builder().bindings(&layout_bindings);
let compute_descriptor_set_layout = unsafe { device.create_descriptor_set_layout(&layout_info, None)? };
```

디스크립터 셋을 업데이트할 때, 이전 프레임과 현재 프레임의 SSBO를 모두 참조하도록 설정합니다. Rust에서는 구조체와 슬라이스를 사용하여 `pBufferInfo`를 안전하게 처리해야 합니다.

```rust
for i in 0..MAX_FRAMES_IN_FLIGHT {
    let uniform_buffer_info = vk::DescriptorBufferInfo::builder()
        .buffer(uniform_buffers[i])
        .offset(0)
        .range(std::mem::size_of::<UniformBufferObject>() as u64)
        .build();

    let storage_buffer_info_last_frame = vk::DescriptorBufferInfo::builder()
        .buffer(shader_storage_buffers[(i + MAX_FRAMES_IN_FLIGHT - 1) % MAX_FRAMES_IN_FLIGHT])
        .offset(0)
        .range(buffer_size)
        .build();

    let storage_buffer_info_current_frame = vk::DescriptorBufferInfo::builder()
        .buffer(shader_storage_buffers[i])
        .offset(0)
        .range(buffer_size)
        .build();

    let descriptor_writes = [
        // UBO 쓰기
        vk::WriteDescriptorSet::builder()
            .dst_set(compute_descriptor_sets[i])
            .dst_binding(0)
            .descriptor_type(vk::DescriptorType::UNIFORM_BUFFER)
            .buffer_info(std::slice::from_ref(&uniform_buffer_info))
            .build(),
        // 이전 프레임 SSBO 쓰기
        vk::WriteDescriptorSet::builder()
            .dst_set(compute_descriptor_sets[i])
            .dst_binding(1)
            .descriptor_type(vk::DescriptorType::STORAGE_BUFFER)
            .buffer_info(std::slice::from_ref(&storage_buffer_info_last_frame))
            .build(),
        // 현재 프레임 SSBO 쓰기
        vk::WriteDescriptorSet::builder()
            .dst_set(compute_descriptor_sets[i])
            .dst_binding(2)
            .descriptor_type(vk::DescriptorType::STORAGE_BUFFER)
            .buffer_info(std::slice::from_ref(&storage_buffer_info_current_frame))
            .build(),
    ];
    
    unsafe { device.update_descriptor_sets(&descriptor_writes, &[]) };
}
```

디스크립터 풀을 생성할 때, 두 개의 SSBO를 사용하므로 필요한 수를 두 배로 요청해야 합니다.

```rust
let pool_sizes = [
    // ...
    vk::DescriptorPoolSize {
        ty: vk::DescriptorType::STORAGE_BUFFER,
        descriptor_count: (MAX_FRAMES_IN_FLIGHT * 2) as u32,
    },
];
```

## 컴퓨트 파이프라인

컴퓨트 파이프라인은 `create_compute_pipelines` 함수로 생성합니다. 그래픽스 파이프라인보다 훨씬 간단합니다.

```rust
let pipeline_layout_info = vk::PipelineLayoutCreateInfo::builder()
    .set_layouts(std::slice::from_ref(&compute_descriptor_set_layout));
let compute_pipeline_layout = unsafe { device.create_pipeline_layout(&pipeline_layout_info, None)? };

let pipeline_info = vk::ComputePipelineCreateInfo::builder()
    .stage(compute_shader_stage_info)
    .layout(compute_pipeline_layout)
    .build();

let compute_pipelines = unsafe {
    device.create_compute_pipelines(vk::PipelineCache::null(), &[pipeline_info], None)
}.map_err(|(_, err)| err)?;

let compute_pipeline = compute_pipelines[0];
// 사용 후에는 `device.destroy_pipeline`으로 파이프라인들을 정리해야 합니다.
```

## 컴퓨트 공간 및 컴퓨트 셰이더

이 개념들은 API에 독립적이므로, C++ 튜토리얼의 설명과 동일합니다. GLSL 셰이더 코드도 변경 없이 그대로 사용할 수 있습니다.

## 컴퓨트 명령 실행하기

### 디스패치

`ash`에서 모든 `cmd_` 함수는 `unsafe` 블록 안에서 호출되어야 합니다. 이는 유효한 커맨드 버퍼 기록 상태에서만 호출되어야 함을 명시하기 위함입니다. `cmd_dispatch`로 컴퓨트 작업을 시작합니다.

```rust
unsafe {
    device.begin_command_buffer(command_buffer, &begin_info)?;

    device.cmd_bind_pipeline(command_buffer, vk::PipelineBindPoint::COMPUTE, compute_pipeline);
    device.cmd_bind_descriptor_sets(
        command_buffer,
        vk::PipelineBindPoint::COMPUTE,
        compute_pipeline_layout,
        0,
        &[compute_descriptor_sets[current_frame]],
        &[],
    );
    
    // 워크 그룹 수 계산
    let group_count = (PARTICLE_COUNT as u32 + 255) / 256;
    device.cmd_dispatch(command_buffer, group_count, 1, 1);

    device.end_command_buffer(command_buffer)?;
}
```

### 작업 제출 및 동기화

컴퓨트 작업과 그래픽스 작업을 동기화하기 위해 세마포어와 펜스를 사용합니다. `ash`를 사용한 제출 로직은 다음과 같습니다.

```rust
// 컴퓨트 작업용 동기화 객체 생성
// compute_in_flight_fences: Vec<vk::Fence>
// compute_finished_semaphores: Vec<vk::Semaphore>
// ...

// draw_frame 함수 내부
// --- 컴퓨트 제출 ---
unsafe {
    device.wait_for_fences(&[compute_in_flight_fences[current_frame]], true, u64::MAX)?;
    device.reset_fences(&[compute_in_flight_fences[current_frame]])?;
}

// ... 컴퓨트 커맨드 버퍼 기록 ...

let compute_submit_info = vk::SubmitInfo::builder()
    .command_buffers(std::slice::from_ref(&compute_command_buffers[current_frame]))
    .signal_semaphores(std::slice::from_ref(&compute_finished_semaphores[current_frame]))
    .build();

unsafe {
    device.queue_submit(
        compute_queue,
        &[compute_submit_info],
        compute_in_flight_fences[current_frame],
    )?;
}

// --- 그래픽스 제출 ---
unsafe {
    device.wait_for_fences(&[in_flight_fences[current_frame]], true, u64::MAX)?;
    device.reset_fences(&[in_flight_fences[current_frame]])?;
}

// ... 그래픽스 커맨드 버퍼 기록 ...

let wait_semaphores = [
    compute_finished_semaphores[current_frame],
    image_available_semaphores[current_frame],
];
let wait_stages = [
    vk::PipelineStageFlags::VERTEX_INPUT,
    vk::PipelineStageFlags::COLOR_ATTACHMENT_OUTPUT,
];

let graphics_submit_info = vk::SubmitInfo::builder()
    .wait_semaphores(&wait_semaphores)
    .wait_dst_stage_mask(&wait_stages)
    .command_buffers(std::slice::from_ref(&command_buffers[current_frame]))
    .signal_semaphores(std::slice::from_ref(&render_finished_semaphores[current_frame]))
    .build();

unsafe {
    device.queue_submit(
        graphics_queue,
        &[graphics_submit_info],
        in_flight_fences[current_frame],
    )?;
}
```

## 파티클 시스템 그리기

SSBO는 정점 버퍼로도 사용될 수 있으므로, 그래픽스 파이프라인에서 바로 바인딩하여 그릴 수 있습니다. Rust에서는 `memoffset` 크레이트를 사용하여 구조체 멤버의 오프셋을 안전하게 계산할 수 있습니다.

```rust
use memoffset::offset_of;
// Particle 구조체 impl 블록 내부
impl Particle {
    pub fn get_attribute_descriptions() -> [vk::VertexInputAttributeDescription; 2] {
        [
            vk::VertexInputAttributeDescription {
                binding: 0,
                location: 0,
                format: vk::Format::R32G32_SFLOAT,
                offset: offset_of!(Particle, position) as u32,
            },
            vk::VertexInputAttributeDescription {
                binding: 0,
                location: 1,
                format: vk::Format::R32G32B32A32_SFLOAT,
                offset: offset_of!(Particle, color) as u32,
            },
        ]
    }
}
```

드로잉 명령은 다음과 같습니다.

```rust
unsafe {
    let offsets = [0];
    device.cmd_bind_vertex_buffers(
        command_buffer,
        0,
        &[shader_storage_buffers[current_frame]],
        &offsets,
    );

    device.cmd_draw(command_buffer, PARTICLE_COUNT as u32, 1, 0, 0);
}
```

## 결론

이 챕터에서는 Rust와 `ash` 라이브러리를 사용하여 컴퓨트 셰이더를 설정하고 실행하는 방법을 배웠습니다. C++과 개념은 동일하지만, Rust의 소유권 시스템, `unsafe` 키워드, 빌더 패턴, 이터레이터, `Result`를 통한 오류 처리 등 언어적 특성을 활용하여 코드를 작성했습니다.

이제 여러분은 Vulkan 컴퓨트 셰이더의 기본을 익혔으며, 이를 바탕으로 공유 메모리, 비동기 컴퓨트, 원자적 연산 등 더 복잡한 GPGPU 기술을 탐구할 준비가 되었습니다.

[Rust 코드](/examples/compute_shader/main.rs) /
[정점 셰이더](/code/31_shader_compute.vert) /
[프래그먼트 셰이더](/code/31_shader_compute.frag) /
[컴퓨트 셰이더](/code/31_shader_compute.comp)