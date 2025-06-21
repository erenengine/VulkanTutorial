## 소개

이 보너스 챕터에서는 컴퓨트 셰이더(compute shader)에 대해 살펴보겠습니다. 지금까지의 모든 챕터는 Vulkan 파이프라인의 전통적인 그래픽스 부분을 다루었습니다. 하지만 OpenGL과 같은 오래된 API와 달리, Vulkan에서 컴퓨트 셰이더 지원은 필수입니다. 이는 고사양 데스크톱 GPU든 저전력 임베디드 장치든, 사용 가능한 모든 Vulkan 구현에서 컴퓨트 셰이더를 사용할 수 있다는 의미입니다.

이는 여러분의 애플리케이션이 어디서 실행되든 상관없이 GPU(그래픽 처리 장치)를 이용한 범용 컴퓨팅(GPGPU, general purpose computing on graphics processor units)의 세계를 열어줍니다. GPGPU는 전통적으로 CPU의 영역이었던 일반적인 계산을 GPU에서 수행할 수 있음을 의미합니다. GPU가 점점 더 강력해지고 유연해짐에 따라, CPU의 범용적인 능력이 필요했던 많은 작업들을 이제 GPU에서 실시간으로 처리할 수 있게 되었습니다.

GPU의 컴퓨팅 능력이 사용될 수 있는 몇 가지 예로는 이미지 처리, 가시성 테스트, 후처리(post processing), 고급 조명 계산, 애니메이션, 물리(예: 파티클 시스템) 등이 있으며, 이 외에도 훨씬 더 많습니다. 심지어 수치 연산이나 AI 관련 작업처럼 그래픽 출력이 전혀 필요 없는 비시각적 계산 전용 작업에도 컴퓨트를 사용할 수 있습니다. 이를 "헤드리스 컴퓨트(headless compute)"라고 합니다.

## 장점

계산 비용이 많이 드는 작업을 GPU에서 수행하면 몇 가지 장점이 있습니다. 가장 명백한 것은 CPU의 작업을 덜어내는 것입니다. 또 다른 장점은 CPU의 주 메모리와 GPU 메모리 간에 데이터를 옮길 필요가 없다는 점입니다. 모든 데이터는 주 메모리로부터의 느린 전송을 기다릴 필요 없이 GPU에 머무를 수 있습니다.

이 외에도 GPU는 수만 개의 작은 연산 유닛으로 고도로 병렬화되어 있습니다. 이 때문에 몇 개의 큰 연산 유닛을 가진 CPU보다 고도로 병렬화된 워크플로우에 더 적합한 경우가 많습니다.

## Vulkan 파이프라인

컴퓨트는 파이프라인의 그래픽스 부분과 완전히 분리되어 있다는 점을 알아두는 것이 중요합니다. 이는 공식 명세서의 다음 Vulkan 파이프라인 블록 다이어그램에서 확인할 수 있습니다.

![](/images/vulkan_pipeline_block_diagram.png)

이 다이어그램의 왼쪽에는 전통적인 그래픽스 파이프라인 부분이 있고, 오른쪽에는 컴퓨트 셰이더 단계를 포함하여 이 그래픽스 파이프라인에 속하지 않는 여러 단계들이 있습니다. 컴퓨트 셰이더 단계가 그래픽스 파이프라인에서 분리되어 있으므로, 우리는 필요하다고 생각되는 어느 곳에서든 이를 사용할 수 있습니다. 이는 항상 정점 셰이더의 변환된 출력에 적용되는 프래그먼트 셰이더와는 매우 다릅니다.

다이어그램 중앙은 디스크립터 셋(descriptor set)과 같은 요소들이 컴퓨트에서도 사용된다는 것을 보여주므로, 우리가 디스크립터 레이아웃, 디스크립터 셋, 디스크립터에 대해 배운 모든 것이 여기에도 적용됩니다.

## 예제

이 챕터에서 구현할 이해하기 쉬운 예제는 GPU 기반 파티클 시스템입니다. 이러한 시스템은 많은 게임에서 사용되며, 종종 상호작용 가능한 프레임 속도로 업데이트되어야 하는 수천 개의 파티클로 구성됩니다. 이러한 시스템을 렌더링하려면 두 가지 주요 구성 요소가 필요합니다: 정점 버퍼로 전달되는 정점들과, 어떤 방정식에 기반하여 이들을 업데이트하는 방법입니다.

"전통적인" CPU 기반 파티클 시스템은 파티클 데이터를 시스템의 주 메모리에 저장한 다음 CPU를 사용하여 업데이트합니다. 업데이트 후에는 다음 프레임에서 업데이트된 파티클을 표시할 수 있도록 정점 데이터를 다시 GPU 메모리로 전송해야 합니다. 가장 간단한 방법은 매 프레임마다 새로운 데이터로 정점 버퍼를 다시 생성하는 것입니다. 이는 명백히 비용이 많이 듭니다. 구현에 따라, CPU가 쓸 수 있도록 GPU 메모리를 매핑하거나("데스크톱 시스템에서는 resizable BAR", 통합 GPU에서는 통합 메모리라고 함), 호스트 로컬 버퍼를 사용하는(PCI-E 대역폭 때문에 가장 느린 방법) 등의 다른 옵션이 있습니다. 하지만 어떤 버퍼 업데이트 방법을 선택하든, 파티클을 업데이트하기 위해 항상 CPU를 거쳐야 하는 과정이 필요합니다.

GPU 기반 파티클 시스템을 사용하면 이러한 과정이 더 이상 필요하지 않습니다. 정점 데이터는 처음에만 GPU에 업로드되며, 모든 업데이트는 컴퓨트 셰이더를 사용하여 GPU 메모리 내에서 이루어집니다. 이것이 더 빠른 주된 이유 중 하나는 GPU와 로컬 메모리 간의 훨씬 높은 대역폭 때문입니다. CPU 기반 시나리오에서는 주 메모리와 PCI-Express 대역폭에 의해 제한되는데, 이는 종종 GPU 메모리 대역폭의 일부에 불과합니다.

전용 컴퓨트 큐가 있는 GPU에서 이 작업을 수행하면 그래픽스 파이프라인의 렌더링 부분과 병렬로 파티클을 업데이트할 수 있습니다. 이를 "비동기 컴퓨트(async compute)"라고 하며, 이 튜토리얼에서는 다루지 않는 고급 주제입니다.

