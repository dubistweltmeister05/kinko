2025-01-26 10:16

Tags: [[Hash Maps]]

It is literally an array with a hash function. 

The hash function takes an input and maps it to a location in the array that we have. Say we have a struct like this
```struct astartes{
	char* chapter;
	char* primarch;
	int yearsServed;
};
```

The hashing function will take objects of this struct and will map a member of the object to an index of the array. 






# Reference

