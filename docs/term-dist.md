---
theme: dashboard
title: Office hours term distribution
---

# Office Hours Term Distribution

Provide the office hours records from [eecsoh.eecs.umich.edu](https://eecsoh.eecs.umich.edu).
For this page, the records must contain office hours visits across multiple semesters.

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

The input contains entries from __${format(fullDateRange.get('min', 0), 'yyyy-MM-dd')}__ to __${format(fullDateRange.get('max', 0), 'yyyy-MM-dd')}__. Select the range of two academic terms to compare them.

```js
function setupSideInputs(side) {
    const selectMin = Inputs.date({
        label: side + ' Starting date',
        min: fullDateRange.get('min', 0),
        max: fullDateRange.get('max', 0)
    });
    const min = Generators.input(selectMin);

    const selectMax = Inputs.date({
        label: side + ' Ending date',
        min: min,
        max: fullDateRange.get('max', 0)
    });

    const nameInput = Inputs.text({
        label: side + ' Semester',
        placeholder: "Fall 2023"
    });

    const studentPopInput = Inputs.text({
        type: "number",
        label: side + ' Student count',
        min: "1",
        max: "2000"
    });

    return {
        inputs: {
            min: selectMin,
            max: selectMax,
            name: nameInput,
            studentCount: studentPopInput
        }
    };
}

const sideA = setupSideInputs('A');
const sideB = setupSideInputs('B');

// We HAVE TO assign to consts in global scope for them to turn into reactive variables from AsyncGenerator
const Amin = Generators.input(sideA.inputs.min);
const Amax = Generators.input(sideA.inputs.max);
const Aname = Generators.input(sideA.inputs.name);
const AstudentCount = Generators.input(sideA.inputs.studentCount);
const Bmin = Generators.input(sideB.inputs.min);
const Bmax = Generators.input(sideB.inputs.max);
const Bname = Generators.input(sideB.inputs.name);
const BstudentCount = Generators.input(sideB.inputs.studentCount);

```

<div class="grid grid-cols-2">
    <div>
        ${sideA.inputs.min}
        ${sideA.inputs.max}
        ${sideA.inputs.name}
        ${sideA.inputs.studentCount}
    </div>
    <div>
        ${sideB.inputs.min}
        ${sideB.inputs.max}
        ${sideB.inputs.name}
        ${sideB.inputs.studentCount}
    </div>
</div>

```js
import { parse, endOfDay, isSameDay } from "npm:date-fns";

function viewDateRange(minDate, maxDate) {
    return dsFull
        .filter(aq.escape(d => d.id_timestamp > minDate))
        .filter(aq.escape(d => d.id_timestamp < endOfDay(maxDate, {timeZone: "America/Detroit"})));
}
const dsA = viewDateRange(Amin, Amax);
const dsB = viewDateRange(Bmin, Bmax);

function join(aTable, bTable) {
    return aTable.derive({'side': aq.escape(d => Aname ?? 'A')})
        .concat(bTable.derive({'side': aq.escape(d => Bname ?? 'B')}));
}
```
---

### Some notes and theoretical thoughts

- Measure the distribution of student demand as a function of queue length
- Consider paired comparisons between corresponding weeks in each semester

# Demand measurement

```js
import { differenceInCalendarWeeks, differenceInDays } from "npm:date-fns";

function maxHourRange(times) {
    const max = times.reduce((a, b) => a.getTime() > b.getTime() ? a : b);
    const min = times.reduce((a, b) => a.getTime() < b.getTime() ? a : b);
    return differenceInMinutes(max, min) / 60;
}

// Distribution of rate of student visits by week
function demandRateByDate(ds, min, studentCount) {
    return ds
        .derive({
            entryHours: aq.escape(d => differenceInMinutes(
                d.id_timestamp,
                startOfDay(d.id_timestamp)) / 60),
            week: aq.escape(d => differenceInCalendarWeeks(
                d.date,
                min
            )),
            dayOfTerm: aq.escape(d => differenceInDays(
                d.date,
                min
            ))
        })
        .groupby('date')
        .rollup({
            'demandRate': d => aq.op.count() / (aq.op.max(d.entryHours) - aq.op.min(d.entryHours)),
            'week': d => aq.op.any(d.week),
            'dayOfTerm': d => aq.op.any(d.dayOfTerm)
        })
        .derive({
            'demandRate': aq.escape(d => d.demandRate / studentCount)
        });
}
```

<div class="card">
    <h2>Daily demand rate (proportion of student population / hour)</h2>
    ${Plot.plot({
        marks: [
            Plot.dotY(
                join(
                    demandRateByDate(dsA, Amin, AstudentCount),
                    demandRateByDate(dsB, Bmin, BstudentCount)
            ), {y: 'demandRate', x: 'dayOfTerm', fill: 'side'}),
        ],
        color: {legend: true},
        x: {label: 'Day of semester'},
        y: {label: 'Average proportion of student population that visited OH per hour each day'}
    })}
</div>

# Frequent visitors

```js
function mostFrequentVisitorsVisits(ds, min, n=7) {
    const mostFrequent = ds.groupby('email')
        .count({ as: 'visit_count' })
        .orderby(aq.desc('visit_count'))
        .slice(0, n);
    return ds.join_right(mostFrequent, 'email')
        .derive({
            name: aq.escape(row => '(' + row['visit_count'] + ') ' + row['email'].split('@')[0]),
            dayOfTerm: aq.escape(d => differenceInDays(
                d.date,
                min
            ))
        });
}
```

<div class="grid">
    <div class="card">
        <h2>Visits from most frequent visitors over time (${Aname})</h2>
        ${Plot.dot(mostFrequentVisitorsVisits(dsA, Amin), {x: "name", y: "dayOfTerm"}).plot()}
    </div>
    <div class="card">
        <h2>Visits from most frequent visitors over time (${Bname})</h2>
        ${Plot.dot(mostFrequentVisitorsVisits(dsB, Bmin), {x: "name", y: "dayOfTerm"}).plot()}
    </div>
</div>
