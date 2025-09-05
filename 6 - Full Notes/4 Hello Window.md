[[Learn openGL - Notes]]


These are the initial functions that are needed while starting any openGL project.
- `glfwInit()` -> To initialize GLFW. Terminates all remaining windows and cursors. 
- `void glfwWindowHint(int hint, int value)`-> Sets the specified window hint to the desired value.


### Window object creation - 
``` c++
GLFWwindow* window = glfwCreateWindow(800, 600, "LearnOpenGL", NULL, NULL); if (window == NULL) { std::cout << "Failed to create GLFW window" << std::endl; 
glfwTerminate(); return -1; 
}
glfwMakeContextCurrent(window);

```
  

- `GLFWwindow* glfwCreateWindow(int width, int height, const char* title, GLFWmonitor* monitor, GLFWwindow* share);` -> It requires the window width and height as its first two arguments respectively. The third argument allows us to create a name for the window. It returns a `GLFWWindow` object that we can use for other `GLFW` Operations.
- `void glfwMakeContextCurrent(GLFWwindow* window)` -> Helps switch the context to the specified window. Makes the context of the window as the main context of the current thread that we are working on. 
- `int gladLoadGLLoader(GLADloadproc load)`-> This helps load the address of teh openGL function pointer, which is OS-Based. 

### Viewport
We need to tell the size of the rendering window, to let openGL to know what is the rendering area and how do we want the display data and coordinates to work  W.R.T the display window. 
- `glViewport(x,y,height,width)` -> Lets us define the size of the display port that we are rendering over
We could actually set the viewport dimensions at values smaller than GLFW’s dimensions; then all the OpenGL rendering would be displayed in a smaller window and we could for example display other elements outside the OpenGL viewport.

But, how about some re-sizing as well? How'd that work? The simple answer - a callback function. First, a callback that implements the `glViewport` function, and then a glfw function that tells glsw that this is the callback that it gotta use to re-size anything. 

``` C++
void framebuffer_size_callback(GLFWwindow* window, int width, int height) { 

glViewport(0, 0, width, height); 

}

glfwSetFramebufferSizeCallback (window, framebuffer_size_callback);
```

- `GLFWAPI GLFWframebuffersizefun glfwSetFramebufferSizeCallback(GLFWwindow* window, GLFWframebuffersizefun callback)`-> This function sets the window content scale callback of the specified window, which is called when the content scale of the specified window changes.

We don’t want the application to draw a single image and then immediately quit and close the window. We want the application to keep drawing images and handling user input until the program has been explicitly told to stop. For this reason we have to create a while loop, that we now call the render loop, that keeps on running until we tell GLFW to stop.

``` C++
while (!glfwWindowShouldClose(window))

{

glfwSwapBuffers(window);

glfwPollEvents();

}
```
  
  - `GLFWAPI int glfwWindowShouldClose(GLFWwindow* window)` -> This function sets the value of the close flag of the specified window. This can be used to override the user's attempt to close the window, or to signal that it should be closed.
  - `void glfwSwapBuffers(GLFWwindow* window)`-> This function sets the swap interval for the current OpenGL or OpenGL ES context, i.e. the number of screen updates to wait from the time @ref `glfwSwapBuffers` was called before swapping the buffers and returning.
  - `void glfwPollEvents(void)`->  This function puts the calling thread to sleep until at least one event is available in the event queue.  Once one or more events are available, it behaves exactly like @ref `glfwPollEvents`, i.e. the events in the queue are processed and the function then returns immediately.  Processing events will cause the window and input callbacks associated with those events to be  called.

The `glfwWindowShouldClose` function checks at the start of each loop iteration if GLFW has been instructed to close. If so, the function returns true and the render loop stops running, after which we can close the application. The `glfwPollEvents` function checks if any events are triggered (like keyboard input or mouse movement events), updates the window state, and calls the corresponding functions (which we can register via callback methods).

The `glfwSwapBuffers` will swap the color buffer (a large 2D buffer that contains color values for each pixel in GLFW’s window) that is used to render to during this render iteration and show it as output to the screen.

%% Double buffer When an application draws in a single buffer the resulting image may
display flickering issues. This is because the resulting output image is not drawn in an
instant, but drawn pixel by pixel and usually from left to right and top to bottom. Because
this image is not displayed at an instant to the user while still being rendered to, the result
may contain artifacts. To circumvent these issues, windowing applications apply a double
buffer for rendering. The front buffer contains the final output image that is shown at
the screen, while all the rendering commands draw to the back buffer. As soon as all
the rendering commands are finished we swap the back buffer to the front buffer so the
image can be displayed without still being rendered to, removing all the aforementioned
artifacts.
 %%
