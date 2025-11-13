---
title: "Lab 2: Subway Staffing"
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
  font-style: bold !important;      /* matches your h2 vibe */
  font-size: 1.25em !important;       /* smaller than h2, larger than captions */
  font-weight: 400 !important;        /* same weight as h2 for consistency */
  color: #444 !important;             /* darker than captions, lighter than h2 */
  margin-top: 1em !important;       
  margin-bottom: 0.5em !important;
  line-height: 1.5 !important;
}
</style>




This page is where you can iterate. Follow the lab instructions in the [readme.md](./README.md).

---
# Lab 2 — Subway Staffing
---

## Question 1: How did local events impact ridership in summer 2025? What effect did the July 15th fare increase have?

```js
const incidents = FileAttachment("./data/incidents.csv").csv({ typed: true })
const local_events = FileAttachment("./data/local_events.csv").csv({ typed: true })
const upcoming_events = FileAttachment("./data/upcoming_events.csv").csv({ typed: true })
const ridership = FileAttachment("./data/ridership.csv").csv({ typed: true })
```

```js
const currentStaffing = {
  "Times Sq-42 St": 19,
  "Grand Central-42 St": 18,
  "34 St-Penn Station": 15,
  "14 St-Union Sq": 4,
  "Fulton St": 17,
  "42 St-Port Authority": 14,
  "Herald Sq-34 St": 15,
  "Canal St": 4,
  "59 St-Columbus Circle": 6,
  "125 St": 7,
  "96 St": 19,
  "86 St": 19,
  "72 St": 10,
  "66 St-Lincoln Center": 15,
  "50 St": 20,
  "28 St": 13,
  "23 St": 8,
  "Christopher St": 15,
  "Houston St": 18,
  "Spring St": 12,
  "Chambers St": 18,
  "Wall St": 9,
  "Bowling Green": 6,
  "West 4 St-Wash Sq": 4,
  "Astor Pl": 7
}
```

<!-- doing manual filtering and calculations instead of inline with transforms in plot -->
```js
const dayKey = d => d.toISOString().slice(0, 10);
```

```js
// attendance per day
const eventByDate = new Map();
for (const e of local_events) {
  const k = dayKey(e.date);
  eventByDate.set(k, (eventByDate.get(k) ?? 0) + (e.estimated_attendance ?? 0));
}
```
```js
const series = Array.from(totalsByDate, ([k, total]) => ({
  date: new Date(k),
  total,
  attendance: eventByDate.get(k) ?? 0
})).sort((a, b) => a.date - b.date);
```
```js
// riders per day
const totalsByDate = new Map();
for (const r of ridership) {
  const k = dayKey(r.date);
  const total = (r.entrances ?? 0) + (r.exits ?? 0);
  totalsByDate.set(k, (totalsByDate.get(k) ?? 0) + total);
}
```
```js
// assigning the date range to the start and end of our dataset to call in the plot
const start = new Date("2025-06-01");
const end   = new Date("2025-08-15");
const summer = series.filter(d => d.date >= start && d.date <= end);
```
```js
// assigning the highest attendance from the dataset to a var (might not use this and manually domain it instead)
const maxAttendance = Math.max(1, ...summer.map(d => d.attendance));
const FARE_CHANGE = new Date("2025-07-15"); // defining the date that the fare change hit
```

```js
// pre/post slices for background and subcircles
const pre  = summer.filter(d => d.date <  FARE_CHANGE);
const post = summer.filter(d => d.date >= FARE_CHANGE);

// highs and lows pre and post 
const preMax  = pre.reduce((a,b)  => (b.total > a.total ? b : a));
const preMin  = pre.reduce((a,b)  => (b.total < a.total ? b : a));
const postMax = post.reduce((a,b) => (b.total > a.total ? b : a));
const postMin = post.reduce((a,b) => (b.total < a.total ? b : a));

// background coloring for pre and post fare hike
const start = new Date("2025-06-01");
const end   = new Date("2025-08-15"); // keep your current window
const bands = [
  { x1: start,       x2: FARE_CHANGE, fill: "#41cd69c5" }, 
  { x1: FARE_CHANGE, x2: end,         fill: "#8e655fb3" }  
];
```
### Summer 2025 Subway Ridership before and after fare hike with event days

