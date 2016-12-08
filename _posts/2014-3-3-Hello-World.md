---
layout: post
title: You're up and running!
---

Next you can update your site name, avatar and other options using the _config.yml file in the root of your repository (shown below).

![_config.yml]({{ site.baseurl }}/images/config.png)

The easiest way to make your first post is to edit this one. Go into /_posts/ and update the Hello World markdown file. For more instructions head over to the [Jekyll Now repository](https://github.com/barryclark/jekyll-now) on GitHub.


<script type="text/javascript" src="https://www.gstatic.com/charts/loader.js"></script>
<script type="text/javascript">

      var g_columnsToHide = null;
      var g_chartShown = "Bar";

      google.charts.load('current', {'packages':['line', 'bar', 'table']});
      //google.charts.load('current', {'packages':['bar']});
      google.charts.setOnLoadCallback(showChart);

      function getData()
      {
        var data = google.visualization.arrayToDataTable([
          ['Digits', 'lwan', 'lwan_lut', 'facebook_digits10', 'facebook_digits10mls', 
           'facebook_fixed', 'facebook_fixed_small', 'facebook_fixed_mls', 
           'UlongToStringOrig', 'UlongToString', 'naive' ],
          [10, 5214597, 4388672, 2474180, 1572706, 2256673, 2315170, 1346498, 1797105, 1588024, 4353699 ],
          [ 9, 4099783, 3463241, 2375665, 1503173, 2168089, 2214055, 1287407, 1657223, 1395933, 3833761 ],
          [ 8, 3620056, 3060712, 1927991, 1255730, 1698211, 1753723, 1036478, 1340737, 1340815, 3428030 ],
          [ 7, 3148311, 2670748, 1917656, 1241670, 1692842, 1742513, 1035170, 1445077, 1433557, 2942412 ],
          [ 6, 2688072, 2297181, 1498133, 1027440, 1217821, 1248814,  765853, 1359173, 1345006, 2560219 ],
          [ 5, 2234748, 1925786, 1487737, 1034546, 1204004, 1247053,  749087, 1339194, 1337977, 2122267 ],
          [ 4, 1775201, 1558691,  968609,  773198,  733284,  737946,  503941, 1138708,  686512, 1746409 ],
          [ 3, 1321050, 1196462,  837773,  608084,  707432,  739894,  486244, 1179078,  720794, 1316359 ],
          [ 2,  922211,  854352,  322532,  339712,  213774,  213000,  211474, 1148852,  378430,  957693 ],
          [ 1,  635005,  589126,  330470,  342688,  224308,  205260,  215061, 1176439,  258919,  630357 ],
          [ 0,  507973,  463477,  287185,  289646,  177525,  163073,  173757, 1129431,  190040,  511062 ]

        ]);

        return data;
      }

      function getColumns()
      {
        return g_columnsToHide;
      }

      function drawTable() {
        var data = getData();

        var opts = {
            "containerId": "columnchart_material",
            "dataTable": data,
            "chartType": g_chartShown,
            "options": {"title": "Convert Integer to String", 
                        "subtitle": 'SWAR and Multiply-Shift techniques'}
        }
        var chartwrapper = new google.visualization.ChartWrapper(opts);

        var columns = getColumns();
        if (null != columns)
          chartwrapper.setView({'columns': columns});

        chartwrapper.draw();
      }

      function showLine()
      {
        g_chartShown = "Line";
        drawTable();
      }

      function showBar()
      {
        g_chartShown = "Bar";
        drawTable();
      }

      function showTable()
      {
        g_chartShown = "Table";
        drawTable();
      }

      function showChart()
      {
        if (g_chartShown == "Line")
        {
          showLine();
        }
        else if (g_chartShown ==  "Bar")
        {
          showBar();
        }
        else
        {
          showTable();
        }
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
<div>
<button onclick='showBar()'>Show Bar Chart</button><span> </span>
<button onclick='showLine()'>Show Line Chart</button><span> </span>
<button onclick='showTable()'>Show Table</button><span> </span>
<label><input type='checkbox' onclick='toggleSlow()' ><span>Hide Slow algorithms</span></label>
<hr />
<div id="columnchart_material" style="width: 1024px; height: 800px;"></div>
</div>
