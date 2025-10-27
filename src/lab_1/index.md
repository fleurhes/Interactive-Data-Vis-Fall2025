---
title: "Lab 1: Passing Pollinators"
toc: true
---

# Lab 1 — Passing Pollinators
## Question 1: What is the body mass and wing span distribution of each pollinator species observed?

```js
// load data
const pollinators = FileAttachment("./data/pollinator_activity_data.csv").csv({ typed: true })
```

```js
Plot.plot({
  title: "Body Mass vs. Wing Span by Pollinator Species",
  height: 400,
  grid: true,
  marks: [
    Plot.frame(),
    Plot.dot(pollinators, {
      x: "avg_body_mass_g",
      y: "avg_wing_span_mm",
      stroke: "pollinator_species",
      fill: "pollinator_species",
      r: 4,
      opacity: 0.8,
      tip: true
    })
  ],
  x: {label: "Average Body Mass in grams", domain: [0, 0.6]},
  y: {label: "Average Wing Span in mms", domain: [10, 55]},
  color: {legend: true, label: "Species"}
})
```
Across polinator species, honeybees are consistently the smallest follwed by bumblebees making most carpenter bees the largest species. Although being the largest species, carpenter bees are the most widely clustered in size with one bee appearing as smaller in wing span than a large bumblebee. 

<br>

## Question 2: What is the ideal weather (conditions, temperature, etc) for pollinating?

```js
//weather on polination plot
Plot.plot({
  title: "Weather’s impact on pollination",
  subtitle: "Dot size = visit count; color = weather condition; Humidity/Wind speed on hover, Use dropdown to change between Humidity and Wind speed",
  height: 400,
  grid: true,
  x: { label: xAxis.label, nice: true },
  y: { label: "Temperature (°C)" },
  color: { type: "categorical", label: "Weather Condition" },
  marks: [
    Plot.frame(),
    Plot.dot(pollinators, {
      x: d => d[xAxis.value],
      y: "temperature",
      fill: "weather_condition",
      stroke: "weather_condition",
      r: "visit_count",
      tip: true
    })
  ]
})
```
```js
// selector for humidity or wind speed
const xOptions = [
  { label: "Humidity (%)", value: "humidity" },
  { label: "Wind Speed (km/h)", value: "wind_speed" }
];
const xAxis = view(Inputs.select(xOptions, {
  label: "Humidity/Windspeed selector",
  value: xOptions[0],
  format: d => d.label
}));
```

Humidity does not appear to have a major impact on pollinating, Wind speed however, appears to impact pollination habits with most pollinators chosing to pollinate on less windy, mid to high 20 degree weather.  

<br>

## Question 3: Which flower has the most nectar production?
```js
Plot.plot({
  title: "Nectar Production by Flower",
  height: 360,
  marginLeft: 120,
  grid: true,
  marks: [
    // individual observations, arranged along y categories (swarm along x)
    Plot.dotX(pollinators, {
      x: "nectar_production",
      y: "flower_species",
      r: 2.5,
      fill: "flower_species",
      opacity: 0.6,
      tip: true
    }),
    // median tick per flower
    Plot.tickX(
      pollinators,
      Plot.groupY({x: "median"}, {y: "flower_species", x: "nectar_production", stroke: "#333", strokeWidth: 2})
    ),
    // mean dot per flower
    Plot.dotX(
      pollinators,
      Plot.groupY({x: "mean"}, {y: "flower_species", x: "nectar_production", r: 5, fill: "#333"})
    ),
    Plot.frame()
  ],
  x: {label: "Nectar Production (μL)"},
  y: {label: "Flower Species"},
  color: {legend: false}
})
```
The sunflower has the highest overall nectar production. The lavender and coneflowers share similar distribution of nectar production with the cone flower having a higher average and max production.