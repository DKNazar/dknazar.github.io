---
layout: page
title: "Procedural Sky"
permalink: /procedural-sky/
---
<h1 align="center">Procedural Sky and Weather</h1>



For Cricket 22 we needed a next-gen sky solution, the artists needed something much better than some layers of noise with a blue gradient background. 

Originally the task was to create a new cloud system that could use the match weather data as input and be much more dynamic than our old key-frame based system. But before I could even think about clouds I knew we needed to start at the atmosphere.

| ![Old Sky](https://user-images.githubusercontent.com/9927690/175769991-b1a08428-98b9-444b-adc9-80b0b6773fa9.png) |
|:--:|
| *Old unrealistic blue graident sky*|

<h2 align="center">Physically Based Atmosphere</h2>

The old sky was a completely unmaintained gradient, with crazy HDR values which destroyed the auto-exposure and tone-mapping (a whole other topic). So the first step was to investigate state of the art techniques in PBR atmospheres. The most popular was a series of precomputed LUTs by [Bruneton and Neyr](https://ebruneton.github.io/precomputed_atmospheric_scattering/). These LUTs included transmittance, scattering, and irradiance to build a realistic Earth sky. Although they were generally too expensive to compute in real-time, I imagine most games would slowly compute over time or blend prebaked LUTs. But a more recent paper from [Hillaire](https://sebh.github.io/publications/egsr2020.pdf) at Epic Games took the Bruneton method and reduced the complexity, specifically in the light scattering steps. This sky was faster and much more scalable with the intention to work on mobile.

<h2 align="center">Atmosphere Components</h2>

he papers provide details and implementations but I’ll give a brief overview. The sky functions with three LUTs computed using atmosphere parameters as inputs. These parameters include sun direction, sun illuminance, Mie scattering and Rayleigh scattering co-efficients.

Rayleigh scattering is light spreading by small atmospheric gas particles, this creates the blue in the sky and red in sunsets. This scattering involves a short wavelength bias and that causes a blue sky.

Mie scattering is light spreading through larger particles in the air like mist, fog or pollution. This is less wavelength dependent and therefore comes off as white.

<h2 align="center">LUTs (Look up tables)</h2>

The LUTs themselves are encoded by an angle from horizon to UV transformation step. So we need to calculate our horizon angle when sampling the LUT which covers the entire atmosphere.

Firstly, the Transmittance LUT is computed, this determines how much sun light is shadowed by the atmosphere ozone (how much light makes it through at that point).

Then using Transmittance, the Multi-Scatter LUT is computed which calculates multi-ordered light scattering. This was a key innovation from Hillarie where their method can “approximate the evaluation of an infinite number of scattering orders.”

Lastly the Sky-View LUT uses these tables to calculate the Mie and Rayleigh scattering results which gives us the colour in the sky.
