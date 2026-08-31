[[RhyGen]]

The below is a note on the interaction of app and PAL, which should serve as a guide on application development using the PAL, or for integrating the PAL with any codebase.  
#  Object-Creation functions
We begin with the app_hw.h function, where we define a master structure that contains all the pal objects that we have used in the project. 

An object of this structure is initialized in the main.c file, and the used PAL objects are then initialized in the main function, via `pal_XXX_create_stm32();` function.