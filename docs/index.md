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

const electricity = FileAttachment("./data/electricityProduction.csv").csv({
  typed: true,
});
```

```js
const getImpactsByProvinces = () => {
  const map = new Map(impactData.map((d) => [d.province, +d.impact]));
  canadaMap.features.forEach((g) => {
    g.properties.impactRate = map.get(g.properties.name);
  });
  return canadaMap;
};
```

```js
function renderMap(width, height) {
  const provinces = getImpactsByProvinces().features;

  return Plot.plot({
    projection: ({ width, height }) =>
      d3
        .geoConicEquidistant()
        .rotate([-86, -129.5, -170])
        .translate([width / 2, height / 2])
        .scale(width * 1.1),
    color: {
      scheme: "rdylgn",
      legend: true,
      tickFormat: "d",
      n: 6,
      label: "Impact Rate (%)",
    },
    width,
    height,

    marks: [
      Plot.geo(provinces, {
        fill: (d) => d.properties.impactRate,
        opacity: 0.8,
      }),

      Plot.tip(
        provinces,
        Plot.pointer(
          Plot.centroid({
            title: (d) => `${d.properties.name}: ${d.properties.impactRate}%`,
          })
        )
      ),
    ],
  });
}
```

```js
const tidy = electricity.columns.slice(1).flatMap((source) =>
  electricity.map((d) => ({
    province: d.ProvinceCode,
    source,
    production: d[source],
  }))
);
```

```js
function renderStackBar(width, height) {
  return Plot.plot({
    width,
    height,
    x: { label: "ProvinceCode" },
    color: {
      scheme: "Spectral",
      legend: true,
      width: 340,
      label: "Source",
    },
    marks: [
      Plot.barY(tidy, {
        tip: true,
        x: "province",
        y: "production",
        fill: "source",
        sort: { color: null, y: "-x", reduce: "first" },
      }),
    ],
  });
}
```

<div class="grid grid-cols-2" style="">
  <div class="card">
    <h2 class="center">Economic Impact</h2>
    ${resize((width) => renderMap(width, width * 0.8))}
  </div>

  <div class="card">
    <h2 class="center">Electricity Generation Source
    <span> (GWh)</span>
    </h2>
    ${resize((width) => renderStackBar(width, width * 0.8))}
  </div>
</div>

<style>
  .center {
    margin: 0;
    text-align: center;
  }
  .card h2 {
    font-size: 35px;
    margin-bottom: 24px;

  }
  </style>
