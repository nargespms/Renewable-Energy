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
  { value: currentGeo, label: "Geo", onChange: console.log }
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
// TODO revise with Plot’s render transform (0.6.10)
function on(mark, listeners = {}) {
  const render = mark.render;
  mark.render = function (facet, { x, y }, channels) {
    // data[i] may or may not be the datum, depending on transforms
    // (at this stage we only have access to the materialized channels we requested)
    // but in simple cases it works
    const data = this.data;

    //  since a point or band scale doesn't have an inverse, one from its domain and range
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
        .scale(width * 0.9),
    color: {
      scheme: "Greens",
      legend: true,
      type: "linear",
      label: "Gigawatt hours",
      domain: [0, 100], // Only include positive values in the color scale domain
    },
    width,
    height,

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
      scheme: "Spectral",
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

<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0-beta3/css/all.min.css">
 <!-- <div class="tooltip">
    <i class="fa fa-circle-info" style="font-size:20px;"></i>
    <span class="tooltiptext">Tooltip text here</span>
  </div> -->

<div class="dashboardTitle">

<h1>
Renewable Energy Transition Progress in Canada
</h1>
</div>

<div>${geoInput}</div>
<div>${input}</div>

<div class="grid grid-cols-2" style="">
  <div class="card relative">
    <div class="tooltip"> 
      <i class="fa fa-circle-info" style="font-size:20px;"></i>
      <span class="tooltiptext">This Geo Map shows the Percentage of renewable generated electricity in each province</span>
    </div>
    <h2 class="center"> % Of Renewable Electricity (${year}) </h2>
    ${resize((width) => renderMap(year, width, width * 0.6 ))}

  </div>

  <div class="card">
    <h2 class="center">Electricity Generation Sources in 
    <span class="selectedGeo">${selectedGeo}</span> (${year})
    <span> (MWh)</span>
    </h2>
    ${resize((width) => drawChart(selectedGeo, width, width * 0.6))}
  </div>
</div>

<style>
.card .selectedGeo {
  color:#4269d0;
}
.relative {
  position:relative;
}
.dashboardTitle {
  /* margin:auto; */
  text-align:center;
  width:100%;
}
.dashboardTitle h1 {
  margin:auto;
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
}

.tooltip .tooltiptext {
  visibility: hidden;
  width: 120px;
  background-color: black;
  color: #fff;
  text-align: center;
  border-radius: 6px;
  padding: 5px 0;
  position: absolute;
  z-index: 1;
  bottom: 150%;
  left: 50%;
  margin-left: -60px;
  opacity: 0;
  transition: opacity 0.3s;
}

.tooltip:hover .tooltiptext {
  visibility: visible;
  opacity: 1;
}
  </style>
