---
title: "Simple Auto Exposure"
permalink: /auto-exposure/
---
<h1 align="center">Auto Exposure</h1>

<h2 align="center">Preamble</h2>

We treat auto-exposure as multiplier on the raw lighting pass in our game, which can brighten or darken our scene. This is to emulate our eyes and cameras which adapt to different lighting environments, in a dark room we can see more after our eyes adjust. In terms of the game it makes your artist's life much easier when it comes to lighting their scenes. Especially useful for dynamic enviornments like day/night cycles and weather effects.

<h2 align="center">Quickly Explain</h2>

To detect the brightness of a scene we take the average luminance of the HDR colour buffer (output of our lighting pass) using a compute shader that sums up each texel's luminance. This is done in two passes, the first dispatch runs through each pixel, it is optimised by working on a 2x2 tile and sampling the 2x2 texel corners using bilinear filtering to get an effective 4x4 sample in total. Each sample's luminance is calculated and converted from a float into a uint to make later use of InterlockedAdd. These 4x4 uint luminance values are summed and stored in groupshared memory, and GroupMemoryBarrierWithGroupSync() is called to guarentee all threads have calculated this value. Finally using prefix sum all the groupshared values are summed together, which is a common optimisation in compute shaders. 

These results are stored output into a 16x16 texture using InterlockedAdd(), which can only be used with uints. Even though we have more that 16x16 values to store we just wrap around the output texture, as all we care about is summing values.

The second pass is very similar
