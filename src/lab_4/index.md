---
title: "Lab 4: Clearwater Crisis"
toc: true
---

<style>
h2 {
  font-style: italic !important;     
  font-size: 1.5em !important;        
  font-weight: 400 !important;        
  margin-top: .5em !important;       
  margin-bottom: 0.5em !important;
  line-height: 1.5 !important;
}
</style>

<style>
h3 {
  font-style: bold !important;      
  font-size: 1.25em !important;       
  font-weight: 400 !important;       
  color: #000000ff !important;            
  margin-top: 1em !important;       
  margin-bottom: 0.5em !important;
  line-height: 1.5 !important;
}
</style>

---
# Lab 4 — Clearwater Crisis Investigation
---

Lake Clearwater's fish populations collapsed over the past two years. Four potential causes for the reduction in population sit on Lake Clearwater: An industrial facility, a farm, a resort, and a fishing lodge. This dashboard uses water quality data, fish populaton surveys, activity records, and monitoring station data to determine who is responsible for the drop in population and what it is impacting the most.

---

## 1. Where Is the Problem?

```js
const waterQuality = await FileAttachment("data/water_quality.csv").csv({ typed: true });
const fishSurveys = await FileAttachment("data/fish_surveys.csv").csv({ typed: true });
const stations = await FileAttachment("data/monitoring_stations.csv").csv({ typed: true });
const activities = await FileAttachment("data/suspect_activities.csv").csv({ typed: true });
```

```js
const stationOrder = ["North", "South", "East", "West"];

const troutByStation = stationOrder.map(station => {
  const stationTrout = fishSurveys
    .filter(d => d.station_id === station && d.species === "Trout")
    .sort((a, b) => new Date(a.date) - new Date(b.date));
  
  const first = stationTrout[0].count;
  const last = stationTrout[stationTrout.length - 1].count;
  const change = last - first;
  const pctChange = ((change / first) * 100).toFixed(0);
  
  return { station, first, last, change, pctChange: +pctChange };
});
```

The fish survey data has quarterly population counts for all three fish species at the four monitoring stations around the lake. According to the survey data, trout are classified as highly sensitive to pollution, making them the most likely population to witness impact.

### Trout Population Change by Station 

<div style="margin-top: 10px; margin-bottom: 20px; color: #666; font-size: .7em;">
  Change in total trout count over the two-year study period. Hover for population counts.
</div>

```js
Plot.plot({
  height: 180,
  width: Math.min(650, width),
  marginLeft: 65,
  marginRight: 40,
  marginBottom: 40,
  x: {
    label: "← Decline · Growth →",
    grid: true,
    domain: [-42, 10]
  },
  y: {
    label: null,
    domain: stationOrder.slice().reverse()
  },
  marks: [
    Plot.barX(troutByStation, {
      x: "change",
      y: "station",
      fill: d => d.station === "West" ? "#ff0d00ff" : "#5900ffff",
      inset: 8,
      tip: {
        format: { x: false, fill: false },
        channels: {
          "Station": "station",
          "Jan 2023": "first",
          "Oct 2024": "last",
          "Net Change": d => (d.change > 0 ? "+" : "") + d.change
        }
      }
    }),
    Plot.ruleX([0], { stroke: "#333", strokeWidth: 1 }),
    Plot.text(troutByStation, {
      x: d => d.change < 0 ? d.change - 6 : d.change + 6,
      y: "station",
      text: d => d.change + " (" + d.pctChange + "%)",
      textAnchor: d => d.change < 0 ? "end" : "start",
      fontSize: 10,
      fontWeight: d => d.station === "West" ? "bold" : "normal",
      fill: d => d.station === "West" ? "#ff0d00ff" : "#333"
    })
  ]
})
```

<div style="margin-top: 40px; margin-bottom: 20px; font-size: 1em;">
  <b>The trout population fell most drastically at the West station, with more moderate loss observed at the East and South stations.</b> <br>Trout at the West station dropped from 43 to 13 or a 70% decline from January 2023 to October 2024. This drastic population change at the West station alludes to a major contamination source falling near the West station
