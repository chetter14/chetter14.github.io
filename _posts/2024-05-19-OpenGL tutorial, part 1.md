---
layout: post
title: OpenGL tutorial. Part 1.
---

This is the start of learning OpenGL. It's not a complete tutorial but just *my notes* on interesting and complicated concepts of OpenGL (to better understand the subject by explaining the details of it). Everything here is based solely on the [Learn OpenGL tutorial](https://learnopengl.com/Getting-started/OpenGL), so for the whole tutorial you can go there and come back here for another perspective (that can be easier to grasp) on OpenGL topics.

**OpenGL** is not totally an API/library (although it's called like that). It's a **standard** that specifies the "what" of functions used (what the function should do, what the result of the function is, etc.). The implementation part - "how" - is developed by other people, mainly graphics card manufacturers.

Before OpenGL 3.2 there was an *immediate* mode. This mode means a simple-to-use OpenGL functionality that you can easily learn. Implementation details were hidden with worse performance and flexibility in general. From the 3.2 version, there is a *core-profile* mode - the opposite of what was there before. It's harder to learn although it provides you the opportunities for more flexibility, faster performance, and different features.

OpenGL is just a huge state machine. You set a state, operate in this state, change to another state, operate there, and so on.

OpenGL is made so that it's *independent on any platform and operating system*. To not worry about functionality required to draw images, set contexts, and create windows (all the user input) we use **GLFW** library (also there is GLUT, SDL, and other libraries like that).

And the last thing. Since the OpenGL implementation is specific to GPU and OS and we want to develop applications that are *not system-specific*, we need to retrieve these OpenGL drivers at run-time. Here **GLAD** comes into play so that you can easily call OpenGL functions without worrying that it won't work for some of the systems.

To set the OpenGL version you want to use with *GLFW*, you call functions like:
```
glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3);
glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 3);
glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_RPOFILE);
```
This way you tell that you want to use the OpenGL of version 3.3 and in core-profile mode.

In order to render with OpenGL you need to specify an area size your rendering will be applied to (in a current glfw window). You do it with:
```
glViewport(0, 0, 800, 600);
```
The first 2 parameters are the XY coordinates of the left bottom corner. The 3rd and 4th - width and height of the rendering area.

Everytime the window resizes the viewport should be resized as well. To achieve that we will use a *callback that gets called on every window resize*:
```
void framebuffer_size_callback(GLFWwindow* window, int width, int height)	// a function prototype used in GLFW to set framebuffer size callback
{
    glViewport(0, 0, width, height);
}  

glfwSetFramebufferSizeCallback(myWindow, framebuffer_size_callback);
```

In render loop we need 2 vital functions (to be called on every frame): *check if any events happened and draw an image*. So here they are:
```
while (shouldNotClose(window))
{
	// ...
	glfwSwapBuffers(window);		// swaps the front and back buffers (containing pixels) to display an image from back buffer; there are two buffers in order to avoid flickering
	glfwPollEvents();			// check if window is resized, mouse is moved, etc. If it is, then the appropriate callback function is called
}
```

At each frame, we want to start from blank, and to do it we use functions like `glClearColor()` and `glClear()`. `glClear()` is called to clear the frame color buffer (using GL_COLOR_BUFFER_BIT). `glClearColor()` sets a color that is being filled in color buffer at frame clearing.
```
while (shouldNotClose(window))
{
	// process input
	
	glClearColor(0.1f, 0.1f, 0.1f, 1.0f);		// sets a color that will be in every pixel after clearing
	glClear(GL_COLOR_BUFFER_BIT); 			// clear the frame
	
	// check on events and swap buffers
}
```
