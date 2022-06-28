---
title: "Level Design Tools"
permalink: /level-design-tool-1/
---
<h1 align="center">Accidentally Developing a Level Design Tool</h1>
<br>

Being a solo dev made me realise how important good tools are to make a game. As a programmer I can configure things as a I like, but as a designer getting too technical all the time makes progress very slow. So I devoted some time to make specific level design tools for my needs. These are very ad-hoc but I think they turned out pretty well.

Originally my journey into tools started off as a procedural house builder for the village and city levels. There are three main components, placing the meshes, creating the roof, and setting up the materials.

<div align="center">
  <img src="/images/HouseAnimBuild.gif" width="600" />
</div>
<br>

This was a very straight forward approach; I hijack the line render tool to create an outline or floor-plan for the house. Then I populate this outline with a library of different house piece quads, like a door section, circle window, square window or other features. To avoid sizing issues the floor-plans can be snapped into perfect distances to avoid squishing segments

<div align="center">
  <img src="/images/houseEditorSmall.gif" width="400" /><br>
  <em> House editor with point snapping</em>
</div>
<br>

After the quads have been placed I use this outline to create a roof on top. Here I create a polygon of the outline which connects to the wall meshes underneath and then use Delaunay triangulation to fill the roof gap.

<div align="center">
<img src="/images/roofTop.png" width="400" /><br>
<em>Generated rooftop geometry</em>
</div>
<br>

Now this geometry is ready I run a mesh combiner, which grabs all the pieces and combines them into one mesh which is much better for performance. They all use the same material, and UVs been setup in Blender per mesh to align to an atlas texture.

<div align="center">
<img src="/images/atlas_blender.png" width="400" /><br>

<em>Blender house tile setup</em><br>

<img src="/images/house_atlas0.png" width="400" /><br>

<em>Wall textures atlas</em><br>

<img src="/images/house_atlas1.png" width="400" /><br>

<em>Wall materials mask atlas</em><br>
</div><br>


This house building system later extended to an entire generic level design tool. I build walls, cliffs, and buildings with a generic version of this tool. Most of my level has been created with this tool, since the aesthetic of folded paper allows for hard edge designs.

<div align="center">
<img src="/images/floorplan_geo.png" width="400" /><br>
<em>Floor-plan for cliffs + grass plane</em>
</div><br>


This is still WIP and could go in many directions but it so far has made levels much simpler to make. It also benefits players as the levels are more consistent and readable since they share the same creation process.


