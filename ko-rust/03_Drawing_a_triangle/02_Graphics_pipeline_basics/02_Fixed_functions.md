이전의 그래픽 API들은 그래픽 파이프라인의 대부분 단계에 대한 기본 상태를 제공했습니다. Vulkan에서는 대부분의 파이프라인 상태를 명시적으로 지정해야 하며, 이 상태들은 불변(immutable)의 파이프라인 상태 객체(PSO)로 구워지기(baked) 때문입니다. 이번 장에서는 이러한 고정 함수(fixed-function) 연산을 구성하기 위한 모든 구조체를 채워 넣을 것입니다. Rust와 `ash` 라이브러리를 사용하면 빌더 패턴을 통해 이 과정을 더 명확하게 진행할 수 있습니다.

## 동적 상태 (Dynamic state)

*대부분의* 파이프라인 상태는 파이프라인 상태 객체에 구워져야 하지만, 제한된 일부 상태는 파이프라인을 다시 만들지 않고도 드로우 타임(draw time)에 변경할 수 *있습니다*. 뷰포트의 크기, 선 두께, 블렌딩 상수 등이 그 예입니다. 만약 동적 상태를 사용하고 이러한 속성들을 파이프라인 생성 시에 고정하지 않으려면, `vk::PipelineDynamicStateCreateInfo` 구조체를 채워야 합니다. `ash`의 빌더를 사용하면 다음과 같이 작성할 수 있습니다.

```rust
let dynamic_states = [vk::DynamicState::VIEWPORT, vk::DynamicState::SCISSOR];

let dynamic_state_info = vk::PipelineDynamicStateCreateInfo::builder()
    .dynamic_states(&dynamic_states);
```

`ash`의 빌더는 슬라이스(`&[T]`)를 받아 내부적으로 개수(`dynamicStateCount`)와 포인터(`pDynamicStates`)를 자동으로 설정해줍니다. 이렇게 하면 이 값들의 구성이 파이프라인 생성 시에는 무시되며, 드로잉 시점에 이 데이터를 지정해야만 합니다. 이는 더 유연한 설정을 가능하게 하며, 뷰포트나 시저 상태처럼 파이프라인 상태에 고정시킬 경우 설정이 더 복잡해질 수 있는 항목들에 대해 매우 일반적인 방식입니다.

## 정점 입력 (Vertex input)

`vk::PipelineVertexInputStateCreateInfo` 구조체는 정점 셰이더로 전달될 정점 데이터의 형식을 설명합니다. 이 설명은 크게 두 가지 방식으로 이루어집니다.

