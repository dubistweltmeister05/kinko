[[Blog Topics]]
[[YOCTO]]


The way this feels weird but familiar to me at the same damn time man!

Hey there y’all! Hope everyone is doing well. Yes, I remembered my password. After about 3 months and 4 blogs teased, this is finally the one that makes out of the drafts. Funnily enough, this is also the latest one to make it out here, so I guess y’all can really say, I got stories stacked in the vault! (Get it? _Stacked_)

Right, with my weak attempts at humor out of the way, let’s talk about the reason that you clicked on this link.

Silly me, where the fuck are my manners? Lemme introduce myself, lads and lasses (yes, this is a real word. Don’t believe me? Look it up then!).  
My name is Kshitij Neeraj Vaze, _(NO LONGER A COLLEGE STUDENT BASED IN PUNE!!, the old ones know ;-)_ ) and I work as a Linux Kernel developer at a hardware startup, based in Surat, Gujarat.

It’s been a week over there for me since the writing of this blog, and lemme tell you something, yeah? I have learnt more here in a week that I did for the last 6 months of my internship. I SWEAR I AIN’T LYING Y’ALL.

Like, ya boi went from copy pasting code from the codebase into an excel sheet and making documents _(requirement verification they called it LMFAO)_, to now Compiling and Configuring Kernels and bootloaders and whatnot for literal processors!

Like, the first day that I came over here, I was given a little custom Dual-Core Arm CORTEX-A7 board that I was supposed to bring back to life. This process basically means that I am give a dev board, which does abso-fucking-lutely NOTHING when powered up, and my job is to make it work. Like, fully functional — I gotta get everything on it up and running. An HDMI, an Ethernet, USB ports, everything! From Scratch!

So, how do I do that (Bear with me, I swear this relates with YOCTO)? Well, to break it down, here is what usually happens when a processor is powered up.

**THE BOOTLOADER :—**  
This is a piece of software that acts like an alarm clock for the processor. Think of it this way — y’all lazy asses would be SLEEP till 10 AM every morning if it wasn’t for an alarm clock, or for the lucky ones, their mothers waking them up, right (and don’t you dare lie to me)?

Well, the processor is kind of similar — it gets power, but it won’t do anything meaningful until it starts executing instructions from a pre-defined location (usually some internal ROM or boot ROM on the chip).That initial code usually pulls in the real bootloader (like U-Boot), which is the one responsible for the heavy lifting — hardware self-tests, memory initialization, setting up clocks and stack pointers, configuring early serial output (so you get some logs on your screen), and more.

And yeah, the bootloader also loads the Linux kernel into RAM and then jumps to the kernel’s entry point — like a relay racer, handing over the flow of the booting process to —

**THE KERNEL: —**

This piece of engineering beauty is what handles the processor and its behavior. See, there’s a ton of stuff on a dev board that the processor has absolutely no clue about by itself, yeah? Like all your comms interfaces, the GPIOs, and stuff that gets added later — Ethernet, HDMI, Wi-Fi, Bluetooth, and all that.

That’s where the kernel comes in. It acts as the big brother of the processor, sitting in between all these I/O devices and the actual hardware, ensuring **safe and controlled access.** The kernel uses drivers to manage each device’s access to memory, schedule tasks, manage permissions, and abstract away the messy bits so you don’t end up killing the system by poking into the wrong memory(_looking directly at you, you null-pointer-dereferencing programmer_). Once it’s done with everything that it needs to do, the kernel mounts…

**THE ROOT FILESYSTEM: —**

Now this one — this is the biggest piece of software out of all three pieces. This is where all the stuff lives that the user actually gets to interact with. Think of it like a well-organized apartment complex. Each room (a directory) has its own specific purpose — one for binaries, one for configs, one for drivers, one for logs, etc.

But here’s the thing **—** the kernel doesn’t magically know where to find this root filesystem**.** That info usually gets passed down from the bootloader using bootargs (`root=/dev/mmcblk0p2` or whatever the hell is the developer, (that’s me) wants to do). Once the kernel gets that hint**, it mounts that location** and hands over control to the init system (like systemd or init). And now your board is officially “alive.”

From this point on, you’ve basically got a working Linux system, capable of running programs, managing hardware, and letting users do whatever they want to do with it!

Sounds pretty straightfrward, right ? BTW, the official term for this thing is called “Board Bring-Up”. Find these three, arrange them in a particular order, load then and BOOM! Bob’s your uncle lads!

Yeah, about that…..

