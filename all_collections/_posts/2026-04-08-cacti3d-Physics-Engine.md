---
layout: post
title: Creating a 3D Physics Engine
date: 2026-04-08
categories: [devlog, math, physics]
---

I am building a 3D physics engine from scratch in C++, with a focus on clean architecture and decoupled system design.

A common issue in many physics engine tutorials is that they tightly couple the simulation with rendering. While this makes demos easier to build, it prevents the engine from being reusable or integrated into other projects.

To avoid this, I started by designing the engine as a standalone static library, separating it completely from the rendering layer. This allows the physics system to be reused, tested, and extended independently.

In this post, I’ll walk through the architectural decisions behind this approach and the development process of my 3D physics engine.

# Starting Out

The first real design question I had to answer was: what separates a physics engine from a software that uses that engine? The distinction sounds obvious until you look at most tutorials — they build the engine and demo in the same Visual Studio project, and by the time you finish, you don't have an engine you can reuse. You have a tangle of dependencies that can't be separated without starting over. So before writing any physics code, I spent time thinking about decoupling. The engine would be a static library — that's what makes it actually distributable, linkable against another project, and independently compilable. No recompiling the whole engine just to add a debug print in the demo.

# Renderer

My Choice of library was raylib because it is very easy to use and very straight forward. Physics engines are already huge topic so i didn't want to bother with the OpenGL boilerplate. raylib gives me everything i need a camera, shaders, texture structs, a window and etc. And since it's simple it could be readable from a person without a prior graphics programming experience. 

```c#
#include "Physics/ShapeBase.h"
#include "Physics/Body.h"
#include "TransformBuffer.h"
#include "Math/Quat.h"

#define SHADOWMAP_RESOLUTION 2048

class RenderModel
{
public:
	Model model;
	Color color;
	static RenderModel BuildFromShape(Cacti::Body body,Cacti::Shape* shape);

	Vector3 position;
	Cacti::Quat orientation;

	RenderModel() = default;
	RenderModel(Model& model, Color color, Vector3 pos);
	void Draw();
};

class Renderer
{
public:
	Renderer();
	~Renderer() = default;

	void Init();
	void Update();
	void Destroy();

	void AddSceneObject(RenderModel& obj);

	std::vector<RenderModel > sceneObjects;

	void SetTransformBuffer(const Cacti::TransformBuffer* buf) { transformBuffer = buf; }


private:
	TextureCubemap GenTextureCubemap(Shader shader, Texture2D panorama, int size, int format);

	RenderTexture2D LoadShadowmapRenderTexture(int width, int height);
	void UnloadShadowmapRenderTexture(RenderTexture2D target);

	const::Cacti::TransformBuffer* transformBuffer = nullptr; // this points to the actual buffer because we don't want to copy buffer values to another buffer.

	Shader shadowShader;
	int lightVPLoc;
	int shadowMapLoc;

	Vector3 lightDir;
	int lightDirLoc;

	RenderTexture2D shadowMap;
	int textureActiveSlot = 10;

	Cam cam;
	Shader shader;

	Model skyboxModel;
	Shader skyboxShader;
};
```

This was my first renderer. It was heavily coupled to the Cacti(engine), notice the includes and cacti's internal transform buffer type. I knew that this shouldn't be like that.

I solved this by the following approach;

Demo is the layer that knows both the engine and the renderer and it's like a choreographer. It tells raylib to create a window, engine to initialize, renderer to load it's shaders, create it's skybox, create it's camera and all and inits the scene by allocating some memory for data conversion to prevent future resizing operations and builds scene geometry from the world objects. so it can be a bridge that allows the communication between them. Ok this sounds nice but how?

# General Architecture

I decided such a simple architecture that engine writes a "transform buffer" of world objects's required data to render them and demo layer converts the internal Cacti type data to our raylib type renderer data as like the following:

<img src="{{ '/assets/architecture.png'}}">

This caused a conversion loop in our game loop but it's cost is negligible and i don't want to push data conversion beyond from that because it wasn't my main interest if it's gonna be bottleneck in the future i would try to make it parallel.

```c#
	while (!WindowShouldClose())
	{
		float dt = GetFrameTime();
		engine.Update(dt);
		
		for (int i = 0; i < engine.world.bodies.size(); i++)
		{
			const Cacti::Vec3 p = engine.transformBuffer.positions[i];
			const Cacti::Quat q = engine.transformBuffer.orientations[i];

			convertedSceneData.positions[i] = { p.x, p.y, p.z };
			convertedSceneData.orientations[i] = { q.x, q.y , q.z, q.w };
		}

		renderer.Update(convertedSceneData);
	}
```