다음은 이 챕터 코드의 스크린샷입니다. 여기에 보이는 파티클은 CPU 상호작용 없이 GPU에서 직접 컴퓨트 셰이더에 의해 업데이트됩니다.

![](/images/compute_shader_particles.png)

## 데이터 조작

이 튜토리얼에서 우리는 프리미티브를 전달하기 위한 정점 및 인덱스 버퍼, 셰이더에 데이터를 전달하기 위한 유니폼 버퍼와 같은 다양한 버퍼 유형에 대해 이미 배웠습니다. 그리고 텍스처 매핑을 위해 이미지를 사용하기도 했습니다. 하지만 지금까지는 항상 CPU를 사용하여 데이터를 쓰고 GPU에서는 읽기만 했습니다.

컴퓨트 셰이더와 함께 도입된 중요한 개념은 버퍼에 **읽고 쓰는 것**을 자유롭게 할 수 있다는 점입니다. 이를 위해 Vulkan은 두 가지 전용 저장소 유형을 제공합니다.

### 셰이더 저장 버퍼 객체 (SSBO)

셰이더 저장 버퍼(SSBO, Shader Storage Buffer Object)는 셰이더가 버퍼에서 읽고 쓸 수 있게 해줍니다. 이를 사용하는 것은 유니폼 버퍼 객체를 사용하는 것과 유사합니다. 가장 큰 차이점은 다른 버퍼 유형을 SSBO로 사용할 수 있으며, 크기에 제한이 없다는 것입니다.

GPU 기반 파티클 시스템으로 돌아가서, 컴퓨트 셰이더에 의해 업데이트(쓰기)되고 정점 셰이더에 의해 읽히는(그리기) 정점을 어떻게 처리해야 할지 궁금할 수 있습니다. 두 사용 사례가 서로 다른 버퍼 유형을 필요로 하는 것처럼 보이기 때문입니다.

하지만 그렇지 않습니다. Vulkan에서는 버퍼와 이미지에 대해 여러 사용 용도를 지정할 수 있습니다. 따라서 파티클 정점 버퍼를 정점 버퍼(그래픽스 패스에서)와 저장 버퍼(컴퓨트 패스에서)로 사용하려면, 해당 두 사용 플래그로 버퍼를 생성하기만 하면 됩니다.

```c++
VkBufferCreateInfo bufferInfo{};
...
bufferInfo.usage = VK_BUFFER_USAGE_VERTEX_BUFFER_BIT | VK_BUFFER_USAGE_STORAGE_BUFFER_BIT | VK_BUFFER_USAGE_TRANSFER_DST_BIT;
...

if (vkCreateBuffer(device, &bufferInfo, nullptr, &shaderStorageBuffers[i]) != VK_SUCCESS) {
    throw std::runtime_error("failed to create vertex buffer!");
}
```
`bufferInfo.usage`에 설정된 `VK_BUFFER_USAGE_VERTEX_BUFFER_BIT`와 `VK_BUFFER_USAGE_STORAGE_BUFFER_BIT` 두 플래그는 구현에 이 버퍼를 두 가지 다른 시나리오, 즉 정점 셰이더의 정점 버퍼와 저장 버퍼로 사용하고 싶다는 것을 알립니다. 또한 호스트에서 GPU로 데이터를 전송할 수 있도록 `VK_BUFFER_USAGE_TRANSFER_DST_BIT` 플래그도 추가했습니다. 셰이더 저장 버퍼를 GPU 메모리에만 유지하기를 원하므로(`VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT`), 호스트에서 이 버퍼로 데이터를 전송해야 하기 때문에 이는 매우 중요합니다.

다음은 `createBuffer` 헬퍼 함수를 사용한 동일한 코드입니다.

```c++
createBuffer(bufferSize, VK_BUFFER_USAGE_STORAGE_BUFFER_BIT | VK_BUFFER_USAGE_VERTEX_BUFFER_BIT | VK_BUFFER_USAGE_TRANSFER_DST_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, shaderStorageBuffers[i], shaderStorageBuffersMemory[i]);
```

이러한 버퍼에 접근하기 위한 GLSL 셰이더 선언은 다음과 같습니다.

```glsl
struct Particle {
  vec2 position;
  vec2 velocity;
  vec4 color;
};

layout(std140, binding = 1) readonly buffer ParticleSSBOIn {
   Particle particlesIn[ ];
};

layout(std140, binding = 2) buffer ParticleSSBOOut {
   Particle particlesOut[ ];
};
```

이 예제에는 각 파티클이 위치와 속도 값을 가지는 타입이 지정된 SSBO가 있습니다(`Particle` 구조체 참조). 그리고 SSBO는 `[]`로 표시된 것처럼 바인딩되지 않은 수의 파티클을 포함합니다. SSBO에서 요소의 수를 지정할 필요가 없다는 것은 유니폼 버퍼 등에 비해 장점 중 하나입니다. `std140`은 셰이더 저장 버퍼의 멤버 요소가 메모리에서 어떻게 정렬되는지를 결정하는 메모리 레이아웃 한정자입니다. 이는 호스트와 GPU 간에 버퍼를 매핑하는 데 필요한 특정 보장을 제공합니다.

컴퓨트 셰이더에서 이러한 저장 버퍼 객체에 쓰는 것은 간단하며, C++ 측에서 버퍼에 쓰는 방식과 유사합니다.

```glsl
particlesOut[index].position = particlesIn[index].position + particlesIn[index].velocity.xy * ubo.deltaTime;
```

### 저장 이미지

*이 챕터에서는 이미지 조작을 다루지 않습니다. 이 단락은 독자들에게 컴퓨트 셰이더가 이미지 조작에도 사용될 수 있음을 알리기 위해 존재합니다.*

저장 이미지(storage image)는 이미지에서 읽고 쓸 수 있게 해줍니다. 일반적인 사용 사례는 텍스처에 이미지 효과를 적용하거나, 후처리를 하거나(매우 유사함), 밉맵을 생성하는 것입니다.

이미지의 경우도 비슷합니다.

```c++
VkImageCreateInfo imageInfo {};
...
imageInfo.usage = VK_IMAGE_USAGE_SAMPLED_BIT | VK_IMAGE_USAGE_STORAGE_BIT;
...

if (vkCreateImage(device, &imageInfo, nullptr, &textureImage) != VK_SUCCESS) {
    throw std::runtime_error("failed to create image!");
}
```

