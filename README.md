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

I did not realise this independence initially, and only saw their usage through `SDL` via the `SDL_VIDEODRIVER=aalib` and `SDL_VIDEODRIVER=caca` avenues.  So for the time-being we will just assume this route, one plus side of using the SDL route is that it doesn't involve recompilation of the target program, although with 3d games without software render support, its irrelevent because it won't work.  

Why it does not work with 3d renderered opengl games is because they send vertices to the gpu and the gpu will render the pixels within its-self.  Thus, aalib would need to `pull` pixels from the gpu.  This is possible, but involves much more interplay between parts of the system so is harder to implement. More on this later. 
## :triangular_flag_on_post: ==SDL==
libaa and libcaca are features within SDL 1.2, not SDL 2.  

You can see here how aalib is used within SDL.  
[https://github.com/libsdl-org/SDL-1.2/tree/main/src/video/aalib](https://github.com/libsdl-org/SDL-1.2/tree/main/src/video/aalib)  
Essentially by defining a video renderer in SDL, it is able to override what the Key SDL API functions perform.  So, eg. upon SDL_Flip(), it will internally call `aa_flush` : [here](https://github.com/libsdl-org/SDL-1.2/blob/0569bc61fe2ad18c8adbba72225d546336242999/src/video/aalib/SDL_aavideo.c#L323C2-L323C23)

So with this setup, you have less fine grain control over the ascii libraries, and have to make-do with how they are arranged within SDL.  That is, they expect you to render/blit to a SDL Surface, and because of how they override the SDL functions via hooks, they break your opengl flow too.

### --Compiling SDL1.2--
So to enable the `aalib` and `caca` sdl renderers/features, you need to ensure some compile time flags are active. I'll walk you through it.


## :triangular_flag_on_post: ==VirtualGL==
## :triangular_flag_on_post: ==TurboVNC==
## :triangular_flag_on_post: ==TightVNC==


