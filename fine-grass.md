---
title: "Fine Grass"
permalink: /fine-grass/
---
<h1 align="center">Rendering Fine Grass</h1>

<h2 align="center">Preamble</h2>

Most games are about adventure and exploration, with tall grass and dense foliage, not sports games where grass is trimmed and tiny. In wild grass focused games like Breath of the Wild and Ghost of Tsushima, rendering grass can be achieved through rendering each individul grass blade using 3D geometry. This is done through instancing, indirect draws, and clever tricks, but these methods do not scale to the amount of geometry that would be required in fine short grass. So we have to go back to textured quads/billboards/cards which represent many grass blades on one mesh. Less fun.

<div align="center">
<img width="600" src="/images/grass-mat.png" alt="grass wth material">
</div>

<h2 align="center">Fully Explain</h2>

First we need to calculate where the grass should be rendered, we don't want to render everywhere and anywhere. On the CPU we calculate the centre point of the grass bounds. This involves a number of plane intersection tests with the camera frustrum planes and the ground (which is always 0 for us). There are a few special considerations, if there is no top corners intersecting or if they are above our maximum distance we need to project them to the maximum. These distances also need to be calculated relative to the bottom corner/near line intersections. Because at small fovs the intersection corners can be extremely far from the camera world position.

```c++
for (u32 i = 0; i < 4; i++)
{
	{
	const Vector3& p = cameraPosition;
	const Vector3& dir = edgeDirections[i];
       
        // if above ground and we got intersection
	isIntersection[i] = corners[i].getY() < 0.0f && LinePlaneIntersection(intersections[i], planeN, dir, p);
        // without intersection just set them to camera far corners on the ground
	if (!isIntersection[i])
	{
		intersections[i] = corners[i];
		intersections[i].setY(0);
	}
	}

      	// should unroll loop since we do these conditional operations
	if (i == TopRight)
	{
        	// find centre point at near intersection line
		nearIntesectCentre = lerp(0.5f, intersections[BotLeft], intersections[BotRight]);
	}

	// use near centre dist so we can use small fovs and still spawn grass
	if (i >= TopRight && lengthSqr(nearIntesectCentre - intersections[i]) > (grassMaxDistSqr))
	{
        	// since our far intersections are too far, we project them towards the direction at the maximum allowed distance
		Vector3 p = cameraPosition + edgeDirections[i] * (GRASS_DISTANCE + length(nearIntesectCentre - cameraPosition));
		// if above ground still then intersect test downwards
		const Vector3 dir = p.getY() > 0.0f ? -cameraUp : normalize(intersections[i] - p); // shoot towards near intersections if we are bl
		LinePlaneIntersection(intersections[i], planeN, dir, p);
	}
}
    
Vector3 centre = lerp(
	0.5f,
	lerp(0.5f, intersections[BotLeft], intersections[BotRight]),
	lerp(0.5f, intersections[TopLeft], intersections[TopRight]));
```

After calculating the centre point we use this as the piviot for our patch of grass positions generated in the compute shader. We use a grass_mesh_density value as our grid density for distances between each grass quad. Using our current grass index (threadId) to calculate the local translation from the centre point. These positions are snapped to the world grid and later used as a seed for the grass quad properties.

```hlsl
   float gridDist = g_grass_mesh_density;
   float2 vec = (float2(int2(threadId.xy) * 2 - grassCountSqrt) + 0.5.xx) * gridDist.xx * 0.5;
   float2 centre = grassCentre.xy + gridDist.xx * 0.5;
   float2 p = centre + vec; // world point
   
   instanceData.position = p;
   float2 dir = normalize((rand2dTo2d(p) * 2 - 1));
   instanceData.facing = dir;
   float nearPlaneDistance = dot(instanceData.position.xy - nearPlanePos.xy, cameraForward.xy);
   // reduce height at distance from near plane, and when grass projection is distant
   float heightScale = (1.0 - saturate(nearPlaneDistance / g_grass_distance_falloff)) * smoothstep(0.0, 0.2, planeVisibility);
   heightScale = smoothstep(0.8, 1.0, heightScale);
   float rngH = g_grass_height + rand2dTo1d(p, 0.68) * g_grass_height_noise;
   
   instanceData.height = rngH * heightScale;
   instanceData.width = g_grass_mesh_width + g_grass_mesh_width_noise * rand2dTo1d(p);
   // structured buffer
   GrassInstanceBuffer[idx] = instanceData;
```


