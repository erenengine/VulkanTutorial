## 소개

이제 여러분의 프로그램은 텍스처가 입혀진 3D 메시를 렌더링할 준비가 되었습니다. 하지만 현재 `vertices`와 `indices` 배열에 있는 지오메트리는 아직 그다지 흥미롭지 않습니다. 이번 챕터에서는 그래픽 카드가 실제로 어떤 작업을 하도록 만들기 위해, 실제 모델 파일에서 정점과 인덱스를 로드하도록 프로그램을 확장할 것입니다.

많은 그래픽 API 튜토리얼에서는 이와 같은 챕터에서 독자에게 직접 OBJ 로더를 작성하도록 합니다. 하지만 이 방식의 문제점은, 조금이라도 흥미로운 3D 애플리케이션이라면 곧 골격 애니메이션(skeletal animation)과 같이 OBJ 파일 형식이 지원하지 않는 기능이 필요해진다는 것입니다. 이번 챕터에서 OBJ 모델로부터 메시 데이터를 로드하긴 하겠지만, 파일에서 메시 데이터를 로드하는 세부 사항보다는, 메시 데이터를 프로그램 자체에 통합하는 데 더 중점을 둘 것입니다.

## 라이브러리

정점과 면(face)을 OBJ 파일에서 로드하기 위해 [tobj](https://github.com/tobj-rs/tobj) 크레이트를 사용할 것입니다. 이 라이브러리는 널리 사용되며 `Cargo`를 통해 쉽게 통합할 수 있습니다.

`Cargo.toml` 파일을 열고 의존성 목록에 `tobj`를 추가하세요. 또한 벡터 수학을 위해 `glam` 라이브러리를 사용하고 있으며, 나중에 해싱을 위해 `Hash` 기능이 필요하므로 함께 추가합니다.

```toml
[dependencies]
ash = "0.37"
# ... 다른 의존성들
tobj = "4.0"
glam = { version = "0.27", features = ["hash"] }
```

## 샘플 메시

이번 챕터에서는 아직 조명을 활성화하지 않을 것이므로, 텍스처에 조명이 미리 구워진(baked) 샘플 모델을 사용하는 것이 도움이 됩니다. 이러한 모델을 찾는 쉬운 방법은 [Sketchfab](https://sketchfab.com/)에서 3D 스캔 모델을 찾아보는 것입니다. 해당 사이트의 많은 모델이 허용적인 라이선스와 함께 OBJ 형식으로 제공됩니다.

이 튜토리얼에서는 [nigelgoh](https://sketchfab.com/nigelgoh)의 [Viking room](https://sketchfab.com/3d-models/viking-room-a49f1b8e4f5c4ecf9e1fe7d81915ad38) 모델([CC BY 4.0](https://web.archive.org/web/20200428202538/https://sketchfab.com/3d-models/viking-room-a49f1b8e4f5c4ecf9e1fe7d81915ad38))을 사용하기로 결정했습니다. 현재 지오메트리를 바로 대체하여 사용할 수 있도록 모델의 크기와 방향을 조정했습니다:

*   [viking_room.obj](/resources/viking_room.obj)
*   [viking_room.png](/resources/viking_room.png)

자신만의 모델을 자유롭게 사용해도 되지만, 해당 모델이 단 하나의 재질(material)로만 구성되어 있고 크기가 약 1.5 x 1.5 x 1.5 단위인지 확인하세요. 이보다 크면 뷰 행렬을 변경해야 합니다. 모델 파일을 `shaders`와 `textures` 옆에 새로운 `models` 디렉터리를 만들어 넣고, 텍스처 이미지는 `textures` 디렉터리에 넣으세요.

모델과 텍스처 경로를 정의하기 위해 프로그램에 두 개의 새로운 상수를 추가하세요:

```rust
const WINDOW_WIDTH: u32 = 800;
const WINDOW_HEIGHT: u32 = 600;

const MODEL_PATH: &str = "models/viking_room.obj";
const TEXTURE_PATH: &str = "textures/viking_room.png";
```

그리고 `create_texture_image`가 이 경로 상수를 사용하도록 업데이트하세요:

```rust
let image = image::open(TEXTURE_PATH)
    .expect("Failed to open texture image!")
    .to_rgba8();
// ...
```

## 정점과 인덱스 로드하기

이제 모델 파일에서 정점과 인덱스를 로드할 것이므로, 하드코딩된 `VERTICES`와 `INDICES` 상수를 제거해야 합니다. 이들을 `App` 구조체의 동적 필드로 교체하세요:

```rust
struct App {
    // ...
    vertices: Vec<Vertex>,
    indices: Vec<u32>,
    vertex_buffer: vk::Buffer,
    vertex_buffer_memory: vk::DeviceMemory,
    // ...
}
```

정점의 개수가 65,535개를 초과할 것이기 때문에, 인덱스의 타입을 `u16`에서 `u32`로 변경해야 합니다. `vkCmdBindIndexBuffer` 호출 시 사용하는 `vk::IndexType`도 잊지 말고 변경하세요:

```rust
device.cmd_bind_index_buffer(command_buffer, self.index_buffer, 0, vk::IndexType::UINT32);
```

이제 `tobj` 크레이트를 사용하여 `vertices`와 `indices` `Vec`을 메시의 정점 데이터로 채우는 `load_model` 함수를 작성할 것입니다. 이 함수는 정점 및 인덱스 버퍼가 생성되기 전 어딘가에서 호출되어야 합니다:

```rust
// in App::new()
// ...
self.load_model();
self.create_vertex_buffer();
self.create_index_buffer();
// ...
```

`App`에 `load_model` 메서드를 구현합시다:

```rust
impl App {
    // ...
    fn load_model(&mut self) {
        let (models, _materials) = tobj::load_obj(
            MODEL_PATH,
            &tobj::LoadOptions {
                triangulate: true,
                ..Default::default()
            },
        )
        .expect("Failed to load OBJ model");

        // 우리는 모든 모델의 지오메트리를 하나로 합칠 것입니다.
        for model in models {
            // ...
        }
    }
}
```

`tobj::load_obj` 함수는 모델을 라이브러리의 데이터 구조로 로드합니다. 이 함수는 모델과 재질의 `Result`를 반환합니다. 우리는 간단하게 `expect`를 사용하여 오류를 처리합니다. `LoadOptions`의 `triangulate` 필드를 `true`로 설정하면 사각형이나 다각형 면을 가진 모델도 자동으로 삼각형으로 변환해줍니다.

`tobj`가 반환하는 `Model` 구조체는 하나의 `Mesh`를 포함하고, 이 `Mesh`는 `positions`, `normals`, `texcoords`, 그리고 `indices`와 같은 `Vec`들을 가지고 있습니다. OBJ 파일의 모든 면을 단일 모델로 결합할 것이므로, 모든 `model`을 순회합니다. 이 튜토리얼에서는 가장 간단한 경우를 다루며, 파일에 하나의 모델만 있다고 가정합니다.

```rust
// load_model 메서드 내부
let model = &models[0];
let mesh = &model.mesh;

for i in 0..mesh.indices.len() {
    let index = mesh.indices[i] as usize;

    let vertex = Vertex {
        pos: glam::vec3(
            mesh.positions[3 * index],
            mesh.positions[3 * index + 1],
            mesh.positions[3 * index + 2],
        ),
        tex_coord: glam::vec2(
            mesh.texcoords[2 * index],
            mesh.texcoords[2 * index + 1],
        ),
        color: glam::vec3(1.0, 1.0, 1.0),
    };

    self.vertices.push(vertex);
    self.indices.push(i as u32);
}
```

`tobj`는 정점 속성을 분리된 `Vec`들로 로드합니다. `mesh.positions`는 `f32`의 `Vec`이며, 3개의 `f32`가 하나의 3D 위치를 구성합니다. `mesh.indices`는 삼각형을 구성하는 정점 인덱스를 담고 있습니다. 우리는 이 인덱스를 사용하여 `positions`와 `texcoords` `Vec`에서 올바른 속성을 조회합니다.

단순화를 위해, 지금은 모든 정점이 고유하다고 가정하고 `0, 1, 2, ...` 순서로 인덱스를 생성합니다.

이제 최적화 빌드로 프로그램을 실행하세요 (예: `cargo run --release`). 최적화가 없으면 모델 로딩이 매우 느려지므로 이 과정이 필요합니다. 다음과 같은 화면을 볼 수 있을 것입니다:

![](/images/inverted_texture_coordinates.png)

좋습니다, 지오메트리는 올바르게 보이지만 텍스처는 왜 저럴까요? OBJ 형식은 수직 좌표 `0`이 이미지의 하단을 의미하는 좌표계를 가정하지만, 우리는 이미지를 Vulkan에 업로드할 때 `0`이 상단을 의미하는 위에서 아래 방향으로 업로드했습니다. 텍스처 좌표의 수직 성분(`v`)을 뒤집어서 이 문제를 해결하세요:

```rust
// ...
tex_coord: glam::vec2(
    mesh.texcoords[2 * index],
    1.0 - mesh.texcoords[2 * index + 1], // Y 좌표 뒤집기
),
// ...
```

프로그램을 다시 실행하면 이제 올바른 결과를 볼 수 있을 것입니다:

![](/images/drawing_model.png)

모든 노력이 마침내 이런 데모로 결실을 맺기 시작했습니다!

> 모델이 회전할 때 뒷면(벽의 뒷부분)이 다소 이상하게 보일 수 있습니다. 이는 정상이며, 모델이 원래 그쪽에서 보도록 설계되지 않았기 때문입니다.

## 정점 중복 제거

불행히도 아직 인덱스 버퍼를 제대로 활용하고 있지 않습니다. `vertices` `Vec`에는 많은 중복된 정점 데이터가 포함되어 있는데, 이는 많은 정점이 여러 삼각형에 포함되기 때문입니다. 고유한 정점만 유지하고, 이들이 나타날 때마다 인덱스 버퍼를 사용해 재사용해야 합니다. 이를 구현하는 간단한 방법은 `HashMap`을 사용하여 고유한 정점과 각각의 인덱스를 추적하는 것입니다:

```rust
// load_model 메서드를 수정
use std::collections::HashMap;

// ...
fn load_model(&mut self) {
    // ... tobj::load_obj 호출
    
    let mut unique_vertices = HashMap::new();
    let model = &models[0];
    let mesh = &model.mesh;

    for index in &mesh.indices {
        let index = *index as usize;
        
        let pos = glam::vec3(
            mesh.positions[3 * index],
            mesh.positions[3 * index + 1],
            mesh.positions[3 * index + 2],
        );
        let tex_coord = glam::vec2(
            mesh.texcoords[2 * index],
            1.0 - mesh.texcoords[2 * index + 1],
        );

        let vertex = Vertex {
            pos,
            tex_coord,
            color: glam::Vec3::ONE,
        };

        if let Some(index) = unique_vertices.get(&vertex) {
            self.indices.push(*index);
        } else {
            let index = self.vertices.len() as u32;
            unique_vertices.insert(vertex, index);
            self.vertices.push(vertex);
            self.indices.push(index);
        }
    }
}
```
*Note: `load_model`의 기존 로직을 이 코드로 완전히 교체하세요.*

OBJ 파일에서 정점을 구성할 때마다, `HashMap`을 사용하여 정확히 동일한 위치와 텍스처 좌표를 가진 정점을 이전에 본 적이 있는지 확인합니다. 만약 본 적이 없다면, `vertices` `Vec`에 추가하고 그 인덱스를 `unique_vertices` 맵에 저장합니다. 그 후 새 정점의 인덱스를 `indices` `Vec`에 추가합니다. 만약 이전에 정확히 동일한 정점을 본 적이 있다면, 맵에서 그 인덱스를 찾아 `indices`에 저장합니다.

`Vertex` 구조체를 `HashMap`의 키로 사용하려면, `Eq` (그리고 `PartialEq`)와 `Hash` 트레이트를 구현해야 합니다. Rust에서는 `f32`가 `NaN` 값 때문에 기본적으로 `Eq`와 `Hash`를 구현하지 않지만, `glam` 크레이트의 `hash` 기능을 활성화하면 이 문제를 해결해 줍니다. `derive` 매크로를 사용하여 이 트레이트들을 쉽게 구현할 수 있습니다. `Vertex` 구조체 정의를 다음과 같이 수정하세요:

```rust
use glam::{Vec2, Vec3};

#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash)]
#[repr(C)]
struct Vertex {
    pos: Vec3,
    color: Vec3,
    tex_coord: Vec2,
}
```
*Note: `glam`의 `Vec3`와 `Vec2`는 `Eq`와 `Hash`를 구현하려면 `hash` 기능이 활성화되어 있어야 합니다. 우리는 이미 `Cargo.toml`에 이를 추가했습니다.*

이제 프로그램을 성공적으로 컴파일하고 실행할 수 있을 것입니다. `vertices`의 길이를 확인해보면, 이 모델의 경우 1,500,000개에 육박하던 것이 265,645개로 줄어든 것을 볼 수 있습니다! 이는 각 정점이 평균적으로 약 6개의 삼각형에서 재사용된다는 것을 의미합니다. 이로써 확실히 많은 GPU 메모리를 절약할 수 있습니다.

[Rust 코드](/code/28_model_loading.rs) /
[정점 셰이더](/code/27_shader_depth.vert) /
[프래그먼트 셰이더](/code/27_shader_depth.frag)