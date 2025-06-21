## 소개

지금까지 우리가 만든 애플리케이션은 성공적으로 삼각형을 그리지만, 아직 제대로 처리하지 못하는 몇 가지 상황이 있습니다. 윈도우 서피스가 변경되어 스왑 체인이 더 이상 호환되지 않는 경우가 발생할 수 있습니다. 이런 상황이 발생하는 원인 중 하나는 윈도우 크기가 변경되는 것입니다. 우리는 이러한 이벤트를 감지하고 스왑 체인을 다시 만들어야 합니다.

## 스왑 체인 재구성

`recreate_swapchain`이라는 새로운 함수를 만들어, `create_swapchain`과 스왑 체인 또는 윈도우 크기에 의존하는 모든 객체들의 생성 함수를 호출하도록 합시다.

```rust
fn recreate_swapchain(&mut self) -> Result<(), Box<dyn std::error::Error>> {
    unsafe {
        self.device.device_wait_idle()?;
    }

    self.create_swapchain()?;
    self.create_image_views()?;
    self.create_framebuffers()?;

    Ok(())
}
```

먼저 `device_wait_idle`을 호출하는데, 이는 이전 장에서와 마찬가지로 아직 사용 중일 수 있는 리소스에 접근해서는 안 되기 때문입니다. 이 함수는 `unsafe` 블록 안에서 호출해야 합니다. 당연히 스왑 체인 자체를 다시 만들어야 합니다. 이미지 뷰는 스왑 체인 이미지에 직접 기반하므로 다시 만들어야 합니다. 마지막으로, 프레임버퍼는 스왑 체인 이미지에 직접 의존하므로 다시 만들어야 합니다.

이러한 객체들의 이전 버전이 재생성되기 전에 확실히 정리되도록, 일부 정리 코드를 별도의 함수로 옮겨 `recreate_swapchain` 함수에서 호출하도록 합시다. 이 함수를 `cleanup_swapchain`이라고 부르겠습니다.

```rust
fn cleanup_swapchain(&mut self) {
    unsafe {
        self.framebuffers
            .iter()
            .for_each(|&framebuffer| self.device.destroy_framebuffer(framebuffer, None));

        self.swapchain_image_views
            .iter()
            .for_each(|&view| self.device.destroy_image_view(view, None));

        self.swapchain_loader
            .destroy_swapchain(self.swapchain, None);
    }
}

fn recreate_swapchain(&mut self) -> Result<(), Box<dyn std::error::Error>> {
    unsafe {
        self.device.device_wait_idle()?;
    }

    self.cleanup_swapchain();

    self.create_swapchain()?;
    self.create_image_views()?;
    self.create_framebuffers()?;

    Ok(())
}
```

여기서는 간단하게 하기 위해 렌더 패스를 다시 만들지 않는다는 점에 유의하세요. 이론적으로는 애플리케이션 실행 중에 스왑 체인 이미지 포맷이 변경될 수 있습니다. 예를 들어, 표준 다이나믹 레인지(SDR) 모니터에서 하이 다이나믹 레인지(HDR) 모니터로 창을 옮기는 경우가 그렇습니다. 이 경우 다이나믹 레인지 간의 변경이 올바르게 반영되도록 애플리케이션이 렌더 패스를 다시 만들어야 할 수도 있습니다.

스왑 체인 갱신의 일부로 재생성되는 모든 객체들의 정리 코드를 `cleanup` (Rust에서는 `Drop` 트레이트 구현)에서 `cleanup_swapchain`으로 옮기겠습니다. Rust에서는 리소스 해제를 위해 `Drop` 트레이트를 구현하는 것이 일반적입니다.

