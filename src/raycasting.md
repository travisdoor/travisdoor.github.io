# Cast the Ray

While working on my game, I encountered a common issue. How to select entities in the game world by clicking on them, and how to handle entity collisions. When you use any modern game engine, those things would most likely be fully covered by a physics engine or simply part of the engine API. But I don’t have such a thing.

Originally, I didn’t want to spend too much time on this; there is a lot more to cover while creating a game from scratch. The laziest approach is relying on AI, just letting it implement all collision-related procedures, and copy-paste them with some possible adjustments. This kinda works until it doesn’t, and you find yourself staring at code you actually don’t understand. So I decided to step back and try to dedicate more time to this.

The usual solution would be to cast a ray from the player/camera position into the world and use the hit information to select an entity or handle collisions, if any.

In this blog post, I’ll try to explain in detail how to implement simple raycasting, focusing on the practical side, and do a little bit of math in a way even I’m able to understand (I’m not really into math).

## What is raycast?

You can think about it as a laser beam pointing from a point in the game world in some direction, and we somehow want to detect if this beam hits something in the world, and get some information about the hit point. So let's define our raycast as origin point `O` and normalized direction vector `D`.

![](raycasting/raycast.svg)

Such a definition of raycast on its own is not enough. We also need to control where we are on this oriented line. Since the direction vector is normalized, we can just multiply it by a scalar, usually called `t` to control how far from the origin we are. Note that when the direction vector is normalized, `t` is actually in world units. So to get, for example, point `P` two units far from the origin in the direction `D` we do `P=O+2D`. 

This brings us to the general definition of our raycast:

![](raycasting/formula.svg)

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

Our plane will be defined by the origin point in space `P0` and a normal vector `N` like this:

![](raycasting/plane.svg)

So in mathematical terms, where `P` is any point lying on the plane, we can write this:

![](raycasting/plane-formula.svg)

Note that the dot product is 0 in case its input vectors are perpendicular.

When we're looking for the intersection point, we can just inject our previous raycast definition into the plane definition and solve it for `t`. Note that I use capital letters for vectors and small letters for scalars (float numbers) here. 

![](raycasting/inject-plane-formula.svg)

Now we can calculate the value of `t`, telling us how far from the raycast origin the ray intersects the plane. Thus, to calculate the hit point position, we can use the raycast definition with our `t` value.

![](raycasting/formula.svg)

As you may have noticed, we divide by `N⋅D`, but the denominator cannot be 0. As already mentioned, the dot product is zero when the input vectors are perpendicular; this means our raycast is perfectly parallel to our plane and thus never hits it. We have to check this explicitly in the code.

The final ray vs plane procedure might be implemented like this:


```bl
ray_vs_plane :: fn (ray: Ray, p0: v3, n: v3, hit: *Hit) bool {
	o :: ray.origin;
	d :: ray.direction;

	denom :: dot(d, n);
	if math.compare(denom, 0.f, EPS) then return false;

	t :: dot(n, sub(p0, o)) / denom;
	if t >= ray.t_min && t <= ray.t_max {
		hit.t = t;
		hit.n = n;
		return true;
	}

	return false;
}
```

## Ray vs Sphere
Similarly, we can find the raycast collision point with a sphere. The whole equation is a bit more complex; however, it applies the same principles. Our sphere is defined as a centre point `P0` and a radius `r`. For any point `P` we can define a sphere as follows:

![](raycasting/sphere.svg)

Note that `||` here represents *vector magnitude*. There is also one simple trick I didn't know, we can represent the squared magnitude of any vector as *dot product* with itself. Thus, we can do the following to make our computation a bit "easier".

![](raycasting/sphere2.svg)

Now we can replace point `P` with the raycast formula:

![](raycasting/sphere3.svg)

In the next steps, we solve this equation for `t`. Since it's easy for me to mess up, I do some substitutions here and there to reduce the risk (`L` and `M` variables).

![](raycasting/sphere4.svg)

Finally, some rearrangement:

![](raycasting/sphere5.svg)

At this point, it's getting quite obvious that we're dealing with a quadratic equation: 

![](raycasting/sphere6.svg)

We can write down all coefficients:

![](raycasting/sphere7.svg)

To solve `t` we can use the quadratic formula (google it):

![](raycasting/sphere8.svg)

Let's take a look at the *determinant* first:

![](raycasting/sphere9.svg)

In case the *determinant* is negative, we cannot calculate its square root. Meaning the raycast completely missed the sphere.

In case the *determinant* value is zero, the quadratic equation has only one solution, meaning the raycast just touched the sphere at a single point.

Otherwise, we have to pick `t` greater than `t_min` and one closer to the raycast origin.

![](raycasting/sphere10.svg)

The final ray vs plane procedure might be implemented like this:

```bl
ray_vs_sphere :: fn (ray: Ray, sphere_center: v3, sphere_radius: f32, hit: *Hit) bool {
	o  :: ray.origin;
	d  :: ray.direction;
	p0 :: sphere_center;
	r  :: sphere_radius;

	op :: sub(o, p0);
	a :: dot(d, d);
	b :: 2.f*dot(d, op);
	c :: dot(op, op)-r*r;

	det :: b*b-4.f*a*c;
	// Quadratic equation does not have any solution, ray misses the sphere.
	if det < 0.f then return false;

	a2 :: 2.f*a;
	detsqrt :: math.sqrt(det);
	t1 :: (-b + detsqrt)/a2;
	t2 :: (-b - detsqrt)/a2;

	// Possible cases:
	// t1 < t_min && t2 < t_min: Both hits are behind the raycast origin.
	// t1 < t_min && t2 > t_min: We're inside the sphere.
	// t1 > t_min && t2 > t_min: We're hitting the sphere and we should pick closest point.

	t := t1;
	if t < ray.t_min {
		t = t2;
	}

	if t >= ray.t_min && t <= ray.t_max {
		p :: add(o, mul(d, t));

		hit.t = t;
		hit.n = div(sub(p, p0), r);
		return true;
	}

	return false;
}
```