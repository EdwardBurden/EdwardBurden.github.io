---
layout: project
priority: 10
title: Loot & Boot
description: Pixel art based dungeon crawler game idea/prototype
status: Closed
thumbnail: Loot1.PNG
---

I was playing Hades and getting excited for the next Diablo release so I thought I would have a little look online to learn about generating dungeons and thought I might as well have a go implementing in Unity.

I used this article from <a href="https://www.gamedeveloper.com/programming/procedural-dungeon-generation-algorithm">gamasutra</a> (which rebranded i guess)
And had some fun replicating part of it but also had some problems applying it to an isometric game.
I started getting some ideas for a fun pixel art style game about exploring an underground flooded city in a cyberpunk future. Only problem was that I had no art experience and I soon realized the project would be very art intensive so I decided to stop working on it.

If you go to the annoucement update for the game you can walk around a dungeon but there isn't much else to do im afraid.

I also had a big problem wil walls and doors so that why they look so weird in the pics below.

<div class="row mb-5">
 <img class="col-6" src="/assets/images/loot2.PNG" alt="">
    <img class="col-6" src="/assets/images/loot3.PNG" alt="">
</div>

The difficult part of making it work was using Delaunay triangulation to create a graph of the dungeon room and then making a minimum spanning tree for connections between the rooms which then became corridors.
  
I started then using the Unity rule tile package to make the walls work but realised there would need to be lots of wall variations because of the proximity of the rooms and possible tile combinations.
<div class="embed-responsive embed-responsive-16by9 mb-5">
  <iframe class="embed-responsive-item" src="https://www.youtube.com/embed/6bjCG9yr8ng" allowfullscreen></iframe>
</div>