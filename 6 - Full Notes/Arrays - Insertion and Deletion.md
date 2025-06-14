2025-01-01 14:16

Tags: [[Arrays]]

# Arrays - Insertion and Deletion

### Insertion

Well, the index insertion is easy enough. Make a function of that returns an integer. Take the following parameters - `int *a, int *size, int element, int index, int capacity`. 
 
They are for the array, the pointer to the size of the array, the element to be inserted, the index of the intended insertion and the capacity of the array (to see if the array is full or not). 
 
If the size of the array is equal to or exceeds the array, it is full and cannot be inserted into. Simply, return -1.

If not, then we start a loop from the end of the array till we are at the index of the array. To move the elements out of the way, we say `a[i+1] = a[i]`. When the loop closes, it means that we are at the index just before the intended index and now, we simply assign `a[index]=element`. Return the index at which we assigned to make sure that everything is alright. 

### Deletion
For this, make a function that returns an integer as well, and take the arguments as `int *a, int *size, int index, int capacity`. 

Check if the input index for the deletion is greater that the size of the array, because then the deletion is literally not possible. Once that check is clear, start a for loop from the index to be deleted till the last element, and just `a[i]=a[i+1]`; and you are good to go. 

# Reference

https://youtu.be/o9WevKSnHL4?si=rWPLrxUxItBmj0KH
https://youtu.be/2jVcRw1jP9I?si=PbWvtNgMx_97VmKt