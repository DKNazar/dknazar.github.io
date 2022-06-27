---
title: "Procedural Sky"
permalink: /procedural-sky/
---
<h1 align="center">Procedural Sky and Weather</h1>

For Cricket 22 we needed a next-gen sky solution, the artists needed something much better than some layers of noise with a blue gradient background. 

Originally the task was to create a new cloud system that could use the match weather data as input and be much more dynamic than our old key-frame based system. But before I could even think about clouds I knew we needed to start at the atmosphere.

<div align="center">
  <img src="/images/clouds_Timelapse.gif" width="50%" />
</div>
<br>

<h2 align="center">Physically Based Atmosphere</h2>

The old sky was a completely unmaintained gradient, with crazy HDR values which destroyed the auto-exposure and tone-mapping (a whole other topic). So the first step was to investigate state of the art techniques in PBR atmospheres. The most popular was a series of precomputed LUTs by [Bruneton and Neyr](https://ebruneton.github.io/precomputed_atmospheric_scattering/). These LUTs included transmittance, scattering, and irradiance to build a realistic Earth sky. Although they were generally too expensive to compute in real-time, I imagine most games would slowly compute over time or blend prebaked LUTs. But a more recent paper from [Hillaire](https://sebh.github.io/publications/egsr2020.pdf) at Epic Games took the Bruneton method and reduced the complexity, specifically in the light scattering steps. This sky was faster and much more scalable with the intention to work on mobile.

<h2 align="center">Atmosphere Components</h2>

The papers provide details and implementations but I’ll give a brief overview. The sky functions with three LUTs computed using atmosphere parameters as inputs. These parameters include sun direction, sun illuminance, Mie scattering and Rayleigh scattering co-efficients.

Rayleigh scattering is light spreading by small atmospheric gas particles, this creates the blue in the sky and red in sunsets. This scattering involves a short wavelength bias and that causes a blue sky.

Mie scattering is light spreading through larger particles in the air like mist, fog or pollution. This is less wavelength dependent and therefore comes off as white.

<h2 align="center">LUTs (Look up tables)</h2>

<div align="center">
  <img src="/images/old_sky.png" width="50%" /><br>
  <em>Old, blue gradient sky<em/><br><br>
  <img src="/images/clouds_Timelapse.gif" width="50%" /><br>
  <em>New PBR sky with procedurally generated volumetric clouds<em/><br><br>
</div>
<br>

The LUTs themselves are encoded by an angle from horizon to UV transformation step. So we need to calculate our horizon angle when sampling the LUT which covers the entire atmosphere.

<div align="center">
  <img src="/images/atmos_paper_fig2.png" width="40%" /><br>
  <em>Sky-View LUT example<em/><br><br>
  <img src="/images/atmos_paper_fig1.png" width="40%" /><br>
  <em>Multi-scatter LUT example<em/><br><br>
</div>
<br>

Firstly, the Transmittance LUT is computed, this determines how much sun light is shadowed by the atmosphere ozone (how much light makes it through at that point).

Then using Transmittance, the Multi-Scatter LUT is computed which calculates multi-ordered light scattering. This was a key innovation from Hillarie where their method can “approximate the evaluation of an infinite number of scattering orders.”

Lastly the Sky-View LUT uses these tables to calculate the Mie and Rayleigh scattering results which gives us the colour in the sky.

<div align="center">
  <img src="/images/transmittance.jpg" width="40%" /><br>
  <em>Transmittance LUT<em/><br><br>
  <img src="/images/skyview.jpg" width="40%"><br>
  <em>Sky-View LUT<em/><br><br>
</div>
<br>

Now that all our LUTs are computed we use the Sky-View LUT in our ray-march where we render the sky. In my implementation it is depth-tested so we only march areas with no geometry (the sky only exists in the void).
    
    
<div align="center">
  <img src="/images/sky_early.png" width="50%" /><br>
  <em>Much better than a gradient<em/><br><br>
  <img src="/images/boys_in_void.png" width="50%" /><br>
  <em>The void!<em/><br><br>
</div>
<br>

Now that we have a beautiful atmosphere we need the sun to be correctly positioned. Since we are a simulation game, we need to convert our time and location into an accurate sun to get exact shadows. A paper by [Zhang](https://www.sciencedirect.com/science/article/pii/S0960148121004031) describes a simplified method to calculate this exact thing in polar co-ordinates. Implementing this as input to our atmosphere meant we have realistic sun positions for all stadiums. Which also when we discovered that all stadiums were setup incorrectly, and believed they were in Australia.
A great test for this was [Svalbard](https://www.visitnorway.com/things-to-do/nature-attractions/midnight-sun/), Norway where during summer the sun never sets. While I imagine no one plays cricket that north, it is fantastic to see if we wanted to in our game the sun would never set accurately!

<h2 align="center">Volumetric Clouds</h2>

Now the sky is done, we need clouds!
My personal goal was to make a completely procedural system without need for artists to interfere, unless they wanted to. Reading many different cloud rendering techniques the closest one to our needs was from [Guerrilla Games](https://www.guerrilla-games.com/read/nubis-authoring-real-time-volumetric-cloudscapes-with-the-decima-engine) for Horizon Zero Dawn.

They used two 3D volume noise textures specifically made to emulate the density of clouds. It is then ray-marched while filtering it through a weather map which influences cloud density by height, position, and precipitation.

<div align="center">
  <img src="/images/noiseLow.jpg" width="25%" /><br>
  <em>Low Frequency Perlin-Worely Noise<em/><br><br>
  <img src="/images/noiseHigh.jpg" width="25%" /><br>
  <em>High Frequency Perlin-Worely Noise<em/><br><br>
</div>
<br>

Luckily a lot of the setup was already done for our sky atmosphere ray-march so I just needed to add clouds to the march. The lighting equation is surprisingly simple through [Beer’s law](https://en.wikipedia.org/wiki/Beer%E2%80%93Lambert_law) and the aptly named “powdered sugar” effect. But other factors such as clouds shadowing themselves requires cone sampling adjacent clouds towards the sun which adds complexity. After much trial and error, I finally got a decent looking lighting model.

<div align="center">
  <img src="/images/oldlapse.gif" width="50%" /><br>
  <em>Early time-lapse of our volumetric clouds<em/><br><br>
</div>
<br>
    
I was fairly happy with these clouds except the weather map ruining the shape and creating boxy square clouds!

Guerrilla clouds were made to create dramatic cloudscapes for their fantasy game. So it had a much larger focus on artists able to author fantastical cloud content. Cricket on the other hand can be less dramatic, so generating a weather map based on weather data was a great solution. This required a lot of noise shaping depending on different factors. For example, humidity impacts cloud thickness, rain increases light absorption (and thickness), wind increase noise frequency (creates a more chaotic sky), and of course cloudiness effects total cloud coverage generated in the map. Each weather map is unique to its match, location, and date, so every game is unique!

<br>
<div align="center">
  <img src="/images/weather_mapRG.jpg" width="30%" /><br>
  <img src="/images/weather_mapHeight.jpg" width="20%" />
  <img src="/images/weather_map.jpg" width="20%" />
  <br>
  This is an example of our weather map, red is cloud coverage, green is cloud type (stratus, cumulus, cumulonimbus).
</div>
<br>


<div align="center">
  <img src="/images/cloud-1.png" width="50%" /><br>
  <img src="/images/cloud-2.png" width="50%" /><br>
  <img src="/images/cloud-3.png" width="50%" /><br>
</div>
<br>
  
<br>
<h2 align="center">Optimisations</h2>

This is very implementation specific, but the original Guerrilla Games optimisations are the major factors. These include doing large ray-march steps when not intersecting clouds, more aggressive stencil-testing, and reprojecting across frames. The reprojecting step is the most complicated where you render the clouds at 1/4 resolution and update those select pixels over, slowly building the full frame.


<div align="center">
  <img src="/images/stencil-test.png" width="50%" /><br>
  <em>Stencil-test of stadium vs sky<em/><br><br>
  <img src="/images/reprojectionmeme.png" width="50%" /><br>
  <em>Reprojection only matters where we see the sky<em/><br><br>
</div>
<br>
  
One key difference is Guerilla Games is a Sony studio meaning they built for PS4 which is faster than an Xbox One. On Xbox the noise texture sampling was hitting bandwidth limits hard, the biggest performance increase was to lower the 3D volume textures bits from R32F to R16 and even R8 if needed. It had very little visual impact and made Xbox One very playable with the sky.

    
