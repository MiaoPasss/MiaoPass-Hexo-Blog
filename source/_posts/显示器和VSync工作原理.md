---
title: 显示器和VSync工作原理
date: 2025-02-11 14:31:30
categories: 图形学笔记
tags: 
- 游戏画面设置
- 渲染
---

## CRT显示器

显示器类别比较：https://blog.csdn.net/m0_69378371/article/details/145129033

根据显示技术和用途的不同，显示器可以分为多种类型，主要包括：
-   **阴极射线管显示器（CRT）**
-   **液晶显示器（LCD）**
-   **发光二极管显示器（LED）**
-   **等离子显示器（PDP）**
-   **有机发光二极管显示器（OLED）**
-   **量子点显示器（QLED）**

<br>

## VSync 工作原理

What is VSync? VSync stands for Vertical Synchronization. The basic idea is that synchronizes your FPS with your monitor's refresh rate. The purpose is to eliminate something called "tearing". I will describe all these things here.  
  
Every CRT monitor has a refresh rate. It's specified in Hz (Hertz, cycles per second). It is the number of times the monitor updates the display per second. Different monitors support different refresh rates at different resolutions. They range from 60Hz at the low end up to 100Hz and higher. Note that this isn't your FPS as your games report it. If your monitor is set at a specific refresh rate, it always updates the screen at that rate, even if nothing on it is changing. On an LCD, things work differently. Pixels on an LCD stay lit until they are told to change; they don't have to be refreshed. However, because of how VGA (and DVI) works, the LCD must still poll the video card at a certain rate for new frames. This is why LCD's still have a "refresh rate" even though they don't actually have to refresh.  
  
I think everyone here understands FPS. It's how many frames the video card can draw per second. Higher is obviously better. However, during a fast paced game, your FPS rarely stays the same all the time. It moves around as the complexity of the image the video card has to draw changes based on what you are seeing. This is where tearing comes in.  
  
Tearing is a phenomenon that gives a disjointed image. The idea is as if you took a photograph of something, then rotated your vew maybe just 1 degree to the left and took a photograph of that, then cut the two pictures in half and taped the top half of one to the bottom half of the other. The images would be similar but there would be a notable difference in the top half from the bottom half. This is what is called tearing on a visual display. It doesn't always have to be cut right in the middle. It can be near the top or the bottom and the separation point can actually move up or down the screen, or seem to jump back and forth between two points.  
  
Why does this happen? Lets take a specific example. Let's say your monitor is set to a refresh rate of 75Hz. You're playing your favorite game and you're getting 100FPS right now. That means that the mointor is updating itself 75 times per second, but the video card is updating the display 100 times per second, that's 33% faster than the mointor. So that means in the time between screen updates, the video card has drawn one frame and a third of another one. That third of the next frame will overwrite the top third of the previous frame and then get drawn on the screen. The video card then finishes the last 2 thirds of that frame, and renders the next 2 thirds of the next frame and then the screen updates again. As you can see this would cause this tearing effect as 2 out of every 3 times the screen updates, either the top third or bottom third is disjointed from the rest of the display. This won't really be noticeable if what is on the screen isn't changing much, but if you're looking around quickly or what not this effect will be very apparant.  
  
Now this is where the common misconception comes in. Some people think that the solution to this problem is to simply create an FPS cap equal to the refresh rate. So long as the video card doesn't go faster than 75 FPS, everything is fine, right? Wrong.  
  
Before I explain why, let me talk about double-buffering. Double-buffering is a technique that mitigates the tearing problem somewhat, but not entirely. Basically you have a frame buffer and a back buffer. Whenever the monitor grabs a frame to refresh with, it pulls it from the frame buffer. The video card draws new frames in the back buffer, then copies it to the frame buffer when it's done. However the copy operation still takes time, so if the monitor refreshes in the middle of the copy operation, it will still have a torn image.  
  
VSync solves this problem by creating a rule that says the back buffer can't copy to the frame buffer until right after the monitor refreshes. With a framerate higher than the refresh rate, this is fine. The back buffer is filled with a frame, the system waits, and after the refresh, the back buffer is copied to the frame buffer and a new frame is drawn in the back buffer, effectively capping your framerate at the refresh rate.  
  
