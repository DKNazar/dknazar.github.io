---
title: "Post Processing in URP"
permalink: /post-processing-urp/
---
<h1 align="center">Post-Processing in URP</h1>

Adding a post processing step in URP is a pretty straight forward task, there is a sample on the [Unity github](https://github.com/Unity-Technologies/UniversalRenderingExamples/tree/master/Assets/Scripts) (DrawFullscreenFeature). You can also create it yourself by implementing the render feature interface but the end results will be the same. Using the camera colour texture as the render target (and texture) means we can output to the main screen. Now it is just need a shader with some small setup to sample the camera texture and the depth buffer.

```
HLSLPROGRAM
TEXTURE2D(_MainTex);
SAMPLER(sampler_MainTex);
...
float4 _MainTex_TexelSize; // useful, not required
...
// Sampling the main scene colour
half4 c0 = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, uv);
```

To sample the depth buffer, its best to include this file which does the setup for you.

```
#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/DeclareDepthTexture.hlsl"

```

For Origami Ninja Star I’m using a spatial edge detection pass ([Roberts cross](https://en.wikipedia.org/wiki/Roberts_cross)) to create sketchy looking outlines. A nice implementation of this can be found here, which does similar edge detection outlines. Since I’m using Unity’s fog which is applied per-object, the world gets foggier/whiter in the distance. This means the edge detection obviously won’t have many edges to detect. I tried screen-space fog but managing transparencies + UI with SS fog gets quite tedious.

```
// Four sample points of the roberts cross operator
float2 uv0 = uv;                                   // TL
float2 uv1 = uv + _MainTex_TexelSize.xy;           // BR
float2 uv2 = uv + float2(_MainTex_TexelSize.x, 0); // TR
float2 uv3 = uv + float2(0, _MainTex_TexelSize.y); // BL

 // Color samples
half3 c1 = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, uv1).rgb;
half3 c2 = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, uv2).rgb;
half3 c3 = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, uv3).rgb;

// Roberts cross operator
half3 g1 = c1 - c0.rgb;
half3 g2 = c3 - c2;
half g = sqrt(dot(g1, g1) + dot(g2, g2)) * 10.0;
```

<div align="center">
 
<img src="/images/fog.png" width="600" />
<img src="/images/ssFog.png" width="600" />
<br>
<em>Unity fog vs Screen-space fog</em><br>
</div>



I feel this adds a unique look to my game, where the paper creases are accentuated. Creating a more immersive paper folded world. Again the issue with this technique is the fog now doesn’t work with transparent objects since they are not on the depth buffer. I’m still working on this but so far the best solution is to increase the edge detection sensitivity by the same fog factor. So even tiny differences in the fog should be visible.


