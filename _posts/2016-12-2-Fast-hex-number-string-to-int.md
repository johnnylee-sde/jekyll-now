---
layout: post
title: Fast hex number string to int
---

Extending [Fast numeric string to int]({{ site.baseurl }}{% post_url 2016-11-28-Fast-numeric-string-to-int %}) to converting hexadecimal number strings is easy.

The main wrinkle is that the range of digits extends from the original `'0'-'9'` (0x30-0x39) to `'0'-'9', 'A'-'F', 'a'-'f'` (0x30-0x39, 0x41-0x46, 0x61-0x66)

The high alphabetic hexadecimal digits (`'A'-'F', 'a'-'f'`  0x41-0x46, 0x61-0x66) are not continuous with the lower decimal digits (`'0'-'9'` 0x30-0x39).

If the code masks off the high nibble as it did when processing numeric strings, the code would be left with hexadecimal digits in the range 0x01-0x06, when it should be in the range 0x0A-0x0F.

So the code needs to somehow add 9 to **ONLY** the hexadecimal digits in the range 0x41-0x46 and 0x61-0x66.

Notice that all of the alphabetic hexadecimal digits have the 0x40 bit set.

The code can key off of the 0x40 bit set, to locate the alphabetic hexadecimal digits.

Shift that bit two times (into the 8 bit and 1 bit) and add to create the number 9 for the appropriate byte.

Then we add to the hex digit and we should be off to the races with slight tweaks to the multiplication constants, since we're using base 16 not base 10 now for the digits.

Here's the final algorithm:

```c
typedef unsigned long long ULL;
ULL n = (*(ULL *)(buffer)) & 0x4F4F4F4F4F4F4F4Full;

ULL alphahex = (ULL)(n & 0x4040404040404040ull);
// ULL  nine = ((hex >> 3) + (hex >> 6));
// ULL n0 = alphahex == 0 ? n : nine + (n & ~alphahex);
ULL n0 = alphahex == 0 ? n 
                       : ((alphahex >> 3) + (alphahex >> 6)) + (n & ~alphahex);
// 0x1001 == 4097 == 256 * 16 + 1
ULL n1 = n0 * 0x1001 >> 8;
// 0x1000001 == 16777217 == 65536 * 256 + 1
ULL n2 = (n1 & 0x00FF00FF00FF00FFull) * 0x1000001 >> 16;
// 0x1000000000001 == 281474976710657 == 4294967296 * 65536 + 1
unsigned long num = (n2 & 0x0000FFFF0000FFFFull) * 0x1000000000001 >> 32;

```


