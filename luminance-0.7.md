# luminance-0.7

You can’t even imagine how hard it was to release [luminance-0.7](http://hackage.haskell.org/package/luminance-0.7).
I came accross several difficulties I had to spend a lot of time on but finally, here it is. I made
a lot of changes for that very special release, and I have a lot to say about it!

# Overview

As for all my projects, I always provide people with a *changelog*. The 0.7 release is a major 
release (read as: it was a major increment). I think it’s good to tell people what’s new, but it
should be **mandatory** to warn them about what has changed so that they can directly jump to their
code and spot the uses of the deprecated / changed interface.

Anyway, you’ll find patch, minor and major changes in luminance-0.7. I’ll describe them in order.

## Patch changes

### Internal architecture and debugging

A lot of code was reviewed internally. You don’t have to worry about that. However, there’s a new
cool thing that was added internally. It could have been marked as a minor change but it’s not
*supposed* to be used by common people – you can use it via a flag if you use `cabal` or `stack`
though. It’s about debugging the *OpenGL* part used in luminance. You shouldn’t have to use it but
it could be interesting if you spot a bug someday. Anyway, you can enable it with the flag
`debug-gl`.

### Uniform Block / Uniform Buffer Objects

The [UBO](https://www.opengl.org/wiki/Uniform_Buffer_Object) system was buggy and was fixed. You
might experience issue with them though. I spotted a bug and reported it – you can find the bug
report [here](https://bugs.freedesktop.org/show_bug.cgi?id=92909). That bug is not Haskell related
and is related to the i915 Intel driver.

## Minor changes

The minor changes were the most important part of luminance-0.7. luminance now officially supports
*OpenGL 3.2*! So any 
