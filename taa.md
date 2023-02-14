---
title: "Notes on TAA"
permalink: /taa/
---
<h1 align="center">Temporal Anti-aliasing</h1>

<h2 align="center">Preamble</h2><hr>

One of my first tasks as a graphics programmer was to improve their TAA implementation. The main issue was constant flickering on stationary camera shots. I started with zero knowledge and no one at the company had much information, the shader was almost a decade old. The first thing I did was read through the shader to try and understand where the flickering was occuring and see if I could just tweak a value to reduce it. Probably not the best first step, I quickly realised I had a lot of research to do, and that most of this job is actually research!

<h2 align="center">Quickly Explain</h2><hr>

TAA is essentially jiggling the camera every frame, then blurring/blending them together to produce a smooth looking image. Aliased pixels are where geometry detail is unable to be represented in the low resolution screen. It lands between pixels (subpixel) and cannot be perfectly represented, causing a jagged edges look. By jiggling (or jittering) the camera these subpixel jagged edges will snap back and forth between different pixel locations. By blending these subpixel movements the image will blur these edges over time, resulting in a smooth image.

<h2 align="center">Cheat Sheet</h2><hr>

Just want the answers? Every link I think is useful and a quick glossary. In retrospect it is easy to find this info, but at the time without a working lexicon it is difficult.

<h3 align="center">Glossary</h3><hr>

<a align="center">
  <li> Current (frame) - the newly rendered aliased image, probably the output of our geometry/lighting passes.</li>
  <li>History (frame) - a texture containing the previous frame's TAA output. A smooth image we sample to compare against our current frame.</li>
  <li>Reprojection - calculating where the current pixel was in the previous frame (history).</li>
  <li>Subpixel - the place between the discrete pixels on of the screen.</li>
  <li>Jitter - moving the camera's translation, usually by a subpixel amount.</li>
  <li>Ghosting - when TAA blends too much during camera movement, causing big smears across the screen.</li>
  <li>Neighbourhood - the surround pixels of the current pixel we are rendering (usually the current frame).</li>
  <li>Neighbourhood Clamping - clamping the history colour into the colour bounds (min/max) of the current neighbourhood. Reduces ghosting.</li>
  <li>Neighbourhood Clipping - instead of just clamping to the bounds, drag the the history colour towards the current colour until it is within the bounding box. Reduce ghosting further.</li>
  <li>Varience Clipping - build a more precise colour bounds to clip by using the mean and the variance of the current neighbourhood. Reduces ghosting EVEN further.</li>
</a>

<ul>
  <li><a href="https://www.youtube.com/watch?v=2XXS5UyNjjU" target="_blank">Temporal Reprojection Anti-Aliasing in Inside</a>: this a great overview and implementation, but a bit stylized since the game had a very unique aesthetic.</li>
  <li><a href="https://de45xmedrsdbp.cloudfront.net/Resources/files/TemporalAA_small-59732822.pdf" target="_blank">Unreal Engine 4's TAA presentation</a>: more focused on refining TAA for higher quality results through better HDR management and neighbourhood clipping.</li>
  <li><a href="https://developer.download.nvidia.com/gameworks/events/GDC2016/msalvi_temporal_supersampling.pdf" target="_blank">Nvidia TAA Overview</a>: nice overview, with a GREAT look at varience clipping!</li>
</ul>


<h2 align="center">To be continued</h2><hr>

