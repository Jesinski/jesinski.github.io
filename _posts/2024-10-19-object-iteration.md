---
layout: post
title: Reduce vs. for...in
---

## Object iteration comparison

I was discussing object iteration with a work colleague.

I had implemented the for...in following the popular way to do it, as it already appeared on another function that used the same approach for object iteration.

> Let's keep the file somewhat standardized, I thought.

Then I asked my colleague if he had a different idea on how to iterate over the object of objects. He talked about reduce (with Object.keys(), I will omit this information from now on, but it is needed for object iteration as we get an array with all the keys from the given object, which can be used to iterate over).

**Both solutions solve the problem, with no major differences other than readability.**

For...in is straightforward, maybe you could confuse it with for...of, but reading the next code line already puts you on the right track.

On the other hand, reduce is quite challenging the first couple of times you see it. - at least it was for me in my early days...

> Of course, you can flex way more throwing a .reduce at the codebase compared to the simpler for...in.

I finished my day and went home, a bit intrigued, wondering if one approach could yield a significant difference.

With that thought tickling my nerves, I did some tests to see if performance would be the decision point to pick one approach over the other.

With a really simple methodology, I created JSON files with one object each, inside the object I added 10, 100, 1.000, 1.000.000, 10.000.000 objects that had a structure like this:

```ts
"obj": {
  "id": number;
  "filter": "keep" | "remove"
}
```

Then I implemented a tester function that reads a file and filters out the objects based on the `filter` property using the two approaches.

For the sake of simplicity, I used `console.time` to measure the time each function took to run.

The for...in approach:

```js
console.time("for loop");
const filteredFor = {};
for (const key in example) {
  if (example[key].filter == "keep") {
    filteredFor[key] = example[key];
  }
}
console.timeEnd("for loop");
```

The reduce approach:

```js
console.time("reduce");
Object.keys(example).reduce((acc, key) => {
  if (example[key].filter == "keep") {
    acc[key] = example[key];
  }
  return acc;
}, {});
console.timeEnd("reduce");
```

I ran 5 times for each test file, the results are summarized in the table below:

| # of objects | for loop  | reduce    |
| ------------ | --------- | --------- |
| 10           | 0.026ms   | 0.014ms   |
| 100          | 0.058ms   | 0.019ms   |
| 1.000        | 0.136ms   | 0.083ms   |
| 100.000      | 6.327ms   | 5.762ms   |
| 1.000.000    | 69.374ms  | 73.511ms  |
| 10.000.000   | 876.326ms | 874.711ms |

> Running Node v20.14.0 on a 2023 MacBook Pro M3 36RAM (OS 14.6.1)

## Reduce wins, that's the end.

> No, I'm joking. Well, reduce is faster, but at what cost?

Firstly, people who don't know it will have a hard time when they need to work with it, secondly if the codebase uses for...in you will be deviating from the standard.

In my _humble_ opinion, the first reason is just dumb. If we don't use something because inexperienced people don't know **only keeps those people unexperienced and lowers the knowledge bar of the whole team.**

The second reason has a greater impact on the decision process, if the gain was significantly huge - as the 10M objects JSON - I would steer to break the standard, and then think about migrating the rest of the code to the new approach.

Back to my work, I don't see a significant improvement, and I rather have the codebase standardized so the good ol' for...in will be kept - and wait to flex a reduce when it is feasible to do so.

## Afterthoughts

The scenario above is a super-small case where we Engineers should leave the automatic thinking and reflect on the code we are outputting and putting into production.

But the same scenario - do as we always did or change to a new approach - will happen in a broader context, with way more impact on the overall performance of our product, and surely, don't make any decisions without data, take some time and do some tests, the sooner we realize we are not on the right track, the better.
