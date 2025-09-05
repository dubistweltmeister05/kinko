[[Learn openGL - Notes]]

Graphics pipeline - the thing that manages the conversion of 3D spaces and coordinates into 2D pixels. Broadly broken up into 2 parts - the first one converts 3D to 2D and the second one converts 2D into pixels. 

The entire pipeline is actually divided into small, separate set(s) of tasks and thus,due to it's parallel nature, modern graphics cards have multiple small processing cores that are able to process data at a rapid rate. These small pieces of code, that are parallelly run are called ***SHADERS***. These things are written in OpenGL Shading Language (GLSL). 

![[Pasted image 20250618124234.png]]

The input to the pipeline is a set of 3D coordinates that form a triangle. This is called as the Vertex Data. Vertex data is a blanket term that refers to any set of data, per 3D coordinate, and is represented using vertex attributes. 

For OpenGL to known what kind of data are we entering, it needs some hints from us, about the type of render that is to be formed via this data. These hints are called primitives and are given to OpenGL when we are calling any draw commands. 

The first part of the pipeline basically re-maps the 3D coordinates that we have given as input, and the vertex shader allows for some pre-processing to be done by us.

The primitive assembly stage takes as input all the vertices (or vertex if GL_POINTS is chosen) from the vertex shader that form a primitive and assembles all the point(s) in the primitive shape given; in this case a triangle.

The output of this stage is passed to the geometry shader, which takes the collection of vertices that form the shape of a primitive, and outputs a different shape via another set of points. 

The geometry shader then passes it's output to the `rasterization` stage, which maps the primitive to the corresponding pixels on the screen. This generates fragments of the final screen, but aren't ready for the display output yet. A clipper discards the fragments that are rendered out of the display window, and the remaining output is then passed on to the `fragmentation` shader. 

%% A fragment in OpenGL is all the data required for OpenGL to render a single pixel. %%

The `fragment` shader calculates the final color of the pixel, and this is the spot where all advanced openGL processes occur. It usually contains data like lights, shadows, color
of the light and so on. 

Finally, there is the `alpha test` and the `blending stage` . This stage checks the depth of a pixel, and uses that data to determine is a pixel is going to be ahead or behind the other, and if it should be discarded or not. Here, the opacity, or the alpha values are checked as well. 

In modern OpenGL we are required to define at least a vertex and fragment shader of our own (there are no default vertex/fragment shaders on the GPU). For this reason it is often quite difficult to start learning modern OpenGL since a great deal of knowledge is required before being able to render your first triangle. 

### Vertex Inputs
Not a surprise that the input to a 3D library like OpenGL is also in the form of a 3D matrix!

OpenGL only transforms the coordinates that are within the range -1.0 to 1.0, in ALL 3 AXES. These are called Normalized Device Coordinates, and these are the ones that are shown on the screen. 

Once your input coordinates have been processed, they should lie within the Normalized Device Coordinates (NDC), and any coordinates that fall outside this range will be discarded/clipped and won’t be visible on your screen.

Unlike usual screen coordinates the positive y-axis points in the up-direction and the (0,0) coordinates are at the center of the graph, instead of top-left. Eventually you want all the (transformed) coordinates to end up in this coordinate space, otherwise they won’t be visible.

Your NDC coordinates will then be transformed to screen-space coordinates via the viewport transform using the data you provided with `glViewport`. The resulting screen-space coordinates are then transformed to fragments as inputs to your fragment shader.

To send this data to the GPU pipeline, we start with creating memory for it on the GPU. Here, we store the vertex data, configure OpenGL on it's interpretation and specify how to send data to the graphics card memory. The vertex shader then processes as much vertices as we tell it to from its memory.

To manage this, we use the so-called Vertex Buffer Object, which can store a large number of vertices on the GPU memory. The advantage of using those buffer objects is that we can send large batches of data all at once to the graphics card, and keep it there if there’s enough memory left, without having to send data one vertex at a time. Once the data is in the graphics card’s memory the vertex shader has almost instant access to the vertices making it extremely fast. 

- `void glGenBuffers(GLsizei n, GLuint *buffers)` -> used to generate buffer object names (IDs) that can later be bound to a buffer target.

The buffer type of a vertex buffer is `GL_ARRAY_BUFFER`. We can bind several buffers of different types at once. 
- `void glBindBuffer(GLenum target, GLuint buffer)` -> used to bind a buffer object (previously generated with `glGenBuffers`) to a specific buffer target. Once bound, any buffer-related calls (like `glBufferData`) will affect the currently bound buffer for that target.
Then we can make a call to the `glBufferData` function that copies the previously defined vertex data into the buffer’s memory:
- `void glBufferData(GLenum target, GLsizeiptr size, const void *data, GLenum usage)` -> used to allocate memory on the GPU for the data that we had defined. 

