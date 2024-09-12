---
title: "Grass Rendering Demo"
permalink: /grass-rendering-demo/
---
# Stylized Grass Rendering with GPU Culling #

### What is this repository for? ###

A stylized grass rendering demo with gpu culling, some physics interactions, and adjustable/dynamic grass placement.
This grass renderer is made for my yet to be released game Origami Ninja Star, which is a game in a stylized paper world. The game is split into seamless procedurally generated rooms, this renderer is built for those requirements.

It demonstrates: using custom URP shaders (not shadergraph), creating a custom SRP renderfeature, indirect instanced rendering, compute shaders, and gpu culling all in Unity.

![Grass Demo screen and interface](https://dknazar.github.io/images/grass_demo.png)

### How do I get set up? ###

[Download a build here](https://bitbucket.org/DKNazar/grass-renderer-demo-unity/downloads/) and give the demo a try!
It has basic camera movement and many options to change the terrain and grass rendering to test features.

You can also open the project in Unity to test it out.

Using Unity 2023.2.20f1
Clone or download repo and open in Unity.

### Usage Note ###

This code is for demo and educational purposes only.

## Background Information ##

When rendering grass a lot of duplicate meshes (grass blades) are scattered across the ground, but each blade should have some small variation to look realistic. A common way to do this is GPU Instancing, where instead of rendering each object individually with its own transformation matrix like most engines do, you can store the position data into a single large data buffer and call the GPU DrawInstanced function. The DrawInstanced function has an Instance Count parameter which determines the amount of duplicates of the mesh to be rendered. Except now each mesh will now have its own InstanceID value that can be used to index into the large data buffer with our positions in it. This MeshData buffer is used to then calculate where our meshes should be rendered. This is a good optimisation as there is lower CPU->GPU interaction since there is only one big render instead of many small ones.

This renderer started off using Instancing but slowly progressed to Indirect Instancing to enable GPU culling. It also includes the process capturing a heightmap of the area to procedurally place grass on any given area. This uses a separate SRP/URP renderer using an orthographic top down view of the desired grass area.


## Program Summary ##

The program starts with:

	Creating a heightmap of the area, subtracting occluders and generating normals from it
	Setting up the computer shader parameters and buffers based on the area size and grass parameters
	Running the init compute shader to populate the instance buffer and setup grass chunk instance counts
	
Every update:

	Checks for rigidbodies within the trigger box and adds the position and radius to an array (to pass along to the compute)
	Runs a compute update that draws force vectors to a FlowMap texture that will be used when rendering grass
	Performs compute culling step based on the camera frustum, appending the valid indexes to an AppendBuffer, and updating the indirect args (instance count)
	Runs DrawIndirectInstance with the indirect args
	The grass material uses its InstanceID to index into the MeshData buffer to find its start and endpoint
	Grass verts are transformed under a bezier curve using the start and endpoint. The endpoint is also transformed by the FlowMap.
	Grass uses the normal value of the ground to blend better with the ground in lighting

## Material ##

The [indirect grass](https://bitbucket.org/DKNazar/grass-renderer-demo-unity/src/main/Assets/Shaders/IndirectGrass.shader) material accesses the instance data buffer which holds the Startpoint in worldspace and Endpoint in localspace. It also has other data like height, width, and ground normal. This isn't the final struct, it may shrink or grow depending on requirements.
This is stored in a structured buffer, which is populated by a compute shader after generating multiple grass maps.

Snippet of instance data:
```hlsl

	struct MeshProperties
	{
		float3 startPoint; // WORLD SPACE
		float halfWidth;

		float3 endPoint; // Gives us rotation xz, IN LOCAL SPACE
		float height;

		float3 groundNormal;
		float padding2;
	};

	StructuredBuffer<MeshProperties> _meshData;
```

To use URP materials with custom shaders with no shader graph (which it does not officially support) I have duplicated URP header files and split them up so I can swap out files where required. An example of this is [LitInputThin.hlsl](https://bitbucket.org/DKNazar/grass-renderer-demo-unity/src/main/Assets/Shaders/URPLibrary/Input/LitInputThin.hlsl) and [LitCBufferBase.hlsl](https://bitbucket.org/DKNazar/grass-renderer-demo-unity/src/main/Assets/Shaders/URPLibrary/CBuffers/LitCBufferBase.hlsl), these files used to be combined but that meant changing cbuffers required duplicating a bunch of functions. Now to declare my own cbuffer I just replace LitCBufferBase.hlsl with my own [LitCBufferGrass.hlsl](https://bitbucket.org/DKNazar/grass-renderer-demo-unity/src/main/Assets/Shaders/URPLibrary/CBuffers/LitCBufferGrass.hlsl).

A snippet of grass shader includes:
```hlsl
	#include "URPLibrary/CBuffers/LitCBufferGrass.hlsl" // custom cbuffer
	#include "URPLibrary/Input/LitInputThin.hlsl" // default input functions
	#include "Packages/com.unity.render-pipelines.universal/Shaders/LitForwardPass.hlsl" // default urp
	#include "InstancedGrassFunctions.hlsl" // specific grass functions header
```

This also means custom shaders will work with the [SRP batcher](https://docs.unity3d.com/Manual/SRPBatcher.html) which is an optimisation on the CPU when rendering by batching common CBuffers across materials to avoid extra CBuffer updates to the GPU. Although grass itself would not benefit from this since it uses GPU instancing.

## Grass Map Generation ##

The grass is to be used on procedural rooms, so each time grass is generated it renders a top down view of the area, using the depth to create a heightmap. This is done twice, the first time it renders the floor objects and generates a normals texture from the depth. The second time it renders occluding obstacle objects and subtracts it from the heightmap so it does not generate grass in covered areas. A third map is also created to store flow/forces, when physics objects collide with the grass so it flattens appropriately.

Grass Height, Normals, and Flow/Force maps:

![Grass Height, Normals, and Flow/Force map](https://dknazar.github.io/images/grass_maps.png)

This rendering is done through the scriptable render pipeline render feature. So when rendering the maps it uses a minimal renderer with the extra [GrassMapRendererFeature.cs](https://bitbucket.org/DKNazar/grass-renderer-demo-unity/src/main/Assets/Scripts/GrassMapRendererFeature.cs) enabled.

Here is how the render feature works, it also alternates behaviours between each render:
```c#
 public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
        {
			ref var cameraData = ref renderingData.cameraData;
            if (m_passCount == 0)
            {
                CommandBuffer cmd = CommandBufferPool.Get("DepthBuffer to RenderTexture Copy");
                cmd.SetRenderTarget(m_depthOutput, 0);
                cmd.ClearRenderTarget(false, true, Color.black);
				// Ideally unity would just render depth only directly to my depth RenderTexture
                // but this doesn't seem to work so I have to copy it over myself. Maybe in URP2

				// Simple Unity blit camera depth -> render texture depth
				Blitter.BlitTexture2D(cmd, cameraData.renderer.cameraDepthTargetHandle, new Vector4(1f, 1f, 0f, 0f), 0, false);
				
				// Generate normals from depth before subtract to avoid any weird edges
				m_materialNormals.SetTexture("_BlitTexture", m_depthOutput);
                CoreUtils.SetRenderTarget(cmd, m_normalsOutput);
                // now calc normals from depth remaining
				CoreUtils.DrawFullScreen(cmd, m_materialNormals, null, 1); // using shader pass 1

			    context.ExecuteCommandBuffer(cmd);
				CommandBufferPool.Release(cmd);
				m_passCount++;

			}
            else if (m_passCount == 1)
            {
				// Just rendered grass occluders to the DepthBuffer, so do a custom blit to our Grass Height map render texture (depthOutput)
				// but we subtract the result instead of copying it.
                CommandBuffer cmd = CommandBufferPool.Get("DepthBuffer to RenderTexture Subtract");
                CoreUtils.SetRenderTarget(cmd, m_depthOutput);
				m_materialSubtract.SetTexture("_BlitTexture", cameraData.renderer.cameraDepthTargetHandle);
				// this material has subtract blend mode.
                CoreUtils.DrawFullScreen(cmd, m_materialSubtract, null, 0);
				context.ExecuteCommandBuffer(cmd);
				CommandBufferPool.Release(cmd);
                m_passCount++;
			}
		}
```

The actual shader is Blit_GrassCapture.shader, it contains 2 passes, the first is just a blit with a big multiply so the subtract fully negates the heightmap. The second pass is the depth to normal map step, while we could tell Unity we want normals, it would do a depth normals prepass. But since our grass maps are only 512x512 the lower res makes sampling the depth more appealing than running every vertex twice to produce normals.

Here is a snippet of depth to normals conversion.
``` hlsl
// Works but pretty low quality
half3 ReconstructNormalDerivative(float2 cs)
{
	float3 viewSpacePos = ViewPosAtPixel(cs);
	float3 hDeriv = ddy(viewSpacePos);
	float3 vDeriv = ddx(viewSpacePos);
	return half3(normalize(cross(hDeriv, vDeriv)));
}
```

This is a common technique where you can use the ddy/x functions to get the derivative of a value (in our case the heightmap) across pixels while rendering. Then using these derivative vectors of the xy we can use the cross product to get the final z component, which is our normal. Now this technique works like this but is fairly low quality, the normals generated are quite low res. Thankfully Unity has functions available to us (from bgolus) which by doing 3x samples we get a much nicer normalmap.

Snippet of the higher quality depth to normals conversion:

```hlsl
// Taken from https://gist.github.com/bgolus/a07ed65602c009d5e2f753826e8078a0
// unity's compiled fragment shader stats: 33 math, 3 tex
half3 ReconstructNormalTap3(float2 cs)
{
	// get current pixel's view space position
	float3 viewSpacePos_c = ViewPosAtPixel(cs + float2(0.0, 0.0));

	// get view space position at 1 pixel offsets in each major direction
	float3 viewSpacePos_r = ViewPosAtPixel(cs + float2(1.0, 0.0));
	float3 viewSpacePos_u = ViewPosAtPixel(cs + float2(0.0, 1.0));

	// get the difference between the current and each offset position
	float3 hDeriv = viewSpacePos_r - viewSpacePos_c;
	float3 vDeriv = viewSpacePos_u - viewSpacePos_c;

	// get view space normal from the cross product of the diffs
	half3 viewNormal = half3(normalize(cross(vDeriv, hDeriv)));

	return viewNormal.xyz;
}
```
## Compute Shader ##

[GrassInteractCompute.cs](https://bitbucket.org/DKNazar/grass-renderer-demo-unity/src/main/Assets/Shaders/GrassInteractCompute.compute)

### Generate Grass (CSVertexInit) ###

Poorly named but this compute function initialises the mesh instance data based on the grass maps generated beforehand. Mainly using the Depth/Heightmap to locate where to position blades of grass. The compute is dispatch is determined by how big the map size is in world space, meaning the threads work in meters.

Snippet of mesh positions being calculated:

```hlsl
	// start at bottom left corner of map
	float3 basePos = _botLeftWorldSpace.xyz;
	// DispatchThreadID is our localposition as we work in meters
	float2 localPos = id.xy;
	basePos.xz += localPos.xy + 0.5.xx; // 0.5 sets it in the middle of the discrete square meter
	// random offset from centre
	float2 offset = Rand2Dto2D(basePos.xz + i.xx) * 2.0 - 1.0;

	basePos.xz += offset;
	basePos.y = height;

	float3 endPoint;
	float3 nrm = _normalMap.SampleLevel(s_linear_clamp_sampler, uv, 0).xyz;
	float3 groundNormal = float3(0, 1, 0);
	groundNormal = nrm;

	// random position for tip of grass for direction
	endPoint.xz = normalize(Rand2Dto2D(basePos.xz) * 2.0 - 1.0);
	endPoint.y = _meshParams.x * saturate(Rand2DTo1D(basePos.xz) + 0.5);

	// finally create the mesh struct
	MeshProperties mp = { basePos, _meshParams.y, endPoint, _meshParams.x, groundNormal, padding };
```

### Update Flow Map (CSUpdate) ###

To have grass react to physics objects I maintain a map called FlowMap which holds displacement vectors, so by default is float4(0,0,0,0). The script [ShaderPositionUpdater.cs](https://bitbucket.org/DKNazar/grass-renderer-demo-unity/src/main/Assets/Scripts/ShaderPositionUpdater.cs) reacts to rigidbodies in its trigger box area and adds their position and approximate radius to an array. The array only holds 8 positions per frame, and these positions are passed to the Compute shader and will be treated as sphere colliders.

Snippet of [ShaderPositionUpdater.cs](https://bitbucket.org/DKNazar/grass-renderer-demo-unity/src/main/Assets/Scripts/ShaderPositionUpdater.cs) adding rigidbodies to the position array:
```c#
private void OnTriggerStay(Collider other)
	{
		if (layerMask == (layerMask | 1 << other.gameObject.layer))
		{
            if (m_frameStayCount < POSITION_COUNT)
            {
                Vector4 p = other.attachedRigidbody.worldCenterOfMass;
                Vector3 e = other.bounds.extents;
				p.w = Mathf.Max(e.x, Mathf.Max(e.y, e.z));
                if(layerMask == (layerMask | 1 << 3))
				    p.w = Mathf.Max(p.w, 1.5f);
				positions[m_frameStayCount++] = p;
			}
		}
	}
```

The CSUpdate dispatch is calculated from the FlowMap's resolution, so each thread is a texel. The texel world position is calculated and used to check if any rigidbody is close enough to cause a repulsion vector. 

Snippet in CSUpdate loop calculating the force vector to move grass away from rigidbody:
```hlsl
	// get vector towards grass and scale by distance ratio
	float3 distVector = basePos.xyz - dynamicPos.xyz;
	float dist = length(distVector);
	float distStep = saturate(1.0 - (dist / radius)); // 1 = close, 0 = far
	float3 dir = (distVector / dist);
	externalForce += dir * distStep * 0.5; // additively so multiple overlaps can work
```

The FlowMap also slowly returns to 0 over time so grass is not permanently flattened. This is simply done by reducing the sample value over time with the timedelta. I also added a step to stomp to 0 whenever it gets below 0.01 to avoid vibrating grass (ping ponging between -small value <-> +small value).

Snippet of FlowMap slowly returning to 0:
```hlsl
float3 flowMapSample = (_flowMap[id.xy].xyz);
float3 signs = sign(flowMapSample.xyz);
// so want to transition texture output towards 0, meaning normal grass here
if (any(flowMapSample))
	{
		flowMapSample -= signs * (0.1) * unity_DeltaTime.z;
		flowMapSample *= step(0.01.xxx, abs(flowMapSample));
	}
```

### GPU Culling (CSFrustumCull) ###


GPU Culling is where we decide whether to render a mesh or not using the GPU through a compute shader. This can be done in many different ways, I am just using camera frustum to cull, meaning I do not cull if an object is blocking the view, only if the camera is not facing the mesh.

Grass frustum culling:

![Grass frustum culling](https://dknazar.github.io/images/grass_mesh_culling.png)

The way to do this is using Indirect Rendering, where the GPU can populate a buffer that is used when the actual mesh rendering occurs. In Unity this is when you call Graphics.DrawMeshInstancedIndirect(), and you pass the GPU args buffer which holds the desired instance/mesh count. But since not all meshes are rendering the material which retrieves mesh data from the instance buffer by using the InstanceID will no longer work. As we might not being rendering at MeshData index 1 or 20 or 25, so to solve this we create an Append buffer which holds the indexes into the MeshData buffer which we want to render.

Snippet of using instanceID to find the actual desired meshData index:
```hlsl
    instanceID = _indirectData[instanceID];
    float3 instancePos = _meshData[instanceID].startPoint;
```

Snippet of CSFrustumCull where I check if the mesh is within the camera view:
```hlsl
// run through each mesh instance
// check if within frustum
// if good then append to another array which holds the current index
// grass shader will access this buffer with instanceID later
MeshProperties mesh = _meshData[flatten];
bool isOutside = GrassCullFrustum(mesh.startPoint, mesh.endPoint, -0.01, _frustumPlanes, 6);
if (!isOutside)
{
	_indirectData.Append(flatten);
	InterlockedAdd(groupInstanceCount, 1); // count
}
```

This of course increases performance since the gpu doesn't have to render as many meshes. But culling itself can be an expensive operation especially if you do it on every single blade of grass.


### Chunking! ###

Not sure if there is an official term for this but to make culling faster I split the grass into spatial chunks. So the grass area is broken into chunks, the size of chunks changes depending on how big the total area. When a grass blade position is calculated the position determines which chunk it belongs to.

Snippet of the bufferIndex being calculated for a specific mesh based on position:
```hlsl
// chunks are determined spatially
uint chunkIdx = FlattenChunk(GetChunkIndex2D(basePos.xz));
// add a grass instance count to our chunk (in the _instanceCount array)
InterlockedAdd(_instanceCount[chunkIdx], 1, bufferIndex);
// the InterlockedAdd returns the current population of this chunk, use it to index the _meshData buffer
bufferIndex += _chunkBufferSize * chunkIdx;
// now we have an index at the chunk offset + the instance count for that chunk 
```

Each chunk has its own section in the MeshData buffer that it populates with data. Each section size is determined by how many blades of grass per metre and chunk size. This is calculated in [GrassRenderer.cs](https://bitbucket.org/DKNazar/grass-renderer-demo-unity/src/main/Assets/Scripts/GrassRenderer.cs) in the UpdateIndirectData() function. This only needs to be called once per grass area.
Finally this all changes our culling function into a much smaller operation. If using chunks culling only needs to run per chunk, which right now is 16x16, otherwise it needs to test every single mesh which is like...a lot (100000+).

```hlsl
	// get the chunk we are
    uint flatten = FlattenChunk(dispatchThreadID.xy);
    uint chunkPopCount = _instanceCount[flatten];
    if (chunkPopCount > 0)
    {
        // run through each chunk check the chunk "quad" is in view
        // if good then append to another array which holds the current index
        // grass shader will access this buffer with instanceID later

        float2 quadPoints[4];
        GetChunkCorners(dispatchThreadID.xy, quadPoints);
        uint bufferOffset = _chunkBufferSize * flatten;

        float4 heights = float4(SampleHeight(quadPoints[0].xy), SampleHeight(quadPoints[1].xy),
                                SampleHeight(quadPoints[2].xy), SampleHeight(quadPoints[3].xy));

        bool isOutside = GrassCullFrustumQuad(float3(quadPoints[0].x, heights[0], quadPoints[0].y), float3(quadPoints[1].x, heights[1], quadPoints[1].y),
                                            float3(quadPoints[2].x, heights[2], quadPoints[2].y), float3(quadPoints[3].x, heights[3], quadPoints[3].y),
                                            -6.0, _frustumPlanes, 6);
        if (!isOutside)
        {
            // need to append the index of the mesh, the chunkOffset + instanceCount in that chunk
            // we are going over the whole chunk though
            for (uint i = 0; i < chunkPopCount; i++)
            {
                _indirectData.Append(bufferOffset + i);
            }
            InterlockedAdd(groupInstanceCount, chunkPopCount); // render count
        }

    }
```

Grass frustum chunk culling:

![Grass frustum chunk culling](https://dknazar.github.io/images/grass_chunk_culling.png)

You can see each chunk is a different colour and the culling now is on a per chunk basis.


<br><br>

