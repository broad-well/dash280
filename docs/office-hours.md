---
theme: dashboard
title: Office hours vitals
---

# Office Hours Vitals

Provide the office hours records from [eecsoh.eecs.umich.edu](https://eecsoh.eecs.umich.edu) for analysis.

```js
const stack = view(Inputs.file({label: "Provide stack.csv", accept: ".csv", required: true}))
```
```js
const csv = stack.csv({typed: true})
```
```js
import { addMinutes } from "npm:date-fns";
function subtractLocalOffset(realDate) {
    return addMinutes(realDate, -new Date().getTimezoneOffset());
}
function addLocalOffset(realDate) {
    return addMinutes(realDate, new Date().getTimezoneOffset());
}
```
```js
import { differenceInMinutes, startOfDay, format } from "npm:date-fns";
const dsFull = aq.from(csv)
    .derive({
        duration: aq.escape(d => differenceInMinutes(d.removed_at, d.id_timestamp)),
        date: aq.escape(d => startOfDay(d.id_timestamp, {timeZone: "America/Detroit"})),
        removedTime: aq.escape(d => new Date(d.removed_at)),
        staff_helped: d => d.helped && d.removed_by !== d.email,
    })
```
```js
const fullDateRange = dsFull.rollup({min: aq.op.min('id_timestamp'), max: aq.op.max('id_timestamp')});
```

The input contains entries from __${format(fullDateRange.get('min', 0), 'yyyy-MM-dd')}__ to __${format(fullDateRange.get('max', 0), 'yyyy-MM-dd')}__. Constrain the date range of your analysis.

```js
const selectMin = view(Inputs.date({
    label: 'Starting date',
    min: fullDateRange.get('min', 0),
    max: fullDateRange.get('max', 0)
}));
```
```js
const selectMax = view(Inputs.date({
    label: 'Ending date',
    min: selectMin,
    max: fullDateRange.get('max', 0)
}));
```
```js
import { parse, endOfDay, isSameDay } from "npm:date-fns";
const ds = dsFull
    .filter(aq.escape(d => d.id_timestamp > selectMin))
    .filter(aq.escape(d => d.id_timestamp < endOfDay(selectMax, {timeZone: "America/Detroit"})))
```
---

# Waiting times


```js
// Entry time of students helped calculation
const ds_entryTime = ds
    .derive({
        entryHours: aq.escape(d => differenceInMinutes(
            d.id_timestamp,
            startOfDay(d.id_timestamp)) / 60),
        staffHelpedString: d => d.staff_helped ? 'Helped' : 'Not helped'
    });
```
<div class="grid grid-cols-2">
    <div class="card">
        <h2>Waiting times over each day</h2>
        ${Plot.plot({
            marks: [
                Plot.boxY(ds, {y: 'duration', x: 'date'})
            ],
            y: {grid: true, domain: [0, 200]},
            x: {type: "band"}
        })}
    </div>
    <div class="card">
        <h2>Entry time of students and whether they were helped</h2>
        ${Plot.tickY(ds_entryTime, {
            x: 'date',
            y: 'entryHours',
            stroke: 'staffHelpedString',
        }).plot({ x: {type: 'band'}, color: {legend: true, scheme: 'Tableau10' }})}
    </div>
</div>

Select a date to see student wait times over the course of that date.
```js
const inspectInput = view(Inputs.date({
    label: 'Date to inspect',
    min: selectMin,
    max: selectMax
}))
```
```js
const inspectInputLocal = addMinutes(inspectInput, new Date().getTimezoneOffset());
```

<div class="card">
    <h2>Distribution over a day</h2>
    ${Plot.plot({
        marks: [
            Plot.dot(
                ds.filter(aq.escape(d => isSameDay(d.removedTime, inspectInputLocal) && isSameDay(d.id_timestamp, inspectInputLocal)))
                    .orderby("removedTime")
                    .derive({
                        staff_helped_str: d => d.staff_helped ? "Helped" : "Not helped",
                        enteredTimeOffset: aq.escape(d => subtractLocalOffset(d.id_timestamp)),
                        removedTimeOffset: aq.escape(d => subtractLocalOffset(d.removedTime))}),
                {x: "enteredTimeOffset", y: "duration", fill: "staff_helped_str" }
            ),
            Plot.ruleY([30, 60], {stroke: "gray"})
        ],
        color: {legend: true, scheme: "Tableau10" },
        x: {label: "Student entry time"},
        y: {label: "Minutes waited"}
    })}
</div>

# Service metrics

```js
ds.groupby('date')
```