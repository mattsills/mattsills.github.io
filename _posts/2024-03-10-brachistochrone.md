---
layout: post
title: The Brachistochrone
---

## The Problem

In the spirit of the post on the Fourier Transform, I'd like to continue to explore simpler ways of approaching (moderately) complex subjects. With that in mind, let's introduce the ***brachistochrone problem***. For those unfamiliar with the problem, it asks: **Given two points (not directly on top of each other), what curve connects them such that a bead sliding along the curve under gravity (without friction) will transit the curve in the *<mark>least time</mark>***. This was among the first problems in what is now known as the calculus of variations, which studies the minima and maxima of functionals (functions that take other functions as inputs). [As an aside, Johann Bernoulli originally proposed this problem, in 1696, in a mathematical journal, as a challenge to his contemporaries. His brother, Jakob, as well as Leibniz and l'Hôpital wrote in with solutions. Newton also solved the problem, apparently within a single night, but did not consider it significant enough to write in.]

Here are a few ways to connect two points:

TODO: Animation

## Searching for a Solution

We are not going to solve the problem analytically, though we will briefly go over the solution at the end. If you are interested, there are plenty of online resources that do a good job explaining it (TODO: link). Instead, let's think about how we might use a computer to search for an optimal curve. Thinking about the problem in this way will also help build an intuition for solving these kinds of problems in general, and also give us an opportunity to touch on some topics in machine learning and global search more broadly.

We will need to figure out how to do three things:

- **Represent** a candidate curve: What is the space of candidate solutions?

- **Evaluate** the candidate curve: How can we score candidates against each other?

- **Search** for new curves: How should we explore the search space?

## Representation -- Discretizing the problem

Let's first consider how we might represent a candidate path in our search. One option might be to search over explicit, closed form curves. This is difficult for a number of reasons. For one, it is not clear what domain should be. Is it polynomials? Rational functions? Something more exotic? There's also the issue of regularity in the search: tuning a parameter might change the curve in a pretty dramatic way (for example changing a zero of a high degree polynomial).

Instead, let's just represent our path by a set of connected line segments, and make the assumption that no energy is lost when moving between segments. Obviously, this introduces some error, but as the number of points increases, the error introduced by the discretization will tend to zero. This is analogous to using a Riemann sum to approximate an integral. By increasing or decreasing the number of points, we can trade off the speed and accuracy of the search.

TODO: [PICTURE]

Let's just use the most straightforward representation we can think of, a list of points:

```python
@dataclasses.dataclass(frozen=True)
class Path:
    xs: list[float]
    ys: list[float]

class Solver(Protocol):
    def brachistochrone(self, *, x0: float, y0: float, x1: float, y1: float) -> Path:
        ...
```

So the curve `Path([0, 0.5, 1], [1, 0.5, 0])` represents a straight line between `(0, 1)` and `(1, 0)`.

## Evaluating -- Scoring a candidate

Given a candidate curve, how long will the bead take to fall between the start and end points? To calcualte this, we just need some high school physics. Our representation of a curve is a list of line segments. At the beginning of each line segment, the bead has some initial velocity, and experiences some acceleration due to gravity.

TODO: Fill in the math

## Searching -- Dynamics of the Problem

A good place to start is by thinking about whether the problem has any substructure we can exploit. To set some notation, let's parameterize the curve in the normal way, by its value at each point along the x-axis $x \rarr f(x)$. Let's call the initial point $(x_0,y_0)$ and the final point $(x_1,y_1)$. And let's call the optimal curve between the two points $F_{opt}(x_0,y_0,x_1,y_1)(x)=f_{opt}(x)$. If we start in the middle of the curve (at $x=(x_1-x_0)/2$), is that portion of the $f_{opt}$ curve also a brachistochrone?

TODO: Picture

After giving it a bit of thought, you can see that it's not. The reason is that except at the beginning, the bead will have some velocity, and that existing velocity changes the shape of the optimal curve. For example, if the bead is starting from a dead stop, it makes sense to build up some speed first, even at the cost of taking a slightly longer path. On the other hand, if the the bead were moving very fast at the beginning, the optimal path would be quite close to a straight line (there's not much to gain in building up speed if the bead is already moving quickly).

[PICTURE HERE]

[PICTURE HERE]

Playing around with the problem like this leads us to our first (small) insight: **the optimal path from a point does not rely on anything that happened before the bead reached that point**. That is: the optimal path between two points depends only on:

- The starting point

- The ending point

- The starting velocity of the bead

Put another way, if $F_{opt}(x_0,y_0,x_1,y_1; v_0)(x)$ is the optimal path for a given starting point, end point, and starting veloicty, .... This is quite easy to see by contradiction: if there were some faster path given the starting point and velocity, taking it would decrease the overall time, and would obviously not impact the time taken to arrive at the starting point.
