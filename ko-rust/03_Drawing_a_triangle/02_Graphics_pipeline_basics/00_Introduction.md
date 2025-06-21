앞으로 몇 개의 챕터에 걸쳐 첫 번째 삼각형을 그리기 위해 구성된 그래픽스 파이프라인을 설정할 것입니다. 그래픽스 파이프라인은 메시의 정점(vertices)과 텍스처(textures)를 가져와 렌더 타겟의 픽셀에 이르는 일련의 작업 순서입니다. 아래에 간소화된 개요가 표시되어 있습니다:

![](/images/vulkan_simplified_pipeline.svg)

*   **입력 조립기(Input Assembler)**는 지정한 버퍼에서 원시 정점 데이터를 수집하고, 인덱스 버퍼를 사용하여 정점 데이터 자체를 복제하지 않고도 특정 요소를 반복할 수도 있습니다.

*   **정점 셰이더(Vertex Shader)**는 모든 정점에 대해 실행되며, 일반적으로 정점 위치를 모델 공간(model space)에서 스크린 공간(screen space)으로 변환하는 작업을 적용합니다. 또한 정점별 데이터를 파이프라인의 다음 단계로 전달합니다.

*   **테셀레이션 셰이더(Tessellation Shaders)**를 사용하면 특정 규칙에 따라 지오메트리(geometry)를 세분화하여 메시 품질을 높일 수 있습니다. 이는 벽돌 벽이나 계단과 같은 표면이 가까이 있을 때 덜 평평하게 보이도록 만드는 데 자주 사용됩니다.

*   **지오메트리 셰이더(Geometry Shader)**는 모든 프리미티브(primitive, 삼각형, 선, 점)에 대해 실행되며, 이를 폐기하거나 들어온 것보다 더 많은 프리미티브를 출력할 수 있습니다. 이는 테셀레이션 셰이더와 유사하지만 훨씬 더 유연합니다. 하지만 Intel의 내장 GPU를 제외한 대부분의 그래픽 카드에서는 성능이 좋지 않기 때문에 오늘날의 애플리케이션에서는 많이 사용되지 않습니다.

*   **래스터화(Rasterization)** 단계는 프리미티브를 *프래그먼트(fragment)*로 이산화(discretize)합니다. 이것들은 프레임버퍼에서 채우는 픽셀 요소입니다. 화면 밖에 있는 모든 프래그먼트는 폐기되고, 정점 셰이더에서 출력된 속성들은 그림과 같이 프래그먼트 전체에 걸쳐 보간(interpolated)됩니다. 일반적으로 다른 프리미티브 프래그먼트 뒤에 있는 프래그먼트도 깊이 테스팅(depth testing)으로 인해 여기서 폐기됩니다.

*   **프래그먼트 셰이더(Fragment Shader)**는 살아남은 모든 프래그먼트에 대해 호출되며, 프래그먼트가 어떤 프레임버퍼에 어떤 색상과 깊이 값으로 기록될지를 결정합니다. 이는 텍스처 좌표 및 조명을 위한 법선(normal)과 같은 것들을 포함할 수 있는, 정점 셰이더로부터 보간된 데이터를 사용하여 수행할 수 있습니다.

*   **색상 혼합(Color Blending)** 단계는 프레임버퍼의 동일한 픽셀에 매핑되는 다른 프래그먼트들을 혼합하는 연산을 적용합니다. 프래그먼트는 서로를 덮어쓰거나, 더해지거나, 투명도에 따라 혼합될 수 있습니다.

녹색으로 표시된 단계는 **고정 기능(fixed-function)** 단계라고 합니다. 이 단계에서는 매개변수를 사용하여 작업을 조정할 수 있지만, 작동 방식은 미리 정의되어 있습니다.

반면에 주황색으로 표시된 단계는 **프로그래밍 가능(programmable)**하며, 이는 그래픽 카드에 자신만의 코드를 업로드하여 원하는 작업을 정확하게 적용할 수 있음을 의미합니다. 이를 통해 예를 들어 프래그먼트 셰이더를 사용하여 텍스처링과 조명에서부터 레이 트레이서에 이르기까지 모든 것을 구현할 수 있습니다. 이 프로그램들은 많은 GPU 코어에서 동시에 실행되어 정점이나 프래그먼트와 같은 많은 객체를 병렬로 처리합니다.

이전에 OpenGL이나 Direct3D와 같은 오래된 API를 사용해 본 적이 있다면, 파이프라인 설정을 마음대로 변경하는 데 익숙할 것입니다. Vulkan의 그래픽스 파이프라인은 거의 완전히 **불변(immutable)**하므로, 셰이더를 변경하거나, 다른 프레임버퍼를 바인딩하거나, 혼합 함수를 변경하려면 파이프라인을 처음부터 다시 생성해야 합니다. 단점은 렌더링 작업에서 사용하려는 모든 다른 상태 조합을 나타내는 여러 개의 파이프라인을 만들어야 한다는 것입니다. 하지만 파이프라인에서 수행할 모든 작업이 미리 알려져 있기 때문에 드라이버가 이를 훨씬 더 잘 최적화할 수 있습니다.

일부 프로그래밍 가능 단계는 무엇을 하려는지에 따라 선택 사항입니다. 예를 들어, 간단한 지오메트리만 그리는 경우 테셀레이션 및 지오메트리 단계를 비활성화할 수 있습니다. 깊이 값에만 관심이 있다면 프래그먼트 셰이더 단계를 비활성화할 수 있으며, 이는 [그림자 맵핑(shadow mapping)](https://en.wikipedia.org/wiki/Shadow_mapping) 생성에 유용합니다.

다음 챕터에서는 먼저 화면에 삼각형을 표시하는 데 필요한 두 가지 프로그래밍 가능 단계인 정점 셰이더와 프래그먼트 셰이더를 만들 것입니다. 혼합 모드, 뷰포트, 래스터화와 같은 고정 기능 구성은 그 다음 챕터에서 설정할 것입니다. Vulkan에서 그래픽스 파이프라인 설정의 마지막 부분은 입력 및 출력 프레임버퍼를 지정하는 것입니다.

Rust에서는 일반적으로 `App` 또는 이와 유사한 `struct` 내에서 Vulkan 객체를 관리합니다. `init_vulkan`과 같은 초기화 함수에서 `create_image_views` 바로 뒤에 `create_graphics_pipeline`을 호출하도록 수정하세요. 앞으로의 챕터에 걸쳐 이 함수를 구현해 나갈 것입니다.

```rust
// In Rust, these functions are typically methods on a struct that holds Vulkan state.
// Let's assume a struct named `App`.
impl App {
    fn init_vulkan(&mut self, window: &Window) -> Result<()> {
        self.create_instance(window)?;
        self.setup_debug_utils()?;
        self.create_surface(window)?;
        self.pick_physical_device()?;
        self.create_logical_device()?;
        self.create_swapchain()?;
        self.create_image_views()?;
        self.create_graphics_pipeline()?; // Add this call
        Ok(())
    }

    // ... other methods ...
    
    fn create_graphics_pipeline(&mut self) -> Result<()> {
        // We will fill this in the upcoming chapters.
        Ok(())
    }
}
```