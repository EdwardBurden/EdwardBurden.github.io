---
layout: project
priority: 2
title: Asteroid Game
description: Simple Unity Game made for a tech test.
status: Closed
github: https://github.com/EdwardBurden/Asteroids
youtube: https://www.youtube.com/watch?v=FH3KS-xamOc
thumbnail: Asteroid_thumbnail.png
---

<div class="row align-items-center mb-5">
<div class="col-md-6" markdown="1">

## Overview

Asteroid Game is an asteroid shooter built in Unity for a technical test. You pilot a ship, fight your way through waves of asteroids, and try to survive as long as you can.

The twist is that there are two very different ships to play as. The video below shows both in action.

</div>
<div class="col-md-6 mt-3 mt-md-0">
<img class="img-fluid rounded shadow" src="/assets/images/asteroid2.png" alt="Asteroid Game gameplay screenshot">
</div>
</div>

<div class="embed-responsive embed-responsive-16by9 mb-5 rounded shadow">
<iframe class="embed-responsive-item" src="https://www.youtube.com/embed/FH3KS-xamOc?si=2-9JB_OWVGnbjIzQ" allowfullscreen></iframe>
</div>

<div class="row align-items-center mb-5">
<div class="col-md-6" markdown="1">

## The brief

The test asked for an asteroids game built in **6 hours**, with **a new, unique idea** that set it apart from the original.

Six hours is not much time, so I knew from the start that I needed an idea that was small enough to finish but different enough to change how the game feels to play. I also wanted a structure that would let me build quickly without painting myself into a corner.

</div>
<div class="col-md-6 mt-3 mt-md-0">
<img class="img-fluid rounded shadow" src="/assets/images/asteroid3.png" alt="Asteroid Game screenshot showing gameplay">
</div>
</div>

<div class="row align-items-center mb-5">
<div class="col-md-6 order-md-2" markdown="1">

## My idea: a second character

In a normal asteroids game you shoot the rocks and take damage when they hit you. My idea was to add a second playable character that flips that around.

The second ship **can't shoot**. Instead it:

- **Lays mines** that asteroids have to run into.
- **Recovers health over time.**
- **Has less health** than the standard ship.

</div>
<div class="col-md-6 order-md-1 mt-3 mt-md-0">
<img class="img-fluid rounded shadow" src="/assets/images/asteroid1.png" alt="Asteroid Game screenshot of the second character">
</div>
</div>

Taking away the gun changes how you think about the whole game. You have to be much more careful about avoiding asteroids, and you have to plan where to place mines so the asteroids run into them.

Because your health regenerates, there is also a risk and reward choice. You can play it safe, or you can deliberately smash through an asteroid and let your health recover afterwards. The low health pool keeps that decision tense.

## How I built it

With only six hours, I wanted to focus on **making the game work quickly with a system that was easy to change**. The aim was flexible game systems rather than polished one-off code.

### Reusable components

I built small, reusable components for the core behaviours, such as **health**, **movement**, and **damaging**. Each one can be added to or removed from a prefab, so new combinations come together quickly. A new enemy or a new player is largely a case of picking the components it needs.

This sped up development a lot, and it left the project open to expansion and changes. The second character itself is a good example, since it is built from the same pieces as the first with a different combination and different settings.

<div class="row align-items-center mb-5">
<div class="col-md-6" markdown="1">

### Data-driven with ScriptableObjects

On top of the components, I used ScriptableObjects to hold the game data. **Data dictates behaviour**: each player type is its own asset with its own gameplay behaviours, and level data, enemies and health can all be configured there too.

Switching players, or rebalancing the game, means editing an asset and not changing code.

### Singletons, used on purpose

I relied on singletons for a few of the central systems. I did that carefully but deliberately, because they let me get everything talking to each other quickly. In a larger project I might structure that differently, but under a six hour deadline it was the right trade-off to get a working game.

</div>
<div class="col-md-6 mt-3 mt-md-0">
<img class="img-fluid rounded shadow" src="/assets/images/asteroid4.png" alt="Asteroid Game screenshot of a later level">
</div>
</div>

## The art

I made all the artwork myself, and I made it quickly. It has a homemade, programmer-art look, which I ended up liking quite a lot. It suits the project, and it kept my time focused on the gameplay and systems.

## What I took from it

Building the game around reusable components and data kept the codebase small and flexible, and made it easy to experiment under a tight deadline. It also showed me that a single simple twist, in this case taking away the gun and adding mines, can change a very familiar game into something that feels fresh.