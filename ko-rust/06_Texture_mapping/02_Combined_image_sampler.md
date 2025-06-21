## 소개

우리는 유니폼 버퍼(uniform buffers) 파트에서 처음으로 디스크립터(descriptor)를 살펴보았습니다. 이번 챕터에서는 새로운 유형의 디스크립터인 **결합 이미지 샘플러(combined image sampler)**에 대해 알아보겠습니다. 이 디스크립터를 사용하면 셰이더가 이전 챕터에서 생성한 것과 같은 샘플러 객체를 통해 이미지 리소스에 접근할 수 있습니다.

우선 디스크립터 셋 레이아웃, 디스크립터 풀, 디스크립터 셋을 수정하여 결합 이미지 샘플러 디스크립터를 포함하는 것부터 시작하겠습니다. 그 후, `Vertex`에 텍스처 좌표를 추가하고 프래그먼트 셰이더를 수정하여 단순히 정점 색상을 보간하는 대신 텍스처에서 색상을 읽도록 할 것입니다.

*이 문서는 기존 Rust 및 `ash`로 작성된 Vulkan 애플리케이션 구조를 따른다고 가정합니다.*

## 디스크립터 업데이트하기

`create_descriptor_set_layout` 함수로 가서 결합 이미지 샘플러 디스크립터를 위한 `vk::DescriptorSetLayoutBinding`을 추가합니다. 단순히 유니폼 버퍼 다음 바인딩에 추가하겠습니다. `ash`의 빌더 패턴을 사용하면 코드가 더 명확해집니다.

```rust
let ubo_layout_binding = vk::DescriptorSetLayoutBinding::builder()
    .binding(0)
    .descriptor_type(vk::DescriptorType::UNIFORM_BUFFER)
    .descriptor_count(1)
    .stage_flags(vk::ShaderStageFlags::VERTEX);

let sampler_layout_binding = vk::DescriptorSetLayoutBinding::builder()
    .binding(1)
    .descriptor_count(1)
    .descriptor_type(vk::DescriptorType::COMBINED_IMAGE_SAMPLER)
    .p_immutable_samplers(&[])
    .stage_flags(vk::ShaderStageFlags::FRAGMENT);

let bindings = [ubo_layout_binding.build(), sampler_layout_binding.build()];
let layout_info = vk::DescriptorSetLayoutCreateInfo::builder()
    .bindings(&bindings);

self.descriptor_set_layout = unsafe {
    device.create_descriptor_set_layout(&layout_info, None)
}?;
```