```rust
impl Drop for HelloTriangleApplication {
    fn drop(&mut self) {
        unsafe {
            self.device.device_wait_idle().unwrap();

            self.cleanup_swapchain();

            self.device.destroy_pipeline(self.graphics_pipeline, None);
            self.device
                .destroy_pipeline_layout(self.pipeline_layout, None);

            self.device.destroy_render_pass(self.render_pass, None);

            for i in 0..MAX_FRAMES_IN_FLIGHT {
                self.device
                    .destroy_semaphore(self.render_finished_semaphores[i], None);
                self.device
                    .destroy_semaphore(self.image_available_semaphores[i], None);
                self.device.destroy_fence(self.in_flight_fences[i], None);
            }

            self.device.destroy_command_pool(self.command_pool, None);

            self.device.destroy_device(None);

            if VALIDATION.is_enable {
                self.debug_utils_loader
                    .destroy_debug_utils_messenger(self.debug_messenger, None);
            }

            self.surface_loader.destroy_surface(self.surface, None);
            self.instance.destroy_instance(None);
        }
    }
}
```
`choose_swap_extent`에서는 이미 새로운 윈도우 해상도를 조회하여 스왑 체인 이미지가 (새로운) 올바른 크기를 갖도록 하고 있으므로, `choose_swap_extent`를 수정할 필요는 없습니다 (스왑 체인을 만들 때 이미 서피스의 해상도를 픽셀 단위로 얻기 위해 `window.get_framebuffer_size()`를 사용해야 했던 것을 기억하세요).

이것만으로도 스왑 체인을 재구성할 수 있습니다! 하지만 이 방법의 단점은 새로운 스왑 체인을 만들기 전에 모든 렌더링을 중단해야 한다는 것입니다. 이전 스왑 체인의 이미지에 대한 그리기 명령이 아직 실행 중인 상태에서 새로운 스왑 체인을 만드는 것도 가능합니다. 그러려면 `vk::SwapchainCreateInfoKHR` 구조체의 `old_swapchain` 필드에 이전 스왑 체인을 전달하고, 이전 스왑 체인 사용이 끝나는 즉시 파괴해야 합니다.

## 준최적(Suboptimal) 또는 오래된(out-of-date) 스왑 체인

이제 스왑 체인 재구성이 언제 필요한지 파악하고 새로운 `recreate_swapchain` 함수를 호출하기만 하면 됩니다. 다행히도 Vulkan은 보통 프레젠테이션 중에 스왑 체인이 더 이상 적합하지 않다고 알려줍니다. Ash 라이브러리는 `acquire_next_image`와 `queue_present` 함수에서 `Result` 타입을 반환하여 이를 명확하게 처리합니다.

*   `Err(vk::Result::ERROR_OUT_OF_DATE_KHR)`: 스왑 체인이 서피스와 호환되지 않게 되어 더 이상 렌더링에 사용할 수 없습니다. 보통 윈도우 리사이즈 후에 발생합니다.
*   `Ok((_image_index, true))`: `acquire_next_image`에서 반환되는 튜플의 두 번째 값이 `true`이면 스왑 체인이 준최적(suboptimal) 상태임을 의미합니다. `queue_present`에서는 `Ok(true)`가 준최적 상태를 나타냅니다.

```rust
let result = unsafe {
    self.swapchain_loader.acquire_next_image(
        self.swapchain,
        u64::MAX,
        self.image_available_semaphores[self.current_frame],
        vk::Fence::null(),
    )
};

let image_index = match result {
    Ok((image_index, _is_suboptimal)) => image_index,
    Err(vk::Result::ERROR_OUT_OF_DATE_KHR) => {
        self.recreate_swapchain()?;
        return Ok(()); // 다음 프레임에서 다시 시도
    }
    Err(error) => return Err(error.into()),
};
```

`acquire_next_image`가 `ERROR_OUT_OF_DATE_KHR` 오류를 반환하면 더 이상 현재 스왑 체인으로 프레젠테이션을 할 수 없습니다. 따라서 즉시 스왑 체인을 재구성하고 `draw_frame`을 종료하여 다음 루프에서 다시 시도해야 합니다. Ash의 `acquire_next_image`는 성공 시 `(u32, bool)` 튜플을 반환하는데, 두 번째 `bool` 값은 준최적 여부를 나타냅니다. 일단은 이미지를 성공적으로 획득했으므로 준최적 상태는 무시하고 진행합니다.

