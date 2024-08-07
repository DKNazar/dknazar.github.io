---
title: "Tech Notes: Origami Ninja Star"
permalink: /ONS_Tech_Notes/
---
<h1 align="center">Tech Notes: Origami Ninja Star</h1>

<div align="center">
<img width="600" src="/images/hdr-calibration-example.png" alt="hdr calibration screen for cricket">
<hr><br>
</div>

<h2 align="center">Preamble</h2>

These are miscellaneous notes on the technology of Origami Ninja Star, my solo developed game using Unity.

<h2 align="center">Unity</h2>

Unity has gone through many phases but in general I find it lately a bit more positive. But Origami was started early in my game dev career so Unity was the most approachable engine at the time. If I were to start again I probably would go Godot since I prefer being able to customise the engine now that I have more experience.

<h2 align="center">My Universal Render Pipeline (URP) Setup</h2>

I'm using the Unity Universal Render Pipeline since it is a stylized game and really doesn't need a lot of the full photorealistic lighting model. Plus I'd like the game to run on mobile and tablets.

The Setup:

I use the Forward+ rendering path with no depth prepass. Forward+ uses clustered lighting which is nice although I do not use many lights.
Avoiding a depth prepass is good since the structure of Origami makes most shapes simple does not produce much overdraw.

The shader structure of URP provides 3 different levels of complexicity: SimpleLit, Lit and ComplexLit. 
These three shaders provide different workflows and features for materials.

SimpleLit is a straight forward Lambert diffuse with optional BlinnPhong specular lighting + specular map.

```hlsl
half3 CalculateBlinnPhong(Light light, InputData inputData, SurfaceData surfaceData)
{
    half3 attenuatedLightColor = light.color * (light.distanceAttenuation * light.shadowAttenuation);
    half3 lightDiffuseColor = LightingLambert(attenuatedLightColor, light.direction, inputData.normalWS);

    half3 lightSpecularColor = half3(0,0,0);
    #if defined(_SPECGLOSSMAP) || defined(_SPECULAR_COLOR)
    half smoothness = exp2(10 * surfaceData.smoothness + 1);

    lightSpecularColor += LightingSpecular(attenuatedLightColor, light.direction, inputData.normalWS, inputData.viewDirectionWS, half4(surfaceData.specular, 1), smoothness);
    #endif

#if _ALPHAPREMULTIPLY_ON
    return lightDiffuseColor * surfaceData.albedo * surfaceData.alpha + lightSpecularColor;
#else
    return lightDiffuseColor * surfaceData.albedo + lightSpecularColor;
#endif
}
```

Lit changes to the more realistic CookTorrance BRDF (lightweight version) which is approaching the typical lighting model used in realistic games.

Great resource on specular brdf
https://graphicrants.blogspot.com/2013/08/specular-brdf-reference.html

ComplexLit seems to follow the same Lit path of CookTorrance but with additional features such as Clearcoat.

BRDF mainly impacts the specular function which is not very useful in a world made in PAPER, where the speculars are quite low...

So I primarily use SimpleLit as without specular having much visual impact I don't need the extra cost of more realistic specular responses.

While Unity have removed Surface shaders (Block shaders are coming EVENTUALLY) in favour of shader graph, I think programming is much faster and easier. So I duplicate the SimpleLit shader and make my own alterations to it where required. This means the shaders will also be SRP Batcher compatible meaning Unity can batch the related buffers to improve CPU performance when rendering.

For example in the procedural levels I have made a custom SimpleLit which handles vertex deformation during scene transitions, and dynamic global colours palettes in the fragment shader.

The post-processing stack is fairly small including:
- SMAA
- SSAO (A stylized variant which applies shading on the final image so no depth prepass)
- Bloom
- Custom edge detection pass to add sketchy line work to the world
- DOF (sometimes)

https://www.youtube.com/watch?v=j-A0mwsJRmk

- Forward+ clustered
- Using no depth prepass
- SimpleLit for majority
- Need to keep shader SRP compatible
- Infinite Shader for custom effects
- 
- Unrealistic SSAO
- Custom post process

<h2 align="center">Procedural Levels</h2>

<h3 align="center">Room Floorplans</h3>

Each level in the main game is a series of randomly selected rooms, each room has a layout or floorplan made out of lines, like the blueprint of a house. These lines are created from a custom editor within Unity, where you can design the room map. The floorplan determines walls, platforms, valid door placements, prop/foliage, enemy types, counts and location.

<h3 align="center">Mesh Tiles</h3>

Floorplans are then constructed through a series of quads shaped meshes. Floors are generated in realtime in most cases (unless some complex floor geometry is required). Prop prefabs are distributed across declared zones, this is the only case of instantiation. 

<br><br>
