## 서론

이제 우리는 각 정점마다 임의의 어트리뷰트를 버텍스 셰이더로 전달할 수 있게 되었습니다. 하지만 전역 변수는 어떨까요? 이번 장부터는 3D 그래픽스로 넘어가게 되는데, 여기에는 모델-뷰-프로젝션(MVP) 행렬이 필요합니다. 이 행렬을 정점 데이터에 포함시킬 수도 있겠지만, 이는 메모리 낭비이며 변환이 변경될 때마다 정점 버퍼를 업데이트해야 합니다. 변환은 매 프레임마다 쉽게 바뀔 수 있습니다.

Vulkan에서 이 문제를 해결하는 올바른 방법은 *리소스 디스크립터(resource descriptor)*를 사용하는 것입니다. 디스크립터는 셰이더가 버퍼나 이미지 같은 리소스에 자유롭게 접근할 수 있게 해주는 방법입니다. 우리는 변환 행렬들을 담고 있는 버퍼를 설정하고, 버텍스 셰이더가 디스크립터를 통해 이들에 접근하도록 할 것입니다. 디스크립터 사용은 세 부분으로 구성됩니다:

*   파이프라인 생성 시 디스크립터 셋 레이아웃 명시
*   디스크립터 풀에서 디스크립터 셋 할당
*   렌더링 시 디스크립터 셋 바인딩

*디스크립터 셋 레이아웃*은 렌더 패스가 접근할 어태치먼트의 타입을 명시하는 것과 유사하게, 파이프라인이 접근할 리소스의 타입을 명시합니다. *디스크립터 셋*은 프레임버퍼가 렌더 패스 어태치먼트에 바인딩할 실제 이미지 뷰를 지정하는 것과 유사하게, 디스크립터에 바인딩될 실제 버퍼나 이미지 리소스를 지정합니다. 그 후 디스크립터 셋은 정점 버퍼나 프레임버퍼처럼 드로우 커맨드를 위해 바인딩됩니다.

많은 종류의 디스크립터가 있지만, 이번 장에서는 uniform buffer object (UBO)를 다룰 것입니다. 다른 종류의 디스크립터는 다음 장들에서 살펴보겠지만, 기본적인 과정은 동일합니다. 버텍스 셰이더가 사용하길 원하는 데이터를 다음과 같은 Rust 구조체에 담는다고 가정해 봅시다:

```rust
#[repr(C)]
#[derive(Copy, Clone, Debug)]
struct UniformBufferObject {
    model: Matrix4<f32>,
    view: Matrix4<f32>,
    proj: Matrix4<f32>,
}
```

그러면 우리는 이 데이터를 `vk::Buffer`로 복사하고, 버텍스 셰이더에서 uniform buffer object 디스크립터를 통해 다음과 같이 접근할 수 있습니다:

```glsl
layout(binding = 0) uniform UniformBufferObject {
    mat4 model;
    mat4 view;
    mat4 proj;
} ubo;

void main() {
    gl_Position = ubo.proj * ubo.view * ubo.model * vec4(inPosition, 0.0, 1.0);
    fragColor = inColor;
}
```

우리는 이전 장의 사각형을 3D 공간에서 회전시키기 위해 매 프레임마다 모델, 뷰, 프로젝션 행렬을 업데이트할 것입니다.

## 버텍스 셰이더

