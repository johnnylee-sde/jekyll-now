---
layout: post
title: Fast time string to seconds
---

Investigating another code execution hotspot, the case involved converting a time string representing 24 hour time into the number of seconds elapsed since midnight.

The 24-hour time format is `HH:MM:SS`.

Code was similar to the following:

```
int Convert2ByteToInt( char *s )
{
    return (s[0] - '0') * 10 + (s[1] - '0');
}

// assme dt[] - array represent date & 24 hour time
// assume offset 11 is where the 24 hour time begins
int secs = Convert2ByteToInt( &dt[11] ) * 3600 + 
           Convert2ByteToInt( &dt[14] ) * 60 + 
           Convert2ByteToInt( &dt[17] );
```

Estimated cost is:

`Convert2ByteToInt` function cost:

- 2 Loads
- 2 Subtracts, 1 Add
- 1 Multiply

Total: 5 ops, 1 Multiply

main code cost:

- 3 Adds for array indexing
- 2 Multiplies
- 2 Adds
 
Total: 5 ops, 2 Multiplies
 
Estimated total cost is 3 x (5 ops, 1 Multiply) + (5 ops, 2 Multiplies) = 20 ops, 5 Multiplies

----

Using the techniques from yesterday's post [Fast numeric string to int]({{ site.baseurl }}{% post_url 2016-11-28-Fast-numeric-string-to-int %}), we can write a faster version.

```
long long sum;

// 24 hour time string of form: "12:34:56"
// loaded into 64-bit register as: 0x36353A34333A3231
sum = *(long long *)dt & 0x0F07000F07000F03;

sum = (sum * 2561) >> 8;

// move seconds into ones position
// multiply mins by 60, multiply hours by 3600 and
// shift right into minutes position
// move converted hours & mins into ones pos
// sum = (sum >> 48) + ((sum & 0x000000003F00001F) * 
//          ( 60 + (((long long) 1<<24 ) * 3600)) ) >> 24;
sum = (sum >> 48) + ((sum & 0x000000003F00001F) * 0xE1000003C) >> 24;

// max number of secs is 24 * 60 * 60
int secs = (int)(sum) & 0x1FFFF;
```

Estimated cost is:

- 1 Load, 1 Bitwise AND
- 1 Multiply, 1 Shift right
- 2 Shift right, 1 Add, 1 Bitwise AND, 1 Multiply
- 1 Bitwise AND

Estimated total cost is 8 ops, 2 Multiplies

Algorithm     | Ops | Multiplies
--------------|-----|-----------
Original      | 20  |    5
SIMD          |  8  |    2

