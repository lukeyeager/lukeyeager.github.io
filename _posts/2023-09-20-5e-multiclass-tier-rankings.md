---
layout: post
title: D&D 5e Multiclass Tier Rankings, Clustered!
date: 2023-09-20 13:38 -0500
---
The Dungeon Dudes on Youtube recently finished up a series on Multiclass Tier Rankings in D&D ([playlist](https://www.youtube.com/playlist?list=PLQMqiULo_05PljVOyAyB7J2Zk3nji-4az), [summary video](https://youtu.be/CjjtbktqVzk)).
The results can be collated to show which combinations are best for multiclassing.

> These results reflect Monty's and Kelly's opinions - yours [and mine!] might be different

Here are the results as a table (if your character has more levels in class A than B, then you are multiclassing out of A into B, and A is your 'primary' class):

<p>
{% include multiclass-tier-rankings-1-table.html %}
</p>

... but that's not very interpretable.

So, I looked around for libraries which can do 2D clustering visualizations and I came up with [d3-force](https://d3js.org/d3-force).
With it, I can setup some forces between nodes in a graph so that classes which are easy to multiclass together will appear close to each other in space.
Of course, with this method we lose information about primary vs. secondary - maybe I'll try that some other day.

* Classes are color-coded by their primary ability (for ambiguous cases I just picked one)
* Hover over each dot to see the class name
* You can click+drag a dot to rotate the visualization

<p>
{% include multiclass-tier-rankings-2-viz.html %}
</p>

The results are not too surprising - the clustering agrees with the general opinions of the Dudes:
* The clustering of Sorcerer and Warlock and Paladin is very tight (they are easy to mix)
* Wizard is way out there on his own with only Artificer for a friend (don't multiclass in or out of Wizard)
* Fighter, Rogue, and Cleric are in the middle (they are easy to multiclass with just about anything)

FYI, here are the forces I used to create the simulation:
```javascript
const simulation = d3.forceSimulation(nodes)
    // keep things centered
    .force("center", d3.forceCenter(width / 2, height / 2))
    // prevent overlaps
    .force("radius", d3.forceCollide(5))
    // set distance based on the quality of the multi-class link
    .force("link", d3.forceLink(links)
        .id(d => d.id)
        .strength(d => (1 + Math.abs(d.value - 2)))
        .distance(d => (200 * (1 - d.value/4) )));
```