위에서 명시된 것처럼 uniform buffer object를 포함하도록 버텍스 셰이더를 수정하세요. MVP 변환에 대해서는 이미 익숙하다고 가정하겠습니다. 만약 익숙하지 않다면, 첫 장에서 언급된 [참고 자료](https://www.opengl-tutorial.org/beginners-tutorials/tutorial-3-matrices/)를 확인하세요.

```glsl
#version 450

layout(binding = 0) uniform UniformBufferObject {
    mat4 model;
    mat4 view;
    mat4 proj;
} ubo;

layout(location = 0) in vec2 inPosition;
layout(location = 1) in vec3 inColor;

layout(location = 0) out vec3 fragColor;

void main() {
    gl_Position = ubo.proj * ubo.view * ubo.model * vec4(inPosition, 0.0, 1.0);
    fragColor = inColor;
}
```

`uniform`, `in`, `out` 선언 순서는 중요하지 않습니다. `binding` 지시어는 어트리뷰트의 `location` 지시어와 유사합니다. 우리는 디스크립터 셋 레이아웃에서 이 바인딩을 참조할 것입니다. `gl_Position`을 계산하는 줄은 변환 행렬들을 사용하여 최종 클립 좌표(clip coordinates)를 계산하도록 변경되었습니다. 2D 삼각형과 달리, 클립 좌표의 마지막 성분은 `1`이 아닐 수 있으며, 이는 화면의 최종 정규화된 장치 좌표(normalized device coordinates)로 변환될 때 나눗셈을 유발합니다. 이것은 원근 투영에서 *원근 분할(perspective division)*로 사용되며, 가까운 물체가 멀리 있는 물체보다 더 크게 보이게 하는 데 필수적입니다.

## 디스크립터 셋 레이아웃

다음 단계는 Rust 측에서 UBO를 정의하고, 버텍스 셰이더의 이 디스크립터에 대해 Vulkan에 알려주는 것입니다. 행렬 연산을 위해 `cgmath` crate를 사용합니다.

```rust
use cgmath::{Matrix4, Point3, Vector3};

#[repr(C)]
#[derive(Copy, Clone, Debug)]
struct UniformBufferObject {
    model: Matrix4<f32>,
    view: Matrix4<f32>,
    proj: Matrix4<f32>,
}
```

`#[repr(C)]` 어트리뷰트는 Rust 컴파일러에게 C와 같은 메모리 레이아웃을 사용하도록 지시하여 셰이더의 기대와 일치하게 만듭니다. `cgmath`의 `Matrix4` 타입은 셰이더의 `mat4`와 바이너리 호환됩니다.

모든 디스크립터 바인딩에 대한 세부 정보를 파이프라인 생성 시 제공해야 합니다. 이 정보를 정의하기 위해 `create_descriptor_set_layout` 함수를 설정할 것입니다. 이 함수는 파이프라인 레이아웃이 필요하므로 `init_vulkan` 내에서 파이프라인 생성 직전에 호출되어야 합니다.

```rust
impl HelloTriangleApplication {
    fn init_vulkan(&mut self) {
        // ...
        self.create_descriptor_set_layout();
        self.create_graphics_pipeline();
        // ...
    }

    // ...

    fn create_descriptor_set_layout(&mut self) {
        // ...
    }
}
```

먼저, `HelloTriangleApplication` 구조체에 `descriptor_set_layout` 멤버를 추가합니다.

```rust
struct HelloTriangleApplication {
    // ...
    pipeline_layout: vk::PipelineLayout,
    descriptor_set_layout: vk::DescriptorSetLayout,
    // ...
}
```

이제 `create_descriptor_set_layout` 함수를 구현합니다. 각 바인딩은 `vk::DescriptorSetLayoutBinding` 구조체로 기술됩니다. `ash`의 빌더 패턴을 사용하면 코드를 더 명확하게 만들 수 있습니다.

```rust
fn create_descriptor_set_layout(&mut self) {
    let ubo_layout_binding = vk::DescriptorSetLayoutBinding::builder()
        .binding(0)
        .descriptor_type(vk::DescriptorType::UNIFORM_BUFFER)
        .descriptor_count(1)
        .stage_flags(vk::ShaderStageFlags::VERTEX)
        .build();

    let bindings = [ubo_layout_binding];
    let layout_info = vk::DescriptorSetLayoutCreateInfo::builder()
        .bindings(&bindings);

    self.descriptor_set_layout = unsafe {
        self.device
            .create_descriptor_set_layout(&layout_info, None)
    }
    .expect("Failed to create descriptor set layout!");
}
```

`binding(0)`은 셰이더의 `layout(binding = 0)`에 해당합니다. `descriptor_type`은 uniform 버퍼임을 명시합니다. `descriptor_count`는 배열의 크기이며, 우리는 단일 UBO만 사용하므로 1입니다. `stage_flags`는 이 디스크립터가 버텍스 셰이더에서 사용됨을 나타냅니다.

그런 다음, 파이프라인 레이아웃을 생성할 때 이 디스크립터 셋 레이아웃을 참조하도록 `create_pipeline_layout` (혹은 `create_graphics_pipeline` 내의 관련 부분)을 수정해야 합니다.

```rust
// In create_graphics_pipeline or a create_pipeline_layout helper
let set_layouts = [self.descriptor_set_layout];
let pipeline_layout_info = vk::PipelineLayoutCreateInfo::builder()
    .set_layouts(&set_layouts);
// ...
self.pipeline_layout = unsafe {
    self.device.create_pipeline_layout(&pipeline_layout_info, None)
}.expect("Failed to create pipeline layout!");
```

`set_layouts` 슬라이스를 통해 하나 이상의 디스크립터 셋 레이아웃을 파이프라인에 바인딩할 수 있습니다.

마지막으로, 애플리케이션 종료 시 리소스를 정리합니다.

```rust
impl Drop for HelloTriangleApplication {
    fn drop(&mut self) {
        unsafe {
            // ...
            self.device.destroy_descriptor_set_layout(self.descriptor_set_layout, None);
            // ...
        }
    }
}
```

## Uniform 버퍼

다음으로, 셰이더를 위한 UBO 데이터를 담을 버퍼를 생성해야 합니다. 매 프레임 UBO를 업데이트할 것이므로 스테이징 버퍼는 불필요한 오버헤드를 유발할 수 있습니다.

여러 프레임이 동시에 처리 중(in-flight)일 수 있으므로, 이전 프레임이 사용 중인 버퍼를 덮어쓰는 것을 방지하기 위해 각 프레임마다 별도의 uniform 버퍼를 가져야 합니다. `HelloTriangleApplication` 구조체에 새로운 멤버들을 추가합니다.

```rust
struct HelloTriangleApplication {
    // ...
    index_buffer: vk::Buffer,
    index_buffer_memory: vk::DeviceMemory,

    uniform_buffers: Vec<vk::Buffer>,
    uniform_buffers_memory: Vec<vk::DeviceMemory>,
    uniform_buffers_mapped: Vec<*mut std::ffi::c_void>,
    // ...
}
```

`create_uniform_buffers` 함수를 만들어 `create_index_buffer` 다음에 호출되도록 합니다.

```rust
impl HelloTriangleApplication {
    fn init_vulkan(&mut self) {
        // ...
        self.create_vertex_buffer();
        self.create_index_buffer();
        self.create_uniform_buffers();
        // ...
    }
    
    // ...

    fn create_uniform_buffers(&mut self) {
        let buffer_size = std::mem::size_of::<UniformBufferObject>() as vk::DeviceSize;

        self.uniform_buffers.resize(MAX_FRAMES_IN_FLIGHT, vk::Buffer::null());
        self.uniform_buffers_memory.resize(MAX_FRAMES_IN_FLIGHT, vk::DeviceMemory::null());
        self.uniform_buffers_mapped.resize(MAX_FRAMES_IN_FLIGHT, std::ptr::null_mut());

        for i in 0..MAX_FRAMES_IN_FLIGHT {
            let (buffer, memory) = self.create_buffer(
                buffer_size,
                vk::BufferUsageFlags::UNIFORM_BUFFER,
                vk::MemoryPropertyFlags::HOST_VISIBLE | vk::MemoryPropertyFlags::HOST_COHERENT,
            );
            self.uniform_buffers[i] = buffer;
            self.uniform_buffers_memory[i] = memory;

            self.uniform_buffers_mapped[i] = unsafe {
                self.device.map_memory(
                    self.uniform_buffers_memory[i],
                    0,
                    buffer_size,
                    vk::MemoryMapFlags::empty(),
                )
            }
            .expect("Failed to map uniform buffer memory!");
        }
    }
}
```
버퍼를 생성한 직후 `map_memory`를 호출하여 CPU에서 접근 가능한 포인터를 얻습니다. 이 포인터는 애플리케이션 수명 동안 유효하며, 이를 **"영구적 매핑(persistent mapping)"**이라고 합니다. 매번 데이터를 업데이트할 때마다 매핑/언매핑을 반복하지 않아도 되므로 성능에 이점이 있습니다.

`drop` 함수에서 이 버퍼들과 메모리를 해제합니다.

```rust
impl Drop for HelloTriangleApplication {
    fn drop(&mut self) {
        unsafe {
            // ...
            for i in 0..MAX_FRAMES_IN_FLIGHT {
                self.device.destroy_buffer(self.uniform_buffers[i], None);
                self.device.free_memory(self.uniform_buffers_memory[i], None);
            }

            self.device.destroy_descriptor_set_layout(self.descriptor_set_layout, None);
            // ...
        }
    }
}
```

## Uniform 데이터 업데이트

`draw_frame` 함수 내에서 커맨드 버퍼를 제출하기 전에 uniform 데이터를 업데이트하는 `update_uniform_buffer` 함수를 호출합니다.

```rust
impl HelloTriangleApplication {
    fn draw_frame(&mut self) {
        // ...
        self.update_uniform_buffer(self.current_frame);
        // ...
        // Submit command buffer
    }

    fn update_uniform_buffer(&mut self, current_image_index: usize) {
        // ...
    }
}
```

시간에 따라 회전하는 애니메이션을 구현하기 위해 `cgmath`와 `std::time`을 사용합니다. `Cargo.toml`에 `cgmath`와 `chrono` (또는 표준 라이브러리의 `std::time`)가 포함되어 있는지 확인하세요.

`HelloTriangleApplication`에 시간을 추적할 필드를 추가합니다.

```rust
struct HelloTriangleApplication {
    // ...
    start_time: std::time::Instant,
    // ...
}
```

`new` 함수에서 `start_time`을 초기화합니다.

```rust
impl HelloTriangleApplication {
    pub fn new(window: &winit::window::Window) -> Self {
        // ...
        let mut app = Self {
            // ...
            start_time: std::time::Instant::now(),
        };
        // ...
    }
}
```

이제 `update_uniform_buffer` 함수를 구현합니다.

```rust
use cgmath::{Deg, Matrix4, Point3, Rad, Vector3};

// ...

fn update_uniform_buffer(&mut self, current_image_index: usize) {
    let time = self.start_time.elapsed().as_secs_f32();

    let model = Matrix4::from_axis_angle(
        Vector3::new(0.0, 0.0, 1.0),
        Deg(90.0 * time),
    );

    let view = Matrix4::look_at_rh(
        Point3::new(2.0, 2.0, 2.0),
        Point3::new(0.0, 0.0, 0.0),
        Vector3::new(0.0, 0.0, 1.0),
    );

    let mut proj = cgmath::perspective(
        Deg(45.0),
        self.swapchain_extent.width as f32 / self.swapchain_extent.height as f32,
        0.1,
        10.0,
    );

    // cgmath는 OpenGL의 클립 좌표계를 기준으로 설계되었습니다.
    // Vulkan은 Y좌표가 반대이므로, 프로젝션 행렬의 Y 스케일링 요소의 부호를 뒤집어줍니다.
    proj[1][1] *= -1.0;

    let ubo = UniformBufferObject { model, view, proj };

    // 매핑된 메모리에 데이터 복사
    unsafe {
        let data_ptr = self.uniform_buffers_mapped[current_image_index];
        std::ptr::copy_nonoverlapping(&ubo, data_ptr as *mut UniformBufferObject, 1);
    }
}
```
**설명:**
1.  `time`: 애플리케이션 시작 후 경과 시간을 초 단위로 계산합니다.
2.  `model`: Z축을 기준으로 초당 90도 회전하는 변환 행렬을 생성합니다.
3.  `view`: 45도 각도 위에서 (2, 2, 2) 위치에서 원점을 바라보는 뷰 행렬을 생성합니다. Vulkan은 오른손 좌표계를 사용하므로 `look_at_rh`를 사용합니다.
4.  `proj`: 45도 시야각을 가진 원근 투영 행렬을 생성합니다. 창 크기 변경에 대응하기 위해 현재 스왑체인의 종횡비를 사용합니다.
5.  `proj[1][1] *= -1.0`: GLM/cgmath는 OpenGL의 클립 공간(Y축이 아래로)을 기준으로 하므로, Vulkan의 클립 공간(Y축이 위로)에 맞추기 위해 Y축을 뒤집어 줍니다.
6.  `std::ptr::copy_nonoverlapping`: 계산된 UBO 구조체 데이터를 이전에 매핑해 둔 버퍼 메모리 위치로 복사합니다. 이는 `unsafe` 블록 내에서 수행되어야 합니다.

자주 변경되는 작은 데이터를 셰이더에 전달하는 데 UBO를 사용하는 것이 항상 가장 효율적인 방법은 아닙니다. 더 효율적인 방법으로는 *푸시 상수(push constants)*가 있으며, 이는 추후 튜토리얼에서 다룰 수 있습니다.

다음 장에서는 디스크립터 셋에 대해 알아보고, 우리가 생성한 `vk::Buffer`를 uniform 버퍼 디스크립터에 실제로 바인딩하여 셰이더가 이 변환 데이터에 접근할 수 있도록 하는 방법을 배울 것입니다.