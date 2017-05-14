Hi, and welcome to my first blog post!

I am not going to bore you with unecessary talk, but let me just say thank you for stopping by and I hope you will find what I have to write useful. So, let us get started!

Now, what are we going to do today? Well, we are going to create a simple program using C++ and OpenGL that allows us to blur our game and use it as our "background" when displaying our menu. It can of course be used for whatever you would like, but that is our main goal here. I will try my best to provide you with as much details as I can, but I will not be learning you everything you need to now. Here are the prerequisites I expect any reader to be familiar with/have in order to be able to complete this tutorial:

A working understanding of C++
A working understanding of OpenGL
A basic understanding of shaders (GLSL)
An OpenGL project with something drawing to the screen
I will however give a bit of a deeper introduction to image processing, framebuffers and shaders, as they will be of great importance for this implementation. Enough talk, let us get going!

Let us start with our basic screen being rendered to:

There is not anything special going on here...I have just two quads that I am rendering two textures onto. Do not mind the yellow lines, they are not relevant to our current goal. Now, how do we blur this? Well, let us first agree on the fact that this is nothing but a 2 dimensional grid of pixel. It is an image and nothing more, and through the fragment shader we can access these pixels just as one would do in Photoshop or Paint. We can manipulate them in order to achieve our desired effect. But unless you are familiar with basic image analysis, this would not help you much. So let us dive into one of image analysis's most useful topics: blurring.

As we have already agreed on, an image is a 2D array of pixels (integers). Here is a small example of what a 5x5 grayscale image could look like:
Figure 1: The black square is the pixel                       we are working with
Figure 1: The black square is the pixel                       we are working with

This is just one example of many. Remember that grayscale images that utilize 8-bit color channels can store values in the range 0-255, and since grayscale images only have one color channel, there is only one integer present in each pixel. Now, before I give you the simplest solution on how to blur this random 5x5 image I encourage you to try and think of a way to solve this yourself. I know...the person writing the tutorial is asking you to do some thinking...how rude! I understand if you just want to continue reading, as I have done in many tutorials myself, but I encourage you to at least try for a few minutes.

OK, the simplest answer is as follows: say we want to blur the pixel in position [1] [1], let us call it P. We could find the mean number among all P's neighbors, and replace the integer in P with this new number that we have found. However, there is one problem with this...do you see it? What if we have the following 5x5 image instead:
                          Figure 2
                          Figure 2

What happens if we iterate over the neighbors of P and select the mean? Well, first we get the following numbers (sorted) that classify as P's neighbors: [0, 0, 0, 0, 0, 0=P, 255, 255, 255]. There are nine numbers in this array, and since they are sorted, the mean is located at index 4 which equals 0. If we were to perform this operation on all of the pixels in this image, the result would be identical to the original image. Try it out if you do not see the pattern. This is of course a rare case, and if we were to do this on the image in Figure 1, we would not get the same output as our input. However, this highlights an issue with simly taking the mean; in the image in Figure 2 one would expect a blurred version of the image to have grey pixels in areas where some of the neighboring pixels are completely black (a value of 0) and the rest of the neighbors are completely white (a value of 255). Let us fix this right away!

The next solution would be to replace the value of our pixel P with the average of its' neighboring pixels. Let us do that: (0 + 0 + 0 + 0 + 0 + 0 + 255 + 255 + 255) / 9 = 765 / 9 = 85 If we do this for every pixel in the image in Figure 2, we get the following output image:
                           Figure 3
                           Figure 3

This looks much more like we would expect. Areas that only have completely white pixels remain white, areas with only completely black pixels remain black, areas with more white than black pixles become light grey, while areas with more black than white pixels become dark grey. Pretty neat huh?! This is not bad, but we can do even better. But before we do, let us define a word used in image analysis a lot: filter.

Simply put, a filter is an operation that is performed on each pixel in an image. When we took the mean of P's neighbors earlier, we utilized what is known as a mean filter. When we took the average of P's neighbors we used a averaging filter. And to improve our results even more, we are going to use what is known as a Gaussian filter. Now, depending on how much mathematics you have been exposed to, you may have heard of a Gaussian functions before. If you have not, do not fear, because we are not going to go into the details of that in this tutorial. We will just be using what a Gaussian function provides; a normal distribution. If you want to learn more about the Gaussian function and its' applications, you can start on Wikipedia.

