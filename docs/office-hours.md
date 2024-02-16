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
const ds_lastEntryTime = ds.filter(d => d.staff_helped)
    .derive({ entryHours: aq.escape(d => differenceInMinutes(
        d.id_timestamp,
        startOfDay(d.id_timestamp)) / 60)
    });
```
<div class="grid grid-cols-2">
    <div class="card">
        <h2>Distribution per day</h2>
        ${Plot.plot({
            marks: [
                Plot.boxY(ds, {y: 'duration', x: 'date'})
            ],
            y: {grid: true, domain: [0, 200]},
            x: {type: "band"}
        })}
    </div>
    <div class="card">
        <h2>Entry time of students helped</h2>
        ${Plot.tickY(ds_lastEntryTime, {
            x: 'date',
            y: 'entryHours'
        }).plot({ x: {type: 'band'}})}
    </div>
</div>

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
    ${Plot.dot(
        ds.filter(aq.escape(d => isSameDay(d.id_timestamp, inspectInputLocal)))
            .orderby("removedTime")
            .derive({
                staff_helped_str: d => d.staff_helped ? "Helped" : "Not helped",
                removedTimeOffset: aq.escape(d => subtractLocalOffset(d.removedTime))}),
        {x: "removedTimeOffset", y: "duration", fill: "staff_helped_str" }
    ).plot({color: {legend: true, scheme: "Category10" }})}
</div>
