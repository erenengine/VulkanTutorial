## 소개

다음 몇 개의 챕터에서는 버텍스 셰이더에 하드코딩된 정점 데이터를 메모리의 버텍스 버퍼로 교체할 것입니다. 가장 쉬운 접근법으로 시작하여, CPU에서 볼 수 있는(visible) 버퍼를 만들고 `memcpy`를 사용해 정점 데이터를 직접 복사하는 방법을 알아볼 것입니다. 그 후에는 스테이징 버퍼(staging buffer)를 사용해 정점 데이터를 고성능 메모리로 복사하는 방법도 살펴볼 것입니다.

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

`inPosition`과 `inColor` 변수는 **정점 속성(vertex attributes)**입니다. 이들은 우리가 이전에 두 배열을 사용해 수동으로 위치와 색상을 지정했던 것처럼, 버텍스 버퍼에서 정점 단위로 지정되는 속성입니다. 버텍스 셰이더를 다시 컴파일하는 것을 잊지 마세요!

`fragColor`와 마찬가지로, `layout(location = x)` 어노테이션은 나중에 참조할 수 있도록 입력에 인덱스를 할당합니다. 64비트 벡터인 `dvec3` 같은 일부 타입은 여러 개의 **슬롯(slot)**을 사용한다는 점을 아는 것이 중요합니다. 즉, 그 다음의 인덱스는 최소 2 이상 커야 합니다.

```glsl
layout(location = 0) in dvec3 inPosition;
layout(location = 2) in vec3 inColor;
```

