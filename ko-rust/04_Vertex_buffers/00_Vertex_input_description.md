## 소개

다음 몇 개의 챕터에서는 버텍스 셰이더에 하드코딩된 정점 데이터를 메모리의 버텍스 버퍼로 교체할 것입니다. 가장 쉬운 접근법으로 시작하여, CPU에서 볼 수 있는(visible) 버퍼를 만들고 메모리 복사를 통해 정점 데이터를 직접 GPU로 전달하는 방법을 알아볼 것입니다. 그 후에는 스테이징 버퍼(staging buffer)를 사용해 정점 데이터를 고성능 메모리로 복사하는 방법도 살펴볼 것입니다.

## 버텍스 셰이더

먼저 버텍스 셰이더를 변경하여, 셰이더 코드 자체에 더 이상 정점 데이터를 포함하지 않도록 합니다. 버텍스 셰이더는 `in` 키워드를 사용하여 버텍스 버퍼로부터 입력을 받습니다.

```glsl
#version 450

layout(location = 0) in vec2 inPosition;
layout(location = 1) in vec3 inColor;

layout(location = 0) out vec3 fragColor;

void main() {
    gl_Position = vec4(inPosition, 0.0, 1.0);
    fragColor = inColor;
}
```

`inPosition`과 `inColor` 변수는 **정점 속성(vertex attributes)**입니다. 이들은 우리가 이전에 수동으로 위치와 색상을 지정했던 것처럼, 버텍스 버퍼에서 정점 단위로 지정되는 속성입니다. 버텍스 셰이더를 다시 컴파일하는 것을 잊지 마세요!

`fragColor`와 마찬가지로, `layout(location = x)` 어노테이션은 나중에 참조할 수 있도록 입력에 인덱스를 할당합니다. 64비트 벡터인 `dvec3` 같은 일부 타입은 여러 개의 **슬롯(slot)**을 사용한다는 점을 아는 것이 중요합니다. 즉, 그 다음의 인덱스는 최소 2 이상 커야 합니다.

```glsl
layout(location = 0) in dvec3 inPosition;
layout(location = 2) in vec3 inColor;
```

