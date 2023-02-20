---
title: "The Mysterious HDR"
permalink: /hdr/
---
<h1 align="center">The Mysterious HDR</h1>

<div align="center">
<img width="600" src="/images/hdr-calibration-example.png" alt="hdr calibration screen for cricket">
<hr><br>
</div>

<h2 align="center">Preamble</h2>

High Dynamic Range (HDR) sounds very cool but what is it? After large amount of research I barely know, but it seems to work in the game for now. It is basically the wild west in HDR, we have some standards but often you can't rely on them, the screens do absurd things to emulate HDR for marketing, and all our assets/artists are in SDR.

<h2 align="center">Quickly Explain</h2>

Should white paint and the sun reach the same brightness? probably not and with the power of HDR you can represent both somewhat more appropriately. Usually in games we render lighting in HDR (greater than 8 bits per channel) where we can represent colour values greater than 1.0. But when outputting to a monitor we must crunch this data back down to meet the requirements of the monitor which can only represent to 1.0. To bring the range back to 1.0 we tonemap the image which applies a curve that looks visually pleasing/natural to our eyes. This curve is mostly important near 0 or 1 (the shadows and highlights), so while the sun was 100.0 and white paint was 1.0, the tonemapper will flatten these values with the sun ending up 0.99 and white paint 0.9.

Previously in SDR land our 0-1 brightness values were relative to the monitor or TV supplied. These screens generally reach max brightnesses of 200-300 nits, so the SDR value of 1.0 simply shows up as 300 nits of brightness on your screen. But with HDR the scale is no longer relative, we can more accurately display the exact nit value we want. If we want the sun to be 10,000 nits we can now ask for that exactly (but no screen can reach that). The HDR standard (PQ) currently defines a 0-10,000 nits brightness on a non-linear scale from 0-1.

With HDR you also respresent more colour information using the REC.2020 colour primaries, but currently all our assets were made for REC.709.


<h3 align="center">Glossary</h3>

  <ul>
    <li>Tonemapping - applying a curve to an image to alter/remap the brightness or tones of that image.</li><br>
    <li>Nits - a measurement for brightness that screens use.</li><br>
    <li>Maximum Nits/Maximum Display Nits - what is the peak brightness the screen can reach.</li><br>
    <li>Paperwhite nits - the brightness that feels white like a piece of paper, but not white like the sun or a bright light. This can depending on your viewing environment, similar to how gamma works.</li><br>
    <li>Electro-Optical Transfer Function (EOTF) - a function/curve that gets applied to the game colour output so it can be properly represented on screen. In SDR this is our gamma function, and in HDR this is our Perceptual Quantizer function.</li>
    <li>Perceptual Quantizer - to display HDR to a screen we must apply the SMPTE ST 2084 PQ function which is based on human perception, and outputs to the range of 0-10,000 nits.</li>
  </ul>

<h3 align="center">Research Links</h3>

<ul>
  <li><a href="https://knarkowicz.wordpress.com/2016/08/31/hdr-display-first-steps/" target="_blank"> HDR First Steps</a>: Nice overview to get started.</li>
  <li><a href="https://displayhdr.org/certified-products/" target="_blank"> HDR Vesa Standards</a>: list of HDR standards that should be followed by manufacturers.</li>
  <li><a href="https://www.youtube.com/watch?v=pWyd835pfeg" target="_blank">High Dynamic Range in DirectX Games</a>: I wish this existed when I was learning about HDR, it sums it up so well.</li>
  <li><a href="https://www.glowybits.com/blog/2016/12/21/ifl_iss_hdr_1/" target="_blank">HDR Display Support in Infamous Part 1</a> and <a href="https://www.glowybits.com/blog/2017/01/04/ifl_iss_hdr_2/" target="_blank"> Part 2</a>: Jasmin Patry on supporting HDR especially with SDR UI elements. Lots of practical info regarding ACES tonemapping, and he made Ghost of Tsushima which means he knows everything.</li>
</ul>

<h2 align="center">Implementation Details</h2>

<h3 align="center">HDR Tonemapping</h3>

By doing our lighting pass using a RG11B10F target means we properly represent HDR values already, so the main changes happen in the tonemapping step. We use the Hejl tonemap and then convert to gamma and output to an RGBA8 texture in the SDR pipeline. A naive assumption is we do not need the tonemapper anymore because we can represent HDR on the screen now...but HDR screens do not represent the full 10,000 nits at all so tonemapping is still needed. In the new HDR pipeline we had to create an extension of the Hejl tonemap to allow for ranges greater than 1.0 as an output. It maps onto the orignal Hejl but with much less falloff at the tail.

