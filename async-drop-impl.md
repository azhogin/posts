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

### scope.rs changes

Drop terminator for async drop is now a hidden Yield, so we need to prepare `drop` successor for Drop terminator in the similar way (as for Yield).
`drop` target block for Yield or async Drop terminator means "where to go, when this coroutine is asked to drop, to perform all necessary internal drops".
```rust
Drop {
        place: Place<'tcx>,
        target: BasicBlock,
        unwind: UnwindAction,
        replace: bool,
        /// Cleanup to be done if the coroutine is dropped at this suspend point (for async drop).
/* =>*/ drop: Option<BasicBlock>,
        /// Prepared async future local (for async drop)
/* =>*/ async_fut: Option<Local>,
    },
```

```rust
fn build_scope_drops<'tcx, F>(
...
    unwind_drops.add_entry(block, unwind_to);
    if let Some(to) = dropline_to && drop_data.kind == DropKind::Value && is_async_drop(local) {
        coroutine_drops.add_entry(block, to);
    }
...
```

### elaborate_drops.rs changes
For building async drop (after drop elaboration decided to really build it), we need to:
* resolve AsyncDrop::drop implementation for the `place` type
* generate pinning of `place` object
* generate AsyncDrop::drop() call to return future
* generate Drop terminator with the prepared future local in `async_fut` field

```rust
// Generates three blocks:
// * #1:pin_obj_bb:   call Pin<ObjTy>::new_unchecked(&mut obj)
// * #2:call_drop_bb: fut = call obj.<AsyncDrop::drop>()
// * #3:drop_term_bb: drop (obj, fut, ...)
// We keep async drop unexpanded to poll-loop here, to expand it later, at StateTransform -
//   into states expand.
fn build_async_drop(&mut self, bb: BasicBlock) {
```

### coroutine.rs changes

Drop terminators for async drops are expanded into Yield poll-loop.
For normal scoped drops (not yet in dropline) we need two poll-loops to organize transition to dropline:
  * first to perform loop and then go to continue block,
  * second to process coroutine drop request - continue this drop until completion and then go to dropline block.

Notes:
* Drops in cleanup blocks (unwind path) are not expanded. So we always need sync version of drop (user-provided or auto-generated).
* Drops in dropline (coroutine drop path) don't need a transition to dropline (already there).

```rust
/// Expand Drop terminator for async drops into mainline poll-switch and dropline poll-switch
fn expand_async_drops<'tcx>(
...
// poll-code:
// state_call_drop:
// #bb_pin: fut_pin = Pin<FutT>::new_unchecked(&mut fut)
// #bb_call: rv = call fut.poll() (or future_drop_poll(fut) for internal future drops)
// #bb_check: match (rv)
//  pending => return rv (yield)
//  ready => *continue_bb|drop_bb*
```

If coroutine has internal async drops, this coroutine's drop function may be only async itself.
So we need to generate async drop poll function with the same coroutine structure:
```rust
// Create a copy of our MIR and use it to create the drop shim for the coroutine
if has_async_drops {
    // If coroutine has async drops, generating async drop shim
    let mut drop_shim = create_coroutine_drop_shim_async(
        tcx, &transform, coroutine_ty, body, drop_clean, can_return, can_unwind
    );
    // Run derefer to fix Derefs that are not in the first place
    deref_finder(tcx, &mut drop_shim);
    body.coroutine.as_mut().unwrap().coroutine_drop_async = Some(drop_shim);
} else {
    // If coroutine has no async drops, generating sync drop shim
    let mut drop_shim = create_coroutine_drop_shim(tcx, &transform, coroutine_ty, body, drop_clean);
    // Run derefer to fix Derefs that are not in the first place
    deref_finder(tcx, &mut drop_shim);
    body.coroutine.as_mut().unwrap().coroutine_drop = Some(drop_shim);
}
```

Async drop shim for coroutine is bound to `future_drop_poll` LangItem function:
```rust
#[lang = "future_drop_poll"]
#[allow(unconditional_recursion)]
pub fn future_drop_poll<F>(fut: Pin<&mut F>, cx: &mut Context<'_>) -> Poll<()>
where
    F: Future,
{
    // Code here does not matter - this is replaced by the
    // real implementation by the compiler.

    future_drop_poll(fut, cx)
}
```

Example of `block_on` function with two loops: future poll loop and future drop poll loop:

```rust
fn block_on<F>(fut_unpin: F) -> F::Output
where
    F: Future,
{
    let mut fut_pin = pin!(ManuallyDrop::new(fut_unpin));
    let mut fut: Pin<&mut F> = unsafe {
		Pin::map_unchecked_mut(fut_pin.as_mut(), |x| &mut **x)
	};
    let (waker, rx) = simple_waker();
    let mut context = Context::from_waker(&waker);
    let rv = loop {
        match fut.as_mut().poll(&mut context) {
            Poll::Ready(out) => break out,
            Poll::Pending => rx.try_recv().unwrap(),
        }
    };
    loop {
        match future_drop_poll(fut.as_mut(), &mut context) {
            Poll::Ready(()) => break,
            Poll::Pending => rx.try_recv().unwrap(),
        }
    }
    rv
}
```

## Remaining problems
### How to sync the async?
Coroutine with internal async drops will have async drop itself.
But for unwind drops and for drops in sync context we need sync version of drop.
In the current implementation, `drop_in_place<T>(...)` for coroutine with async drop will link to `drop_in_place_future` with panic.

```rust
#[unstable(feature = "async_drop", issue = "none")]
#[cfg(all(not(no_global_oom_handling), not(test)))]
#[lang = "drop_in_place_future"]
fn drop_in_place_future<F>(to_drop: *mut F)
where
    F: Future,
{
    // TODO: Support poll loop for async drop of future (future_drop_poll)
    panic!("Unimplemented drop_in_place_future");
}
```

User-defined types with AsyncDrop must also implement sync Drop.