</div>

<div style="padding: 15px; background: #f5f5f5; border-radius: 4px; font-family: monospace; font-size: 0.9em; margin-bottom: 20px;">
  West Station: 43 fish (Jan 2023) → 13 fish (Oct 2024) = <b style="color: #ff0d00ff;">−30 fish (−70%)</b><br>
  North Station: 40 fish (Jan 2023) → 38 fish (Oct 2024) = −2 (−5%) <br>South Station: 40 fish (Jan 2023) → 34 fish (Oct 2024) = −6 (−15%) <br>East Station: 42 fish (Jan 2023) → 37 fish (Oct 2024) = −5 (−12%)
</div>

---

## 2. Who Operates Near West Station?

Monitoring stations are placed near the four possible culprits; one on each cardinal direction of the lake. The source file `monitoring_stations.csv` includes exact distance measurements from each monitoring station to the culprit.

```js
const westDistances = [
  { suspect: "ChemTech Manufacturing", distance: 800, type: "Industrial chemical facility" },
  { suspect: "Riverside Farm", distance: 3000, type: "Agricultural operation" },
  { suspect: "Lakeview Resort", distance: 4200, type: "Hotel and marina" },
  { suspect: "Clearwater Lodge", distance: 5600, type: "Fishing lodge" }
];
```

### Distance from West Station to Each Suspect

<div style="margin-top: 10px; margin-bottom: 20px; color: #666; font-size: .7em;">
  Distance in meters. Pollutant concentrations are highest near the source.
</div>

```js
Plot.plot({
  height: 180,
  width: Math.min(650, width),
  marginLeft: 155,
  marginRight: 80,
  x: {
    label: "Distance (meters) →",
    grid: true,
    domain: [0, 6500]
  },
  y: {
    label: null
  },
  marks: [
    Plot.barX(westDistances, {
      x: "distance",
      y: "suspect",
      fill: d => d.distance < 1000 ? "#ff0d00ff" : "#5900ffff",
      inset: 8,
      sort: { y: "x" },
      tip: {
        format: { x: false, fill: false },
        channels: {
          "Suspect": "suspect",
          "Operation": "type",
          "Distance": d => d.distance.toLocaleString() + " meters"
        }
      }
    }),
    Plot.text(westDistances, {
      x: d => d.distance + 120,
      y: "suspect",
      text: d => d.distance.toLocaleString() + "m",
      textAnchor: "start",
      fontSize: 10,
      fontWeight: d => d.distance < 1000 ? "bold" : "normal"
    })
  ]
})
```

<div style="margin-top: 40px; margin-bottom: 10px; font-size: 1em;">
  <b>ChemTech Manufacturing is only 800 meters from West station.</b> <br>This is nearly 4x closer than any other potential source. According to public records; ChemTech Manufacturing was allowed a daily discharge limit of 2.5 kg heavy metals. They also conduct quarterly maintenance shutdowns that require "temporary process changes."
</div>

---

## 3. What Contaminant Is Elevated at West Station?

The water quality data data tracked several parameterssuch as nitrogen, phosphorus, heavy metals, turbidity, pH, and dissolved oxygen. In our exploration, we discovered that the West branch saw higher than average amount of Heavy Metals and an overwhelmingly higher maximum amount of heavy metals.

```js
const waterWithDates = waterQuality.map(d => ({
  ...d,
  date: new Date(d.date)
}));

const metalsByStation = stationOrder.map(station => {
  const stationData = waterWithDates.filter(d => d.station_id === station);
  const avg = stationData.reduce((s, d) => s + d.heavy_metals_ppb, 0) / stationData.length;
  const max = Math.max(...stationData.map(d => d.heavy_metals_ppb));
  return { station, avg, max };
});

// Create two rows per station for the grouped bar
const groupedData = metalsByStation.flatMap(d => [
  { station: d.station, metric: "Maximum", value: d.max, avg: d.avg, max: d.max },
  { station: d.station, metric: "Average", value: d.avg, avg: d.avg, max: d.max }
]);
```

