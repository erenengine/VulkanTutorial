## 소개

이전 장에서 다룬 디스크립터 셋 레이아웃은 바인딩할 수 있는 디스크립터의 유형을 설명합니다. 이번 장에서는 각 `vk::Buffer` 리소스마다 디스크립터 셋을 만들어서 유니폼 버퍼 디스크립터에 바인딩할 것입니다.

## 디스크립터 풀 (Descriptor Pool)

디스크립터 셋은 직접 생성할 수 없으며, 커맨드 버퍼처럼 풀(pool)에서 할당해야 합니다. 디스크립터 셋을 위한 이러한 풀은 *디스크립터 풀(descriptor pool)*이라고 불립니다. 이를 설정하기 위해 새로운 함수 `create_descriptor_pool`을 작성하겠습니다.

```rust
impl VulkanApp {
    fn init_vulkan(&mut self) -> Result<()> {
        ...
        self.create_uniform_buffers()?;
        self.create_descriptor_pool()?;
        ...
    }

    ...

    fn create_descriptor_pool(&mut self) -> Result<()> {
        // 이 함수 내용을 채워나갑니다.
        Ok(())
    }
}
```

먼저 `vk::DescriptorPoolSize` 구조체를 사용해 우리 디스크립터 셋이 어떤 유형의 디스크립터를 얼마나 포함할지 기술해야 합니다.

```rust
let pool_size = vk::DescriptorPoolSize {
    ty: vk::DescriptorType::UNIFORM_BUFFER,
    descriptor_count: MAX_FRAMES_IN_FLIGHT as u32,
};
```

우리는 프레임마다 하나씩 이 디스크립터를 할당할 것입니다. 이 풀 크기 구조체는 메인 `vk::DescriptorPoolCreateInfo`에서 참조됩니다. `ash`의 빌더 패턴을 사용하면 코드가 더 깔끔해집니다.

```rust
let pool_sizes = [pool_size];
let pool_info = vk::DescriptorPoolCreateInfo::builder()
    .pool_sizes(&pool_sizes)
    // ...
```

개별 디스크립터의 최대 개수 외에도, 할당될 수 있는 디스크립터 셋의 최대 개수도 지정해야 합니다.

```rust
    .max_sets(MAX_FRAMES_IN_FLIGHT as u32);
```

이 구조체는 커맨드 풀과 유사한 선택적 플래그 `vk::DescriptorPoolCreateFlags::FREE_DESCRIPTOR_SET`를 가집니다. 이 플래그는 개별 디스크립터 셋을 해제할 수 있는지 여부를 결정합니다. 우리는 디스크립터 셋을 생성한 후에는 수정하지 않을 것이므로 이 플래그는 필요 없습니다. `flags`는 기본값으로 둘 수 있습니다.

애플리케이션 구조체에 디스크립터 풀 핸들을 저장할 필드를 추가하고 `create_descriptor_pool`을 호출하여 생성합니다.

```rust
// 구조체 정의
struct VulkanApp {
    ...
    descriptor_pool: vk::DescriptorPool,
    ...
}

// create_descriptor_pool 함수 내부
let pool_info = vk::DescriptorPoolCreateInfo::builder()
    .pool_sizes(&pool_sizes)
    .max_sets(MAX_FRAMES_IN_FLIGHT as u32);

self.descriptor_pool = unsafe {
    self.device
        .create_descriptor_pool(&pool_info, None)
}?;
```

## 디스크립터 셋 (Descriptor Set)

이제 디스크립터 셋 자체를 할당할 수 있습니다. 이를 위해 `create_descriptor_sets` 함수를 추가합시다.

```rust
// init_vulkan 함수 내부
...
self.create_descriptor_pool()?;
self.create_descriptor_sets()?;
...

// VulkanApp impl 블록 내부
fn create_descriptor_sets(&mut self) -> Result<()> {
    // 이 함수 내용을 채워나갑니다.
    Ok(())
}
```

`vk::DescriptorSetAllocateInfo` 구조체로 디스크립터 셋 할당을 기술합니다. 할당할 디스크립터 풀, 할당할 디스크립터 셋의 개수, 그리고 기반으로 할 디스크립터 셋 레이아웃을 지정해야 합니다.

```rust
// create_descriptor_sets 함수 내부
let layouts = vec![self.descriptor_set_layout; MAX_FRAMES_IN_FLIGHT];
let alloc_info = vk::DescriptorSetAllocateInfo::builder()
    .descriptor_pool(self.descriptor_pool)
    .set_layouts(&layouts);
```