레이아웃 한정자(layout qualifier)에 대한 더 많은 정보는 [OpenGL 위키](https://www.khronos.org/opengl/wiki/Layout_Qualifier_(GLSL))에서 찾을 수 있습니다.

## 정점 데이터

이제 정점 데이터를 셰이더 코드에서 우리 프로그램 코드의 배열로 옮길 것입니다. 먼저 벡터나 행렬 같은 선형대수 관련 타입을 제공하는 GLM 라이브러리를 포함합니다. 이 타입들을 사용하여 위치와 색상 벡터를 지정할 것입니다.

```c++
#include <glm/glm.hpp>
```

`Vertex`라는 새로운 구조체를 만들고, 그 안에 버텍스 셰이더에서 사용할 두 속성을 넣습니다.

```c++
struct Vertex {
    glm::vec2 pos;
    glm::vec3 color;
};
```

GLM은 셰이더 언어에서 사용되는 벡터 타입과 정확히 일치하는 C++ 타입을 편리하게 제공합니다.

```c++
const std::vector<Vertex> vertices = {
    {{0.0f, -0.5f}, {1.0f, 0.0f, 0.0f}},
    {{0.5f, 0.5f}, {0.0f, 1.0f, 0.0f}},
    {{-0.5f, 0.5f}, {0.0f, 0.0f, 1.0f}}
};
```

이제 `Vertex` 구조체를 사용해 정점 데이터의 배열을 지정합니다. 이전과 정확히 같은 위치와 색상 값을 사용하지만, 이제는 하나의 정점 배열로 결합되었습니다. 이를 **인터리빙(interleaving)** 정점 속성이라고 합니다.

## 바인딩 서술 (Binding descriptions)

다음 단계는 이 데이터 포맷이 GPU 메모리에 업로드된 후, 버텍스 셰이더로 어떻게 전달될지를 Vulkan에게 알려주는 것입니다. 이 정보를 전달하기 위해서는 두 가지 종류의 구조체가 필요합니다.

첫 번째 구조체는 `VkVertexInputBindingDescription`이며, `Vertex` 구조체에 멤버 함수를 추가하여 올바른 데이터로 채우도록 할 것입니다.

```c++
struct Vertex {
    glm::vec2 pos;
    glm::vec3 color;

    static VkVertexInputBindingDescription getBindingDescription() {
        VkVertexInputBindingDescription bindingDescription{};

        return bindingDescription;
    }
};
```

정점 바인딩(vertex binding)은 정점들 전체에서 메모리로부터 데이터를 어떤 속도(rate)로 로드할지 서술합니다. 이는 데이터 항목 사이의 바이트 수와 각 정점 또는 각 인스턴스 이후에 다음 데이터 항목으로 이동할지 여부를 지정합니다.

```c++
VkVertexInputBindingDescription bindingDescription{};
bindingDescription.binding = 0;
bindingDescription.stride = sizeof(Vertex);
bindingDescription.inputRate = VK_VERTEX_INPUT_RATE_VERTEX;
```

우리의 모든 정점별 데이터는 하나의 배열에 함께 묶여 있으므로, 우리는 하나의 바인딩만 가질 것입니다. `binding` 파라미터는 바인딩 배열에서의 인덱스를 지정합니다. `stride` 파라미터는 한 항목에서 다음 항목까지의 바이트 수를 지정하며, `inputRate` 파라미터는 다음 값 중 하나를 가질 수 있습니다.

*   `VK_VERTEX_INPUT_RATE_VERTEX`: 각 정점마다 다음 데이터 항목으로 이동
*   `VK_VERTEX_INPUT_RATE_INSTANCE`: 각 인스턴스마다 다음 데이터 항목으로 이동

우리는 인스턴스 렌더링을 사용하지 않을 것이므로, 정점별(per-vertex) 데이터를 고수할 것입니다.

## 속성 서술 (Attribute descriptions)

정점 입력을 처리하는 방법을 서술하는 두 번째 구조체는 `VkVertexInputAttributeDescription`입니다. 이 구조체들을 채우기 위해 `Vertex`에 또 다른 헬퍼 함수를 추가할 것입니다.

```c++
#include <array>

...

static std::array<VkVertexInputAttributeDescription, 2> getAttributeDescriptions() {
    std::array<VkVertexInputAttributeDescription, 2> attributeDescriptions{};

    return attributeDescriptions;
}
```

함수 프로토타입에서 알 수 있듯이, 이 구조체는 두 개가 될 것입니다. 속성 서술(attribute description) 구조체는 바인딩 서술에서 비롯된 정점 데이터 덩어리(chunk)로부터 어떻게 정점 속성을 추출할지를 서술합니다. 우리는 위치와 색상이라는 두 가지 속성을 가지고 있으므로, 두 개의 속성 서술 구조체가 필요합니다.

```c++
attributeDescriptions[0].binding = 0;
attributeDescriptions[0].location = 0;
attributeDescriptions[0].format = VK_FORMAT_R32G32_SFLOAT;
attributeDescriptions[0].offset = offsetof(Vertex, pos);
```

`binding` 파라미터는 정점별 데이터가 어느 바인딩에서 오는지 Vulkan에게 알려줍니다. `location` 파라미터는 버텍스 셰이더의 입력에 있는 `location` 지시어를 참조합니다. `location`이 `0`인 버텍스 셰이더의 입력은 위치(position)이며, 이는 2개의 32비트 부동소수점 컴포넌트를 가집니다.

`format` 파라미터는 속성의 데이터 타입을 서술합니다. 조금 혼란스러울 수 있지만, 포맷은 색상 포맷과 동일한 열거형(enumeration)으로 지정됩니다. 다음 셰이더 타입과 포맷은 일반적으로 함께 사용됩니다:

*   `float`: `VK_FORMAT_R32_SFLOAT`
*   `vec2`: `VK_FORMAT_R32G32_SFLOAT`
*   `vec3`: `VK_FORMAT_R32G32B32_SFLOAT`
*   `vec4`: `VK_FORMAT_R32G32B32A32_SFLOAT`

보시다시피, 색상 채널의 수가 셰이더 데이터 타입의 컴포넌트 수와 일치하는 포맷을 사용해야 합니다. 셰이더의 컴포넌트 수보다 더 많은 채널을 사용하는 것은 허용되지만, 초과된 채널은 조용히 무시됩니다. 채널 수가 컴포넌트 수보다 적으면, BGA 컴포넌트는 기본값인 `(0, 0, 1)`을 사용하게 됩니다. 색상 타입(`SFLOAT`, `UINT`, `SINT`)과 비트 폭 또한 셰이더 입력의 타입과 일치해야 합니다. 다음 예시를 보세요:

*   `ivec2`: `VK_FORMAT_R32G32_SINT`, 2-컴포넌트 32비트 부호 있는 정수 벡터
*   `uvec4`: `VK_FORMAT_R32G32B32A32_UINT`, 4-컴포넌트 32비트 부호 없는 정수 벡터
*   `double`: `VK_FORMAT_R64_SFLOAT`, 배정밀도(64비트) 부동소수점

`format` 파라미터는 속성 데이터의 바이트 크기를 암시적으로 정의하며, `offset` 파라미터는 정점별 데이터의 시작 부분으로부터 몇 바이트를 읽어야 하는지 지정합니다. 바인딩은 한 번에 하나의 `Vertex`를 로드하며, 위치 속성(`pos`)은 이 구조체의 시작으로부터 `0` 바이트 오프셋에 있습니다. 이는 `offsetof` 매크로를 사용해 자동으로 계산됩니다.

```c++
attributeDescriptions[1].binding = 0;
attributeDescriptions[1].location = 1;
attributeDescriptions[1].format = VK_FORMAT_R32G32B32_SFLOAT;
attributeDescriptions[1].offset = offsetof(Vertex, color);
```

색상 속성도 거의 같은 방식으로 서술됩니다.

## 파이프라인 정점 입력

이제 `createGraphicsPipeline`에서 구조체들을 참조하여, 이 포맷의 정점 데이터를 받도록 그래픽 파이프라인을 설정해야 합니다. `vertexInputInfo` 구조체를 찾아 두 서술을 참조하도록 수정합니다:

```c++
auto bindingDescription = Vertex::getBindingDescription();
auto attributeDescriptions = Vertex::getAttributeDescriptions();

vertexInputInfo.vertexBindingDescriptionCount = 1;
vertexInputInfo.vertexAttributeDescriptionCount = static_cast<uint32_t>(attributeDescriptions.size());
vertexInputInfo.pVertexBindingDescriptions = &bindingDescription;
vertexInputInfo.pVertexAttributeDescriptions = attributeDescriptions.data();
```

이제 파이프라인은 `vertices` 컨테이너 포맷의 정점 데이터를 받아들여 우리 버텍스 셰이더로 전달할 준비가 되었습니다. 만약 지금 검증 레이어를 활성화한 상태로 프로그램을 실행하면, 바인딩에 연결된 버텍스 버퍼가 없다고 불평하는 것을 볼 수 있을 것입니다. 다음 단계는 버텍스 버퍼를 생성하고 정점 데이터를 그곳으로 옮겨 GPU가 접근할 수 있도록 하는 것입니다.

[C++ 코드](/code/18_vertex_input.cpp) /
[버텍스 셰이더](/code/18_shader_vertexbuffer.vert) /
[프래그먼트 셰이더](/code/18_shader_vertexbuffer.frag)