### Heavy Metal Levels by Station

<div style="margin-top: 10px; margin-bottom: 20px; color: #666; font-size: .7em;">
  Top bar = highest recorded reading. Bottom bar = two-year average. Yellow line = EPA concern (20 ppb). Red line = EPA limit (30 ppb).
</div>

```js
Plot.plot({
  height: 260,
  width: Math.min(720, width),
  marginLeft: 60,
  marginRight: 60,
  marginTop: 35,
  x: {
    label: "Heavy Metals (ppb) →",
    grid: true,
    domain: [0, 58]
  },
  y: {
    label: null,
    domain: ["Maximum", "Average"],
    axis: null
  },
  fy: {
    label: null,
    domain: stationOrder
  },
  color: {
    domain: ["Maximum", "Average"],
    range: ["#ff0d00ff", "#5900ffff"],
    legend: true
  },
  marks: [
    Plot.barX(groupedData, {
      x: "value",
      y: "metric",
      fy: "station",
      fill: "metric",
      inset: 2,
      tip: {
        format: { x: false, y: false, fy: false, fill: false },
        channels: {
          "Station": "station",
          "Average": d => d.avg.toFixed(1) + " ppb",
          "Maximum": d => d.max.toFixed(1) + " ppb"
        }
      }
    }),
    Plot.ruleX([20], { stroke: "#e6ab02", strokeWidth: 1.5, strokeDasharray: "6,3" }),
    Plot.ruleX([30], { stroke: "#ff0d00ff", strokeWidth: 1.5, strokeDasharray: "6,3" }),
    Plot.text(groupedData.filter(d => d.metric === "Maximum"), {
      x: d => d.value + 2,
      y: "metric",
      fy: "station",
      text: d => d.value.toFixed(1),
      textAnchor: "start",
      fontSize: 9,
      fontWeight: d => d.station === "West" ? "bold" : "normal",
      fill: d => d.station === "West" ? "#ff0d00ff" : "#666"
    }),
    Plot.text(
      [
        { val: 20, label: "Concern" }, 
        { val: 30, label: "EPA Limit" }
      ], 
      {
        x: "val",
        text: "label",
        fy: () => "North",
        y: () => "Maximum",
        dy: -22,
        dx: 0,
        textAnchor: "middle",
        fill: "black",
        fontSize: 8
      }
    )
  ]
})
```

<div style="margin-top: 40px; margin-bottom: 20px; font-size: 1em;">
  <b>West station shows a different pattern than the others.</b> <br>All four stations have similar maximums of (15-22 ppb), West on the other hand has a maximum of 48.8 ppb which is more than 3x higher than any other station and exceeds the EPA limit of 30 ppb. The average is higher as well, however, this may be due to the extereme maximum inflating the average.
</div>

<div style="padding: 15px; background: #f5f5f5; border-radius: 4px; font-family: monospace; font-size: 0.9em; margin-bottom: 20px;">
  West: avg 15.5 ppb, max <b style="color: #ff0d00ff;">48.8 ppb</b> (exceeds 30 ppb limit)<br>
  North: avg 11.0 ppb, max 13.5 ppb <br> South: avg 11.2 ppb, max 13.8 ppb <br>East: avg 11.1 ppb, max 13.9 ppb
</div>


---

## 4. When Do Heavy Metal Spikes Occur?

ChemTech conducts maintenance shutdowns every quarter. If their operations are the source of contamination, heavy metal readings should spike during those periods.

```js
const chemtechEvents = activities
  .filter(d => d.suspect === "ChemTech Manufacturing")
  .map(d => ({
    ...d,
    date: new Date(d.date),
    endDate: new Date(new Date(d.date).getTime() + d.duration_days * 24 * 60 * 60 * 1000)
  }));

const concernThreshold = 20;
const regulatoryLimit = 30;

const westWater = waterWithDates.filter(d => d.station_id === "West");
```

### Heavy Metals at West Station Over Time

