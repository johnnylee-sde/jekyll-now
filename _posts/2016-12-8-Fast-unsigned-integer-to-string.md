---
layout: post
title: Fast unsigned integer to string
---

While perusing [Hacker's Delight miscellaneous material](http://www.hackersdelight.org/corres.txt),
I read `(15) Majority function and BCD algorithms`

> (15) From Paolo Bonzini, 10/18/2013:
>
> ...
> 
> Conversion to BCD, instead, is the really interesting routine.  A trivial
> implementation requires 7 multiply-high instructions, 7 multiplications
> by 10, and 21 other arithmetic/logical instructions:
> 
> ...
> 
> This method however does not scale too well to 64-bit machines, which can
> hold 16 BCD digits in a register.  Of course it is possible to split the
> 16 digits in two groups of 8, so that only one 64-bit multiply high is
> used.
> 
> But then, one wonders if a divide-by-conquer algorithm exists, and
> whether it is already efficient enough on a 32-bit machine.
> 
> It turns out that the algorithm exists, and it showcases a large number
> of techniques from the book.  Of course, a large number of divisions
> by constants are there; most of them can be optimized to avoid a
> multiply-high instruction.  Some steps even treat a register as multiple
> subfields and perform divisions in parallel on all subfields.
> 
> The magic numbers are (103,10) for division by 10, and (5243,19)
> for division by 100.
> 
> Because the divide-and-conquer only goes down to 4 bits in this function,
> all three division steps actually have different code.  The recursive
> structure is a bit more visible in the 64-bit version.  The compression
> step (an unrolled version of the "parallel prefix" algorithm) is more
> clearly identifiable in the 64-bit version, too.
> 
> The following routine processes up to 16 decimal digits in four steps.
> A new magic number is needed for division by 10000.  The magicgu Python
> routine computes it as (109951163,40).  Two steps are now able to use
> the parallel divisions trick, the final one doing four divisions with
> a single multiplication:

```c
    /* Let the compiler figure out a divmod instruction.  */
    h = x/100000000; l = x%100000000;

    /* Two divisions and remainders by 10000 (27 bits needed).  */
    hh = (h * 109951163) >> 40; h -= hh * 10000; h |= hh << 32;
    ll = (l * 109951163) >> 40; l -= ll * 10000; l |= ll << 32;

    /* Four divisions and remainders by 100.  */
    hh = ((h * 5243) >> 19) & 0x000000FF000000FF; h -= hh * 100; h |= hh << 16;
    ll = ((l * 5243) >> 19) & 0x000000FF000000FF; l -= ll * 100; l |= ll << 16;

    /* Eight divisions by 10 (14 bits needed), followed by compression of the
     * nybbles instead of a remainder operation.
     */
    hh = ((h * 103) >> 9) & 0x001E001E001E001E; h += hh * 3;
    ll = ((l * 103) >> 9) & 0x001E001E001E001E; l += ll * 3;

    /* Compress h and l right with mask 0x00FF00FF00FF00FF.  */
    hh = (h & 0x00FF000000FF0000l); h ^= hh ^ (hh >> 8);  // 0x0000aabb0000ccdd
    ll = (l & 0x00FF000000FF0000l); l ^= ll ^ (ll >> 8);  // 0x0000aabb0000ccdd

    hh = (h & 0x0000FFFF00000000l); h ^= hh ^ (hh >> 16); // 0x00000000aabbccdd
    ll = (l & 0x0000FFFF00000000l); l ^= ll ^ (ll >> 16); // 0x00000000aabbccdd

    /* Combine the two (simpler than compressing h left, also because masks
     * are reused).  */
    return l|(h<<32);
```

## Concept #1

Converting a binary number to BCD is halfway to converting a binary number to an ASCII number string.

- remove compression step - BCD numbers are useful at this point for conversion to ASCII number characters
- add code to position numbers for little-endian Intel CPU copying
- add code to convert from BCD numbers to ASCII number characters
- limit code for conversion to 32-bit numbers, which means `h` only needs to worry about 0-42 since unsigned 32-bit numbers can range 0-4294967295
- copy only the appropriate number of bytes to the resulting string buffer - an N digit number only results in N bytes in the string buffer

# Original solution

```c
uint32_t UlongToStringOrig(uint64_t x, char *s)
{
	uint64_t low;
	uint64_t ll;
	uint32_t digits;

	// 8 digits or less?
	if (x < 100000000)
	{
		// fits into single 64-bit CPU register
		// no division/modulus needed
		low = x;

		// calc num digits
		// can use binary search
		// assume smaller numbers more frequent
		if (low <= 9999) {
			// less than 3 digits?
			if (low <= 99) {
				digits = (low > 9) ? 2 : 1;
			} else {
				digits = (low > 999) ? 4 : 3;
			}
		} else {
			// more than 6 digits?
			if (low > 999999) {
				digits = (low > 9999999) ? 8 : 7;
			} else {
				digits = (low > 99999) ? 6 : 5;
			}
		}
	}
	else
	{
		// Let the compiler figure out a divmod instruction
		low = (uint64_t)x % 100000000;
		uint64_t high = (uint64_t)x / 100000000;

		// h will be at most 42 for an unsigned 32-bit num
		// calc num digits
		digits = (high > 9) ? 10 : 9;

		if (high > 9)
		{
			uint64_t hh = ((high * 103) >> 9) & 0x1E; high += hh * 3;
			*(uint16_t *)s = ((high & 0xF0) >> 4) | 
								((high & 0xF) << 8) | 0x3030;
		}
		else
		{
			*s = (uint8_t)(high | 0x30);
		}
	}

	ll = (low * 109951163) >> 40; low -= ll * 10000; low |= ll << 32;

	// Four divisions and remainders by 100
	ll = ((low * 5243) >> 19) & 0x000000FF000000FF; low -= ll * 100; low = (low << 16) | ll;

	// Eight divisions by 10 (14 bits needed)
	ll = ((low * 103) >> 9) & 0x001E001E001E001E; low += ll * 3;

	// move digits into correct spot
	ll = ((low & 0x00F000F000F000F0) >> 4) | (low & 0x000F000F000F000F) << 8;
	ll = (ll >> 32) | (ll << 32);

	// convert from decimal digits to ASCII number digit range
	ll |= 0x3030303030303030;

	if (digits >= 8)
	{
		*(uint64_t *)(s + digits - 8) = ll;
	}
	else
	{
		auto d = digits;
		auto s1 = s;
		auto pll = (char *)&(((char *)&ll)[8 - digits]);

		if (d >= 4) {
			*(uint32_t *)s1 = *(uint32_t *)pll;
			s1 += 4; pll += 4; d -= 4;
		}
		if (d >= 2) {
			*(uint16_t *)s1 = *(uint16_t *)pll;
			s1 += 2; pll += 2; d -= 2;
		}
		if (d > 0) {
			*(uint8_t *)s1 = *(uint8_t *)pll;
		}
	}

	return digits;
}
```

After running tests that proved the solution was correct for all 32-bit numbers, I looked to see if there were other solutions.

One of the first search results listed was [Leandro Pereira - Integer to string conversion](https://tia.mat.br/posts/2014/06/23/integer_to_string_conversion.html).

The post also included several alternative code solutions and a benchmark harness. Cool.

Integrating my solution into the code, compiling, and then running the code showed the following results:

<script type="text/javascript" src="https://www.gstatic.com/charts/loader.js"></script>
<script type="text/javascript">

  var g_columnsToHide = null;
  var g_chartShown = "Bar";
  var g_chartShownOrig = "Bar";

  google.charts.load('current');
  google.charts.setOnLoadCallback(onloadChart);

  function getData()
  {
    var data = google.visualization.arrayToDataTable([
      ['Digits', 'lwan', 'lwan_lut', 'facebook_digits10', 'facebook_digits10_ms', 
       'facebook_fixed', 'facebook_fixed_small', 'facebook_fixed_ms', 
       'UlongToStringOrig', 'UlongToString', 'naive' ],

      // 64-bit x64 runtime results
["10", 836376, 763143, 682906, 616637, 594116, 590061, 495791, 428920, 398849, 974404],
["9", 734883, 666082, 640903, 562000, 539946, 537070, 438721, 371752, 338366, 871984],
["8", 642295, 590821, 529924, 540064, 446368, 450037, 406663, 305061, 306329, 770118],
["7", 569980, 519349, 520255, 496812, 422382, 421909, 363080, 376086, 364594, 650836],
["6", 483983, 447918, 465131, 485883, 345268, 341595, 330906, 361391, 346025, 573873],
["5", 418821, 374268, 441607, 437650, 315884, 315513, 284799, 360712, 329428, 477959],
["4", 341954, 313013, 364007, 384138, 243475, 250063, 253229, 347418, 239839, 396536],
["3", 275946, 254391, 303738, 309478, 211572, 217094, 212804, 358780, 254118, 309294],
["2", 221098, 207912, 270915, 286023, 195459, 178224, 218695, 349499, 204183, 255709],
["1", 157026, 151115, 232975, 220301, 163470, 152116, 168830, 339154, 164269, 187902],
["0", 116729, 110848, 201005, 186903, 132940, 119737, 133592, 309126, 133511, 147195]

    ]);

    return data;
  }

  function getColumns()
  {
    return g_columnsToHide;
  }

  function getColumnsOrig()
  {
    return [0, 1, 2, 3, 5, 6, 8, 10];
  }

  function drawTable(type, div, cols) {
    var data = getData();

    var opts = {
        containerId: div,
        dataTable: data,
        chartType: type,
        options: {"title": "Convert Integer to String"}
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

  function showTable2(type)
  {
    g_chartShownOrig = type;
    var cols = getColumnsOrig();
    drawTable(g_chartShownOrig, "columnchart_origmaterial", cols);
  }

  function showChart()
  {
    showTable(g_chartShown);
  }

  function onloadChart()
  {
    showChart();
    showTable2("Bar");
  }

  function toggleSlow()
  {
    if (null == g_columnsToHide)
    {
      g_columnsToHide = [0, 3, 4, 5, 6, 7, 8, 9];
    }
    else
    {
      g_columnsToHide = null;
    }
    showChart();
  }
</script>
<button onclick='showTable2("Bar")'>Show Bar Chart</button><span> </span>
<button onclick='showTable2("Line")'>Show Line Chart</button><span> </span>
<button onclick='showTable2("Table")'>Show Table</button><span> </span>
<hr />
<div id="columnchart_origmaterial" style="width: 900px; height: 600px;"></div>



### Original Solution Performance

For large number of digits, my solution was faster than the other solutions listed.

But as sample numbers consisted of fewer and fewer digits, between 6 and 3,
the high constant cost of my O(lg N) solution outweighed the speed benefits and the simpler O(N) solutions consumed much less time.

But perhaps I could reduce the large constant cost for smaller numbers...

## Concept #2

For a one digit number, all the multiplies and shifts were a waste of time. Same thing for two to four digit numbers.

Since we're trying to avoid divides, we can use a multiply and shift-right to replace the divide by 100000000.

# Faster solution

```c
uint32_t SmallDecimalToString(uint64_t x, char *s)
{
	if (x <= 9)
	{
		*s = (char)(x | 0x30);
		return 1;
	}
	else if (x <= 99)
	{
		uint64_t low = x;
		uint64_t ll = ((low * 103) >> 9) & 0x1E; low += ll * 3;
		ll = ((low & 0xF0) >> 4) | ((low & 0x0F) << 8);
		*(uint16_t *)s = (uint16_t)(ll | 0x3030);
		return 2;
	}
	return 0;
}

uint32_t SmallUlongToString(uint64_t x, char *s)
{
	uint64_t low;
	uint64_t ll;
	uint32_t digits;

	if (x <= 99)
		return SmallDecimalToString(x, s);

	low = x;
	digits = (low > 999) ? 4 : 3;

	// division and remainder by 100
	// Simply dividing by 100 instead of multiply-and-shift
	// is about 50% more expensive timewise on my box
	ll = ((low * 5243) >> 19) & 0xFF; low -= ll * 100;

	low = (low << 16) | ll;

	// Two divisions by 10 (14 bits needed)
	ll = ((low * 103) >> 9) & 0x1E001E; low += ll * 3;

	// move digits into correct spot
	ll = ((low & 0x00F000F0) << 28) | (low & 0x000F000F) << 40;

	// convert from decimal digits to ASCII number digit range
	ll |= 0x3030303000000000;

	uint8_t *p = (uint8_t *)&ll;
	if (digits == 4) {
		*(uint32_t *)s = *(uint32_t *)(&p[4]);
	} else {
		*(uint16_t *)s = *(uint16_t *)(&p[5]);
		*(((uint8_t *)s) + 2) = *(uint8_t *)(&p[7]);
	}

	return digits;
}

uint32_t UlongToString(uint64_t x, char *s)
{
	uint64_t low;
	uint64_t ll;
	uint32_t digits;

	// 8 digits or less?
	// fits into single 64-bit CPU register
	if (x <= 9999)
	{
		return SmallUlongToString(x, s);
	}
	else if (x < 100000000)
	{
		low = x;

		// more than 6 digits?
		if (low > 999999) {
			digits = (low > 9999999) ? 8 : 7;
		} else {
			digits = (low > 99999) ? 6 : 5;
		}
	}
	else
	{
		uint64_t high = (((uint64_t)x) * 0x55E63B89) >> 57;
		low = x - (high * 100000000);
		// h will be at most 42
		// calc num digits
		digits = SmallDecimalToString(high, s);
		digits += 8;
	}

	ll = (low * 109951163) >> 40; low -= ll * 10000; low |= ll << 32;

	// Four divisions and remainders by 100
	ll = ((low * 5243) >> 19) & 0x000000FF000000FF; low -= ll * 100; low = (low << 16) | ll;

	// Eight divisions by 10 (14 bits needed)
	ll = ((low * 103) >> 9) & 0x001E001E001E001E; low += ll * 3;

	// move digits into correct spot
	ll = ((low & 0x00F000F000F000F0) >> 4) | (low & 0x000F000F000F000F) << 8;
	ll = (ll >> 32) | (ll << 32);

	// convert from decimal digits to ASCII number digit range
	ll |= 0x3030303030303030;

	if (digits >= 8)
	{
		*(uint64_t *)(s + digits - 8) = ll;
	}
	else
	{
		auto d = digits;
		auto s1 = s;
		auto pll = (char *)&(((char *)&ll)[8 - digits]);

		if (d >= 4) {
			*(uint32_t *)s1 = *(uint32_t *)pll;
			
			s1 += 4; pll += 4; d -= 4;
		}
		if (d >= 2) {
			*(uint16_t *)s1 = *(uint16_t *)pll;
			
			s1 += 2; pll += 2; d -= 2;
		}
		if (d > 0) {
			*(uint8_t *)s1 = *(uint8_t *)pll;
		}
	}

	return digits;
}

```


Compiling and running the proposed faster code showed the following results:

<button onclick='showTable("Bar")'>Show Bar Chart</button><span> </span>
<button onclick='showTable("Line")'>Show Line Chart</button><span> </span>
<button onclick='showTable("Table")'>Show Table</button><span> </span>
<label><input type='checkbox' onclick='toggleSlow()' ><span>Hide Slow algorithms</span></label><br />
<hr />
<div id="columnchart_material" style="width: 900px; height: 600px;"></div>


### Faster Solution Performance

The faster solution performs much better at smaller digits than the original solution - it's almost equal in performance except for 3-digit numbers.

I tried replacing the division by 100 in the facebook algorithms with multiply and shift-right operations to see if it would help performance.

There does seem to be a reduction in runtime for larger digit numbers, but by 6-8 digit numbers, there's hardly any difference.
