이전의 그래픽 API들은 그래픽 파이프라인의 대부분 단계에 대한 기본 상태를 제공했습니다. Vulkan에서는 대부분의 파이프라인 상태를 명시적으로 지정해야 하며, 이 상태들은 불변(immutable)의 파이프라인 상태 객체(PSO)로 구워지기(baked) 때문입니다. 이번 장에서는 이러한 고정 함수(fixed-function) 연산을 구성하기 위한 모든 구조체를 채워 넣을 것입니다.

## 동적 상태 (Dynamic state)

*대부분의* 파이프라인 상태는 파이프라인 상태 객체에 구워져야 하지만, 제한된 일부 상태는 파이프라인을 다시 만들지 않고도 드로우 타임(draw time)에 변경할 수 *있습니다*. 뷰포트의 크기, 선 두께, 블렌딩 상수 등이 그 예입니다. 만약 동적 상태를 사용하고 이러한 속성들을 파이프라인 생성 시에 고정하지 않으려면, 다음과 같이 `VkPipelineDynamicStateCreateInfo` 구조체를 채워야 합니다.

```c++
std::vector<VkDynamicState> dynamicStates = {
    VK_DYNAMIC_STATE_VIEWPORT,
    VK_DYNAMIC_STATE_SCISSOR
};

VkPipelineDynamicStateCreateInfo dynamicState{};
dynamicState.sType = VK_STRUCTURE_TYPE_PIPELINE_DYNAMIC_STATE_CREATE_INFO;
dynamicState.dynamicStateCount = static_cast<uint32_t>(dynamicStates.size());
dynamicState.pDynamicStates = dynamicStates.data();
```

이렇게 하면 이 값들의 구성이 파이프라인 생성 시에는 무시되며, 드로잉 시점에 이 데이터를 지정할 수 있게 되고 또 지정해야만 합니다. 이는 더 유연한 설정을 가능하게 하며, 뷰포트나 시저 상태처럼 파이프라인 상태에 고정시킬 경우 설정이 더 복잡해질 수 있는 항목들에 대해 매우 일반적인 방식입니다.

## 정점 입력 (Vertex input)

`VkPipelineVertexInputStateCreateInfo` 구조체는 정점 셰이더로 전달될 정점 데이터의 형식을 설명합니다. 이 설명은 크게 두 가지 방식으로 이루어집니다.

