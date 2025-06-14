2025-01-01 23:20

Tags: [[Linked Lists]] 

So, there are 3 cases here too. They are as follows - 
1. Deleting the first node
2. Deleting the last node
3. Deleting a node in between.
### Deleting the first node. 
Quite possibly, the most simple case for deletion. The algo is literally head=head->next, and free(head).

### Deleting the last node

1. **Input:**
   - A pointer `head` to the head of the singly linked list.

2. **Edge Case Handling:**
   - If `head == NULL` (list is empty), return `NULL`.
   - If `head->next == NULL` (single-node list):
     - Free the memory for the node.
     - Return `NULL`.

3. **Traverse to the Second-to-Last Node:**
   - Initialize a pointer `current = head`.
   - Use a loop to traverse until `current->next->next == NULL`:
     - Update `current = current->next`.

4. **Unlink the Tail Node:**
   - Store the tail node (`current->next`) in a temporary pointer `temp`.
   - Update `current->next = NULL`.

5. **Free the Tail Node:**
   - Deallocate memory for `temp`.

6. **Return the Updated Head Pointer.**

### Deleting a node in between
1. **Input:**
   - A pointer `head` to the head of the singly linked list.
   - An integer `index` representing the position of the node to be deleted (0-based index).

2. **Edge Case Handling:**
   - If `head == NULL` (list is empty), return `NULL`.
   - If `index < 0`, return `head` (invalid index).

3. **Delete the Head Node (index == 0):**
   - Store `head` in a temporary pointer `temp`.
   - Update `head = head->next`.
   - Free the memory of the old head node (`temp`).
   - Return the updated `head`.

4. **Traverse to the Node Before the Target Node:**
   - Initialize a pointer `current = head` and `prev = NULL`.
   - Iterate through the list using a loop:
     - For each iteration `i` from `0` to `index - 1`:
       - Update `prev = current`.
       - Update `current = current->next`.
       - If `current == NULL`, return `head` (index out of bounds).

5. **Unlink the Target Node:**
   - Update `prev->next = current->next`.

6. **Free the Target Node:**
   - Deallocate memory for `current`.

7. **Return the Updated Head Pointer.**
# Reference

https://youtu.be/R_7qJzAWrMg?si=npBnKO1o_VySN_mZ

https://youtu.be/UQIJNobtzVY?si=mRX3vNIOuL2CHEwS