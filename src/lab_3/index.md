---
title: "Lab 3: Mayoral Mystery"
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
# Lab 3 — Campaign Performance Analysis
---

This dashboard examines and visualizes how a Mayoral candidate performed across NYC to drive actionable insights to improve poll performance next election cycle

---

## 1. Where Did the Candidate Win and Lose?

<!-- Import Data -->
```js
const nyc = await FileAttachment("data/nyc.json").json();
const results = await FileAttachment("data/election_results.csv").csv({ typed: true });
const survey = await FileAttachment("data/survey_responses.csv").csv({ typed: true });
const events = await FileAttachment("data/campaign_events.csv").csv({ typed: true });
```

```js
// Convert topoJSON to geoJSON
const districts = topojson.feature(nyc, nyc.objects.districts);

// Create a lookup map from boro_cd to election results
const resultsMap = new Map(results.map(d => [d.boro_cd, d]));

// Borough lookup used throughout
const boroughNames = {1: 'Manhattan', 2: 'Bronx', 3: 'Brooklyn', 4: 'Queens', 5: 'Staten Island'};
```

```js
// Join election results csv features to the corresponding boro geometries in the json
const districtsWithResults = {
  type: "FeatureCollection",
  features: districts.features.map(feature => {
    const boro_cd = feature.properties.BoroCD;
    const result = resultsMap.get(boro_cd);
    
    //Null handling for districts that were appearing as 0 votes (mishandled data)
    if (!result) return null;
    
    //Creating vars for metrics that will be mapped by district
    const totalVotes = result.votes_candidate + result.votes_opponent;
    const voteShare = (result.votes_candidate / totalVotes) * 100;
    const margin = result.votes_candidate - result.votes_opponent;
    const borough = boroughNames[Math.floor(boro_cd / 100)];
    
    return {
      ...feature,
      properties: {
        ...feature.properties,
        ...result,
        borough,
        voteShare,
        margin
      }
    };
  }).filter(d => d !== null)
};
```

### Vote Share by District

<div style="margin-top: 10px; margin-bottom: 20px; color: #666; font-size: .7em;">
  Green = win (>50%), Red = loss (<50%). Hover over any district to see Poll Results, Vote Share, Total Votes for candidate & opponent Vote Margin (win/loss), District Avg Income Level, Registered Voter Turnout.
</div>

```js
Plot.plot({
  height: 600,
  width: Math.min(800, width), //Make sure that the width stays below 800 
  marginLeft: 20,
  marginRight: 20,
  projection: {
    domain: districts,
    type: "mercator"
  },
  color: {
    type: "diverging",
    scheme: "RdYlGn",// Using this color scheme to simulate heatmap feeling while also avoiding red/blue scheme that can be confused for party affiliation
    domain: [25, 75], //Setting domain between the lower and upper bounds of our data with a slight padding making it 0,100 makes it too confusing
    pivot: 50, //Switiching the colorscheme to make red less votes and green more votes
    label: "Vote Share (%)",
    legend: true
  },
  marks: [
    Plot.geo(districtsWithResults, {
      fill: d => d.properties.voteShare,
      stroke: "#fff",
      strokeWidth: 1,
      tip: {
        format: {
          fill: false
        },
        channels: { //Mapping features to tooltip
          "District": d => d.properties.BoroCD,
          "Borough": d => d.properties.borough,
          "Result": d => d.properties.voteShare > 50 ? "WON" : "LOST",
          "Vote Share": d => d.properties.voteShare.toFixed(1) + "%",
          "Votes for Candidate": d => d.properties.votes_candidate,
          "Votes for Opponent": d=> d.properties.votes_opponent,
          "Vote Margin": d => (d.properties.margin > 0 ? "+" : "") + d.properties.margin.toLocaleString(),
          " ": () => "———",
          "Income Level": d => d.properties.income_category,
          "Turnout": d => d.properties.turnout_rate.toFixed(1) + "%" //Fix for percentage not displaying correctly
        }
      }
    })
  ]
})
```
<div style="margin-top: 40px; margin-bottom: 20px; ; font-size: 1em;">
  <b>The Pattern:</b> <br>Income appears to be an indicator of election results.
