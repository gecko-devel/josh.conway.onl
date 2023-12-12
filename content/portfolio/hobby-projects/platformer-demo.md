---
title: 2D Parkour Prototype
portfolioCover: img/platformer-cover.png
weight: 4
---
Created August 2022

<iframe src="/games/platformer-demo/Platformer2.html"></iframe>

(video below in case you can't play)

# Controls

- **A** and **D** to run.
- **Space** to jump.
- **S** to crouch.
  - **Crouch** at speed to slide.
- **Jump** at speed along a billboard to wall-run.
- **Jump** while on a wall to wall-jump.
  - Use **WASD** to angle your wall-jump.

# What is it?

- **Engine**: Godot 3
- **Language**: GDScript

This is a 2D platforming demo with parkour-style gameplay with the goal of incorporating various features that contribute to **game feel**.

These features include:
- **Wall-Jump Coyote Time**, letting the player wall-jump even after setting their input to the opposite direction of the wall, just for a second.
- **Variable jump height** that lets the player choose how high they jump by holding Space.
- **Jump buffering**, which detects jump input *just* before the player hits the ground and buffers it so they jump when they land.
- **Slide boosting** gives a speed boost to the player when they slide. This recharges after a short while to prevent abuse.

# Video
{{< youtube JxtUzDhFnIU >}}

# Credits

- **Assets used**:
	- [BrickTileMap](https://arithmetic.itch.io/bricktilemap) by Arithmetic
	- [Midi Waffle](https://opengameart.org/content/midi-waffle) by Kelvin Shadewing.