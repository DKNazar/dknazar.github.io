---
title: "Notes on TAA"
permalink: /taa/
---
<h1 align="center">Temporal Anti-aliasing</h1>
<br>
<h2 align="center">Preamble</h2><hr>

One of my first tasks as a graphics programmer was to improve their TAA implementation. The main issue was constant flickering on stationary camera shots. I started with zero knowledge and no one at the company had much information, the shader was almost a decade old. The first thing I did was read through the shader to try and understand where the flickering was occuring and see if I could just tweak a value to reduce it. Probably not the best first step, I quickly realised I had a lot of research to do, and that most of this job is actually research!

<br>
<h2 align="center">Quickly Explain</h2><hr>

TAA is essentially jiggling the camera every frame, then blurring/blending them together to produce a smooth looking image. Aliased pixels are where geometry (and other) detail is unable to be represented on the low resolution screen. It lands between pixels (subpixel) and cannot be perfectly represented, causing jagged edges. By jiggling (or jittering) the camera a tiny subpixel amount these jagged edges will snap back and forth between different pixel locations. The results of each frame  By blending these subpixel movements every frame the image will blur these edges over time, resulting in a smooth image.

<h2 align="center">Cheat Sheet</h2><hr>

Just want the answers? Every link I think is useful and a quick glossary. In retrospect it is easy to find this info, but at the time without a working lexicon it is difficult.

<h3 align="center">Glossary</h3>

  <ul>
    <li>Current (frame) - the newly rendered aliased image, probably the output of our geometry/lighting passes.</li><br>
    <li>History (frame) - a texture containing the previous frame's TAA output. A smooth image we sample to compare against our current frame.</li><br>
    <li>Reprojection - calculating where the current pixel was in the previous frame (history).</li><br>
    <li>Subpixel - the place between the discrete pixels on of the screen.</li><br>
    <li>Jitter - moving the camera's translation, usually by a subpixel amount.</li><br>
    <li>Ghosting - when TAA blends too much during camera movement, causing big smears across the screen.</li><br>
    <li>Disocclusion - when an object moves and reveals surfaces that were not previously visible and therefore would not exist in history which can cause ghosting.</li><br>
    <li>Neighbourhood - the surround pixels of the current pixel we are rendering (usually the current frame). Eg 3x3 pixels surrounding the current.</li><br>
    <li>Neighbourhood Clamping - clamping the history colour into the colour bounds (min/max) of the current neighbourhood. Reduces ghosting.</li><br>
    <li>Neighbourhood Clipping - instead of just clamping to the bounds, drag the the history colour towards the current colour until it is within the bounding box. Reduce ghosting further.</li><br>
    <li>Varience Clipping - build a more precise colour bounds to clip by using the mean and the variance of the current neighbourhood. Reduces ghosting EVEN further.</li><br>
  </ul>

<h3 align="center">Research Links</h3>

<ul>
  <li><a href="https://www.youtube.com/watch?v=2XXS5UyNjjU" target="_blank">Playdead's Temporal Reprojection Anti-Aliasing in Inside</a>: this a great overview and implementation, but a bit stylized since the game had a very unique aesthetic.</li>
  <li><a href="https://de45xmedrsdbp.cloudfront.net/Resources/files/TemporalAA_small-59732822.pdf" target="_blank">Unreal Engine 4's TAA presentation</a>: more focused on refining TAA for higher quality results through better HDR management and neighbourhood clipping.</li>
  <li><a href="https://developer.download.nvidia.com/gameworks/events/GDC2016/msalvi_temporal_supersampling.pdf" target="_blank">Nvidia TAA Overview</a>: nice overview, with a GREAT look at varience clipping!</li>
  <li><a href="https://alextardif.com/TAA.html" target="_blank">Alex Tardif's TAA Starter Pack </a>: after understanding the basics this really ties it all together into a very simple and clean implementation.</li>
  <li><a href="https://www.elopezr.com/temporal-aa-and-the-quest-for-the-holy-trail/"> TAA and the Holy Trail</a>: a detailed and comprehensive guide to TAA with a good amount of implementation provided.</li>
</ul>


<h2 align="center">TAA Overview</h2><hr>

<h3 align="center">Core Steps</h3><hr>

So I recommend reading/watching the links above in that order but the key steps are: <br><br>
<ul>
      - Store the previous frame's TAA output in a history texture (at frame 0 it will just be a copy of the current frame)
    <br><br>
      - On the next frame move the camera by a subpixel amount so rendered geometry will shift slightly.
    <br><br>
      - Sample the history texture by calculating where the current frame pixel would be in history using the previous view projection matrix.
    <br><br>
      - Blend the current frame with the history frame by some factor like 10% just a lerp(historyColour, currentColour, 0.1).
    <br><br>
      - The still image should look smooth but moving the camera will cause ghosting/blurriness due to slow blending and disocclusions.
    <br><br>
      - So before blending we must first use Varience Neighbourhood Clipping to bring history to a similar colour to our current frame colour.
    <br><br>
      - This will massively reduce ghosting on camera movement but on individual object movements we need to be able to track their old pixel location. Before we did this with just camera data, but now we need the motion vector of the object itself.
    <br><br>
      - We do this by rendering any moving object to a velocity buffer which is the calculation of the current pixel postion vs the previous.
    <br><br>
      - So when reprojecting the current pixel we also check the velocity buffer which might contain extra movement information for the location of the history pixel.
    <br><br>
</ul>
<br>
    That is what I consider the core of TAA, the rest is somewhat more application specific.
    <br>
    <h3 align="center">Increasing Quality</h3><hr>
    
   - TAA in motion can look too blurry, to help this when sampling history using a bicubic Catmull-Rom filter (using 5 samples) can increase the sharpness of the image. <br>
    
   - Flickering is caused by many different things, and decreasing ghosting often causes more flickering which is why this is somewhat application specific (how important is having no ghosting is vs flickering).
    
   1. When you have tiny geometry on screen like a wire fence if you jitter the camera that geometry no longer exists in the next frame, but when we jitter it for the frame after that it will exist again and pop back in causing flickering. A method briefly mentioned in UE4's talk is by recording these impulses in another buffer (or alpha channel) and effectively decreasing that pixel's importance. I achieved this by increasing the acceptable colour bounds in varience clipping, it would allow more ghosting in these flickering spots causing them to blend nicely.
    
   2. Specular flickering where each frame the specular material causes very bright highlights to jitter across the edges of geometry, similar to the tiny geometry flickering. These can have very high HDR values which is difficult to handle, doing a tonemapping/luminance filtering step to both history and current colour (as seen in Alex Tardif's TAA) can greatly mitigate this, this was the main solution to our own TAA flickering.
    
<br><br>
<h2 align="center">My Notes on TAA</h2><hr>

In the end the TAA I had to fix up had many issues, and unpacking them we almost impossible. In the end I did almost a full rewrite and backtracked from there to figure out what went wrong in our previous TAA. Turns out in an effort to reduce ghosting we increased the blend factor of 10% to much larger amounts when velocity increased (camera or object motion). This maybe sounds good at first glance since taking more of the current frame will reduce ghosting and result in sharper images. But we lose the main strength of TAA that it is temporally stable, most other screen-space anti-aliasing break during camera motion, not TAA. This was effecively throwing out most of our work as soon as the camera moved. There were a number of other small issues like tonemapping incorrectly and oddities with jitter calculations, but that was the most egregious.

<br>
<br>
