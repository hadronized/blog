# The smoother the better!

It’s been a while I haven’t written anything on my blog. A bit of refreshment
doesn’t hurt much, what do you think?

As a demoscener[^1] I attend demoparties, and there will be a very important and
fun one in about a month. I’m rushing on my 3D application so that I can finish
something to show up, but I’m not sure I’ll have enough spare time. That being
said, I need to be able to represent smooth moves and transitions without any
tearing. I had a look into a few **Haskell** *spline* libraries, but I haven’t
found anything interesting – or non-discontinued.

Because I do need splines, I decided to write my own package. Meet
[smoothie](https://github.com/phaazon/smoothie), my **BSD3** Haskell spline
library.

## Splines

A /spline/ is a curve defined by several polynomials. It has several uses, like
vectorial graphics, signal interpolation, animation tweening or simply plotting
a spline to see how neat and smooth it looks!

![](http://phaazon.net/pub/spline.png)



[^1]: http://en.wikipedia.org/wiki/Demoscene