```rust
let present_info = vk::PresentInfoKHR::builder()
    // ...
    ;

let result = unsafe { self.swapchain_loader.queue_present(self.present_queue, &present_info) };

let is_resized = match result {
    Ok(true) | Err(vk::Result::ERROR_OUT_OF_DATE_KHR) => {
        self.framebuffer_resized = false;
        self.recreate_swapchain()?;
    }
    Err(e) => return Err(e.into()),
    _ => {}
};

self.current_frame = (self.current_frame + 1) % MAX_FRAMES_IN_FLIGHT;
```

`queue_present` 함수도 `Result`를 반환합니다. `Ok(true)`는 준최적 상태를, `Err(vk::Result::ERROR_OUT_OF_DATE_KHR)`는 오래된 상태를 의미합니다. 두 경우 모두 최상의 결과를 위해 스왑 체인을 재구성합니다.

## 데드락 해결하기

지금 코드를 실행하면 데드락이 발생할 수 있습니다. 코드를 디버깅해보면, 애플리케이션이 `wait_for_fences`에 도달한 후 더 이상 진행하지 못하고 멈추는 것을 발견할 수 있습니다. 이는 `acquire_next_image`가 `ERROR_OUT_OF_DATE_KHR`를 반환할 때, 우리가 스왑 체인을 재구성한 후 `draw_frame`에서 즉시 반환하기 때문입니다. 하지만 그 전에, 현재 프레임의 펜스는 대기 상태에 들어간 후 리셋되었습니다. 우리가 즉시 반환하므로 아무 작업도 제출되지 않고, 따라서 펜스는 절대 신호를 받지 못하게 되어 `wait_for_fences`가 영원히 멈추게 됩니다.

다행히 간단한 해결책이 있습니다. 펜스를 리셋하는 것을, 우리가 확실히 작업을 제출할 것이라는 것을 안 이후로 미루는 것입니다. 이렇게 하면, 우리가 일찍 반환하더라도 펜스는 여전히 신호를 받은 상태(signaled)로 남아있어, 다음에 같은 펜스 객체를 사용할 때 `wait_for_fences`가 데드락을 일으키지 않을 것입니다.

이제 `draw_frame` 함수의 시작 부분은 다음과 같아야 합니다:
```rust
unsafe {
    self.device.wait_for_fences(
        &[self.in_flight_fences[self.current_frame]],
        true,
        u64::MAX,
    )?;
}

let result = unsafe {
    self.swapchain_loader.acquire_next_image(
        self.swapchain,
        u64::MAX,
        self.image_available_semaphores[self.current_frame],
        vk::Fence::null(),
    )
};

let image_index = match result {
    Ok((image_index, _is_suboptimal)) => image_index,
    Err(vk::Result::ERROR_OUT_OF_DATE_KHR) => {
        self.recreate_swapchain()?;
        return Ok(()); // 다음 프레임에서 다시 시도
    }
    Err(error) => return Err(error.into()),
};

// 작업을 제출할 때만 펜스를 리셋합니다.
unsafe {
    self.device
        .reset_fences(&[self.in_flight_fences[self.current_frame]])?;
}
```
## 명시적으로 리사이즈 처리하기

많은 드라이버와 플랫폼이 윈도우 리사이즈 후 자동으로 `ERROR_OUT_OF_DATE_KHR`를 발생시키지만, 이것이 보장되지는 않습니다. 그래서 우리는 리사이즈를 명시적으로 처리하는 코드를 추가할 것입니다. 먼저 구조체에 리사이즈가 발생했음을 알리는 플래그 멤버 변수를 추가합니다:

```rust
struct HelloTriangleApplication {
    // ...
    in_flight_fences: Vec<vk::Fence>,
    framebuffer_resized: bool,
    // ...
}
```

`draw_frame` 함수를 이 플래그도 확인하도록 수정해야 합니다. `queue_present` 후의 로직에 이 플래그를 함께 검사합니다.

