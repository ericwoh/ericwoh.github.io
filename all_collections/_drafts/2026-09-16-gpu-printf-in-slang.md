---
layout: post
title: GPU printf and debug lines from Slang shaders
date: 2026-09-16
categories: [graphics, slang, vulkan, tooling]
---

<!--
Draft. Lives in all_collections/_drafts/ so it doesn't publish; `jekyll serve --drafts` to preview.
To publish: move to all_collections/_posts/ and set the date to the publish day.
-->

Why I wanted `GDD::Printf("light %d w=%f", iLight, w)` to work from inside a compute shader, and what it took.

<!-- TODO: one-paragraph hook. The pain: shader reporting only worked with validation layers on (Debug Printf),
     which distorts perf, floods a shared buffer, can't carry a world position, and can't draw anything. -->

## What it looks like in use

```hlsl
GDD::DrawLine(posWorld, posWorldPrev, float3(1, 0, 0));
GDD::Printf(posWorld, "reservoir W=%f M=%d", W, M);   // world-anchored text
GDD::Printf("dropped %u bytes", cBDropped);            // console
```

<!-- TODO: screenshot of a line + label at the mouse pixel. -->

## Prior art

<!-- TODO: MJP's GPU printf post is the only detailed implementation write-up I found; Unity / Blender / others
     have the feature but not the details. Link MJP, note what's the same and what's different here. -->

## The pipeline

1. Shader writes packets into a GPU buffer (bump allocator via `InterlockedAdd` on byte 0).
2. Host memory barrier before `vkEndCommandBuffer`, readback ring of `g_cFrameRender` buffers.
3. CPU walks packets: lines go to the debug draw manager, text is formatted and handed to ImGui.

<!-- TODO: small diagram. -->

## The readback buffer

<!-- TODO:
- VMA_ALLOCATION_CREATE_HOST_ACCESS_RANDOM_BIT, not SEQUENTIAL_WRITE (write-combined memory is ~100x slower to
  *read* — easiest thing to get wrong in the whole feature).
- MAPPED_BIT; invalidate before reading, flush after resetting the cursor.
- Why the VkMemoryBarrier2 to HOST_BIT / HOST_READ_BIT is needed at all.
- Fence-wait -> invalidate -> read -> reset -> flush ordering.
- Alternative (MJP): device-local buffer + vkCmdCopyBuffer + vkCmdFillBuffer reset; tradeoffs.
-->

## Getting a buffer to every shader without plumbing

<!-- TODO: fixed slot in the bindless set 0 (bound for every shader anyway) vs BDA pointer. The static
     `g_pGloshdd` pointer + `InitGlobals()` at entry for the frame index. The push-constant alias dead end
     (spirv-val: one PushConstant interface per entry point). -->

## Packets

<!-- TODO: PacketHeader / PacketText / PacketFormat* / PacketLine. Byte address buffer + Store<T>. Overflow: let
     the counter run past the end, write a Nil header if it fits, count dropped bytes on the CPU. -->

## Variadic `Printf` in Slang

The part I couldn't find written up anywhere.

```hlsl
interface IPrintfArg { u32 CB(); void WriteAsPacket(inout u32 iB); };
extension int   : IPrintfArg { ... }
extension float : IPrintfArg { ... }

void Printf<each T : IPrintfArg>(float3 pos, String strFmt, expand each T args)
{
    u32 cB = sizeof(PacketText);
    expand cB += (each args).CB();
    ...
    expand (each args).WriteAsPacket(iBOut);
}
```

<!-- TODO, the things that bit me:
- the pack must be interface-constrained; bare `<each T>` fails overload resolution
- one `each` per `expand`; the comma operator breaks it
- `countof(args)`, not `sizeof` (that's bytes)
- default member initializers are NOT applied on a bare `T x;` declaration — set the header explicitly
- `getStringHash` on a `String` param works through the call chain
-->

## Recovering the strings on the CPU

<!-- TODO: `IModule::getLayout()` -> `getHashedStringCount()` / `getHashedString()` at compile time,
     `spComputeStringHash` for the key, HashMap<u32, String>. Harvest before the module is released. -->

## Formatting and drawing

<!-- TODO: walking the format string, matching %d/%u/%x/%f/%s to packet types, bailing loudly on mismatch.
     ImGui world-space text: project with the *current* frame's view-proj, `posH.w <= 0` cull, viewport
     anchoring with multi-viewport enabled. Vulkan y-down NDC gotcha. -->

## What I'd do differently

<!-- TODO -->

## Closing

<!-- TODO: total latency (~3 frames), cost when idle, what it unlocked (validating motion vectors in one glance
     before ReSTIR depends on them). -->