<div align="center">
<iframe src="https://www.desmos.com/calculator/aehn9phert?embed" align="center" width="500" height="500" style="border: 1px solid #ccc" frameborder=0></iframe>
</div>

Another consideration is that each HDR screen has a different maximum nits, this means the tonemapper needs to adapt for different screens. The adaption factor is calculated as maxNits/10000, this scales the tonemapper to extend or reduce the range of brights. We discuss this more later.

Finally now that we have the tonemapped colour instead of applying the gamma function we apply the PQ function. This is the EOTF which the HDR screen expects and will interpret through to display the image with the desired brightnesses. It is a non-linear function which maps 0-1 to 0-10,000 nits, it more precise at the lower/darker ranges as this is where more obvious banding would occur.

```hlsl
float4 ConvertToHDR10(float4 hdrSceneValue, float paperWhiteNits)
{   
    float3 rec2020 = mul(from709to2020, hdrSceneValue.rgb);                             // Rotate Rec.709 color primaries into Rec.2020 color primaries
    float3 normalizedLinearValue = NormalizeHDRSceneValue(rec2020, paperWhiteNits);     // Normalize using paper white nits to prepare for ST.2084     
    float3 HDR10 = LinearToST2084(normalizedLinearValue);                               // Apply ST.2084 curve

    return float4(HDR10.rgb, hdrSceneValue.a);
}
```
<br>
<h3 align="center">Blending UI</h3>

Unfortunately HDR PQ and sRGB UI do not alpha blend properly...so we need a common space to blend them in called sRGB10, this is described in Jasmin Patry's blog. In the HDR pipeline we now render the UI into sRGB10 instead of sRGB, and during our tonemapper instead of PQ we convert to sRGB10. Now blending works properly, we can have transparent UI on top, after blending we can run a fullscreen PQ function pass to get our HDR output.

```hlsl
// both the UI and tonemapper outputs to this encoding.
srgb10 = Linear_To_SRGB(linearColour)/Linear_To_SRGB(10);

// in a later pass after compositing the UI and the 3D scene convert to HDR10 (PQ)
linearColour = SRBG_To_Linear(srgb10) * 10.0;
ConvertToHDR10(linearColour, paperWhiteNits);
```
You might've noticed we never discussed where we get paperWhiteNits or maximum nits from, that leads us into the final section of HDR calibration.

<br>
<h3 align="center">HDR Calibration</h3>

So we now have <a href="https://displayhdr.org/certified-products/" target="_blank">HDR standards</a> the problem is no one cares especially people marketing displays. Not sure what happened but most displays claim HDR10, which just means they are capable of accepting the HDR10 PQ signal and displaying some sort of result. It does not mean it actually reaches any brightness exceeding SDR and most often they just apply a tonemapper onto the 0-10,000 curve and ruin your perfectly curated image. Alternatively it is great when displays follow the standards like HDR400 because they will not apply any weird tonemapper to ruin your image. But HDR400 can only reach 400 nits! So anything you want above that will be clipped.

This means we need to know the Maximum Nits to adjust our tonemapper so it displays all brights as best as possible on your screen. This maximum brightness value is usually given to us by console APIs by tools like the PS5 HDR calibration screen available to consumers.

```hlsl
// Based off Hejl curve with easier alterable parameters
float3 ToneMap_Hejl_HDR_Dynamic(float3 hdr, float maxDisplayInterpolator)
{
	// maxDisplayInterpolator = (maxDisplayNits / peakTonemapNits) 
	// peakTonemapNits = 1000.0 nit
	float a = 1.11;
	float b = 0.9;
	float c = max(1.0 - maxDisplayInterpolator, 0.01);
	float d = 1.05;
	float e = 1.67;

	float3 x = max(0.0, hdr - 0.004);
	return (x.rgb * (a * x.rgb + b)) / (x.rgb * (c * x.rgb + d) + e);
}
```

While this is great, for some reason they don't give us Paperwhite Nits which is very important!!!

Paperwhite nits defines what the user considers the maximum SDR brightness (like how bright text should be), this can change depending on the viewing environment. In a dark room 80 nits may seem very bright, but in a typically lit room to achieve a decent bright white just for reading text - paperwhite ends up being around 200+ nits. So we need to calibrate our game's paperwhite like we do for gamma. The best way (according to microsoft too) is to just show an image of the game in different environments, and let the user decide what looks good.


<h2 align="center">End Notes</h2><br>

There is a lot to HDR, I probably missed somethings, hopefully the links will fill any gaps. But best method is to get a HDR screen (at least a HDR400) and test it yourself, you will quicky find the major flaws.

<br><br>
