[[Linux Concepts]]
[[Twitter Posting]]

![[Graphics_Blog_thumbnail.png]]

Let's start with some disclaimers. The following writeup are my NOTES from these SOURCES — [PT1](https://lwn.net/Articles/955376/) & [PT2](https://lwn.net/Articles/955708/) This is not my original work, and nor do I claim it to be so. With that being said - Hello there lads! 

My name is Kshitij Neeraj Vaze, and I used to  work as a Linux Kernel developer at a hardware startup, around this time last year. While I was trying to figure out things there, one of the first things that I encountered was the process of _board bring-up,_ which is a collection of techniques that us engineers use to take a brand-new development board, and work with it to get it to boot (Kinda, sort of, not really tbh, there’s much more to this).

Now, while I was having my struggles, the problem that I struggled with the most, was to get an HDMI output via the dev board. Which, led me down the rabbit hole of a window manager, desktop environment, and eventually, the entire Linux Graphics Stack!

Following are a set of notes that I made all throughout the week (in a more structured manner), as I attempt to present to you, my reader, the complete working of our beloved Linux Kernel Stack (idk if y’all love it, but I certainly do now!) It is a collection of notes from 2 articles from the amazing lwn.net, which I have linked at the very top. I hope you stick around till then, and check out the blog.

Happy reading y’all!

# 1. Application and Scene Graphs

The story begins with the application, which is responsible for the formatting of the data that is to be visualized. The common data structure used for this is the **scene graph**, which is a collection of nodes of a graph or a tree. The rule followed here is that there can only be one parent of a node, but a node can have many children. The effect of the parent node is applied to _all_ its children — any operation on a parent node is automatically done on its children nodes.

The reason scene graphs exist is because modern graphical applications are deeply hierarchical in nature. A browser window may contain tabs, buttons, text, overlays, animations and images, all of which inherit positioning and transformation rules from parent containers. Rather than recalculating the position and state of every single element every frame, the application stores these relationships structurally as a graph.

This makes operations like scaling, translation, rotation, clipping and opacity propagation significantly easier. If a parent node moves, every child automatically inherits the transformation via matrix propagation through the tree.

Here, nodes of this graph contain the data that is to be visualized. The application will traverse the tree/graph from top to bottom, and from left to right, performing operations on the attributes and rendering the model accordingly onto the on-screen image.

Should be simple enough, init? Yeah… no. As all things Linux, it is _not_.

---

# 2. OpenGL/Vulkan and Buffer Objects

Now, to simplify this mess, the application uses a set of standard APIs like **OpenGL** and **Vulkan**. (_Do I know what they are and what they do? Fuck no. But will I find out soon enough? Bet your life I will!_)

GPUs are not general-purpose processors in the traditional sense. They expose absurdly hardware-specific execution pipelines, memory layouts and synchronization mechanisms. APIs like OpenGL and Vulkan exist to standardize communication between applications and wildly different graphics hardware vendors.

Both provide functionality for the application to interface with graphics memory, fill it with data, and render what is already stored there.

