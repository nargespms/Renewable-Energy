---
theme: dashboard
title: Home
toc: false
---

<!-- Load the data -->

```js
// Load the economic impact data and the GeoJSON map
const impactData = FileAttachment("./data/economicImpact.json").json();
const canadaMap = FileAttachment("./data/canadaProvinces.geojson").json();

const electAllTotal = FileAttachment("./data/elect-allTotal.csv").csv({
  typed: true,
});
const electTotal = FileAttachment("./data/elect-total.csv").csv({
  typed: true,
});
const electSolar = FileAttachment("./data/elect-solar.csv").csv({
  typed: true,
});
const electTidal = FileAttachment("./data/elect-tidal-power-turbine.csv").csv({
  typed: true,
});
const electWind = FileAttachment("./data/elect-wind-power-turbine.csv").csv({
  typed: true,
});
const electOther = FileAttachment("./data/elect-other.csv").csv({
  typed: true,
});
const electHydraulic = FileAttachment("./data/elect-hydraulic-turbine.csv").csv(
  { typed: true }
);
```

```js
const currentGeo = Mutable("");
const updateGeo = (val) => {
  if (val === currentGeo.value) {
    currentGeo.value = "Canada";
  } else {
    currentGeo.value = val;
  }
};
```

```js
const geoInput = Inputs.select(
  [
    "Canada",
    "Newfoundland and Labrador",
    "Prince Edward Island",
    "Nova Scotia",
    "New Brunswick",
    "Quebec",
    "Ontario",
    "Manitoba",
    "Saskatchewan",
    "Alberta",
    "British Columbia",
    "Yukon Territory",
    "Northwest Territories",
    "Nunavut",
  ],
  { value: currentGeo, label: "Select Location", onChange: console.log }
);
const selectedGeo = view(geoInput);
```

```js
function toNum(val) {
  if (typeof val === "number") return val;
  return parseFloat(val.replace(",", ""));
}
```

```js
function getPercentageByProvinces(year = "2012") {
  const data = electTotal.slice(1);
  const allData = electAllTotal.slice(1);
  const map = new Map(
    data.map((d, i) => [
      d.Geography,
      Math.round((toNum(d[year]) / toNum(allData[i][year])) * 100),
    ])
  );

  canadaMap.features.forEach((g) => {
    g.properties.percentage = map.get(g.properties.name);
  });
  return canadaMap;
}
```

```js
function on(mark, listeners = {}) {
  const render = mark.render;
  mark.render = function (facet, { x, y }, channels) {
    const data = this.data;

    if (x && x.invert === undefined)
      x.invert = d3.scaleQuantize(x.range(), x.domain());
    if (y && y.invert === undefined)
      y.invert = d3.scaleQuantize(y.range(), y.domain());

    const g = render.apply(this, arguments);
    const r = d3.select(g).selectChildren();
    for (const [type, callback] of Object.entries(listeners)) {
      r.on(type, function (event, i) {
        const p = d3.pointer(event, g);
        callback(event, {
          type,
          p,
          datum: data[i],
          i,
          facet,
          data,
          ...(x && { x: x.invert(p[0]) }),
          ...(y && { y: y.invert(p[1]) }),
          ...(x && channels.x2 && { x2: x.invert(channels.x2[i]) }),
          ...(y && channels.y2 && { y2: y.invert(channels.y2[i]) }),
        });
      });
    }
    return g;
  };
  return mark;
}
```

