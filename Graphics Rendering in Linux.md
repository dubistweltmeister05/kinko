[[Linux]]
[[Blog Topics]]

The story begins with the application, which is responsible for the formatting of the data that is to be Visualized. The common data structure used for this is the Scene graph, which is  a collection of nodes of a graph or a tree. The rule followed there is that there can only be one parent of a node, but a node can have many children. The effect of the parent node is applied to ALL it's children, effectively - any operation on a parent node is automatically done on it's children nodes. 

Nodes of this graph contain the data that is to be visualized. The application will traverse the tree/graph from top to bottom, and from left to right, performing operations on the attributes and renders the model accordingly, onto the on-screen image. 

Now, to simplify this mess, the application uses a set of standard APIs, like OpenGL and Vulkan. (*Do I know what they are and what do they do? Fuck no. But will I find out soon enough? Bet your life I will!*) Both provide functionality for the application to interface with the graphics memory, fill it with data and to render what is already stored there. 

The graphics data is held in [[buffer objectshttps://www.khronos.org/opengl/wiki/Buffer_Object | buffer objects]], which are within a defined range of the graphics memory, and have a handle attached to them. For example, the textures go in the Texture-Buffer-Object (TBO), Shader Info in the Shader-Storage-Buffer-Object, and so on and so forth. Funnily enough, the output image is stored in another buffer object, so you could say that all this exercise that the process of graphics rendering does nothing but some memory management and playing around with some pointer referencing and de-referencing, lol!

Now, the application can give data in any format that it likes, as long as the shader can process it. *What's a shader,  you say? Is there more to the think that i used to install to make my minecraft look better, you say?* Well, a shader is what actually does the conversion to the imput data to the output image. Think of it like a translator for the OS - it translates the data, regardless of it's format into an output image. It gets a value from the application that tells it where is a pixel located on a scene according to the scene, and the shader translates it to a value, that tells the kernel where the pixel is supposed to be on the screen.

How does that happen? Simple. Translation matrices, and matrix multiplication, of course!

In the most minimal setup required, the most common shader operations are vertex transformations and texture lookups. When it comes to vertex transforms, think of a vertex as a corner of a polygon. To transform the corner from the scenery to the application window, a complex set of algorithms are performed (*could be simple matrix multiplication, or something like convoluted inverse transformation*). The shader, is what handles the exec of these algorithms.

The output of the shader is typically held in an output buffer. It is usually a coilred pixel, with a depth value *z* . Running these shader instructions on the whole scene graph, pixel by pixel, for each node, generates the complete output image for the application.


Jumping over to the linux side of things now, the rendering interfaces and support for various hardware is provided by MESA 3D Library (MESA, for short). To the applications, it has OpenGL /Vulkan for graphics and OpenGL_ES for mobile systems. On the hardware side, Mesa implements drivers for most of today's graphics hardware.

Basically, it gives us shared resource and common logic like ike OpenGL function parsing, shader handling, etc. The GPU-specific driver just fills in the hardware-specific blanks, helping us ensure consistency across different GPUs. 

For stateful interfaces, such as OpenGL, Mesa's Gallium3D framework connects interfaces and drivers with each other. This is called a state tracker. Mesa contains state trackers for various versions of OpenGL, OpenGL ES, and OpenCL. When the application uses the API, it modifies the state tracker for the given interface. A hardware driver within Mesa further converts the state-tracker information to hardware state and rendering instructions.

For stateless interfaces link Vulkan, Mesa instead offers the Vulkan runtime to help with their implementation. If there is a Vulkan driver available, though, there might not be a need for Gallium3D-based OpenGL support at all for that hardware. Zink is a Mesa driver that maps Gallium3D to Vulkan. With Zink, OpenGL state turns into Gallium3D state, which is then forwarded to hardware via standard Vulkan interfaces. In principle, this works with any hardware's Vulkan driver. One can imagine that future drivers within Mesa only implement Vulkan and rely on Zink for OpenGL compatibility.

Now, for everyone's favorite part of computers - MEMORY MANAGEMENT!

Any memory that is accessible to the graphics hardware is called as the graphics memory. This is the foundation on which the entire stack stands, and the more, the merrier it is, lads. In terms of hardware, the graphics memory comes in all shapes and sizes. It can be a fully dedicated memory bank, as found of graphics adapters or it can be the simple, on-system memory that we see is so many SoCs, there are shared-memory banks for graphics applications that can be found on some chips as well.

The access to this, as all things in the Kernel, is managed by a thing called the DRM (Direct Rendering Manager) subsystem. Mesa gives access to the DRM under the `/dev/dri` , via a device file of the same name as your graphics card. Whenever needed, the device file exposes the graphics memory as and when required by the application(s), in the form of buffer objects.

The DRM manager gives us a memory manager in itself. Most of the cards from Intel, AMD, and NVIDIA use something called as the Translation Table Manager. It supports Discrete memory Graphics Address Remapping Table memory, and the basic system memory. 

For simple framebuffer devices, it uses the SHMEM drivers, that allocate the buffer objects in the shared memory. Here, regular system memory acts as a shadow buffer for the device's limited resources. The graphics driver maintains the device's graphics memory internally, but exposes buffer objects in system memory to the outside. This also makes it possible to memory-map buffer objects of devices on the USB or I2C bus, even though these buses do not support page mappings of device memory; the shadow buffer can be mapped instead.

Lastly, there's the most friendly, totally not unsafe to use, DMA managers, which, well, give you direct access to the graphics memory, via managment of the DMA-accessible memory part of the system's total memory. 

Each Driver implements what is called the Graphics Execution Manager (GEM). he GEM interface allows mapping a buffer object's memory pages to user-space or kernel address space, allows pinning the pages at a certain location, or exporting them to other drivers. GEM exposes a set of standard memory-related operations to userspace and a set of helper functions to drivers, and let drivers implement hardware-specific operations with their own private API. 

Also, GEM is data-agnostic. It manages abstract buffer objects without knowing what individual buffers contain. APIs that require knowledge of buffer contents or purpose, such as buffer allocation or synchronization primitives, are thus outside of the scope of GEM and must be implemented using driver-specific ioctls.

The one common operation that GEM does not provide is buffer allocation. Each buffer object has a specific use case, which affects and is affected by the object's allocation parameters, memory location, or hardware constraints. Hence, each DRM driver offers a dedicated ioctl() operation for buffer-object allocation that captures these hardware-specific settings. The DRM driver's counterpart in Mesa invokes said ioctl() operation accordingly.