`imageInfo.usage`에 설정된 `VK_IMAGE_USAGE_SAMPLED_BIT`와 `VK_IMAGE_USAGE_STORAGE_BIT` 두 플래그는 이 이미지를 두 가지 다른 시나리오, 즉 프래그먼트 셰이더에서 샘플링되는 이미지와 컴퓨트 셰이더의 저장 이미지로 사용하고 싶다는 것을 구현에 알립니다.

저장 이미지에 대한 GLSL 셰이더 선언은 프래그먼트 셰이더 등에서 사용되는 샘플링된 이미지와 유사합니다.

```glsl
layout (binding = 0, rgba8) uniform readonly image2D inputImage;
layout (binding = 1, rgba8) uniform writeonly image2D outputImage;
```

여기서 몇 가지 차이점은 이미지의 형식을 위한 `rgba8`과 같은 추가 속성, 입력 이미지에서는 읽기만 하고 출력 이미지에는 쓰기만 할 것임을 구현에 알리는 `readonly` 및 `writeonly` 한정자입니다. 그리고 마지막으로 저장 이미지를 선언하기 위해 `image2D` 타입을 사용해야 합니다.

컴퓨트 셰이더에서 저장 이미지에 읽고 쓰는 것은 `imageLoad`와 `imageStore`를 사용하여 수행됩니다.

```glsl
vec3 pixel = imageLoad(inputImage, ivec2(gl_GlobalInvocationID.xy)).rgb;
imageStore(outputImage, ivec2(gl_GlobalInvocationID.xy), pixel);
```

## 컴퓨트 큐 패밀리