Now that the GrassInstanceBuffer (StructuredBuffer) has been populated the next step is to call our instance draw, using the grass buffer with the InstanceID to retrieve the appropriate instance data in our vertex shader. 

```hlsl
   GrassInstanceData data = GrassInstanceBuffer[instanceId];
   float4 position = float4(localpos.xyz, 1);
   
   // we shear/flatten our grass dependant on the relative camera height to give grass more volume at high angles
   float3 toCamera = cameraPosition.xyz - float3(data.position.x, 0.0f, data.position.y);
   float cameraDistance = max(length(toCamera), 0.001);
   toCamera /= cameraDistance;
   
   float3 up = float3(0, 1, 0);
   float2 facing = data.facing;
   float yFactor = saturate((dot(up, toCamera)) * g_grass_heightShear);
   
   position.z -= yFactor * localpos.y;
   position.xy *= float2(data.width, data.height);
   
   position.xz = rotate(position.xz, float2(facing.y, -facing.x));
   position.xz += data.position.xy;
```
<div align="center">
<img width="600" src="/images/grass-flat.png" alt="grass flattening to camera">
</div>
	
Now using the instance data to get our vertex positions, we need a depth prepass to avoid overdraw, the cost of depth overdraw is much much less than the lit pass overdraw. After that we render the lit grass which involves a lot of small tricks to blend the colour with the ground while also having its own detail. In the pixel shader we blend between the grass texture and the field texture based on distance from the camera near intersection (bottom corners). We also blend the material and normals based on a factor driven by camera height angle. This means when the camera is low and close to the grass we use the grass material maps but when the camera is higher we use the default ground values.

```hlsl
   float nearPosDistance = dot(position.xz - nearPlanePos.xy, position.xz - nearPlanePos.xy);
   float falloff = saturate((nearPosDistance / g_grass_mat_falloff) + (1.0 - planeVisibility));
   
   // ground grass colour
   float3 fieldGrassColor = field_grass.Sample(field_grass_s, grassdiffuseUV).rgb;
   float4 grassPlaneColour = grass_col.Sample(grass_col_s, uv);
   fieldGrassColor = lerp(grassPlaneColour.rgb, fieldGrassColor, falloff);
   
   float cameraHeightBlend = saturate(dot(toCamera, float3(0, 1, 0)) * g_grass_mat_heightBlendBias + falloff * 0.5);
   N = lerp(nrm_sample.xyz, N, cameraHeightBlend);
   float4 matSample = grass_mat.Sample(ClampedLinear, uv);
   
   // match field only when far or high angle
   material.metalness = lerp(matSample.r, material.metalness, cameraHeightBlend);
   material.roughness = lerp(matSample.g, material.roughness, cameraHeightBlend);
   material.cavity = lerp(matSample.b, material.cavity, cameraHeightBlend);
        
```

<div align="center">
<img width="600" src="/images/grass-full-blend.png" alt="grass fully blended with ground">
</div>

<br><br>
<h2 align="center">End Notes</h2><hr>

The grass is far from perfect, it has decent coverage and performance is ok but the field shader is way too expensive which the grass needs to match. While I used a simplified version of the shader it still has much too many texture samples. Compared to the old grass system it is a massive improvement and much more adaptable, being generated based on parameters instead of a static mesh like before.

<br><br>