The graphics data is held in [buffer objects](https://www.khronos.org/opengl/wiki/Buffer_Object), which are regions within graphics memory that have handles attached to them. Textures go into Texture Buffer Objects (TBOs), shader data may go into Shader Storage Buffer Objects (SSBOs), vertex information goes elsewhere, and so on.

A draw call is effectively a structured instruction telling the GPU:

- Which buffers to read from
- Which shaders to execute
- What geometry to process
- Where the output framebuffer should be written

Funnily enough, the output image itself is also stored in another buffer object — so you could say that the entire exercise of graphics rendering is just memory management and playing around with pointer referencing and de-referencing, lol.

OpenGL hides a large amount of internal state management from the application, while Vulkan exposes much lower-level control. Vulkan expects the application itself to explicitly manage synchronization, command buffers and memory ownership. In return, it gives significantly lower overhead and finer control over the hardware pipeline.

---

# 3. Shader Magic (aka Minecraft Mods on Steroids)

The application can give data in any format it likes, as long as the **shader** can process it.

_What’s a shader, you say? Is there more to the thing I used to install to make my Minecraft look better, you say?_

Well, a shader is what actually performs the conversion of input data into the final output image. Think of it like a translator for the GPU pipeline — it translates arbitrary graphical data into actual displayable pixels.

It gets values from the application that describe where vertices and pixels exist within a scene, and transforms them into values that tell the kernel and display pipeline where those pixels should appear on the screen.

How does that happen? Simple. Translation matrices and matrix multiplication, of course!

What makes shaders terrifyingly powerful is that they execute massively in parallel. A modern GPU may run thousands of shader invocations simultaneously, each operating on different vertices or pixels at the same time. Graphics rendering is fundamentally a parallel mathematics problem disguised as visual output.

In the most minimal setup, the most common shader operations are:

- **Vertex transformations**
- **Texture lookups**

When it comes to vertex transforms, think of a vertex as a corner of a polygon. To transform the corner from the scene into the application window, a series of matrix operations are performed. This could be something straightforward like affine transformations, or something significantly more cursed involving inverse transformations and projection matrices. The shader is what performs these calculations.

You can loosely think of it like this:

```
Vertex Shader:3D Object -> Screen CoordinatesFragment Shader:Screen Coordinates -> Final Pixel Color
```

The output of the shader is typically held in an output buffer. It is usually a colored pixel paired with a depth value `z`. Running these shader instructions across the entire scene graph, node-by-node and pixel-by-pixel, generates the complete image for the application.

---

# 4. Enter Mesa: The GPU Whisperer

Jumping over to the Linux side of things now — the rendering interfaces and support for various hardware are provided by the **Mesa 3D Library (Mesa, for short)**.

Mesa sits in user-space and acts as the gigantic translation layer between high-level graphics APIs and the Linux kernel graphics subsystem. It is not the kernel driver itself, but rather the compatibility layer that makes Linux graphics remotely sane.

Without Mesa, every application would effectively need vendor-specific implementations for every GPU architecture in existence, which would be absolute chaos.

To applications, Mesa offers:

- **OpenGL / Vulkan** for graphics
- **OpenGL ES** for embedded/mobile systems

On the hardware side, Mesa implements drivers for most modern graphics hardware. It provides shared infrastructure and common logic like:

- OpenGL function parsing
- Shader compilation
- Pipeline handling
- State tracking

The **GPU-specific driver** only fills in the hardware-specific blanks.

For **stateful interfaces** like OpenGL, Mesa’s **Gallium3D framework** connects interfaces and drivers through something called a **state tracker**.

Mesa contains state trackers for various versions of:

- OpenGL
- OpenGL ES
- OpenCL

When the application uses the API, it modifies the state tracker. A hardware driver inside Mesa then converts this state information into actual hardware rendering instructions.

For **stateless interfaces** like Vulkan, Mesa instead exposes a Vulkan runtime. Vulkan shifts much more responsibility onto the application itself, which is why Gallium3D may not even be required for some Vulkan-capable hardware stacks.

**Zink** is one of the funniest things in this ecosystem. It is a Mesa driver that maps Gallium3D to Vulkan. With Zink, OpenGL state becomes Gallium3D state, which then gets translated into Vulkan commands before finally reaching the hardware.

Because apparently abstraction layers were not enough already.

---

# 5. Memory Management Mayhem

Any memory accessible to the graphics hardware is called **graphics memory**. This is the foundation on which the entire stack stands.

More memory = happier graphics pipeline.

Graphics workloads are absurdly bandwidth-heavy. High-resolution textures, depth buffers, shader data and framebuffers constantly move between memory and execution units. A shocking amount of graphics engineering is therefore not about rendering itself, but about minimizing unnecessary memory movement.

Graphics memory can be:

- Dedicated VRAM on graphics cards
- Shared RAM on SoCs
- System memory acting as shadow buffers

Access to this memory is managed by the **Direct Rendering Manager (DRM)** subsystem.

Mesa exposes DRM devices under `/dev/dri`, where applications communicate with graphics hardware via device files corresponding to the GPU.

The device file exposes graphics memory in the form of buffer objects.

**Translation Table Manager (TTM)** is the memory manager commonly used by Intel, AMD and NVIDIA drivers. It handles:

- Discrete graphics memory
- System memory
- Remapped memory regions

For simpler framebuffer devices, **SHMEM drivers** allocate graphics buffers directly in shared memory. This is extremely common in embedded systems and SoCs where the CPU and GPU share the same physical RAM.

In such systems, synchronization becomes critically important because both processors may operate on the same memory simultaneously.

And then there are the **DMA managers**, which provide direct access to graphics memory through DMA-capable memory regions.

---

# 6. GEM: The Middleman

Each DRM driver implements a **Graphics Execution Manager (GEM)**.

You can think of GEM as the bookkeeping layer of the Linux graphics stack. It tracks ownership, visibility and lifecycle information for graphics buffers while abstracting away the uglier details of memory mapping and synchronization.

GEM allows:

- Mapping buffer memory into user-space/kernel-space
- Pinning memory pages
- Exporting buffer objects to other drivers

GEM itself is data-agnostic. It does not care whether the buffer contains textures, shaders, framebuffers or cursed developer mistakes.

Operations that require awareness of hardware-specific behavior — like allocation, synchronization and execution — are exposed through driver-specific ioctls.

The reason these ioctls exist is because different GPUs have wildly different:

- Alignment requirements
- Caching behavior
- Memory tiling formats
- Execution pipelines

One thing GEM explicitly does _not_ do is buffer allocation. Allocation depends heavily on hardware constraints, so each DRM driver exposes its own allocation ioctls.

Mesa then invokes these ioctls appropriately and directs the DRM driver to place the buffers into graphics memory before running the active shader program.

---

# 7. Compositors and Wayland

After rendering is done, how do we actually display the output?

Enter: **Compositing** — the Pablo Picasso of computing.

Modern desktops are essentially real-time compositing engines. Every application renders independently into its own buffer, and the compositor continuously assembles these buffers into a single coherent desktop image.

Effects like:

- Transparency
- Blur
- Shadows
- Smooth animations
- Window movement

all become possible because the compositor controls the final composition stage rather than applications drawing directly to the screen hardware.

Back in the day, the **X Window System** handled practically everything:

- Drawing
- Compositing
- Input
- Printing
- Window management

Eventually it became an unholy blob of responsibilities, so along came **Wayland**, implementing a significantly cleaner client-server protocol.

Each application acts as a Wayland client.

Wayland itself only defines the protocol. The heavy lifting is performed by compositors like:

- Weston
- Sway
- KWin
- Mutter

A Wayland **surface** represents an application window. It contains a Wayland buffer holding:

- Displayable pixel data
- Color formats
- Dimensions

The application’s output buffer becomes the compositor’s input buffer.

Whenever content changes, the application sends a **surface damage** message to the compositor, indicating which regions must be redrawn.

The compositor maintains:

- Backgrounds
- Application windows
- Desktop UI elements
- Mouse cursors
- Z-ordering information

Ironically enough, even the compositor itself uses Mesa/OpenGL/Vulkan internally to render its own interface.

---

# 8. Shared Memory and dma-buf

Wayland assumes the application and compositor exist on the same host, allowing buffer transfer to happen extremely efficiently.

For shared-memory rendering:

- The application sends a file descriptor
- The compositor maps it into its address space
- Both processes now reference the same memory region

This creates a low-overhead mechanism for exchanging pixel data.

But for hardware-accelerated rendering, shared memory quickly becomes a bottleneck.

Copying framebuffer data between processes is catastrophically expensive at modern resolutions and refresh rates. A single 4K framebuffer can occupy tens of megabytes, and duplicating that repeatedly would annihilate performance.

Wayland solves this through the **dma-buf protocol extension**.

Instead of copying framebuffer contents, the application sends a **dma-buf reference**. The compositor directly consumes the same graphics buffer already residing in GPU-accessible memory.

No unnecessary copies.  
No pointless memory movement.  
No bandwidth wastage.

The graphics buffer remains in graphics memory the entire time.

---

# 9. DRM Mode-Setting Pipeline

So now we finally have a completed image.

How does it _actually_ reach the monitor?

The answer is the **DRM mode-setting subsystem**, which manages the hardware display pipeline responsible for scanout.

Pipeline stages:

- **Framebuffer**
    - The rendered image itself + metadata
    - Acts as the scanout buffer
- **Plane**
    - Determines where and how the framebuffer appears
    - Handles scaling, positioning and z-order
- **CRTC**
    - Controls display timings, refresh rates and blending
- **Encoder**
    - Converts raw pixel streams into standards like HDMI/DP/VGA
- **Connector**
    - Physical display interface
    - Handles EDID parsing, hotplugging and link configuration

You can loosely think of it as:

```
Framebuffer = What to displayPlane       = Where to displayCRTC        = How to displayEncoder     = How to transmitConnector   = Where to transmit
```

Once the framebuffer enters scanout, the display controller continuously reads pixel data line-by-line and streams it to the monitor according to extremely strict timing requirements.

This process is brutally timing-sensitive. Missing deadlines can produce:

- Screen tearing
- Flickering
- Sync failures
- Complete signal loss

Older GPUs may only support a single pipeline, while modern GPUs support multiple CRTCs and planes simultaneously for complex multi-monitor setups.

And that’s the insanity of the Linux graphics stack.

From shader programs to DMA memory, from compositors to display scanout, the entire thing is a gigantic orchestration of memory management, synchronization, abstractions and low-level hardware control. Linux doesn’t just draw pixels. It orchestrates a pixel symphony. A symphony that gets most of us software engineers paid!

I hope you walk away with somethin of value. Do consider following me for more such blogs!

See ya later!

