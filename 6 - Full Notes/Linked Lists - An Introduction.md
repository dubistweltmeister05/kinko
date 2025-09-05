2025-01-01 16:01

Tags: [[Linked Lists]]

# Linked Lists - An Introduction

This is a way to get establish a link between the blocks of run-time memory(heap) that we request to store incoming data.

Blocks are requested in the form of a node. These nodes are linked together to create a structure where we have a chain of links from the first node(head) till the last one (tail). An example of a node that stored an integer in it is as follow - 
`typedef struct node`
`{`
    `int data;`
    `struct node *next;`
`};`

Some common algorithms and operations are as follows - 
- [x] Traversal
- [x] Insertion
- [x] Deletion
- [ ] Reversal
- [ ] Searching
- [ ] Detecting a Loop
- [ ] Merging Two Sorted Lists
- [ ] Finding the Middle Node
- [ ] Removing Duplicates
- [x] Finding the Nth Node from the End

# Reference

https://youtu.be/BHphhqL9EOE?si=ZC7iGWEs9RW696pG
