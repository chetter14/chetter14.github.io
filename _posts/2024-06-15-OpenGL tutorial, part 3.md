---
layout: post
title: OpenGL tutorial. Part 3.
---

**Transformations part.**

It's fascinating how useful vectors, matrices, and related mathematical concepts are. The way you can combine and use them is fascinating as well.

At first, to apply this math to OpenGL I've added *GLM* (Open**GL** **M**athematics) library - a header-only library. This library contains things like different types of matrices (3x3, 4x4, etc.), vectors, and all the required operations on matrices (translation, rotation, scaling, etc.). 

Then I created a *"transformation"* matrix. This matrix *"contains" all the required operations (and values for them) that I want to apply to the vertices' coordinates*. An example:
```
glm::mat4 trans1 = glm::mat4(1.0f);							// initialize an identity matrix
trans1 = glm::translate(trans1, glm::vec3(0.5f, -0.5f, 0.0f));				// tell the matrix that I want to move coordinates to 0.5 right and 0.5 down
trans1 = glm::rotate(trans1, (float)glfwGetTime(), glm::vec3(0.0f, 0.0f, 1.0f));	// tell the matrix to rotate around Z-axis on *glfwGetTime()* radians
```
After that I've changed a vertex shader. Now I have to take the transformation matrix into account:
```
layout(location = 0) in vec3 aPos;

...

uniform mat4 transform;

void main()
{
	gl_Position = transform * vec4(aPos, 1.0);
	...
}
```
And, finally, in C++ code I've set the transformation matrix and have became able to draw elements:
```
myShader.setTransformMatrix("transform", trans1);
glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_INT, 0);	// to draw a rectangle
```
In my `myShader.setTransformMatrix()` class function the `glUniformMatrix4fv()` function is called.

Eventually, *using such transformation matrices I can do everything with vertices` coordinates - scale the up/down, translate (move) to whatever positions, rotate around different axises, etc.*

**Coordinate Systems part.**

Here I've added work with the camera and perspective that are required for 3D space.

At first, let's put objects into positions in a 3D world space (by filling matrices the way described earlier):
```
for (int i = 0; i < 10; ++i)
{
	...
	glm::mat4 model = glm::mat4(1.0f);
	model = glm::translate(model, cubePositions[i]);
	myShader.setTransformMatrix("model", model);
	...
}
```

By default, the camera origin is at (0,0,0). I want to move the camera backward to see an object at this (0,0,0) position. I've done it by *moving all the objects further from the camera (instead of moving the camera itself).* Example:
```
glm::mat4 view = glm::mat4(1.0f);
view = glm::translate(view, glm::vec3(0.0f, 0.0f, -3.0f));		// move object by 3.0 from the camera (in right-handed coordinate systems)
myShader.setTransformMatrix("view", view);
```

Then I've added *a **perspective projection matrix** (not the orthographic one) to see objects in distance smaller than those that are closer*:
```
glm::mat4 projection;	
projection = glm::perspective(glm::radians(45.0f), 800.0f / 600.0f, 0.1f, 100.0f);	// set FOV to 45.0, aspect ratio to 4/3
myShader.setTransformMatrix("projection", projection);
```

And in the last step I've "applied" all these matrices to local vertices' coordinates in vertex shader:
```
...
layout(location = 0) in vec3 aPos;
...
uniform mat4 model;
uniform mat4 view;
uniform mat4 projection;

void main()
{
	gl_Position = projection * view * model * vec4(aPos, 1.0);
	...
}
```
The order of multiplication is *vital* in matrices multiplication. It *reads from right to left*.

**Camera part.**

I've developed a Camera class that detects mouse movements to look around, keyword presses' to walk around, and functionality to zoom in and out with a scroll wheel.

At first, I wrote a callback `mouseCallback()` that is called on every cursor movement. In GLFW it sets with the `glfwSetCursorPosCallback()` function.
```
void mouseCallback(GLFWwindow*, double xpos, double ypos)
{
	if (firstMouseInput)		// the first time cursor enters the window
	{
		lastX = xpos;
		lastY = ypos;
		firstMouseInput = false;
	}

	float xoffset = xpos - lastX;
	float yoffset = lastY - ypos;			// Y-axis grows down so (lastY - ypos) and not vice versa
	lastX = xpos;
	lastY = ypos;

	xoffset *= mouseSensitivity;			// multiply by coef < 1.0 
	yoffset *= mouseSensitivity;			// so that mouse movement wouldn't be so rapid

	yaw += xoffset;					// yaw - right/left rotation
	pitch += yoffset;				// pitch - up/down rotation

	if (pitch > 89.0f)				// can't look further up than 89 degrees
		pitch = 89.0f;
	else if (pitch < -89.0f)			// and further down than -89 degrees
		pitch = -89.0f;

	glm::vec3 direction;					// calculate the direction camera will look at after mouse movements
	direction.x = cos(glm::radians(yaw));
	direction.y = sin(glm::radians(pitch));
	direction.z = sin(glm::radians(yaw)) * cos(glm::radians(pitch));
	
	// calculate the view space Z, X and Y axises respectively
	front = glm::normalize(direction);
	right = glm::normalize(glm::cross(front, worldUp));
	up = glm::normalize(glm::cross(right, front));
}
```
The *`front` and `up` variables (namely, axises) are used in the `glm::lookAt(position, target, up)` function that returns a 4x4 matrix.* This matrix will be assigned to the `glm::mat4 view` variable mentioned earlier.

To respond to presses of WASD keys I've developed a `processInput()` function - a member of the Camera class.
```
void processInput(GLFWwindow* window)
{
	float currentFrame = glfwGetTime();
	deltaTime = currentFrame - lastFrame;
	lastFrame = currentFrame;
	
	// speed is dependent on deltaTime because of different time of execution on absolutely different machines
	// so on different machines for the same amount of time the camera position will be moved exactly the same
	float speed = movementSpeed * deltaTime;

	// front - where the camera looks at
	// right - points to the right side of camera
	if (glfwGetKey(window, GLFW_KEY_W) == GLFW_PRESS)
		pos += speed * front;
	if (glfwGetKey(window, GLFW_KEY_S) == GLFW_PRESS)
		pos -= speed * front;
	if (glfwGetKey(window, GLFW_KEY_A) == GLFW_PRESS)
		pos -= right * speed;
	if (glfwGetKey(window, GLFW_KEY_D) == GLFW_PRESS)
		pos += right * speed;
}
```

The case with a scroll wheel is easier. I've defined a `scrollCallback()` function that is a callback (like with the mouse movements one). 
```
void scrollCallback(GLFWwindow*, double xoffset, double yoffset)
{
	fov -= (float)yoffset;
	if (fov < 1.0f)
		fov = 1.0f;			// can't look narrower
	else if (fov > 45.0f)
		fov = 45.0f;			// can't look wider
}
```

Also, I've experienced difficulty with understanding the `glm::lookAt(position, target, up)` parameters, namely the `target` one. After searching the Internet, I realized that *`target` can be calculated as `position + D` where `D` is the direction you want your camera to look*. So, in my case, the final version of calling this function and passing arguments to it looks like this: 
```
{	
	...
	glm::mat4 view = glm::lookAt(pos, pos + front, up);
	...
}
```
