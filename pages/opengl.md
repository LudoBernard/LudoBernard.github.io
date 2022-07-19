
# Computer Graphics

## Introduction and Context

For our module GPR5300, we had to create a 3D scene using OpenGL. The goal was to understand the implementation of the main features that make a scene.

To do so, we used OpenGL ES (Embedded System) 3.0, which allows us to run our program on multiple platforms. We also created the engine and the window with the SDL window API.

In this blogpost, I will explain the features that figure in my scene and the steps to implement them.

## Displaying a Triangle

The very first step was to display a triangle on the scene. To draw anything on the screen, ==we need a vertex shader, and a fragment shader:==

```glsl
#version 310 es
layout(location = 0) in vec3 aPos;

void main()
{
    gl_Position = vec4(aPos, 1.0);
}
```

 *<center> Vertex shader </center>*



## Creating lights

## Loading a model

## Post-Processing with Framebuffer

## Adding a Cubemap

## Instancing multiple models

## Normal mapping

## An attempt at adding shadow mapping

## Conclusion