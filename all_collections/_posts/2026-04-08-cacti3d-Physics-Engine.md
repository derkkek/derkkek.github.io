---
layout: post
title: Creating a 3D Physics Engine
date: 2026-04-08
categories: [devlog, math, physics]
---

In this blog post i will share my journey and engineering insights of creating a 3d physics engine. Since there are quite a few sources on this topic i hope this will be beneficial for you :).

# Starting Out

I needed to decide what is a physics engine and what is not? The answer was easy but in the same way it wasn't because quite of the tutorials and sources don't mention decoupling like they mostly build engines and their demo in the same visual studio project guess what when you finish the tutorial you don't have an engine in your hand you got a project with lots of dependencies and coupling that cannot be seperated without building the engine from scratch again for your new game. So initially i spend some time on decoupling my engine and my "renderer" and i knew that my engine should be a static library why? Because that's a way that makes what's engine an engine, without it i don't know how to link it against another project or a game and i don't need to recompile the whole engine just for adding a printing statement to my demo.

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