`stage_flags`를 `vk::ShaderStageFlags::FRAGMENT`로 설정하여 프래그먼트 셰이더에서 결합 이미지 샘플러 디스크립터를 사용하려는 의도를 나타내야 합니다. 프래그먼트의 색상이 결정되는 곳이 바로 여기입니다. 버텍스 셰이더에서 텍스처 샘플링을 사용하는 것도 가능합니다. 예를 들어, [하이트맵(heightmap)](https://en.wikipedia.org/wiki/Heightmap)을 사용하여 정점 그리드를 동적으로 변형시킬 수 있습니다.

또한 `vk::DescriptorType::COMBINED_IMAGE_SAMPLER` 타입의 풀 크기를 추가하여 결합 이미지 샘플러 할당을 위한 공간을 만들기 위해 더 큰 디스크립터 풀을 생성해야 합니다. `create_descriptor_pool` 함수로 가서 이 디스크립터를 위한 `vk::DescriptorPoolSize`를 포함하도록 수정합니다.

```rust
let pool_sizes = [
    vk::DescriptorPoolSize {
        ty: vk::DescriptorType::UNIFORM_BUFFER,
        descriptor_count: MAX_FRAMES_IN_FLIGHT as u32,
    },
    vk::DescriptorPoolSize {
        ty: vk::DescriptorType::COMBINED_IMAGE_SAMPLER,
        descriptor_count: MAX_FRAMES_IN_FLIGHT as u32,
    },
];

let pool_info = vk::DescriptorPoolCreateInfo::builder()
    .pool_sizes(&pool_sizes)
    .max_sets(MAX_FRAMES_IN_FLIGHT as u32);

self.descriptor_pool = unsafe {
    device.create_descriptor_pool(&pool_info, None)
}?;
```

부적절한 디스크립터 풀은 검증 레이어가 잡아내지 못하는 문제의 좋은 예입니다. Vulkan 1.1부터, 풀이 충분히 크지 않으면 `vkAllocateDescriptorSets`가 `VK_ERROR_POOL_OUT_OF_MEMORY` 오류 코드로 실패할 수 있지만, 드라이버가 내부적으로 이 문제를 해결하려고 시도할 수도 있습니다. 이는 때때로 (하드웨어, 풀 크기, 할당 크기에 따라) 드라이버가 디스크립터 풀의 한도를 초과하는 할당을 허용할 수 있음을 의미합니다. 다른 경우에는 `vkAllocateDescriptorSets`가 실패하고 `VK_ERROR_POOL_OUT_OF_MEMORY`를 반환합니다. 이는 일부 머신에서는 할당이 성공하고 다른 머신에서는 실패할 경우 특히 좌절스러울 수 있습니다.

Vulkan은 할당에 대한 책임을 드라이버에게 넘기므로, 디스크립터 풀 생성 시 해당 `descriptor_count` 멤버로 지정된 만큼만 특정 유형의 디스크립터를 할당하는 것이 더 이상 엄격한 요구 사항은 아닙니다. 하지만, 여전히 그렇게 하는 것이 모범 사례로 남아 있습니다.

마지막 단계는 실제 이미지와 샘플러 리소스를 디스크립터 셋의 디스크립터에 바인딩하는 것입니다. `create_descriptor_sets` 함수로 가세요.

```rust
for i in 0..MAX_FRAMES_IN_FLIGHT {
    let buffer_info = [vk::DescriptorBufferInfo {
        buffer: self.uniform_buffers[i],
        offset: 0,
        range: std::mem::size_of::<UniformBufferObject>() as u64,
    }];

    let image_info = [vk::DescriptorImageInfo {
        image_layout: vk::ImageLayout::SHADER_READ_ONLY_OPTIMAL,
        image_view: self.texture_image_view,
        sampler: self.texture_sampler,
    }];

    // ...
}
```

결합 이미지 샘플러 구조를 위한 리소스는 유니폼 버퍼 디스크립터의 버퍼 리소스가 `vk::DescriptorBufferInfo` 구조체에 지정되는 것과 마찬가지로, `vk::DescriptorImageInfo` 구조체에 지정되어야 합니다. 여기서 이전 챕터의 객체들이 함께 사용됩니다.

```rust
let descriptor_writes = [
    vk::WriteDescriptorSet::builder()
        .dst_set(self.descriptor_sets[i])
        .dst_binding(0)
        .dst_array_element(0)
        .descriptor_type(vk::DescriptorType::UNIFORM_BUFFER)
        .buffer_info(&buffer_info)
        .build(),
    vk::WriteDescriptorSet::builder()
        .dst_set(self.descriptor_sets[i])
        .dst_binding(1)
        .dst_array_element(0)
        .descriptor_type(vk::DescriptorType::COMBINED_IMAGE_SAMPLER)
        .image_info(&image_info)
        .build(),
];

unsafe {
    device.update_descriptor_sets(&descriptor_writes, &[]);
}
```

디스크립터는 이 이미지 정보로 업데이트되어야 합니다. 이번에는 `buffer_info` 슬라이스 대신 `image_info` 슬라이스를 사용합니다. 이제 디스크립터는 셰이더에서 사용할 준비가 되었습니다!

## 텍스처 좌표

텍스처 매핑에 있어 아직 빠진 중요한 요소가 하나 있는데, 바로 각 정점에 대한 실제 텍스처 좌표입니다. 텍스처 좌표는 이미지가 지오메트리에 실제로 어떻게 매핑될지 결정합니다.

```rust
// glam이나 다른 수학 라이브러리를 사용한다고 가정합니다.
use glam::{Vec2, Vec3};
// C++의 offsetof와 같은 기능을 위해 memoffset 크레이트가 필요합니다.
// Cargo.toml에 memoffset = "0.9" 를 추가하세요.
use memoffset::offset_of;

#[repr(C)]
#[derive(Copy, Clone, Debug)]
struct Vertex {
    pos: Vec2,
    color: Vec3,
    tex_coord: Vec2,
}

impl Vertex {
    fn get_binding_description() -> vk::VertexInputBindingDescription {
        vk::VertexInputBindingDescription {
            binding: 0,
            stride: std::mem::size_of::<Self>() as u32,
            input_rate: vk::VertexInputRate::VERTEX,
        }
    }

    fn get_attribute_descriptions() -> [vk::VertexInputAttributeDescription; 3] {
        [
            vk::VertexInputAttributeDescription {
                binding: 0,
                location: 0,
                format: vk::Format::R32G32_SFLOAT,
                offset: offset_of!(Self, pos) as u32,
            },
            vk::VertexInputAttributeDescription {
                binding: 0,
                location: 1,
                format: vk::Format::R32G32B32_SFLOAT,
                offset: offset_of!(Self, color) as u32,
            },
            vk::VertexInputAttributeDescription {
                binding: 0,
                location: 2,
                format: vk::Format::R32G32_SFLOAT,
                offset: offset_of!(Self, tex_coord) as u32,
            },
        ]
    }
}
```

`Vertex` 구조체를 수정하여 텍스처 좌표를 위한 `Vec2` 타입의 `tex_coord` 필드를 포함시킵니다. 또한 버텍스 셰이더에서 텍스처 좌표를 입력으로 접근할 수 있도록 `vk::VertexInputAttributeDescription`을 추가해야 합니다. 이는 사각형 표면 전체에 걸쳐 보간을 위해 프래그먼트 셰이더로 전달하는 데 필요합니다. Rust에서는 멤버의 오프셋을 얻기 위해 `memoffset` 크레이트의 `offset_of!` 매크로를 사용하는 것이 일반적입니다.

```rust
const VERTICES: [Vertex; 4] = [
    Vertex { pos: Vec2::new(-0.5, -0.5), color: Vec3::new(1.0, 0.0, 0.0), tex_coord: Vec2::new(1.0, 0.0) },
    Vertex { pos: Vec2::new(0.5, -0.5), color: Vec3::new(0.0, 1.0, 0.0), tex_coord: Vec2::new(0.0, 0.0) },
    Vertex { pos: Vec2::new(0.5, 0.5), color: Vec3::new(0.0, 0.0, 1.0), tex_coord: Vec2::new(0.0, 1.0) },
    Vertex { pos: Vec2::new(-0.5, 0.5), color: Vec3::new(1.0, 1.0, 1.0), tex_coord: Vec2::new(1.0, 1.0) },
];
```

이 튜토리얼에서는 왼쪽 위 모서리의 `(0, 0)`에서 오른쪽 아래 모서리의 `(1, 1)`까지의 좌표를 사용하여 텍스처로 사각형을 채울 것입니다. 자유롭게 다른 좌표로 실험해보세요. `0` 미만 또는 `1` 초과의 좌표를 사용하여 주소 지정 모드가 실제로 어떻게 작동하는지 확인해보세요!

## 셰이더

마지막 단계는 셰이더를 수정하여 텍스처에서 색상을 샘플링하는 것입니다. 먼저 버텍스 셰이더를 수정하여 텍스처 좌표를 프래그먼트 셰이더로 전달해야 합니다.

```glsl
// vertex shader
layout(location = 0) in vec2 inPosition;
layout(location = 1) in vec3 inColor;
layout(location = 2) in vec2 inTexCoord;

layout(location = 0) out vec3 fragColor;
layout(location = 1) out vec2 fragTexCoord;

void main() {
    gl_Position = ubo.proj * ubo.view * ubo.model * vec4(inPosition, 0.0, 1.0);
    fragColor = inColor;
    fragTexCoord = inTexCoord;
}
```

정점별 색상과 마찬가지로 `fragTexCoord` 값은 래스터라이저에 의해 사각형 영역 전체에 걸쳐 부드럽게 보간됩니다. 프래그먼트 셰이더가 텍스처 좌표를 색상으로 출력하게 하여 이를 시각화할 수 있습니다.

```glsl
// fragment shader
#version 450

layout(location = 0) in vec3 fragColor;
layout(location = 1) in vec2 fragTexCoord;

layout(location = 0) out vec4 outColor;

void main() {
    outColor = vec4(fragTexCoord, 0.0, 1.0);
}
```

셰이더를 다시 컴파일하는 것을 잊지 마세요! 아래와 같은 이미지를 보게 될 것입니다.

![](/images/texcoord_visualization.png)

녹색 채널은 수평 좌표를, 적색 채널은 수직 좌표를 나타냅니다. 검은색과 노란색 모서리는 텍스처 좌표가 사각형에 걸쳐 `0, 0`에서 `1, 1`까지 올바르게 보간되었음을 확인시켜 줍니다. 색상을 사용한 데이터 시각화는 셰이더 프로그래밍에서 더 나은 대안이 없을 때 사용하는 `println!` 디버깅과 같습니다!

결합 이미지 샘플러 디스크립터는 GLSL에서 샘플러 유니폼으로 표현됩니다. 프래그먼트 셰이더에 이에 대한 참조를 추가하세요.

```glsl
// fragment shader
layout(binding = 1) uniform sampler2D texSampler;
```

다른 유형의 이미지를 위한 `sampler1D` 및 `sampler3D`와 같은 타입도 있습니다. 여기서 올바른 바인딩(`binding = 1`)을 사용해야 합니다.

```glsl
// fragment shader main()
void main() {
    outColor = texture(texSampler, fragTexCoord);
}
```

텍스처는 내장 함수 `texture`를 사용하여 샘플링됩니다. 이 함수는 `sampler`와 좌표를 인수로 받습니다. 샘플러는 백그라운드에서 필터링과 변환을 자동으로 처리합니다. 이제 애플리케이션을 실행하면 사각형 위에 텍스처가 표시될 것입니다.

![](/images/texture_on_square.png)

텍스처 좌표를 `1`보다 큰 값으로 조정하여 주소 지정 모드를 실험해보세요. 예를 들어, 다음 프래그먼트 셰이더는 `VK_SAMPLER_ADDRESS_MODE_REPEAT`를 사용할 때 아래 이미지와 같은 결과를 생성합니다.

```glsl
// fragment shader main()
void main() {
    outColor = texture(texSampler, fragTexCoord * 2.0);
}
```

![](/images/texture_on_square_repeated.png)

정점 색상을 사용하여 텍스처 색상을 조작할 수도 있습니다.

```glsl
// fragment shader main()
void main() {
    outColor = vec4(fragColor * texture(texSampler, fragTexCoord).rgb, 1.0);
}
```

알파 채널이 변하지 않도록 RGB와 알파 채널을 분리했습니다.

![](/images/texture_on_square_colorized.png)

이제 셰이더에서 이미지에 접근하는 방법을 알게 되었습니다! 이 기술은 프레임버퍼에 쓰여지기도 하는 이미지와 결합될 때 매우 강력합니다. 이러한 이미지를 입력으로 사용하여 후처리(post-processing)나 3D 세계 내 카메라 디스플레이와 같은 멋진 효과를 구현할 수 있습니다.