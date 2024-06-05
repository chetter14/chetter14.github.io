---
layout: post
title: OpenGL tutorial. Part 2.
---

**Hello Triangle part.**

A *graphics pipeline* is a process of transforming 3D coordinates into 2D pixels on your screen. This process is not that complicated. However, two important stages *must be defined by the programmer* - **vertex and fragment shaders** because they are not defined by the GPU by default.

Briefly about them, vertex shader - does a transformation of vertices coordinates to the *Normalized Device Coordinates* (NDC), where x, y, and z values vary from -1.0 to 1.0. Fragment shader - calculates the color of every pixel on the screen. The output of the fragment shader is an array of 4 values: *RGBA* - Red, Green, Blue, and Alpha (opaqueness of the pixel). The result color of a pixel is defined by the mix of red, green, and blue colors.

To send vertices data to vertex shader (and the whole pipeline) we do the following: create an array buffer on the GPU, copy vertices' data into it, and later on the GPU will be able to rapidly access vertices' data for shaders' calculations. Code sample:
```
unsigned int VBO;
glGenBuffers(1, &VBO);
glBindBuffer(GL_ARRAY_BUFFER, VBO);											// we assign VBO (it's like an ID) to the internal GPU array buffer
glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);	// that stores exactly those vertices data we passed to it
```
There can be many VBOs in your C++ program because vertices data can differ depending on what you want to draw.

Vertex and fragment shaders are *like programs*. The language used to write such programs is *GLSL* (OpenGL Shading Language). For now, consider this line of code `layout (location = 0) in vec3 aPos;` as - the input of size of 3 floats is located at 0 position (or, from my perspective, id; I'll explain it later).

Your *custom-defined vertex and fragment shaders* should be attached to one whole shader program to have the graphics pipeline set up entirely.
Example of it:
```
unsigned int shaderProgram;
shaderProgram = glCreateProgram();
glAttachShader(shaderProgram, vertexShader);
glAttachShader(shaderProgram, fragmentShader);
glLinkProgram(shaderProgram);

// clean them up, no need in them separately
glDeleteShader(vertexShader);	
glDeleteShader(fragmentShader); 

// now on every rendering call your vertex and fragment shaders will be applied (in the order that is specifed by the graphics pipeline)
glUseProgram(shaderProgram);
```

*Vertex attributes are just properties that any vertex can have*: coordinates, color, etc. You set vertex attribute this way:
```
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), (void*)0);
glEnableVertexAttribArray(0);  
```
Remember about `(location = 0)`? The first argument in `glVertexAttribPointer()` is exactly for this. In C++ code you set the ID (I call it that way for more clarity) of an attribute and in your shader program (vertex/fragment shaders) you access that attribute with `(location = 0)`.

With the `glVertexAttribPointer()` function you define *the attribute that is related to the current VBO*. At first, you bind some VBO to the GL_ARRAY_BUFFER, then you create a vertex attribute and enable it in the current VBO.

To not repeat the code below on every iteration there is an object called Vertex Array Object (VAO).
```
glBindBuffer(GL_ARRAY_BUFFER, VBO);
glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), (void*)0);
glEnableVertexAttribArray(0);

glUseProgram(shaderProgram);

someOpenGLFunctionThatDrawsOurTriangle();  
```
**VAO** *stores the VBO and all the vertex attribute pointers related to this VBO in one object* enabling to switch easier between different configurations of VBOs or vertex attributes.

*EBO (Element Buffer Object)* is another buffer object (like VBO) that is put in VAO. You work with EBO this way: you provide an array of unique vertices to VBO (i.e., rectangle) and specify the indices of vertices that you want to draw your shape (triangle) with. A clear example:
```
float vertices[] = {
     0.5f,  0.5f, 0.0f,  // top right
     0.5f, -0.5f, 0.0f,  // bottom right
    -0.5f, -0.5f, 0.0f,  // bottom left
    -0.5f,  0.5f, 0.0f   // top left 
};
unsigned int indices[] = {  // note that we start from 0!
    0, 1, 3,   // first triangle
    1, 2, 3    // second triangle
}; 
```

**Shaders part.**

*Uniforms* are like global variables that you can access in the whole shader program and set their value from C++ code. I've not experienced a difficulty with other concepts in this chapter.

**Textures part.**

*Texture wrapping determines the "behavior" of the texture outside of its default scope*, which is (0,0)-(1,1). If, for some reason, your texture coordinates get outside of this default scope you set the way the texture is drawn there (repeated, mirror_repeated, etc.).

*Texture filtering.* Your texture coordinates can be of any value between 0.0 and 1.0 and it does not necessarily mean that whatever coordinates are going to map to the exact placement of texture pixels (texels). And *to handle cases when texture coordinates are on the cross of a few texels, you define the way what pixel (RGBA values, etc.) it is going to be finally* (on these coordinates of this texture).

Important to remember: *sampling* - retrieving the texture color by texture coordinates.

Sampler1D, sampler2D, and sampler3D types are just types of texture objects. That's essentially what they are.

Texture units (GL_TEXTURE0, GL_TEXTURE1, etc.) are default texture objects and you have many of them to be able to draw a few textures. So, there are pairs: texture unit - your texture. Usage code sample:
```
myShader.use();
myShader.setInt("texture1", 0);		// set sampler2D variable in a shader program to GL_TEXTURE0 
myShader.setInt("texture2", 1); 	// set another sampler2D variable in a shader program to GL_TEXTURE1 

while (...)
{
	// ...
	glActiveTexture(GL_TEXTURE0);						// activate the 0 texture unit
	glBindTexture(GL_TEXTURE_2D, textures[0]);			// bind textures[0] to GL_TEXTURE0 (or textures[0] to "sampler2D texture1" in the shader program
	glActiveTexture(GL_TEXTURE1);						// activate the 1 texture unit
	glBindTexture(GL_TEXTURE_2D, textures[1]);			// bind textures[1] to GL_TEXTURE1 (or textures[1] to "sampler2D texture2" in the shader program
	// ...
}
```
This is a rough description in a sense that I intentionally avoided diving deeper in order to keep things simple to grasp.
