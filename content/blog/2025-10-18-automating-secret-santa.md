+++
title = "Automating Secret Santa"
description = "When our family's manual Secret Santa organizer wanted to participate, I created an over-engineered Python solution using the Hungarian algorithm to find optimal gift assignments while minimizing repeat pairings."
date = 2025-10-18
template = "blog/page.html"
[extra]
author = "Guy Lewin"
[taxonomies]
tags = [
  "secret-santa",
  "hungarian-algorithm",
  "optimization",
  "python",
  "family",
  "christmas",
]
+++

Every December, our family organizes a Secret Santa gift exchange. But this year presented a new problem - our usual organizer wanted to participate instead of managing the assignments.

Suddenly, we needed a fair way to assign Secret Santas that could handle exclusions (no one should get their spouse or immediate family), avoid repeat pairings from previous years, and work without someone manually coordinating everything.

## The Over-Engineered Solution
Instead of finding a simple solution, I (but mostly Cursor) decided to treat this as an optimization problem. Enter the [Hungarian algorithm](https://en.wikipedia.org/wiki/Hungarian_algorithm) - a mathematical method for solving assignment problems that finds the globally optimal solution.

The system works by:
1. **Handling constraints** like exclusions
2. **Building a cost matrix** where each potential giver-receiver pair gets a "penalty score" based on historical assignments (with recent assignment more penalized) supporting individuals within groups
3. **Applying the Hungarian algorithm** to find the assignment that minimizes total cost
4. **Outputting ideal solution** to either console (default) or by email so even the script runner doesn't see the assignments

You can find the complete code on GitHub: [https://github.com/GuyLewin/secret-santa](https://github.com/GuyLewin/secret-santa)

