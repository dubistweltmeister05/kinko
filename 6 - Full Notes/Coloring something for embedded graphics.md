[[Basic Concepts]]

The representation of a pixel is something complex yet elegant. It is not a combination of 3 different values - R, G, and B. It is a 3-dimentional vector that is rendered into a 2-DImenetional space, with a lot of complexity involved in the conversion that makes the rendering possible. 

Each pixel on the display screen is a combination of 3 - Red, Blue and Green. However, the representation of the final color that we see is not a bit part addition of 3 8-bit values. Rather, it is the additive combination of 3 8-bit vectors, each vector being the representation of a particular color of the final display image that is being shown. The 3-d space does not exist - it is a mathematical concept that helps the calculation, and the final projection of the vector is then rendered onto a 2-Dimentional space. 

### How Does Color Work

The color theory states that any and every color in this world can be formulated with a combination of Red, Blue, and Green. Think of these as the 3 vertices of a cube. The space formed by these (the cube) is called the color space. Every single color in this world can be represented as a point within the cube. Now, it is intuitive to associate size with a shape. What then, could be the size of this color cube? When defining true color, we have 8-bits per color, making it possible to have 16.7 MILLION points within the color cube. This corresponds to the total number of unique colors that can be produces. Does that mean, that this is also the number of colors that we can see? Not really. 

The eyes have cones that are responsive to certain wavelengths - L-cones respond to red, M-cones respond to green, and S-cones respond to blue wavelengths. This is why the RGB mapping feels the most responsive to us. The problem, is the non-linear scaling within  the RGB system. Simply put, doubling the channel length from 8-bits to 16-bits will not make the colors appear more bight, or make it pop out more; doing that will simply fuck up your computer with the number of calculations that it takes. Also, the eye is not uniformly sensitive to all light. Our imperfect vision is more sensitive to green than it is to red and blue, and since all colors in RGB are uniform, that does not translate well into actual vision. 

There are other color mapping, yes. But us embedded software engineers can only work with the mere Megabytes of RAM and flash that we are given, even with the externals that we connect. We are bound to constrain ourselves with the limits of the hardware that we have. However, that does not mean we cannot do amazing things with what we have!

### RGB Coding 
Each color within the RGB space ranges from 0 to 255. Absolute black is 0,0,0; absolute white would be 255,255,255. And then quite literally, everything else in between. But there seems to be one crucial thing that's missing. Commonly called as brightness, the engineering world calls it Alpha. As if display theory wasn't complex enough, we extend the 3-D space to a 4-D space. The 4th vector, alpha, includes information about the Opacity of each layer, with it's values ranging from 0 to 1. When something is completely opaque, that color is not blended with any other layer. The technique of mixing is called alpha blending. 
```
X_share = 100% - Y_alpha
Y_share = Y_alpha
new_X_rgb = X_rgb × X_share + Y_rgb × Y_share
```
That is the math behind alpha blending. 
### Color Depth
The number of bits that are used per pixel to store the color info within a framebuffer is called the color depth of the display being used. It depends on both the display that is available, and the software that does the rendering of of the final display image. One of the more commonly used standard is 24 BPP. Since a bit can be either on or off, 24 BPP = 2^24 colors, or 16.7 MILLION. 32 BPP would be the RGBA equivalent for 24BPP RGB, since you have to account for 8 bits of opacity too.

### Pixel Color Formats
There are defined ways of writing the bits of a framebuffer, called Pixel Color Formats. There things are nothing but a software convention, which defines which bits of a 32 bit integer are responsible for the RGB values. Some of them are - 

RGB888

Ina little endian system, the first byte(0-7)holds blue, the next(8-15) holds green and the last(16-23) holds red color values. When written to a `uint32_t` variable, the last byte (24-31) will be left reserved. 
``` 
uint32_t brightPurpleRGB888 = 255 << 16 | 0 << 8 | 255 << 0;
```

There are a bunch of others too, like RGB565, RGB222, and a few more like these, but I am not getting into that. Lol.

### Framebuffer Formats

Well ,the final piece of the puzzle - this is what defines the way an image is stored in something that gets read from for the display (The Framebuffer). The 24 bits format RGB888, and 32 bits format ARGB888, is often accessed using a byte pointer. When doing that it is necessary to understand that the pixels are stored in little endian order.

Take as an example the 32 bits color 0xFFFF7700 (alpha = 0xFF, red = 0xFF, green = 0x77, blue = 0x00). When the color is in a 32 bit variable or register, the value is 0xFFFF7700. When the color is stored in memory the bytes stored are { 0x00, 0x77, 0xFF, 0xFF }. This corresponds to the order BGRA.

Similarly, the 16 bits format, RGB565, is always accessed through a 16 bit pointer, so the byte order is not interesting, but it is swapped in memory.

For the 8 bits formats, e.g. ARGB2222, the color fits into a byte (alpha in the two highest bits), which is stored without change.

The smaller formats, GRAY4, GRAY2, and BW, can be stored in two orders. The low bits can be the leftmost pixel or the rightmost pixel. If the low bits are the leftmost we call this LSB-mode, otherwise it is MSB-mode.

### Visual Quality. 

I'll be honest - the level of memory that is needed for graphics is not really available for embedded settings. Even with an external flash, it is extremely tough to get a working display on there. 2^24 points on the display - there is no way in hell that an MCU can handle all those color space points, while doing other things that it is meant to do, like running an RTOS. We have to make a compromise somewhere, and God knows we are not compromising on any of the core embedded functionalities. So, what does get cut? The color space. 

Well, here is the thing - I am greedy man, and I want to have it all. Or at least, I want it to look like I have it all. So when I cut my Color Depth from 24BPP to 16BPP, I want to have a way of faking the image quality, so that it does not look horrendous. That way, is called dithering. The point is to add a bit of noise while cutting the color space, so that the image looks of higher quality. Now, as a photographer, it went against everything that I learnt about pictures. But then, I read about it. Let me explain. 

The science behind human visual sensing is understood to be performed by mixing individual pixels into a cohesive picture. By that logic, placing individual noise/grains in a lower-quality or distorted image can give the impression that the quality is better than what it actually is. It looks to the eye like the image being viewed is much better. The dithering process is done by breaking down the present pixels into fragments and filling them with varying patterns of colored pixels. As these small groups are viewed from afar, they blend together to create new tones and hues not present in the original image.

Un-intuitive, but EXTREMELY useful, lol.