우리의 경우, 각 프레임마다 하나의 디스크립터 셋을 생성하며, 모두 동일한 레이아웃을 가집니다. `ash`를 사용하면 `allocate_descriptor_sets` 함수가 `Vec<vk::DescriptorSet>`을 반환하므로 미리 벡터 크기를 조절할 필요가 없습니다.

구조체에 디스크립터 셋 핸들을 담을 필드를 추가하고 `allocate_descriptor_sets`로 할당합니다.

```rust
// 구조체 정의
struct VulkanApp {
    ...
    descriptor_sets: Vec<vk::DescriptorSet>,
    ...
}

// create_descriptor_sets 함수 내부
self.descriptor_sets = unsafe { self.device.allocate_descriptor_sets(&alloc_info) }?;
```

디스크립터 풀이 파괴될 때 자동으로 해제되므로, 디스크립터 셋을 명시적으로 정리할 필요는 없습니다. `allocate_descriptor_sets` 호출은 각각 하나의 유니폼 버퍼 디스크립터를 가진 디스크립터 셋들을 할당할 것입니다. `cleanup` (또는 `Drop` 구현)에서는 디스크립터 풀만 파괴하면 됩니다.

```rust
// cleanup 함수 또는 Drop 트레이트 구현 내부
...
unsafe {
    self.device.destroy_descriptor_pool(self.descriptor_pool, None);
    self.device.destroy_descriptor_set_layout(self.descriptor_set_layout, None);
}
...
```

이제 디스크립터 셋은 할당되었지만, 그 안의 디스크립터들은 아직 설정이 필요합니다. 이제 모든 디스크립터를 채우기 위한 루프를 추가합니다.

우리의 유니폼 버퍼 디스크립터처럼 버퍼를 참조하는 디스크립터는 `vk::DescriptorBufferInfo` 구조체로 설정합니다. 이 구조체는 버퍼와 디스크립터 데이터를 포함하는 버퍼 내의 영역을 지정합니다.

```rust
// create_descriptor_sets 함수 내부, 할당 후
for i in 0..MAX_FRAMES_IN_FLIGHT {
    let buffer_info = vk::DescriptorBufferInfo {
        buffer: self.uniform_buffers[i],
        offset: 0,
        range: std::mem::size_of::<UniformBufferObject>() as vk::DeviceSize,
    };
```

우리처럼 버퍼 전체를 덮어쓰는 경우, `range`에 `vk::WHOLE_SIZE` 값을 사용하는 것도 가능합니다. 디스크립터 설정은 `vk::WriteDescriptorSet` 구조체의 배열을 파라미터로 받는 `update_descriptor_sets` 함수를 사용하여 업데이트됩니다. `ash`의 빌더 패턴을 사용합시다.

```rust
    let buffer_infos = [buffer_info];
    let descriptor_write = vk::WriteDescriptorSet::builder()
        .dst_set(self.descriptor_sets[i])
        .dst_binding(0)
        .dst_array_element(0)
        .descriptor_type(vk::DescriptorType::UNIFORM_BUFFER)
        .buffer_info(&buffer_infos);
```

`ash`의 `update_descriptor_sets` 함수는 쓰기(write)와 복사(copy)를 위한 슬라이스를 받습니다. 우리는 쓰기만 할 것입니다.

```rust
    unsafe {
        self.device.update_descriptor_sets(&[descriptor_write.build()], &[]);
    }
}
```

`update_descriptor_sets`는 두 종류의 배열 슬라이스를 파라미터로 받습니다: `&[vk::WriteDescriptorSet]`와 `&[vk::CopyDescriptorSet]`입니다. 후자는 이름에서 알 수 있듯이 디스크립터를 서로 복사하는 데 사용할 수 있습니다.

## 디스크립터 셋 사용하기

이제 `record_command_buffer` 함수를 업데이트하여 `cmd_draw_indexed` 호출 전에 `cmd_bind_descriptor_sets`로 셰이더의 디스크립터에 각 프레임에 맞는 디스크립터 셋을 실제로 바인딩해야 합니다.