*   **바인딩(Bindings)**: 데이터 간의 간격 및 데이터가 정점별(per-vertex)인지 인스턴스별(per-instance)인지 여부 (자세한 내용은 [인스턴싱](https://ko.wikipedia.org/wiki/인스턴싱) 참조)
*   **속성 서술(Attribute descriptions)**: 정점 셰이더로 전달되는 속성의 유형, 어떤 바인딩에서 로드할지, 그리고 어떤 오프셋에 있는지

지금은 정점 데이터를 정점 셰이더에 직접 하드코딩하고 있으므로, 이 구조체를 채워서 로드할 정점 데이터가 없음을 명시할 것입니다. `ash` 빌더의 기본값은 비어있는 상태이므로 코드가 매우 간결해집니다.

```rust
let vertex_input_info = vk::PipelineVertexInputStateCreateInfo::builder()
    .vertex_binding_descriptions(&[])
    .vertex_attribute_descriptions(&[]);
```

`vertex_binding_descriptions`와 `vertex_attribute_descriptions` 메서드에 빈 슬라이스(`&[]`)를 전달하면, `ash`가 자동으로 카운트를 0으로, 포인터는 널(null)로 설정합니다. 이 구조체는 나중에 정점 버퍼 장에서 다시 다룰 것입니다. `create_graphics_pipeline` 함수에서 셰이더 단계 정의 바로 다음에 이 코드를 추가하세요.

## 입력 조립 (Input assembly)

`vk::PipelineInputAssemblyStateCreateInfo` 구조체는 정점들로부터 어떤 종류의 지오메트리를 그릴 것인지와 프리미티브 재시작(primitive restart) 활성화 여부를 설명합니다.

사용 가능한 `topology`의 종류는 다음과 같습니다:

*   `vk::PrimitiveTopology::POINT_LIST`: 점
*   `vk::PrimitiveTopology::LINE_LIST`: 선 (재사용 없음)
*   `vk::PrimitiveTopology::LINE_STRIP`: 연결된 선
*   `vk::PrimitiveTopology::TRIANGLE_LIST`: 삼각형 (재사용 없음)
*   `vk::PrimitiveTopology::TRIANGLE_STRIP `: 연결된 삼각형

이 튜토리얼에서는 삼각형을 그릴 것이므로 다음과 같이 구조체를 설정합니다.

```rust
let input_assembly = vk::PipelineInputAssemblyStateCreateInfo::builder()
    .topology(vk::PrimitiveTopology::TRIANGLE_LIST)
    .primitive_restart_enable(false);
```

`primitive_restart_enable`을 `true`로 설정하면, `_STRIP` 토폴로지 모드에서 특수 인덱스(`0xFFFF` 또는 `0xFFFFFFFF`)를 사용하여 프리미티브를 끊을 수 있습니다.

## 뷰포트와 시저 (Viewports and scissors)

뷰포트(viewport)는 출력이 렌더링될 프레임버퍼의 영역을 설명합니다. 시저 사각형(scissor rectangle)은 픽셀이 실제로 저장될 영역을 정의하며, 이 영역 밖의 픽셀은 버려집니다.

```rust
let viewport = vk::Viewport {
    x: 0.0,
    y: 0.0,
    width: self.swapchain_extent.width as f32,
    height: self.swapchain_extent.height as f32,
    min_depth: 0.0,
    max_depth: 1.0,
};

let scissor = vk::Rect2D {
    offset: vk::Offset2D { x: 0, y: 0 },
    extent: self.swapchain_extent,
};
```

뷰포트와 시저는 정적 또는 동적 상태로 설정할 수 있습니다. 동적 상태를 사용하는 것이 일반적이며 성능 저하도 없습니다.

동적 상태를 사용할 경우, 앞서 정의한 `dynamic_state_info`가 이 역할을 합니다. 파이프라인 생성 시에는 뷰포트와 시저의 개수만 지정하면 됩니다.

```rust
let viewport_state = vk::PipelineViewportStateCreateInfo::builder()
    .viewport_count(1)
    .scissor_count(1);
```

이 경우 실제 `viewport`와 `scissor` 데이터는 나중에 커맨드 버퍼에 직접 기록해야 합니다.

만약 동적 상태를 사용하지 않고 파이프라인에 이 값들을 고정시키려면, 빌더에 실제 데이터를 전달해야 합니다.

```rust
// 동적 상태를 사용하지 않을 경우
let viewport_state = vk::PipelineViewportStateCreateInfo::builder()
    .viewports(&[viewport])
    .scissors(&[scissor]);
```
`ash` 빌더의 `.viewports()`와 `.scissors()` 메서드는 슬라이스를 인자로 받습니다. 우리는 동적 상태를 사용할 것이므로 위의 첫 번째 방법을 따릅니다.

## 래스터라이저 (Rasterizer)

래스터라이저는 정점 셰이더가 만든 지오메트리를 프래그먼트로 변환합니다. `vk::PipelineRasterizationStateCreateInfo`로 이 단계를 설정합니다.

```rust
let rasterizer = vk::PipelineRasterizationStateCreateInfo::builder()
    .depth_clamp_enable(false)
    .rasterizer_discard_enable(false)
    .polygon_mode(vk::PolygonMode::FILL)
    .line_width(1.0)
    .cull_mode(vk::CullModeFlags::BACK)
    .front_face(vk::FrontFace::CLOCKWISE)
    .depth_bias_enable(false)
    .depth_bias_constant_factor(0.0) // Optional
    .depth_bias_clamp(0.0)           // Optional
    .depth_bias_slope_factor(0.0);   // Optional
```
*   `depth_clamp_enable`: `true`이면 깊이 범위를 벗어난 프래그먼트를 버리는 대신 클램핑합니다 (GPU 기능 필요).
*   `rasterizer_discard_enable`: `true`이면 지오메트리가 래스터라이저를 통과하지 않아 프레임버퍼에 아무것도 출력되지 않습니다.
*   `polygon_mode`: `FILL`(채우기), `LINE`(선), `POINT`(점) 모드를 설정합니다 (`FILL` 외에는 GPU 기능 필요).
*   `line_width`: 선의 두께입니다 (`1.0f` 초과는 GPU 기능 필요).
*   `cull_mode`, `front_face`: 면 컬링(face culling) 방식과 앞면(front-face)으로 간주할 정점 순서(시계/반시계 방향)를 결정합니다.
*   `depth_bias...`: 섀도우 매핑 등에서 깊이 값에 편향을 줄 때 사용합니다.

## 멀티샘플링 (Multisampling)

`vk::PipelineMultisampleStateCreateInfo`는 안티 앨리어싱 기법 중 하나인 멀티샘플링을 설정합니다. 지금은 비활성화합니다 (GPU 기능 필요).

```rust
let multisampling = vk::PipelineMultisampleStateCreateInfo::builder()
    .sample_shading_enable(false)
    .rasterization_samples(vk::SampleCountFlags::TYPE_1)
    .min_sample_shading(1.0)                // Optional
    .sample_mask(&[])                        // Optional
    .alpha_to_coverage_enable(false)        // Optional
    .alpha_to_one_enable(false);            // Optional
```
이 부분은 나중 장에서 다시 다룰 것입니다. 지금은 샘플링을 1회만 수행하도록 설정합니다.

## 깊이 및 스텐실 테스팅 (Depth and stencil testing)

깊이/스텐실 버퍼를 사용한다면 `vk::PipelineDepthStencilStateCreateInfo`로 관련 테스트를 설정해야 합니다. 지금은 사용하지 않으므로, 최종 파이프라인 생성 정보에 이 부분은 널 포인터(`std::ptr::null()`)를 전달하여 비활성화할 것입니다.

## 색상 혼합 (Color blending)

프래그먼트 셰이더가 반환한 색상을 프레임버퍼의 기존 색상과 결합하는 단계입니다. 프레임버퍼별로 `vk::PipelineColorBlendAttachmentState`를, 전역적으로 `vk::PipelineColorBlendStateCreateInfo`를 설정합니다.

먼저, 단일 프레임버퍼에 대한 혼합 상태입니다. 혼합을 비활성화하면 프래그먼트 셰이더의 출력이 그대로 프레임버퍼에 쓰입니다.

```rust
let color_blend_attachment = vk::PipelineColorBlendAttachmentState::builder()
    .color_write_mask(vk::ColorComponentFlags::RGBA)
    .blend_enable(false)
    .src_color_blend_factor(vk::BlendFactor::ONE)   // Optional
    .dst_color_blend_factor(vk::BlendFactor::ZERO)  // Optional
    .color_blend_op(vk::BlendOp::ADD)               // Optional
    .src_alpha_blend_factor(vk::BlendFactor::ONE)   // Optional
    .dst_alpha_blend_factor(vk::BlendFactor::ZERO)  // Optional
    .alpha_blend_op(vk::BlendOp::ADD);              // Optional
```
`ash`에서는 `vk::ColorComponentFlags::R | G | B | A` 대신 `vk::ColorComponentFlags::RGBA`라는 편리한 상수를 제공합니다.

알파 블렌딩(반투명 효과)을 구현하려면 `blend_enable`을 `true`로 설정하고 관련 인자들을 다음과 같이 조정해야 합니다.
`finalColor.rgb = newAlpha * newColor + (1 - newAlpha) * oldColor;`
```rust
// 알파 블렌딩 예시
let color_blend_attachment_alpha = vk::PipelineColorBlendAttachmentState::builder()
    .color_write_mask(vk::ColorComponentFlags::RGBA)
    .blend_enable(true)
    .src_color_blend_factor(vk::BlendFactor::SRC_ALPHA)
    .dst_color_blend_factor(vk::BlendFactor::ONE_MINUS_SRC_ALPHA)
    .color_blend_op(vk::BlendOp::ADD)
    .src_alpha_blend_factor(vk::BlendFactor::ONE)
    .dst_alpha_blend_factor(vk::BlendFactor::ZERO)
    .alpha_blend_op(vk::BlendOp::ADD);
```

이제 전역 색상 혼합 상태를 설정합니다. 위에서 만든 첨부 상태(attachment state)를 참조합니다.

```rust
let color_blend_attachment_states = [color_blend_attachment.build()];
let color_blending = vk::PipelineColorBlendStateCreateInfo::builder()
    .logic_op_enable(false)
    .logic_op(vk::LogicOp::COPY) // Optional
    .attachments(&color_blend_attachment_states)
    .blend_constants([0.0, 0.0, 0.0, 0.0]); // Optional
```
`logic_op_enable`을 `true`로 설정하면 전통적인 혼합 대신 비트 연산을 사용하게 됩니다. 우리는 두 방식 모두 비활성화하여, 프래그먼트 색상이 수정 없이 프레임버퍼에 쓰이도록 합니다.

## 파이프라인 레이아웃 (Pipeline layout)

셰이더에서 사용하는 `uniform` 값(예: 변환 행렬, 텍스처 샘플러)을 파이프라인에 바인딩하기 위해 `vk::PipelineLayout` 객체가 필요합니다. 지금 당장 uniform을 사용하지 않더라도, 비어 있는 파이프라인 레이아웃을 반드시 생성해야 합니다.

`App` 구조체에 필드를 추가하여 이 객체를 저장합니다.
```rust
struct App {
    // ...
    pipeline_layout: vk::PipelineLayout,
    // ...
}
```

`create_graphics_pipeline` 함수 내에서 객체를 생성합니다.
```rust
let pipeline_layout_info = vk::PipelineLayoutCreateInfo::builder()
    .set_layouts(&[])
    .push_constant_ranges(&[]);

self.pipeline_layout = unsafe {
    self.device
        .create_pipeline_layout(&pipeline_layout_info, None)
        .expect("Failed to create pipeline layout!")
};
```
`ash`의 빌더는 빈 슬라이스를 처리하여 `setLayoutCount`와 `pushConstantRangeCount`를 0으로 설정합니다. `create_pipeline_layout`은 `unsafe` 함수이며 `Result`를 반환하므로, `expect`로 오류를 처리합니다.

파이프라인 레이아웃은 프로그램이 실행되는 동안 계속 사용되므로, 프로그램 종료 시 파괴해야 합니다. 이는 Rust의 `Drop` 트레이트를 구현하여 관리하는 것이 가장 이상적입니다.
```rust
impl Drop for App {
    fn drop(&mut self) {
        unsafe {
            self.device.destroy_pipeline_layout(self.pipeline_layout, None);
            // ... other cleanup ...
        }
    }
}
```

## 결론

이것으로 모든 고정 함수 상태 설정이 끝났습니다! 처음부터 모든 것을 설정하는 것은 많은 작업이지만, 그 장점은 이제 그래픽 파이프라인에서 일어나는 거의 모든 일을 완전히 인지하게 되었다는 것입니다. 이는 특정 컴포넌트의 기본 상태가 예상과 달라 예기치 않은 동작에 부딪힐 가능성을 줄여줍니다.

하지만 그래픽 파이프라인을 최종적으로 생성하기 전에 만들어야 할 객체가 하나 더 있으며, 그것은 바로 [렌더 패스](!en/Drawing_a_triangle/Graphics_pipeline_basics/Render_passes)입니다.