That's all well and good, but now let's look at a different example. Let's say you're playing the sequel to your favorite game, which has better graphics. You're at 75Hz refresh rate still, but now you're only getting 50FPS, 33% slower than the refresh rate. That means every time the monitor updates the screen, the video card draws 2/3 of the next frame. So lets track how this works. The monitor just refreshed, and frame 1 is copied into the frame buffer. 2/3 of frame 2 gets drawn in the back buffer, and the monitor refreshes again. It grabs frame 1 from the frame buffer for the first time. Now the video card finishes the last third of frame 2, but it has to wait, because it can't update until right after a refresh. The monitor refreshes, grabbing frame 1 the second time, and frame 2 is put in the frame buffer. The video card draws 2/3 of frame 3 in the back buffer, and a refresh happens, grabbing frame 2 for the first time. The last third of frame 3 is draw, and again we must wait for the refresh, and when it happens, frame 2 is grabbed for the second time, and frame 3 is copied in. We went through 4 refresh cycles but only 2 frames were drawn. At a refresh rate of 75Hz, that means we'll see 37.5FPS. That's noticeably less than 50FPS which the video card is capable of. This happens because the video card is forced to waste time after finishing a frame in the back buffer as it can't copy it out and it has nowhere else to draw frames.  
  
Essentially this means that with double-buffered VSync, the framerate can only be equal to a discrete set of values equal to Refresh / N where N is some positive integer. That means if you're talking about 60Hz refresh rate, the only framerates you can get are 60, 30, 20, 15, 12, 10, etc etc. You can see the big gap between 60 and 30 there. Any framerate between 60 and 30 your video card would normally put out would get dropped to 30.  
  
Now maybe you can see why people loathe it. Let's go back to the original example. You're playing your favorite game at 75Hz refresh and 100FPS. You turn VSync on, and the game limits you to 75FPS. No problem, right? Fixed the tearing issue, it looks better. You get to an area that's particularly graphically intensive, an area that would drop your FPS down to about 60 without VSync. Now your card cannot do the 75FPS it was doing before, and since VSync is on, it has to do the next highest one on the list, which is 37.5FPS. So now your game which was running at 75FPS just halved it's framerate to 37.5 instantly. Whether or not you find 37.5FPS smooth doesn't change the fact that the framerate just cut in half suddenly, which you would notice. This is what people hate about it.  
  
If you're playing a game that has a framerate that routinely stays above your refresh rate, then VSync will generally be a good thing. However if it's a game that moves above and below it, then VSync can become annoying. Even worse, if the game plays at an FPS that is just below the refresh rate (say you get 65FPS most of the time on a refresh rate of 75Hz), the video card will have to settle for putting out much less FPS than it could (37.5FPS in that instance). This second example is where the percieved drop in performance comes in. It looks like VSync just killed your framerate. It did, technically, but it isn't because it's a graphically intensive operation. It's simply the way it works.  
  
All hope is not lost however. There is a technique called triple-buffering that solves this VSync problem. Lets go back to our 50FPS, 75Hz example. Frame 1 is in the frame buffer, and 2/3 of frame 2 are drawn in the back buffer. The refresh happens and frame 1 is grabbed for the first time. The last third of frame 2 are drawn in the back buffer, and the first third of frame 3 is drawn in the second back buffer (hence the term triple-buffering). The refresh happens, frame 1 is grabbed for the second time, and frame 2 is copied into the frame buffer and the first part of frame 3 into the back buffer. The last 2/3 of frame 3 are drawn in the back buffer, the refresh happens, frame 2 is grabbed for the first time, and frame 3 is copied to the frame buffer. The process starts over. This time we still got 2 frames, but in only 3 refresh cycles. That's 2/3 of the refresh rate, which is 50FPS, exactly what we would have gotten without it. Triple-buffering essentially gives the video card someplace to keep doing work while it waits to transfer the back buffer to the frame buffer, so it doesn't have to waste time. Unfortunately, triple-buffering isn't available in every game, and in fact it isn't too common. It also can cost a little performance to utilize, as it requires extra VRAM for the buffers, and time spent copying all of them around. However, triple-buffered VSync really is the key to the best experience as you eliminate tearing without the downsides of normal VSync (unless you consider the fact that your FPS is capped a downside... which is silly because you can't see an FPS higher than your refresh anyway).  
  
I hope this was informative, and will help people understand the intracacies of VSync (and hopefully curb the "VSync, yes or no?" debates!). Generally, if triple buffering isn't available, you have to decide whether the discrete framerate limitations of VSync and the issues that can cause are worth the visual improvement of the elimination of tearing. It's a personal preference, and it's entirely up to you.