```js
function renderMap(year, width, height) {
  const provinces = getPercentageByProvinces(year).features;
  const geo = Plot.geo(provinces, {
    fill: (d) => d.properties.percentage,
    stroke: (d) => (d.properties.name === selectedGeo ? "Orange" : "Green"),
    strokeWidth: (d) => (d.properties.name === selectedGeo ? 3 : 0),
  });

  return Plot.plot({
    projection: ({ width, height }) =>
      d3
        .geoConicEquidistant()
        .rotate([-86, -129.5, -170])
        .translate([width / 2, height / 2])
        .scale(width * 1.1),
    color: {
      scheme: "Greens",
      legend: true,
      type: "linear",
      label: "Gigawatt hours",
      domain: [0, 100], // Only include positive values in the color scale domain
    },
    width,
    height: height * 1.4,

    marks: [
      on(geo, {
        click: function (e, { datum }) {
          updateGeo(datum.properties.name);
        },
        mouseover: function (e, { datum }) {
          if (datum.properties.name === selectedGeo) return;
          e.target.setAttribute("stroke-width", "2px");
          e.target.setAttribute("stroke", "Blue");
        },
        mouseout: function (e, { datum }) {
          if (datum.properties.name === selectedGeo) return;
          e.target.setAttribute("stroke-width", "0");
          e.target.style.transform = "scale(1)";
        },
      }),
      Plot.tip(
        provinces,
        Plot.pointer(
          Plot.centroid({
            title: (d) => `${d.properties.name}: ${d.properties.percentage} %`,
          })
        )
      ),
    ],
  });
}
```

```js
function getData(geo = "", year = "2022") {
  if (geo === "") geo = "Canada";

  // Load data from each source
  const solar = electSolar.filter((d) => d.Geography === geo)[0][year];
  const tidal = electTidal.filter((d) => d.Geography === geo)[0][year];
  const wind = electWind.filter((d) => d.Geography === geo)[0][year];
  const other = electOther.filter((d) => d.Geography === geo)[0][year];
  const hydraulic = electHydraulic.filter((d) => d.Geography === geo)[0][year];

  return [
    { source: "Solar", value: toNum(solar) },
    { source: "Tidal", value: toNum(tidal) },
    { source: "Wind", value: toNum(wind) },
    { source: "Other", value: toNum(other) },
    { source: "Hydraulic", value: toNum(hydraulic) },
  ];
}
```

```js
function getFilteredData(geo = "Canada") {
  const years = d3.range(2012, 2023); // From 2012 to 2022
  const sources = ["Solar", "Tidal", "Wind", "Other", "Hydraulic"];

  // Helper function to extract and convert data
  const extractValue = (data, year) => {
    const entry = data.find((d) => d.Geography === geo);
    return entry && entry[year] ? toNum(entry[year]) : 0;
  };

  let formattedData = [];

  // Iterate over each year
  for (let year of years) {
    sources.forEach((source) => {
      const value = extractValue(
        {
          Solar: electSolar,
          Tidal: electTidal,
          Wind: electWind,
          Other: electOther,
          Hydraulic: electHydraulic,
        }[source],
        year
      );

      formattedData.push({
        year,
        source,
        mwh: value,
      });
    });
  }

  return formattedData;
}
```

```js
// Function to draw an area chart with the filtered data
function drawAreaChart(geo, width, height) {
  const data = getFilteredData(geo);
  return Plot.plot({
    width,
    height,
    x: {
      label: "Year →",
      tickFormat: (d) => d.toString(),
    },
    y: {
      label: "Electricity Generated (MWh) →",
      grid: true,
    },
    marks: [
      Plot.areaY(data, {
        x: "year",
        y: "mwh",
        fill: "source",
        order: "sum",
        tip: true,
        opacity: 0.6,
      }),
    ],
    color: {
      scheme: "Observable10",
      legend: true, // Adds a legend to distinguish sources
    },
  });
}
```

```js
function drawChart(geo, width, height) {
  return Plot.plot({
    width,
    height,
    marks: [
      Plot.barY(getData(geo, year), {
        x: "source",
        y: "value",
        fill: "source",
        tip: true,
      }),
    ],
    color: {
      scheme: "Observable10",
      legend: true,
      width: 340,
      label: "Source",
    },
    x: {
      label: "Electricity Source",
    },
    y: {
      label: "Generation (in MWh)",
      grid: true,
    },
  });
}
```

```js
const input = Inputs.range([2012, 2022], {
  step: 1,
  value: 2012,
  label: "Year",
});
const year = view(input);
```

