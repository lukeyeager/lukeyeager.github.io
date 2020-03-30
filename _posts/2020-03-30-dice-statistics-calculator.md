---
layout: post
title: Dice Statistics Calculator
date: 2020-03-30 12:53 -0700
---
A simple utility to compare statistics of two different rolls in D&D.

Utilities:
* `ROLL(x,y)`: Roll X Y-sided dice.
* `ADD(x,y)`: Add two things. X and Y can be either numbers or arrays. If both are arrays, generate all possible combinations.
* `MAX(x,y)`: Take the max of two things.
* `MIN(x,y)`: Take the min of two things.
* `REPLACE(x,a,b)`: For all elements E in X, if E is in A, replace it with B. If B is an array, insert all possible values of B in place of E.

Examples:

{% include dice-calculator.html %}