Let us agree on one more thing before we introduce Gaussian filters. The average filter just used can be expressed as follows:
       Figure 4
       Figure 4

You might be wondering if I have entered wrong numbers, or if I am trying to play some trick on you. I assure you that I am not. What you are seeing here is known as a kernel (not to be confused with kernels related to operating systems). A kernel is simply a grid, 1 or 2 dimensional, that is convolved over an image. Convolved basically means moved across every pixel. Let me try to explain how we can implement our average filter using the image in Figure 4 and convolution.

Given our image in Figure 2 and our goal of averaging each pixel using its' neighbors, what happens if we move our kernel in Figure 4 over each pixel in our image, and at each pixel, we multiply the value in each of the positions in our kernel with the value of the pixel at the same location in our image, with an offset of where the kernel is currently centered in our image. Let me illustrate to allow you to visualize what we are doing:
                         Figure 5
                         Figure 5

What we have done here is to put the center of our kernel ([1] [1]) at the top left pixel in our image ([0] [0]). This results in the black pixels in the image in Figure 5 - they are all covered by our filter. If we move our filter one pixel to the right, the following pixels in our image would be covered:
                         Figure 6
                         Figure 6

If we put the center of our filter on pixel [2] [2], the covered pixels in our image would be:
                         Figure 7
                         Figure 7

Do you see the pattern? If you are having trouble with it, I recommend checkoing out the following explanation - go to the convolution section. Now, given the image in Figure 7 and our kernel in Figure 4, what do we do next. Well, we multiply each value in our kernel with the value located in the pixel it is currently covering and sum the results. This results in the following expression:

(1 * 0) + (1 * 0) + (1 * 0) + (1 * 255) + (1 * 255) + (1 * 255) + (1 * 255) + (1 * 255) + (1 * 255) = 1530

Now, recall that when I presented you with the resulting image after blurring it with an averaging filter, the value in position [2] [2] was 170. How do we make 1530 become 170? Well, notice that in order to get 1530, we multiplied nine locations in our kernel with their corresponding image pixels values. So what do we do next, we divide 1530 by 9:

1530 / 9 = 170

There we go, we have successfully blurred pixel [2] [2] in our image using an averaging filter which we represented using the kernel in Figure 4. Some of you might be wondering what happens if the center of our filter is located on one of the edges of our image or in the corner. Well, that is up to you. You can either always divide by 9, or you can only divide by the number of kernel - image multiplications you are able to perform in your current scenario.

We can now move on to the Gaussian filter which can be represented by the following kernel:
       Figure 8
       Figure 8

As you can see, it is similar to our average kernel in Figure 4, only that instead of all 1's, we now have varying values depending on location. This has the purpose known as weighting. When we perform our convolution as before, by moving the kernel around the image, multiplying, summing and dividing, the weights will act as a way for us to be able to specify a pixels importance. Since the center value in our Gaussian kernel is 4, the center pixel we are currently convolving on (the pixel that we are trying to blur according to its' neighbors) will be weighted 4 times as much as a neighbor located under one of the corners of our kernel. You will either have to accept this, or start reading up on Gaussian filter and their place in image analysis (1 and 2). Now, there is one more important thing to consider before our Gaussian filter is working correctly. When using our average kernel, remember that we divded by 9. Well, the reason we divided by 9 was that the sum of the values in our averaging kernel summed to 9. In our Gaussian kernel however, they sum to 16. We will therefore be diving by 16 this time around.

So, that is really all we need in terms of "new" image processing knowledge. There is however one thing to note; the size of the kernels I have shown you are all 3x3 - they do not have to be that size. They can be any size you want, but it is common to use sizes 3x3, 5x5, 7x7, etc. That way, there is always a center value in our kernel that acts as our reference point when we are convolving our image with it.

Phew...that was quite a lot. If you got through it all, the rest is going to be easier as long as you have your OpenGL project set up and ready to go. Let us continue!

Let me start this part by explaining my project setup. First of all, I am using SDL 2.0 for windowing and keyboard input. I will let you know whenever I am using any of the functions SDL provides or I am using something derived from SDL. OK, so I have a main game-loop which looks as follows:

while (window->isOpen())
{
    window->clear();
    switch (gameState)
    {
    case GAME:
        drawGame();
        break;
    case MENU:
        drawMenu();
        break;
    }
    window->update();
}
Note that window is a class that contains a SDL_Window*. So, as long as the window is open, we start by clearing our previous rendered frame, followed by checking our game state, and lastly we update our window, i.e. render our newest frame. I am going to let you create your own menu. What we are interested in however is what happens when we call drawGame(). This is where we are going to capture the last frame of the game, performing our image processing on it, storing it, and finally using it when we draw our menu.

Let us have a look at our drawGame() function:

void Game::drawGame()
{
    //Check for events and act accordingly
    Events::checkForGameEvents(player, &gameState);
    if (gameState == GameState::MENU)
        menuBackgroundFramebuffer->renderTo();

    //Do your game logic here
    ...

    //Finish by drawing your game
    ...

    //Clean up after switching to displaying the game menu
    if (gameState == GameState::MENU)
        blur(quadWithUV, menuBackgroundFramebuffer, intermediateFramebuffer, 5, gaussianBlurShader, quadShader);
}
Let us take it from the top. We start by checking for any events. Here I am using SDL, but that should not make any real impact on the code as you should simply detect what keys are pressed etc. We pass a reference to our gamestate into this function which is altered inside the function if Escape is pressed. Let us now assume Escape is indeed pressed and gamestate is updated to hold the value GameState::MENU. Now it starts getting exiting!

We check the state of gamestate and since we did press Escape, we enter the if-statement, which calls menuBackgroundFramebuffer->renderTo(). Let us have a look in there to see what is going on. First off we have the struct called Framebuffer located in Framebuffer.h:

struct Framebuffer
{
    GLuint texture, buffer;
    unsigned int width, height;

    Framebuffer(unsigned int, unsigned int);
    ~Framebuffer();
    inline void renderTo()
    {
        glBindFramebuffer(GL_FRAMEBUFFER, buffer);
        glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
    }

    inline void renderDefault()
    {
        glBindFramebuffer(GL_FRAMEBUFFER, 0);
        glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
    }
};
In Framebuffer.cpp we have the follwowing constructor and destructor:

#include <iostream>
#include "Framebuffer.h"
Framebuffer::Framebuffer(unsigned int _width, unsigned int _height)
{
    width = _width;
    height = _height;

    glGenTextures(1, &texture);
    glBindTexture(GL_TEXTURE_2D, texture);
    glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA,
        width, height, 0, GL_RGBA, GL_FLOAT, NULL);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP);

    glGenFramebuffers(1, &buffer);
    glBindFramebuffer(GL_FRAMEBUFFER, buffer);
    glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D, texture, 0);
    GLenum status = glCheckFramebufferStatus(GL_FRAMEBUFFER);
    if (glCheckFramebufferStatus(GL_FRAMEBUFFER) != GL_FRAMEBUFFER_COMPLETE)
    {
        std::cout << "Frame buffer failed: " << status << std::endl;
        system("pause");
        exit(1);
    }

    glBindFramebuffer(GL_FRAMEBUFFER, 0);
    glBindTexture(GL_TEXTURE_2D, 0);
}

Framebuffer::~Framebuffer()
{
    glDeleteTextures(1, &texture);
    glDeleteFramebuffers(1, &buffer);
}
Now, if you have never worked with framebuffers before, this might seem a bit daunting. No need to worry; although they can be a pain at times, think of them as a texture that can be rendered to. This means that instead of rendering to the screen, we can render our scene to a framebuffer and do with it as we please. In the constructor we generate a texture, then a framebuffer object and attach the texture to the framebuffer object. The _width and _height you pass in when creating it should be the size of the window you are running your game in. We finish by doing some error checking to make sure that the framebuffer was created successfully and clean up after ourselves. The destrcutor should be self-explanatory.

If you recall, what brought us into Framebuffer.h was the call to renderTo(). In this function we simply set the framebuffer as the render target for OpenGL and clear the previous contents of it. What this means is that as long as we do not specify anything else, our window will not be rendered to, only our framebuffer.