레이아웃 한정자(layout qualifier)에 대한 더 많은 정보는 [OpenGL 위키](https://www.khronos.org/opengl/wiki/Layout_Qualifier_(GLSL))에서 찾을 수 있습니다.

## 정점 데이터

이제 정점 데이터를 셰이더 코드에서 우리 Rust 애플리케이션의 코드로 옮길 것입니다. 먼저 Rust에서 가장 널리 사용되는 선형대수 라이브러리인 `glam`을 `Cargo.toml`에 추가합니다. `glam`은 벡터나 행렬 같은 타입을 제공합니다.

```toml
[dependencies]
glam = "0.24"
```

이제 버텍스 셰이더에서 사용할 두 속성을 포함하는 `Vertex` 구조체를 새로 정의합니다. `#[repr(C)]` 어트리뷰트는 Rust 컴파일러가 C와 호환되도록 필드 순서를 보장하게 만들어, 메모리 레이아웃을 예측 가능하게 합니다. 이는 Vulkan API와 상호작용할 때 필수적입니다.

```rust
use glam::{Vec2, Vec3};

#[repr(C)]
#[derive(Clone, Debug, Copy)]
struct Vertex {
    pos: Vec2,
    color: Vec3,
}
```

`glam`은 셰이더 언어에서 사용되는 벡터 타입과 정확히 일치하는 Rust 타입을 편리하게 제공합니다.

이제 `Vertex` 구조체를 사용하여 정점 데이터 배열을 정의합니다. 이전과 정확히 같은 위치와 색상 값을 사용하지만, 이제는 하나의 정점 배열로 결합되었습니다. 이를 **인터리빙(interleaving)** 정점 속성이라고 합니다.

```rust
const VERTICES: [Vertex; 3] = [
    Vertex { pos: Vec2::new(0.0, -0.5), color: Vec3::new(1.0, 0.0, 0.0) },
    Vertex { pos: Vec2::new(0.5, 0.5), color: Vec3::new(0.0, 1.0, 0.0) },
    Vertex { pos: Vec2::new(-0.5, 0.5), color: Vec3::new(0.0, 0.0, 1.0) },
];
```

## 바인딩 서술 (Binding descriptions)

다음 단계는 이 데이터 포맷이 GPU 메모리에 업로드된 후, 버텍스 셰이더로 어떻게 전달될지를 Vulkan에게 알려주는 것입니다. 이를 위해 두 종류의 구조체를 설정해야 합니다.

첫 번째 구조체는 `ash::vk::VertexInputBindingDescription`입니다. `Vertex` 구조체에 대한 `impl` 블록 내에 연관 함수(associated function)를 추가하여 이 구조체를 생성하도록 하겠습니다.

```rust
use ash::vk;

impl Vertex {
    pub fn get_binding_description() -> vk::VertexInputBindingDescription {
        vk::VertexInputBindingDescription {
            binding: 0,
            stride: std::mem::size_of::<Self>() as u32,
            input_rate: vk::VertexInputRate::VERTEX,
        }
    }
}
```

정점 바인딩(vertex binding)은 정점들 전체에서 메모리로부터 데이터를 어떤 속도(rate)로 로드할지 서술합니다. 이는 데이터 항목 사이의 바이트 수와 각 정점 또는 각 인스턴스 이후에 다음 데이터 항목으로 이동할지 여부를 지정합니다.

*   `binding`: 바인딩 배열에서의 인덱스를 지정합니다. 우리는 하나의 바인딩만 사용하므로 `0`입니다.
*   `stride`: 한 정점 데이터에서 다음 정점 데이터까지의 바이트 거리입니다. `std::mem::size_of`를 사용하여 `Vertex` 구조체의 크기를 가져옵니다.
*   `input_rate`: 다음 값 중 하나를 가집니다.
    *   `vk::VertexInputRate::VERTEX`: 각 정점마다 다음 데이터 항목으로 이동합니다.
    *   `vk::VertexInputRate::INSTANCE`: 각 인스턴스마다 다음 데이터 항목으로 이동합니다.

우리는 인스턴스 렌더링을 사용하지 않으므로, 정점별 데이터(`VERTEX`)를 사용합니다.

## 속성 서술 (Attribute descriptions)

정점 입력을 처리하는 방법을 서술하는 두 번째 구조체는 `ash::vk::VertexInputAttributeDescription`입니다. 이 구조체들을 채우기 위해 `Vertex`에 또 다른 연관 함수를 추가할 것입니다.

필드 오프셋을 안전하게 계산하기 위해 `memoffset` 크레이트가 필요합니다. `Cargo.toml`에 추가해 주세요.

```toml
[dependencies]
memoffset = "0.9"
```

```rust
use memoffset::offset_of;

impl Vertex {
    //... get_binding_description() ...

    pub fn get_attribute_descriptions() -> [vk::VertexInputAttributeDescription; 2] {
        [
            vk::VertexInputAttributeDescription {
                binding: 0,
                location: 0,
                format: vk::Format::R32G32_SFLOAT,
                offset: offset_of!(Vertex, pos) as u32,
            },
            vk::VertexInputAttributeDescription {
                binding: 0,
                location: 1,
                format: vk::Format::R32G32B32_SFLOAT,
                offset: offset_of!(Vertex, color) as u32,
            },
        ]
    }
}
```

속성 서술(attribute description)은 바인딩에서 제공된 정점 데이터 덩어리로부터 특정 속성을 어떻게 추출할지 설명합니다. 우리는 위치와 색상, 두 가지 속성을 가지고 있으므로 두 개의 속성 서술이 필요합니다.

*   `binding`: 정점별 데이터가 어느 바인딩에서 오는지를 지정합니다. (`0`)
*   `location`: 버텍스 셰이더의 `layout(location = ...)` 지시어에 해당합니다. `location 0`은 위치, `location 1`은 색상입니다.
*   `format`: 속성의 데이터 타입을 서술합니다. 포맷은 색상 포맷과 동일한 열거형으로 지정됩니다.
    *   `Vec2`: `vk::Format::R32G32_SFLOAT` (2 x 32비트 부동소수점)
    *   `Vec3`: `vk::Format::R32G32B32_SFLOAT` (3 x 32비트 부동소수점)
*   `offset`: 정점 데이터의 시작 지점으로부터 해당 속성까지의 바이트 오프셋입니다. `memoffset::offset_of!` 매크로를 사용하여 컴파일 타임에 안전하게 계산합니다.

## 파이프라인 정점 입력

이제 `create_graphics_pipeline` 함수에서 이 정보들을 참조하여, 그래픽 파이프라인이 해당 포맷의 정점 데이터를 받도록 설정해야 합니다. `ash`가 제공하는 빌더(builder) 패턴을 사용하면 코드를 더 안전하고 간결하게 작성할 수 있습니다.

```rust
let binding_descriptions = [Vertex::get_binding_description()];
let attribute_descriptions = Vertex::get_attribute_descriptions();

let vertex_input_info = vk::PipelineVertexInputStateCreateInfo::builder()
    .vertex_binding_descriptions(&binding_descriptions)
    .vertex_attribute_descriptions(&attribute_descriptions);
```
`ash`의 빌더는 슬라이스(`&[...]`)를 인자로 받으므로, C++ 버전처럼 개수(count)와 포인터(pointer)를 수동으로 설정할 필요가 없습니다. 빌더가 내부적으로 처리해주기 때문에 메모리 안전성이 높고 코드가 깔끔해집니다. 이 `vertex_input_info` 빌더를 파이프라인 생성 정보에 전달하면 됩니다.

이제 파이프라인은 `VERTICES` 배열과 같은 포맷의 정점 데이터를 받아들여 우리 버텍스 셰이더로 전달할 준비가 되었습니다. 만약 지금 검증 레이어를 활성화한 상태로 프로그램을 실행하면, 바인딩에 연결된 버텍스 버퍼가 없다고 경고하는 것을 볼 수 있을 것입니다. 다음 단계는 버텍스 버퍼를 생성하고 정점 데이터를 그곳으로 옮겨 GPU가 접근할 수 있도록 하는 것입니다.

[Rust 코드](/rust_code/src/part18_vertex_input/main.rs) /
[버텍스 셰이더](/code/18_shader_vertexbuffer.vert) /
[프래그먼트 셰이더](/code/18_shader_vertexbuffer.frag)