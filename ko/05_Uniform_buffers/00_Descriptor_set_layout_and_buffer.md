## 서론

이제 우리는 각 정점마다 임의의 어트리뷰트를 버텍스 셰이더로 전달할 수 있게 되었습니다. 하지만 전역 변수는 어떨까요? 이번 장부터는 3D 그래픽스로 넘어가게 되는데, 여기에는 모델-뷰-프로젝션(MVP) 행렬이 필요합니다. 이 행렬을 정점 데이터에 포함시킬 수도 있겠지만, 이는 메모리 낭비이며 변환이 변경될 때마다 정점 버퍼를 업데이트해야 합니다. 변환은 매 프레임마다 쉽게 바뀔 수 있습니다.

Vulkan에서 이 문제를 해결하는 올바른 방법은 *리소스 디스크립터(resource descriptor)*를 사용하는 것입니다. 디스크립터는 셰이더가 버퍼나 이미지 같은 리소스에 자유롭게 접근할 수 있게 해주는 방법입니다. 우리는 변환 행렬들을 담고 있는 버퍼를 설정하고, 버텍스 셰이더가 디스크립터를 통해 이들에 접근하도록 할 것입니다. 디스크립터 사용은 세 부분으로 구성됩니다:

*   파이프라인 생성 시 디스크립터 셋 레이아웃 명시
*   디스크립터 풀에서 디스크립터 셋 할당
*   렌더링 시 디스크립터 셋 바인딩

*디스크립터 셋 레이아웃*은 렌더 패스가 접근할 어태치먼트의 타입을 명시하는 것과 유사하게, 파이프라인이 접근할 리소스의 타입을 명시합니다. *디스크립터 셋*은 프레임버퍼가 렌더 패스 어태치먼트에 바인딩할 실제 이미지 뷰를 지정하는 것과 유사하게, 디스크립터에 바인딩될 실제 버퍼나 이미지 리소스를 지정합니다. 그 후 디스크립터 셋은 정점 버퍼나 프레임버퍼처럼 드로우 커맨드를 위해 바인딩됩니다.

많은 종류의 디스크립터가 있지만, 이번 장에서는 uniform buffer object (UBO)를 다룰 것입니다. 다른 종류의 디스크립터는 다음 장들에서 살펴보겠지만, 기본적인 과정은 동일합니다. 버텍스 셰이더가 사용하길 원하는 데이터를 다음과 같은 C++ 구조체에 담는다고 가정해 봅시다:

```c++
struct UniformBufferObject {
    glm::mat4 model;
    glm::mat4 view;
    glm::mat4 proj;
};
```

그러면 우리는 이 데이터를 `VkBuffer`로 복사하고, 버텍스 셰이더에서 uniform buffer object 디스크립터를 통해 다음과 같이 접근할 수 있습니다:

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

다음 단계는 C++ 측에서 UBO를 정의하고, 버텍스 셰이더의 이 디스크립터에 대해 Vulkan에 알려주는 것입니다.

```c++
struct UniformBufferObject {
    glm::mat4 model;
    glm::mat4 view;
    glm::mat4 proj;
};
```

GLM의 데이터 타입을 사용하면 셰이더의 정의와 정확히 일치시킬 수 있습니다. 행렬의 데이터는 셰이더가 기대하는 방식과 바이너리 호환되므로, 나중에 `UniformBufferObject`를 `VkBuffer`에 `memcpy`하기만 하면 됩니다.

모든 정점 어트리뷰트와 그 `location` 인덱스에 대해 했던 것처럼, 파이프라인 생성을 위해 셰이더에서 사용되는 모든 디스크립터 바인딩에 대한 세부 정보를 제공해야 합니다. 이 모든 정보를 정의하기 위해 `createDescriptorSetLayout`이라는 새 함수를 설정할 것입니다. 이 함수는 파이프라인 생성 직전에 호출되어야 합니다. 왜냐하면 거기서 필요하기 때문입니다.

```c++
void initVulkan() {
    ...
    createDescriptorSetLayout();
    createGraphicsPipeline();
    ...
}

...

void createDescriptorSetLayout() {

}
```

모든 바인딩은 `VkDescriptorSetLayoutBinding` 구조체를 통해 기술되어야 합니다.

```c++
void createDescriptorSetLayout() {
    VkDescriptorSetLayoutBinding uboLayoutBinding{};
    uboLayoutBinding.binding = 0;
    uboLayoutBinding.descriptorType = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
    uboLayoutBinding.descriptorCount = 1;
}
```