```js
const provinceMap = {
  Alberta: "AB",
  "British Columbia": "BC",
  Manitoba: "MB",
  "New Brunswick": "NB",
  "Newfoundland and Labrador": "NL",
  "Nova Scotia": "NS",
  Ontario: "ON",
  "Prince Edward Island": "PE",
  Quebec: "QC",
  Saskatchewan: "SK",
  "Northwest Territories": "NT",
  Nunavut: "NU",
  "Yukon Territory": "YT",
};

function getGradientColorWithTextColor(color, percentage) {
  // Ensure the percentage is between 0 and 1
  percentage = Math.max(0, Math.min(1, percentage));

  // Converts hex color to RGB
  const hexToRgb = (hex) => {
    hex = hex.replace(/^#/, "");
    const bigint = parseInt(hex, 16);
    const r = (bigint >> 16) & 255;
    const g = (bigint >> 8) & 255;
    const b = bigint & 255;
    return { r, g, b };
  };

  // Converts RGB to hex
  const rgbToHex = (r, g, b) => {
    return (
      "#" +
      [r, g, b]
        .map((x) => {
          const hex = x.toString(16);
          return hex.length === 1 ? "0" + hex : hex;
        })
        .join("")
    );
  };

  // Linear interpolation between white (255,255,255) and the target color
  const interpolateColor = (color, percentage) => {
    const { r, g, b } = hexToRgb(color);
    const newR = Math.round((255 - r) * (1 - percentage) + r);
    const newG = Math.round((255 - g) * (1 - percentage) + g);
    const newB = Math.round((255 - b) * (1 - percentage) + b);
    return { r: newR, g: newG, b: newB };
  };

  // Calculate luminance
  const luminance = (r, g, b) => {
    const a = [r, g, b].map(function (v) {
      v /= 255;
      return v <= 0.03928 ? v / 12.92 : Math.pow((v + 0.055) / 1.055, 2.4);
    });
    return a[0] * 0.2126 + a[1] * 0.7152 + a[2] * 0.0722;
  };

  // Get the interpolated RGB color
  const { r, g, b } = interpolateColor(color, percentage);
  const hexColor = rgbToHex(r, g, b);

  // Decide text color based on luminance
  const textLuminance = luminance(r, g, b);
  const textColor = textLuminance > 0.179 ? "black" : "white";

  return {
    backgroundColor: hexColor,
    textColor: textColor,
  };
}

function getTable(year, id) {
  const provinces = getPercentageByProvinces(year)
    .features.map((p) => ({
      name: p.properties.name,
      percentage: p.properties.percentage,
    }))
    .sort((a, b) => b.percentage - a.percentage);
  let html = "";

  provinces.forEach((p) => {
    p.code = provinceMap[p.name];
    p.color = getGradientColorWithTextColor("#1A4320", p.percentage / 100);

    html += `<td style="background-color:${p.color.backgroundColor}; color: ${p.color.textColor}" title="${p.name}">${p.code}</td>`;
  });

  document.getElementById(id).innerHTML = html;

  return "";
}
```

<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0-beta3/css/all.min.css">

<div class="dashboardTitle">
  <h1>
  Renewable Energy Transition Progress in Canada
  </h1>