<div style="margin-top: 10px; margin-bottom: 20px; color: #666; font-size: .7em;">
  Weekly readings at West station. Orange shading = ChemTech maintenance periods. Yellow dashed = EPA concern (20 ppb). Red dashed = EPA limit (30 ppb). Hover red dots for details.
</div>

```js
Plot.plot({
  height: 320,
  width: Math.min(900, width),
  marginLeft: 55,
  marginRight: 80,
  marginBottom: 40,
  marginTop: 20,
  x: {
    label: null,
    type: "time"
  },
  y: {
    label: "Heavy Metals (ppb)",
    grid: true,
    domain: [0, 55]
  },
  marks: [
    Plot.rectY(chemtechEvents, {
      x1: "date",
      x2: "endDate",
      y1: 0,
      y2: 55,
      fill: "#fdae61",
      fillOpacity: 0.4
    }),
    Plot.ruleY([concernThreshold], { stroke: "#cfa819ff", strokeWidth: 1.5, strokeDasharray: "6,3" }),
    Plot.ruleY([regulatoryLimit], { stroke: "#ff0d00ff", strokeWidth: 1.5, strokeDasharray: "6,3" }),
    Plot.text([{}], {
      x: new Date("2024-12-01"),
      y: concernThreshold,
      text: ["Concern"],
      dy: -8,
      fontSize: 9,
      fill: "#000000ff",
      textAnchor: "start"
    }),
    Plot.text([{}], {
      x: new Date("2024-12-01"),
      y: regulatoryLimit,
      text: ["EPA Limit"],
      dy: -8,
      fontSize: 9,
      fill: "#000000ff",
      textAnchor: "start"
    }),
    Plot.line(westWater, {
      x: "date",
      y: "heavy_metals_ppb",
      stroke: "#5900ffff",
      strokeWidth: 1.5
    }),
    Plot.dot(westWater.filter(d => d.heavy_metals_ppb > concernThreshold), {
      x: "date",
      y: "heavy_metals_ppb",
      fill: "#ff0d00ff",
      r: 5,
      tip: {
        format: { fill: false },
        channels: {
          "Date": d => d.date.toLocaleDateString('en-US', { year: 'numeric', month: 'short', day: 'numeric' }),
          "Reading": d => d.heavy_metals_ppb.toFixed(1) + " ppb",
          "Status": d => d.heavy_metals_ppb > regulatoryLimit ? "Exceeds EPA limit" : "Above concern threshold"
        }
      }
    })
  ]
})
```

```js
const spikes = westWater.filter(d => d.heavy_metals_ppb > concernThreshold);
const spikesDuringMaintenance = spikes.filter(spike => {
  return chemtechEvents.some(event => {
    const eventStart = event.date.getTime();
    const eventEnd = eventStart + (event.duration_days + 7) * 24 * 60 * 60 * 1000;
    return spike.date.getTime() >= eventStart && spike.date.getTime() <= eventEnd;
  });
});
```

<div style="margin-top: 40px; margin-bottom: 20px; font-size: 1em;">
  <b>The spikes align with ChemTech's maintenance schedule.</b> <br>Of ${spikes.length} readings above the EPA concern threshold, all of them occurred during or within a week after a maintenance shutdown. 
</div>

<div style="padding: 15px; background: #f5f5f5; border-radius: 4px; font-family: monospace; font-size: 0.9em; margin-bottom: 20px;">
  Readings above 20 ppb: <b>${spikes.length}</b><br>
  During/after ChemTech maintenance: <b>${spikesDuringMaintenance.length}</b><br>
</div>

---

## 5. Do Other Species Tell the Same Story?

The fish survey data classifies Trout as high sensitivity, Bass as medium, and Carp as low. If heavy metals are the cause, we'd expect different species to respond differently based on their pollution sensitivity. 

```js
const westFish = fishSurveys.filter(d => d.station_id === "West");

const speciesChange = ["Trout", "Bass", "Carp"].map(species => {
  const data = westFish.filter(d => d.species === species).sort((a, b) => new Date(a.date) - new Date(b.date));
  const first = data[0].count;
  const last = data[data.length - 1].count;
  const sensitivity = data[0].pollution_sensitivity;
  return { 
    species, 
    sensitivity,
    first, 
    last, 
    change: last - first,
    pctChange: Math.round((last - first) / first * 100)
  };
});
```

