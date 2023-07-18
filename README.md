# WGPU-Async

[WGPU](https://github.com/gfx-rs/wgpu) offers some `async` methods when initialising adapters and devices, but during program execution much of the timing between the CPU and GPU is managed through callbacks and polling. A common pattern is to do something like the following:

```rust compile_fail
wgpu.do_something();
wgpu.on_something_done(|result| { /* Handle results */ });
wgpu.poll();
```

This is a very JavaScript-esque pattern, while in Rust we might expect to write code that looks more like:

```rust compile_fail
let result = wgpu.do_something().await;
```

Or, if we still wanted a callback:

```rust compile_fail
wgpu.do_something().then(|result| { /* Handle results */ }).await;
```

This crate adds a global poll loop thread on non-WASM platforms that can be used to create a `WgpuFuture` holding the completion of a task. The poll loop is conservative, parking itself when no futures are waiting on it, meaining that this crate adds little to no overhead in changing paradigms.

Note that this crate does not aim to improve the performance of anything, and fast applications should reduce CPU-GPU communication and synchronisation as much as possible, irrespective of the paradigm used.

## Usage

To do things in an `async` way, your `wgpu::Device` and `wgpu::Queue` need to be wrapped in async smart-pointer versions. These implement `Deref<Device>` and `Deref<Queue>`, so can be used as a slot-in replacement for existing wgpu code. 

```rust compile_fail
// Create a device and queue like normal
let instance = wgpu::Instance::new(wgpu::InstanceDescriptor::default());

let adapter = instance
    .request_adapter(&wgpu::RequestAdapterOptions {
        power_preference: wgpu::PowerPreference::HighPerformance,
        compatible_surface: None,
        force_fallback_adapter: true,
    })
    .await
    .expect("missing adapter");

let (device, queue) = adapter
    .request_device(
        &wgpu::DeviceDescriptor {
            features: wgpu::Features::empty(),
            limits: adapter.limits(),
            label: None,
        },
        None,
    )
    .await
    .expect("missing device");

// Make them async
let (device, queue) = wgpu_async::wrap(Arc::new(device), Arc::new(queue));
```

Then you can use shadowed `wgpu` methods with the exact same signatures, but with extra `async`-ness:

```rust compile_fail
queue.submit(&[/* commands */]).await; // An awaitable `Queue::submit`!
```

Just like their base `wgpu` counterparts, these methods begin their work on the GPU immediately. However the device won't begin to be polled until the future is awaited.

You can also convert any non-shadowed callback-and-poll method to an async one using `AsyncDevice::do_async`:

```rust compile_fail
wgpu.do_something();
let future = device.do_async(move |callback| {
    wgpu.on_something_done(|result| callback(result));
});

let result = future.await;
```