Then, as the comment states, you perform you regular game logic and raw your game - nothing fancy here. Remember that since we set our framebuffer as the rendertarget, your game will not be drawn to the screen this time. Then we again check if the gamestate has been altered to equal GameState::MENU. If it has, we proceed into the if-statement and make a call to blur() which looks as follows:

void blur(Quad* quad, Framebuffer* target, Framebuffer* intermediate, const unsigned short numberOfPasses, Shader* blurShader, Shader* passShader)
{
    bool renderTarget = false;
    for (int i = 0; i < numberOfPasses; i++)
    {
        if (renderTarget)
        {
            target->renderTo();
            quad->draw(intermediate->texture, blurShader);
        }
        else
        {
            intermediate->renderTo();
            quad->draw(target->texture, blurShader);
        }

        renderTarget = !renderTarget;
    }

    if (numberOfPasses % 2 != 0)
    {
        target->renderTo();
        quad->draw(intermediate->texture, passShader);
    }

    target->renderDefault();
}
Let us take is step by step, and start with the parameters:

Quad is simply a rectangle that is the size of the screen that has the ability to draw itself with a texture on itself
Framebuffer target is the framebuffer containing our game rendered to its' texture
Framebuffer intermediate is a framebuffer, just as target, but that is empty and who's only purpose is to serve as a temporary framebuffer which we will discuss shortly
numberOfPasses tells us how many times we are going to blur the texture located in target
Shader blurShader is the shader that is capable of blurring our texture
Shader passShader is the shader used to simply pass a texture between two framebuffers
Let us first discuss numberOfPasses. Up until now I have never mentioned the fact that we can blur an image multiple times; well we can. The more times we blur it, the more blurry it will become, shocker right. Here is an example:
This texture has been blurred one time
This texture has been blurred one time

This texture has been blurred 20 times
This texture has been blurred 20 times

As you can see, the difference is quite large, and you should blur it the number of times required until you get your desired effect. However, keep in mind that each blur pass is not free, and doing an excessive amount of blurs per frame will increase your frame time.