```rust
// record_command_buffer 함수 내부
unsafe {
    self.device.cmd_bind_descriptor_sets(
        command_buffer,
        vk::PipelineBindPoint::GRAPHICS,
        self.pipeline_layout,
        0,
        &[self.descriptor_sets[self.current_frame]],
        &[],
    );
    self.device.cmd_draw_indexed(
        command_buffer,
        self.indices.len() as u32,
        1,
        0,
        0,
        0,
    );
}
```

정점 및 인덱스 버퍼와 달리, 디스크립터 셋은 그래픽스 파이프라인에만 국한되지 않습니다. 따라서 디스크립터 셋을 그래픽스 파이프라인에 바인딩할지, 컴퓨트 파이프라인에 바인딩할지 지정해야 합니다. 다음 파라미터는 디스크립터가 기반으로 하는 레이아웃입니다. 그 다음 세 파라미터는 첫 번째 디스크립터 셋의 인덱스, 바인딩할 셋의 개수, 그리고 바인딩할 셋의 슬라이스를 지정합니다. 마지막 파라미터는 동적 디스크립터에 사용되는 오프셋 슬라이스이며, 이는 다음 장에서 살펴보겠습니다.

지금 프로그램을 실행해보면 안타깝게도 아무것도 보이지 않을 수 있습니다. 문제는 투영 행렬에서 Y축을 뒤집었기 때문에, 정점들이 시계 방향 대신 반시계 방향으로 그려진다는 것입니다. 이로 인해 후면 컬링(backface culling)이 작동하여 지오메트리가 그려지지 않게 됩니다. `create_graphics_pipeline` 함수로 가서 래스터화 상태의 `front_face`를 수정하여 이를 바로잡습니다.

```rust
// create_graphics_pipeline 함수 내부
let rasterizer = vk::PipelineRasterizationStateCreateInfo::builder()
    ...
    .cull_mode(vk::CullModeFlags::BACK)
    .front_face(vk::FrontFace::COUNTER_CLOCKWISE);
```

프로그램을 다시 실행하면 다음과 같은 화면을 볼 수 있습니다.

![](/images/spinning_quad.png)

투영 행렬이 이제 화면 비율을 보정하기 때문에 직사각형이 정사각형으로 변경되었습니다. `update_uniform_buffer`가 화면 크기 조정을 처리하므로 `recreate_swapchain`에서 디스크립터 셋을 다시 만들 필요는 없습니다.

## 정렬 요구사항 (Alignment Requirements)

지금까지 간과한 한 가지는 Rust 구조체의 데이터가 셰이더의 유니폼 정의와 정확히 어떻게 일치해야 하는가입니다. 예를 들어, 수학 라이브러리로 `glam`을 사용한다고 가정합시다.

```rust
// #[repr(C)]는 필드 순서를 보장합니다.
#[repr(C)] 
struct UniformBufferObject {
    model: glam::Mat4,
    view: glam::Mat4,
    proj: glam::Mat4,
}

// 셰이더
layout(binding = 0) uniform UniformBufferObject {
    mat4 model;
    mat4 view;
    mat4 proj;
} ubo;
```

하지만 이게 전부가 아닙니다. 예를 들어, 구조체와 셰이더를 다음과 같이 수정해보세요.

```rust
#[repr(C)]
struct UniformBufferObject {
    foo: glam::Vec2,
    model: glam::Mat4,
    view: glam::Mat4,
    proj: glam::Mat4,
}
```

셰이더와 프로그램을 다시 컴파일하고 실행하면, 다채로운 사각형이 사라질 것입니다! 이는 우리가 *정렬 요구사항(alignment requirements)*을 고려하지 않았기 때문입니다.

Vulkan은 구조체의 데이터가 메모리에서 특정 방식으로 정렬되기를 기대합니다. 예를 들면 다음과 같습니다:

*   스칼라는 N(32비트 부동소수점의 경우 4바이트)으로 정렬되어야 합니다.
*   `vec2`는 2N(8바이트)으로 정렬되어야 합니다.
*   `vec3` 또는 `vec4`는 4N(16바이트)으로 정렬되어야 합니다.
*   중첩 구조체는 멤버의 기본 정렬을 16의 배수로 올림한 값으로 정렬되어야 합니다.
*   `mat4` 행렬은 `vec4`와 동일한 정렬을 가져야 합니다.

`glam::Mat4`는 이미 16바이트 정렬이 되어 있어 원래 구조체는 문제가 없었습니다. 그러나 `glam::Vec2`는 8바이트 정렬을 가지므로, `model` 필드의 오프셋이 8이 되어 16의 배수가 아니게 됩니다. 이 문제를 해결하기 위해 Rust에서는 `#[repr(C, align(N))]` 속성을 사용할 수 있습니다.