### Vertex Shader

This is one of the few shaders that is user programmable in OpenGL. In C, you gotta declare the entire thing as a string. 

``` C
#version 330 core
layout (location = 0) in vec3 aPos;
void main()
{
gl_Position = vec4(aPos.x, aPos.y, aPos.z, 1.0);
}
```
##### VECTOR
It is literally the mathematics concept, extensively used in graphics programming, since it represents the position/direction of anything, and is quite useful from the mathematical perspective of things. 
A vector in GLSL has a maximum size of 4 and each of its values can be retrieved via `vec.x, vec.y, vec.z and vec.w` respectively where each of them represents a coordinate in space. The `vec.w` is used for something called perspective division. 

The input is a vector of 3 fields, hence the `vec3 `declaration. As the output is supposed to be a `vec4`, we pass the input values in the `vec4` constructor and then set the last value, `vec.w`  = 1.0f. 

Note how we did fuck-all with the input data, and simply passed it to the output. 

##### Compiling the shaders. 
We put the thing in a string, and as always, we put it in a shader object, and reference it with an ID. 
```C
unsigned int vertexShader;
vertexShader = glCreateShader(GL_VERTEX_SHADER);
```

- `GLuint glCreateShader(GLenum shaderType)` -> **creates an empty shader object** and returns its **unique ID** (handle).  This ID is used to attach source code and compile the shader.

Add checks for the shader compilation - 
``` C++
int success;
char infoLog[512];
glGetShaderiv(vertexShader, GL_COMPILE_STATUS, &success);
```

- `void glGetShaderiv(GLuint shader, GLenum pname, GLint *params)` -> queries information about a shader object — for example, whether it compiled successfully or how long its info log is. It stores the result of the query in the variable pointed to by `params`.

### Fragment Shader
The fragment shader is all about calculating the color output of your pixels.

Colors in computer graphics are represented as an array of 4 values: the red, green, blue
and alpha (opacity) component, commonly abbreviated to RGBA. When defining a color
in OpenGL or GLSL we set the strength of each component to a value between 0.0 and
1.0. If, for example, we would set red to 1.0 and green to 1.0 we would get a mixture
of both colors and get the color yellow. Given those 3 color components we can generate
over 16 million different colors!

##### Shader Program
A shader program object is the final linked version of multiple shaders combined. To use the recently compiled shaders we have to link them to a shader program object and then activate this shader program when rendering objects. The activated shader program’s shaders will be used when we issue render calls.

Creating a program is easy enough - 
- `GLuint glCreateProgram(void)` -> creates a new empty shader program object and returns its unique ID (handle).  You use this ID to **attach compiled shaders** (vertex, fragment, etc.) and **link them together** into an executable GPU program.

Attaching shaders to the program - 
- `void glAttachShader(GLuint program, GLuint shader)` -> attaches a compiled shader (vertex, fragment, etc.) to a shader program object. You typically attach one vertex and one fragment shader (optionally others) before linking the program with `glLinkProgram`.
Linking the program - 
- `void glLinkProgram(GLuint program)` -> Links all the attached shaders (e.g., vertex and fragment shaders) into a complete, executable shader program.  After linking, the program can be used with `glUseProgram`.
Using the bastard - 
 - `void glUseProgram(GLuint program)` -> ets the specified linked shader program as the **current GPU program** used for rendering. All subsequent drawing operations will use this program’s shaders until another program is bound or `0` is passed to disable shaders.
### Linking Vertex Attributes
While we can specify any and every input that we want, this also means that we have to manually specify what part of our input data goes to which vertex attribute in the vertex shader.

This is done with the help of the following function - 
- `void glVertexAttribPointer(GLuint index, GLint size, GLenum type, GLboolean normalized, GLsizei stride, const void *pointer)` -> 
  
  Here is a brief about the parameters being passed - 
  - `GLuint index` -> This specifies the vertex that we want to configure. 
  - `GLint size` -> This specifies the size of the attribute of the vertex. 
  - `GLenum type` -> This specifies the data type used to represent the attribute. 
  - `GLboolean normalized` -> Specifies if the data is to be normalized or not. Set to true if you are imputing data that is INT or BYTE.
  - `GLsizei stride` -> This specifies the width of a vertex attribute. 
  - `const void *pointer` -> This is the offset of where the position data begins in the buffer.

##### Vertex Array Objects (VAO)
This can be bound to a buffer like a vertex buffer object, and all subsequent vertex attribute calls will be stored here. A VAO stores the following - 
 - Calls to `glEnableVertexAttribArray` or `glDisableVertexAttrib` Array.
 - Vertex attribute configurations via `glVertexAttribPointer`.
 - Vertex buffer objects associated with vertex attributes by calls to `glVertexAttribPointer`.

