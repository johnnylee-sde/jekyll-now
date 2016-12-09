---
layout: post
title: Fast unsigned integer to hex string
---

Once I started down this road with
[Fast unsigned integer to string]({{ site.baseurl }}{% post_url 2016-12-8-Fast-unsigned-integer-to-string %}),
the need for symmetry forced me to write an algorithm to convert unsigned integers to hex strings.

Here we go.

# Concept #1

Use bitmasks and bit-shifts to isolate individual nibbles of the 32-bit number. Code can do this in O(lg N) steps.

# Concept #2

Wrinkle for conversion to hexadecimal characters is that the code has to map a hexadecimal nibble (4 bits) 
into 2 different ranges (0x30-0x39, '0'-'9') and (0x41-0x46, 'A"-'F' or the lower hex digit equivalent, 0x61-0x66, 'a'-'f')

Code can add 6 to each isolated nibble per byte.
If the nibble addition overflows to the higher nibble, then that nibble is in the alphanumeric hexadecimal character range.
Shift the overflow carry bit into the low bit of the byte and we've created a mask for all 8 bytes in a 64-bit register.

```c
uint32_t UlongToHexString(uint64_t num, char *s, bool lowerAlpha)
{
	uint64_t x = num;

	// use bitwise-ANDs and bit-shifts to isolate
	// each nibble into its own byte
	// also need to position relevant nibble/byte into
	// proper location for little-endian copy
	x = ((x & 0xFFFF) << 32) | ((x & 0xFFFF0000) >> 16);
	x = ((x & 0x0000FF000000FF00) >> 8) | (x & 0x000000FF000000FF) << 16;
	x = ((x & 0x00F000F000F000F0) >> 4) | (x & 0x000F000F000F000F) << 8;

	// Now isolated hex digit in each byte
	// Ex: 0x1234FACE => 0x0E0C0A0F04030201

	// Create bitmask of bytes containing alpha hex digits
	// - add 6 to each digit
	// - if the digit is a high alpha hex digit, then the addition
	//   will overflow to the high nibble of the byte
	// - shift the high nibble down to the low nibble and mask
	//   to create the relevant bitmask
	//
	// Using above example:
	// 0x0E0C0A0F04030201 + 0x0606060606060606 = 0x141210150a090807
	// >> 4 == 0x0141210150a09080 & 0x0101010101010101
	// == 0x0101010100000000
	//
	uint64_t mask = ((x + 0x0606060606060606) >> 4) & 0x0101010101010101;

	// convert to ASCII numeral characters
	x |= 0x3030303030303030;

	// if there are high hexadecimal characters, need to adjust
	// for uppercase alpha hex digits, need to add 0x07
	//   to move 0x3A-0x3F to 0x41-0x46 (A-F)
	// for lowercase alpha hex digits, need to add 0x27
	//   to move 0x3A-0x3F to 0x61-0x66 (a-f)
	// it's actually more expensive to test if mask non-null
	//   and then run the following stmt
	x += ((lowerAlpha) ? 0x27 : 0x07) * mask;

	//copy string to output buffer
	*(uint64_t *)s = x;

	return 0;
}
```

I'll use a modified benchmark harness from
[Fast unsigned integer to string]({{ site.baseurl }}{% post_url 2016-12-8-Fast-unsigned-integer-to-string %})
to test for hexadecimal digit groups.

But we'll need other alternative solutions.

Easiest one is to lean on the standard library:

```c
uint32_t stdlibHexString(uint64_t num, char *s, bool lowerAlpha)
{
	sprintf_s(s, 9, (lowerAlpha) ? "%8.8x" : "8.8X", (uint32_t)num);
	return 0;
}
```

Next is a simple naive solution:

```c
uint32_t naiveHexString(uint64_t num, char *s, bool lowerAlpha)
{
	uint32_t x = (uint32_t)num;
	int i = 7;
	while (i >= 0)
	{
		int digit = (x & 0x0F);
		uint8_t ch = digit + '0';
		if (ch > '9')
			ch += (lowerAlpha) ? 0x27 : 7;

		s[i] = ch;
		x >>= 4;
		i -= 1;
	}
	return 0;
}

```

Finally, I think a lookup-table-based solution would work.

It turns out that checking each char to see if it's an alphabetic hexadecimal character is expensive timewise,
so using a separate lookup table for lowercase hex digits is faster.

```c
uint32_t lutHexString(uint64_t num, char *s, bool lowerAlpha)
{
	static const char digits[513] =
		"000102030405060708090A0B0C0D0E0F"
		"101112131415161718191A1B1C1D1E1F"
		"202122232425262728292A2B2C2D2E2F"
		"303132333435363738393A3B3C3D3E3F"
		"404142434445464748494A4B4C4D4E4F"
		"505152535455565758595A5B5C5D5E5F"
		"606162636465666768696A6B6C6D6E6F"
		"707172737475767778797A7B7C7D7E7F"
		"808182838485868788898A8B8C8D8E8F"
		"909192939495969798999A9B9C9D9E9F"
		"A0A1A2A3A4A5A6A7A8A9AAABACADAEAF"
		"B0B1B2B3B4B5B6B7B8B9BABBBCBDBEBF"
		"C0C1C2C3C4C5C6C7C8C9CACBCCCDCECF"
		"D0D1D2D3D4D5D6D7D8D9DADBDCDDDEDF"
		"E0E1E2E3E4E5E6E7E8E9EAEBECEDEEEF"
		"F0F1F2F3F4F5F6F7F8F9FAFBFCFDFEFF";
	static const char digitsLowerAlpha[513] =
		"000102030405060708090a0b0c0d0e0f"
		"101112131415161718191a1b1c1d1e1f"
		"202122232425262728292a2b2c2d2e2f"
		"303132333435363738393a3b3c3d3e3f"
		"404142434445464748494a4b4c4d4e4f"
		"505152535455565758595a5b5c5d5e5f"
		"606162636465666768696a6b6c6d6e6f"
		"707172737475767778797a7b7c7d7e7f"
		"808182838485868788898a8b8c8d8e8f"
		"909192939495969798999a9b9c9d9e9f"
		"a0a1a2a3a4a5a6a7a8a9aaabacadaeaf"
		"b0b1b2b3b4b5b6b7b8b9babbbcbdbebf"
		"c0c1c2c3c4c5c6c7c8c9cacbcccdcecf"
		"d0d1d2d3d4d5d6d7d8d9dadbdcdddedf"
		"e0e1e2e3e4e5e6e7e8e9eaebecedeeef"
		"f0f1f2f3f4f5f6f7f8f9fafbfcfdfeff";
	uint32_t x = (uint32_t)num;
	int i = 3;
	char *lut = (char *)((lowerAlpha) ? digitsLowerAlpha : digits);
	while (i >= 0)
	{
		int pos = (x & 0xFF) * 2;
		char ch = lut[pos];
		s[i * 2] = ch;

		ch = lut[pos + 1];
		s[i * 2 + 1] = ch;

		x >>= 8;
		i -= 1;
	}

	return 0;
}
```

Get ready, set, GO!

Here are the results:

<script type="text/javascript" src="https://www.gstatic.com/charts/loader.js"></script>
<script type="text/javascript">
  var g_columnsToHide = null;
  var g_chartShown = "Bar";

  google.charts.load('current');
  google.charts.setOnLoadCallback(onloadChart);

  function getData()
  {
    var data = google.visualization.arrayToDataTable([
      ['Digits', 'stdlib', 'naive', 'lut', 'UlongToHexString' ],

      // 64-bit x64 runtime results
["8", 3330617, 1145806, 197356, 213372 ],
["7", 3312310, 1042496, 196911, 212314 ],
["6", 3312378,  947453, 197164, 212771 ],
["5", 3312369,  863413, 196546, 213381 ],
["4", 3313405,  774617, 198129, 212905 ],
["3", 3307872,  680277, 196613, 214423 ],
["2", 3313773,  584070, 196707, 212968 ],
["1", 3311035,  483939, 197435, 213434 ]

    ]);

    return data;
  }

  function getColumns()
  {
    return g_columnsToHide;
  }

  function drawTable(type, div, cols) {
    var data = getData();

    var opts = {
        containerId: div,
        dataTable: data,
        chartType: type,
        options: {"title": "Convert Integer to Hex String"}
    };
    var chartwrapper = new google.visualization.ChartWrapper(opts);

    if (null != cols)
      chartwrapper.setView({'columns': cols});

    chartwrapper.draw();
  }

  function showTable(type)
  {
    g_chartShown = type;
    var cols = getColumns();
    drawTable(g_chartShown, "columnchart_material", cols);
  }

  function showChart()
  {
    showTable(g_chartShown);
  }

  function onloadChart()
  {
    showChart();
  }

  function toggleSlow()
  {
    if (null == g_columnsToHide)
    {
      g_columnsToHide = [0, 2, 3, 4];
    }
    else
    {
      g_columnsToHide = null;
    }
    showChart();
  }

</script>

<button onclick='showTable("Bar")'>Show Bar Chart</button><span> </span>
<button onclick='showTable("Line")'>Show Line Chart</button><span> </span>
<button onclick='showTable("Table")'>Show Table</button><span> </span>
<label><input type='checkbox' onclick='toggleSlow()' ><span>Hide Slow algorithm</span></label>
<hr />
<div id="columnchart_material" style="width: 900px; height: 600px;"></div>

# Solution Performance

The standard library solution was the slowest, but it's a lot more flexible.

One can tell that the naive solution is O(N) from the nice slope of the execution time.

My solution has a nice mostly-flat O(lg N) execution slope. It's about 8% slower than the fastest solution,
but it doesn't require the 1KB of lookup tables.

The lookup table solution turned out to be the fastest - space vs speed tradeoff in play.

