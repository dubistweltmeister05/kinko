2025-01-07 11:44

Tags: [[Linked Lists]]

The algorithm to do this is quite simple to be honest. 

## Algorithm to Reverse a Singly Linked List

### Iterative Approach
1. **Initialize**:
   - `prev` = `NULL` (to mark the end of the reversed list).
   - `curr` = `head` (to start traversing the list).

2. **Traverse the List**:
   - While `curr` is not `NULL`:
     1. Store the next node: `next = curr->next`.
     2. Reverse the link: `curr->next = prev`.
     3. Move `prev` to the current node: `prev = curr`.
     4. Move `curr` to the next node: `curr = next`.

3. **Return**:
   - After the loop, `prev` will be the new head of the reversed list. Return `prev`.

---

### Recursive Approach
1. **Base Case**:
   - If the list is empty (`head == NULL`) or contains only one node (`head->next == NULL`), return `head`.

2. **Recursive Call**:
   - Reverse the rest of the list: `newHead = reverse(head->next)`.

3. **Reverse the Link**:
   - Make the next node point back to the current node: `head->next->next = head`.
   - Set `head->next = NULL` to disconnect the reversed part from the rest.

4. **Return**:
   - Return `newHead`, which is the head of the reversed list.

---

### Complexity Analysis
- **Time Complexity**: 
  - O(n) for both iterative and recursive approaches (each node is processed once).
- **Space Complexity**:
  - Iterative: O(1) (constant space used for pointers).
  - Recursive: O(n) due to the recursion stack.

---

### Example
#### Input:
`1 -> 2 -> 3 -> 4 -> NULL`

#### Output:
`4 -> 3 -> 2 -> 1 -> NULL`

# Reference

