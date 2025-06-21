이전 API들과 달리, Vulkan의 셰이더 코드는 [GLSL](https://en.wikipedia.org/wiki/OpenGL_Shading_Language)이나 [HLSL](https://en.wikipedia.org/wiki/High-Level_Shading_Language)처럼 사람이 읽을 수 있는 문법이 아닌 바이트코드 형식으로 명시되어야 합니다. 이 바이트코드 형식을 [SPIR-V](https://www.khronos.org/spir)라고 부르며, Vulkan과 OpenCL(둘 다 Khronos API) 모두에서 사용하도록 설계되었습니다. SPIR-V는 그래픽 및 컴퓨트 셰이더를 작성하는 데 사용될 수 있지만, 이 튜토리얼에서는 Vulkan의 그래픽 파이프라인에서 사용되는 셰이더에 초점을 맞출 것입니다.

바이트코드 형식을 사용하는 것의 장점은, GPU 제조사가 셰이더 코드를 네이티브 코드로 변환하기 위해 작성하는 컴파일러가 훨씬 덜 복잡해진다는 점입니다. 과거 GLSL과 같이 사람이 읽을 수 있는 문법의 경우, 일부 GPU 제조사는 표준을 다소 유연하게 해석하는 경향이 있었습니다. 만약 여러분이 이런 제조사 중 하나의 GPU로 복잡한 셰이더를 작성했다면, 다른 제조사의 드라이버가 문법 오류로 코드를 거부하거나, 더 심하게는 컴파일러 버그로 셰이더가 다르게 동작할 위험이 있었습니다. SPIR-V와 같은 직관적인 바이트코드 형식을 사용하면 이러한 문제를 피할 수 있을 것입니다.

하지만 그렇다고 해서 우리가 이 바이트코드를 직접 손으로 작성해야 한다는 의미는 아닙니다. Khronos는 GLSL을 SPIR-V로 컴파일하는 자체적인 벤더 독립적 컴파일러를 출시했습니다. 이 컴파일러는 여러분의 셰이더 코드가 표준을 완벽하게 준수하는지 확인하고, 프로그램과 함께 배포할 수 있는 단일 SPIR-V 바이너리를 생성하도록 설계되었습니다. 우리는 `glslc` 컴파일러를 사용하여 GLSL 셰이더를 SPIR-V로 변환할 것입니다. `glslc`는 GCC나 Clang과 같은 잘 알려진 컴파일러와 동일한 파라미터 형식을 사용하고, *인클루드*와 같은 추가 기능을 포함합니다. 이 컴파일러는 Vulkan SDK에 이미 포함되어 있으므로 추가로 다운로드할 필요가 없습니다.

*(이후 GLSL에 대한 설명, Vertex/Fragment 셰이더 코드, 컴파일 방법은 원문과 동일하므로 생략하고 Rust 코드 구현 부분부터 시작하겠습니다. 아래의 GLSL 코드를 각각 `shaders/shader.vert`와 `shaders/shader.frag` 파일로 저장하고 컴파일해 `vert.spv`와 `frag.spv` 파일을 준비했다고 가정합니다.)*

**Vertex Shader (`shader.vert`)**
```glsl
#version 450

layout(location = 0) out vec3 fragColor;

vec2 positions[3] = vec2[](
    vec2(0.0, -0.5),
    vec2(0.5, 0.5),
    vec2(-0.5, 0.5)
);

vec3 colors[3] = vec3[](
    vec3(1.0, 0.0, 0.0),
    vec3(0.0, 1.0, 0.0),
    vec3(0.0, 0.0, 1.0)
);

void main() {
    gl_Position = vec4(positions[gl_VertexIndex], 0.0, 1.0);
    fragColor = colors[gl_VertexIndex];
}
```

**Fragment Shader (`shader.frag`)**
```glsl
#version 450

layout(location = 0) in vec3 fragColor;

layout(location = 0) out vec4 outColor;

void main() {
    outColor = vec4(fragColor, 1.0);
}
```

## 셰이더 로드하기

이제 SPIR-V 셰이더를 생성했으니, 프로그램에 로드하여 그래픽 파이프라인에 연결할 시간입니다. Rust의 표준 라이브러리를 사용하면 파일에서 바이너리 데이터를 매우 쉽게 읽을 수 있습니다.

`create_graphics_pipeline` 함수에서 두 셰이더의 바이트코드를 로드합니다.

```rust
// In `VulkanApp::create_graphics_pipeline`
fn create_graphics_pipeline(&mut self) {
    let vert_shader_code = std::fs::read("shaders/vert.spv")
        .expect("Failed to read vertex shader file!");
    let frag_shader_code = std::fs::read("shaders/frag.spv")
        .expect("Failed to read fragment shader file!");
    // ...
}
```

`std::fs::read` 함수는 파일의 모든 바이트를 읽어 `Vec<u8>` (바이트 벡터)로 반환합니다. C++ 예제처럼 수동으로 파일 크기를 계산하고 버퍼를 할당할 필요가 없습니다. 파일 읽기에 실패하면 `expect`가 프로그램을 패닉시키므로, 실제 애플리케이션에서는 `Result`를 적절히 처리해야 합니다.

## 셰이더 모듈 생성하기

코드를 파이프라인에 전달하기 전에 `ash::vk::ShaderModule` 객체로 감싸야 합니다. 이를 위해 `create_shader_module` 헬퍼 함수를 만들어 보겠습니다.

```rust
// In `impl VulkanApp`
fn create_shader_module(&self, code: &[u8]) -> vk::ShaderModule {
    // ...
}
```

이 함수는 바이트코드가 담긴 슬라이스를 파라미터로 받아 `vk::ShaderModule`을 생성합니다.

셰이더 모듈을 만드는 것은 간단합니다. 바이트코드와 그 길이를 지정하면 됩니다. C++ 버전에서는 `char` 포인터를 `uint32_t` 포인터로 `reinterpret_cast`해야 했고, 이 과정에서 데이터 정렬(alignment) 문제가 발생할 수 있었습니다. SPIR-V는 `u32`의 슬라이스를 기대하기 때문입니다.

Rust에서는 `ash`가 제공하는 유틸리티 함수를 사용해 이 과정을 안전하고 간단하게 처리할 수 있습니다. `ash::util::read_spv`는 바이트 슬라이스(`&[u8]`)를 받아 `Vec<u32>`로 변환해주므로 정렬 문제를 걱정할 필요가 없습니다.

```rust
use ash::{util, vk};
use std::io::Cursor;

// ...

fn create_shader_module(&self, code: &[u8]) -> vk::ShaderModule {
    let mut cursor = Cursor::new(code);
    let code = util::read_spv(&mut cursor).expect("Failed to read SPV-V shader code");
    
    let create_info = vk::ShaderModuleCreateInfo::builder()
        .code(&code);

    unsafe {
        self.device
            .create_shader_module(&create_info, None)
            .expect("Failed to create shader module!")
    }
}
```

이제 `create_shader_module` 함수를 `create_graphics_pipeline` 내에서 호출합니다.

```rust
// In `VulkanApp::create_graphics_pipeline`
fn create_graphics_pipeline(&mut self) {
    let vert_shader_code = std::fs::read("shaders/vert.spv")
        .expect("Failed to read vertex shader file!");
    let frag_shader_code = std::fs::read("shaders/frag.spv")
        .expect("Failed to read fragment shader file!");

    let vert_shader_module = self.create_shader_module(&vert_shader_code);
    let frag_shader_module = self.create_shader_module(&frag_shader_code);
    
    // ...
}
```

셰이더 모듈은 우리가 이전에 파일에서 로드한 셰이더 바이트코드와 그 안에 정의된 함수들을 얇게 감싼 래퍼일 뿐입니다. SPIR-V 바이트코드를 GPU가 실행할 수 있는 기계어 코드로 컴파일하고 링크하는 작업은 그래픽 파이프라인이 생성될 때까지 일어나지 않습니다.

이는 파이프라인 생성이 완료되는 즉시 셰이더 모듈을 파괴해도 된다는 의미입니다. Rust의 소유권(ownership) 모델 덕분에 이 과정은 자동으로 처리됩니다. `vert_shader_module`과 `frag_shader_module` 변수는 `create_graphics_pipeline` 함수가 끝날 때 범위를 벗어나고, 이때 자동으로 `vkDestroyShaderModule`이 호출됩니다. C++ 예제처럼 수동으로 `vkDestroyShaderModule`을 호출할 필요가 없어 코드가 더 깔끔하고 안전합니다.

## 셰이더 스테이지 생성

셰이더를 실제로 사용하려면 실제 파이프라인 생성 과정의 일부로서 `vk::PipelineShaderStageCreateInfo` 구조체를 통해 특정 파이프라인 단계에 할당해야 합니다.

`create_graphics_pipeline` 함수에서 버텍스 셰이더를 위한 구조체를 채우는 것으로 시작하겠습니다. `ash`의 빌더 패턴을 사용하면 코드가 더 읽기 쉬워집니다.

```rust
// ... inside create_graphics_pipeline, after creating shader modules

// Vulkan은 C 문자열을 기대하므로 CString으로 변환해야 합니다.
let main_function_name = std::ffi::CString::new("main").unwrap();

let vert_shader_stage_info = vk::PipelineShaderStageCreateInfo::builder()
    .stage(vk::ShaderStageFlags::VERTEX)
    .module(vert_shader_module)
    .name(&main_function_name);
```

첫 번째 단계는 셰이더가 사용될 파이프라인 단계(`stage`)를 알려주는 것입니다. 여기서는 `VERTEX` 단계를 지정합니다. 그 다음 코드를 포함하는 셰이더 모듈(`module`)과 호출할 함수, 즉 *엔트리포인트(entrypoint)*의 이름(`name`)을 지정합니다. 여기서는 표준적인 `main`을 사용합니다.

`pSpecializationInfo`라는 선택적 멤버도 있지만, 여기서는 사용하지 않으므로 빌더에서 설정하지 않으면 기본값인 `null`로 처리됩니다.

프래그먼트 셰이더에 맞게 구조체를 수정하는 것은 쉽습니다.

```rust
let frag_shader_stage_info = vk::PipelineShaderStageCreateInfo::builder()
    .stage(vk::ShaderStageFlags::FRAGMENT)
    .module(frag_shader_module)
    .name(&main_function_name);
```

마지막으로 이 두 구조체를 포함하는 슬라이스를 정의합니다. 이 슬라이스는 나중에 실제 파이프라인 생성 단계에서 이들을 참조하는 데 사용될 것입니다.

```rust
let shader_stages = [vert_shader_stage_info.build(), frag_shader_stage_info.build()];
```
`.build()`를 호출하여 빌더를 최종 구조체로 변환하는 것을 잊지 마세요.

이제 파이프라인의 프로그래밍 가능 단계를 모두 기술했습니다. 함수 마지막에 셰이더 모듈을 수동으로 정리하는 코드를 추가합니다.

```rust
// ... at the end of create_graphics_pipeline
    // 셰이더 모듈은 더 이상 필요 없으므로 파이프라인 생성 후 즉시 파괴합니다.
    unsafe {
        self.device.destroy_shader_module(frag_shader_module, None);
        self.device.destroy_shader_module(vert_shader_module, None);
    }
}
```
*참고: 위에서는 RAII의 장점을 설명했지만, 튜토리얼의 흐름상 C++ 코드와 동일하게 명시적으로 파괴하는 코드를 추가했습니다. Rust에서는 변수가 범위를 벗어날 때 자동으로 리소스가 해제되도록 래퍼 타입을 만들어 관리하는 것이 일반적입니다.*

다음 장에서는 고정 함수 단계를 살펴보겠습니다.

[Rust 코드](/rust_code/09_shader_modules.rs) /
[버텍스 셰이더](/code/09_shader_base.vert) /
[프래그먼트 셰이더](/code/09_shader_base.frag)