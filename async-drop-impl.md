# The asynchronous drop code generation implementation

The implementation is WIP and works with only final async drop function (async drop glue preparation in separate branch).

## Public interface of AsyncDrop

```rust
#[lang = "async_drop"]
pub trait AsyncDrop {
    #[allow(async_fn_in_trait)]
    async fn drop(self: Pin<&mut Self>);
}

impl Drop for Foo {
    fn drop(&mut self) {
        println!("Foo::drop({})", self.my_resource_handle);
    }
}

impl AsyncDrop for Foo {
    async fn drop(self: Pin<&mut Self>) {
        println!("Foo::async drop({})", self.my_resource_handle);
    }
}
```

## Implementation details

Implementation of async drop requires changes:
* in compiler/rustc_mir_build/src/build/scope.rs to setup drop entry to dropline (coroutine drop path) for async drop terminator.
* in compiler/rustc_mir_dataflow/src/elaborate_drops.rs to resolve async drop implementation and prepare the corresponding future.
* in compiler/rustc_mir_transform/src/coroutine.rs to expand async drop into one or two yield points (two - for dropline transition) and generate async drop function of coroutine itself.