<div style="margin-top: 10px; margin-bottom: 20px; color: #666; font-size: 0.7em;">
  Event days are shown as dots (size of dot correlates with number in attendance); dashed line marks July 15 fare increase.
</div>

```js
//define plot size margins etc
Plot.plot({
  marginTop: 50, 
  height: 420,
  marginRight: 60,
  marginBottom: 48,
  marginLeft: 72,
  grid: true,
  style: { fontSize: 13, background: "transparent" },

  x: { label: "Date", type: "utc" },
  y: { label: "Total riders (entrances + exits)", domain: [520000, 700000] },
  r: { type: "sqrt", domain: [0, maxAttendance], range: [3, 11], label: "Event attendance" },

  marks: [
    Plot.rectX(bands, {
      x1: "x1", 
      x2: "x2",
      fill: "fill",
      opacity: 0.15 
    }),

    //fare hike line
    Plot.ruleX([FARE_CHANGE], { stroke: "#000000ff", strokeWidth: 2, strokeDasharray: "4,4"}),
    
    // making the text to put next to the hike line (i feel like there is a better way to control where it is)
    Plot.text([{ date: FARE_CHANGE }], {
      x: "date",
      y: () => Math.max(...summer.map(d => d.total)),
      text: () => "Fare increase (Jul 15)",
      dy: -20,
      dx: 65,
      frameAnchor: "top",
      fill: "#000000ff", stroke: "white", strokeWidth: 2
    }),
    
    // plotting the fare throughout the summer on one line
    Plot.line(summer, {
      x: "date",
      y: "total",
      curve: "catmull-rom",
      stroke: "#000000ff",
      strokeWidth: 2,
      opacity: 0.95
    }),

    // event dot plot
    Plot.dot(summer.filter(d => d.attendance > 0), {
      x: "date",
      y: "total",
      r: "attendance",
      fill: "#ca25ebff",
      stroke: "white",
      strokeWidth: 1.25,
      opacity: 0.98,
      tip: true,
      title: d => {
        const dt = d.date.toLocaleDateString("en-US", { month: "short", day: "numeric" });
        const rid = d.total.toLocaleString();
        const att = d.attendance.toLocaleString();
        return `${dt} · Event day\nRidership: ${rid}\nAttendance: ${att}`;
      }
    }),

    // pre/post rings
    Plot.dot([preMax, preMin], {
      x: "date", y: "total",
      r: 6,
      fill: "none",
      stroke: "#000000ff", strokeWidth: 2,
      tip: false,
      title: d => `PRE · ${d.date.toLocaleDateString("en-US",{month:"short",day:"numeric"})}\nRidership: ${d.total.toLocaleString()}`
    }),
    
    //making rings for the pre and post min and max riders
    Plot.dot([postMax, postMin], {
      x: "date", y: "total",
      r: 6,
      fill: "none",
      stroke: "#000000ff", strokeWidth: 2,
      tip: false,
      title: d => `POST · ${d.date.toLocaleDateString("en-US",{month:"short",day:"numeric"})}\nRidership: ${d.total.toLocaleString()}`
    }),

    // labels above min max rings
    Plot.text(
      [
        { ...preMax,  label: "Pre max"  },
        { ...preMin,  label: "Pre min"  },
        { ...postMax, label: "Post max" },
        { ...postMin, label: "Post min" }
      ],
      {
        x: "date", y: "total", text: "label",
        dy: -8, dx: 6, textAnchor: "start",
        fill: "#000000ff", stroke: "white", strokeWidth: 2
      }
    )
  ]
})

```
<div style="margin-top: 10px; margin-bottom: 20px; ; font-size: 0.9em;">
  According to our chart, the highest ridership days occured on local event days both before and after the fare changes. Despite this, not all days with events correlated to higher than average attendance. Ridership across the board declined significantly post fare hike with the maximum ridership decreasing by nearly 70,000 riders, and average ridership hovering around the pre fare hike minimum.
</div>

---
## Question 2: How do the stations compare when it comes to response time? Which are the best, which are the worst?

### Station Response Times (Best to Worst)

<div style="margin-top: 10px; margin-bottom: 20px; color: #666; font-size: 0.7em;">
  Bars represent average response time per station. The red line is the system average. Green bars have lower than average response times while red bars have higher than average response times
</div>