첫 두 필드는 셰이더에서 사용되는 `binding`과 디스크립터의 타입, 즉 uniform buffer object를 명시합니다. 셰이더 변수가 uniform buffer object의 배열을 나타내는 것도 가능하며, `descriptorCount`는 배열에 있는 값의 수를 지정합니다. 예를 들어, 이는 스켈레탈 애니메이션에서 각 뼈에 대한 변환을 지정하는 데 사용될 수 있습니다. 우리의 MVP 변환은 단일 uniform buffer object에 있으므로, `descriptorCount`는 `1`을 사용합니다.

```c++
uboLayoutBinding.stageFlags = VK_SHADER_STAGE_VERTEX_BIT;
```

우리는 또한 디스크립터가 어느 셰이더 단계에서 참조될 것인지 명시해야 합니다. `stageFlags` 필드는 `VkShaderStageFlagBits` 값들의 조합이거나 `VK_SHADER_STAGE_ALL_GRAPHICS` 값일 수 있습니다. 우리의 경우, 버텍스 셰이더에서만 디스크립터를 참조합니다.

```c++
uboLayoutBinding.pImmutableSamplers = nullptr; // Optional
```

`pImmutableSamplers` 필드는 이미지 샘플링 관련 디스크립터에만 관련이 있으며, 이는 나중에 살펴볼 것입니다. 이 값은 기본값으로 남겨둘 수 있습니다.

모든 디스크립터 바인딩은 단일 `VkDescriptorSetLayout` 객체로 결합됩니다. `pipelineLayout` 위에 새로운 클래스 멤버를 정의하세요:

```c++
VkDescriptorSetLayout descriptorSetLayout;
VkPipelineLayout pipelineLayout;
```

그런 다음 `vkCreateDescriptorSetLayout`을 사용하여 이를 생성할 수 있습니다. 이 함수는 바인딩 배열을 포함하는 간단한 `VkDescriptorSetLayoutCreateInfo`를 받습니다:

```c++
VkDescriptorSetLayoutCreateInfo layoutInfo{};
layoutInfo.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO;
layoutInfo.bindingCount = 1;
layoutInfo.pBindings = &uboLayoutBinding;

if (vkCreateDescriptorSetLayout(device, &layoutInfo, nullptr, &descriptorSetLayout) != VK_SUCCESS) {
    throw std::runtime_error("failed to create descriptor set layout!");
}
```

셰이더가 어떤 디스크립터를 사용할지 Vulkan에 알리기 위해 파이프라인 생성 중에 디스크립터 셋 레이아웃을 지정해야 합니다. 디스크립터 셋 레이아웃은 파이프라인 레이아웃 객체에 지정됩니다. 레이아웃 객체를 참조하도록 `VkPipelineLayoutCreateInfo`를 수정하세요:

```c++
VkPipelineLayoutCreateInfo pipelineLayoutInfo{};
pipelineLayoutInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_LAYOUT_CREATE_INFO;
pipelineLayoutInfo.setLayoutCount = 1;
pipelineLayoutInfo.pSetLayouts = &descriptorSetLayout;
```

왜 하나의 레이아웃에 모든 바인딩이 포함되어 있는데도 여러 디스크립터 셋 레이아웃을 지정할 수 있는지 궁금할 수 있습니다. 이에 대해서는 다음 장에서 디스크립터 풀과 디스크립터 셋을 다룰 때 다시 돌아올 것입니다.

디스크립터 셋 레이아웃은 우리가 새로운 그래픽스 파이프라인을 생성할 수 있는 동안, 즉 프로그램이 끝날 때까지 유지되어야 합니다:

```c++
void cleanup() {
    cleanupSwapChain();

    vkDestroyDescriptorSetLayout(device, descriptorSetLayout, nullptr);

    ...
}
```

## Uniform 버퍼

다음 장에서는 셰이더를 위한 UBO 데이터를 담고 있는 버퍼를 명시할 것이지만, 먼저 이 버퍼를 생성해야 합니다. 우리는 매 프레임마다 uniform 버퍼에 새로운 데이터를 복사할 것이므로, 스테이징 버퍼를 사용하는 것은 별 의미가 없습니다. 이 경우 추가적인 오버헤드만 발생시키고 성능을 향상시키기보다는 저하시킬 가능성이 높습니다.

