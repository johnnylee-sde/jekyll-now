---
layout: post
title: Fast unsigned integer to time string
---

Finishing off the output trilogy, a fast solution to converting unsigned 32-bit integers to time strings of the form HH:MM:SS.

# Concept #1

Use multiply and bitwise-shifts-right instead of divide operations by 3600 and 60  to calculate the hours, miniutes, and seconds.

# Concept #2

Convert the hours, minute, and seconds to string by bitwise-ORing with 0x30 where the numbers are located 
and with 0x3A, ':' to position the colon separators.

```c
uint32_t UlongToTimeString(uint64_t secs, char *s)
{
	// divide by 3600 to calculate hours
	uint64_t hours = (secs * 0x91A3) >> 27;
	uint64_t xrem = secs - (hours * 3600);

	// divide by 60 to calculate minutes
	uint64_t mins = (xrem * 0x889) >> 17;
	xrem = xrem - (mins * 60);

	// position hours, minutes, and seconds in one var
	uint64_t timeBuffer = hours + (mins << 24) + (xrem << 48);

	// convert to decimal representation
	xrem = ((timeBuffer * 103) >> 9) & 0x001E00001E00001E;
	timeBuffer += xrem * 3;

	// move high nibbles into low mibble position in current byte
	// move lower nibble into left-side byte 
	timeBuffer = ((timeBuffer & 0x00F00000F00000F0) >> 4) |
		((timeBuffer & 0x000F00000F00000F) << 8);

	// bitwise-OR in colons and convert numbers into ASCII number characters
	timeBuffer |= 0x30303A30303A3030;

	// copy to buffer
	*(uint64_t *)s = timeBuffer;

	return 0;
}

```

I'll use a modified benchmark harness from
[Fast unsigned integer to string]({{ site.baseurl }}{% post_url 2016-12-8-Fast-unsigned-integer-to-string %})
to test for integer-to-time conversion.

But we'll need other alternative solutions.

Easiest one is to lean on the standard library:

```c
uint32_t stdlibUlongToTimeString(uint64_t secs, char *s)
{
	uint64_t hours = (secs / 3600);
	uint64_t xrem = secs - (hours * 3600);
	// divide by 60 to calculate minutes
	uint64_t mins = (xrem / 60);
	xrem -= (mins * 60);
	sprintf_s(s, 9, "%2.2d:%2.2d:%2.2d", (int)hours, (int)mins, (int)xrem);

	return 0;
}

```

Next is a simple naive solution:

```c
uint32_t naiveUlongToTimeString(uint64_t secs, char *s)
{
	uint64_t hours = (secs / 3600);
	uint64_t xrem = secs - (hours * 3600);
	// divide by 60 to calculate minutes
	uint64_t mins = (xrem / 60);
	xrem -= (mins * 60);

	s[0] = (char)((hours / 10) + '0');
	s[1] = (char)((hours % 10) + '0');
	s[2] = ':';
	s[3] = (char)((mins / 10) + '0');
	s[4] = (char)((mins % 10) + '0');
	s[5] = ':';
	s[6] = (char)((xrem / 10) + '0');
	s[7] = (char)((xrem % 10) + '0');

	return 0;
}

```

Let's remove the divide operations and use multiply-shift instead:

```c
uint32_t fastUlongToTimeString(uint64_t secs, char *s)
{
	// divide by 3600 to calculate hours
	uint64_t hours = (secs * 0x91A3) >> 27;
	uint64_t xrem = secs - (hours * 3600);

	// divide by 60 to calculate minutes
	uint64_t mins = (xrem * 0x889) >> 17;
	xrem = xrem - (mins * 60);

	s[0] = (char)((hours / 10) + '0');
	s[1] = (char)((hours % 10) + '0');
	s[2] = ':';
	s[3] = (char)((mins / 10) + '0');
	s[4] = (char)((mins % 10) + '0');
	s[5] = ':';
	s[6] = (char)((xrem / 10) + '0');
	s[7] = (char)((xrem % 10) + '0');

	return 0;
}
```

On to the results:

<script type="text/javascript" src="https://www.gstatic.com/charts/loader.js"></script>
<script type="text/javascript">

  var g_columnsToHide = null;
  var g_chartShown = "Bar";

  google.charts.load('current');
  google.charts.setOnLoadCallback(onloadChart);

  function getData()
  {
    var data = google.visualization.arrayToDataTable([
      ['algorithm', 'stdlib', 'naive', 'fast', 'UlongToTimeString' ],

      // 64-bit x64 runtime results
['time', 21565428, 354223, 319178, 197627  ]

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
        options: {"title": "Convert Integer to Time String"}
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
      g_columnsToHide = [0, 2, 3];
    }
    else
    {
      g_columnsToHide = null;
    }
    showChart();
  }


</script>
<button onclick='showTable("Bar")'>Show Bar Chart</button><span> </span>
<button onclick='showTable("Table")'>Show Table</button><span> </span>
<label><input type='checkbox' onclick='toggleSlow()' ><span>Hide Slow algorithm</span></label>
<hr />
<div id="columnchart_material" style="width: 900px; height: 600px;"></div>


# Performance

The standard library solution is outclassed by the others.

Not much to say about the naive solution. Reasonably small and fast.

Remove the divide operations for the fast solution reduces runtime by about 10%.

My solution turned out to be the fastest - about 56% of the naive solution's running time.

And we're done. Phew.

