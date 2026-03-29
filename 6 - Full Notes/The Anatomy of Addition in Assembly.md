[[Twitter Posting]]

```

SECTION .data  
       extern printf  
       global main  
  
fmt:  
       db "%d", 10, 0  
  
SECTION .text  
  
main :  
       mov eax, 69  
       mov ebx, 420  
       add eax,ebx  
  
       push eax  
       push fmt  
       call printf  
       add esp, 8  
  
       mov eax,0  
       ret
```

What you see here is an assembly program that takes 2 numbers, adds them, and writes to standard output (stdout), which typically maps to the terminal. For those of you who are curious enough to see if this works, please do so on a Linux machine. It will not work on Windows. Now, while I was looking up online for methods to do a simple numeric addition in assembly - because what else are you supposed to do on a Sunday morning - I came across this stack overflow post (https://stackoverflow.com/a/51433261). So I tried to run it, make sense of the code that I saw, and decided to make an article about it. 

## The sections of an asm code

There are 3 prominent ones to be precise, the Data section, the text section and the bss section. 

The data section contains all initialized variables that shall be used in the code. It holds constants that shall not change during the runtime of the code. You can declare various constant values, file names, or buffer size, etc., in this section.

The text section holds the actual code, the instructions that you are intending to execute with the program that you are writing for. 

The bss - block separated by symbol - section holds uninitialized data that is zero-initialized at runtime. 

## The Sections within THIS code to be specific.


### Le Data section
The data section of this code contains 2 references, one for `printf` and the other for the main. Here, `printf` is treated as an external function that can be called by the code. The function's definition, and the correct linking is done when we try to compile the code with `gcc`, which takes care of attaching the correct library - `glibc` that implements `printf` with the `asm` code that we wrote. Keep in mind, we are dynamically linking the `glibc` with our `asm`, so within the code - we do not need to include all that additional overhead, we simply need to have a reference to the `printf` function, and something that will tell the assembler that it is implemented somewhere else - that something being the `extern` keyword.

The `global main` line, simply tells the assembler that this is a symbol that is defined down the line, somewhere, and we can guarantee it's existence. If the compiler does not find a reference to this, well, it throws up an error. Think of the global declaration of main, as, well, exactly that - a declaration of the function, and a promise to the compiler. 

Next, is the fmt label. This is simply pointing to a location in the memory that holds the string formatting for the eventual result that is to be printed via the code. `db "%d", 10, 0` translates to define byte = "%d\n\0" (10 represents \n and 0 represents \0  in `asm`).

### Le Text section
The text section is quite straightforward to start with. The first 3 instructions are quite self explanatory, we move the number 69 into the register `eax`, 420 to the register `ebx`, and add these two ,storing the result in `eax` itself. Then, in order to print this to the terminal, we need to call the `printf` function (obviously). However, we cannot just `printf(some_arg1, some_arg2);` our way out of this, oh no. We have to put the arguments onto the stack, and then call `printf` so that it can fetch the args for itself, and then do it's job. Now, as we (hopefully) know, the stack in computer memory grows downwards. What is pushed last, is read first. Thus, we need to push the arguments in reverse order, so that `printf` can read em in the intended order. 

Finally, we need to clean the stack as well. Because of course, `asm` does NOT help with that. The byte size of the arguments is 4 bytes each, so we need to clean 8 bytes of data. This is done by moving the stack pointer (esp) by 8 bytes, so that the first 8 bytes are dropped, essentially cleaning the 2 args that we pushed there. And finally, we clear the `eax` register to indicate that everything is, well, done. 

And then we use the `ret` instruction, which returns control to the C runtime (`_start`), which then exits the program. And then, the system is supposed to go back to doing whatever the hell it was doing in the first place. 


## Conclusion
Lol, what did you expect here? We are adding 2 digits after all. But, it is fascinating to dive into the specifics, tear down the abstraction, and have a look at what truly goes on underneath the hood, to appreciate the way things just, WORK in our era, isn't it. Pretty cool stuff, no?