우리는 여러 개의 버퍼를 가져야 합니다. 왜냐하면 여러 프레임이 동시에 처리 중(in flight)일 수 있고, 이전 프레임이 여전히 버퍼에서 읽고 있는 동안 다음 프레임을 준비하기 위해 버퍼를 업데이트하고 싶지 않기 때문입니다! 따라서, 우리는 동시에 처리 중인 프레임 수만큼 uniform 버퍼를 가져야 하며, 현재 GPU가 읽고 있지 않은 uniform 버퍼에 써야 합니다.

이를 위해 `uniformBuffers`와 `uniformBuffersMemory`, `uniformBuffersMapped`를 위한 새로운 클래스 멤버를 추가하세요:

```c++
VkBuffer indexBuffer;
VkDeviceMemory indexBufferMemory;

std::vector<VkBuffer> uniformBuffers;
std::vector<VkDeviceMemory> uniformBuffersMemory;
std::vector<void*> uniformBuffersMapped;
```

유사하게, `createIndexBuffer` 다음에 호출되어 버퍼들을 할당하는 `createUniformBuffers`라는 새 함수를 생성하세요:

```c++
void initVulkan() {
    ...
    createVertexBuffer();
    createIndexBuffer();
    createUniformBuffers();
    ...
}

...

void createUniformBuffers() {
    VkDeviceSize bufferSize = sizeof(UniformBufferObject);

    uniformBuffers.resize(MAX_FRAMES_IN_FLIGHT);
    uniformBuffersMemory.resize(MAX_FRAMES_IN_FLIGHT);
    uniformBuffersMapped.resize(MAX_FRAMES_IN_FLIGHT);

    for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHT; i++) {
        createBuffer(bufferSize, VK_BUFFER_USAGE_UNIFORM_BUFFER_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, uniformBuffers[i], uniformBuffersMemory[i]);

        vkMapMemory(device, uniformBuffersMemory[i], 0, bufferSize, 0, &uniformBuffersMapped[i]);
    }
}
```

`vkMapMemory`를 사용하여 버퍼를 생성한 직후 매핑하여, 나중에 데이터를 쓸 수 있는 포인터를 얻습니다. 버퍼는 애플리케이션의 전체 수명 동안 이 포인터에 매핑된 상태로 유지됩니다. 이 기법은 **"영구적 매핑(persistent mapping)"**이라고 불리며 모든 Vulkan 구현에서 작동합니다. 업데이트가 필요할 때마다 버퍼를 매핑할 필요가 없으므로 성능이 향상됩니다. 매핑은 공짜가 아니기 때문입니다.

uniform 데이터는 모든 드로우 콜에 사용되므로, 이를 담고 있는 버퍼는 렌더링을 멈출 때만 파괴되어야 합니다.

```c++
void cleanup() {
    ...

    for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHT; i++) {
        vkDestroyBuffer(device, uniformBuffers[i], nullptr);
        vkFreeMemory(device, uniformBuffersMemory[i], nullptr);
    }

    vkDestroyDescriptorSetLayout(device, descriptorSetLayout, nullptr);

    ...

}
```

## Uniform 데이터 업데이트

`updateUniformBuffer`라는 새 함수를 만들고, 다음 프레임을 제출하기 전에 `drawFrame` 함수에서 이 함수를 호출하도록 추가하세요:

```c++
void drawFrame() {
    ...

    updateUniformBuffer(currentFrame);

    ...

    VkSubmitInfo submitInfo{};
    submitInfo.sType = VK_STRUCTURE_TYPE_SUBMIT_INFO;

    ...
}

...

void updateUniformBuffer(uint32_t currentImage) {

}
```

이 함수는 지오메트리가 회전하도록 매 프레임 새로운 변환을 생성할 것입니다. 이 기능을 구현하기 위해 두 개의 새로운 헤더를 포함해야 합니다:

```c++
#define GLM_FORCE_RADIANS
#include <glm/glm.hpp>
#include <glm/gtc/matrix_transform.hpp>

#include <chrono>
```

`glm/gtc/matrix_transform.hpp` 헤더는 `glm::rotate`와 같은 모델 변환, `glm::lookAt`과 같은 뷰 변환, `glm::perspective`와 같은 프로젝션 변환을 생성하는 데 사용할 수 있는 함수들을 제공합니다. `GLM_FORCE_RADIANS` 정의는 `glm::rotate`와 같은 함수들이 라디안을 인자로 사용하도록 하여 혼동의 여지를 없애는 데 필요합니다.

