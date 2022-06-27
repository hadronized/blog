It’s 2022. Today, developing a library, an application or a service is a totally different experience from what it used
to be only a few years ago. Every professional engineer or programmer is expected to know tools like `git`, a coding
editor, a terminal to run various commands, like compiling, testing, etc. You might also know a lot of people (if you
are not one already) using modern editors and IDEs, such as **VS Code**, **IntelliJ**, etc.

I belong to the category of people who enjoy _composing_ tools. That is, I don’t use an IDE: I make my own IDE by
combining several tools I like. In the end, we all use “IDEs” (we all need some kind of _development environment_).
For the sake of generality, I will call those environments “DE” for _development environments_.

The tools I use are, basically:

- A terminal. Lately, I’ve been enjoying [kitty] a lot, for it allows me to do _almost_ everything I do with [tmux]
  (read below).
- A shell. I’m using [zsh] mostly but I’ve been playing with others, such as [nushell]. I’m dissatisfied with [zsh]
  because I think it’s way more too complicated (I barely use 10% of its features).
- A text editor. I’ve been using lots lately, but mainly, [vim] / [neovim].
- A git client. I simply use the [git] CLI (_Command Line Interface_) program, but also [fugitive] in [neovim].
- A couple of other programs, such as [fzf], CLI of various package managers and compilers, etc.
- I have plugins in [neovim] for note taking and task management, but I’ve also been using [Notion] at work lately to
  try it out.
- I used to use [tmux] a lot but I’m moving away from it, so I’m basically left with [kitty]’s own way to do the same
  thing.

The more that I think about all the things I use and all the things people enjoying modern environments use, the more I
feel I try to “force” a workflow that used to be the right one a couple of years ago, but not necessarily today. I’m the
kind of person who _constantly_ put things into perspective and try to balance my decisions regarding the time I have,
the budget (for work) and the knowledge. A couple of months / years ago, I spent a lot of time on [emacs] and realized
that I needed to review my perspective, because even though [emacs] is super old, it’s fascinating how bleeding edge it
still feels (and I can’t even imagine how bleeding edge it was twenty years ago!).

I want to use this blog article to ask questions and not necessarily answer them, but discuss about them. The questions
are about “how we should be developing in the modern era” and is not about “is [emacs] better than [neovim]” or that
kind of useless debates.

<!-- vim-markdown-toc GFM -->

