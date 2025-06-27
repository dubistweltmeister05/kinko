[[Learn openGL - Notes]]

Right, these are nothing but small programs that define a relation between an input and an output, essentially transforming the i to the o. How are they written? Well, I am glad that you asked - 

### GLSL - (OpenGL Shading Language)

The usual shader structure is as follows - 
```c
#version version_number
in type in_variable_name;
in type in_variable_name;
out type out_variable_name;
uniform type uniform_name;
void main()
{
// process input(s) and do some weird graphics stuff
...
// output processed stuff to output variable
out_variable_name = weird_stuff_we_processed;
}

```

Talking about vertexes, there's the input variable called the vertex attribute. There is a maximum number of vertexes that we are allowed to allocate. OpenGL guarantees that there are at least **16 4-component Vertex** attributes that are available for us to use, but the actual number can be found out by querying `GL_MAX_VERTEX_ATTRIBS`.

##### TYPES in GLSL
Apart from the usual, int, float, bool, etc - there are 2 special ones. They are technically container types, caclled Vectors and Matrices. Let's look at vectors now.

A vector is a 1,2,3,or 4 element container type in GLSL. It can be  - 
-  vecn: the default vector of n floats.
-  bvecn: a vector of n booleans.
-  ivecn: a vector of n integers.
-  uvecn: a vector of n unsigned integers.
-  dvecn: a vector of n double components.

Components of a vector can be accessed via vec.x where x is the first component of the vector.

##### Ins and Outs

These are for defining inputs and outputs to shaders. Each shader can specify inputs and outputs using those keywords and wherever an output variable matches with an input variable of the next shader stage they’re passed along. The vertex and fragment shader differ a bit though.

