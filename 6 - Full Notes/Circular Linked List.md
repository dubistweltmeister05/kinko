2025-01-07 08:17

Tags:[[Linked Lists]] 

A linked List where the tail points to the head instead of NULL is a Circular Linked List. It can be singly or doubly linked. 

The biggest issue that you are going to face while traversal is the problem of infinite loops, so do keep that in mind and implement additional checks.

### ADVANTAGES
- Like the name suggests, it is perfect for circular tasks, such as round-robin scheduling. 
- Since the last node points to the first one, traversal becomes faster and a little more efficient (although I do not necessarily agree with this). Apparently, the absence of a need to check for the NULL (end) condition makes it faster to work with. 
- Another BS point in my opinion but apparently, a CLL needs less space in some cases as it does not need a separate pointer to keep track of the end node, thus needing less memory when compared to other LL implementations.
- The circular structure of the list allows for greater flexibility in certain applications. For example, a circular linked list can be used to represent a ring or circular buffer, where new elements can be added and old elements can be removed without having to shift the entire list.
- The time taken to insert and delete a node at any position is CONSTANT, as you don't need to go to the end of the list in any case. 

### DISADVANTAGES
- **Complexity**: Circular linked lists can be more complex than other types of linked lists, especially when it comes to algorithms for insertion and deletion operations. For example, determining the correct position for a new node can be more difficult in a circular linked list than in a linear linked list.
- **Memory** **leaks**: If the pointers in a circular linked list are not managed properly, memory leaks can occur. This happens when a node is removed from the list but its memory is not freed, leading to a buildup of unused memory over time.
- **Traversal can be more difficult:** While traversal of a circular linked list can be efficient, it can also be more difficult than linear traversal, especially if the circular list has a complex structure. Traversing a circular linked list requires careful management of the pointers to ensure that each node is visited exactly once.
- **Lack** **of a natural end:** The circular structure of the list can make it difficult to determine when the end of the list has been reached. This can be a problem in certain applications, such as when processing a list of data in a linear fashion. 

# Reference 