* [Is tooling composition that any good?](#is-tooling-composition-that-any-good)
  * [The frozen world](#the-frozen-world)
  * [The nightmare that is composition iteration](#the-nightmare-that-is-composition-iteration)
  * [A small word on TUIs, the land of the unicode hacks](#a-small-word-on-tuis-the-land-of-the-unicode-hacks)
  * [Summary of the problems](#summary-of-the-problems)
* [What would be the (next) perfect productivity platform in 2022?](#what-would-be-the-next-perfect-productivity-platform-in-2022)
  * [The terminal needs to change](#the-terminal-needs-to-change)
  * [Editing needs to come back to editing](#editing-needs-to-come-back-to-editing)
  * [Applications need to compose in a new, modern way](#applications-need-to-compose-in-a-new-modern-way)
  * [Applications need to change](#applications-need-to-change)

<!-- vim-markdown-toc -->

# Is tooling composition that any good?

The first question I’m wondering is that whether using several tools and composing them is actually good in 2022. For
instance, the best example I have in mind is [Kubernetes] and [fzf]. At work, I have a workflow where I use [Kubernetes]
and filter/select its output via [fzf] to run other commands. I do the same with [git], btw. I have some scripts where I
can do something like:

```sh
gr -s # equivalent to git branch | fzf | git rebase origin --auto-stash
gs # equivalent to git branch | fzf | git switch
gm # equivalent to git branch | fzf | git merge
kubectx # switch via FZF k8s clusters (kubectl command)
kubens # same but with k8s namespaces
# etc. etc.
```

## The frozen world

All of that seems to be pretty good. We can leverage the power of small tools such as `git`, `kubectl` and `fzf` and
compose them. Shell pipes allow to connect applications together, reading and writing from and to `stdin` and `stdout`.

The “main platform”, the “DE” is then the terminal and the shell, and building a DE requires two main ingredients:

- A set of programs, each of them doing _one thing right_. `fzf` is excellent at building a list of items and let the
  user pick one of them, for instance. `kubectl` allows to operate Kubernetes resources one operation by one operation.
- Some “glue” to combine the results of the programs. This is typically the shell, with its pipes and output pushed to
  the terminal.

However, I think that even though all of that seems pretty great, it suffers from a major drawback: everything is text
and everything is static. See, all of the programs I mentioned here are CLI, not TUI (_Text User Interface_). That is,
the UX is the following:

1. The user enters a command in their terminal, starting with the program name, passing arguments.
2. The program runs, might have side effects on remote resources, etc.
3. The program finishes and displays its results.

That way of working has been a default in the industry for decades. If the program printed more than one terminal page,
you will have to scroll to find your information back up. Once you find the information you need, how do you pick that
information? If you are using a terminal that doesn’t support hinting, you will have to use your mouse (which completely
defeats the purpose of using CLI / TUI applications to me) or the keyboard-base navigation system of your terminal to
make selections, which is, let’s be honest, really poor in most terminal. If, like me, you use a terminal supporting
hinting (i.e. being able to enter a mode that will allow you to visually select parts of the screen by simply typing a
few characters — similar to my [hop.nvim] plugin, but for the terminal), you will be able to copy parts of the screen,
but that’s all. If a program returned a JSON array, it will cost you a lot of keystrokes to reduce the list and, for
instance, use an item in the array as input of another `curl` request, for instance.

The [nushell] has a very interesting concept to solve that kind of problem. Programs write lines of text as output, but
they can also write _cells_, which is a [nushell] encoding that allows it to “speak the same language.” It comes with
some built-in programs such as `open` that will be able to recognize a lot of formats (JSON, XML, YAML, TOML, etc.) and
convert them into _cells_, allowing to display them in the terminal in a very convenient and “smart” way. However, it
doesn’t change the next problem: composition iteration.

## The nightmare that is composition iteration

I think pretty much every programmer out there can relate to the following: you will oftentimes run a command, see its
result, then run the command again, piping it to a filter command or sort or field extractor like [jq]. If the output is
too long, it’s likely you will output into a file and will work on that file (which is basically a cached version of
your command output ;) ).

Iterating on a “session” is a pretty bad experience in a terminal to me. To my knowledge, it would be quite easy to fix
by simply allowing to automatically cache the result of all commands, and get access to it via a shell variable (such as
what we have with `$?` for the last command status, for instance). Some hacks exist to **recompute** the last command
(i.e. `$(!!)`) but it will still re-run the last command.

I thought [nushell] supported that, because the variable support and Nu language are pretty excellent, but to my
surprise, that shell doesn’t have that. Maybe it’s a technical challenge, but I really doubt it. We work with computers
with at least 8 GB of RAM these days and most of us have between 16 and 32 GB of RAM. I don’t think it would be that
hard to support.

But even if we had that caching… it would still require us to run a command, look at a static text, run a filter command
on it, look at the new output that would be appended to the screen… it doesn’t seem super nice in terms of UX to me. And
imagine having to change the second or third command in the pipe list while you already have six pipes. Duh. The
problem, to me, is that the terminal application scope is being overloaded. The CLI is great when we want to run a
command and expect a _report_. As soon as we need to work with _data_, I think the CLI is already lacking features.

## A small word on TUIs, the land of the unicode hacks

And now, let’s talk about TUIs. As a lot of people, I’m using TUIs, so don’t get what I’m about to say twisted. I use
[neovim] as my daily editor driver, and so far, I haven’t really found a way out and use something else. However, it’s
not because nothing better exists that what we use is perfect (far from that).

I have always despised TUIs. I think the idea, which dates from a while, was a fun and interesting idea back then, when
graphical elements and graphics kits were not mature and not a thing yet. But today, we know we can have graphical
elements that align at the pixel level. We know that we can have real vector images in applications. We can use the GPU
to draw arbitrary shapes and some terminals (text-based only!) use the GPU to render glyphs (text), and then people use
TUIs that will use the glyphs to mimic graphical elements! That hitches me!

I dislike TUIs because they are sub-optimal versions of GUIs. And here, I think a lot of people make a confusion with
UX and UI. A GUI can be beautiful, rich of graphical elements, and have a terrible UX (it’s the case most of the time).
People writing GUIs tend to focus on mouse-driven interactions… which doesn’t have to be. So people tend to associate
GUIs with mouse-only workflows, while a GUI can completely do everything a TUI does… but it can _also_ do much better.

A couple of arguments against TUIs:

- Graphical elements are faked by using unicode symbols (drawing blocks), which implies having all the elements laid out
  on a display cell grid. Oftentimes, the unicode symbols will not be correctly aligned, resulting in graphical
  glitches, or simply ugly pop-ups borders, etc.
- Also, still about unicode glyphs, if the application doesn’t implement it correctly, that text is just “part of the
  text of the application”, so copy-pasting can be challenging and very boring.
- Since everything has to be put on a display cell grid, you have no way to have pixel-based movements, alignment or
  anything based on pixel. The smallest unit, the basic primitive, is a unicode character. About that, I think the most
  interesting “fake” thing is [wfxr/minimap.vim](https://github.com/wfxr/minimap.vim), which is a minimap for [neovim]
  in a terminal. And you can’t do any better than that, because, well, you can’t go any smaller / different than a
  glyph.
- Applications run in a terminal, which drives the performance of the TUI application. That results in non-deterministic
  experiences. Basically, you can write a super great application that will run like an elder snail on some terminal
  emulators.

## Summary of the problems

So let’s do a quick recap of what’s wrong with today terminals:

- TUIs are not rich enough in terms of graphical experience, and everything graphical (splits, lines, pop-ups, etc.) are
  somehow faked.
- CLI-based composition often ends up being a nightmare of running the same command over and over by adding more piped
  commands.
- A terminal is a static world, where (besides TUIs) everything is just static text and nothing changes unless you run a
  command again, wiping the previous result and replacing it with the new one.

Those are the three main pain points I feel about using a terminal. Now, you might already have used something like a
monitor application (I work at [DataDog], so I’m used to them) or used a web browser or native GUI and see that you can
have animations, drop-down lists that actually look good, etc. Putting all that in contrast with terminals makes me a
bit sad, especially in 2022.

# What would be the (next) perfect productivity platform in 2022?

My latest blog articles are centered around that topic:

- [My thoughts about editors in 2020](https://phaazon.net/blog/editors-in-2020)
- [My thoughts about editors in (early) 2021](https://phaazon.net/blog/editors-in-2021)

So it’s been an important topic to me lately. I have been thinking about productivity platforms for a while now, and the
conclusion I came to is as follows.

## The terminal needs to change

Instead of switching to an IDE like **IntelliJ** or **VS Code**, I still think we need composition, but not in the form
of pipe shells. I think the CLI has to evolve. I have started a project on my spare-time to try and change how programs
would work with their environment. See, a shell is basically just a way to call the `exec` syscall (`execve` on Linux
for instance) and set its arguments, environment and standard input / output (plus all the candies you can get from a
shell, such as scripting languages, etc.).

The interpretation of the inputs and outputs of a program is actually completely left to… the running process, which is
most of the time the shell for CLI applications. This is why [nushell] is such an interesting new shell: it can
interpret _more_ than just lines of text. And I think we need to move in a direction similar to this.

I think we should see more alternative forms of “terminals”. We could even imagine a program that doesn’t have the
concept of “shell” nor “terminal” anymore. It’s easy to imagine having an application that can simply call `execve` and
do whatever it wants the environment, inputs and outputs. It could have some primitives implemented. For instance,
imagine that a program could be run in your “new terminal” and output its results in a tree view, while other programs
are also running, displaying their own things. You could have a program run without exiting, updating the content of the
tree view or even accepting filters, sorts, field extractor or whatever see fit.

## Editing needs to come back to editing

Today, text editors are overloaded. Heck, even small editors such as [neovim] now have a community developing plugins
that have nothing to do with editing. For instance, fuzzy searching files or Git management. When you think about it, if
the platform (i.e. terminal) had better primitives, it should be possible to compose programs in a way that doesn’t
require people to rewrite the same kind of logic (i.e. fuzzy finders) in all the programs they use. [fzf] is a good
example of that, but it’s limited to the CLI (you can still use it in TUI but to me it’s a bit hacky). Terminals should
have a primitive to, for instance, run a program in overlay and pass the output to another, running program. That would
allow people to write “pure generic fuzzy finders” once and for all, and let people use the editors they like.

Today, all that logic doesn’t really exist outside of individual and scoped efforts. Yes, you can probably find programs
supporting that, but they will certainly not run in a terminal and will have their own protocols.

## Applications need to compose in a new, modern way

Composing applications with pipes is easy to wrap your fingers around, but it gets limited and limiting as soon as
you use something like a GUI. Imagine that programs could speak a common language, based on, for instance, local
sockets, to exchange messages in a way that whatever the kind of programs you write, you can still compose them.
Composing two GUIs between each other should be something that is widely supported, and standardized.

Something else with piping applications: it’s limited to _how we use CLI applications_, which is via the command line.
Running programs in the command line takes the form of:

```sh
program <arg1> <arg2> … | another_program <arg1> <arg2>
```

This is highly summarized, but you get the idea. CLI programs don’t provide a great discoverability experience, and it’s
harder to iterate on them. For instance, in a GUI, if you filter a list with a predicate, it’s super easy to
“experiment around” by changing the filter name. In a CLI session with piped programs, you are likely required to use
your up arrow on your keyboard (or whatever shortcut) to invoke the last command again, and change the arguments in the
middle of the line. While doing this, you will still have to run all intermediary commands, which are going to perform
side-effects, etc. etc. And if you want to be smarter and use the `>` or `>>` operator to cache the results, then you
will still have to remember the name of the file you used, and `cat` or `tee` that file again. It gets time to get used
to using that, and I truly think that it’s not really worth it, because today I’m 30 and I’ve been developing since I’m
11, and I still don’t really enjoy that kind of process.

## Applications need to change

The essence of an application, today, is actually platform-dependent. Heck, it’s even environment dependent. At work,
an “application” can be as simple as a small Python script or be a [Kubernetes] cluster composed of several deployments,
stateful sets and DNS rules. However, when we think about applications we use on a daily basis (what people would call
“heavy” or “native” applications), they are pretty similar to the following:

- For CLI applications, they are akin to effectful computations. They take inputs (environment, `stdin`, arguments),
  they output something (`stdout`) and they can have side-effects (calling another program, changing shared memory,
  doing stuff on your file system, various kinds of I/O, etc.).
- For TUI applications, it’s similar but they don’t read from `stdin` nor output to `stdout` in a one-shot way. Instead,
  it’s like an _event loop_ that reads from `stdin` and a _render loop_ that writes to `stdout`.
- For GUI, it depends on the platform but most of the time, GUIs will be closer to the hardware, because they are not
  restricted by the protocol they are written for (i.e. CLIs / TUIs cannot really go any lower `stdin` and `stdout` in
  terms of UX; GUIs can). A GUI application can decide to use `OpenGL`, `Vulkan` or something higher such as `Gtk`,
  `Qt`, or a Web kit.

I think that we should change that. Remember that running an application is just a matter of calling `execve` (or
similar on non-Linux), which basically requires a path to the application, the list of arguments (the `argv`) and the
environment (`envp`). Setting the environment is typically done by the shell (and some variables come from the terminal
too), but it could be done completely differently. On the same idea, the `argv` is filled by the CLI arguments you
provide, but they could be computed and filled in a completely different way. And finally, the interpretation of
`stdout` (mostly done by the terminal) could be interpreted in a completely different way as well.

Imagine that we could write to `stdout` the port / socket on which we would like to open a communication with a new,
modern protocol, that would allow us to manipulate a new kind of “terminal.” Instead of using a regular text-based
terminal, it would be a piece of software allowing to manipulate pop-ups, text buffers, image / vector image buffers,
notification queues, etc. etc.

Obviously, I’m not inventing all that. If you know about [emacs], you know that that’s basically what they do, but
instead of using `stdin`, `stdout` or `execve`, they just evaluate Elisp. However, what I think could be very
interesting would be to allow native applications to do the same thing, with a native platform that would ship without
any real applications (so unlike [emacs]), and let applications implement their own workflow.

That’s basically what I’ve been working on and experimenting lately. A platform that provides buffers, tree views,
notifications, pop-pups, overlays, etc. to allow any kind of applications that can read and write to sockets to use
those features as if they were native, and platform-independent, in the same way that you use `stdout` without even
thinking how it will be displayed in the terminal.

Leveraging such a platform would solve most of the problems I mentioned above, because:

- We could write a code / text editor that wouldn’t have to be worried about extensions like Git: the Git support would
  come as a native application (equivalent of the CLI `git` application) that would open an overlay, for instance. That
  application wouldn’t even know about the code editor: you would make that connection via the platform.
- Applications  could communicate between each other to manipulate buffers, get the contents of the buffer and do
  various kind of  operations on them (LSP servers, for instance).
- Because the protocol would be completely rendering-agnostic, that would allow people to write an application that
  would work for any kind of “platform implementations.”
- Composition would use the low-level protocol to compose applications. You could imagine supporting an interface between
  applications allowing to filter the contents of the buffer of one by using another application. Applications such as
  [fzf] could _branch_ on the buffers of a text editor, for instance. Or you could even open a buffer in the text editor
  by running a fuzzy finder as another application. All of that would compose in a seamless way.
- Etc. etc.

Eh, of course, I know that such a platform protocol will probably never appear unless I make it, very opinionated and
subject to never be adopted by anyone else but me (or a small group of enthusiasts about that kind of ideas). Maybe this
blog article would have given you some ideas or hindsight about your own tools. Or maybe someone knows such a platform
(that I don’t) and will make me realize it already exists, or is a _work in progress_. Maybe, after all, it will just
launch a discussion about what _developing_ means in 2022. Because to me, using tools that used to be efficient and
proved to be great ones a couple of years ago, doesn’t mean they are still a thing today. And I really mean it, because
I’m currently writing this very article in [neovim], in a [kitty] tab among dozens, and I just wish I had the DE I just
described above.

Keep the vibe!

> [Reddit discussion](https://www.reddit.com/r/programming/comments/vlp76z/development_environments_discussion_about)

[kitty]: https://sw.kovidgoyal.net/kitty
[zsh]: https://www.zsh.org
[nushell]: https://www.nushell.sh
[vim]: https://www.vim.org
[neovim]: https://neovim.io
[git]: https://git-scm.com
[fugitive]: https://github.com/tpope/vim-fugitive
[fzf]: https://github.com/junegunn/fzf
[Notion]: https://www.notion.so
[tmux]: https://github.com/tmux/tmux
[emacs]: https://www.gnu.org/software/emacs
[Kubernetes]: https://kubernetes.io
[hop.nvim]: https://github.com/phaazon/hop.nvim
[DataDog]: https://www.datadoghq.com
