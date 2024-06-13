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

## :triangular_flag_on_post: XServer Primer
DISPLAY= _HOSTNAME_:_SERVER_NUM_ID_._SCREEN_MONITOR_BUFFER_  
`man 7 X`  
Each X11 server running on the same machine needs to have a different display number

A display just refers to some X server somewhere.

 "screens" is referring the different virtual monitors (framebuffers) of the X server.
## :triangular_flag_on_post: ==VirtualGL==
TLDR: The X11 Transport with TurboVNC is the most popular option.  
So we need to get the pixel buffer that is on the gpu for this to ever work. We have learnt that SDL isn't going to give us what we want. Its down to us to get the pixels from the gpu.  

We could had written a software renderer, or found some SDL Wrapper that alters the calls to software, but thats either a lot of work or I just could not find such a thing existing.  And it would also decrease performance of the game. So...  

I researched the fastest way to get access to the screen pixels.  `glReadPixels()` is slow because it waits for the gpu to : `flush its pipeline aka finish processing all commands` , During these operations, the CPU must wait for the GPU to complete the data transfer.  So, the solution is : [https://www.khronos.org/opengl/wiki/Pixel_Buffer_Object](https://www.khronos.org/opengl/wiki/Pixel_Buffer_Object) , PBOs allow asynchronous pixel transfer operations. When a PBO is used, glReadPixels() can write pixel data to a buffer object without stalling the CPU. The CPU can then map this buffer to its address space and read the data at a later time, overlapping computation and data transfer.  

So, there we have it, we can write a runtime library that forces the app to render into an off-screen pbuffer instead of on-screen, use the libaa or libcaca and render to X11 or your window manager of choice.  But I didn't want to go down this route because it again involved too much complexity that I wasnt' ready to be involved in.  That is when I discovered virtualGL and learnt that it too is using pbuffers , if I could leverage virtualGL, I wouldn't need to write any code related to them, Super!.  Also, you can learn a lot by looking at virtualGL source code, if you did decide to implement pbuffers yourself.  

Another option would be to disable READBACK in virtualGL and just convert to ascii there.  
Or compile a custom dummy xvfb/vncserver that does nothing but convert to ascii text.  
Currently my solution is modifying a tight vnc viewer, which is further down the chain.

[virtualGL Docs](https://rawcdn.githack.com/VirtualGL/virtualgl/main/doc/index.html)  
[virtualGL source](https://github.com/VirtualGL/virtualgl)

### --VirtuaGL Primer--
VirtualGL is a library that is force-loaded using `LD_PRELOAD` at application runtime, to render your opengl app into pbuffers instead of on-screen ones.   

Once it has the pbuffers it will try to share them to another system.  There are 4 ways it can do this:  
* VGL Image Transport (Formerly “Direct Mode”)
* X11 Image Transport (Formerly “Raw Mode”)
* XV Transport
* Custom Plugin Transport

`VGL_COMPRESS` environment variable controls which transport is used.  
or the `-c` option passed to vglrun  
Every X Server represents a set of screens and input devices, if a Client renders into an Server, the screens attached to that server will show the client. 

#### VGL Image Transport. (JPEG, RGB, YUB) compression
Here, the X server can live across a network and your game can render to it via the `DISPLAY` environment variable.  The vglclient program will exist on the non-gpu host and act as a receiver of the image frames.  The VirtualGL Client is responsible for decompressing the images and drawing the pixels into the appropriate X window.  So it injects the 3d frames into window that was created over the network.  Notice that it talks over x11 protocol via networks for events.  Its a home-brewed implementation of sending 3d accelerated images over the network with compression.  By injecting like this, each `app` on the server, will generate its own unique window on the client.  This isn't true for the XProxy(vnc) method when 1 vncserver is showing all windows within one window.  You'd have to create multiple vncservers's but this seems unpractical.
#### X11 Image Transport. no compression
The x11 game/client will share its images through the X11 supported ways, like Shared Memory/Unix Sockets/Tcp Socket.  
functions like XFlush and XSync are used to control when they are sent to the server.  [See Here](https://github.com/VirtualGL/virtualgl/blob/a51eacf49fa5bd017ce0e312b923ae877b2271fd/util/fbx.c#L620), [AndHere](https://github.com/VirtualGL/virtualgl/blob/a51eacf49fa5bd017ce0e312b923ae877b2271fd/util/fbx.c#L511)  
Handshake: When an X client connects to an X server, they negotiate capabilities. If both support MIT-SHM, it's usually the preferred method.  
[SharedMemory](https://github.com/VirtualGL/virtualgl/blob/a51eacf49fa5bd017ce0e312b923ae877b2271fd/util/fbx.c#L326)  
`XShmAttach, shmctl, XShmCreatePixmap` ..  `VGL_USEXSHM=0 to disable`  
getFrame() calls f->init(), which uses shm.  
Obviously SHM has to be a local xserver or xproxy.  
It seems this method can support network too, I guess without compression it is too slow for network? The vnc XProxy is giving the nice compression.  
Yes I think the default X11 protocol do not support compression, that is why VGL_Transport exists, if not using a vnc server.
#### XV Transport. transcode YUV420P
The XV Transport is a special flavor of the X11 Transport that encodes rendered frames as YUV420P and draws them directly to the 2D X server using the X Video extension. This is mainly useful in conjunction with X proxies that support the X Video extension. The idea is that, if the X proxy is going to have to transcode the frame into YUV anyhow, VirtualGL may be faster at doing this, since it has a SIMD-accelerated YUV encoder.
#### Custom Plugin Transport
You provide your own. (This could be useful for this project.)

[https://github.com/VirtualGL/virtualgl/blob/a51eacf49fa5bd017ce0e312b923ae877b2271fd/server/VirtualWin.cpp#L336](https://github.com/VirtualGL/virtualgl/blob/a51eacf49fa5bd017ce0e312b923ae877b2271fd/server/VirtualWin.cpp#L336)  
This code above reveals that the X11 method is intended for proxy usage, where no compression occurs on the server, that is why it was named `raw`.
## :triangular_flag_on_post: ==TurboVNC==
By the same creators as VirtualGL, this supports SharedMemory and acts as receiver for the X11 Image Transport method.  Then this will compress the frames to the vncviewers, allowing for collaboration.
## :triangular_flag_on_post: ==TightVNC==
This is an extremely ancient lightweight vncviewer, I like it because its not java based, unlike the turboVNC viewer. So on computers with limitted cpu budget, its handy.
## :triangular_flag_on_post: ==My Thoughts on Optimal Solution==
I like the non vnc-viewer edit solutions the best because of the issue where vncviewers do not receive cursor updates correctly from the client App.  Like eg. in FirstPersonShooter games they center the mouse each frame.

## :triangular_flag_on_post: ==The vnc viewer workaround for mouse glitch==
