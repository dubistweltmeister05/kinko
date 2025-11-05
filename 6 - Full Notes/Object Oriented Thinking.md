[[06 OOP and Encapsulation]]

Object oriented thinking is a way of understanding and breaking down the things that we see around us. A vase on a table is easily distinguished between, as we can see a boundary between the table and the vase; we know the difference between the color of the vase and the table; the difference between their shapes, etc.

This observation tells us about the the way we see this world, we create a reflection of the things we see in our minds. In the same vein of thought, we write software that models the things that we can physically see and feel, in computer games, 3-D models, etc.

OOP is about bringing object-oriented thinking to software design and development. Object-oriented thinking is our default way of processing our surroundings, and that's why OOP has become the most commonly used paradigm for writing software. 

Of course, there are problems that would be hard to solve if you go with the object-oriented approach, and they would have been analyzed and resolved easier if you chose another paradigm, but these problems can be considered relatively rare.

Take the example of naming things in the software that we are writing. If a human writes a piece of code, object-oriented thinking is quite evident in the names of variables that we have -   
```
char*  student_first_names[10];
char*  student_surnames[10];
int   student_ages[10];
double   student_marks[10];
```

The idea is that we inherently group similar data, under a common name, and in such a way that the common name gives to the group of data actually tells us something about what it stores.
That is a lot more convenient than something like - 
```
char*  aaa[10];
char*  bbb[10];
int   ccc[10];
double   ddd[10];
```

Variable naming is – and has always been – important, because the names remind us of the concepts in our mind and the relationships between data and those concepts. By using this kind of ad hoc naming, we lose those concepts and their relationships in the code. Of course, the computer does not care about it in the code, it'll just go on about it's day, executing the next block.

The poor soul, however, who shall eventually end up debugging the piece of code that we wrote? Yeah, he/she'll curse the hell out of the one who wrote something like this!

Thus, the objectification of data and thus, object-oriented design of software is much more fundamental and inherent to programmers than what you and I realized!

Now, concepts are quite important to OOP, as if we aren't able to establish an understanding of things, forget about extracting information and maintaining interrelations between them!

So, object-oriented thinking is about thinking in terms of concepts and their relationships. It follows, then, that if you want to write a proper object-oriented program, you need to have a proper understanding of all the relevant objects, their corresponding concepts, and also their relationships, in your mind.

An object-oriented map formed in your mind, which consists of many concepts and their mutual interrelations, cannot be easily communicated to others, for instance when approaching a task as a team. More than that, such mental concepts are volatile and elusive, and they can get forgotten very easily. 

This also puts an extra emphasis on the fact that you will need models and other tools for representation, in order to translate your mind map into communicable ideas.

### Objects are not in the code

Think of the human as a machine, and the life lived as the run-time of the machine. The life of a human, is filled with relations and inter-relations, that make up a complex set of connections, all useful to each other. 

The same way, objects, can only exist in a running program. Paradoxically, there are no objects when we are writing a program, as it doesn't exist yet, and thus cannot be running! How the hell do we write object oriented code when there are no objects, and no running program?!?!

Well, the important thing to note, when thinking about this issue, is that OOP is not about creating objects, it is about creating a set of instructions that will lead to a fully dynamic object model when the program is run. So, the object-oriented code should be able to create, modify, relate, and even delete objects, once compiled and run.

The art of imagining something which is not yet created and describing or engineering its various details is usually called design. That's why this process is usually called object-oriented design (OOD) in object-oriented programming.OOP leads to a set of instructions for when and how an object should be created. 

Of course, it is not only about creation. All the operations regarding an object can be detailed using a programming language. An OOP language is a language that has a set of instructions (and grammar rules) that allow you to write and plan different object-related operations.

Every object has a dedicated life cycle. This is also true for concepts in the mind. At some point, an idea comes to mind and creates a mental image as a concept, and at some other point, it fades away. The same is true for objects. An object is constructed at one point and is destructed at another time.

### Object Attributes
The defining features of any object, that help us distinguish one object from the other, are the attributes of the object. A collection of all the attributes of an object is called as the state of the object. It can be thought of as a list of values that help us define the object, at a particular point in time. 

The state of an object can be changed anytime, and such an object is called a mutable object. Objects can also be written to be stateless, or even immutable, like the number 2! The immutable objects are really important, as their states cannot be changed, even in a multithreaded environment! 

### Domain
No matter how small, every program has a well defined domain for itself. It is the the boundary in which software exhibits its functionality. It also defines the requirements that software should address. A domain uses a specific and predetermined terminology (glossary) to deliver its mission and have engineers stay within its boundaries. Everyone participating in a software project should be aware of the domain in which their project is defined.

If a programming language doesn't provide facilities for working with the concepts specific to a given domain (such as the concepts of patients and medicines in the healthcare domain), it would be difficult to write the software for that domain using that programming language – not impossible, but certainly complex. Moreover, the bigger the software is, the harder it becomes to develop and maintain. 

