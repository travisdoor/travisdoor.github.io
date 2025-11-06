# Cast the Ray

While working on my game, I encountered a common issue. How to select entities in the game world by clicking on them, and how to handle entity collisions. When you use any modern game engine, those things would most likely be fully covered by a physics engine or simply part of the engine API. But I don’t have such a thing.

Originally, I didn’t want to spend too much time on this; there is a lot more to cover while creating a game from scratch. The laziest approach is relying on AI, just letting it implement all collision-related procedures, and copy-paste them with some possible adjustements. This kinda works until it doesn’t, and you find yourself staring at code you actually don’t understand. So I decided to step back and try to dedicate more time to this.

The usual solution would be to cast a ray from the player/camera position into the world and use the hit information to select an entity or handle collisions, if any.

In this blog post, I’ll try to explain in detail how to implement simple raycasting, focusing on the practical side, and do a little bit of math in a way even I’m able to understand (I’m not really into math).

## What is raycast?

You can think about it as a laser beam pointing from a point in the game world in some direction, and we somehow want to detect if this beam hits something in the world, and get some information about the hit point. So let's define our raycast as origin point `O` and normalized direction vector `D`.

![](raycasting-raycast.svg)

Such a definition of raycast on its own is not enough. We also need to control where we are on this oriented line. Since the direction vector is normalized, we can just multiply it by a scalar, usually called `t` to control how far from the origin we are. Note that when the direction vector is normalized, `t` is actually in world units. So to get, for example, point `P` two units far from the origin in the direction `D` we do `P=O+2D`.

This brings us to the general definition of our raycast:

![](raycasting-formula.svg)

From a practical side, it's good to define some bounds for the `t` value. We probably don't want it to be negative, and we usually don't want to include hits too far from the origin. To have full control over these properties, we can define `t_min` and `t_max` values.

So the full definition of the raycast in code looks like this:

```bl
Ray :: struct {
	origin: v3;
	direction: v3;
	t_min: f32;
	t_max: f32;
}
```

## Hit Point

By hit point, we mean the exact intersection point of our raycast and the object in the scene. In other words, we're trying to find one point lying on the raycast and geometry at the same time. Based on the raycast definition, we're looking for `t` value of such a point.

To store collected information about the hit point, I use the following structure:

```bl
Hit :: struct {
	id: u32;
	t:  f32;
	n:  v3;
}
```

Where `id` is a unique identifier of the hit object, `t` value and hit surface normal `n`. You can include any other information needed.

As you can imagine, calculating the intersection of our ray with a full model triangle geometry might be too complex to start with, so let's take a look at the simplest case possible first.

## Ray vs Plane

Probably the simplest case we can implement is checking if our raycast intersects a plane. Crucial construct used in the following calculations is *dot product*, I'll not go into details explaining it here, you can watch [video](https://www.youtube.com/live/fjOdtSu4Lm4?si=qgWHYcV_Co30lcU0) from Freya Holmér containing a very detailed explanation.

Let's just highlight some algebra-like properties of the dot product (`⋅`):

* Commutative: `A⋅B=B⋅A`
* Distributive: `A⋅(B+C)=A⋅B+A⋅C`
* Scalar multiplication: `(sA)⋅B=s(A⋅B)`

Our plane will be defined by the origin point in space `P0` and its normal vector `N` like this:

![](raycasting-plane.svg)

So in mathematical terms, where `P` is any point lying on the plane, we can write this:

![](raycasting-plane-formula.svg)

Note that the dot product is 0 in case its input vectors are perpendicular.

When we're looking for the intersection point, we can just inject our previous raycast definition into the plane definition. Note that I use capital letters for vectors and small letters for scalars (float numbers) here.

![](raycasting-inject-plane-formula.svg)

As you may have noticed, we divide by `N⋅D`, but the denominator cannot be 0. As already mentioned, the dot product is zero in case the input vectors are perpendicular; in this case, it means our raycast is perfectly parallel to our plane, thus never hitting it. We have to check this explicitly in the code.

Now we can calculate the value of `t`, telling us how far from the raycast origin the ray intersects the plane. Thus, to calculate the hit point position, we can use the raycast definition with our `t` value.

![](raycasting-formula.svg)

The final ray vs plane procedure might be implemented like this: