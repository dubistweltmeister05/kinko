[[Linux]]
[[Blog Topics]]

## 1. Application and Scene Graphs

The story begins with the application, which is responsible for the formatting of the data that is to be visualized. The common data structure used for this is the **scene graph**, which is a collection of nodes of a graph or a tree. The rule followed here is that there can only be one parent of a node, but a node can have many children. The effect of the parent node is applied to _all_ its children — any operation on a parent node is automatically done on its children nodes.

Here, nodes of this graph contain the data that is to be visualized. The application will traverse the tree/graph from top to bottom, and from left to right, performing operations on the attributes and rendering the model accordingly onto the on-screen image.

Should be simple enough, init? Yeah... no. As all things Linux, it is _not_.

---

## 2. OpenGL/Vulkan and Buffer Objects

Now, to simplify this mess, the application uses a set of standard APIs like **OpenGL** and **Vulkan**. (_Do I know what they are and what they do? Fuck no. But will I find out soon enough? Bet your life I will!_) Both provide functionality for the application to interface with the graphics memory, fill it with data, and render what is already stored there.

The graphics data is held in [buffer objects](https://www.khronos.org/opengl/wiki/Buffer_Object), which are within a defined range of the graphics memory and have a handle attached to them. For example, the textures go in the Texture Buffer Object (TBO), shader info in the Shader Storage Buffer Object (SSBO), and so on.

Funnily enough, the output image is stored in another buffer object — so you could say that the entire exercise of graphics rendering is just memory management and playing around with pointer referencing and de-referencing, lol.

---

## 3. Shader Magic (aka Minecraft Mods on Steroids)

The application can give data in any format it likes, as long as the **shader** can process it. _What’s a shader, you say? Is there more to the thing I used to install to make my Minecraft look better, you say?_ Well, a shader is what actually does the conversion of input data to the output image. Think of it like a translator for the OS — it translates the data, regardless of its format, into an output image.

It gets a value from the application that tells it where a pixel is located on a scene, and the shader translates it to a value that tells the kernel where the pixel is supposed to be on the screen.

How does that happen? Simple. Translation matrices and matrix multiplication, of course!

In the most minimal setup, the most common shader operations are **vertex transformations** and **texture lookups**. When it comes to vertex transforms, think of a vertex as a corner of a polygon. To transform the corner from the scenery to the application window, a complex set of algorithms are performed (_could be simple matrix multiplication or something like convoluted inverse transformation_). The shader is what handles the execution of these algorithms.

The output of the shader is typically held in an output buffer. It is usually a colored pixel, with a depth value `z`. Running these shader instructions on the whole scene graph, pixel by pixel, for each node, generates the complete output image for the application.

---

## 4. Enter Mesa: The GPU Whisperer

Jumping over to the Linux side of things now — the rendering interfaces and support for various hardware are provided by the **Mesa 3D Library (Mesa, for short)**.

To the applications, Mesa offers **OpenGL / Vulkan** for graphics and **OpenGL ES** for mobile systems. On the hardware side, Mesa implements drivers for most of today’s graphics hardware. It gives us shared resources and common logic like OpenGL function parsing, shader handling, etc. The **GPU-specific driver** just fills in the hardware-specific blanks, ensuring consistency across different GPUs.

For **stateful interfaces**, such as OpenGL, Mesa's **Gallium3D framework** connects interfaces and drivers. This is called a **state tracker**. Mesa contains state trackers for various versions of OpenGL, OpenGL ES, and OpenCL.

When the application uses the API, it modifies the state tracker. A hardware driver within Mesa then converts the state-tracker information to hardware state and rendering instructions.

For **stateless interfaces** like Vulkan, Mesa instead offers a Vulkan runtime. If there’s a Vulkan driver available, Gallium3D-based OpenGL support may not even be needed for that hardware.

**Zink** is a Mesa driver that maps Gallium3D to Vulkan. With Zink, OpenGL state turns into Gallium3D state, which is then forwarded to hardware via standard Vulkan interfaces.

---

## 5. Memory Management Mayhem

Any memory that is accessible to the graphics hardware is called **graphics memory**. This is the foundation on which the entire stack stands. More memory = happier graphics pipeline.

Graphics memory can be:

- Dedicated (on graphics cards)
    
- Shared (in SoCs)
    
- System memory used as shadow buffers
    

Access to this memory is managed by the **Direct Rendering Manager (DRM)** subsystem. Mesa provides access to DRM under `/dev/dri`, via a device file named after your graphics card. The device file exposes graphics memory as needed by the application(s), in the form of buffer objects.

**Translation Table Manager (TTM)** is the memory manager used by Intel, AMD, NVIDIA cards. It supports discrete, system, and remapped memory.

For simple framebuffer devices, **SHMEM drivers** allocate buffer objects in shared memory. This makes system memory act as a shadow buffer for limited GPU resources. It also enables memory-mapping of USB/I2C buffers.

And then there are the **DMA managers**, which give direct access to the graphics memory via DMA-accessible memory areas.

---

## 6. GEM: The Middleman

Each DRM driver implements a **Graphics Execution Manager (GEM)**. GEM allows:

- Mapping buffer memory to user-space/kernel-space
    
- Pinning memory pages
    
- Exporting buffer objects to other drivers
    

GEM is data-agnostic. It doesn't care what the buffer contains. APIs that need to know the buffer contents (like allocation or synchronization) are handled with **driver-specific ioctls**.

One thing GEM does _not_ do is **buffer allocation**. Since allocation depends on hardware-specific constraints, each DRM driver provides its own ioctl for buffer allocation. Mesa then invokes this ioctl accordingly.

Once the buffers are in place, **Mesa directs the DRM driver** to place them in graphics memory and runs the active shader program.

---

## 7. Compositors and Wayland

After rendering is done, how do we actually display the output? Enter: **Compositing** — the Pablo Picasso of computing.

It takes the output buffer and draws it on the screen. Stacking and tiling are common compositing methods.

Back in the day, **X Window Manager** handled everything (compositing, drawing, printing, etc.), but it became bloated. So along came **Wayland**, implementing a clean client-server protocol. Each application is a client of the Wayland display server.

Wayland only handles **compositing** — not drawing or printing. Compositors (e.g., Weston, Sway, KWin, Mutter) do the heavy lifting.

A Wayland **surface** represents an application window. It has a Wayland buffer with displayable data, color format, and size. The **application’s output buffer** becomes the **compositor’s input buffer**.

When content changes, the app sends a _surface-damage_ message to the compositor, which updates the on-screen content.

The compositor maintains:

- A background (wallpaper or color)
    
- Application windows (as textured rectangles)
    
- Its own UI (e.g., taskbars)
    
- The mouse pointer (highest z-order)
    

Even the compositor itself uses Mesa and OpenGL/Vulkan to render its UI.

---

## 8. Shared Memory and dma-buf

Wayland assumes **application and compositor run on the same host**, so buffer transfer can be fast and simple.

For shared memory buffers:

- App sends a **file descriptor** of the memory
    
- Compositor maps it into its address space
    
- Low-overhead channel created for pixel exchange
    

But for **hardware-accelerated rendering**, shared memory is a bottleneck.

Wayland solves this via a **dma-buf protocol extension**. Instead of copying pixels, the app sends a **dma-buf reference**, which the compositor can use directly. The buffer remains in **graphics memory** the entire time — no data transfer needed.

---

## 9. DRM Mode-Setting Pipeline

So now we have a final image. How does it _actually_ get to the monitor?

**DRM’s mode-setting subsystem** manages the hardware pipeline for displaying rendered images.

### Pipeline Stages:

- **Framebuffer**: The rendered image + metadata (resolution, format). Acts as the scanout buffer.
    
- **Plane**: Determines _where and how_ the framebuffer appears on screen. Includes scaling, position, and z-ordering.
    
- **CRTC (Cathode Ray Tube Controller)**: Handles mode (resolution, refresh rate), timing, blending, etc.
    
- **Encoder**: Converts raw pixels to display standards (HDMI, DP, VGA…).
    
- **Connector**: Physical output port. Handles EDID parsing, hotplug events, link config.
    

Older GPUs may only support one pipeline; modern ones support multiple CRTCs and planes for rich multi-display setups.