*   **바인딩(Bindings)**: 데이터 간의 간격 및 데이터가 정점별(per-vertex)인지 인스턴스별(per-instance)인지 여부 (자세한 내용은 [인스턴싱](https://ko.wikipedia.org/wiki/인스턴싱) 참조)
*   **속성 서술(Attribute descriptions)**: 정점 셰이더로 전달되는 속성의 유형, 어떤 바인딩에서 로드할지, 그리고 어떤 오프셋에 있는지

지금은 정점 데이터를 정점 셰이더에 직접 하드코딩하고 있으므로, 이 구조체를 채워서 로드할 정점 데이터가 없음을 명시할 것입니다. 이 부분은 정점 버퍼(vertex buffer) 장에서 다시 다룰 것입니다.

```c++
VkPipelineVertexInputStateCreateInfo vertexInputInfo{};
vertexInputInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_VERTEX_INPUT_STATE_CREATE_INFO;
vertexInputInfo.vertexBindingDescriptionCount = 0;
vertexInputInfo.pVertexBindingDescriptions = nullptr; // Optional
vertexInputInfo.vertexAttributeDescriptionCount = 0;
vertexInputInfo.pVertexAttributeDescriptions = nullptr; // Optional
```

`pVertexBindingDescriptions`와 `pVertexAttributeDescriptions` 멤버는 정점 데이터 로딩에 대한 위 세부 사항들을 설명하는 구조체 배열을 가리킵니다. `createGraphicsPipeline` 함수에서 `shaderStages` 배열 바로 다음에 이 구조체를 추가하세요.

## 입력 조립 (Input assembly)

`VkPipelineInputAssemblyStateCreateInfo` 구조체는 두 가지를 설명합니다: 정점들로부터 어떤 종류의 지오메트리를 그릴 것인지, 그리고 프리미티브 재시작(primitive restart)을 활성화할 것인지입니다. 전자는 `topology` 멤버에서 지정하며, 다음과 같은 값을 가질 수 있습니다.

*   `VK_PRIMITIVE_TOPOLOGY_POINT_LIST`: 정점들로부터 점을 그림
*   `VK_PRIMITIVE_TOPOLOGY_LINE_LIST`: 2개의 정점마다 재사용 없이 선을 그림
*   `VK_PRIMITIVE_TOPOLOGY_LINE_STRIP`: 각 선의 끝 정점이 다음 선의 시작 정점으로 사용됨
*   `VK_PRIMITIVE_TOPOLOGY_TRIANGLE_LIST`: 3개의 정점마다 재사용 없이 삼각형을 그림
*   `VK_PRIMITIVE_TOPOLOGY_TRIANGLE_STRIP `: 각 삼각형의 두 번째와 세 번째 정점이 다음 삼각형의 첫 두 정점으로 사용됨

일반적으로 정점들은 정점 버퍼에서 인덱스 순서대로 로드되지만, *엘리먼트 버퍼(element buffer)*를 사용하면 사용할 인덱스를 직접 지정할 수 있습니다. 이를 통해 정점 재사용과 같은 최적화를 수행할 수 있습니다. 만약 `primitiveRestartEnable` 멤버를 `VK_TRUE`로 설정하면, `0xFFFF` 또는 `0xFFFFFFFF` 같은 특수 인덱스를 사용하여 `_STRIP` 토폴로지 모드에서 선과 삼각형을 분리할 수 있습니다.

이 튜토리얼에서는 계속 삼각형을 그릴 것이므로, 구조체에 다음 데이터를 사용하겠습니다.

```c++
VkPipelineInputAssemblyStateCreateInfo inputAssembly{};
inputAssembly.sType = VK_STRUCTURE_TYPE_PIPELINE_INPUT_ASSEMBLY_STATE_CREATE_INFO;
inputAssembly.topology = VK_PRIMITIVE_TOPOLOGY_TRIANGLE_LIST;
inputAssembly.primitiveRestartEnable = VK_FALSE;
```

## 뷰포트와 시저 (Viewports and scissors)

뷰포트(viewport)는 기본적으로 출력이 렌더링될 프레임버퍼의 영역을 설명합니다. 이는 거의 항상 `(0, 0)`에서 `(width, height)`까지이며, 이 튜토리얼에서도 마찬가지입니다.

```c++
VkViewport viewport{};
viewport.x = 0.0f;
viewport.y = 0.0f;
viewport.width = (float) swapChainExtent.width;
viewport.height = (float) swapChainExtent.height;
viewport.minDepth = 0.0f;
viewport.maxDepth = 1.0f;
```

스왑체인과 그 이미지들의 크기는 창의 `WIDTH`와 `HEIGHT`와 다를 수 있음을 기억하세요. 스왑체인 이미지들은 나중에 프레임버퍼로 사용될 것이므로, 그 크기를 따라야 합니다.

`minDepth`와 `maxDepth` 값은 프레임버퍼에 사용할 깊이 값의 범위를 지정합니다. 이 값들은 `[0.0f, 1.0f]` 범위 내에 있어야 하지만, `minDepth`가 `maxDepth`보다 클 수도 있습니다. 특별한 작업을 하지 않는다면 표준 값인 `0.0f`와 `1.0f`를 사용해야 합니다.

뷰포트가 이미지에서 프레임버퍼로의 변환을 정의하는 반면, 시저 사각형(scissor rectangle)은 실제로 픽셀이 저장될 영역을 정의합니다. 시저 사각형 밖의 모든 픽셀은 래스터라이저에 의해 버려집니다. 이는 변환이라기보다는 필터처럼 작동합니다. 차이점은 아래 그림에 설명되어 있습니다. 왼쪽 시저 사각형은 뷰포트보다 크기만 하다면 해당 이미지를 만들어내는 많은 가능성 중 하나일 뿐입니다.

![](/images/viewports_scissors.png)

따라서 전체 프레임버퍼에 그리고 싶다면, 프레임버퍼 전체를 덮는 시저 사각형을 지정하면 됩니다.

```c++
VkRect2D scissor{};
scissor.offset = {0, 0};
scissor.extent = swapChainExtent;
```

뷰포트와 시저 사각형은 파이프라인의 정적(static) 부분으로 지정하거나, 커맨드 버퍼에서 설정되는 [동적 상태](#동적-상태)로 지정할 수 있습니다. 전자가 다른 상태들과 더 일관성이 있지만, 뷰포트와 시저 상태를 동적으로 만드는 것이 훨씬 더 많은 유연성을 제공하기 때문에 종종 편리합니다. 이는 매우 일반적인 방식이며 모든 구현체에서 성능 저하 없이 이 동적 상태를 처리할 수 있습니다.

동적 뷰포트와 시저 사각형을 선택하는 경우, 파이프라인에 대해 해당 동적 상태를 활성화해야 합니다.

```c++
std::vector<VkDynamicState> dynamicStates = {
    VK_DYNAMIC_STATE_VIEWPORT,
    VK_DYNAMIC_STATE_SCISSOR
};

VkPipelineDynamicStateCreateInfo dynamicState{};
dynamicState.sType = VK_STRUCTURE_TYPE_PIPELINE_DYNAMIC_STATE_CREATE_INFO;
dynamicState.dynamicStateCount = static_cast<uint32_t>(dynamicStates.size());
dynamicState.pDynamicStates = dynamicStates.data();
```

그런 다음 파이프라인 생성 시에는 그들의 개수만 지정하면 됩니다.

```c++
VkPipelineViewportStateCreateInfo viewportState{};
viewportState.sType = VK_STRUCTURE_TYPE_PIPELINE_VIEWPORT_STATE_CREATE_INFO;
viewportState.viewportCount = 1;
viewportState.scissorCount = 1;
```

실제 뷰포트와 시저 사각형은 나중에 드로잉 시점에 설정될 것입니다. 동적 상태를 사용하면 단일 커맨드 버퍼 내에서 다른 뷰포트 및/또는 시저 사각형을 지정하는 것조차 가능합니다.

동적 상태를 사용하지 않는다면, 뷰포트와 시저 사각형은 `VkPipelineViewportStateCreateInfo` 구조체를 사용하여 파이프라인에 설정되어야 합니다. 이는 이 파이프라인의 뷰포트와 시저 사각형을 불변으로 만듭니다. 이 값들을 변경하려면 새 값으로 새 파이프라인을 생성해야 합니다.

```c++
VkPipelineViewportStateCreateInfo viewportState{};
viewportState.sType = VK_STRUCTURE_TYPE_PIPELINE_VIEWPORT_STATE_CREATE_INFO;
viewportState.viewportCount = 1;
viewportState.pViewports = &viewport;
viewportState.scissorCount = 1;
viewportState.pScissors = &scissor;
```

어떻게 설정하든, 일부 그래픽 카드에서는 여러 개의 뷰포트와 시저 사각형을 사용할 수 있으며, 이를 위해 구조체 멤버들이 배열을 참조합니다. 여러 개를 사용하려면 GPU 기능을 활성화해야 합니다(논리 장치 생성 참조).

## 래스터라이저 (Rasterizer)

래스터라이저는 정점 셰이더에서 만들어진 지오메트리를 가져와 프래그먼트 셰이더에서 색상을 칠할 프래그먼트(fragment)로 변환합니다. 또한 [깊이 테스팅](https://ko.wikipedia.org/wiki/Z-버퍼링), [면 컬링](https://ko.wikipedia.org/wiki/후면_추려내기), 시저 테스트를 수행하며, 폴리곤 전체를 채우는 프래그먼트나 가장자리만(와이어프레임 렌더링) 출력하도록 구성할 수 있습니다. 이 모든 것은 `VkPipelineRasterizationStateCreateInfo` 구조체를 사용하여 구성됩니다.

```c++
VkPipelineRasterizationStateCreateInfo rasterizer{};
rasterizer.sType = VK_STRUCTURE_TYPE_PIPELINE_RASTERIZATION_STATE_CREATE_INFO;
rasterizer.depthClampEnable = VK_FALSE;
```

`depthClampEnable`이 `VK_TRUE`로 설정되면, 근평면(near plane)과 원평면(far plane)을 벗어나는 프래그먼트는 버려지는 대신 평면에 클램핑됩니다. 이는 섀도우 맵과 같은 일부 특수한 경우에 유용합니다. 이를 사용하려면 GPU 기능을 활성화해야 합니다.

```c++
rasterizer.rasterizerDiscardEnable = VK_FALSE;
```

`rasterizerDiscardEnable`이 `VK_TRUE`로 설정되면, 지오메트리는 래스터라이저 단계를 통과하지 않습니다. 이는 기본적으로 프레임버퍼로의 모든 출력을 비활성화합니다.

```c++
rasterizer.polygonMode = VK_POLYGON_MODE_FILL;
```

`polygonMode`는 지오메트리에 대해 프래그먼트가 생성되는 방식을 결정합니다. 사용 가능한 모드는 다음과 같습니다.

*   `VK_POLYGON_MODE_FILL`: 폴리곤 영역을 프래그먼트로 채움
*   `VK_POLYGON_MODE_LINE`: 폴리곤 가장자리를 선으로 그림
*   `VK_POLYGON_MODE_POINT`: 폴리곤 정점을 점으로 그림

`fill` 이외의 모드를 사용하려면 GPU 기능을 활성화해야 합니다.

```c++
rasterizer.lineWidth = 1.0f;
```

`lineWidth` 멤버는 직관적으로, 프래그먼트 개수 단위로 선의 두께를 나타냅니다. 지원되는 최대 선 두께는 하드웨어에 따라 다르며, `1.0f`보다 두꺼운 선을 사용하려면 `wideLines` GPU 기능을 활성화해야 합니다.

```c++
rasterizer.cullMode = VK_CULL_MODE_BACK_BIT;
rasterizer.frontFace = VK_FRONT_FACE_CLOCKWISE;
```

`cullMode` 변수는 사용할 면 컬링(face culling)의 유형을 결정합니다. 컬링을 비활성화하거나, 앞면(front face)을 컬링하거나, 뒷면(back face)을 컬링하거나, 둘 다 컬링할 수 있습니다. `frontFace` 변수는 앞면으로 간주될 면의 정점 순서를 지정하며, 시계 방향(clockwise) 또는 반시계 방향(counter-clockwise)이 될 수 있습니다.

```c++
rasterizer.depthBiasEnable = VK_FALSE;
rasterizer.depthBiasConstantFactor = 0.0f; // Optional
rasterizer.depthBiasClamp = 0.0f; // Optional
rasterizer.depthBiasSlopeFactor = 0.0f; // Optional
```

래스터라이저는 상수 값을 더하거나 프래그먼트의 기울기에 따라 깊이 값을 변경할 수 있습니다. 이는 때때로 섀도우 매핑에 사용되지만, 우리는 사용하지 않을 것입니다. `depthBiasEnable`을 `VK_FALSE`로 설정하세요.

## 멀티샘플링 (Multisampling)

`VkPipelineMultisampleStateCreateInfo` 구조체는 [안티 앨리어싱](https://ko.wikipedia.org/wiki/다중샘플링_안티에일리어싱)을 수행하는 방법 중 하나인 멀티샘플링을 구성합니다. 동일한 픽셀로 래스터화되는 여러 폴리곤의 프래그먼트 셰이더 결과를 결합하여 작동합니다. 이는 주로 가장자리에서 발생하며, 가장 눈에 띄는 앨리어싱 현상이 발생하는 곳이기도 합니다. 하나의 폴리곤만 픽셀에 매핑되는 경우에는 프래그먼트 셰이더를 여러 번 실행할 필요가 없기 때문에, 단순히 더 높은 해상도로 렌더링한 다음 다운스케일링하는 것보다 훨씬 비용이 적게 듭니다. 이를 활성화하려면 GPU 기능을 활성화해야 합니다.

```c++
VkPipelineMultisampleStateCreateInfo multisampling{};
multisampling.sType = VK_STRUCTURE_TYPE_PIPELINE_MULTISAMPLE_STATE_CREATE_INFO;
multisampling.sampleShadingEnable = VK_FALSE;
multisampling.rasterizationSamples = VK_SAMPLE_COUNT_1_BIT;
multisampling.minSampleShading = 1.0f; // Optional
multisampling.pSampleMask = nullptr; // Optional
multisampling.alphaToCoverageEnable = VK_FALSE; // Optional
multisampling.alphaToOneEnable = VK_FALSE; // Optional
```

멀티샘플링은 나중 장에서 다시 다룰 것이므로, 지금은 비활성화 상태로 두겠습니다.

## 깊이 및 스텐실 테스팅 (Depth and stencil testing)

깊이 및/또는 스텐실 버퍼를 사용하는 경우, `VkPipelineDepthStencilStateCreateInfo`를 사용하여 깊이 및 스텐실 테스트도 구성해야 합니다. 지금은 없으므로 해당 구조체에 대한 포인터 대신 단순히 `nullptr`를 전달할 수 있습니다. 이 부분은 깊이 버퍼링 장에서 다시 다루겠습니다.

## 색상 혼합 (Color blending)

프래그먼트 셰이더가 색상을 반환한 후, 프레임버퍼에 이미 있는 색상과 결합되어야 합니다. 이 변환은 색상 혼합(color blending)으로 알려져 있으며, 두 가지 방법이 있습니다.

*   이전 값과 새 값을 혼합하여 최종 색상을 생성
*   비트 연산을 사용하여 이전 값과 새 값을 결합

색상 혼합을 구성하는 데는 두 가지 유형의 구조체가 있습니다. 첫 번째 구조체인 `VkPipelineColorBlendAttachmentState`는 연결된 각 프레임버퍼별 구성을 포함하고, 두 번째 구조체인 `VkPipelineColorBlendStateCreateInfo`는 *전역* 색상 혼합 설정을 포함합니다. 우리의 경우 프레임버퍼는 하나뿐입니다.

```c++
VkPipelineColorBlendAttachmentState colorBlendAttachment{};
colorBlendAttachment.colorWriteMask = VK_COLOR_COMPONENT_R_BIT | VK_COLOR_COMPONENT_G_BIT | VK_COLOR_COMPONENT_B_BIT | VK_COLOR_COMPONENT_A_BIT;
colorBlendAttachment.blendEnable = VK_FALSE;
colorBlendAttachment.srcColorBlendFactor = VK_BLEND_FACTOR_ONE; // Optional
colorBlendAttachment.dstColorBlendFactor = VK_BLEND_FACTOR_ZERO; // Optional
colorBlendAttachment.colorBlendOp = VK_BLEND_OP_ADD; // Optional
colorBlendAttachment.srcAlphaBlendFactor = VK_BLEND_FACTOR_ONE; // Optional
colorBlendAttachment.dstAlphaBlendFactor = VK_BLEND_FACTOR_ZERO; // Optional
colorBlendAttachment.alphaBlendOp = VK_BLEND_OP_ADD; // Optional
```

이 프레임버퍼별 구조체를 사용하면 첫 번째 방식의 색상 혼합을 구성할 수 있습니다. 수행될 연산은 다음 의사 코드로 가장 잘 설명할 수 있습니다.

```c++
if (blendEnable) {
    finalColor.rgb = (srcColorBlendFactor * newColor.rgb) <colorBlendOp> (dstColorBlendFactor * oldColor.rgb);
    finalColor.a = (srcAlphaBlendFactor * newColor.a) <alphaBlendOp> (dstAlphaBlendFactor * oldColor.a);
} else {
    finalColor = newColor;
}

finalColor = finalColor & colorWriteMask;
```

`blendEnable`이 `VK_FALSE`로 설정되면 프래그먼트 셰이더의 새 색상이 수정 없이 그대로 전달됩니다. 그렇지 않으면 두 혼합 연산이 수행되어 새 색상을 계산합니다. 결과 색상은 `colorWriteMask`와 AND 연산되어 실제로 어떤 채널이 통과할지 결정됩니다.

색상 혼합을 사용하는 가장 일반적인 방법은 알파 블렌딩을 구현하는 것입니다. 여기서 우리는 새 색상이 불투명도에 따라 이전 색상과 혼합되기를 원합니다. `finalColor`는 다음과 같이 계산되어야 합니다.

```c++
finalColor.rgb = newAlpha * newColor + (1 - newAlpha) * oldColor;
finalColor.a = newAlpha.a;
```

이는 다음 매개변수로 달성할 수 있습니다.

```c++
colorBlendAttachment.blendEnable = VK_TRUE;
colorBlendAttachment.srcColorBlendFactor = VK_BLEND_FACTOR_SRC_ALPHA;
colorBlendAttachment.dstColorBlendFactor = VK_BLEND_FACTOR_ONE_MINUS_SRC_ALPHA;
colorBlendAttachment.colorBlendOp = VK_BLEND_OP_ADD;
colorBlendAttachment.srcAlphaBlendFactor = VK_BLEND_FACTOR_ONE;
colorBlendAttachment.dstAlphaBlendFactor = VK_BLEND_FACTOR_ZERO;
colorBlendAttachment.alphaBlendOp = VK_BLEND_OP_ADD;
```

모든 가능한 연산은 명세서의 `VkBlendFactor` 및 `VkBlendOp` 열거형에서 찾을 수 있습니다.

두 번째 구조체는 모든 프레임버퍼에 대한 구조체 배열을 참조하며, 앞서 언급한 계산에서 혼합 인자로 사용할 수 있는 블렌드 상수를 설정할 수 있습니다.

```c++
VkPipelineColorBlendStateCreateInfo colorBlending{};
colorBlending.sType = VK_STRUCTURE_TYPE_PIPELINE_COLOR_BLEND_STATE_CREATE_INFO;
colorBlending.logicOpEnable = VK_FALSE;
colorBlending.logicOp = VK_LOGIC_OP_COPY; // Optional
colorBlending.attachmentCount = 1;
colorBlending.pAttachments = &colorBlendAttachment;
colorBlending.blendConstants[0] = 0.0f; // Optional
colorBlending.blendConstants[1] = 0.0f; // Optional
colorBlending.blendConstants[2] = 0.0f; // Optional
colorBlending.blendConstants[3] = 0.0f; // Optional
```

두 번째 혼합 방법(비트 조합)을 사용하려면 `logicOpEnable`을 `VK_TRUE`로 설정해야 합니다. 비트 연산은 `logicOp` 필드에서 지정할 수 있습니다. 이렇게 하면 연결된 모든 프레임버퍼에 대해 `blendEnable`을 `VK_FALSE`로 설정한 것처럼 첫 번째 방법이 자동으로 비활성화됩니다! `colorWriteMask`는 이 모드에서도 프레임버퍼의 어떤 채널이 실제로 영향을 받을지 결정하는 데 사용됩니다. 여기서처럼 두 모드를 모두 비활성화하는 것도 가능하며, 이 경우 프래그먼트 색상이 수정 없이 프레임버퍼에 쓰여집니다.

## 파이프라인 레이아웃 (Pipeline layout)

셰이더에서 `uniform` 값을 사용할 수 있습니다. 이는 동적 상태 변수와 유사한 전역 변수로, 드로잉 시점에 변경하여 셰이더를 다시 만들지 않고도 동작을 변경할 수 있습니다. 일반적으로 변환 행렬을 정점 셰이더에 전달하거나 프래그먼트 셰이더에서 텍스처 샘플러를 생성하는 데 사용됩니다.

이러한 uniform 값은 파이프라인 생성 중에 `VkPipelineLayout` 객체를 생성하여 지정해야 합니다. 나중 장까지는 사용하지 않겠지만, 비어있는 파이프라인 레이아웃이라도 반드시 생성해야 합니다.

나중에 다른 함수에서 이 객체를 참조할 것이므로 클래스 멤버를 만들어 이 객체를 저장합니다.

```c++
VkPipelineLayout pipelineLayout;
```

그리고 `createGraphicsPipeline` 함수에서 객체를 생성합니다.

```c++
VkPipelineLayoutCreateInfo pipelineLayoutInfo{};
pipelineLayoutInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_LAYOUT_CREATE_INFO;
pipelineLayoutInfo.setLayoutCount = 0; // Optional
pipelineLayoutInfo.pSetLayouts = nullptr; // Optional
pipelineLayoutInfo.pushConstantRangeCount = 0; // Optional
pipelineLayoutInfo.pPushConstantRanges = nullptr; // Optional

if (vkCreatePipelineLayout(device, &pipelineLayoutInfo, nullptr, &pipelineLayout) != VK_SUCCESS) {
    throw std::runtime_error("failed to create pipeline layout!");
}
```

이 구조체는 *푸시 상수(push constants)*도 지정하는데, 이는 셰이더에 동적 값을 전달하는 또 다른 방법으로, 나중에 다룰 수 있습니다. 파이프라인 레이아웃은 프로그램 수명 내내 참조되므로 마지막에 파괴되어야 합니다.

```c++
void cleanup() {
    vkDestroyPipelineLayout(device, pipelineLayout, nullptr);
    ...
}
```

## 결론

이것으로 모든 고정 함수 상태에 대한 설정이 끝났습니다! 모든 것을 처음부터 설정하는 것은 많은 작업이지만, 그 장점은 이제 그래픽 파이프라인에서 일어나는 거의 모든 일을 완전히 인지하게 되었다는 것입니다! 이는 특정 컴포넌트의 기본 상태가 예상과 달라 예기치 않은 동작에 부딪힐 가능성을 줄여줍니다.

하지만 그래픽 파이프라인을 최종적으로 생성하기 전에 만들어야 할 객체가 하나 더 있으며, 그것은 바로 [렌더 패스](!en/Drawing_a_triangle/Graphics_pipeline_basics/Render_passes)입니다.

[C++ 코드](/code/10_fixed_functions.cpp) /
[정점 셰이더](/code/09_shader_base.vert) /
[프래그먼트 셰이더](/code/09_shader_base.frag)