# Ascii Art rendering for Opengl linux games over VNC
## :triangular_flag_on_post: ==The Objective==
After being introduced to UnrealTournament libcaca demo, I wanted to test it on other linux 3D games.  However, I was faced with disappointment, if the game does not have an in-built software renderer solution, it would not work.
## :triangular_flag_on_post: ==libaa==
libaa seems to be a grayscale 8bit compatible image/frame renderer. That is it converts a pixel frame of 8bit format into a grayscale ascii art representation.
### --Source links--
[https://github.com/1480c1/aalib](https://github.com/1480c1/aalib)  
[https://aa-project.sourceforge.net/aalib/](https://aa-project.sourceforge.net/aalib/)  
### --AAOPTS environment variable--
[https://aa-project.sourceforge.net/aalib/aalib_9.html](https://aa-project.sourceforge.net/aalib/aalib_9.html)  
## :triangular_flag_on_post: ==libcaca==
libcaca seems to be a colour 32bit compatible image/frame renderer. That is it converts a pixel frame of 32bit format into a colour ascii art representation.
## :triangular_flag_on_post: ==What went wrong? Shouldn't it be easy?==
First of all, if you use the libraries directly, then obviously you just need as input a series of frames/images that you parse to it, then they get re-rendered in their new form. Either as pixels or sometimes as plain text in a terminal, how, is defined by the 'driver' option you give to them.  

I did not realise this independence initially, and only saw their usage through `SDL` via the **SDL_VIDEODRIVER=aalib** and **SDL_VIDEODRIVER=caca** avenues.  So for the time-being we will just assume this route, one plus side of using the SDL route is that it doesn't involve recompilation of the target program, although with 3d games without software render support, its irrelevent because it won't work.  

Why it does not work with 3d renderered opengl games is because they send vertices to the gpu and the gpu will render the pixels within its-self.  Thus, aalib would need to `pull` pixels from the gpu.  This is possible, but involves much more interplay between parts of the system so is harder to implement. More on this later. 
## :triangular_flag_on_post: ==SDL==
libaa and libcaca are features within SDL 1.2, not SDL 2.  

You can see here how aalib is used within SDL.  
[https://github.com/libsdl-org/SDL-1.2/tree/main/src/video/aalib](https://github.com/libsdl-org/SDL-1.2/tree/main/src/video/aalib)  
Essentially by defining a video renderer in SDL, it is able to override what the Key SDL API functions perform.  So, eg. upon SDL_Flip(), it will internally call `aa_flush` : [here](https://github.com/libsdl-org/SDL-1.2/blob/0569bc61fe2ad18c8adbba72225d546336242999/src/video/aalib/SDL_aavideo.c#L323C2-L323C23)

So with this setup, you have less fine grain control over the ascii libraries, and have to make-do with how they are arranged within SDL.  That is, they expect you to render/blit to a SDL Surface, and because of how they override the SDL functions via hooks, they break your opengl flow too.

### --Compiling SDL1.2--
So to enable the `aalib` and `caca` sdl renderers/features, you need to ensure some compile time flags are active. I'll walk you through it.  
```bash
sudo apt build-dep libsdl1.2debian
git clone https://github.com/libsdl-org/SDL-1.2 && cd SDL-1.2
mkdir build && cd build
../configure --enable-video-aalib --enable-video-caca
make
```

The output libraries are in a hidden folder within `build/build/.libs`.  If you want to install them you can run `sudo make install`.

For 32-bit builds, replace the configure line with :  
```bash
../configure CC="gcc -m32" CXX="g++ -m32" --host=i686-pc-linux-gnu --enable-video-aalib --enable-video-caca
```
and obviously get the :i386 package depedencies too.  

Now if you launch an SDL app with `SDL_VIDEODRIVER=aalib ./yourApp` , it will attempt to convert the rendered screen surface into ascii art
## :triangular_flag_on_post: ==VirtualGL==
So we need to get the pixel buffer that is on the gpu for this to ever work. We have learnt that SDL isn't going to give us what we want. Its down to us to get the pixels from the gpu.  

We could had written a software renderer, or found some SDL Wrapper that alters the calls to software, but thats either a lot of work or I just could not find such a thing existing.  And it would also decrease performance of the game. So...  

I researched the fastest way to get access to the screen pixels.  `glReadPixels()` is slow because it waits for the gpu to : `flush its pipeline aka finish processing all commands` , During these operations, the CPU must wait for the GPU to complete the data transfer.  So, the solution is : [https://www.khronos.org/opengl/wiki/Pixel_Buffer_Object](https://www.khronos.org/opengl/wiki/Pixel_Buffer_Object) , PBOs allow asynchronous pixel transfer operations. When a PBO is used, glReadPixels() can write pixel data to a buffer object without stalling the CPU. The CPU can then map this buffer to its address space and read the data at a later time, overlapping computation and data transfer.  

So, there we have it, we can write a runtime library that forces the app to render into an off-screen pbuffer instead of on-screen, use the libaa or libcaca and render to X11 or your window manager of choice.  But I didn't want to go down this route because it again involved too much complexity that I wasnt' ready to be involved in.  That is when I discovered virtualGL and learnt that it too is using pbuffers , if I could leverage virtualGL, I wouldn't need to write any code related to them, Super!.  Also, you can learn a lot by looking at virtualGL source code, if you did decide to implement pbuffers yourself.
## :triangular_flag_on_post: ==TurboVNC==
## :triangular_flag_on_post: ==TightVNC==