Another diagram  

<img src="{{ '/assets/dataflow.png' | relative_url }}">

Ok it seems like we solved the data transportation without coupling two different subsystems. But what about this? 

"static RenderModel BuildFromShape(Cacti::Body body,Cacti::Shape* shape);"

Basically i used the same approach, i've could carry it to the demo layer because it knows both raylib and the engine. And indeed i did this.

```c#
void Program::InitScene()
{
	renderer.sceneObjects.reserve(engine.world.bodies.size());

	convertedSceneData.positions.resize(engine.transformBuffer.positions.size());
	convertedSceneData.orientations.resize(engine.transformBuffer.orientations.size());

	for (int i = 0; i < engine.world.bodies.size(); i++) 
	{
		RenderModel sceneObject = BuildRenderModelFromPhysicsGeometry(engine.world.bodies[i], engine.world.bodies[i].shape);
		renderer.AddSceneObject(sceneObject);
	}
}
```

Also i realized that RenderModels don't need to carry position and orientation data since it's frame dependent read-only data so since my converted data is a struct of arrays i just draw renderable objects by indexed position and orientation from the arrays in the renderer.

```c#
class RenderModel
{
public:
	Model model;
	Color color;

	RenderModel() = default;
	RenderModel(Model& model, Color color);
	void Draw(const Vector3 pos, const Quaternion& orient);
};
```
Why i didn't make the render models SoA? Well in this case the bottleneck is not data alignment, it's how raylib holds Model data in it's structs. There are various traversal heap allocated fields so that's make raylib Models already cache-hostile.

```c#
typedef struct Model {
    Matrix transform;       // Local transform matrix

    int meshCount;          // Number of meshes
    int materialCount;      // Number of materials
    Mesh *meshes;           // Meshes array
    Material *materials;    // Materials array
    int *meshMaterial;      // Mesh material number

    // Animation data
    int boneCount;          // Number of bones
    BoneInfo *bones;        // Bones information (skeleton)
    Transform *bindPose;    // Bones base transformation (pose)
} Model;
```

So yeah that's enough renderer performance for my case after all, I'm not going to create an industry-standard physics engine.

So i successfully decoupled the subsystems so that in future if want to i can change my renderer or create demos in different graphics libraries or i can distribute my physics engine as a static library easily. These were my initial working progress from this point i will upload my logs daily because i decided to keep a devlog after some progress. 

One last thing before the close-up today, i decided to keep bodies aa stack allocated in bodies vector of the world to make them cache friendly and  

#### You can check the initial state of the project [from here](https://github.com/derkkek/Cacti3D/archive/refs/tags/v0.1.0.zip)

One important notice, after you compile the code copy and paste the resources folder to the out/build/Debug(or release) so the same folder with our exe otherwise it couldn't be able to find shaders and textures.

>I will use "scene objects" to refer renderable, visual representations of our objects and "world objects" to our physical objects that collide and interact with their environments. 

# Day 1

I moved from raw shape pointers to unique pointers in bodies to solve ownership confusion. This required more changes in the codebase for example now body initalization list moves shape pointer to the bodie's shape member variable because unique pointers doesn't allow copying so ownership of the pointer of the initialized shape is moves to the body. 

```c#
	Body::Body(std::unique_ptr<Shape> shape, Vec3 position)
		:position(position), orientation(Quat(0,0,0,1)), shape(std::move(shape))
	{
	}
```

Also BuildRenderModelFromPhysicsGeometry function takes body as reference now because unique pointer doesn't allow to copy body to a function either and to pass raw pointer as a second argument i had to use get() function, 

# Day 2

I did some system description. Because of my previous experiments i knew my system of the engine should be look like the following

>“Every frame the engine takes a list of objects in the world, applies impulses(gravity, drag, etc.), checks if any of them are touching or overlapping, resolves those contacts by applying contact impulses then updates their positions and rotations for the next frame.”

#### Nouns

Engine, object, world, impulse, intersection, contact,

#### Verbs

Resolve, update

Implemented sphere-sphere intersection test, studied why some engines populate contact points as local space some as world space and learned that when you store contact points in world space it causes floating point drifting and can be error-prone because objects' position changes constantly and contact world points became stale and need to being re-calculated in the each frame and impulse solvers re-uses the same contact points so keeping contact in world space makes the calculations stable.

