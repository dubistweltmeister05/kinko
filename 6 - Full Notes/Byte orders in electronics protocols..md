[[Blog Topics]]
[[Twitter Posting]]

Things like byte orders are among the prime example of why electronics and communication protocols are a bitch to work with. So, to elaborate a bit on what am I on about - there's multiple ways of representing how a 32-bit value is written in software. I am assuming we all know that a byte is 8 bits, and 32 bits would mean that the value, has 4 bytes. 

Let's name these bytes as A, B, C, and D.  Meaning, for a 32-bit variable X[0:31], A -> X[0:7], B -> X[8:15],  C -> X[16:23], and  D -> X[24:31]. Wouldn't it all be so simple if all  the computing machinery in the world, just, STICK TO THIS SIMPLE ELABORATION? 

But nope, there's a bunch of byte orders that exist in the world, and it actually relates to the evolution of computers. Let me explain. You see, there's this thing called "endineness" in compute. It is, essentially, the convention used to decide which byte of a multi-byte value comes first when that value has to be represented somewhere outside the abstract mathematical value itself - most commonly in memory or across a communication interface.

Let's take our 32-bit value again:

X[32] = [A B C D]; 
From a purely mathematical perspective, there isn't really any ambiguity here. `A` is the least-significant byte and `D` is the most-significant byte. The value is effectively:

`D × 2²⁴ + C × 2¹⁶ + B × 2⁸ + A`

The trouble begins when we ask a very simple question:

**If I put this value into four consecutive bytes of memory, which byte goes at the lowest address?**

And this is where computers decided that apparently one answer wasn't enough. In **little-endian** representation, the least-significant byte comes first:
`A B C D`

But in **big-endian** representation, the most-significant byte comes first:  
`D C B A`

The underlying 32-bit value hasn't changed at all. Only the way that it has been written, changes. And it only gets crazier. You see - Endianness becomes relevant when you have **more than one byte** and need to establish an ordering between those bytes.

And, naturally, this becomes particularly interesting when computers start talking to each other. Imagine one machine sends the 32-bit value:
`0x12345678`

If we split that into our four bytes:
`12 34 56 78`

A big-endian system naturally lays those bytes out as:
`12 34 56 78`

while a little-endian system lays them out in memory as:
`78 56 34 12`

Now imagine copying those four bytes directly onto a communication bus.  If the sender transmits its memory representation without thinking about the distinction, the receiver might reconstruct an entirely different number. And this is where communication protocols had to step in and say:

**"Fine. We'll define the byte order ourselves."**

Which is why you'll encounter terms like **network byte order**, **Modbus byte order**, **word order**, **CDAB**, **BADC**, and a handful of other permutations when dealing with actual hardware. And this is where things get considerably more entertaining, because now we're no longer just dealing with _endianness_.

We're dealing with the fact that engineers can take four bytes and apparently decide that **ABCD, DCBA, BADC, CDAB, and every other permutation known to mankind** are all perfectly reasonable ways of transmitting the same 32-bit value.