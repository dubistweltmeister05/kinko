{{date}} {{time}}

Tags: [[02 From Source To Binary]]

In this section, we are going to talk about the preprocessor component, which drives the preprocessing step, in greater depth.


Basically, we get a single chunk of code by combining all the headers and the course files. This piece of code is called as the translation unit, which can be dumped by using the "-E" flag in gcc. (SIDE NOTE: IT'S VERY COOL)

During cross compilation, examining this can be a useful step while figuring out some weird bugs.

# Reference

