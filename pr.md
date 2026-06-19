# Add `allows ref struct` constraint to `TBufferWriter` for .NET 9+

## The Problem
When using `MemoryPack` with highly optimized, custom memory allocators (like native/unmanaged memory arenas), implementing `IBufferWriter<byte>` is required. However, `IBufferWriter<T>.GetMemory()` mandates returning a `Memory<byte>`, which cannot wrap a raw pointer without allocating a heap-based `MemoryManager<T>` per buffer. Since `MemoryPackWriter` never actually calls `GetMemory()` (it strictly uses `GetSpan()`), this forces an unnecessary allocation just to satisfy the interface.

## The Solution
With C# 13 and .NET 9, we can use the `allows ref struct` anti-constraint on the `TBufferWriter` generic parameter. By applying this constraint to `MemoryPackWriter<TBufferWriter>` and all related core interfaces under `#if NET9_0_OR_GREATER`, users can implement their custom `IBufferWriter<byte>` as a `ref struct`. This allows them to throw `NotSupportedException` in `GetMemory()` while completely avoiding heap allocations for native memory buffers.

## The Workaround (CS9050)
During implementation, updating `MemoryPackWriter<TBufferWriter>` to store the `TBufferWriter` as a `ref` field results in compiler error **CS9050: A ref field cannot refer to a ref struct**. Even though `MemoryPackWriter` itself is a `ref struct`, C# currently forbids declaring a `ref T` field if `T` is an anti-constraint (`allows ref struct`).

To bypass this safely while maintaining zero-allocation performance, the internal field was mapped to a `ref byte`:
```csharp
#if NET9_0_OR_GREATER
    ref byte bufferWriterRef;
    ref TBufferWriter bufferWriter => ref Unsafe.As<byte, TBufferWriter>(ref bufferWriterRef);
#endif
```
Since `MemoryPackWriter` is a `ref struct`, the GC's tracking of `ref byte` is identical to tracking `ref TBufferWriter`. The memory is guaranteed not to escape the stack/pinned context.

## Breaking Changes
This change is wrapped in `#if NET9_0_OR_GREATER` compilation directives, so existing .NET 7 and .NET 8 users are completely unaffected.

For .NET 9 users, this introduces a **source-breaking change** for custom formatters. Anyone who has written a custom formatter by inheriting from `MemoryPackFormatter<T>` or using `[MemoryPackOnSerializing]` / `[MemoryPackOnSerialized]` with a `TBufferWriter` parameter will need to add the `allows ref struct` constraint to their method signatures to match the updated base class/interface.

**Note:** The MemoryPack Source Generator does **not** need updates, as it emits explicit interface implementations (`static void IMemoryPackable<T>.Serialize<TBufferWriter>(...)`), which implicitly inherit the constraints from the interface.
