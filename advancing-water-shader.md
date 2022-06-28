---
title: "Advancing the Water Shader"
permalink: /advancing-water-shader/
---
<h1 align="center">Advancing the Water Shader</h1>

One of my first shaders was the [“water-coloured”](https://dknazar.wordpress.com/portfolio/watercolour-water-shader/) water shader. The aesthetic was to fit liquid in a paper world, paint/water-coloured seemed quite fitting. While the shader has changed over time the basics remained, a gradient + noise + remap, now we have fancier things like specular shading and normal maps.

<div align="center">
todo:images old water vs new water
  
  ![gif](/images/waterfallspec.gif)

  <em> New water </em>
</div>

In other projects I had modelled the water mesh in a 3D modelling program but to make Origami Ninja Star a more dynamic solution was needed. I decided to use an in-engine tool to create rivers through Bezier curves. Specifically cubic Bezier curves, here is a link to a [great primer](https://pomax.github.io/bezierinfo/), the evaluation function is fairly simple.

```
float3 Bezier(float3 p0, float3 p1, floa3 p2, float3 p3, float t):
  float t2 = t * t;
  float t3 = t2 * t;
  float mt = 1-t;
  float mt2 = mt * mt;
  float mt3 = mt2 * mt;
  return p0*mt3 + 3*p1*mt2*t + 3*p2*mt*t2 + p3*t3;
```

I found a similar free road building tool with curve editor capabilities and heavily modify it to instead create rivers. I also added in additional features such as water thickness, varying width, and different/multiple materials.

<div align="center">
<img src="/images/watereditor.gif">
<img src="/images/watereditor1.gif">
</div>

Another issue was water detail was lacking with this method due to the lack of polys. This lead me to everyone’s favourite water feature tessellation. Implementing tessellation in URP was surprisingly straight forward and extremely messy but it adds much greater visual fidelity to the water, especially with waves.

Now finally the purpose of all these components is to make the water physical. In Origami Ninja Star you can surf the water and waves. For waves a more realistic approach is to use [Gerstner waves](https://developer.nvidia.com/gpugems/gpugems/part-i-natural-effects/chapter-1-effective-water-simulation-physical-models), which look great but I feel are a bit excessive for my more stylised game. So instead I’m using a simple Sin wave + simplex noise to add visual variation. This is more performant and easy to use later on.

<div align="center">
<img src="/images/wave_no_tess.png" width="50%"></img>

<em> No tessellation wave</em>

<img src="/images/wave_tess.png" width="50%"></img>

<em> With tessellation wave</em>
</div>

Thanks to our tessellation the waves will always have excellent detail with the player interacts with them. This means the collisions will always look quite accurate. To perform these on the CPU side we just need to sync the data from the shader to the collision function, which again is just a Sin wave. Some additional information required for a robust water collision system, we need to also calculate the normals. Thanks to our simple wave equation finding the derivative for the wave is simple, just Cos and some basic transformations.
```
   // x is our distance along the sin wave (sin(x))
   float cosDx = 0.5f * heightMultiplier * waveFrequency * Mathf.Cos(x);
   Vector2 normal2D = new Vector2(-cosDx / Mathf.Sqrt(cosDx * cosDx + 1.0f),
                                   1.0f / Mathf.Sqrt(cosDx * cosDx + 1.0f));
   Vector3 normal = normal2D.x * transform.forward + normal2D.y * transform.up;
   return normal;
```

Now we can use the normal in physics calculations.

<div align="center">
<img src="/images/waternormals.gif">

<em>Surface point (red), Surface normal (blue)</em>

<img src="/images/waternormalsmoving.gif">

<em>With moving rigid-bodies</em>
</div>