### Population Change at West Station by Species

<div style="margin-top: 10px; margin-bottom: 20px; color: #666; font-size: .7em;">
  Net change from Jan 2023 to Oct 2024. Sensitivity classification from fish survey data.
</div>

```js
Plot.plot({
  height: 180,
  width: Math.min(680, width),
  marginLeft: 60,
  marginRight: 50,
  marginBottom: 40,
  x: {
    label: "← Decline · Growth →",
    grid: true,
    domain: [-52, 38]
  },
  y: {
    label: null,
    domain: ["Carp", "Bass", "Trout"]
  },
  marks: [
    Plot.barX(speciesChange, {
      x: "change",
      y: "species",
      fill: d => d.change < 0 ? "#ff0d00ff" : "#1a9850",
      inset: 10,
      tip: {
        format: { x: false, fill: false },
        channels: {
          "Species": "species",
          "Sensitivity": "sensitivity",
          "Jan 2023": "first",
          "Oct 2024": "last",
          "Change": d => (d.change > 0 ? "+" : "") + d.change + " fish"
        }
      }
    }),
    Plot.ruleX([0], { stroke: "#333", strokeWidth: 1 }),
    Plot.text(speciesChange, {
      x: d => d.change > 0 ? d.change + 4 : d.change - 4,
      y: "species",
      text: d => (d.change > 0 ? "+" : "") + d.change + " (" + (d.change > 0 ? "+" : "") + d.pctChange + "%)",
      textAnchor: d => d.change > 0 ? "start" : "end",
      fontSize: 10,
      fontWeight: "bold"
    }),
    Plot.text(speciesChange, {
      x: 26,
      y: "species",
      text: d => d.sensitivity + " sensitivity",
      textAnchor: "start",
      fontSize: 9,
      fill: "#666"
    })
  ]
})
```

<div style="margin-top: 40px; margin-bottom: 20px; font-size: 1em;">
  <b>The pattern matches what we'd expect from pollution.</b> <br>Trout (highest sensitivity) declined 70%. Bass (medium) declined 38%. Carp (low sensitivity) actually increased 35%. This supports ChemTech's operation as the cause.
</div>

<div style="padding: 15px; background: #f5f5f5; border-radius: 4px; font-family: monospace; font-size: 0.9em; margin-bottom: 20px;">
  Trout (High): 43 → 13 = <b style="color: #ff0d00ff;">−70%</b><br>
  Bass (Medium): 60 → 37 = <b style="color: #ff0d00ff;">−38%</b><br>
  Carp (Low): 23 → 31 = <b style="color: #1a9850;">+35%</b>
</div>

---

## 6. Conclusion

<div style="margin-top: 20px; margin-bottom: 30px; padding: 20px; background: #f5f5f5; border-left: 4px solid #ff0d00ff;">

<b>ChemTech Manufacturing is most likely responsible for the Clearwater crisis.</b><br><br>

We found evidence of ChemTech being responsible according to:<br><br>

<b>Location:</b> West station, where fish populations collapsed, is 800m from ChemTech nearly 4x closer than to any other suspect. The other stations show healthy populations.<br><br>

<b>Contaminant:</b> Heavy metals at West station reached 48.8 ppb, exceeding the EPA limit. Other stations never exceeded 14 ppb. ChemTech is the only suspect permitted to discharge heavy metals.<br><br>

<b>Timing:</b> All elevated readings occurred during or after ChemTech's quarterly maintenance shutdowns. 

</div>

<div style="margin-top: 20px; font-size: 0.95em;">
  <b>Other Suspects:</b><br><br>
  <b>Riverside Farm</b> — North station (nearest to farm) shows healthy fish.<br><br>
  <b>Lakeview Resort</b> — East station shows no contamination.<br><br>
  <b>Clearwater Lodge</b> — South station is healthy.
</div>
