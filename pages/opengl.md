
# Computer Graphics

## Introduction and Context

For our module GPR5300, we had to create a 3D scene using OpenGL. The goal was to understand the functioning of the main features that make a scene.

To do so, we used OpenGL ES (Embedded System) 3.0, which allows us to run our program on multiple platforms. We also created the engine and the window with the SDL window API.

To create this scene, I followed the tutorials provided by [LearnOpenGL](https://learnopengl.com). You can find all the features that I will talk about on the website as well as in-depth explanations for each of them.

In this blogpost, I will explain the features that figure in my scene and the steps to implement them.

## Displaying a Triangle

The very first step was to display a triangle on the scene. To draw anything on the screen, **we need a vertex shader and a fragment shader:**

```glsl
#version 310 es
precision highp float;

out vec3 fragColor;

// Give our vertices positions
vec2 positions[3] = vec2[](
        [...]
);

// Give the colors at each vertex
vec3 colors[3] = vec3[](
        [...]
);

void main() {
    // Set the positions and send the color to the Fragment shader
    gl_Position = vec4(positions[gl_VertexID], 0.0, 1.0);
    fragColor = colors[gl_VertexID];
}
```

 *<center> Vertex shader </center>*


 ```glsl
 #version 310 es
precision highp float;

in vec3 fragColor;

layout(location = 0) out vec4 outColor;

void main() {
    // Set the color for each pixel
    outColor = vec4(fragColor, 1.0);
}
```
 *<center> Fragment shader </center>*
 

 Now that our shaders are written, we need to compile them and bind them to our program, and draw our triangle between the clear and the swap, which gives us this result:

 <p align="center">
 <img width="400" height="300" src="../img/opengl/triangle.png"><br>
 </p>

 
 We can now create colored shapes in our scene, in this example our triangle was in 2D, but we can create cubes and other shapes in 3D, and add textures to them.

## Creating lights

To add some realism to our scene we can create lights that will affect the way our objects appear and make them look brighter or darker.

In my scene, I used the **Blinn-Phong** shading model, which uses light maps to simulate lighting (diffuse & specular), and uses a halfway vector instead of a reflection vector between the view direction and the light direction to compare it to the normal. The closer the halfway is to the normal, the higher the specular will contribute.

<p align="center">
 <img width="800" height="216" src="../img/opengl/phong.png">
 <em> Phong Model </em><br><br>
 <img width="400" height="300" src="../img/opengl/blinnphong.png"><br>
 <em> Blinn-Phong Model </em>
 </p>

There are many types of lights we can use in our scene, in my case, I used a directional light for my final scene, which simulates a light source with no position and only a direction.

To implement a directional light, or any type of light, we first need to create a structure for our light and for our material in the fragment shader:

```glsl
struct DirLight {
    vec3 direction;
    vec3 ambientStrength;
    vec3 diffuseStrength;
    vec3 specularStrength;
    };

struct Material{
    sampler2D diffuseMap;
    sampler2D specularMap;
    float shininess;
};
```

After adding those two elements, we can then calculate the lighting inside our main function:

```glsl
void main()
{ 
    // Normalize the directions to calculate the halfway vector
    vec3 lightDir   = normalize(lightPos - FragPos);
    vec3 viewDir    = normalize(viewPos - FragPos);
    vec3 halfwayDir = normalize(lightDir + viewDir);

    // Diffuse calculation
    float diff = max(dot(normal, lightDir), 0.0);

    // Specular calculation with Blinn-Phong
    float spec = pow(max(dot(normal, halfwayDir), 0.0), material.shininess);

    // Combine results
    vec3 ambient = light.ambientStrength * vec3(texture(material.diffuseMap, TexCoords));
    vec3 diffuse = light.diffuseStrength * diff * vec3(texture(material.diffuseMap, TexCoords));
    vec3 specular = light.specularStrength * spec * vec3(texture(material.specularMap, TexCoords));

    // Return final result
    result = ambient + diffuse + specular;
    FragColor = vec4(result, 1.0);
}
```

After implementing the lights correctly, our scene should look like this: 

<p align="center">
<img width="494" height="364" src="../img/opengl/directional.png"><br>
</p>


We now have lights in our scene! 

## Loading a model

Playing around with cubes and triangles is fun, **but having 3D models is even more fun**, which is why I implemented model loading into my scene.

To do so, I used the [Assimp](https://assimp-docs.readthedocs.io/en/v5.1.0/) library, which allows us to easily load dozens of model file formats such as **.obj** which is the format that I am using in my scene. Assimp uses *aiNodes* to load the model from the file, which prevents us from doing it all in code.

But, that does not mean we don't have to code anything, in fact, we need to create two new classes for our model loading to work: Mesh & Model.

```glsl
class Mesh
	{
	public:
        // Initialize the Mesh and give it its attributes
		void InitMesh(std::vector<Vertex> vertices, std::vector<unsigned> indices, std::vector<Texture> textures);

        // Bind our texture to our Mesh
		void BindTexture(const Shader shader) const;

        // Draw our Mesh
		void Draw(const Shader& pipeline) const;

        // Delete the Mesh
		void Delete();

        // Variables
		std::vector<Vertex> vertices;
		std::vector<unsigned int> indices;
		std::vector<Texture> textures;
		unsigned int vao_{}, vbo_{}, ebo_{};

	private:
		void SetupMesh();
	};
```

 *<center> Mesh class </center>*

 ```glsl
 class Model
	{
	public:
        // Keeping a count of the textures we loaded
		std::vector<Texture> textures_loaded;
		
		void InitModel(const char* path);
		void Draw(const Shader& pipeline) const;
		void Delete();

        // Contains all the meshes inside the model
		std::vector<Mesh> meshes;

	private:
        // File directory
		std::string directory;

        // Load the model in the program with the directory
		void loadModel(const std::string& path);

        // Assimp functions
		void processNode(const aiNode* node, const aiScene* scene);
		Mesh processMesh(aiMesh* mesh, const aiScene* scene);
		std::vector<Texture> loadMaterialTextures(aiMaterial* mat, aiTextureType type,
			std::string typeName);
	};
 ```

 *<center> Model class </center>*

 The functions inside the Model class are going to iterate through each mesh, and get their vertices and indices. After that, we will be able to draw our model and display it on the screen. We will also be able to move it with translations or rotations, or even scaling:

 ```glsl
 Mesh processMesh(aiMesh *mesh, const aiScene *scene)
{
    vector<Vertex> vertices;
    vector<unsigned int> indices;
    vector<Texture> textures;

    for(unsigned int i = 0; i < mesh->mNumVertices; i++)
    {
        Vertex vertex;
        // Process vertex positions, normals and texture coordinates
        [...]
        vertices.push_back(vertex);
    }
    // Process indices
    [...]
    // Process material
    if(mesh->mMaterialIndex >= 0)
    {
        [...]
    }

    return Mesh(vertices, indices, textures);
} 
 ```

 Once we initialized and bound our model, and called the draw function inside our Update method, this is what we get:

 <p align="center">
<img width="300" height="300" src="../img/opengl/model.png"><br>
</p>



Our model is correctly implemented in our scene! However the scene looks a bit empty...

## Adding a Cubemap

Having a completely black background can feel empty and lifeless. To change that, we can add a cubemap to our scene.

A cubemap is a texture that contains 6 2D textures that when combined, form a cube!
To implement it, we simply need to create it just like we would any textures, with the exception of using *GL_TEXTURE_CUBE_MAP* as a specifier in our *glBindTexture()* function:

```glsl
unsigned int cubemapTexture;
glGenTextures(1, &cubemapTexture);
glBindTexture(GL_TEXTURE_CUBE_MAP, cubemapTexture);
```

 We also can't forget that OpenGL gives us 6 special texture targets when it comes to the faces of a map: 

|Texture target|Orientation|
|---|---|
|GL_TEXTURE_CUBE_MAP_POSITIVE_X|Right|
|GL_TEXTURE_CUBE_MAP_NEGATIVE_X|Left|
|GL_TEXTURE_CUBE_MAP_POSITIVE_Y|Top|
|GL_TEXTURE_CUBE_MAP_NEGATIVE_Y|Bottom|
|GL_TEXTURE_CUBE_MAP_POSITIVE_Z|Back|
|GL_TEXTURE_CUBE_MAP_NEGATIVE_Z|Front|

Since a cubemap contains six 2D textures, we need to call *glTexImage2D()* 6 times:

```glsl
int width, height, nrChannels;
unsigned char *data;  
for(unsigned int i = 0; i < textures_faces.size(); i++)
{
    // Load our textures
    data = stbi_load(textures_faces[i].c_str(), &width, &height, &nrChannels, 0);
    glTexImage2D(GL_TEXTURE_CUBE_MAP_POSITIVE_X + i, 0, GL_RGB, width, height, 0, GL_RGB, GL_UNSIGNED_BYTE, data);
}
```

After loading our cubemap, we need to specify the faces, and create simple shaders that will display the cubemap:

```glsl
vector<std::string> faces;
{
    "right.jpg",
    "left.jpg",
    "top.jpg",
    "bottom.jpg",
    "front.jpg",
    "back.jpg"
};
```

One thing to note is that we need to make sure the cubemap is drawn on another cube that we created, with its own VAO and VBO.

We also need to make sure the cubemap is drawn after the scene. This helps us save some bandwidth since we don't need to draw on a pixel twice.

The last thing we need to be careful of is the cubemap's transformation matrix, indeed, we need to remove the translation matrix from our cubemap if we want it to stay still while we are moving:

```glsl
glm::mat4 view = glm::mat4(glm::mat3(camera.GetViewMatrix()));  
```

Our scene should now look more realistic:

 <p align="center">
<img width="800" height="400" src="../img/opengl/cubemap.png"><br>
</p>


Perfect! We gave our scene more life and more realism with this simple implementation.

## Post-Processing with Framebuffers

To add effects to our screen, or to create render textures, we can use what's called a Framebuffer. OpenGL gives us the flexibility to define our own framebuffers and define our own color (or depth & stencil) on them.

To create a framebuffer, just like any object, we need a *framebuffer object* (also FBO): 

```glsl
// Generate frame buffer
glGenFramebuffers(1, &framebuffer);
glBindFramebuffer(GL_FRAMEBUFFER, framebuffer);

// Check if framebuffer is complete
if (glCheckFramebufferStatus(GL_FRAMEBUFFER) != GL_FRAMEBUFFER_COMPLETE)
{
	// Else execute sad dance
	std::cerr << "ERROR: FRAMEBUFFER NOT COMPLETE." << std::endl;
}

// Back to default buffer
glBindFramebuffer(GL_FRAMEBUFFER, 0);
```

When we attach a texture to a framebuffer, all the draw calls will write to that texture as if it was a normal color buffer. To create a texture for a framebuffer we simply need to do it like normal:

```glsl
// Create the texture
glGenTextures(1, &textureColorbuffer);
glBindTexture(GL_TEXTURE_2D, textureColorbuffer);
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB, 1280, 720, 0, GL_RGB, GL_UNSIGNED_BYTE, NULL);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
glBindTexture(GL_TEXTURE_2D, 0);

// Attach it to currently bound framebuffer object
glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D, textureColorbuffer, 0);
```

To switch between buffers faster, and to also store depth and stencil data into a write-only buffer, creating Renderbuffers is the perfect occasion. To create one we have to do almost the same thing as we did with the framebuffer:

```glsl
// Generate and bind rbo
glGenRenderbuffers(1, &rbo);
glBindRenderbuffer(GL_RENDERBUFFER, rbo);
glRenderbufferStorage(GL_RENDERBUFFER, GL_DEPTH24_STENCIL8, 1280, 720);
glBindRenderbuffer(GL_RENDERBUFFER, 0);
glFramebufferRenderbuffer(GL_FRAMEBUFFER, GL_DEPTH_STENCIL_ATTACHMENT, GL_RENDERBUFFER, rbo);
```

We now have our Framebuffer implemented! To now draw the scene onto a single texture we have to go through two passes, drawing the scene on the framebuffer's texture, then bind to the default framebuffer and draw a quad on screen with the framebuffer's color buffer as its texture:

```glsl
// First pass
glBindFramebuffer(GL_FRAMEBUFFER, framebuffer);
glClearColor(0.1f, 0.1f, 0.1f, 1.0f);
glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT); // We're not using the stencil buffer now
glEnable(GL_DEPTH_TEST);
DrawScene();	
  
// Second pass
glBindFramebuffer(GL_FRAMEBUFFER, 0); // Back to default
glClearColor(1.0f, 1.0f, 1.0f, 1.0f); 
glClear(GL_COLOR_BUFFER_BIT);
  
screenShader.use();  
glBindVertexArray(quadVAO);
glDisable(GL_DEPTH_TEST);
glBindTexture(GL_TEXTURE_2D, textureColorbuffer);
glDrawArrays(GL_TRIANGLES, 0, 6);  
```

If we run the code using this implementation, nothing is going to appear different, so to make sure we correctly implemented the framebuffer, we are going to add some post-processing effects!

There are many post-processing effects out there, but the one I'm going to be using is the blur effect. To create it, we need to create a shader for it, and add a *kernel*, which is a convolution matrix, which is going to modify every pixel based on it's neighbouring pixels:

```glsl
float kernel[9] = float[](
        1.0/16.0, 2.0/16.0, 1.0/16.0,
        2.0/16.0, 4.0/16.0, 2.0/16.0,
        1.0/16.0, 2.0/16.0, 1.0/16.0
);
```
*<center>Blur kernel</center>*

We can now run our program and...

 <p align="center">
<img width="800" height="400" src="../img/opengl/blur.png"><br>
</p>

We now have Post-Processing effects!

## Instancing multiple models

Drawing one model at a time is very slow and not optimized at all. To fix that, we could try drawing each of the same model's meshes at the **same time**. Which is the reason why **Instancing** exists.

To do some instancing, we first need to change something about our vertex shader:
```glsl
layout (location = 0) in vec3 aPos;
layout (location = 2) in vec2 aTexCoords;
layout (location = 3) in mat4 instanceMatrix;
```

We need to add a new 4x4 matrix input called *instanceMatrix* to do our calculations for each matrix we are going to pass through this shader. We're no longer using a model uniform variable.

Since a matrix is basically 4 *vec4*s, we need to reserve 4 vertex attributes then set each of these attribute pointers:

```glsl
// Vertex buffer object
unsigned int buffer;
glGenBuffers(1, &buffer);
glBindBuffer(GL_ARRAY_BUFFER, buffer);
glBufferData(GL_ARRAY_BUFFER, amount * sizeof(glm::mat4), &modelMatrices[0], GL_STATIC_DRAW);
  
for(unsigned int i = 0; i < rock.meshes.size(); i++)
{
    unsigned int VAO = rock.meshes[i].VAO;
    glBindVertexArray(VAO);

    // Vertex attributes
    std::size_t vec4Size = sizeof(glm::vec4);
    glEnableVertexAttribArray(3); 
    glVertexAttribPointer(3, 4, GL_FLOAT, GL_FALSE, 4 * vec4Size, (void*)0);
    glEnableVertexAttribArray(4); 
    glVertexAttribPointer(4, 4, GL_FLOAT, GL_FALSE, 4 * vec4Size, (void*)(1 * vec4Size));
    glEnableVertexAttribArray(5); 
    glVertexAttribPointer(5, 4, GL_FLOAT, GL_FALSE, 4 * vec4Size, (void*)(2 * vec4Size));
    glEnableVertexAttribArray(6); 
    glVertexAttribPointer(6, 4, GL_FLOAT, GL_FALSE, 4 * vec4Size, (void*)(3 * vec4Size));

    glVertexAttribDivisor(3, 1);
    glVertexAttribDivisor(4, 1);
    glVertexAttribDivisor(5, 1);
    glVertexAttribDivisor(6, 1);

    glBindVertexArray(0);
}  
```

Now that everything has been declared, we simply need to create another *Draw()* function, which takes in an *amount* parameter, and use the *glDrawElementsInstanced()* function instead of the usual *glDrawElements()* function.

Let's run our program:

<p align="center">
<img width="800" height="400" src="../img/opengl/instancing.png"><br>
</p>

We can now render multiple elements at the same time!
And this does it for my 3D scene in OpenGL, we did a lot of things, but of course there are so many other things left that I haven't implemented.

## Features I couldn't implement

### Shadow Mapping

To render realistic shadows, we can use shadow mapping to calculate a pixel's depth value and either set it to black or color. After trying to implement shadow mapping for a week I couldn't make it work and ended up with a pretty interesting bug:

<p align="center">
<img width="800" height="400" src="../img/opengl/shadowdisaster.png"><br>
</p>

### Frustum culling

To optimize my scene, I used backface culling, which prevented the faces that we couldn't see from being drawn on screen. To optimize my scene even further, I could have implemented frustum culling, which uses the *frustum*, the region of space in the world that appears on screen.
Frustum culling discards any object that is not visible on screen and doesn't call the draw function on it, which saves a lot more time and makes the program run way faster and smoother.
Unfortunately due to the complexity of the implementation and my lack of time I decided to skip it.

## Conclusion

To conclude this blogpost, this was a very fun experience, even if I hit some roadblocks pretty frequently. But having a visual feedback felt good, especially after not working with graphical projects too much, and seeing where things went wrong made bug-fixing way easier.

As mentionned earlier, there are a lot of features that I wanted to implement but couldn't, for example HDR and Bloom, or Blending, or Frustum culling.

There are many other features that I implemented but didn't mention, such as backface culling, normal mapping, camera, stencil, and adding textures.
The goal of this post was to showcase the biggest steps.

Though I'm not 100% proud of my scene, I'm pretty proud of the progress I made and the features I implemented, because it was definitely not easy.
