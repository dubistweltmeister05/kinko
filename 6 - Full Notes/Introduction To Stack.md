2025-01-13 23:28

Tags: [[3 - Tags/Stack]]

A stack is a LIFO data structure. It accepts the items from from one end and they can be removed in the order that they were entered in. 

There's 2 ways to implement this, via an array or via a Linked List. It would be safe to assume that is the LL implementation of the stack, the Linked List would be Singly Linked. 

### USES

The use cases of a stack would be - obviously, the emulation of embedded memory, among other things. Here are a few more -
 - **Function calls** - A stack can be used to track the way functions are called, thus, in the case of an interrupt occurring in the execution of the code, the processor can return to the normal flow of code once the ISR has been serviced. 
 - **Recursion** - They are used to keep track of the local variables, allowing the program to keep track of the current state of the recursive execution of the function. 
 - **Expression Evaluation** - Prefix and postfix notation of any expression are implemented using stacks, which make the computation of the function inexpensive when compared to the conventional method of evaluation. 
 - **Syntax Parsing** - The validity of the syntax being written for a programming language is often checked using a stack. 
 - **Memory Management** - Come on son. 

### ADVANTAGES
 - **Simplicity** - Stacks are a simple and easy-to-understand data structure, making them suitable for a wide range of applications
 - **Efficiency** - Push and pop operations on a stack can be performed in constant time (O(1)), providing efficient access to data
 - **LIFO FORM** -  Stacks follow the LIFO principle, ensuring that the last element added to the stack is the first one removed. This behavior is useful in many scenarios, such as function calls and expression evaluation.
 - **Limited Memory Usage** - Stacks only need to store the elements that have been pushed onto them, making them memory-efficient compared to other data structures.

### DISADVANTAGES
 - **Limited Access** - Elements in a stack can only be accessed from the top, making it difficult to retrieve or modify elements in the middle of the stack.  
 - **Overflow Potential** - If more elements are pushed onto a stack than it can hold, an overflow error will occur, resulting in a loss of data. 
 - **Unsuitable for Random Access** - Stacks do not allow for random access to elements, making them unsuitable for applications where elements need to be accessed in a specific order.
 - **Limited Capacity** - Stacks have a fixed capacity, which can be a limitation if the number of elements that need to be stored is unknown or highly variable.

# Reference

[Implementing Stack Using Array in Data Structures - YouTube](https://www.youtube.com/watch?v=MD3mFgFwqBE&list=PLu0W_9lII9ahIappRPN0MCAgtOu3lQjQi&index=23)

[Applications, Advantages and Disadvantages of Stack - GeeksforGeeks](https://www.geeksforgeeks.org/applications-advantages-and-disadvantages-of-stack/?ref=lbp)