```js
// creating vars 
const totalResponseTime = incidents.reduce((sum, d) => sum + d.response_time_minutes, 0); //sum of response times
const globalAverage = totalResponseTime / incidents.length; //average of sum of response times 
```
```js
Plot.plot({
  marginLeft: 140, 
  height: 600,   
  grid: true,
  x: { label: "Avg Response Time (mins)", nice: true },
  y: { label: "Station" }, 

  color: {
    type: "threshold",
    domain: [globalAverage],
    range: ["#19cc22ff", "#d12e2e"] 
  },

  marks: [
    Plot.barX(incidents, Plot.groupY( //using the group transformation to group the rows within incidents based on their station 
      { x: "mean", fill: "mean" },  //assigning the avg response time for each group to its station
      { 
        y: "station", 
        x: "response_time_minutes",
        fill: "response_time_minutes", 
        sort: { y: "x" },
        
        // FIX: Customize the tooltip to hide unwanted data
        tip: {
          format: {
            y: null,    // Hide the Station name (it's already on the axis)
            fill: null, // Hide the duplicate number from the color channel
            x: true     // Keep the Average Response Time
          }
        }
      }
    )),
    // line to show average response time 
    Plot.ruleX([globalAverage], {
      stroke: "#000000ff",
      strokeWidth: 2,
      strokeDasharray: "4,4"
    }),
    // actually showing the average response tiem next to the line
    Plot.text([globalAverage], {
      x: d => d,          
      frameAnchor: "top", 
      dy: 10,            
      dx: 80,
      text: d => `Avg response time: ${Number(d).toFixed(2)} min`,
      fill: "#000000ff",
      fontWeight: "bold"
    })
  ]
})
```

<div style="margin-top: 10px; margin-bottom: 20px; ; font-size: 0.9em;">
  Our chart shows that 13 out of 25 stations had quicker than average response times with Fulton St, Houston St, Times Sq - 42 St, 86 St, and Grand Central had response times of under 5.5 minutes. Our chart also shows that the longest response time stations took over 17 minutes. 
</div>

---

## Question 3: Which three stations need the most staffing help for next summer based on the 2026 event calendar?

### Forecasted Event Traffic: Summer 2026

<div style="margin-top: 10px; margin-bottom: 20px; color: #666; font-size: 0.7em;">
  Bar length shows how many events are scheduled. Color shows total expected crowds (Darker = More People)..
</div>

```js
Plot.plot({
  marginLeft: 140, 
  marginBottom: 48,
  height: 600,
  grid: true,
  
  x: { 
    label: "Number of Scheduled Events", 
    tickFormat: "d" 
  },
  y: { label: null },
  
  color: { 
    scheme: "BuGn", 
    legend: true, 
    label: "Total Expected Attendance" 
  },

  marks: [
    Plot.barX(upcoming_events, Plot.groupY(
      //group transform to group all upcoming events by station 
      { x: "count", fill: "sum", title: "sum" }, //take count of events per station on the x and take the total of expected attendance and tie it to fill the bar
      { 
        y: "nearby_station", 
        fill: "expected_attendance", 
        title: "expected_attendance", // <--- FIX: This was missing! Now it sums the right column.
        sort: { y: "x", reverse: true }, 
        tip: {
          format: {
            x: false,    
            y: false,    
            fill: false, // Hide Color Box
            title: d => `Expected Attendance: ${Math.round(d).toLocaleString()}`
          }
        }
      }
    )),
    
    Plot.text(upcoming_events, Plot.groupY(
      { x: "count", text: "count" },
      { 
        y: "nearby_station", 
        textAnchor: "start",
        dx: 5, 
        fill: "black"
      }
    ))
  ]
})
```
<div style="margin-top: 10px; margin-bottom: 20px; ; font-size: 0.9em;">
  The chart shows that events occuring near Chambers St, 34 St-Penn Station, and Canal St have both more events as well as significantly higher expected attendees than other events. Due to these highly anticipated events, staffing should be increased at these three stations.
</div>

---

## Question 4: If you had to prioritize one station to get increased staffing, which would it be and why?

<div style="margin-top: 10px; margin-bottom: 20px; ; font-size: 0.9em;">
  Looking at our third chart, we see that Canal Street had the highest expected attendance for events in 2026 as well as the most events expected to occur. Looking at the historical data from the second chart, we see that the average response time was among the worst meaning that given the expected crowds, total events, and high average response time, it is suggested to increase staffing here.
</div>

