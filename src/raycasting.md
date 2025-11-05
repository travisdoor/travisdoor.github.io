# Cast the Ray

While working on my game, I encountered a common issue. How to select entities in the game world by clicking on them, and how to handle entity collisions. When you use any modern game engine, those things would most likely be fully covered by a physics engine or simply part of the engine API. But I don’t have such a thing.

Originally, I didn’t want to spend too much time on this; there is a lot more to cover while creating a game from scratch. The laziest approach is relying on AI, just letting it implement all collision-related procedures, and copy-paste them with some possible adjustements. This kinda works until it doesn’t, and you find yourself staring at code you actually don’t understand. So I decided to step back and try to dedicate more time to this.

The usual solution would be to cast a ray from the player/camera position into the world and use the hit information to select an entity or handle collisions, if any.

In this blog post, I’ll try to explain in detail how to implement simple raycasting, focusing on the practical side, and do a little bit of math in a way even I’m able to understand (I’m not really into math).

## What is raycast?

You can think about it as a laser beam pointing from a point in the game world in some direction, and we somehow want to detect if this beam hits something in the world, and get some information about the hit point.

![](raycast.png)