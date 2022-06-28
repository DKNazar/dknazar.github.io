---
title: "Moving to URP"
permalink: /moving-to-urp/
---
<h1 align="center">Moving to URP</h1>
Switching to URP was a daunting task especially from how Unity keeps changing things and often leave core features out. But after many years of URP it was time to switch and surprisingly it was not that much of a struggle. The key difference was Unity’s legacy renderer had a great feature “custom surface shaders” these were easy to program custom shaders which allowed you to hook into Unity’s default lighting pipeline. 

Origami Ninja Star uses a lot of custom surface shaders, from characters, players, foliage and rivers. URP provides no surface shader feature in code, I believe they are using shader graph now but I much prefer code to nodes. So this is quick start to URP with custom shaders, if like me you are using the legacy renderer, follow the [URP transition steps](https://docs.unity3d.com/Packages/com.unity.render-pipelines.universal@13.1/manual/InstallURPIntoAProject.html), and be prepared to see a lot of pink.

<h2 align="center">Quick start to URP with custom written shaders</h2>

In URP there is also three lighting models SimpleLit, Lit and ComplexLit, these seem to increasingly follow the PBR lighting model. I mainly have experience in the PBR workflow with the Cook-Torrance microfacet specular BRDF. But since Origami Ninja Star is obviously quite stylised and I’d love to get it running on mobile I decided mainly to use the SimpleLit shader (with some exceptions).

Now finally to actually write the custom shader, luckily there is a great starting off point I’d recommend [Cyanilux’s guide to URP](https://www.cyanilux.com/tutorials/urp-shader-code/). This guide provides the setup for the shader, like the inspector/properties setup, and each pass URP might need like depth (important), normals (less important). I’m going go over the key areas, for more detail definitely check out [Cyanilux’s page.](https://www.cyanilux.com/tutorials/urp-shader-code/)

<h2 align="center">URP shader break down</h2>

A custom shader I use for most enemies is a colour swapping shader, which replaces RGB channels from a texture with colours set per material. This is useful since I like to avoid making multiple textures, and it means I can instance lots of colours later on to get good enemy colour variety.

<h3 align="left">Properties:</h3>

This is the interface for the material to set values from the unity inspector to the shader.

```
Shader "Origami/SimpleLitKeyable" {
	Properties {
	[MainTexture] _BaseMap("Base Map (RGB) Smoothness / Alpha (A)", 2D) = "white" {}
        [MainColor]   _BaseColor("Base Color", Color) = (1, 1, 1, 1)

	[Space(20)]
	_KeyTex("Keying Map (RGB)", 2D) = "white" {}
	_KeyColor1("Key Color 1", Color) = (1, 1, 1, 1)
	_KeyColor2("Key Color 2", Color) = (1, 1, 1, 1)

	[Toggle(_NORMALMAP)] _NormalMapToggle ("Normal Mapping", Float) = 0
	_BumpMap("Normal Map", 2D) = "bump" {}
```

<h3 align="left">Shader Features:</h3>

These all define what features we want in the shader, some are attached to the properties interface but if we want a certain feature we can force it on/off here.

```
HLSLPROGRAM
	#pragma vertex LitPassVertex
	#pragma fragment LitPassFragment

	// Material Keywords, toggleable in inspector
	#pragma shader_feature_local _NORMALMAP
        #pragma shader_feature_local_fragment _EMISSION
        #pragma shader_feature_local _RECEIVE_SHADOWS_OFF
        //#pragma shader_feature_local_fragment _SURFACE_TYPE_TRANSPARENT
        #pragma shader_feature_local_fragment _ALPHATEST_ON
        #pragma shader_feature_local_fragment _ALPHAPREMULTIPLY_ON
        //#pragma shader_feature_local_fragment _ _SPECGLOSSMAP _SPECULAR_COLOR
	#pragma shader_feature_local_fragment _ _SPECGLOSSMAP
	#define _SPECULAR_COLOR // always on
        #pragma shader_feature_local_fragment _GLOSSINESS_FROM_BASE_ALPHA
```
<h3 align="left"> CBuffer section:</h3>

It is important to take care here since one of SRP’s biggest features is its CPU side batching when materials use similar CBuffers. To get this performance boost you need to make sure the shader is SRP compatible, which it tells you on the shader info. So far this hasn’t been an issue though.

```
  HLSLINCLUDE
	#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
	TEXTURE2D(_KeyTex);
	SAMPLER(sampler_KeyTex);

	CBUFFER_START(UnityPerMaterial)
	float4 _BaseMap_ST;
	float4 _BumpMap_ST;
	float4 _BaseColor;
	float4 _EmissionColor;
	float4 _SpecColor;
	float4 _KeyColor1;
	float4 _KeyColor2;
	float _Cutoff;
	float _Smoothness;
	float _EmissionAmount;
	CBUFFER_END
  ENDHLSL
```
<h3 align="left">Shader Features:</h3>
Each pass URP requires is declared here, you only need to worry about the forward pass unless you are moving vertices, like a wind shader. In that case you will need to do the same vert displacement in each pass. This is important otherwise you will get very odd behaviour in terms of clipping, or in post-processing effects.

```
Pass {
	Name "ForwardLit"
	Tags { "LightMode"="UniversalForward" }

	HLSLPROGRAM
	#pragma vertex LitPassVertex
	#pragma fragment LitPassFragment
      ...
    }
    // ShadowCaster, for casting shadows
Pass {
	Name "ShadowCaster"
	Tags { "LightMode"="ShadowCaster" }

	ZWrite On
	ZTest LEqual

	HLSLPROGRAM
	#pragma vertex ShadowPassVertex
	#pragma fragment ShadowPassFragment
      ...
      }
    // DepthOnly, used for Camera Depth Texture (if cannot copy depth buffer instead, and the DepthNormals below isn't used)
Pass {
	Name "DepthOnly"
	Tags { "LightMode"="DepthOnly" }

	ColorMask 0
	ZWrite On
	ZTest LEqual

	HLSLPROGRAM
	#pragma vertex DepthOnlyVertex
	#pragma fragment DepthOnlyFragment
      ...
      }
```

<h3 align="left">Vertex and Fragment shader functions:</h3>

These need to be setup and maintained to match Unity’s shading pipeline. While its quite messy the great thing is we can now pick and choose what lighting we want, and alter it as much as we want. For example since for my water shader I could use the PBR Cook-Torrence model to get more realistic water lighting. Similarly with my foliage shader I use the SimpleLit Blinn-Phong model but add my own translucent lighting version which looks much more realistic.

```
// Fragment Shader
half4 LitPassFragment(Varyings IN) : SV_Target {
	//UNITY_SETUP_INSTANCE_ID(IN);
    	//UNITY_SETUP_STEREO_EYE_INDEX_POST_VERTEX(IN);

	// Setup SurfaceData
	SurfaceData surfaceData;
	InitalizeSurfaceData(IN, surfaceData);

	// Setup InputData
	InputData inputData;
	InitializeInputData(IN, surfaceData.normalTS, inputData);

	// Key the colors and tint _KeyTex
	// Albedo comes from a texture tinted by color
	half3 c = surfaceData.albedo.rgb * _BaseColor.rgb;
	half4 key = SAMPLE_TEXTURE2D(_KeyTex, sampler_KeyTex, IN.uv.xy);
	half4 key2 = key;
	key = key.r * _KeyColor1;
	key2 = key2.b * _KeyColor2;
	key += key2;

	c = c * key.rgb;
	surfaceData.albedo = c.rgb;
	surfaceData.emission.rgb = _EmissionAmount * key.rgb;

	// Simple Lighting (Lambert & BlinnPhong)
	half4 color = UniversalFragmentBlinnPhong(inputData, surfaceData);

	color.rgb = MixFog(color.rgb, inputData.fogCoord);
	//color.a = OutputAlpha(color.a, _Surface);
	return color;
}
```

While this workflow is isn’t perfect and a bit messy to maintain, it does give me great control over URP shaders, meaning I can easily add features such as tessellation which is currently not possible with shader graph.

<br><br><br>