</div>

```js
// Win rate by income
const incomeCategories = ["Low", "Middle", "High"]; //Setting incomes array of names based on the categories in the results filter

//Mapping the results of election to income rather than the initial boro
const winRateByIncome = incomeCategories.map(income => {
  const subset = results.filter(d => d.income_category === income);
  const wins = subset.filter(d => d.votes_candidate > d.votes_opponent).length;
  return {
    income,
    wins,
    total: subset.length,
    winRate: (wins / subset.length) * 100
  };
});
```

### Win Rate by Income Level

```js
Plot.plot({
  height: 200,
  width: Math.min(500, width),
  marginLeft: 80,
  marginRight: 40,
  x: {
    label: "Win Rate % →",
    domain: [0, 100],
    grid: true
  },
  y: {
    label: null,
    domain: ["High", "Middle", "Low"]
  },
  marks: [ //Crating bars based off of the earlier mapped election results based on income
    Plot.barX(winRateByIncome, {
      x: "winRate",
      y: "income",
      fill: d => d.winRate === 100 ? "#000000ff" : "#000000ff",
      inset:10,
      tip: {
        format: {
          x: false,
          fill: false
        },
        channels: {
          "Income Level": "income",
          "Win Rate": d => d.winRate.toFixed(0) + "%",
          "Record": d => `${d.wins} wins / ${d.total} districts`
        }
      }
    }),
    Plot.text(winRateByIncome, {
      x: d => d.winRate + 3,
      y: "income",
      text: d => `${d.wins}/${d.total}`,
      textAnchor: "start",
      fontWeight: "bold"
    }),
    Plot.ruleX([0])
  ]
})
```
<div style="margin-top: 40px; margin-bottom: 20px; ; font-size: 1em;">
  <b>Low income: 26/26 wins. Middle income: 24/24 wins. High income: 0/9 wins.</b> <br> This split suggests that there may be a link between income levels and election results, however, it is important to analyze each candidates campaign efforts.</br>
</div>

---
## 2. Did Campaign Effort Actually Matter?

The campaign invested in voter outreach such as door knocking (GOTV), Events, and Candidate Appearances. We combined these into a weighted **Campaign Effort Score** which was composed as:

> **Effort Score** = Doors Knocked (1) + (Event Attendees × 2) + (Candidate Hours × 500)

<div style="color: #000000ff; font-size: .7em;"> <b>Note:</b> NYC is a diverse city with different population densities in each boro, this effort score attempts to account for this by scaling Doors Knocked and Event Attendees lower than Candidate Hours.</div>

```js
// Map event features by district 
const eventsByDistrict = new Map();
for (const e of events) {
  if (!eventsByDistrict.has(e.boro_cd)) {
    eventsByDistrict.set(e.boro_cd, { count: 0, attendance: 0 });
  }
  const current = eventsByDistrict.get(e.boro_cd);
  current.count += 1;
  current.attendance += e.estimated_attendance;
}

// Map event features by boro from district map
const byBorough = ['Manhattan', 'Bronx', 'Brooklyn', 'Queens', 'Staten Island'].map(borough => {
  const boroCode = Number(Object.entries(boroughNames).find(([k, v]) => v === borough)[0]);
  const boroResults = results.filter(d => Math.floor(d.boro_cd / 100) === boroCode);
  const boroEvents = events.filter(d => Math.floor(d.boro_cd / 100) === boroCode);
  
  const totalDoors = boroResults.reduce((s, d) => s + d.gotv_doors_knocked, 0);
  const totalHours = boroResults.reduce((s, d) => s + d.candidate_hours_spent, 0);
  const totalEvents = boroEvents.length;
  const totalAttendance = boroEvents.reduce((s, d) => s + d.estimated_attendance, 0);
  
  const totalCandidate = boroResults.reduce((s, d) => s + d.votes_candidate, 0);
  const totalOpponent = boroResults.reduce((s, d) => s + d.votes_opponent, 0);
  
  const lowCount = boroResults.filter(d => d.income_category === "Low").length;
  const midCount = boroResults.filter(d => d.income_category === "Middle").length;
  const highCount = boroResults.filter(d => d.income_category === "High").length;
  
  //Effort score scaled for population density considerations
  const effortScore = totalDoors + totalAttendance * 2 + totalHours * 500;
  
  return {
    borough,
    totalDoors,
    totalHours,
    totalEvents,
    totalAttendance,
    effortScore,
    margin: totalCandidate - totalOpponent,
    won: totalCandidate > totalOpponent,
    lowCount,
    midCount,
    highCount
  };
});

const maxEffort = Math.max(...byBorough.map(d => d.effortScore));
const maxMargin = Math.max(...byBorough.map(d => Math.abs(d.margin)));

const effortData = byBorough.map(d => ({
  borough: d.borough,
  metric: "Campaign Effort",
  value: d.effortScore / maxEffort * 100,
  raw: d.effortScore,
  totalDoors: d.totalDoors,
  totalHours: d.totalHours,
  totalEvents: d.totalEvents,
  totalAttendance: d.totalAttendance
}));

const marginData = byBorough.map(d => ({
  borough: d.borough,
  metric: "Vote Margin",
  value: d.margin / maxMargin * 100,
  raw: d.margin,
  won: d.won,
  lowCount: d.lowCount,
  midCount: d.midCount,
  highCount: d.highCount
}));
```

