## 서론

지금까지는 정점별(per-vertex) 색상을 사용해 지오메트리를 색칠해왔는데, 이는 다소 제한적인 방법입니다. 이번 장에서는 텍스처 매핑을 구현하여 지오메트리가 더 흥미롭게 보이도록 만들 것입니다. 이를 통해 다음 장에서는 기본적인 3D 모델을 로드하고 그릴 수도 있게 됩니다.

애플리케이션에 텍스처를 추가하는 작업은 다음 단계를 포함합니다.

*   디바이스 메모리를 기반으로 하는 이미지 객체 생성
*   이미지 파일의 픽셀로 채우기
*   이미지 샘플러 생성
*   텍스처에서 색상을 샘플링하기 위한 결합 이미지 샘플러 디스크립터 추가

이전에도 이미지 객체를 다룬 적이 있지만, 그것들은 스왑체인 확장에 의해 자동으로 생성되었습니다. 이번에는 직접 하나를 만들어야 합니다. 이미지를 생성하고 데이터를 채우는 과정은 정점 버퍼 생성과 유사합니다. 먼저 스테이징 리소스를 생성하고 픽셀 데이터로 채운 다음, 렌더링에 사용할 최종 이미지 객체로 복사합니다. 이 목적으로 스테이징 이미지를 생성할 수도 있지만, Vulkan은 `VkBuffer`에서 이미지로 픽셀을 복사하는 것도 허용하며, 이 API는 [일부 하드웨어에서 실제로 더 빠릅니다](https://developer.nvidia.com/vulkan-memory-management). 우리는 먼저 이 버퍼를 생성하고 픽셀 값으로 채운 다음, 픽셀을 복사할 이미지를 생성할 것입니다. 이미지 생성은 버퍼 생성과 크게 다르지 않습니다. 이전에 보았듯이 메모리 요구사항을 쿼리하고, 디바이스 메모리를 할당하고, 바인딩하는 과정이 포함됩니다.

하지만 이미지 작업 시에는 추가적으로 신경 써야 할 것이 있습니다. 이미지는 메모리에서 픽셀이 구성되는 방식에 영향을 미치는 다양한 *레이아웃(layouts)*을 가질 수 있습니다. 그래픽 하드웨어의 작동 방식 때문에, 단순히 픽셀을 행 단위로 저장하는 것이 최상의 성능으로 이어지지 않을 수 있습니다. 이미지에 대한 어떠한 작업을 수행할 때든, 해당 작업에 최적화된 레이아웃을 가지고 있는지 확인해야 합니다. 렌더 패스를 지정할 때 이미 이러한 레이아웃 중 일부를 본 적이 있습니다:

*   `vk::ImageLayout::PRESENT_SRC_KHR`: 화면 표시에 최적화
*   `vk::ImageLayout::COLOR_ATTACHMENT_OPTIMAL`: 프래그먼트 셰이더에서 색상을 쓰는 어태치먼트로 최적화
*   `vk::ImageLayout::TRANSFER_SRC_OPTIMAL`: `vkCmdCopyImageToBuffer`와 같은 전송 작업의 소스로 최적화
*   `vk::ImageLayout::TRANSFER_DST_OPTIMAL`: `vkCmdCopyBufferToImage`와 같은 전송 작업의 대상으로 최적화
*   `vk::ImageLayout::SHADER_READ_ONLY_OPTIMAL`: 셰이더에서 샘플링하기에 최적화

이미지의 레이아웃을 전환하는 가장 일반적인 방법 중 하나는 *파이프라인 배리어(pipeline barrier)*입니다. 파이프라인 배리어는 주로 리소스 접근을 동기화하는 데 사용됩니다. 예를 들어, 이미지를 읽기 전에 쓰기가 완료되었는지 확인하는 것과 같지만, 레이아웃을 전환하는 데에도 사용할 수 있습니다. 이번 장에서는 파이프라인 배리어가 이 목적으로 어떻게 사용되는지 볼 것입니다. 배리어는 `vk::SharingMode::EXCLUSIVE`를 사용할 때 큐 패밀리 소유권을 이전하는 데에도 추가적으로 사용될 수 있습니다.

## 이미지 라이브러리

이미지를 로드하기 위한 많은 라이브러리가 있으며, Rust에서는 `image` 크레이트가 가장 대중적이고 강력한 선택지입니다. `stb_image`와 마찬가지로 다양한 이미지 형식을 지원하며 사용하기 매우 쉽습니다.

`Cargo.toml` 파일의 `[dependencies]` 섹션에 `image` 크레이트를 추가하세요.

```toml
[dependencies]
ash = "0.37"
# ... other dependencies
image = "0.24"
```

## 이미지 로딩하기

이제 `image` 크레이트를 사용하여 텍스처를 로드할 수 있습니다. 먼저 애플리케이션 구조체에 관련 필드를 추가해야 합니다.

```rust
// In VulkanApp struct
struct VulkanApp {
    // ...
    texture_image: vk::Image,
    texture_image_memory: vk::DeviceMemory,
    // ...
}
```

그리고 `init_vulkan` 함수에서 `create_texture_image` 함수를 호출하도록 순서를 조정합니다. 커맨드 버퍼를 사용하므로 커맨드 풀 생성 이후에 호출되어야 합니다.

```rust
impl VulkanApp {
    pub fn init_vulkan(&mut self) -> Result<(), Box<dyn Error>> {
        // ...
        self.create_command_pool()?;
        self.create_texture_image()?;
        self.create_vertex_buffer()?;
        // ...
        Ok(())
    }

    // ...

    fn create_texture_image(&mut self) -> Result<(), Box<dyn Error>> {
        // ...
        Ok(())
    }
}
```

이제 `create_texture_image` 함수에서 이미지를 로드합니다. `shaders` 디렉토리 옆에 `textures` 디렉토리를 만들고, 이 튜토리얼에서 사용할 512x512 픽셀 크기의 `texture.jpg` 이미지를 저장하세요.

![](/images/texture.jpg)

`image` 크레이트로 이미지를 로드하는 것은 매우 간단합니다.

```rust
use image::GenericImageView;
// ... inside create_texture_image
fn create_texture_image(&mut self) -> Result<(), Box<dyn Error>> {
    let image_object = image::open("textures/texture.jpg")?;
    let (tex_width, tex_height) = image_object.dimensions();
    let image_data = image_object.to_rgba8().into_raw();
    let image_size = (tex_width * tex_height * 4) as vk::DeviceSize;

    // ... rest of the function
    Ok(())
}
```

`image::open`은 파일 경로에서 이미지를 로드합니다. `.to_rgba8()`는 이미지를 RGBA 형식으로 변환하여 알파 채널이 없는 이미지도 일관성 있게 처리합니다. `.into_raw()`는 이 이미지 데이터를 `Vec<u8>` 형태로 반환합니다. 이 벡터는 `tex_width * tex_height * 4` 바이트 크기를 가집니다.

## 스테이징 버퍼

이제 호스트 가시성(host-visible) 메모리에 버퍼를 생성하여 `vkMapMemory`를 사용하고 픽셀 데이터를 복사할 수 있도록 합니다.

```rust
// ... inside create_texture_image
let (staging_buffer, staging_buffer_memory) = self.create_buffer(
    image_size,
    vk::BufferUsageFlags::TRANSFER_SRC,
    vk::MemoryPropertyFlags::HOST_VISIBLE | vk::MemoryPropertyFlags::HOST_COHERENT,
)?;

unsafe {
    let data_ptr = self.device.map_memory(
        staging_buffer_memory,
        0,
        image_size,
        vk::MemoryMapFlags::empty(),
    )? as *mut u8;

    data_ptr.copy_from_nonoverlapping(image_data.as_ptr(), image_data.len());

    self.device.unmap_memory(staging_buffer_memory);
}
```

`create_buffer` 헬퍼 함수를 사용하여 스테이징 버퍼를 생성합니다. 이 버퍼는 호스트에서 접근 가능해야 하고(`HOST_VISIBLE`), 전송 소스로 사용될 수 있어야 합니다(`TRANSFER_SRC`). `HOST_COHERENT` 플래그는 매핑된 메모리에 쓴 내용이 자동으로 디바이스에 보이도록 보장합니다.

`map_memory`로 메모리 포인터를 얻은 후, `copy_from_nonoverlapping`을 사용하여 이미지 픽셀 데이터를 버퍼로 복사합니다. 작업이 끝나면 `unmap_memory`를 호출합니다. Rust의 `image_data`는 `Vec<u8>`이므로 범위를 벗어나면 자동으로 메모리가 해제됩니다. C++의 `stbi_image_free`처럼 수동으로 해제할 필요가 없습니다.

## 텍스처 이미지

셰이더가 버퍼의 픽셀 값에 접근하도록 설정할 수도 있지만, Vulkan에서는 이미지 객체를 사용하는 것이 더 좋습니다. 이미지 객체는 2D 좌표를 사용할 수 있게 하여 색상을 더 쉽고 빠르게 가져올 수 있게 해줍니다. 이미지 객체 내의 픽셀은 텍셀(texel)이라고 하며, 지금부터 이 용어를 사용하겠습니다.

`createTextureImage` 함수의 나머지 부분에서 텍스처 이미지를 생성합니다.

```rust
// ... inside create_texture_image
let (texture_image, texture_image_memory) = self.create_image(
    tex_width,
    tex_height,
    vk::Format::R8G8B8A8_SRGB,
    vk::ImageTiling::OPTIMAL,
    vk::ImageUsageFlags::TRANSFER_DST | vk::ImageUsageFlags::SAMPLED,
    vk::MemoryPropertyFlags::DEVICE_LOCAL,
)?;

self.texture_image = texture_image;
self.texture_image_memory = texture_image_memory;
```

버퍼 생성 로직을 `create_buffer`로 리팩토링했듯이, 이미지 생성 로직도 `create_image`라는 헬퍼 함수로 추상화하는 것이 좋습니다.

```rust
impl VulkanApp {
    fn create_image(
        &self,
        width: u32,
        height: u32,
        format: vk::Format,
        tiling: vk::ImageTiling,
        usage: vk::ImageUsageFlags,
        properties: vk::MemoryPropertyFlags,
    ) -> Result<(vk::Image, vk::DeviceMemory), Box<dyn Error>> {
        let image_info = vk::ImageCreateInfo::builder()
            .image_type(vk::ImageType::TYPE_2D)
            .extent(vk::Extent3D { width, height, depth: 1 })
            .mip_levels(1)
            .array_layers(1)
            .format(format)
            .tiling(tiling)
            .initial_layout(vk::ImageLayout::UNDEFINED)
            .usage(usage)
            .samples(vk::SampleCountFlags::TYPE_1)
            .sharing_mode(vk::SharingMode::EXCLUSIVE);

        let image = unsafe { self.device.create_image(&image_info, None)? };

        let mem_requirements = unsafe { self.device.get_image_memory_requirements(image) };

        let alloc_info = vk::MemoryAllocateInfo::builder()
            .allocation_size(mem_requirements.size)
            .memory_type_index(self.find_memory_type(
                mem_requirements.memory_type_bits,
                properties,
            )?);

        let image_memory = unsafe { self.device.allocate_memory(&alloc_info, None)? };

        unsafe { self.device.bind_image_memory(image, image_memory, 0)? };

        Ok((image, image_memory))
    }
}
```

`create_image` 함수는 너비, 높이, 포맷, 타일링, 사용법, 메모리 속성을 인자로 받습니다.
*   `image_type`: 2D 텍스처이므로 `TYPE_2D`입니다.
*   `extent`: 이미지의 크기를 지정합니다. 2D 이미지이므로 `depth`는 1입니다.
*   `format`: `VK_FORMAT_R8G8B8A8_SRGB`는 8비트 RGBA 채널을 사용하며, sRGB 색 공간에 있음을 의미합니다. 픽셀 데이터와 형식이 일치해야 합니다.
*   `tiling`: `OPTIMAL`은 셰이더에서 효율적으로 접근하기 위한 구현 정의 레이아웃을 사용합니다.
*   `initial_layout`: `UNDEFINED`로 설정합니다. 첫 전환 시 텍셀 내용이 필요 없기 때문입니다.
*   `usage`: `TRANSFER_DST`는 이 이미지가 복사 작업의 대상이 될 수 있음을, `SAMPLED`는 셰이더에서 샘플링할 수 있음을 의미합니다.
*   나머지 필드는 버퍼 생성과 유사합니다.

이미지를 생성한 후, `get_image_memory_requirements`로 메모리 요구사항을 얻고, 적절한 메모리 타입을 찾아 `allocate_memory`로 메모리를 할당한 뒤, `bind_image_memory`로 이미지와 메모리를 바인딩합니다.

## 레이아웃 전환

이제부터 커맨드 버퍼를 기록하고 실행하는 작업이 반복되므로, 이 로직을 헬퍼 함수로 분리합시다.

```rust
impl VulkanApp {
    fn begin_single_time_commands(&self) -> Result<vk::CommandBuffer, Box<dyn Error>> {
        let alloc_info = vk::CommandBufferAllocateInfo::builder()
            .level(vk::CommandBufferLevel::PRIMARY)
            .command_pool(self.command_pool)
            .command_buffer_count(1);

        let command_buffer = unsafe { self.device.allocate_command_buffers(&alloc_info)?[0] };

        let begin_info = vk::CommandBufferBeginInfo::builder()
            .flags(vk::CommandBufferUsageFlags::ONE_TIME_SUBMIT);

        unsafe { self.device.begin_command_buffer(command_buffer, &begin_info)? };

        Ok(command_buffer)
    }

    fn end_single_time_commands(&self, command_buffer: vk::CommandBuffer) -> Result<(), Box<dyn Error>> {
        unsafe {
            self.device.end_command_buffer(command_buffer)?;

            let submit_info = vk::SubmitInfo::builder()
                .command_buffers(&[command_buffer]);

            self.device.queue_submit(self.graphics_queue, &[submit_info.build()], vk::Fence::null())?;
            self.device.queue_wait_idle(self.graphics_queue)?;

            self.device.free_command_buffers(self.command_pool, &[command_buffer]);
        }
        Ok(())
    }
}
```

이 두 함수는 이전에 `copy_buffer`에서 사용했던 로직과 동일합니다. 일회성 커맨드 버퍼를 할당하고 시작하며, 제출 후 대기하고 해제합니다. 이제 `copy_buffer` 함수를 이 헬퍼들을 사용해 단순화할 수 있습니다.

```rust
// In VulkanApp impl
fn copy_buffer(
    &self,
    src_buffer: vk::Buffer,
    dst_buffer: vk::Buffer,
    size: vk::DeviceSize,
) -> Result<(), Box<dyn Error>> {
    let command_buffer = self.begin_single_time_commands()?;

    let copy_region = vk::BufferCopy::builder().size(size);
    unsafe {
        self.device.cmd_copy_buffer(command_buffer, src_buffer, dst_buffer, &[copy_region.build()]);
    }

    self.end_single_time_commands(command_buffer)?;

    Ok(())
}
```

이제 이미지 레이아웃 전환을 위한 함수를 만듭니다. 레이아웃 전환에는 *이미지 메모리 배리어*를 사용한 파이프라인 배리어가 주로 사용됩니다.

```rust
// In VulkanApp impl
fn transition_image_layout(
    &self,
    image: vk::Image,
    format: vk::Format, // format is not used yet, but will be for depth buffer
    old_layout: vk::ImageLayout,
    new_layout: vk::ImageLayout,
) -> Result<(), Box<dyn Error>> {
    let command_buffer = self.begin_single_time_commands()?;

    let (src_access_mask, dst_access_mask, src_stage, dst_stage) =
        match (old_layout, new_layout) {
            (
                vk::ImageLayout::UNDEFINED,
                vk::ImageLayout::TRANSFER_DST_OPTIMAL,
            ) => (
                vk::AccessFlags::empty(),
                vk::AccessFlags::TRANSFER_WRITE,
                vk::PipelineStageFlags::TOP_OF_PIPE,
                vk::PipelineStageFlags::TRANSFER,
            ),
            (
                vk::ImageLayout::TRANSFER_DST_OPTIMAL,
                vk::ImageLayout::SHADER_READ_ONLY_OPTIMAL,
            ) => (
                vk::AccessFlags::TRANSFER_WRITE,
                vk::AccessFlags::SHADER_READ,
                vk::PipelineStageFlags::TRANSFER,
                vk::PipelineStageFlags::FRAGMENT_SHADER,
            ),
            _ => return Err("Unsupported layout transition!".into()),
        };

    let barrier = vk::ImageMemoryBarrier::builder()
        .old_layout(old_layout)
        .new_layout(new_layout)
        .src_queue_family_index(vk::QUEUE_FAMILY_IGNORED)
        .dst_queue_family_index(vk::QUEUE_FAMILY_IGNORED)
        .image(image)
        .subresource_range(vk::ImageSubresourceRange {
            aspect_mask: vk::ImageAspectFlags::COLOR,
            base_mip_level: 0,
            level_count: 1,
            base_array_layer: 0,
            layer_count: 1,
        })
        .src_access_mask(src_access_mask)
        .dst_access_mask(dst_access_mask);

    unsafe {
        self.device.cmd_pipeline_barrier(
            command_buffer,
            src_stage,
            dst_stage,
            vk::DependencyFlags::empty(),
            &[],
            &[],
            &[barrier.build()],
        );
    }

    self.end_single_time_commands(command_buffer)?;

    Ok(())
}
```

배리어는 리소스 접근을 동기화하는 데 사용됩니다.
*   `old_layout`, `new_layout`: 전환 전후의 레이아웃을 지정합니다.
*   `src_queue_family_index`, `dst_queue_family_index`: 큐 패밀리 소유권 이전이 없으면 `IGNORED`로 설정합니다.
*   `image`, `subresource_range`: 어떤 이미지의 어떤 부분에 배리어를 적용할지 지정합니다.
*   `src_access_mask`, `dst_access_mask`: 배리어 이전 작업과 배리어 이후 작업을 동기화합니다.
*   `source_stage`, `destination_stage`: 위 작업들이 발생하는 파이프라인 단계를 지정합니다.

우리가 처리할 두 가지 전환은 다음과 같습니다.
1.  **Undefined → Transfer Destination**: 데이터를 이미지에 쓰기 전입니다. 이전 작업은 없으므로 `src_access_mask`는 비어있고, `src_stage`는 파이프라인의 가장 처음인 `TOP_OF_PIPE`입니다. 쓰기 작업은 전송(transfer) 작업이므로 `dst_access_mask`는 `TRANSFER_WRITE`, `dst_stage`는 `TRANSFER`입니다.
2.  **Transfer Destination → Shader Read Only**: 이미지 쓰기가 완료된 후, 셰이더에서 읽을 준비를 합니다. 이전 작업은 전송 쓰기(`TRANSFER_WRITE` in `TRANSFER` stage)였고, 이후 작업은 프래그먼트 셰이더에서의 읽기(`SHADER_READ` in `FRAGMENT_SHADER` stage)가 될 것입니다.

## 버퍼를 이미지로 복사하기

스테이징 버퍼의 데이터를 이미지로 복사하는 헬퍼 함수도 만듭니다.

```rust
// In VulkanApp impl
fn copy_buffer_to_image(
    &self,
    buffer: vk::Buffer,
    image: vk::Image,
    width: u32,
    height: u32,
) -> Result<(), Box<dyn Error>> {
    let command_buffer = self.begin_single_time_commands()?;

    let region = vk::BufferImageCopy::builder()
        .buffer_offset(0)
        .buffer_row_length(0)
        .buffer_image_height(0)
        .image_subresource(vk::ImageSubresourceLayers {
            aspect_mask: vk::ImageAspectFlags::COLOR,
            mip_level: 0,
            base_array_layer: 0,
            layer_count: 1,
        })
        .image_offset(vk::Offset3D { x: 0, y: 0, z: 0 })
        .image_extent(vk::Extent3D { width, height, depth: 1 });

    unsafe {
        self.device.cmd_copy_buffer_to_image(
            command_buffer,
            buffer,
            image,
            vk::ImageLayout::TRANSFER_DST_OPTIMAL,
            &[region.build()],
        );
    }

    self.end_single_time_commands(command_buffer)?;

    Ok(())
}
```
`vkCmdCopyBufferToImage`는 버퍼의 어느 영역을 이미지의 어느 영역으로 복사할지 `VkBufferImageCopy` 구조체로 지정받습니다. 여기서 이미지는 복사 대상에 최적화된 레이아웃인 `TRANSFER_DST_OPTIMAL` 상태여야 합니다.

## 텍스처 이미지 준비하기

이제 모든 헬퍼 함수를 사용하여 `create_texture_image` 함수를 완성할 수 있습니다.

```rust
// final version of create_texture_image
fn create_texture_image(&mut self) -> Result<(), Box<dyn Error>> {
    let image_object = image::open("textures/texture.jpg")?;
    let (tex_width, tex_height) = image_object.dimensions();
    let image_data = image_object.to_rgba8().into_raw();
    let image_size = (tex_width * tex_height * 4) as vk::DeviceSize;

    let (staging_buffer, staging_buffer_memory) = self.create_buffer(
        image_size,
        vk::BufferUsageFlags::TRANSFER_SRC,
        vk::MemoryPropertyFlags::HOST_VISIBLE | vk::MemoryPropertyFlags::HOST_COHERENT,
    )?;

    // Copy data to staging buffer
    unsafe {
        let data_ptr = self.device.map_memory(staging_buffer_memory, 0, image_size, vk::MemoryMapFlags::empty())? as *mut u8;
        data_ptr.copy_from_nonoverlapping(image_data.as_ptr(), image_data.len());
        self.device.unmap_memory(staging_buffer_memory);
    }

    let (texture_image, texture_image_memory) = self.create_image(
        tex_width,
        tex_height,
        vk::Format::R8G8B8A8_SRGB,
        vk::ImageTiling::OPTIMAL,
        vk::ImageUsageFlags::TRANSFER_DST | vk::ImageUsageFlags::SAMPLED,
        vk::MemoryPropertyFlags::DEVICE_LOCAL,
    )?;
    self.texture_image = texture_image;
    self.texture_image_memory = texture_image_memory;

    // Transition layout and copy buffer
    self.transition_image_layout(
        self.texture_image,
        vk::Format::R8G8B8A8_SRGB,
        vk::ImageLayout::UNDEFINED,
        vk::ImageLayout::TRANSFER_DST_OPTIMAL,
    )?;
    self.copy_buffer_to_image(staging_buffer, self.texture_image, tex_width, tex_height)?;
    self.transition_image_layout(
        self.texture_image,
        vk::Format::R8G8B8A8_SRGB,
        vk::ImageLayout::TRANSFER_DST_OPTIMAL,
        vk::ImageLayout::SHADER_READ_ONLY_OPTIMAL,
    )?;

    // Cleanup staging buffer
    unsafe {
        self.device.destroy_buffer(staging_buffer, None);
        self.device.free_memory(staging_buffer_memory, None);
    }

    Ok(())
}
```

전체 프로세스는 다음과 같습니다:
1. 이미지를 파일에서 로드합니다.
2. 픽셀 데이터를 담을 스테이징 버퍼를 생성하고 데이터를 복사합니다.
3. 최종 텍스처 이미지(`DEVICE_LOCAL`)를 생성합니다.
4. 이미지 레이아웃을 `UNDEFINED`에서 `TRANSFER_DST_OPTIMAL`로 전환합니다.
5. 스테이징 버퍼에서 텍스처 이미지로 데이터를 복사합니다.
6. 셰이더에서 읽을 수 있도록 레이아웃을 `TRANSFER_DST_OPTIMAL`에서 `SHADER_READ_ONLY_OPTIMAL`로 전환합니다.
7. 스테이징 버퍼와 그 메모리를 해제합니다.

## 정리

애플리케이션이 종료될 때 텍스처 이미지와 메모리도 해제해야 합니다. `cleanup` 함수를 수정하세요.

```rust
impl VulkanApp {
    fn cleanup(&mut self) {
        unsafe {
            // ...
            self.device.destroy_image(self.texture_image, None);
            self.device.free_memory(self.texture_image_memory, None);
            // ...
        }
    }
}
```

이제 이미지가 텍스처를 포함하게 되었지만, 아직 그래픽 파이프라인에서 접근할 방법이 없습니다. 다음 장에서 이 부분을 다루겠습니다.