Let us move on into the code of blur(). We start by setting bool renderTarget = false;. This variable is simply used to keep track if we rendered to target or intermediate, and since we start by wanting to render to intermediate we set it to false. We then start to loop over the number of blur passes we want to make. For each iteration, we check which framebuffer we are going to render to. Since renderTarget was set to false, we start off by entering the else-statement which does the following: it sets the intermediate framebuffer as the rendertarget (remember this means that OpenGL will render to this framebuffer's texture). It finishes by calling draw() which is a function that is a memeber of Quad. It looks like this:

inline void draw(GLuint texture, Shader* shader)
{
    shader->bind();
    glBindVertexArray(vao);
    glBindTexture(GL_TEXTURE_2D, texture);
    glDrawArrays(GL_TRIANGLES, 0, 6);

    glBindTexture(GL_TEXTURE_2D, 0);
    glBindVertexArray(0);
}
It basically draws a quad that covers the entire screen with the texture and shader specified. At this point it is worth noting that Shader is a simple class that holds an GLuint that represents our shader program. You should be familiar with this. All shader->bind() does is call glUseProgram(program) and passing in the shader program we want to use. Before we go onto discussing the actual shaders, let us finish with blur().

We finish each iteration by altering the value of renderTarget so that next time (if there is a next time) we render to the other framebuffer available. The reason for this altering of rendertargets is that a framebuffer can only hold one texture which contains our data, and rendering to the texture we are reading from does not seem like a good idea. I feel the need to emphasize what we are passing into to quad->draw(). Notice that we are setting one framebuffer as our rendertarget and passing in the other to the draw method.

Finally, once we are done with all our blur passes, we check if the number of passes we did was odd or even. If it was odd, we know that the intermediate framebuffer holds our final blurred texture, but we do not want that, we want our target framebuffer to hold the final texture. We therefore do an "empty pass" where we simply render the contents in the texture of intermediate onto the texture in target. This essentially copies the texture from one framebuffer to the other. We finish by calling target->renderDefault();. It looks like this:

inline void renderDefault()
{
    glBindFramebuffer(GL_FRAMEBUFFER, 0);
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
}
This will reset our rendertarget to OpenGL's default rendertarget which will allow us to render to our screen again. If we do not make this call, we will continue to render to the texture attached to the target framebuffer and we will never get to see our beautiful blurred background.

Wow...let us take a small breather...breath in...and out...in...and out....

If you were reading thoroughly, or were paying close attention, or both, you should notice that we are missing a crucial part of our program; the shader(s). Let us start of easy with the passShader we passed in as the last parameter into blur(). The vertex shader is as simple as it gets when using a texture:

#version 430

in layout(location=0) vec3 position;
in layout(location=1) vec2 uv;

out vec2 texCoord;

void main()
{
    texCoord = uv;
    gl_Position = vec4(position, 1.0f);
}
And the fragment shader is just as simple:

#version 430

in vec2 texCoord;

uniform sampler2D sampler;

out vec4 color;

void main()
{
    color = texture(sampler, texCoord);
}
None of these shaders should look strange to you. But now comes the fun part, the shaders that performs the actual blurring of the texture we pass in. Here we go; the vertex shader is identical to that of the pass shader so nothing new there, but here comes the fragment shader...it's a handful.

#version 430

in vec2 texCoord;

uniform sampler2D sampler;

out vec4 color;

void main()
{
    float kernel[5][5] = { 
        {1.0f, 4.0f,  6.0f,     4.0f,  1.0f},
        {4.0f, 16.0f, 24.0f, 16.0f, 4.0f},
        {6.0f, 24.0f, 36.0f, 24.0f, 6.0f},
        {4.0f, 16.0f, 24.0f, 16.0f, 4.0f},
        {1.0f, 4.0f,  6.0f,  4.0f,  1.0f}
    };
    const float sum = 256.0f;

    const ivec2 textureSize2D = textureSize(sampler, 0);
    const float xOffset = 1.0f / textureSize2D.x;
    const float yOffset = 1.0f / textureSize2D.y;


    //Row 1
    vec4 blurredNonNormalized = texture(sampler, vec2(texCoord.x - (2 * xOffset), texCoord.y - (2 * yOffset))) * kernel[0][0];
    blurredNonNormalized +=        texture(sampler, vec2(texCoord.x -        xOffset,  texCoord.y - (2 * yOffset))) * kernel[1][0];
    blurredNonNormalized +=        texture(sampler, vec2(texCoord.x,                  texCoord.y - (2 * yOffset))) * kernel[2][0];
    blurredNonNormalized +=        texture(sampler, vec2(texCoord.x +        xOffset,  texCoord.y - (2 * yOffset))) * kernel[3][0];
    blurredNonNormalized +=        texture(sampler, vec2(texCoord.x + (2 * xOffset), texCoord.y - (2 * yOffset))) * kernel[4][0];

    //Row 2
    blurredNonNormalized +=        texture(sampler, vec2(texCoord.x - (2 * xOffset), texCoord.y -        yOffset))  * kernel[0][1];
    blurredNonNormalized +=        texture(sampler, vec2(texCoord.x -        xOffset,  texCoord.y -        yOffset))  * kernel[1][1];
    blurredNonNormalized +=        texture(sampler, vec2(texCoord.x,                  texCoord.y -        yOffset))  * kernel[2][1];
    blurredNonNormalized +=        texture(sampler, vec2(texCoord.x +        xOffset,  texCoord.y -        yOffset))  * kernel[3][1];
    blurredNonNormalized +=        texture(sampler, vec2(texCoord.x + (2 * xOffset), texCoord.y -        yOffset))  * kernel[4][1];

    //Row 3
    blurredNonNormalized +=        texture(sampler, vec2(texCoord.x - (2 * xOffset), texCoord.y))                   * kernel[0][2];
    blurredNonNormalized +=        texture(sampler, vec2(texCoord.x -        xOffset,  texCoord.y))                   * kernel[1][2];
    blurredNonNormalized +=        texture(sampler, vec2(texCoord.x,                  texCoord.y))                   * kernel[2][2];
    blurredNonNormalized +=        texture(sampler, vec2(texCoord.x +        xOffset,  texCoord.y))                   * kernel[3][2];
    blurredNonNormalized +=        texture(sampler, vec2(texCoord.x + (2 * xOffset), texCoord.y))                   * kernel[4][2];

    //Row 4
    blurredNonNormalized +=        texture(sampler, vec2(texCoord.x - (2 * xOffset), texCoord.y +        yOffset))  * kernel[0][3];
    blurredNonNormalized +=        texture(sampler, vec2(texCoord.x -        xOffset,  texCoord.y +        yOffset))  * kernel[1][3];
    blurredNonNormalized +=        texture(sampler, vec2(texCoord.x,                  texCoord.y +        yOffset))  * kernel[2][3];
    blurredNonNormalized +=        texture(sampler, vec2(texCoord.x +        xOffset,  texCoord.y +        yOffset))  * kernel[3][3];
    blurredNonNormalized +=        texture(sampler, vec2(texCoord.x + (2 * xOffset), texCoord.y +        yOffset))  * kernel[4][3];

    //Row 5
    blurredNonNormalized +=        texture(sampler, vec2(texCoord.x - (2 * xOffset), texCoord.y + (2 * yOffset))) * kernel[0][4];
    blurredNonNormalized +=        texture(sampler, vec2(texCoord.x -        xOffset,  texCoord.y + (2 * yOffset))) * kernel[1][4];
    blurredNonNormalized +=        texture(sampler, vec2(texCoord.x,                  texCoord.y + (2 * yOffset))) * kernel[2][4];
    blurredNonNormalized +=        texture(sampler, vec2(texCoord.x +        xOffset,  texCoord.y + (2 * yOffset))) * kernel[3][4];
    blurredNonNormalized +=        texture(sampler, vec2(texCoord.x + (2 * xOffset), texCoord.y + (2 * yOffset))) * kernel[4][4];

    //Normalize
    vec4 blurredNormalized = blurredNonNormalized / sum;

    color = blurredNormalized;
}
(I recommend copying this shader into a text editor to get a better overview)
Do not freak out, once we step through it, it should all make sense if I have done a good enough job explaining the image processing part.

We start by creating a 5x5 Gaussian kernel called kernel. Recall that a kernel can be of any size, not just 3x3. Since we are now using a 5x5 kernel, which essentially means that we will be taking more neighbors into account when blurring, we need a larger normal distribution. The numbers we use are 1.0f, 4.0f, 6.0f, 16.0f, 24.0f and 36.0f. Gaussian kernels of many size can be found if you do a bit of googleling. We then need to remember that we should divide by the sum of all the values in our kernel at the end, which in this case equald 256. Hopefully this makes sense to you. Then we need to calculate our current coordinate in the texture we are working with. Not that GLSL's coordinate system looks like this:

We start by retrieving the size of the texture and store it in a vec2. We then calculate our actual x- and y-cooridnates by dividing 1.0f by the width or height. The reason for it being 1.0f is that the max x- and y-coordinates are 1.0f. We then begin the "tedious" of multiplying each entry in our kernel with the values located in the neighboring fragments to our current fragment. Notice that we are storing the results in a vec4 because our texture holds RGBA values for each framgment and we want to take all the color channels into consideration when we are blurring the texture. Although the code looks like a lot, it is simply a matter of looking up at a point in the texture and multiplying it with a value in the kernel. There is a clear pattern, so you should not need more than a few minutes to see the logic behind the offsets etc.

We then normalize our result vector by dividing it by the sum of the kernel. Finally, we output the blurr color to the current fragment we are processing. And that is it! If everything went smoothly, you should have a blurred texture located in your framebuffer's texture. Now, all that remains is to draw our menu, which is done simply by first rendering a quad with your blurred texture on it, and then rendering you actual menu on top of it. Remember that regardless of how many blur passes you decide to do, you final blurred texture will always be found in the target framebuffer you pass into the blur() function.
Whooaa...I did not expect this post to be this long. Oh well, better to be thorough I suppose. I truly hope you were able to soak up all of this, I know it is much. I would most likely not be able to do all of this in one sitting myself if I knew nothing of image processing beforehand. Anyway, do not hesitate to ask any questions, and regardless of if you got it working right away, have yourself a small break, lean back and breath. And if you got it working, then hey; you deserve a cookie :)

Until next time