## 소개

이제 여러분의 프로그램은 텍스처가 입혀진 3D 메시를 렌더링할 준비가 되었습니다. 하지만 현재 `vertices`와 `indices` 배열에 있는 지오메트리는 아직 그다지 흥미롭지 않습니다. 이번 챕터에서는 그래픽 카드가 실제로 어떤 작업을 하도록 만들기 위해, 실제 모델 파일에서 정점과 인덱스를 로드하도록 프로그램을 확장할 것입니다.

많은 그래픽 API 튜토리얼에서는 이와 같은 챕터에서 독자에게 직접 OBJ 로더를 작성하도록 합니다. 하지만 이 방식의 문제점은, 조금이라도 흥미로운 3D 애플리케이션이라면 곧 골격 애니메이션(skeletal animation)과 같이 OBJ 파일 형식이 지원하지 않는 기능이 필요해진다는 것입니다. 이번 챕터에서 OBJ 모델로부터 메시 데이터를 로드하긴 하겠지만, 파일에서 메시 데이터를 로드하는 세부 사항보다는, 메시 데이터를 프로그램 자체에 통합하는 데 더 중점을 둘 것입니다.

## 라이브러리

정점과 면(face)을 OBJ 파일에서 로드하기 위해 [tinyobjloader](https://github.com/syoyo/tinyobjloader) 라이브러리를 사용할 것입니다. 이 라이브러리는 빠르고, `stb_image`처럼 단일 파일 라이브러리라서 통합하기 쉽습니다. 위 링크의 저장소로 가서 `tiny_obj_loader.h` 파일을 다운로드하여 여러분의 라이브러리 디렉터리 내의 폴더에 넣으세요.

**Visual Studio**

`tiny_obj_loader.h`가 있는 디렉터리를 `추가 포함 디렉터리(Additional Include Directories)` 경로에 추가하세요.

![](/images/include_dirs_tinyobjloader.png)

**Makefile**

`tiny_obj_loader.h`가 있는 디렉터리를 GCC의 포함 디렉터리에 추가하세요:

```text
VULKAN_SDK_PATH = /home/user/VulkanSDK/x.x.x.x/x86_64
STB_INCLUDE_PATH = /home/user/libraries/stb
TINYOBJ_INCLUDE_PATH = /home/user/libraries/tinyobjloader

...

CFLAGS = -std=c++17 -I$(VULKAN_SDK_PATH)/include -I$(STB_INCLUDE_PATH) -I$(TINYOBJ_INCLUDE_PATH)
```

## 샘플 메시

이번 챕터에서는 아직 조명을 활성화하지 않을 것이므로, 텍스처에 조명이 미리 구워진(baked) 샘플 모델을 사용하는 것이 도움이 됩니다. 이러한 모델을 찾는 쉬운 방법은 [Sketchfab](https://sketchfab.com/)에서 3D 스캔 모델을 찾아보는 것입니다. 해당 사이트의 많은 모델이 허용적인 라이선스와 함께 OBJ 형식으로 제공됩니다.

이 튜토리얼에서는 [nigelgoh](https://sketchfab.com/nigelgoh)의 [Viking room](https://sketchfab.com/3d-models/viking-room-a49f1b8e4f5c4ecf9e1fe7d81915ad38) 모델([CC BY 4.0](https://web.archive.org/web/20200428202538/https://sketchfab.com/3d-models/viking-room-a49f1b8e4f5c4ecf9e1fe7d81915ad38))을 사용하기로 결정했습니다. 현재 지오메트리를 바로 대체하여 사용할 수 있도록 모델의 크기와 방향을 조정했습니다:

*   [viking_room.obj](/resources/viking_room.obj)
*   [viking_room.png](/resources/viking_room.png)

자신만의 모델을 자유롭게 사용해도 되지만, 해당 모델이 단 하나의 재질(material)로만 구성되어 있고 크기가 약 1.5 x 1.5 x 1.5 단위인지 확인하세요. 이보다 크면 뷰 행렬을 변경해야 합니다. 모델 파일을 `shaders`와 `textures` 옆에 새로운 `models` 디렉터리를 만들어 넣고, 텍스처 이미지는 `textures` 디렉터리에 넣으세요.

모델과 텍스처 경로를 정의하기 위해 프로그램에 두 개의 새로운 설정 변수를 추가하세요:

```c++
const uint32_t WIDTH = 800;
const uint32_t HEIGHT = 600;

const std::string MODEL_PATH = "models/viking_room.obj";
const std::string TEXTURE_PATH = "textures/viking_room.png";
```

그리고 `createTextureImage`가 이 경로 변수를 사용하도록 업데이트하세요:

```c++
stbi_uc* pixels = stbi_load(TEXTURE_PATH.c_str(), &texWidth, &texHeight, &texChannels, STBI_rgb_alpha);
```

## 정점과 인덱스 로드하기

이제 모델 파일에서 정점과 인덱스를 로드할 것이므로, 전역 `vertices`와 `indices` 배열을 제거해야 합니다. 이들을 `const`가 아닌 컨테이너 클래스 멤버로 교체하세요:

```c++
std::vector<Vertex> vertices;
std::vector<uint32_t> indices;
VkBuffer vertexBuffer;
VkDeviceMemory vertexBufferMemory;
```

정점의 개수가 65535개를 초과할 것이기 때문에, 인덱스의 타입을 `uint16_t`에서 `uint32_t`로 변경해야 합니다. `vkCmdBindIndexBuffer`의 파라미터도 변경하는 것을 잊지 마세요:

```c++
vkCmdBindIndexBuffer(commandBuffer, indexBuffer, 0, VK_INDEX_TYPE_UINT32);
```

tinyobjloader 라이브러리는 STB 라이브러리와 같은 방식으로 포함됩니다. `tiny_obj_loader.h` 파일을 포함하고, 링커 오류를 피하기 위해 하나의 소스 파일에 `TINYOBJLOADER_IMPLEMENTATION`을 정의하여 함수 본문을 포함시켜야 합니다:

```c++
#define TINYOBJLOADER_IMPLEMENTATION
#include <tiny_obj_loader.h>
```

이제 이 라이브러리를 사용하여 `vertices`와 `indices` 컨테이너를 메시의 정점 데이터로 채우는 `loadModel` 함수를 작성할 것입니다. 이 함수는 정점 및 인덱스 버퍼가 생성되기 전 어딘가에서 호출되어야 합니다:

```c++
void initVulkan() {
    ...
    loadModel();
    createVertexBuffer();
    createIndexBuffer();
    ...
}

...

void loadModel() {

}
```

모델은 `tinyobj::LoadObj` 함수를 호출하여 라이브러리의 데이터 구조로 로드됩니다:

```c++
void loadModel() {
    tinyobj::attrib_t attrib;
    std::vector<tinyobj::shape_t> shapes;
    std::vector<tinyobj::material_t> materials;
    std::string err;

    if (!tinyobj::LoadObj(&attrib, &shapes, &materials, &err, MODEL_PATH.c_str())) {
        throw std::runtime_error(err);
    }
}
```

OBJ 파일은 위치, 법선(normal), 텍스처 좌표, 그리고 면(face)으로 구성됩니다. 면은 임의의 수의 정점으로 구성되며, 각 정점은 인덱스를 통해 위치, 법선 및/또는 텍스처 좌표를 참조합니다. 이를 통해 전체 정점뿐만 아니라 개별 속성도 재사용할 수 있습니다.

`attrib` 컨테이너는 `attrib.vertices`, `attrib.normals`, `attrib.texcoords` 벡터에 모든 위치, 법선, 텍스처 좌표를 담고 있습니다. `shapes` 컨테이너는 모든 개별 객체와 그 면들을 포함합니다. 각 면은 정점 배열로 구성되며, 각 정점은 위치, 법선, 텍스처 좌표 속성의 인덱스를 포함합니다. OBJ 모델은 면마다 재질과 텍스처를 정의할 수도 있지만, 우리는 이들을 무시할 것입니다.

`err` 문자열에는 파일 로딩 중 발생한 오류나, 재질 정의 누락과 같은 경고가 포함됩니다. 로딩은 `LoadObj` 함수가 `false`를 반환할 때만 실패한 것으로 간주합니다. 위에서 언급했듯이, OBJ 파일의 면은 실제로는 임의의 수의 정점을 가질 수 있지만, 우리 애플리케이션은 삼각형만 렌더링할 수 있습니다. 다행히도 `LoadObj`는 이러한 면을 자동으로 삼각형으로 변환하는 선택적 파라미터를 가지고 있으며, 이는 기본적으로 활성화되어 있습니다.

파일의 모든 면을 단일 모델로 결합할 것이므로, 모든 shape를 순회하기만 하면 됩니다:

```c++
for (const auto& shape : shapes) {

}
```

삼각화 기능 덕분에 이미 면당 3개의 정점이 보장되므로, 이제 바로 정점들을 순회하며 `vertices` 벡터에 직접 넣을 수 있습니다:

```c++
for (const auto& shape : shapes) {
    for (const auto& index : shape.mesh.indices) {
        Vertex vertex{};

        vertices.push_back(vertex);
        indices.push_back(indices.size());
    }
}
```

단순화를 위해, 지금은 모든 정점이 고유하다고 가정하므로 간단하게 자동 증가하는 인덱스를 사용합니다. `index` 변수는 `tinyobj::index_t` 타입으로, `vertex_index`, `normal_index`, `texcoord_index` 멤버를 포함합니다. 이 인덱스들을 사용하여 `attrib` 배열에서 실제 정점 속성을 찾아야 합니다:

```c++
vertex.pos = {
    attrib.vertices[3 * index.vertex_index + 0],
    attrib.vertices[3 * index.vertex_index + 1],
    attrib.vertices[3 * index.vertex_index + 2]
};

vertex.texCoord = {
    attrib.texcoords[2 * index.texcoord_index + 0],
    attrib.texcoords[2 * index.texcoord_index + 1]
};

vertex.color = {1.0f, 1.0f, 1.0f};
```

안타깝게도 `attrib.vertices` 배열은 `glm::vec3` 같은 것이 아니라 `float` 값의 배열이므로, 인덱스에 `3`을 곱해야 합니다. 마찬가지로, 항목당 두 개의 텍스처 좌표 성분이 있습니다. 오프셋 `0`, `1`, `2`는 X, Y, Z 성분에 접근하는 데 사용되며, 텍스처 좌표의 경우 U, V 성분에 접근하는 데 사용됩니다.

이제 최적화를 활성화한 상태로 프로그램을 실행하세요 (예: Visual Studio에서는 `Release` 모드, GCC에서는 `-O3` 컴파일러 플래그 사용). 최적화가 없으면 모델 로딩이 매우 느려지므로 이 과정이 필요합니다. 다음과 같은 화면을 볼 수 있을 것입니다:

![](/images/inverted_texture_coordinates.png)

좋습니다, 지오메트리는 올바르게 보이지만 텍스처는 왜 저럴까요? OBJ 형식은 수직 좌표 `0`이 이미지의 하단을 의미하는 좌표계를 가정하지만, 우리는 이미지를 Vulkan에 업로드할 때 `0`이 상단을 의미하는 위에서 아래 방향으로 업로드했습니다. 텍스처 좌표의 수직 성분을 뒤집어서 이 문제를 해결하세요:

```c++
vertex.texCoord = {
    attrib.texcoords[2 * index.texcoord_index + 0],
    1.0f - attrib.texcoords[2 * index.texcoord_index + 1]
};
```

프로그램을 다시 실행하면 이제 올바른 결과를 볼 수 있을 것입니다:

![](/images/drawing_model.png)

모든 노력이 마침내 이런 데모로 결실을 맺기 시작했습니다!

> 모델이 회전할 때 뒷면(벽의 뒷부분)이 다소 이상하게 보일 수 있습니다. 이는 정상이며, 모델이 원래 그쪽에서 보도록 설계되지 않았기 때문입니다.

## 정점 중복 제거

불행히도 아직 인덱스 버퍼를 제대로 활용하고 있지 않습니다. `vertices` 벡터에는 많은 중복된 정점 데이터가 포함되어 있는데, 이는 많은 정점이 여러 삼각형에 포함되기 때문입니다. 고유한 정점만 유지하고, 이들이 나타날 때마다 인덱스 버퍼를 사용해 재사용해야 합니다. 이를 구현하는 간단한 방법은 `map`이나 `unordered_map`을 사용하여 고유한 정점과 각각의 인덱스를 추적하는 것입니다:

```c++
#include <unordered_map>

...

std::unordered_map<Vertex, uint32_t> uniqueVertices{};

for (const auto& shape : shapes) {
    for (const auto& index : shape.mesh.indices) {
        Vertex vertex{};

        ...

        if (uniqueVertices.count(vertex) == 0) {
            uniqueVertices[vertex] = static_cast<uint32_t>(vertices.size());
            vertices.push_back(vertex);
        }

        indices.push_back(uniqueVertices[vertex]);
    }
}
```

OBJ 파일에서 정점을 읽을 때마다, 정확히 동일한 위치와 텍스처 좌표를 가진 정점을 이전에 본 적이 있는지 확인합니다. 만약 본 적이 없다면, `vertices`에 추가하고 그 인덱스를 `uniqueVertices` 컨테이너에 저장합니다. 그 후 새 정점의 인덱스를 `indices`에 추가합니다. 만약 이전에 정확히 동일한 정점을 본 적이 있다면, `uniqueVertices`에서 그 인덱스를 찾아 `indices`에 저장합니다.

사용자 정의 타입인 `Vertex` 구조체를 해시 테이블의 키로 사용하려면 두 가지 함수, 즉 동등성 검사와 해시 계산을 구현해야 하므로, 지금은 프로그램이 컴파일되지 않을 것입니다. 전자는 `Vertex` 구조체에서 `==` 연산자를 오버라이드하여 쉽게 구현할 수 있습니다:

```c++
bool operator==(const Vertex& other) const {
    return pos == other.pos && color == other.color && texCoord == other.texCoord;
}
```

`Vertex`에 대한 해시 함수는 `std::hash<T>`에 대한 템플릿 특수화를 지정하여 구현합니다. 해시 함수는 복잡한 주제이지만, [cppreference.com은](http://en.cppreference.com/w/cpp/utility/hash) 구조체의 필드들을 결합하여 괜찮은 품질의 해시 함수를 만드는 다음 접근 방식을 권장합니다:

```c++
namespace std {
    template<> struct hash<Vertex> {
        size_t operator()(Vertex const& vertex) const {
            return ((hash<glm::vec3>()(vertex.pos) ^
                   (hash<glm::vec3>()(vertex.color) << 1)) >> 1) ^
                   (hash<glm::vec2>()(vertex.texCoord) << 1);
        }
    };
}
```

이 코드는 `Vertex` 구조체 외부에 위치해야 합니다. GLM 타입에 대한 해시 함수는 다음 헤더를 사용하여 포함해야 합니다:

```c++
#define GLM_ENABLE_EXPERIMENTAL
#include <glm/gtx/hash.hpp>
```

해시 함수는 `gtx` 폴더에 정의되어 있는데, 이는 기술적으로는 아직 GLM의 실험적인 확장 기능이라는 것을 의미합니다. 따라서 이를 사용하려면 `GLM_ENABLE_EXPERIMENTAL`을 정의해야 합니다. 이는 향후 GLM의 새 버전에서 API가 변경될 수 있음을 의미하지만, 실제로는 API가 매우 안정적입니다.

이제 프로그램을 성공적으로 컴파일하고 실행할 수 있을 것입니다. `vertices`의 크기를 확인해보면 1,500,000개에서 265,645개로 줄어든 것을 확인할 수 있습니다! 이는 각 정점이 평균적으로 약 6개의 삼각형에서 재사용된다는 것을 의미합니다. 이로써 확실히 많은 GPU 메모리를 절약할 수 있습니다.

[C++ 코드](/code/28_model_loading.cpp) /
[정점 셰이더](/code/27_shader_depth.vert) /
[프래그먼트 셰이더](/code/27_shader_depth.frag)