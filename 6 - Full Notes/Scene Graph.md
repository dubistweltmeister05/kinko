[[Data Structures and Algorithms]]
[[Linux]]
[[Blog Topics]]

A data structure that is used in vector-based graphics editing applications and modern computer games to arrange the logical and spatial representation of the graphics scene. 

It is a collection of nodes of a graph or a tree, and the rule followed there is that there can only be one parent of a node, but a node can have many children. The effect of the parent node is applied to ALL it's children, effectively - any operation on a parent node is automatically done on it's children nodes. 

Usually, associating a transformation matrix at each level, concatenating such matrices from each level and then running them is an efficient way of processing operations on the scene graph. 

A common feature, for instance, is the ability to group related shapes and objects into a compound object that can then be manipulated as easily as a single object.