```rust
let present_result = unsafe { self.swapchain_loader.queue_present(self.present_queue, &present_info) };

let suboptimal = match present_result {
    Ok(suboptimal) => suboptimal,
    Err(vk::Result::ERROR_OUT_OF_DATE_KHR) => true, // 오래된 경우도 리사이즈로 취급
    Err(e) => return Err(e.into()),
};

if suboptimal || self.framebuffer_resized {
    self.framebuffer_resized = false;
    self.recreate_swapchain()?;
}
```

이제 실제로 리사이즈를 감지하기 위해 `glfw` 크레이트를 사용하여 콜백을 설정할 수 있습니다. Rust에서 FFI(Foreign Function Interface)를 통해 C 라이브러리 콜백을 다루는 것은 `unsafe` 코드를 필요로 합니다.

```rust
// init_window 함수 내에서
// 'self'에 대한 포인터를 윈도우의 user data로 저장합니다.
self.window.set_user_data(self as *mut _ as *mut c_void);
self.window.set_framebuffer_size_callback(framebuffer_resize_callback);

// ...

// 애플리케이션 구조체 밖의 static 함수
extern "C" fn framebuffer_resize_callback(
    window: *mut glfw::ffi::GLFWwindow,
    _width: i32,
    _height: i32,
) {
    unsafe {
        let app_ptr = glfw::ffi::glfwGetWindowUserPointer(window) as *mut HelloTriangleApplication;
        if !app_ptr.is_null() {
            (*app_ptr).framebuffer_resized = true;
        }
    }
}
```

C++ 예제와 마찬가지로, `glfw`는 Rust의 멤버 함수를 직접 호출하는 방법을 모르기 때문에, `static` 함수(Rust에서는 `extern "C" fn`)를 사용합니다. `set_user_data`를 통해 `self`의 포인터를 윈도우에 저장하고, 콜백 함수 내에서 `glfwGetWindowUserPointer`로 다시 가져와 애플리케이션의 `framebuffer_resized` 플래그를 설정합니다. 이 과정은 `unsafe` 블록 안에서 수행되어야 합니다.

이제 프로그램을 실행하고 윈도우 크기를 조절하여 프레임버퍼가 윈도우에 맞게 올바르게 리사이즈되는지 확인해 보세요.

## 창 최소화 처리하기

스왑 체인이 오래될 수 있는 또 다른 경우는 특별한 종류의 윈도우 리사이즈인 창 최소화입니다. 이 경우는 프레임버퍼 크기가 `(0, 0)`이 되기 때문에 특별합니다. 이 튜토리얼에서는 윈도우가 다시 전경에 올 때까지 일시 중지하는 방식으로 이 문제를 처리할 것입니다. `recreate_swapchain` 함수를 다음과 같이 확장합니다:

```rust
fn recreate_swapchain(&mut self) -> Result<(), Box<dyn std::error::Error>> {
    let (mut width, mut height) = self.window.get_framebuffer_size();
    while width == 0 || height == 0 {
        (width, height) = self.window.get_framebuffer_size();
        self.glfw.wait_events();
    }

    unsafe {
        self.device.device_wait_idle()?;
    }
    
    self.cleanup_swapchain();
    self.create_swapchain()?;
    self.create_image_views()?;
    self.create_framebuffers()?;

    Ok(())
}
```
초기 `get_framebuffer_size` 호출은 이미 크기가 올바르고 `wait_events`가 기다릴 것이 없는 경우를 처리합니다.

축하합니다, 여러분은 이제 최초의 잘 동작하는(well-behaved) Rust-Vulkan 프로그램을 완성했습니다! 다음 장에서는 버텍스 셰이더에 하드코딩된 정점들을 제거하고 실제로 정점 버퍼(vertex buffer)를 사용할 것입니다.

[Rust 코드](https://github.com/vulkan-tutorial-rs/vulkan-tutorial-rs-code-new/blob/main/src/17_swap_chain_recreation.rs) /
[버텍스 셰이더](/code/09_shader_base.vert) /
[프래그먼트 셰이더](/code/09_shader_base.frag)