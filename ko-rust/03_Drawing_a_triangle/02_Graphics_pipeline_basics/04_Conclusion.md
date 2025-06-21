이제 이전 챕터들에서 다룬 모든 구조체와 객체를 조합하여 그래픽스 파이프라인을 만들 수 있습니다! 우리가 지금까지 다룬 객체 유형을 간단히 요약하면 다음과 같습니다:

*   **셰이더 스테이지(Shader stages)**: 그래픽스 파이프라인의 프로그래밍 가능한 스테이지의 기능을 정의하는 셰이더 모듈
*   **고정 함수 상태(Fixed-function state)**: 입력 어셈블리, 래스터라이저, 뷰포트, 색상 혼합과 같이 파이프라인의 고정 함수 스테이지를 정의하는 모든 구조체
*   **파이프라인 레이아웃(Pipeline layout)**: 셰이더에서 참조하며 드로우 타임에 업데이트할 수 있는 유니폼 및 푸시 값
*   **렌더 패스(Render pass)**: 파이프라인 스테이지에서 참조하는 어태치먼트와 그 사용법

이 모든 것을 합치면 그래픽스 파이프라인의 기능이 완벽하게 정의됩니다. 따라서 이제 `create_graphics_pipeline` 함수의 끝부분에서 `vk::GraphicsPipelineCreateInfo` 구조체를 채워 넣기 시작할 수 있습니다. 단, 셰이더 모듈은 파이프라인 생성 중에 여전히 사용되므로 셰이더 모듈을 파괴하는 코드보다는 앞에 위치해야 합니다.

`ash`에서는 C++처럼 구조체의 각 필드를 수동으로 할당하는 대신, 타입-세이프한 빌더(builder) 패턴을 사용하는 것이 일반적입니다.

```rust
let pipeline_info = vk::GraphicsPipelineCreateInfo::builder()
    .stages(shader_stages)
```

먼저 `.stages()` 메서드를 사용하여 `vk::PipelineShaderStageCreateInfo` 구조체 슬라이스(`&[..]`)를 참조하는 것으로 시작합니다.

```rust
    .vertex_input_state(&vertex_input_info)
    .input_assembly_state(&input_assembly)
    .viewport_state(&viewport_state)
    .rasterization_state(&rasterizer)
    .multisample_state(&multisampling)
    // .depth_stencil_state(&depth_stencil_info) // 선택 사항
    .color_blend_state(&color_blending)
    .dynamic_state(&dynamic_state);
```

그다음, 고정 함수 스테이지를 설명하는 모든 구조체를 참조합니다. `depth_stencil_state`와 같은 선택적 필드는 빌더에서 해당 메서드를 호출하지 않으면 자동으로 null 포인터로 설정되므로 코드가 더 간결해집니다.

```rust
    .layout(pipeline_layout)
```

그다음은 파이프라인 레이아웃인데, 이것은 구조체 포인터가 아닌 Vulkan 핸들입니다.

```rust
    .render_pass(render_pass)
    .subpass(0)
```