```rust
#[repr(C)]
struct UniformBufferObject {
    foo: glam::Vec2,
    #[repr(align(16))]
    model: glam::Mat4,
    view: glam::Mat4,
    proj: glam::Mat4,
}
```

하지만 이 방법은 필드별로 적용하기 번거롭습니다. 더 나은 방법은 구조체 전체에 정렬을 적용하는 것입니다.

```rust
#[repr(C, align(16))]
struct UniformBufferObject {
    foo: glam::Vec2,
    model: glam::Mat4,
    view: glam::Mat4,
    proj: glam::Mat4,
}
```

이 코드는 작동하지 않습니다. `foo` 필드 뒤에 `model` 필드가 올바르게 정렬되도록 컴파일러가 패딩을 추가해야 하는데, `#[repr(C)]`는 이를 보장하지 않을 수 있습니다. 가장 안전하고 명확한 방법은 정렬이 필요한 각 멤버에 명시적으로 `align`을 지정하는 대신, `glam`과 같은 라이브러리가 제공하는 이미 정렬된 타입을 사용하는 것입니다. 다행히 `glam::Mat4`, `glam::Vec4` 등은 기본적으로 16바이트 정렬이 되어 있습니다. 문제가 발생한 `Vec2` 같은 타입의 경우, 수동으로 패딩을 추가하거나 구조를 변경해야 할 수 있습니다.

이러한 함정을 피하기 위해, 항상 정렬에 대해 명시적인 것이 좋습니다. 최종적으로 우리의 UBO 구조체는 다음과 같이 명시적으로 정렬을 보장하는 것이 가장 안전합니다.

```rust
#[repr(C)]
#[derive(Copy, Clone, Debug)]
pub struct UniformBufferObject {
    pub model: glam::Mat4,
    pub view: glam::Mat4,
    pub proj: glam::Mat4,
}
```
위 `glam`의 기본 타입들은 이미 16바이트 정렬이 되어 있어 추가적인 `align` 속성 없이도 대부분의 경우 잘 동작합니다. 하지만 중첩 구조체에서는 문제가 발생할 수 있습니다.
```rust
#[repr(C)]
struct Foo {
    v: glam::Vec2, // 8바이트 정렬
}

#[repr(C)]
struct UniformBufferObject {
    f1: Foo,
    f2: Foo, // 이 필드의 오프셋은 8이 되며, 16이어야 합니다.
}
```
이런 경우, 명시적으로 정렬을 지정해야 합니다.
```rust
#[repr(C, align(16))]
struct Foo {
    v: glam::Vec2,
}

#[repr(C)]
struct UniformBufferObject {
    f1: Foo,
    f2: Foo, // 이제 f1과 f2 모두 16바이트 경계에 정렬됩니다.
}
```

정렬 오류는 디버깅하기 매우 까다로우므로, 유니폼 버퍼로 사용할 구조체는 `#[repr(C)]`와 함께 필요에 따라 `align` 속성을 명시하는 것이 좋습니다.

우리의 원래 `UniformBufferObject`로 돌아가 `foo` 필드를 제거하고 셰이더를 다시 컴파일하는 것을 잊지 마세요.

## 여러 개의 디스크립터 셋 (Multiple Descriptor Sets)

일부 구조체와 함수 호출에서 암시되었듯이, 여러 디스크립터 셋을 동시에 바인딩하는 것도 가능합니다. 파이프라인 레이아웃을 생성할 때 각 디스크립터 셋에 대한 디스크립터 셋 레이아웃을 지정해야 합니다. 그러면 셰이더는 다음과 같이 특정 디스크립터 셋을 참조할 수 있습니다.

```glsl
layout(set = 0, binding = 0) uniform UniformBufferObject { ... }
```

이 기능을 사용하면 객체별로 다른 디스크립터와 공유되는 디스크립터를 별도의 디스크립터 셋에 넣을 수 있습니다. 이 경우 드로우 콜 간에 대부분의 디스크립터를 다시 바인딩하는 것을 피할 수 있어 잠재적으로 더 효율적입니다.

[Rust 코드](/path/to/your/code.rs) /
[정점 셰이더](/code/22_shader_ubo.vert) /
[프래그먼트 셰이더](/code/22_shader_ubo.frag)