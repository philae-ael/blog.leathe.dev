---
title: Some memory things with vulkan
date: 2024-10-28
draft: true
layout: post
---


## Motivation
Understand how memory works under Vulkan. This also allows for better memory management.

Storage -> Memory -> Decompression -> GPU

Building an asset pipeline maybe ?

Authoring -> Preprocessing -> Storage -> Memory -> Decompression -> GPU

With Direct Storage:
Authoring -> Preprocessing -> Storage ->  Decompression -> GPU

https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_EXT_external_memory_dma_buf.html
=> Supported only on pro cards, it seems

https://gpuopen.com/compressonator/

API Idea:
VkStorageQueue
-> Has a transfer queue behind
Take a FD / Pointer -> Copy memory to 
+ Possibly can decompress the memory
+ Should choose between a staging buffer and a direct copy to host memory
=> Allow for DMA Or using directly Resizable Bar
See https://github.com/KhronosGroup/Vulkan-Docs/issues/1667
For Decompression See https://docs.vulkan.org/spec/latest/chapters/VK_NV_memory_decompression.html
And https://gpuopen.com/brotlig-sdk/ and GDeflate

---

```
VkCmdCopyLocalMemoryToBuffer(cmd, src*)
VkCmdCopyFileToBuffer(cmd, fd, offset, size);
```
-> This is just a direct write to buffer memory
-> Requires Buffer's memory to be VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT
+ Add a pipeline Barrier 
Which io POLLING!
Requires IO Polling See libUV, libEvent or epoll or ??? what are Godot unreal using?

J'en reviens encore et toujours à l'asynchrone!
Alors visiblement on a toujours le même soucis si on a un staging buffer
=> We can't have finite memory on host side 
=> Without ReBAR only 256MB SO! We need do use a staging buffer
But this staging buffer can be filled -> 
Thus we must have a queue memory to upload

NAH This API doesn't look nice after trying do that.
I should drop that idea


--- 


Let's think

With staging buffer 
Way 1 Load from storage:
If we upload 8Go of Data with a staging buffer of 128Mb:

Fill staging buffer ~ 7Go/s NVME => 18ms
Upload to device ~ 30Go/s PCIE 5.0 => 4ms
22ms
Fill staging buffer ~ 7Go/s 
Upload
...
1.3s ~ !

If we upload 8Go of Data with a staging buffer of 256Mb:
Fill staging buffer ~ 7Go/s NVME => 36ms
Upload to device ~ 30Go/s PCIE 5.0 => 8ms
44ms
...
1.3s SAME! 

If we upload 8Go of Data with 2 staging buffers of 128Mb:
Fill staging buffer ~ 7Go/s NVME => 35ms   |  Upload to device ~ 30Go/s PCIE 5.0 => 8ms
 Upload to device ~ 30Go/s PCIE 5.0 => 8ms | Fill staging buffer ~ 7Go/s NVME => 35ms
35ms per 
...
1.1s! -15%!
=> Note this is the best time possible as we are limited by the thoughput of the NVME!


TODO: 
A table of PCIE 5.0 vs PCIE 4.0 bandwidth speed

The stage of the staging buffer does not seem to matter but it matter 
Because there is CPU overhead i think


So how to make it work:
Circular buffer of size 2*N With N well chosen
How it does it's thing
=> Read N bytes 
=> Setup and Upload of the N bytes 
=> Read N bytes while the N first bytes are being read...
=> And so on!


This can be further improved!

LoadData 
Decompress on CPU 
Upload On Gpu  | LoadData
Decompress on CPU 
Upload On Gpu  | LoadData

But but but 
JPEG Is SLOOOWWWWW to decode 
I found somewhere that JPEG decode speed is ~100Mb/s (probably less than that with stb_image for decoding)
-> 80s ! For 8Go (with only jpeg data)

How to solve that ? Compressed Data Format baby!

I remember that vulkan has compressed format and like the BCn formats and ASTC ones also
Issue: How to deal with format compatibility?
ASTC seems more AMDy and 

A quick look at meshoptimizer (gltfpack) seems to use KTX2 with BasisU supercompression
And a quick googling learns me that it seems like what i need
And the header of the paper seems like 

I could also chose a given format and use that
Idk


Direct Storage seems to be a big scam 
- direct loading of data Storage -> GPU then decompression on GPU

OK for the GPU decompression but the Storage -> GPU is not new?!
Before: 
Using a Staging buffer (BAR) chunks of up to 256Mb of data could be transfered 

Using ReBAR we can stream all the data 

Or Direct Storage was the psyop of Microsoft to force developpers to not do 
Storage -> RAM  -> (decompress) -> RAM -> GPU ?

but do 
Storage -> GPU -> (decompress) -> GPU???

But to be fair it doesn't change much as seen above except maybe so that people use a better compression format?

So it's just a psyop that does... Nothing good?

It seems that Microsoft communication was bullshit (so was Nvidia's i think)

[This post](https://www.reddit.com/r/Games/comments/tlfqdk/clearing_up_misconceptions_about_directstorage/) Seems to give a response !
It's a psyop so that developer use better way than the C file api for handling a lot of files 

This confirms that:
- https://www.youtube.com/watch?v=zolAIEH0n1c
- https://www.youtube.com/watch?v=vz_2kXeChCw

Ok So direct storage is a push for async io?? (by batching)
So it's only a dumbed down version of libuv on which you glue a staging buffer

Not sure of myself on that...

## Exploration

Maybe! 
Exploration of every single line of code to upload data to a vulkan buffer 
From in memory to the GPU

- https://www.reddit.com/r/vulkan/comments/kkmu5z/resizable_bar_staging_buffers/
 Very goof remarks from smart people
- [Enable ReBAR on RTX30 in linux](https://forums.developer.nvidia.com/t/enabling-resizable-bar-on-rtx-30-series-gpus-in-linux/239950)
- AMD 
- https://www.kernel.org/doc/html/latest/gpu/introduction.html#simple-drm-drivers-to-use-as-examples
- https://gpuopen.com/presentations/2022/gpuopen-gpu_submission-reboot_blue_2022.pdf
- https://gpuopen.com/gdc-presentations/2023/GDC-2023-DirectStorage-optimizing-load-time-and-streaming.pdf

- NVK source + NVIDIA Open Driver