[물리 장치 및 큐 패밀리 챕터](03_Drawing_a_triangle/00_Setup/03_Physical_devices_and_queue_families.md#page_Queue-families)에서 우리는 이미 큐 패밀리와 그래픽스 큐 패밀리를 선택하는 방법에 대해 배웠습니다. 컴퓨트는 큐 패밀리 속성 플래그 비트 `VK_QUEUE_COMPUTE_BIT`를 사용합니다. 따라서 컴퓨트 작업을 하려면 컴퓨트를 지원하는 큐 패밀리에서 큐를 가져와야 합니다.

Vulkan은 그래픽스 작업을 지원하는 구현이 그래픽스와 컴퓨트 작업을 모두 지원하는 큐 패밀리를 최소 하나 이상 가지도록 요구하지만, 구현이 전용 컴퓨트 큐를 제공할 수도 있다는 점에 유의해야 합니다. 이 전용 컴퓨트 큐(그래픽스 비트가 없는)는 비동기 컴퓨트 큐를 암시합니다. 하지만 이 튜토리얼은 초심자에게 친숙하도록 그래픽스와 컴퓨트 작업을 모두 할 수 있는 큐를 사용할 것입니다. 이는 또한 여러 고급 동기화 메커니즘을 다루는 것을 피하게 해줍니다.

컴퓨트 샘플을 위해 장치 생성 코드를 약간 변경해야 합니다.

```c++
uint32_t queueFamilyCount = 0;
vkGetPhysicalDeviceQueueFamilyProperties(device, &queueFamilyCount, nullptr);

std::vector<VkQueueFamilyProperties> queueFamilies(queueFamilyCount);
vkGetPhysicalDeviceQueueFamilyProperties(device, &queueFamilyCount, queueFamilies.data());

int i = 0;
for (const auto& queueFamily : queueFamilies) {
    if ((queueFamily.queueFlags & VK_QUEUE_GRAPHICS_BIT) && (queueFamily.queueFlags & VK_QUEUE_COMPUTE_BIT)) {
        indices.graphicsAndComputeFamily = i;
    }

    i++;
}
```

변경된 큐 패밀리 인덱스 선택 코드는 이제 그래픽스와 컴퓨트를 모두 지원하는 큐 패밀리를 찾으려고 시도할 것입니다.

그런 다음 `createLogicalDevice`에서 이 큐 패밀리로부터 컴퓨트 큐를 가져올 수 있습니다.

```c++
vkGetDeviceQueue(device, indices.graphicsAndComputeFamily.value(), 0, &computeQueue);
```

## 컴퓨트 셰이더 단계

그래픽스 샘플에서는 셰이더를 로드하고 디스크립터에 접근하기 위해 다른 파이프라인 단계를 사용했습니다. 컴퓨트 셰이더는 `VK_SHADER_STAGE_COMPUTE_BIT` 파이프라인을 사용하여 비슷한 방식으로 접근됩니다. 따라서 컴퓨트 셰이더를 로드하는 것은 정점 셰이더를 로드하는 것과 동일하지만 셰이더 단계가 다릅니다. 이에 대해서는 다음 단락에서 자세히 다룰 것입니다. 컴퓨트는 또한 나중에 사용해야 할 `VK_PIPELINE_BIND_POINT_COMPUTE`라는 새로운 디스크립터 및 파이프라인 바인딩 포인트 유형을 도입합니다.

## 컴퓨트 셰이더 로드하기

애플리케이션에서 컴퓨트 셰이더를 로드하는 것은 다른 셰이더를 로드하는 것과 동일합니다. 유일한 실제 차이점은 위에서 언급한 `VK_SHADER_STAGE_COMPUTE_BIT`를 사용해야 한다는 것입니다.

```c++
auto computeShaderCode = readFile("shaders/compute.spv");

VkShaderModule computeShaderModule = createShaderModule(computeShaderCode);

VkPipelineShaderStageCreateInfo computeShaderStageInfo{};
computeShaderStageInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO;
computeShaderStageInfo.stage = VK_SHADER_STAGE_COMPUTE_BIT;
computeShaderStageInfo.module = computeShaderModule;
computeShaderStageInfo.pName = "main";
...
```

## 셰이더 저장 버퍼 준비하기

앞서 우리는 임의의 데이터를 컴퓨트 셰이더에 전달하기 위해 셰이더 저장 버퍼를 사용할 수 있다고 배웠습니다. 이 예제에서는 파티클 배열을 GPU에 업로드하여 GPU 메모리에서 직접 조작할 수 있도록 할 것입니다.

[Frames in flight](03_Drawing_a_triangle/03_Drawing/03_Frames_in_flight.md) 챕터에서 우리는 CPU와 GPU를 계속 바쁘게 유지하기 위해 프레임별로 리소스를 복제하는 것에 대해 이야기했습니다. 먼저 버퍼 객체와 이를 지원하는 장치 메모리를 위한 벡터를 선언합니다.

```c++
std::vector<VkBuffer> shaderStorageBuffers;
std::vector<VkDeviceMemory> shaderStorageBuffersMemory;
```

`createShaderStorageBuffers`에서 이 벡터들의 크기를 최대 프레임 수에 맞게 조정합니다.

```c++
shaderStorageBuffers.resize(MAX_FRAMES_IN_FLIGHT);
shaderStorageBuffersMemory.resize(MAX_FRAMES_IN_FLIGHT);
```

이 설정이 완료되면 초기 파티클 정보를 GPU로 옮기기 시작할 수 있습니다. 먼저 호스트 측에서 파티클 벡터를 초기화합니다.

```c++
    // 파티클 초기화
    std::default_random_engine rndEngine((unsigned)time(nullptr));
    std::uniform_real_distribution<float> rndDist(0.0f, 1.0f);

    // 원 위에 초기 파티클 위치 지정
    std::vector<Particle> particles(PARTICLE_COUNT);
    for (auto& particle : particles) {
        float r = 0.25f * sqrt(rndDist(rndEngine));
        float theta = rndDist(rndEngine) * 2 * 3.14159265358979323846;
        float x = r * cos(theta) * HEIGHT / WIDTH;
        float y = r * sin(theta);
        particle.position = glm::vec2(x, y);
        particle.velocity = glm::normalize(glm::vec2(x,y)) * 0.00025f;
        particle.color = glm::vec4(rndDist(rndEngine), rndDist(rndEngine), rndDist(rndEngine), 1.0f);
    }

```

그런 다음 초기 파티클 속성을 담을 호스트 메모리에 [스테이징 버퍼](04_Vertex_buffers/02_Staging_buffer.md)를 생성합니다.

```c++
    VkDeviceSize bufferSize = sizeof(Particle) * PARTICLE_COUNT;

    VkBuffer stagingBuffer;
    VkDeviceMemory stagingBufferMemory;
    createBuffer(bufferSize, VK_BUFFER_USAGE_TRANSFER_SRC_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, stagingBuffer, stagingBufferMemory);

    void* data;
    vkMapMemory(device, stagingBufferMemory, 0, bufferSize, 0, &data);
    memcpy(data, particles.data(), (size_t)bufferSize);
    vkUnmapMemory(device, stagingBufferMemory);
```    

이 스테이징 버퍼를 소스로 사용하여 프레임별 셰이더 저장 버퍼를 생성하고 파티클 속성을 스테이징 버퍼에서 각각으로 복사합니다.

```c++
    for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHT; i++) {
        createBuffer(bufferSize, VK_BUFFER_USAGE_STORAGE_BUFFER_BIT | VK_BUFFER_USAGE_VERTEX_BUFFER_BIT | VK_BUFFER_USAGE_TRANSFER_DST_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, shaderStorageBuffers[i], shaderStorageBuffersMemory[i]);
        // 스테이징 버퍼(호스트)에서 셰이더 저장 버퍼(GPU)로 데이터 복사
        copyBuffer(stagingBuffer, shaderStorageBuffers[i], bufferSize);
    }
}
```

## 디스크립터

컴퓨트를 위한 디스크립터 설정은 그래픽스와 거의 동일합니다. 유일한 차이점은 디스크립터가 컴퓨트 단계에서 접근할 수 있도록 `VK_SHADER_STAGE_COMPUTE_BIT`가 설정되어야 한다는 것입니다.

```c++
std::array<VkDescriptorSetLayoutBinding, 3> layoutBindings{};
layoutBindings[0].binding = 0;
layoutBindings[0].descriptorCount = 1;
layoutBindings[0].descriptorType = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
layoutBindings[0].pImmutableSamplers = nullptr;
layoutBindings[0].stageFlags = VK_SHADER_STAGE_COMPUTE_BIT;
...
```

여기서 셰이더 단계를 결합할 수 있습니다. 예를 들어, 정점 및 컴퓨트 단계에서 접근 가능한 디스크립터(예: 이들 간에 공유되는 매개변수가 있는 유니폼 버퍼)를 원한다면, 두 단계에 대한 비트를 모두 설정하면 됩니다.

```c++
layoutBindings[0].stageFlags = VK_SHADER_STAGE_VERTEX_BIT | VK_SHADER_STAGE_COMPUTE_BIT;
```

다음은 우리 샘플의 디스크립터 설정입니다. 레이아웃은 다음과 같습니다.

```c++
std::array<VkDescriptorSetLayoutBinding, 3> layoutBindings{};
layoutBindings[0].binding = 0;
layoutBindings[0].descriptorCount = 1;
layoutBindings[0].descriptorType = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
layoutBindings[0].pImmutableSamplers = nullptr;
layoutBindings[0].stageFlags = VK_SHADER_STAGE_COMPUTE_BIT;

layoutBindings[1].binding = 1;
layoutBindings[1].descriptorCount = 1;
layoutBindings[1].descriptorType = VK_DESCRIPTOR_TYPE_STORAGE_BUFFER;
layoutBindings[1].pImmutableSamplers = nullptr;
layoutBindings[1].stageFlags = VK_SHADER_STAGE_COMPUTE_BIT;

layoutBindings[2].binding = 2;
layoutBindings[2].descriptorCount = 1;
layoutBindings[2].descriptorType = VK_DESCRIPTOR_TYPE_STORAGE_BUFFER;
layoutBindings[2].pImmutableSamplers = nullptr;
layoutBindings[2].stageFlags = VK_SHADER_STAGE_COMPUTE_BIT;

VkDescriptorSetLayoutCreateInfo layoutInfo{};
layoutInfo.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO;
layoutInfo.bindingCount = 3;
layoutInfo.pBindings = layoutBindings.data();

if (vkCreateDescriptorSetLayout(device, &layoutInfo, nullptr, &computeDescriptorSetLayout) != VK_SUCCESS) {
    throw std::runtime_error("failed to create compute descriptor set layout!");
}
```

이 설정을 보면, 단일 파티클 시스템만 렌더링하는데도 왜 셰이더 저장 버퍼 객체에 대한 레이아웃 바인딩이 두 개나 있는지 궁금할 수 있습니다. 이는 파티클 위치가 델타 타임에 따라 프레임별로 업데이트되기 때문입니다. 즉, 각 프레임은 이전 프레임의 파티클 위치를 알아야 새로운 델타 타임으로 업데이트하고 자신의 SSBO에 기록할 수 있습니다.

![](/images/compute_ssbo_read_write.svg)

이를 위해 컴퓨트 셰이더는 이전 프레임과 현재 프레임의 SSBO에 접근해야 합니다. 이는 디스크립터 설정에서 두 SSBO를 모두 컴퓨트 셰이더에 전달함으로써 이루어집니다. `storageBufferInfoLastFrame`과 `storageBufferInfoCurrentFrame`을 확인하세요.

```c++
for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHT; i++) {
    VkDescriptorBufferInfo uniformBufferInfo{};
    uniformBufferInfo.buffer = uniformBuffers[i];
    uniformBufferInfo.offset = 0;
    uniformBufferInfo.range = sizeof(UniformBufferObject);

    std::array<VkWriteDescriptorSet, 3> descriptorWrites{};
    ...

    VkDescriptorBufferInfo storageBufferInfoLastFrame{};
    // (i - 1) % MAX_FRAMES_IN_FLIGHT를 통해 이전 프레임의 버퍼에 접근
    storageBufferInfoLastFrame.buffer = shaderStorageBuffers[(i - 1 + MAX_FRAMES_IN_FLIGHT) % MAX_FRAMES_IN_FLIGHT];
    storageBufferInfoLastFrame.offset = 0;
    storageBufferInfoLastFrame.range = sizeof(Particle) * PARTICLE_COUNT;

    descriptorWrites[1].sType = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET;
    descriptorWrites[1].dstSet = computeDescriptorSets[i];
    descriptorWrites[1].dstBinding = 1;
    descriptorWrites[1].dstArrayElement = 0;
    descriptorWrites[1].descriptorType = VK_DESCRIPTOR_TYPE_STORAGE_BUFFER;
    descriptorWrites[1].descriptorCount = 1;
    descriptorWrites[1].pBufferInfo = &storageBufferInfoLastFrame;

    VkDescriptorBufferInfo storageBufferInfoCurrentFrame{};
    storageBufferInfoCurrentFrame.buffer = shaderStorageBuffers[i];
    storageBufferInfoCurrentFrame.offset = 0;
    storageBufferInfoCurrentFrame.range = sizeof(Particle) * PARTICLE_COUNT;

    descriptorWrites[2].sType = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET;
    descriptorWrites[2].dstSet = computeDescriptorSets[i];
    descriptorWrites[2].dstBinding = 2;
    descriptorWrites[2].dstArrayElement = 0;
    descriptorWrites[2].descriptorType = VK_DESCRIPTOR_TYPE_STORAGE_BUFFER;
    descriptorWrites[2].descriptorCount = 1;
    descriptorWrites[2].pBufferInfo = &storageBufferInfoCurrentFrame;

    vkUpdateDescriptorSets(device, 3, descriptorWrites.data(), 0, nullptr);
}
```

우리의 디스크립터 풀에서 SSBO에 대한 디스크립터 유형을 요청해야 한다는 것을 기억하세요.

```c++
std::array<VkDescriptorPoolSize, 2> poolSizes{};
...

poolSizes[1].type = VK_DESCRIPTOR_TYPE_STORAGE_BUFFER;
poolSizes[1].descriptorCount = static_cast<uint32_t>(MAX_FRAMES_IN_FLIGHT) * 2;
```

우리의 셋이 이전 프레임과 현재 프레임의 SSBO를 참조하기 때문에 풀에서 요청하는 `VK_DESCRIPTOR_TYPE_STORAGE_BUFFER` 유형의 수를 두 배로 늘려야 합니다.

## 컴퓨트 파이프라인

컴퓨트는 그래픽스 파이프라인의 일부가 아니므로 `vkCreateGraphicsPipelines`를 사용할 수 없습니다. 대신, 컴퓨트 명령을 실행하기 위해 `vkCreateComputePipelines`로 전용 컴퓨트 파이프라인을 생성해야 합니다. 컴퓨트 파이프라인은 래스터화 상태를 건드리지 않기 때문에 그래픽스 파이프라인보다 상태가 훨씬 적습니다.

```c++
VkComputePipelineCreateInfo pipelineInfo{};
pipelineInfo.sType = VK_STRUCTURE_TYPE_COMPUTE_PIPELINE_CREATE_INFO;
pipelineInfo.layout = computePipelineLayout;
pipelineInfo.stage = computeShaderStageInfo;

if (vkCreateComputePipelines(device, VK_NULL_HANDLE, 1, &pipelineInfo, nullptr, &computePipeline) != VK_SUCCESS) {
    throw std::runtime_error("failed to create compute pipeline!");
}
```

하나의 셰이더 단계와 파이프라인 레이아웃만 필요하므로 설정이 훨씬 간단합니다. 파이프라인 레이아웃은 그래픽스 파이프라인과 동일하게 작동합니다.

```c++
VkPipelineLayoutCreateInfo pipelineLayoutInfo{};
pipelineLayoutInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_LAYOUT_CREATE_INFO;
pipelineLayoutInfo.setLayoutCount = 1;
pipelineLayoutInfo.pSetLayouts = &computeDescriptorSetLayout;

if (vkCreatePipelineLayout(device, &pipelineLayoutInfo, nullptr, &computePipelineLayout) != VK_SUCCESS) {
    throw std::runtime_error("failed to create compute pipeline layout!");
}
```

## 컴퓨트 공간

컴퓨트 셰이더가 어떻게 작동하고 GPU에 컴퓨트 워크로드를 제출하는지 알아보기 전에, 두 가지 중요한 컴퓨트 개념인 **워크 그룹(work groups)**과 **인보케이션(invocations)**에 대해 이야기해야 합니다. 이들은 컴퓨트 워크로드가 GPU의 컴퓨트 하드웨어에 의해 3차원(x, y, z)으로 어떻게 처리되는지에 대한 추상적인 실행 모델을 정의합니다.

**워크 그룹**은 컴퓨트 워크로드가 GPU의 컴퓨트 하드웨어에 의해 어떻게 형성되고 처리되는지를 정의합니다. GPU가 처리해야 할 작업 항목이라고 생각할 수 있습니다. 워크 그룹의 차원은 애플리케이션에서 커맨드 버퍼 기록 시 디스패치 명령을 사용하여 설정됩니다.

그리고 각 워크 그룹은 동일한 컴퓨트 셰이더를 실행하는 **인보케이션**의 모음입니다. 인보케이션은 잠재적으로 병렬로 실행될 수 있으며, 그 차원은 컴퓨트 셰이더에서 설정됩니다. 단일 워크 그룹 내의 인보케이션들은 공유 메모리에 접근할 수 있습니다.

이 이미지는 이 둘의 관계를 3차원으로 보여줍니다.

![](/images/compute_space.svg)

워크 그룹(`vkCmdDispatch`로 정의)과 인보케이션(컴퓨트 셰이더의 로컬 크기로 정의)의 차원 수는 입력 데이터가 어떻게 구조화되어 있는지에 따라 달라집니다. 예를 들어 이 챕터에서처럼 1차원 배열 작업을 하는 경우, 둘 다에 대해 x 차원만 지정하면 됩니다.

예를 들어, 워크 그룹 수를 [64, 1, 1]로 디스패치하고 컴퓨트 셰이더의 로컬 크기를 [32, 32, 1]로 설정하면, 컴퓨트 셰이더는 64 x 32 x 32 = 65,536번 호출됩니다.

워크 그룹과 로컬 크기의 최대 개수는 구현마다 다르므로, 항상 `VkPhysicalDeviceLimits`의 컴퓨트 관련 `maxComputeWorkGroupCount`, `maxComputeWorkGroupInvocations`, `maxComputeWorkGroupSize` 제한을 확인해야 합니다.

## 컴퓨트 셰이더

이제 컴퓨트 셰이더 파이프라인을 설정하는 데 필요한 모든 부분을 배웠으니, 컴퓨트 셰이더 자체를 살펴볼 차례입니다. 정점 및 프래그먼트 셰이더와 같이 GLSL 셰이더를 사용하는 것에 대해 배운 모든 것이 컴퓨트 셰이더에도 적용됩니다. 문법은 동일하며, 애플리케이션과 셰이더 간의 데이터 전달과 같은 많은 개념도 동일합니다. 하지만 몇 가지 중요한 차이점이 있습니다.

선형 파티클 배열을 업데이트하기 위한 아주 기본적인 컴퓨트 셰이더는 다음과 같습니다.

```glsl
#version 450

layout (binding = 0) uniform ParameterUBO {
    float deltaTime;
} ubo;

struct Particle {
    vec2 position;
    vec2 velocity;
    vec4 color;
};

layout(std140, binding = 1) readonly buffer ParticleSSBOIn {
   Particle particlesIn[ ];
};

layout(std140, binding = 2) buffer ParticleSSBOOut {
   Particle particlesOut[ ];
};

layout (local_size_x = 256, local_size_y = 1, local_size_z = 1) in;

void main() 
{
    uint index = gl_GlobalInvocationID.x;  

    Particle particleIn = particlesIn[index];

    particlesOut[index].position = particleIn.position + particleIn.velocity.xy * ubo.deltaTime;
    particlesOut[index].velocity = particleIn.velocity;
    ...
}
```

셰이더의 상단 부분은 셰이더 입력을 위한 선언을 포함합니다. 첫 번째는 바인딩 0에 있는 유니폼 버퍼 객체로, 이 튜토리얼에서 이미 배운 것입니다. 그 아래에는 C++ 코드의 선언과 일치하는 `Particle` 구조체를 선언합니다. 바인딩 1은 이전 프레임의 파티클 데이터가 있는 셰이더 저장 버퍼 객체(디스크립터 설정 참조)를, 바인딩 2는 현재 프레임의 SSBO를 가리키며, 이 셰이더로 업데이트할 대상입니다.

흥미로운 점은 컴퓨트 공간과 관련된 이 컴퓨트 전용 선언입니다.

```glsl
layout (local_size_x = 256, local_size_y = 1, local_size_z = 1) in;
```
이는 현재 워크 그룹에서 이 컴퓨트 셰이더의 인보케이션 수를 정의합니다. 앞서 언급했듯이, 이것은 컴퓨트 공간의 로컬 부분입니다. 그래서 접두사로 `local_`이 붙습니다. 우리는 선형 1D 파티클 배열에서 작업하므로, `local_size_x`에 x 차원에 대한 숫자만 지정하면 됩니다.

`main` 함수는 이전 프레임의 SSBO에서 읽고 업데이트된 파티클 위치를 현재 프레임의 SSBO에 씁니다. 다른 셰이더 유형과 마찬가지로 컴퓨트 셰이더는 고유한 내장 입력 변수 집합을 가집니다. 내장 변수는 항상 `gl_` 접두사가 붙습니다. 그러한 내장 변수 중 하나가 `gl_GlobalInvocationID`이며, 이는 현재 디스패치 내에서 현재 컴퓨트 셰이더 인보케이션을 고유하게 식별하는 변수입니다. 우리는 이것을 사용하여 파티클 배열에 인덱싱합니다.

## 컴퓨트 명령 실행하기

### 디스패치

이제 GPU에 실제로 컴퓨트 작업을 하도록 지시할 차례입니다. 이는 커맨드 버퍼 내에서 `vkCmdDispatch`를 호출하여 수행됩니다. 완벽하게 맞지는 않지만, 디스패치는 컴퓨트에서 그래픽스의 `vkCmdDraw`와 같은 드로우 콜과 유사한 역할을 합니다. 이는 최대 3차원의 주어진 수의 컴퓨트 작업 항목을 디스패치합니다.

```c++
VkCommandBufferBeginInfo beginInfo{};
beginInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO;

if (vkBeginCommandBuffer(commandBuffer, &beginInfo) != VK_SUCCESS) {
    throw std::runtime_error("failed to begin recording command buffer!");
}

...

vkCmdBindPipeline(commandBuffer, VK_PIPELINE_BIND_POINT_COMPUTE, computePipeline);
vkCmdBindDescriptorSets(commandBuffer, VK_PIPELINE_BIND_POINT_COMPUTE, computePipelineLayout, 0, 1, &computeDescriptorSets[i], 0, 0);

vkCmdDispatch(computeCommandBuffer, PARTICLE_COUNT / 256, 1, 1);

...

if (vkEndCommandBuffer(commandBuffer) != VK_SUCCESS) {
    throw std::runtime_error("failed to record command buffer!");
}
```

`vkCmdDispatch`는 x 차원에서 `PARTICLE_COUNT / 256` 개의 로컬 워크 그룹을 디스패치합니다. 우리 파티클 배열은 선형이므로 다른 두 차원은 1로 남겨두어 1차원 디스패치가 됩니다. 그런데 왜 파티클 수(우리 배열의)를 256으로 나누는 걸까요? 이전 단락에서 워크 그룹의 모든 컴퓨트 셰이더가 256번의 인보케이션을 수행한다고 정의했기 때문입니다. 따라서 4096개의 파티클이 있다면 16개의 워크 그룹을 디스패치하고, 각 워크 그룹은 256개의 컴퓨트 셰이더 인보케이션을 실행합니다. 두 숫자를 올바르게 맞추는 것은 일반적으로 워크로드와 실행 중인 하드웨어에 따라 약간의 조정과 프로파일링이 필요합니다. 만약 파티클 크기가 동적이고 항상 256으로 나누어지지 않는다면, 컴퓨트 셰이더 시작 부분에서 `gl_GlobalInvocationID`를 사용하여 전역 인보케이션 인덱스가 파티클 수보다 크면 반환할 수 있습니다.

그리고 컴퓨트 파이프라인의 경우와 마찬가지로, 컴퓨트 커맨드 버퍼는 그래픽스 커맨드 버퍼보다 훨씬 적은 상태를 포함합니다. 렌더 패스를 시작하거나 뷰포트를 설정할 필요가 없습니다.

### 작업 제출하기

우리 샘플은 컴퓨트와 그래픽스 작업을 모두 수행하므로, 프레임당 그래픽스와 컴퓨트 큐에 두 번의 제출을 할 것입니다(`drawFrame` 함수 참조).

```c++
...
if (vkQueueSubmit(computeQueue, 1, &submitInfo, nullptr) != VK_SUCCESS) {
    throw std::runtime_error("failed to submit compute command buffer!");
};
...
if (vkQueueSubmit(graphicsQueue, 1, &submitInfo, inFlightFences[currentFrame]) != VK_SUCCESS) {
    throw std::runtime_error("failed to submit draw command buffer!");
}
```

컴퓨트 큐에 대한 첫 번째 제출은 컴퓨트 셰이더를 사용하여 파티클 위치를 업데이트하고, 두 번째 제출은 그 업데이트된 데이터를 사용하여 파티클 시스템을 그릴 것입니다.

### 그래픽스와 컴퓨트 동기화하기

동기화는 Vulkan의 중요한 부분이며, 그래픽스와 함께 컴퓨트를 할 때는 더욱 그렇습니다. 잘못되거나 부족한 동기화는 컴퓨트 셰이더가 파티클 업데이트(쓰기)를 마치기 전에 정점 단계가 파티클 그리기(읽기)를 시작하거나(읽기 후 쓰기(read-after-write) 위험), 정점 파이프라인 부분에서 아직 사용 중인 파티클을 컴퓨트 셰이더가 업데이트하기 시작하는(쓰기 후 읽기(write-after-read) 위험) 결과를 초래할 수 있습니다.

따라서 그래픽스와 컴퓨트 부하를 적절하게 동기화하여 이러한 경우가 발생하지 않도록 해야 합니다. 컴퓨트 워크로드를 제출하는 방식에 따라 여러 가지 방법이 있지만, 우리 경우처럼 두 개의 별도 제출을 하는 경우에는 [세마포어](03_Drawing_a_triangle/03_Drawing/02_Rendering_and_presentation.md#page_Semaphores)와 [펜스](03_Drawing_a_triangle/03_Drawing/02_Rendering_and_presentation.md#page_Fences)를 사용하여 정점 셰이더가 컴퓨트 셰이더의 업데이트가 끝날 때까지 정점 가져오기를 시작하지 않도록 보장할 것입니다.

두 제출이 차례로 순서대로 이루어지더라도 GPU에서 이 순서대로 실행된다는 보장이 없기 때문에 이는 필수적입니다. 대기 및 신호 세마포어를 추가하면 이 실행 순서가 보장됩니다.

먼저 `createSyncObjects`에서 컴퓨트 작업을 위한 새로운 동기화 프리미티브 세트를 추가합니다. 컴퓨트 펜스는 그래픽스 펜스와 마찬가지로 신호(signaled) 상태로 생성됩니다. 그렇지 않으면 첫 번째 드로우가 펜스가 신호되기를 기다리다 타임아웃될 것이기 때문입니다. ([이전 프레임 기다리기](03_Drawing_a_triangle/03_Drawing/02_Rendering_and_presentation.md#page_Waiting-for-the-previous-frame)에서 자세히 설명).

```c++
std::vector<VkFence> computeInFlightFences;
std::vector<VkSemaphore> computeFinishedSemaphores;
...
computeInFlightFences.resize(MAX_FRAMES_IN_FLIGHT);
computeFinishedSemaphores.resize(MAX_FRAMES_IN_FLIGHT);

VkSemaphoreCreateInfo semaphoreInfo{};
semaphoreInfo.sType = VK_STRUCTURE_TYPE_SEMAPHORE_CREATE_INFO;

VkFenceCreateInfo fenceInfo{};
fenceInfo.sType = VK_STRUCTURE_TYPE_FENCE_CREATE_INFO;
fenceInfo.flags = VK_FENCE_CREATE_SIGNALED_BIT;

for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHT; i++) {
    ...
    if (vkCreateSemaphore(device, &semaphoreInfo, nullptr, &computeFinishedSemaphores[i]) != VK_SUCCESS ||
        vkCreateFence(device, &fenceInfo, nullptr, &computeInFlightFences[i]) != VK_SUCCESS) {
        throw std::runtime_error("failed to create compute synchronization objects for a frame!");
    }
}
```
그런 다음 이것들을 사용하여 컴퓨트 버퍼 제출을 그래픽스 제출과 동기화합니다.

```c++
// 컴퓨트 제출
vkWaitForFences(device, 1, &computeInFlightFences[currentFrame], VK_TRUE, UINT64_MAX);

updateUniformBuffer(currentFrame);

vkResetFences(device, 1, &computeInFlightFences[currentFrame]);

vkResetCommandBuffer(computeCommandBuffers[currentFrame], /*VkCommandBufferResetFlagBits*/ 0);
recordComputeCommandBuffer(computeCommandBuffers[currentFrame]);

submitInfo.commandBufferCount = 1;
submitInfo.pCommandBuffers = &computeCommandBuffers[currentFrame];
submitInfo.signalSemaphoreCount = 1;
submitInfo.pSignalSemaphores = &computeFinishedSemaphores[currentFrame];

if (vkQueueSubmit(computeQueue, 1, &submitInfo, computeInFlightFences[currentFrame]) != VK_SUCCESS) {
    throw std::runtime_error("failed to submit compute command buffer!");
};

// 그래픽스 제출
vkWaitForFences(device, 1, &inFlightFences[currentFrame], VK_TRUE, UINT64_MAX);

...

vkResetFences(device, 1, &inFlightFences[currentFrame]);

vkResetCommandBuffer(commandBuffers[currentFrame], /*VkCommandBufferResetFlagBits*/ 0);
recordCommandBuffer(commandBuffers[currentFrame], imageIndex);

VkSemaphore waitSemaphores[] = { computeFinishedSemaphores[currentFrame], imageAvailableSemaphores[currentFrame] };
VkPipelineStageFlags waitStages[] = { VK_PIPELINE_STAGE_VERTEX_INPUT_BIT, VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT };
submitInfo = {};
submitInfo.sType = VK_STRUCTURE_TYPE_SUBMIT_INFO;

submitInfo.waitSemaphoreCount = 2;
submitInfo.pWaitSemaphores = waitSemaphores;
submitInfo.pWaitDstStageMask = waitStages;
submitInfo.commandBufferCount = 1;
submitInfo.pCommandBuffers = &commandBuffers[currentFrame];
submitInfo.signalSemaphoreCount = 1;
submitInfo.pSignalSemaphores = &renderFinishedSemaphores[currentFrame];

if (vkQueueSubmit(graphicsQueue, 1, &submitInfo, inFlightFences[currentFrame]) != VK_SUCCESS) {
    throw std::runtime_error("failed to submit draw command buffer!");
}
```

[세마포어 챕터](03_Drawing_a_triangle/03_Drawing/02_Rendering_and_presentation.md#page_Semaphores)의 샘플과 유사하게, 이 설정은 대기 세마포어를 지정하지 않았기 때문에 컴퓨트 셰이더를 즉시 실행합니다. `vkWaitForFences` 명령으로 현재 프레임의 컴퓨트 커맨드 버퍼가 실행을 마칠 때까지 기다리기 때문에 이는 괜찮습니다.

반면에 그래픽스 제출은 컴퓨트 작업이 끝나기를 기다려야 컴퓨트 버퍼가 아직 업데이트 중일 때 정점을 가져오기 시작하지 않습니다. 따라서 현재 프레임의 `computeFinishedSemaphores`를 기다리고, 그래픽스 제출이 정점이 소비되는 `VK_PIPELINE_STAGE_VERTEX_INPUT_BIT` 단계에서 기다리도록 합니다.

하지만 프래그먼트 셰이더가 이미지가 제시될 때까지 컬러 어태치먼트에 출력하지 않도록 프레젠테이션도 기다려야 합니다. 따라서 `VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT` 단계에서 현재 프레임의 `imageAvailableSemaphores`도 기다립니다.

## 파티클 시스템 그리기

앞서 우리는 Vulkan의 버퍼가 여러 사용 사례를 가질 수 있다는 것을 배웠고, 그래서 우리는 파티클을 포함하는 셰이더 저장 버퍼를 셰이더 저장 버퍼 비트와 정점 버퍼 비트를 모두 사용하여 생성했습니다. 이는 이전 챕터에서 "순수" 정점 버퍼를 사용했던 것처럼 드로잉을 위해 셰이더 저장 버퍼를 사용할 수 있다는 것을 의미합니다.

먼저 우리 파티클 구조체와 일치하도록 정점 입력 상태를 설정합니다.

```c++
struct Particle {
    ...

    static std::array<VkVertexInputAttributeDescription, 2> getAttributeDescriptions() {
        std::array<VkVertexInputAttributeDescription, 2> attributeDescriptions{};

        attributeDescriptions[0].binding = 0;
        attributeDescriptions[0].location = 0;
        attributeDescriptions[0].format = VK_FORMAT_R32G32_SFLOAT;
        attributeDescriptions[0].offset = offsetof(Particle, position);

        attributeDescriptions[1].binding = 0;
        attributeDescriptions[1].location = 1;
        attributeDescriptions[1].format = VK_FORMAT_R32G32B32A32_SFLOAT;
        attributeDescriptions[1].offset = offsetof(Particle, color);

        return attributeDescriptions;
    }
};
```

`velocity`는 컴퓨트 셰이더에서만 사용되므로 정점 입력 속성에 추가하지 않는다는 점에 유의하세요.

그런 다음 다른 정점 버퍼처럼 바인딩하고 그립니다.

```c++
vkCmdBindVertexBuffers(commandBuffer, 0, 1, &shaderStorageBuffers[currentFrame], offsets);

vkCmdDraw(commandBuffer, PARTICLE_COUNT, 1, 0, 0);
```

## 결론

이 챕터에서는 CPU의 작업을 GPU로 오프로드하기 위해 컴퓨트 셰이더를 사용하는 방법을 배웠습니다. 컴퓨트 셰이더가 없다면 현대 게임과 애플리케이션의 많은 효과들이 불가능하거나 훨씬 느리게 실행될 것입니다. 하지만 그래픽스보다 더 많은 분야에서 컴퓨트는 많은 사용 사례를 가지고 있으며, 이 챕터는 가능한 것의 일부만을 보여줍니다. 이제 컴퓨트 셰이더를 사용하는 방법을 알았으니, 다음과 같은 고급 컴퓨트 주제들을 살펴보는 것도 좋습니다.

- 공유 메모리
- [비동기 컴퓨트 (Asynchronous compute)](https://github.com/KhronosGroup/Vulkan-Samples/tree/master/samples/performance/async_compute)
- 원자적 연산 (Atomic operations)
- [서브그룹 (Subgroups)](https://www.khronos.org/blog/vulkan-subgroup-tutorial)

[공식 Khronos Vulkan 샘플 저장소](https://github.com/KhronosGroup/Vulkan-Samples/tree/master/samples/api)에서 몇 가지 고급 컴퓨트 샘플을 찾을 수 있습니다.

[C++ 코드](/code/31_compute_shader.cpp) /
[정점 셰이더](/code/31_shader_compute.vert) /
[프래그먼트 셰이더](/code/31_shader_compute.frag) /
[컴퓨트 셰이더](/code/31_shader_compute.comp)