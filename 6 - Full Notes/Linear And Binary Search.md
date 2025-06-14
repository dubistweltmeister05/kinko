2025-01-01 15:32

Tags: [[Searching Algorithms]]

## Linear Search
Come on mate.

### Binary Search
Right, this is the first algorithm that we are going to learn about, so you better pay attention son. 

The array that you are going to look at HAS to be SORTED for this to work. 

Steps after ensuring that are as follows - 
1. Set two pointers, low and high at the first and the last element of the array.
2. While low<=high:
	1. compute the middle index as (low)+(low - high)/2 
	2. compare the element array[mid] to what you are searching for.
		1. If you have a match, return mid.
		2. if array[mid]< target, low = mid+1.
		3. if array[mid]>target, high =mid-1.
3. if low>high, the element is NOT in it. 

`int binary_iterative(int *a, int target, int low, int high)`
`{`
    `while (low <= high)`
    `{`
        `int mid = low + (high - low) / 2;`
        `if (a[mid] == target)`
            `return mid;`
        `else if (a[mid] > target)`
            `high = mid - 1;`
        `else if (a[mid] < target)`
            `low = mid + 1;`
    `}`
    `return -1;`
`}`

`int binary_recursive(int *a, int target, int low, int high)`
`{`
    `if (low > high)`
    `{`
        `return -1;`
    `}`

    `int mid = low + (high - low) / 2;`

    `if (a[mid] == target)`
        `return mid;`
    `else if (a[mid] > target)`
        `binary_recursive(a, target, low, (mid - 1));`
    `else if (a[mid] < target)`
        `binary_recursive(a, target, (mid + 1), high);`
`}`
# Reference

https://youtu.be/ZHCP9vFOJiU?si=G7uTBCUrbBL0NvVs