Oh my… 2023 has been such a trek so far… If you have missed my
[previous article about my thoughts about editors and development platforms](https://phaazon.net/blog/editors-in-2022),
I think it’s probably the moment to have a look at it.

Today is April 2023. [Helix](https://helix-editor.com/) has been my primary editor for months now. I haven’t come back
to [Neovim](https://neovim.io/). In the previous article, I mentioned that I couldn’t give a fair opinion about Helix
because I had just started using it. Today, I think I have enough experience and usage (5 / 6 months) to share what I
think about Helix, and go a little bit further, especially regarding software development in general and, of course,
editing software.

However, before starting, I think I need to make a clear disclaimer. If you use Vim, Neovim, Emacs, VS Code, Sublime
Text, whatever, and you think that _“Everyone should just use whatever they want”_, then we agree and **this is not
the point of this blog article**. The goal is to discuss a little bit more than just speaking out obvious takes, but
please don’t use that argument if you start talking about what you use via arguments instead of personal preferences.
If you use arguments, then it means you want to debate, and then you need to be ready to have someone with different
arguments that will not necessarily go your way nor your favorite editor.

Finally, if you think I haven’t used Vim / Neovim enough, just keep in mind that I have been using (notice the tense)
Vim (and by extension, Neovim) since I was 15, and I’m 31, so 16 years.

# My philosophy about software in general

I have received different comments on Reddit on my previous blog articles about people not understanding completely
which side I am when it comes down to editor philosophy. I think it’s a good idea to explain all of that in a more
explicit way, and use the occasion to talk about software in a more general term.

I have always been a Linux user, and I have been embracing the joy of UNIX and its simplicity ever since. The idea of
_“do one thing and do it well”_ is appealing at first, powerful when you start understanding it. Then you realize you
have never completely understood what it meant, and then you start to understand. Yes, it’s appealing and yes, it’s
powerful, but oh damn it’s _necessary_. I’m an advanced and experienced software engineer and I’ve seen many problems,
both at work and on my spare-time projects. There are different approaches and methods to detect and solve those
problems, but there is one thing that I think tells a good software engineer from a more regular developer: being able
to get some hindsight about the _source_ of the problem, the _category_ and _context_ that eventually led to the
problem to even be possible.

All of that is abstract, and I might have lost a couple of readers already. Let me take an example. The one that I
really like is the fuzzy menu one. Imagine that you have to design a fuzzy menu that allows some other code (plugins,
whatever) to prompt the user to select an item in a provided list, and then do something with the provided item. The
correct way to do this, at least to me, is to _split the scope and responsibilities_. Plugins, or really any code that
has a list of items and wants the user to pick one or many, will want _an interface_ to work with. Fuzzy implementations
will implement such interface and might provide additional configuration. So when we think about it:

- Plugins do not have to know how the fuzzy menu is implemented, and they should be using a small / weak interface
  (more on that below).
- The implementations can do whatever they want, as long as they eventually implement the interface.

I would expect trained and experienced developers to split the problem in two: the interface, and the
implementations / backends, and recognize that plugins _only need_ an interface, not an implementation. The split of
scope is important here, because if you let plugins know how to call a specific implementation’s API for fuzzy menu,
then you divide your plugin ecosystem into many fuzzy implementation derived sub-ecosystem. This problem might sound
pretty naive but this is currently a problem that is everywhere in the Neovim community. Many plugins directly depend
on an implementation (FZF, Telescope, etc.) instead of depending on the `vim.ui.select` interface.

0. My philosophy about software in general
1. Why Helix is superior vs. the rest
2. What is itching me about Helix (cognitive dissonance)
3. What is so interesting about Kakoune
4. Any change on my mind?