### Campaign Effort vs Vote Margin by Borough

<div style="margin-top: 10px; margin-bottom: 20px; color: #666; font-size: 0.7em;">
  Top bars show campaign effort, Bottom bars show vote margin. Hover over bars to see deeper insights.
</div>

```js
Plot.plot({
  height: 380,
  width: Math.min(800, width),
  marginLeft: 100,
  marginRight: 80,
  marginBottom: 35,
  marginTop: 10,
  x: {
    label: "← Loss — Win/Effort →",
    grid: true,
    domain: [-110, 110]
  },
  y: {
    label: null,
    domain: ['Staten Island', 'Queens', 'Manhattan', 'Brooklyn', 'Bronx']
  },
  fy: {
    label: null,
    domain: ["Campaign Effort", "Vote Margin"]
  },
  color: {
    domain: ["Campaign Effort", "Vote Margin"],
    range: ["#0015ffff", "#ff00f2ff"],
    legend: true
  },
  marks: [
    Plot.barX(effortData, {
      x: "value",
      y: "borough",
      fill: "metric",
      fy: "metric",
      inset: 8,
      tip: {
        format: { x: false, y: false, fy: false, fill: false },
        channels: {
          "Effort Score": d => d.raw.toLocaleString() + " pts",
          "Doors Knocked": d => d.totalDoors.toLocaleString(),
          "Events Held": "totalEvents",
          "Event Attendees": d => d.totalAttendance.toLocaleString(),
          "Candidate Hours": "totalHours"
        }
      }
    }),
    Plot.barX(marginData, {
      x: "value",
      y: "borough",
      fill: "metric",
      fy: "metric",
      inset: 8,
      tip: {
        format: { x: false, y: false, fy: false, fill: false },
        channels: {
          "Result": d => d.won ? "WON" : "LOST",
          "Vote Margin": d => (d.raw > 0 ? "+" : "") + d.raw.toLocaleString() + " votes",
          "Income Mix": d => `${d.lowCount} Low / ${d.midCount} Mid / ${d.highCount} High`
        }
      }
    }),
    Plot.ruleX([0], { stroke: "#333", strokeWidth: 1 }),
    Plot.text(marginData.filter(d => Math.abs(d.value) < 8), {
      x: d => d.value > 0 ? d.value + 4 : d.value - 4,
      y: "borough",
      fy: "metric",
      text: d => d.won ? "▸" : "◂",
      fontSize: 10,
      fill: "#ff00f2ff"
    })
  ]
})
```
<div style="margin-top: 40px; margin-bottom: 10px; ; font-size: 1em;">
  <b>Effort did not always result in outcome.</b> <br>Although Manhattan had the third highest Campaign Effort Score, the candidate lost by a considerable margin. Queens on the other hand won with a significantly lower amount of campaign effort. Our data points to the biggest factor being income as Manhattan had five High-income districts (all losses) and Staten Island had one (loss). Manhattan had the highest ratio of High-income districts and showed that the effort put in by the campaign could not overcome the underlying demographics.
