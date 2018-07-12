# How Roblox things show up in microprofiler: a confounding guide to figuring out performance issues
 This material reveals some of the internal workings of the engine and tries to explain some common bottlenecks a game can inadvertently trigger.

## Overview
Roblox engine schedules all significant work to several threads, that show up on the main screen as vertically separated areas. Cryptic labels like "0000:GPU" or "84d8: TS 04B45F90" on the left-hand side of the screen are thread names, the most important being "0000: GPU" (graphics card), "####: Main" (main thread), and "####: TS #####" (Task Scheduler threads), where the majority of the work occurs.

## Rendering
Rendering starts on one of the Task Scheduler threads: you can find a bar called "Render" that spans for quite some time (up to tens of milliseconds). This is our first stop.

If you have any scripts bound to RenderStepped event, you'll see a bar called Step under Render. This is exactly how it works - the engine honestly calls your scripts before it renders a new frame. This is also one of the bottlenecks: too much work in RenderStepped will directly affect the framerate because the rendering cannot proceed until the script is done processing. This is exactly why we strongly recommend putting the entirety of your game logic to Heartbeat instead. Ideally, only things that must be in sync with the camera should be processed in RenderStepped.

Notice that after Step, there isn't much under Render; nothing at all in fact. Note, however, that the end of Step coincides with "Prepare" on the main thread. This is no coincidence: Render schedules some work on the main thread (due to limitations of Direct3D9 and OpenGL), and then waits for a while - we'll get to it.

Our next stop is the Prepare bar on the main thread. This is where the 3D engine examines the state of the world, and figures our what needs to change graphics-wise. Three important bars you can encounter under Prepare are:

1. Pass3DAdorn
2. Pass2D
   
This is where the GUI is processed, 3D for SurfaceGUIs, 2D for ScreenGUIs. If these bars pop up, you have too many GUI objects/elements.

1. updatePrepare - this is where the 3D engine figures out parts, major contributing performance-degrading factors are: too many parts moving/changing visual properties, e.g. transparency; and some initial light grid computations.

The 3D engine is well aware of these two bottlenecks, and tries to limit the amount of work dedicated to updates by delaying the remaining work to the next frame, but the process is not entirely perfect.

The geometry update work is sliced into update buckets that show up as "updateGeometry" somewhere under UpdatePrepare. The current time budget for such updates is 4 milliseconds per frame, but if you have a lot of polygon-heavy meshes or CSG parts moving, or other parts moving in the vicinity of heavy meshes/CSGs, the update job can take way longer than 4 ms.

It's pretty hard to make a solid recommendation here, but the worst-case performance is going to be when you have a single part moving for a few seconds, stopping for a few seconds, and moving again - this will trigger geometry updates on state transitions (moving/sleeping). This is also why we don't recommend custom Level-of-Detail solutions: an LoD switch on a single mesh part triggers a geometry update of everything in the vicinity of it. Reducing the number of parts and polygon counts on meshes/CSGs helps a lot, esp. the framerate stability.

The second part of updatePrepare is some initial Lightgrid work, where it tries to figure out which parts it needs to re-render. This is usually fast, but a lot of parts on the same spot can take quite a while to process. If this pops up, consider reducing the number of parts in that area.

Now, all this work, as the name suggests, is merely _preparatory_, i.e. nothing is actually being rendered yet, just being updated for rendering. Notice that the end of updatePrepare coincides with the end of Render. Also notice that after this, something else starts running on one of the Task Scheduler threads. What happens here is the end of the so called "Datamodel write lock", by which only one job can access and change the state of the world. (Remember that RenderStepped calls into developer scripts that can change anything in the game.) So the 3D engine is designed to quicky scoop up the relevant object state/updates while the lock is on, and then proceed to rendering and never look at the state of the world until the next frame. In the meantime, other important jobs (Physics, Networking) may proceed work with the game data.

Which brings us to our third stop: _Perform_ on the main thread. No more dawdling: _Perform_ performs rendering.

Three important things happen during _Perform_:

1. Lightgrid rendering (under computeLightingPerform)

This is where our Lightgrid rasterizes polygons into a 3D grid of voxels. A lot of parts as well as part size greatly affect the performance here. On the bright side, Lightgrid does not render all of the parts every frame. In fact, if your player character (technically, it's rather the camera's point of focus) is stationary, and there are no moving parts in a certain area, Lightgrid does not re-render the voxels for that area.

If this pops up constantly across many frames, consider reducing the number of parts as well as the speed at which the camera can travel.

1. Scene rendering (under Scene)
This is where all the objects are sent for rendering to the graphics card.
Major points of interest are: 
    1. queryFrustumOrdered - figures out what's on the screen - too many overly long parts
    2. updateRenderQueue - figures out the order of rendering - too many parts
    3. Id_Opaque - submits non-transparent parts - too many parts
    4. Id_Transparent (all) - submits transparent parts, decals, etc. - too many transparent parts. The 3D engine and the graphics card are much more sensitive to the amount of transparent objects being rendered, than anything else.
    5. render3D/render2D - submits GUIs - too many GUI elements
 
2. Graphics API/GPU driver submission (under Present)

This is where the Direct3D or OpenGL runtime sends all of the rendering work to the graphics card. There's no way to tell what's happening inside, because it's operating system code, but there's one thing to look out for.

Modern GPUs operate independently (asynchronously) of the CPU, so the CPU can do other things while the GPU deals with the actual rendering. The problem is that all GPUs have different computing power, so different GPUs will take different amount of time to render the same scene. So in order to not let the GPU lag behind too much (i.e. render the frame that should've been rendered two seconds ago), the 3D engine waits for the GPU to finish rendering the _previous_ frame before telling the OS/driver to proceed with the current one. This waiting time is also accounted for under _Present_, so if a user's GPU is taking more than the current frame time (16.67 for 60fps) to render the previous frame, the _Present_ bar will pop up, indicating a CPU-side rendering stall.

Our last stop is going to be the GPU itself. (This feature is unavailable on Mac, mobile and is disabled on Windows PCs for Intel integrated graphics chips.)

The 0000: GPU thread is a "pseudothread": it displays timings taken from the GPU directly (via queries), which are fairly accurate, but depending on the actual hardware, the driver may submit additional GPU work without anyone knowing it, timings of which will be implicitly added to (any of) the GPU bars.

Normally, the GPU starts processing the work at the end of _Present_, when the GPU on-board command processor receives our rendering command lists from the OS/driver.

Several points of interest here:

1. Id_Opaque - this is where all the non-transparent parts are rendered. What matters is not just how many polygons there are to render, but also how big they appear on the screen. The bigger they are, the more per-pixel work (pixel shaders) the GPU has to perform in order to render them. You can figure out the impact of pixel shaders on your game by resizing the main window (or the main view tab if you're using Roblox Studio), and observing how the length of this bar changes with respect to the total number of pixels the window occupies. If it scales linearly, then the part rendering is definitely pixel-bound.

2. Id_Transparent (all) - transparent objects. Same as above, only that transparent objects are considerably more expensive to render, because they create and additional GPU memory bandwidth pressure as well as a few other effects.

3. SSAO, Blur, SunRays, etc. - those pertain to the post-processing effects, and they are always pixel-bound. However, their worst-cast performance is constant too. With their performance only scaling with the size of the render window, you should be able to anticipate the performance drain and counteract accordingly by e.g. having fewer parts (esp. transparent) in the game.

---
Special thanks to wiki user MaxV as this was transcribed from [MaxV](http://wiki.roblox.com/index.php?title=User:MaxV "MaxV")
