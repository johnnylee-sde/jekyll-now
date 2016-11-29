---
layout: post
title: Fast numeric string to int
---

#Fast numeric string to int

I was working on code to convert a string of ASCII numbers to an integer.

In one code execution hotspot, the case involved code that had to convert 8 decimal number ASCII characters to a number, the result would be an integer in the range 0 - 99,999,999.

Typical code:
```c
// given num[] - array of ASCII chars containing decimal digits 0-9
int sum = 0;
for (int i = 7; i >= 0; i--)
{
	sum = (sum * 10) + (num[i] - '0');
}
```

One way to speed up this code would be to unroll the loop:
```c
int sum;
sum = (num[7] - '0') * 10000000 +
      (num[6] - '0') * 1000000 +
      (num[5] - '0') * 100000 +
      (num[4] - '0') * 10000 +
      (num[3] - '0') * 1000 +
      (num[2] - '0') * 100 +
      (num[1] - '0') * 10 +
      (num[0] - '0');
```

Either solution is O(N) in execution speed, where N is the number of numeric ASCII digits.

We can estimate the cost of the code by calculating the number of loads, adds/subtracts, shifts, and multiplies.

For the unrolled loop code, we have
- 8 loads
- 8 subtracts, 7 adds, 8 adds to index into num array
- 7 multiplies

Let's assume all the operations **except** multiplies are the same cost. Multiplies cost more.

Unrolled loop algorithm cost total = 31 ops and 7 multiplies.

---
But to go faster you have to think **bigger**.

Now that 64-bit CPUs and OSes are common, we can start unleashing the full power of the 64-bit CPU registers on the problem.

## Concept #1
In the ASCII character set, number characters ('0'-'9') are in the range 0x30 - 0x39 (48-57).

If we bitwise-AND each ASCII number character with 0x0F, we convert the ASCII number character to its corresponding decimal number for that number character. 

'0' - '9' == 0x30 - 0x39, (0x30 - 0x39) & 0x0F => 0x0 - 0x9

Now if we load an 8 digit numeric string into a 64-bit CPU register,  on an Intel CPU (little-endian), we'll see the following:
```c
// given the string "87654321", on little-endian Intel CPUs we see the reversed:
sum = 0x3132333435363738
```
Doing a bitwise-AND of the value with 0x0F0F0F0F0F0F0F0F will give us the decimal digits involved:
```c
// given the string "87654321", bitwise-AND with 0x0F0F0F0F0F0F0F0F
sum = *((long long *)num) & 0x0F0F0F0F0F0F0F0F;
sum == 0x0102030405060708
```
## Concept #2
Let's rewrite the above result so we can distinguish the high digit from the low digit of a number.

Due to the load order of little-endian Intel CPUs, the high digit is stored in the byte below the low digit.
```c
// given the string "ZzYyXxWw", bitwise-AND with 0x0F0F0F0F0F0F0F0F
sum = 0x0w0W0x0X0y0Y0z0Z
```
We want to combine the low digit in the ones position and the high digit in the tens position into one number.

- Bitwise-AND ALL the high digits and multiply the high digits by 10 to get them into the tens position
- Right shift the low digits into the same spot as the high digits - no need to multiply since the low digits are already in the ones position
- Add both of them together.
```c
// isolate the high digit, multiply by 10, shift over the low digit and add in
sum = ((sum & 0x000F000F000F000F) * 10) + ((sum >> 8) & 0x000F000F000F000F);

// now we have the following
// where [Nn] represents the decimal number Nn in a byte, 0 <= Nn <= 99
// N represents the decimal digit in the tens position
// n represents the decimal digit in the ones position
sum = 0x00[Ww]00[Xx]00[Yy]00[Zz];
```

Extend the concept to combine the numbers into larger groups.
```c
// numbers are in range 0-99 (0x0-0x63) now
// - isolate the high number (use 0x7F since that encompasses number range)
// - multiply by 100 to move high number into thousands & hundreds position
// - shift the low number over to tens and ones position and isolate
// - add the two numbers together
sum = ((sum & 0x0000007F0000007F) * 100) + ((sum >> 16) & 0x0000007F0000007F);
```

There are now two groups of numbers,  in the range 0-9,999.

Once more, extend concept.
```c
// numbers are in range 0-9,999 (0x0-0x270F) now
// isolate the high number (use 0x3FFF since that covers number range)
//   then multiply by 10000 to move high number into position
// shift the low number over and isolate
// add the two numbers together
sum = ((sum & 0x3FFF) * 10000) + ((sum >> 32) & 0x3FFF);
```