</div>
<div class="mainWrapper">
  <div class="grid grid-cols-2">
   <div class="filterSec">
      <div>${geoInput}</div>
      <div>${input}</div>
    </div>
    <div>
    <h4 class="mb-8">Order of Transition as of ${year} <i class="fa-solid fa-arrow-right ml-4"></i></h4>
    <table>
      <tr id="provincesTable">
        ${getTable(year, 'provincesTable') }
      </tr>
    </table>
    </div>
   
  </div>

  <div class="grid grid-cols-2 clearBoth">
  <!-- Geo Map -->
    <div class="card relative GeoMap">
    <div class="tooltip"> 
        <i class="fa fa-circle-info" style="font-size:13px;"></i>
        <span class="tooltiptext">This Geo Map shows the Percentage of renewable generated electricity in each province</span>
      </div>
      <h3 class="center">% Of Renewable Electricity Generated (${year})</h3>
      ${resize((width) => renderMap(year, width, width * 0.5 ))} 
    </div>
    <div>
    <!-- BarChart -->
      <div class="card relative marginTop0">
        <div class="tooltip">
        <i class="fa fa-circle-info" style="font-size:13px;"></i>
        <span class="tooltiptext">This bar chart displays the composition of electricity generation by source within the selected geographical region, such as Canada or a specific province, for a given year.   </span>
        </div>
        <h3 class="center">Electricity Generation Sources in
          <span class="selectedGeo">${selectedGeo}</span> (${year})<span> (MWh)</span>
        </h3>
        ${resize((width) => drawChart(selectedGeo, width, width * 0.3 ))}
      </div>
      <!-- Area chart -->
      <div class="card relative marginBottom0">
        <div class="tooltip">
          <i class="fa fa-circle-info" style="font-size:13px;"></i>
          <span class="tooltiptext">This area chart illustrates the trend of electricity generated from different sources in Canada over a decade, measured in megawatt-hours (MWh).  </span>
        </div>
        <h3 class="center">Trends in Electricity Generation by Source in 
        <span class="selectedGeo">${selectedGeo}</span>
        <span> (MWh)</span>
        </h3>
        ${resize((width) => drawAreaChart(selectedGeo, width, width * 0.3 ))}
      </div>
    </div>

  </div>
</div>

<style>
  body {
    font-family:initial;
  }
  #observablehq-main {
   margin-top: 12px; 
  }
  .clearBoth {
    clear:both;
  }
  .filterSec {
    margin-bottom: 12px;
  }
  .mb-8 {
    margin-bottom:8px;
  }
  .mb-16 {
    margin-bottom:16px;
  }
  .ml-4 {
    margin-left:4px;
  }
  .filterSec label {
    font-weight: 700;
  }
  .marginTop0 {
    margin-top:0;!important
  }
  .marginBottom0 {
    margin-bottom:0;!important
  }
  
  .card {
    background-color:#e8e8e8;
  }
  .card h3 {
    padding: 0 16px 8px 16px;
    color: #000;
    font-weight:700;
    font-size:22px;

  }
  .card .selectedGeo {
    color:#4269d0;
  }
.relative {
  position:relative;
}
.dashboardTitle {
  margin-bottom:12px;
  text-align:center;
  width:100%;
}
.dashboardTitle h1 {
  margin:auto;
  max-width:80%
}
  .center {
    margin: 0;
    text-align: center;
  }
  .card h2 {
    font-size: 35px;
    margin-bottom: 24px;
  }
  .inputs-3a86ea-input > input[type=range] {
    cursor:pointer;

}
  input [type=range] {
    cursor:pointer;
  }

  /* tooltip */
  .tooltip {
  position: absolute;
  top:12px;
  right:12px;
  display: inline-block;
  cursor:help;
}

.tooltip .tooltiptext {
  visibility: hidden;
  width: 320px;
  background-color: black;
  color: #fff;
  text-align: center;
  border-radius: 6px;
  padding: 5px 12px;

  /* Positioning */
  position: absolute;
  z-index: 1;
  bottom: 100%; /* Adjust this if you want more space between the icon and the tooltip */
  right : 0%; /* Start from the far left */
  
  
  /* Visibility transition */
  opacity: 0;
  transition: opacity 0.3s, visibility 0.3s;
}

.tooltip:hover .tooltiptext {
  visibility: visible;
  opacity: 1;
}


.mainWrapper {
    padding: 24px;
    background-color: #b3b3b3;
    border-radius: 9px;
  }

  /* table */
  table {
    width: 100%;
    border-collapse: collapse;
    margin:0;
  }
  th, td {
    border: 1px solid black;
    padding: 5px;
    text-align: center;
    cursor:default;
  }
  th {
    background-color: #f2f2f2;
  }
  .GeoMap svg path {
    cursor:pointer;
  }
  .card {
    font-family:initial;
  }

  </style>