</div>

---

## 3. Why Did High-Income Voters Reject the Candidate?

The post-election survey reveals how policy impacted voting habits. Due to the link between districts won and voting habits, we broke down policy agreement by income level.

```js
// Add income category to survey responses
const surveyWithIncome = survey.map(d => {
  const result = resultsMap.get(d.boro_cd);
  return {
    ...d,
    income_category: result ? result.income_category : null
  };
}).filter(d => d.income_category !== null);

// Calculate policy scores by income
const policies = [
  { key: 'affordable_housing_alignment', label: 'Affordable Housing' },
  { key: 'public_transit_alignment', label: 'Public Transit' },
  { key: 'childcare_support_alignment', label: 'Childcare Support' },
  { key: 'small_business_tax_alignment', label: 'Small Business Tax' },
  { key: 'police_reform_alignment', label: 'Police Reform' }
];

const policyByIncome = [];
for (const income of ["Low", "Middle", "High"]) {
  const subset = surveyWithIncome.filter(d => d.income_category === income);
  for (const p of policies) {
    const values = subset.map(d => d[p.key]);
    const avg = values.reduce((a, b) => a + b, 0) / values.length;
    policyByIncome.push({
      income,
      policy: p.label,
      avgScore: avg,
      count: subset.length
    });
  }
}
```

### Policy Agreement by Income Level

<div style="margin-top: 10px; margin-bottom: 20px; color: #666; font-size: 0.7em;">
  Dashed line marks neutral (2.5). Scores above 2.5 indicate agreement, below indicate disagreement.
</div>

```js
Plot.plot({
  height: 350,
  width: Math.min(750, width),
  marginLeft: 130,
  marginRight: 40,
  marginBottom: 40,
  x: {
    label: "Average Agreement Score →",
    domain: [0, 5],
    grid: true,
    ticks: 5
  },
  y: {
    label: null
  },
  fy: {
    label: null,
    domain: ["Low", "Middle", "High"]
  },
  color: {
    domain: ["Low", "Middle", "High"],
    range: ["#00ccffff", "#6f00ffff", "#ff0015ff"],
    legend: true
  },
  marks: [
    Plot.ruleX([2.5], { stroke: "#999", strokeDasharray: "4,4" }),
    Plot.barX(policyByIncome, {
      x: "avgScore",
      y: "policy",
      fy: "income",
      fill: "income",
      inset: 4,
      tip: {
        format: { x: false, fy: false, fill: false },
        channels: {
          "Policy": "policy",
          "Income Level": "income",
          "Avg Score": d => d.avgScore.toFixed(2) + " / 5",
          "Respondents": "count"
        }
      }
    }),
    Plot.text(policyByIncome, {
      x: d => d.avgScore + 0.12,
      y: "policy",
      fy: "income",
      text: d => d.avgScore.toFixed(1),
      textAnchor: "start",
      fontSize: 10
    }),
    Plot.text([{}], {
      x: 2.6,
      fy: "Low",
      y: "Affordable Housing",
      text: ["Neutral"],
      dy: -20,
      fontSize: 9,
      fill: "#999"
    })
  ]
})
```
<div style="margin-top: 40px; margin-bottom: 10px; ; font-size: 1em;">
  This election was a tale of two different voting groups, Low and Middle income voters vs. High income voters. <b>Low and Middle income voters were much in favor of all but one of the candidates policies. High income voters on the other hand were overwhelmingly not in favor of the candidates policies.</b> Notably, none of the voter groups were in favor of the candidates police reform policy with it averaging a 1.4/5 in agreement score. The data further suggests that in spite of campaign effort, overwhelming disagreement with the candidates policies were what resulted in High income districts voting against the candidate.
</div>

---

## 4. How efficient was the campaign?

This final map puts together all of the previous data and shows where the campaign was most and least effective.
- **Efficient Win (Green):** Won with modest investment 
- **Potential Overkill (Orange):** Won but invested heavily 
- **Wasted Effort (Red):** Lost despite effort  

