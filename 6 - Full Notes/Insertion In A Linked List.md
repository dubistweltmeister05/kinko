2025-01-01 22:11

Tags: [[Linked Lists]]

Right, so there's 4 ways that you can insert a node in the Linked List. Either at the top, at the end, in the middle, or after. Let's take a look at each of these - 
### Insert at the Top
1. **Create a temp node**:
   - Allocate memory for a new node (e.g., `temp = (struct node *)malloc(sizeof(struct node));`).
   - Set the `data` field of the node to the value you wish to insert (e.g., `temp->data = value;`).

2. **Make the temp node point to head**:
   - The `temp` node should now point to the current `head` of the list (e.g., `temp->next = head;`).

3. **Make the temp node as the head**:
   - Update the `head` of the list to point to the `temp` node (e.g., `head = temp;`).
   - The `temp` node is now the first node in the list, and the list has been updated with the new element at the beginning.

### Insert at the End
1. **Create a temp node to traverse to the last node**:
   - Initialize a pointer `temp` to the `head` of the list.
   - Traverse the list by moving `temp` through each node until you find the last node, where the `next` pointer is `NULL`.
     - `while (temp->next != NULL) { temp = temp->next; }`

2. **Create a new node (`temp1`)**:
   - Allocate memory for a new node (`temp1`).
   - Set the `data` field of `temp1` to the value that you want to insert.
     - `temp1->data = value;`

3. **Update the `next` pointer of the last node**:
   - Once the last node is found (i.e., `temp->next == NULL`), update the `next` pointer of `temp` to point to `temp1`.
     - `temp->next = temp1;`

4. **Set the `next` pointer of `temp1` to `NULL`**:
   - Ensure that the new node (`temp1`) is the new last node by setting `temp1->next = NULL;`.

5. **Return the updated head of the list**:
   - The `head` remains the same, but if the list was empty, `head` would have been updated.
### Insert after an index

1. **Create a new node:**
   - Allocate memory for the new node.
   - Set the `data` field of the new node to the value to be inserted.

2. **Handle the special case of index 0:**
   - If the index is `0`:
     - Make the `next` pointer of the new node point to the current `head`.
     - Update the `head` to the new node.
     - Exit the function.

3. **Traverse to the node before the target index:**
   - Initialize a pointer (`p`) to the `head` and a counter (`count`) to `0`.
   - Move `p` to the next node until `count` equals `index - 1` or the end of the list is reached.

4. **Check for invalid index:**
   - If the list ends before reaching `index - 1`:
     - Print an error message or handle the error appropriately.
     - Exit the function without making changes.

5. **Insert the new node:**
   - Set the `next` pointer of the new node to the `next` pointer of the node at `index - 1`.
   - Update the `next` pointer of the node at `index - 1` to point to the new node.

6. **Return the updated head of the linked list.**

### Edge Cases to Handle:
- Inserting at index `0` in an empty or non-empty list.
- Invalid index (e.g., negative index or index greater than the list size).
- Memory allocation failure.

# Reference

https://youtu.be/ewCc7O2K5SM?si=4a8_ORFtrPUC5MIK