![](https://miro.medium.com/v2/resize:fit:700/1*_F8Obvi6YAcqbESPNdw47g.png)

Lemme tell you something brev. IT AIN’T.

It isn’t always that easy, you see. There’re a thousand things that can go wrong in this entire process. You can compile the bootloader in the wrong configuration than the one which is intended for your board, you compile the kernel with the wrong defconfig, the RFS that you got just happens to not have a display server, so there’s no HDMI output for your dumbass, or you could mount the filesystem in the read-only mode, and so on and so forth (_I have done every single one of these personally. Within like, the first 5 days_)

Wouldn’t it be just amazing if you could just…idk, write a file that says what you want, and press enter, and BOOM — you get everything that you want?!?!

Well, allow me to re-introduce myself, my name is HOV — I mean YOCTO (_if you don’t get the reference, listen to_ [_THIS_](https://youtu.be/9XEWVM1IhGY?si=ian7xAnBTkMzqGXK) _song_).

It does exactly what I said above.

So, lemme drop the bullshit and the jokes, and let’s be a bit more technical for a while.

The YOCTO project, is NOT a distro of Linux. Instead, it helps you make your own distro. You can specify exactly what you want, which bootloader, which version of the kernel, what type of rootfs, which utilities, which packages, which modules and the whole thing, in a config file, and through that file, the YOCTO project will (_take 5 fucking hours on the first build_) give you exactly what you want. A bootable disk image that just works. Out of the box.

Matter of fact, the YOCTO project even let’s you choose what you want as an output, is it just the u-boot binaries, the kernel images, or the whole thing.

The cool thing is that it maintains everything that we are doing in a cache directory, and upon subsequent builds, it will compare the configuration requested by the user with the cache data. If there are some matching elements, it will skip building those artefacts (things, but a fancier word, lads), and thus, drastically save our time.

The output files can also be configured to be dumped to an upstream path, like a server for instance, from where network booting and fleet management ([Balena Cloud](https://www.balena.io/cloud) says Hello!) is also possible, and getting popular these days.

The YOCTO project is made up of mainly 3 components — POKY, BitBake and Metadata. Let’s understand what these are, albeit in a brief manner —

## POKY

Poky serves as the reference distribution of the Yocto Project. It is not just a sample distro, but rather a collection that combines the BitBake build engine, core metadata, and example configurations into a single cohesive package. Developers typically start with Poky as the baseline for their custom Linux systems. It comes with a set of predefined layers, configuration files (like local.conf and bblayers.conf), and commonly used recipes.

The POKY layer is the basis of all further builds. It is a canvas, if you will, that allows you to select the paints and the brushes that you’d like to paint a disk image of your heart’s (or your client’s heart’s) desire! It is designed to be extended or customized — developers can simply add their own layers or adjust configurations to build for different hardware platforms or application needs.

## BitBake

BitBake is the build engine that drives the Yocto Project. Inspired by the make build tool, BitBake takes it several levels further by supporting advanced dependency resolution, task execution order, cross-compilation toolchains, and parallel builds.

When a user issues a BitBake command (for example, bitbake core-image-minimal), it reads the configuration files and recipes, calculates all the dependencies, and executes a series of tasks (called recipes) — such as fetching source code, unpacking archives, applying patches, compiling binaries, installing outputs, and packaging the final artifacts.

It uses a task-based model, where each recipe defines a set of tasks (e.g., do_fetch, do_compile, do_install) and dependencies between them. BitBake’s ability to manage thousands of components across multiple layers, architectures, and configurations is what makes it so integral to Yocto’s flexibility.

## MetaData

This is the instruction set that tells BitBake exactly what to do. It encompasses all the recipes (.bb files), configuration files (.conf), class files (.bbclass), and layer definitions that define how each software component in the distribution should be built and integrated. A recipe typically defines where to fetch the source code, what patches to apply, how to compile it, and what to install. Class files contain shared logic — for example, common build systems like autotools or cmake can be abstracted into classes that recipes can inherit from.

Configuration files determine machine settings, distribution policies, and global variables that influence how the build behaves. Yocto uses a layered structure for organizing metadata — each layer can represent a board support package (BSP), a software stack, or a set of applications. Think of layers like modular DLCs for your Linux build — want to support a new board? Add the vendor’s BSP layer. Need Qt or GNOME? Add the UI layer. It’s like stacking LEGO pieces — clean, isolated, but when combined, magic.

This modularity allows for scalability and easy maintenance: hardware vendors can maintain BSP layers, while application developers focus on higher layers without touching the core system logic.

Also, there’s a teeny tiny problem with all this. The FIRST BUILD, takes 6 hours, and will brick your machine multiple times, unless you have like 32 cores on your machine (that’s what it took for me to build this bastard.)

**_The reasoning behind this too trivial for the author, and is left as as exercise for the reader._**

🙃

I hope that this does make things a lot clearer about what the YOCTO Project is and what it can be used to do, lads! I won’t say what have I been doing with it, that’s gon be a secret. But yeah, feels good to be back, writing stuff for the sake of it, you know. Will I keep these coming? IDK.  
No broken promises, if there isn’t one made in the first place!

As always, my DMs, comments, and inbox is open for criticism, debates and doubts.

See you soon!!