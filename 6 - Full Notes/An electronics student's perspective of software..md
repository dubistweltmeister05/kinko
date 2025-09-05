[[Twitter Posting]]

Have you ever wondered, how does the code that we write actually run? 

And no, I am not talking about the compilation pipeline, but rather the physical implementation of the very processors that our code eventually interacts with? 

Think about it. Let's take the example of a micro-controller pin being set to high. Does it not amaze you to think about the fact that a line of text (code) can eventually lead to shifting of voltage levels and drive a real, physical signal? 

I know this is way beyond software engineering, and the usual HTML, CSS and JS crowd is totally lost at this point, but I am speaking directly to the engineer and true enthusiast within you. Have you ever wondered?

Thanks to my electronics and embedded systems background, I know that the line of code that gets translated to machine code, that is zeroes and ones, eventually gets written to registers, which are nothing but a collection of latches/flip-flops. These ones and zeroes represent physical charges that govern the switching of transistor or CMOS based switches that then control the pins that are used as GPIO outputs. 

Or, talking about it in the opposite direction, the pin receives an external charge when configured as a GPIO input pin and the value of the charge held by this pin is then used to drive an input switch, which leads to a data register holding the corresponding value of zero or one, which can be read by the code that you just wrote, and your system moves on. 

As a software engineering student in India, I know you don't give a fuck about any of this, because you were told that this doesn't matter, or that this doesn't get you paid, or make you look cool. So, you are oblivious to the inner-workings of the very basis of the life that you lead right this very moment.

But think about it for a second. Wonder about it. Regardless of where you are or what you do in your life, app dev, web dev, AI/ML, Cybersec, I don't care. Think about "How does the code that I write daily actually work". And I promise you, you'll never be the same.
