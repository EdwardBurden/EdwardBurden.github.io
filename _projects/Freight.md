---
layout: project
priority: -1
title: Freight
description: New resource trading game i've been working on and finally starting to show.
status: Active

thumbnail: freight1.PNG
---

Freight is a resource management and logistics game set in a sci-fi future where the player manages a space trading company. Your objective is make money and become a profitable and expansive business. You do this by buying and selling resources from settlements scattered across an alien planet however you must deliver every resource yourself. To meet this challenge you can build a fleet of ships which transport Freight across your empire.You’ll face challenges managing the cost of operating this fleet and must optimize your business to survive.  

Right now I'm still working on Milestone 1 of the project which is set to be achieved in December 2022, i only get to work on the game a few hours a week so progress has been slow but steady. This version of the game will have a variety of settlements with different behaviors, resources produced and consumed. For example farms only produce crops in Autumn forcing the player to adjust there trade routes per season, while other settlements like mines and wells produce goods year round but have a chance to flood or implode causing the player to pay to repair them or find another supplier.

 It will also have ships the player can move around to collect and deliver resources to settlements, these orders can also be chained together and repeated to create efficient trade routes. It’s up to the player to decide the best strategy to make a profitable business and each playthrough should force the player to decide which resources to focus on trading.

<div class="row mb-5">
  <img class="col-md-6 col-12" src="/assets/images/Freight3.PNG"  alt="">
 <div class="col-md-6 col-12">
 <p>
 Here's a look at one of the procedurally generated islands, the islands are made by evenly distributing points on a plane so they have enough distance between each other for the island base. I place down an island base which is a mesh I've already made in Blender and scatter points on the mesh where island blocks are to be created. Then I create a Voronoi diagram from these points, cut the diagram into a shape which culls points outside the island base and then use the Voronoi diagram to create a mesh per Voronoi Cell. </p><p>Thats the simplified explanation of how the islands are made but there's a lot more that goes into it that i might cover in a video in the future. I prefered this technique over hexagons because i wanted my islands to feel very unique and have jagged edges that looks a little more alien and inhospitable.
   </p>
</div>
</div>
<div class="row mb-5">
<img class="col-md-6 col-12" src="/assets/images/Freight2.PNG"  alt="">
 <p class="col-md-6 col-12">
  I started working on Freight around 5 months ago and the progress has been slow but most of all its been consistent improvement. I think this project is one I can see myself finishing even if it means taking a long time like 1-2 years just because of the size of the game's scope.
I’ve also been saving clips of the game at different stages in order to make a video showing the progress in MileStone 1. Being able to go back and see the progress since May has been great motivation to keep working on the project.
     </p>
</div>

<div class="row mb-5">
<img class="col-md-6 col-12" src="/assets/images/freight.gif"  alt="">
 <p class="col-md-6 col-12">
  Here's a little sneek peek of one of the ships i modelled in blender that will be making into the next milestone :)
     </p>
</div>


<div class="embed-responsive embed-responsive-16by9 mb-5">
  <iframe class="embed-responsive-item" src="https://www.youtube.com/embed/CtE99975mFs" allowfullscreen></iframe>
</div>
