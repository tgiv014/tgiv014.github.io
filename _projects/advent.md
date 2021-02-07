---
title: Advent of Code
featured: true
meta: Starting in 2020, I've made it a personal goal to complete Advent of Code in a new language every year. In 2020, I kicked this tradition off with Golang.
icon: mdi:flare
layout: page
---

Starting in 2020, I've made it a personal goal to complete Advent of Code in a new language every year. In 2020, I kicked this tradition off with Golang.

# 2020
This year I decided to pick up Go after watching a [free conference](https://systemsconf.io/) hosted by the folks at [Dgraph](https://dgraph.io/). This was an awesome way to stretch my legs with Go and get comfortable with the syntax and built-in data structures of Go. All my solutions are on [my Github](https://github.com/tgiv014/advent2020).

### Noteworthy Puzzles
- [Day 10](https://adventofcode.com/2020/day/10)
  - The first part of the puzzle was easy to solve with built-in sorting functions and iteration, but the second part convinced me to build a graph of every adapter and the other adapters they are compatible with. This solution worked, but desperately needed optimization. In came [memoization](https://en.wikipedia.org/wiki/Memoization) (a topic that never came up in my EE curriculum).
- [Day 20](https://adventofcode.com/2020/day/20)
  - This was a fun puzzle, but it was also *mean*. It essentially required you to implement transforms (mirrors and rotations) on 2d boolean arrays and also correlate the edges of the arrays with each other to figure out their position and orientation in a larger image. It took quite a bit of pen and paper sketching to come up with a sensible coordinate system and accurate transforms.