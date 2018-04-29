# Intro #1
I recently decided that I should start to work on my own path tracing "engine" and so here we are. I will try to share as much as possible without it just becoming too much to read. Hopefully what I write will be of some use to others who might run into the same problems I do during this process. Just as a quick disclaimer, here is my setup which I plan to use for throughout the project (hate it or love it, this is what I'm working with):
* Windows 10
* Intel i7-6700K @4.2GHz
* Nvidia GTX 970 (4GB)
* 16GB RAM
* Microsoft Visual Studio Community 2017
* CUDA Toolkit 9.0
* OpenGL 4.3 (I will occasionally work from my laptop which does not support v4.5)
* SDL 2.0

# Intro #2
So, in this first installment we'll have a look at setting up CUDA and OpenGL in such a way that we can create our image through CUDA and display it using OpenGL. I'm completely new to CUDA, but have 2 years of OpenGL experience, so for me the challenge was getting CUDA to cooperate in an understandable way. Also, the OpenGL part of this is rather simple, so I won't go into much detail there. Let's get started!

# CUDA Toolkit
We will obviously need CUDA for this, so if you don't already have it installed, head over to https://developer.nvidia.com/cuda-downloads and downland and install it. As stated above, I'm using Windows, but it's available for both Linux and OSX as well. I only followed the default installation and it worked fine straight away. If you're on Windows, using Visual Studio will greatly simply compiling the coda we'll write, so if you don't have that installed, I'd recommend you do that too (before you install the CUDA toolkit) https://www.visualstudio.com/downloads/ . After you've installed both, simply create a new CUDA project (which should now be an option in Visual Studio).

# OpenGL
Next, magically make an OpenGL window appear. If you don't know how to do this, this tutorial is might not be what you are looking for (just maybe). Make sure you remember adding the necessary additional _include_, _lib_ and _dependencies_ in Visual Studio's project manager. To make sure everything works, draw two triangles forming a square covering the entrire screen. Also, give each vertex respective texture coordinates. My code located in **_Display.cpp_** looks like this:

```cpp
void GLInit(const std::string& vertex_shader, const std::string& fragment_shader)
{
    shader = ShaderCreate(vertex_shader, fragment_shader);
    const int num_vertices_points = 24;
    GLfloat vertices[num_vertices_points] = {
        -1.0f, +1.0f,	0.0f, 1.0f,
        -1.0f, -1.0f,	0.0f, 0.0f,
        +1.0f, -1.0f,	1.0f, 0.0f,

        +1.0f, -1.0f,	1.0f, 0.0f,
        +1.0f, +1.0f,	1.0f, 1.0f,
        -1.0f, +1.0f,	0.0f, 1.0f
    };
    glGenBuffers(1, &vbo);
    glBindBuffer(GL_ARRAY_BUFFER, vbo);
    glBufferData(GL_ARRAY_BUFFER, num_vertices_points * sizeof(GLfloat), &vertice[0], GL_STATIC_DRAW);
    glGenVertexArrays(1, &vao);
    glBindVertexArray(vao);
    glEnableVertexAttribArray(0);
    glEnableVertexAttribArray(1);
    glVertexAttribPointer(0, 2, GL_FLOAT, GL_FALSE, 4 * sizeof(GLfloat), 0);
    glVertexAttribPointer(1, 2, GL_FLOAT, GL_FALSE, 4 * sizeof(GLfloat), (void*(2 * sizeof(GLfloat)));
    glBindVertexArray(0);
    glBindBuffer(GL_ARRAY_BUFFER, 0);
}
```

Also, notice how we are creating a shader that has the following **vertex**

```c
#version 430
in layout(location=0) vec2 position;
in layout(location=1) vec2 v_uv;
out vec2 f_uv;
void main()
{
    f_uv = v_uv;
    gl_Position = vec4(position, 0.0f, 1.0f);
}
```

and **fragment** shader

```c
#version 430
in vec2 f_uv;
uniform sampler2D s;
out vec4 frag_color;
void main()
{
    frag_color = texture(s, f_uv);
}
```

The function that will draw this to the screen looks like this:

```cpp
void DisplayUpdate(SDL_Window* window, GLuint texture)
{
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);

    glUseProgram(shader);
    glBindBuffer(GL_ARRAY_BUFFER, vbo);
    glBindVertexArray(vao);
    glBindTexture(GL_TEXTURE_2D, texture);

    glDrawArrays(GL_TRIANGLES, 0, 6);

    glBindTexture(GL_TEXTURE_2D, NULL);
    glBindVertexArray(NULL);
    glBindBuffer(GL_ARRAY_BUFFER, NULL);
    glUseProgram(NULL);

    SDL_GL_SwapWindow(window);
}
```

This about does it for the "pure" OpenGL part. All of the code above should be self explanatory, and so I will not go into any more detail. Next, we need to deal with the texture that is being passed in as an argument to **_DisplayUpdate_**.

# CUDA-OpenGL interoperability
First off, let's briefly mention what's great about the concept of CUDA-OpenGL interoperability. Well, as you probably know, both CUDA and OpenGL utilize the GPU to do its work. This means that if we create a 2D image using CUDA, that data is located in the GPU's memory. The same goes for OpenGL, if we make a call to _glTexImage2D_, we are uploading that data to the GPU's memory. We can from this clearly see that if we made it so that both CUDA and OpenGL referred to the same memory on the GPU, we could write to it from CUDA, and display it using OpenGL without having the data needing to go via the system's main memory. That's the concept; the implementation was not so trivial to work with, and it took me several hours to get it working properly.

The following code is located in **_GLCUDA.h_**

```cpp
#include "include\GL\glew.h"
#include "cuda_gl_interop.h"
void GLCUDAInit();
void GLCUDARunKernel();
GLuint GLCUDAGetTexture();
```
Take note of the fact that _glew.h_ is included before _cuda_gl_interop.h_, as this is required. The rest of the code is from **_GLCUDA.cu_**, which has the following global variables:
```cpp
static GLuint gl_texture;
static cudaGraphicsResource* cuda_resource;
static cudaArray* gl_cuda_array;
surface<void, cudaSurfaceType2D> cuda_surface_write;
```

Let's go through the functions one by one, code first, followed by an explanation.

```cpp
void GLCUDAInit()
{
    //OpenGL
    glGenTextures(1, &gl_texture);
    glBindTexture(GL_TEXTURE_2D, gl_texture);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_REPEAT);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_REPEAT);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
    glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA8, DISPLAY_WIDTH, DISPLAY_HEIGHT, 0,  GL_RGBA, GL_UNSIGNED_BYTE, NULL);
    glBindTexture(GL_TEXTURE_2D, 0);

    //CUDA
    cudaGraphicsGLRegisterImage(&cuda_resource, gl_texture, GL_TEXTURE_2D,  cudaGraphicsRegisterFlagsSurfaceLoadStore);
    cudaGraphicsMapResources(1, &cuda_resource, 0);
    cudaGraphicsSubResourceGetMappedArray(&gl_cuda_array, cuda_resource, 0, 0);
    cudaGraphicsUnmapResources(1, &cuda_resource, 0);
}
```

We first create an empty 2D texture in OpenGL. Then we register the texture with a CUDA resource. Then we _map_ it, allowing CUDA to access the resource (can be thought of as _glBind*_). The crucial function call is _cudaGraphicsSubResourceGetMappedArray_, which given a resource and a pointer, will give us the starting memory address of the texture attached to the resource. We then call _unmap_ (just as in OpenGL where we unbind). Great, we now have a pointer to te texture which we can write to. However, since the memory belongs to OpenGL, we can't "just" write to it like we would usualy do on the CPU. Additionally, what we have is a pointer to a _cudaArray_, which is a struct. So, to write to it, we do the following

```cpp
void GLCUDARunKernel()
{
    cudaBindSurfaceToArray(cuda_surface_write, gl_cuda_array);
    //https://en.wikipedia.org/wiki/CUDA
    //Each block has 1024 threads = 32 warps
    //The image has 1920*1080=2,073,600 pixels -> 2,073,600/1024=2025 blocks    needed in grid
    dim3 block_size(32, 32); //32*32=1024 threads per block (=max)
    dim3 grid_size(60, 34); //32*60=1920, 32*34=1088 -> remeber to check if y_dim>1080
    GLCUDAWriteToTex <<<grid_size, block_size>>>(DISPLAY_WIDTH, DISPLAY_HEIGHT);
}
```

We bind out _cudaArray_ to a surface which we defined at the top of the file to be a _cudaSurfaceType2D_ (CUDA does not have any _unbind_ functions like OpenGL). We then specify how many threads we want. If you don't understand this part, I encourage you to watch this presentation https://youtu.be/nRSxp5ZKwhQ?t=27m6s. For simplicity, I know that my OpenGL window is 1920x1080, so I hardcoded the dimensions of my blocks and grid. We then launch our kernel with those dimensions, providing the window width and height. The reason we send the width and the height is that as you can see, our grid will launch too many threads in the y-dimension, and so we have to make sure that we only use the threads we need, so that we do not write to memory locations not part of the texture we are trying to write to. The actual kernel looks like this

```cpp
__global__ void GLCUDAWriteToTex(int width, int height)
{
    int y = blockIdx.y * blockDim.y + threadIdx.y;
    if (y < height)
    {
        int x = blockIdx.x * blockDim.x + threadIdx.x;
        surf2Dwrite(make_uchar4(127, 127, 0, 255), cuda_surface_write, x * sizeof(uchar4), y);
    }
}
```

What we do here is first calculate our y index, making sure that we are inside within the height of the texture, calculate the x index, and then write to the texture. You should take note of how we in this case are using a _uchar4_. This is because the texture we created with OpenGL has an internal format of _GL_RGBA8_ which means that each texel is composed of 4 values, each 8 bits. An unsigned char (_uchar_) is also 8 bits, and so we need 4 of them to fill each texel with the necessary data it needs. Lastly, _surf2DWrite_ does not inherently know know this, so we have to specify the x-coordinate with a byte offset. That is why we pass

```cpp
x * sizeof(uchar4)
```

as our x index.

# Finishing up
Alright, when integrating this with the rest of the code, it should produce a yellow-brown-ish window (remember that you need to compile **_GLCUDA.cu_** with _nvcc_ and not _msvc_). My own _main_ loop looks like this

```cpp
SDL_Window* window = DisplayInit();
GLInit("shader/shader.vert", "shader/shader.frag");
GLCUDAInit();
while (DisplayOpen())
{
    GLCUDARunKernel();
    cudaDeviceSynchronize();
    DisplayUpdate(window, GLCUDAGetTexture());
}
GLCUDADestroy();
GLDestroy();
DisplayDestroy(window);
```

Should you run into any problems, feel free to comment and I'll try my best to help.

Thanks, Daniel