`chrono` 표준 라이브러리 헤더는 정밀한 시간 측정을 위한 함수들을 제공합니다. 우리는 이를 사용하여 프레임 속도에 관계없이 지오메트리가 초당 90도 회전하도록 할 것입니다.

```c++
void updateUniformBuffer(uint32_t currentImage) {
    static auto startTime = std::chrono::high_resolution_clock::now();

    auto currentTime = std::chrono::high_resolution_clock::now();
    float time = std::chrono::duration<float, std::chrono::seconds::period>(currentTime - startTime).count();
}
```

`updateUniformBuffer` 함수는 렌더링 시작 후 경과 시간을 부동소수점 정밀도의 초 단위로 계산하는 로직으로 시작합니다.

이제 uniform buffer object에 모델, 뷰, 프로젝션 변환을 정의하겠습니다. 모델 회전은 `time` 변수를 사용하여 Z축을 중심으로 한 단순한 회전이 될 것입니다:

```c++
UniformBufferObject ubo{};
ubo.model = glm::rotate(glm::mat4(1.0f), time * glm::radians(90.0f), glm::vec3(0.0f, 0.0f, 1.0f));
```

`glm::rotate` 함수는 기존 변환, 회전 각도, 회전 축을 매개변수로 받습니다. `glm::mat4(1.0f)` 생성자는 단위 행렬(identity matrix)을 반환합니다. `time * glm::radians(90.0f)`의 회전 각도를 사용하면 초당 90도 회전하는 목적을 달성합니다.

```c++
ubo.view = glm::lookAt(glm::vec3(2.0f, 2.0f, 2.0f), glm::vec3(0.0f, 0.0f, 0.0f), glm::vec3(0.0f, 0.0f, 1.0f));
```

뷰 변환의 경우, 45도 각도에서 위에서 지오메트리를 바라보도록 결정했습니다. `glm::lookAt` 함수는 눈의 위치, 바라보는 중심점, 위쪽 축을 매개변수로 받습니다.

```c++
ubo.proj = glm::perspective(glm::radians(45.0f), swapChainExtent.width / (float) swapChainExtent.height, 0.1f, 10.0f);
```

45도의 수직 시야각을 가진 원근 투영을 사용하기로 결정했습니다. 다른 매개변수는 종횡비(aspect ratio), 근평면(near plane), 원평면(far plane)입니다. 창 크기 조정 후의 새로운 너비와 높이를 고려하여 종횡비를 계산하기 위해 현재 스왑 체인의 크기(`swapChainExtent`)를 사용하는 것이 중요합니다.

```c++
ubo.proj[1][1] *= -1;
```

GLM은 원래 Y 좌표가 반전된 OpenGL을 위해 설계되었습니다. 이를 보정하는 가장 쉬운 방법은 투영 행렬에서 Y축의 스케일링 팩터 부호를 뒤집는 것입니다. 이렇게 하지 않으면 이미지가 거꾸로 렌더링됩니다.

이제 모든 변환이 정의되었으므로, uniform buffer object의 데이터를 현재 uniform 버퍼로 복사할 수 있습니다. 이것은 스테이징 버퍼 없이 정점 버퍼에 대해 했던 것과 정확히 같은 방식으로 일어납니다. 앞서 언급했듯이, uniform 버퍼는 한 번만 매핑하므로, 다시 매핑할 필요 없이 직접 쓸 수 있습니다:

```c++
memcpy(uniformBuffersMapped[currentImage], &ubo, sizeof(ubo));
```

이런 식으로 UBO를 사용하는 것은 자주 변경되는 값을 셰이더에 전달하는 가장 효율적인 방법은 아닙니다. 작은 데이터 버퍼를 셰이더에 전달하는 더 효율적인 방법은 *푸시 상수(push constants)*입니다. 이에 대해서는 향후 챕터에서 살펴볼 수 있습니다.

다음 장에서는 디스크립터 셋에 대해 살펴보고, 실제로 `VkBuffer`들을 uniform 버퍼 디스크립터에 바인딩하여 셰이더가 이 변환 데이터에 접근할 수 있도록 할 것입니다.

[C++ 코드](/code/22_descriptor_set_layout.cpp) /
[버텍스 셰이더](/code/22_shader_ubo.vert) /
[프래그먼트 셰이더](/code/22_shader_ubo.frag)