```js
// Calculate efficiency for each district
const districtsWithEfficiency = {
  type: "FeatureCollection",
  features: districts.features
    .map(feature => {
      const boro_cd = feature.properties.BoroCD;
      const result = resultsMap.get(boro_cd);
      
      if (!result) return null;
      
      const eventData = eventsByDistrict.get(boro_cd) || { count: 0, attendance: 0 };
      
      const effortScore = result.gotv_doors_knocked + 
                          eventData.attendance * 2 + 
                          result.candidate_hours_spent * 500;
      
      const margin = result.votes_candidate - result.votes_opponent;
      const won = margin > 0;
      const borough = boroughNames[Math.floor(boro_cd / 100)];
      
      let efficiency;
      if (!won) {
        efficiency = "Wasted Effort";
      } else if (effortScore > 10000 && margin > 5000) {
        efficiency = "Potential Overkill";
      } else {
        efficiency = "Efficient Win";
      }
      
      return {
        ...feature,
        properties: {
          ...feature.properties,
          ...result,
          borough,
          effortScore,
          margin,
          won,
          numEvents: eventData.count,
          totalAttendance: eventData.attendance,
          efficiency
        }
      };
    })
    .filter(d => d !== null)
};
```

### Campaign Efficiency by District

<div style="margin-top: 10px; margin-bottom: 20px; color: #666; font-size: 0.7em;">
  Green = efficient win, Orange = potential overkill, Red = wasted effort. Hover for detailed metrics.
</div>

```js
Plot.plot({
  height: 600,
  width: Math.min(800, width),
  marginLeft: 20,
  marginRight: 20,
  projection: {
    domain: districts,
    type: "mercator"
  },
  color: {
    domain: ["Wasted Effort", "Potential Overkill", "Efficient Win"],
    range: ["#ff0015ff", "#ffa600ff", "#48c500d5"],
    legend: true
  },
  marks: [
    Plot.geo(districtsWithEfficiency, {
      fill: d => d.properties.efficiency,
      stroke: "#fff",
      strokeWidth: 1,
      tip: {
        format: { fill: false },
        channels: {
          "District": d => d.properties.BoroCD,
          "Borough": d => d.properties.borough,
          "Efficiency": d => d.properties.efficiency,
          "Result": d => d.properties.won ? "WON" : "LOST",
          "Vote Margin": d => (d.properties.margin > 0 ? "+" : "") + d.properties.margin.toLocaleString(),
          " ": () => "———",
          "Income Level": d => d.properties.income_category,
          "Effort Score": d => d.properties.effortScore.toLocaleString(),
          "Doors Knocked": d => d.properties.gotv_doors_knocked.toLocaleString(),
          "Events": d => d.properties.numEvents,
          "Candidate Hours": d => d.properties.candidate_hours_spent,
          "Turnout": d => d.properties.turnout_rate.toFixed(1) + "%"

        }
      }
    })
  ]
})
```
<div style="margin-top: 40px; margin-bottom: 10px; ; font-size: 1em;">Our map shows that the election outcome was not determined due to merely campaign effort as some of the highest effort scoring districts resulted in losses. Increased door knocking and event holding may have increased the turnout and the margin in some districts, however, many of these districts had margins of over 5000+ votes leading us to believe the effort put in may have been overkill and opting for more conservative efforts as seen in the green regions would result in a win albeit narrower. The red areas represent the ultimate deciding factor of this election which was income. All of the High income districts voted against the candidate despite ample resources having been spent in them. Post election results from the chart above show that these efforts were in vain and voters strongly disagreed with the candidate on a policy level. </div>

---

## 5. Strategic Recommendations for Next Campaign.

<div style="margin-top: 10px; margin-bottom: 10px; ; font-size: 1em;">The next campaign should consider experimenting with better messaging or voter outreach in High-income areas to relieve voters' concers that they showed towards policy post-election. Along with this, data shows that in several districts, the candidate won by large margins while having a large amount of resources used. Although decisive, these victories may have been wasteful and taken resources away from time that could have been spent tailoring messaging around policy to High-income areas. Consider modeling efforts similarly to the green districts where solid victories were achieved with relatively lower resource cost.</div>