마지막으로 렌더 패스와, 이 그래픽스 파이프라인이 사용될 서브패스의 인덱스에 대한 참조가 있습니다. 이 파이프라인을 이 특정 인스턴스 대신 다른 렌더 패스와 함께 사용하는 것도 가능하지만, 그 렌더 패스들은 `render_pass`와 *호환 가능(compatible)*해야 합니다. 호환성 요구 사항은 [여기](https://www.khronos.org/registry/vulkan/specs/1.3-extensions/html/chap8.html#renderpass-compatibility)에 설명되어 있지만, 이 튜토리얼에서는 해당 기능을 사용하지 않을 것입니다.

```rust
    .base_pipeline_handle(vk::Pipeline::null()) // 선택 사항
    .base_pipeline_index(-1); // 선택 사항
```

실제로는 두 개의 필드가 더 있습니다: `base_pipeline_handle`과 `base_pipeline_index`. Vulkan에서는 기존 파이프라인에서 파생하여 새로운 그래픽스 파이프라인을 생성할 수 있습니다. 파이프라인 파생(derivatives)의 개념은, 기존 파이프라인과 많은 기능이 공통될 때 파이프라인을 설정하는 비용이 저렴해지고, 동일한 부모에서 파생된 파이프라인 간의 전환도 더 빠르게 수행할 수 있다는 것입니다. `base_pipeline_handle`로 기존 파이프라인의 핸들을 지정하거나, `base_pipeline_index`로 지금 생성하려는 다른 파이프라인을 인덱스로 참조할 수 있습니다. 지금은 파이프라인이 하나뿐이므로, null 핸들(`vk::Pipeline::null()`)과 유효하지 않은 인덱스(-1)를 지정하겠습니다. 이 값들은 `vk::PipelineCreateFlags::DERIVATIVE` 플래그가 지정된 경우에만 사용됩니다. 빌더 패턴에서는 이들을 명시적으로 설정하지 않으면 기본값으로 설정됩니다.

이제 마지막 단계를 위해 `VkPipeline` 객체를 담을 구조체 필드를 준비합니다.

```rust
struct HelloTriangleApplication {
    // ...
    graphics_pipeline: vk::Pipeline,
    // ...
}
```

그리고 마침내 그래픽스 파이프라인을 생성합니다:

```rust
self.graphics_pipeline = unsafe {
    device
        .create_graphics_pipelines(vk::PipelineCache::null(), &[pipeline_info.build()], None)
        .expect("Failed to create graphics pipeline!")
}[0];
```

`ash`의 `create_graphics_pipelines` 함수는 Vulkan C API와 약간 다릅니다. 이 함수는 여러 개의 `vk::GraphicsPipelineCreateInfo` 객체를 받아 한 번의 호출로 여러 `vk::Pipeline` 객체를 생성하도록 설계되었습니다.

*   첫 번째 인자(`vk::PipelineCache::null()`)는 선택적인 파이프라인 캐시 객체를 참조합니다. 파이프라인 캐시는 여러 `create_graphics_pipelines` 호출에 걸쳐 파이프라인 생성과 관련된 데이터를 저장하고 재사용하는 데 사용될 수 있으며, 캐시를 파일에 저장하면 프로그램 실행 간에도 재사용할 수 있습니다. 이 내용은 파이프라인 캐시 챕터에서 다룰 것입니다.
*   두 번째 인자는 생성 정보 구조체의 슬라이스입니다. 우리는 하나만 생성하므로, `.build()`로 빌더를 완료한 후 슬라이스(`&[..]`)로 감싸줍니다.
*   세 번째 인자는 할당자 콜백으로, 여기서는 `None`을 사용합니다.
*   이 함수는 `Result<Vec<vk::Pipeline>, vk::Result>`를 반환합니다. `expect()`를 사용해 에러를 처리하고, 우리는 파이프라인을 하나만 생성했으므로 반환된 `Vec`의 첫 번째 요소(`[0]`)를 가져옵니다. 이 작업은 `unsafe` 블록 안에서 수행해야 합니다.

그래픽스 파이프라인은 모든 일반적인 드로잉 작업에 필요하므로 프로그램이 끝날 때 파괴되어야 합니다. Rust에서는 보통 `Drop` 트레잇 내에서 리소스 해제를 처리하지만, 이 튜토리얼의 구조를 따라 `cleanup` 함수에 추가하겠습니다.

```rust
unsafe fn cleanup(&mut self) {
    self.device.destroy_pipeline(self.graphics_pipeline, None);
    self.device.destroy_pipeline_layout(self.pipeline_layout, None);
    // ...
}
```

이제 프로그램을 실행하여 이 모든 노력이 성공적인 파이프라인 생성으로 이어졌는지 확인하세요! 이제 화면에 무언가 나타나는 것에 꽤 가까워졌습니다. 다음 몇 개의 챕터에서는 스왑 체인 이미지로부터 실제 프레임버퍼를 설정하고 드로잉 커맨드를 준비할 것입니다.