In the other hand keeping world space contact points is beneficial for user/ game code because user can play sounds, create particles and etc. at the world points.

>Erwin Coumans (Bullet’s author) explicitly described it this way in the Bullet forums: keep contacts in local space, transform to world each frame only to validate them.

In summary 
- World space → simpler, fine for non-persistent one-shot solvers

- Local space → necessary for persistent manifolds, warm starting, and stable stacking behavior

I'm sticking with the local space caching.

# Day 3

I studied world space to local space vector conversion and implemented the classic conversion which benefits from quaternions.

Basically i translate the coordinate space by substracting body's position from the vector's world position (remember graph translations from high school) then i create a new quaternion and filling the x, y, z members by using the coordinates of the translated vector and then i apply quaternion rotation to it.

$$
\mathbf{v'} = q \, (0, \mathbf{v}) \, q^{-1}
$$

where $q$ is a rotation quaternion $v$ is a vector we need to convert from world space to local space and $q^{-1}$ is the inverse of the rotation quaternion.

Finally $\mathbf{v'}$ is the translated local point of a body from the world space. 

Furthermore i implemented sphere-sphere collision test and visual debugging to check my coordinate transformations are correct.

Here's the debug visualizations of a contact:

<iframe width="560" height="315" src="https://www.youtube.com/embed/ioYZyehQO3Q?si=ndeJ773AJFGPzQYH" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

# Day 4

I studied momentum and impulse relationship, mainly to understand their role in collision resolution. Momentum and impulses bridge mass and velocity relationship, this saves us from a "velocity guess" of collided objects and tells velocities of collided objects with respect to their massess. 

Also i tried to answer why don’t we just calculate a collision force and apply it to the objects? If we do this, we would need to integrate the force over time to update the velocity, which is computationally expensive and can introduce instabilities, especially over very short collision durations. Impulses, on the other hand, model collisions as instantaneous changes in velocity, making them more suitable for discrete, step-by-step simulations on a CPU.

Furthermore read [Lisitsa Nikita's great blog post about quaternions](https://lisyarus.github.io/blog/posts/introduction-to-quaternions.html) to learn and implement my quaternion class, it's one of the best resources on this topic, he doesn't just provide the formulas; he explains their mathematical basis. Great for those who are curious about the why.

# Day 5

After some reading and a quaternion class implementation finally i managed to rotate my bodies. I don't discuss why do we use quaternions for representing our orientations in here again because there are tons of great explanations on the web.

> But my insights are just don't be scared of it, the core ideas are; [Rotation vectors (axis-angle)](https://youtu.be/PMvIWws8WEo?si=aTiQotgGQmmr1g5l&t=1809) - actually in the end our angular velocity is our rotation axis and angle -, how quaternions encode angles and creating a new orientation quaternion in each frame and multiply it with the member one and then changed the member with it. That wasn't intuitive for me i guess because of i'm familiar with doing things in "OOPy" way that store object's state and manipule them inside in the object's encapsulated environment.

```c#
	void Body::Update(float dt)
	{
		float angle = angularVelocity.GetMagnitude() * dt;
		Vec3 axis = angularVelocity.Normalized();
		Quaternion deltaRotation(axis, angle);
		orientation = deltaRotation * orientation; // this order works for world space rotations.
		orientation.Normalize();
	}
```

Here's the result!

<iframe width="560" height="315" src="https://www.youtube.com/embed/ECo3num-SnI?si=Yy4KslHGqq_nHg3Q" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

# Day 6

I started to implementing collisin resolution. As like other engines my engine also response collisions in two steps;

1. Collision detection.
2. Collision Resolution.

I populate contact structs in the detection step and solve them in resolution step as independently from the collision type (box - sphere or sphere-sphere, etc.)

At first i just depenetrated collisions but i shocked how unstable it is and tried to find mathematical errors in the steps;

<iframe width="560" height="315" src="https://www.youtube.com/embed/HIghVLoAzoY?si=O3WXOfgE4aJCm4VN" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

after few mins of debugging and flipping equations i realized that i don't clear forces from the bodies(i was applying gravity in each frame) at the frame end. After some time little sphere's velocity increase dramatically and causes the instability. So as a collision resolution i just divide it's velocity by two and this is my initial collision response and just for now i just gave little sphere initial velocity without applying gravity for the sake of simplicity;

<iframe width="560" height="315" src="https://www.youtube.com/embed/u8L0CjYqKOc?si=eAjaBK2yHEO1pWCB" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>