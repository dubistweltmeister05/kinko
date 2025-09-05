[[bitbak]]

## What is it then?
A generic task execution engine that helps automate python file exec. Regardless of what the docs say, it is a build system to run a bunch of python files. Nothing else.

OpenEmbedded extends this functionality of bitbake to create an entire Linux Distro image. 

## How does it do it all? 

It is quite similar to GNU's Make. But there are a few subtle differences here and there - 
 - It executes the tasks according to metadata provided. The metadata can be see in a `.bb` file and a related recipe is seen in a `.bbappend` file, a configuration `.conf` file and underlying include (`.inc`) files, and in class (`.bbclass`) files.The metadata provides BitBake with instructions on what tasks to run and the dependencies between those tasks.
 - There are fetcher libraries to get code form the web, through a website or a SVC system - Github / Gitlab.
 - The instructions for each unit to be built (e.g. a piece of software) are known as “recipe” files and contain all the information about the unit (dependencies, source file locations, checksums, description and so on).

## Concepts

BitBake is a program written in the Python language. At the highest level, BitBake interprets metadata, decides what tasks are required to run, and executes those tasks. Similar to GNU Make, BitBake controls how software is built. GNU Make achieves its control through “makefiles”, while BitBake uses “recipes”. 

BitBake extends the capabilities of a simple tool like GNU Make by allowing for the definition of much more complex tasks, such as assembling entire embedded Linux distributions.

### Recipes
The bitbake recipes denote the following -
- Descriptive information about the package (author, homepage, license, and so on)
    
- The version of the recipe
- Existing dependencies (both build and runtime dependencies)
- Where the source code resides and how to fetch it
- Whether the source code requires any patches, where to find them, and how to apply them
- How to configure and compile the source code
- How to assemble the generated artifacts into one or more installable packages
- Where on the target machine to install the package or packages created

### Config Files
These are the `.conf` files that hold and define all the configuration variables that are used to define the parameters to be used in the build. The main ones are held in the `bitbake.conf` file. 

### Classes 
`.bbclass` files, these hold info that is shared with multiple metadata files. The source tree has a single metadata file called `base.bbclass`, in the `classes ` directory. This is included in ALL files and recipes. **This class contains definitions for standard basic tasks such as fetching, unpacking, configuring (empty by default), compiling (runs any Makefile present), installing (empty by default) and packaging (empty by default).**

While BitBake comes with just the one `base.bbclass` file in the `classes` directory, it supports class files also being installed in related directories `classes-global` and `classes-recipe` and will automatically search all three directories for a selected class file.

### Layers

Metadata with extra steps. Or as the professional term is - modularity. This is used to divide the metadata according to it's purpose, helps keep track of where's what and stuff. 

### Append Files

`.bbappend` files. They are used to extend or override any existing recipe file. A pain in the ass when it comes to naming tho. The recipe and the append file NEEDS to have the same name. 
E.g. -  `formfactor_0.0.bb` and `formfactor_0.0.bbappend`
Information in append files extends or overrides the information in the underlying, similarly-named recipe files. Also, we can use the `%` wildcard for matching the append file with multiple versions of a recipe. This works only in front of the append files. 

Yeah, that's about it. For the intro, that is. LMAO.