### Final algorithm

```c
// given num[] - array of ASCII chars containing decimal digits 0-9
long long sum;
sum = *((long long*)num) & 0x0F0F0F0F0F0F0F0F;
sum = ((sum & 0x000F000F000F000F) * 10 )   + ((sum >>  8) & 0x000F000F000F000F);
sum = ((sum & 0x0000007F0000007F) * 100)   + ((sum >> 16) & 0x0000007F0000007F);
sum = ((sum & 0x3FFF)             * 10000) + ((sum >> 32) & 0x3FFF);
```
The solution is O(lg N) in execution speed, where N is the number of numeric ASCII digits.

Final algorithm cost is
- 1 load
- 7 bitwise ANDs
- 3 right shifts
- 3 adds
- 3 multiplies

Assuming equal cost of non-multiply operations results in 14 ops and 3 multiplies.

Algorithm     | Ops | Multiplies
--------------|-----|-----------
Unrolled loop | 31  |    7
SIMD          | 14  |    3


----------
#Prior work

Off to the web to see if someone has developed anything similar or better to this algorithm.

First search result leads to <http://govnokod.ru/13461>

The first response's solution resembles my algorithm.

The third response's solution is impressive:
```c
str.a = (str.a & 0x0F0F0F0F0F0F0F0F) * 2561 >> 8;
str.a = (str.a & 0x00FF00FF00FF00FF) * 6553601 >> 16;
str.a = (str.a & 0x0000FFFF0000FFFF) * 42949672960001 >> 32;
```
The hex constants used in the bitwise-AND operations are similar to my algorithm and serve the same purpose.

But **what are those  magic constants used in the multiplication**?

Let's take a look at the first statement:
```c
str.a = (str.a & 0x0F0F0F0F0F0F0F0F) * 2561 >> 8;
```
Recall our high school linear algebra:
```
5x + 3x + x = 9x
```
So the above multiplication can be rewritten as
```c
tmp = (str.a & 0x0F0F0F0F0F0F0F0F)
tmp2 = (((256 * 10) * tmp) + 1 * tmp);
     // multiply by 256 is the same as left shift by 8
     == ((10 * tmp) << 8) + (1 * tmp);
str.a = tmp2 >> 8;
```
**So one multiplication has the same effect as 1 multiply, 1 left shift, and 1 add!!!**

Remember the load order on little-endian Intel CPUs.
```c
sum = 0x0w0W0x0X0y0Y0z0Z
```
So the statement `(10 * tmp) << 8` is moving the high digit into the tens position and then shifting the result into the same byte position as the low digits.

The `+ (1 * tmp)` expression adds the low digits to the above high digits expression.

The final `>> 8` moves the combined number into the lower byte.

The result is the high and low digits combined in the tens and ones position in those bytes.

The two other statements combine the ever-larger groups of numbers together.
```c
// number groups are in range 0-99 now
// tmp1 = (str.a & 0x00FF00FF00FF00FF);
// str.a = ( ((65536 * 100) * tmp1) + (1 * tmp1) ) >> 16;
//       == ( ((100 * tmp1) << 16) + (1 * tmp1) ) >> 16;
str.a = (str.a & 0x00FF00FF00FF00FF) * 6553601 >> 16;

// number groups are in range 0-9,999 now
// tmp2 = (str.a & 0x0000FFFF0000FFFF);
// str.a = ( ((4294967296 * 10000) * tmp2) + (1 * tmp2) ) >> 32;
//       == ( ((10000 * tmp2) << 32) + (1 * tmp2) ) >> 32;
str.a = (str.a & 0x0000FFFF0000FFFF) * 42949672960001 >> 32;

```
----------
Let's calculate the algorithm cost:
```c
str.a = *(long long *)num;
str.a = (str.a & 0x0F0F0F0F0F0F0F0F) * 2561 >> 8;
str.a = (str.a & 0x00FF00FF00FF00FF) * 6553601 >> 16;
str.a = (str.a & 0x0000FFFF0000FFFF) * 42949672960001 >> 32;
```
Cost is:
- 1 load
- 3 bitwise ANDs
- 3 right shifts
- 3 multiplies.
Assuming equal cost of non-multiply operations results in 7 ops and 3 multiplies!

Algorithm     | Ops | Multiplies
--------------|-----|-----------
Unrolled loop | 31  |    7
SIMD          | 14  |    3
super-SIMD    |  7  |    3

