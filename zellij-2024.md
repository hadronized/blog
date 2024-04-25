Today, I want to write about [Zellij] and especially its latest news update[¹].

# Zellij, the modern tmux?

I came into Zellij a wild ago, while I was looking for something more modern over [tmux]. It had the terminal
multiplexer features (windows, splits, focus, etc.), but was lacking features (sessions, for instance, were added
mid-2021[²]). As time passed, it started to get new features and promising. For instance, its layout system, allowing
to have a more minimalistic view (the default one being really all-over-the-place).

And then I realized that some very basic features from tmux were absent:

- When I focus on a pane (`prefix+z` by default in tmux; `ToggleFocusFullscreen` action in Zellij), in Zellij, I don’t
  have any indication the pane is being focused, while tmux has this nice and neat `Z` marker in the statusbar. That
  doesn’t seem like important, but it is, especially when you keep zooming in and out.
- The statusbar… well, tmux has a very-well defined one, highly customizable. Zellij… doesn’t really have one. A
  statusbar is just a plugin that runs on a 1-row layout. You can hide it by switching to a different layout, that
  sounds overly complicated for such a basic feature.
- Sessions cannot be renamed easily; you have to enter something they call the “session manager”.

## Session manager, and the beginning of problems

What I like about a software, like, for instance, [tmux] or [kakoune], is that they are _minimal_. They do one thing and
they do it well. If you are not sure about whether something is _minimal_, ask yourself a simple question: _“What do
we expect from this tool to solve; to do?”_. For a terminal multiplexer, we expect:

- To be able to split a window into panes; navigate the panes; focus them, etc.
- To have several windows (tabs).
- To have sessions.
- To have a customizable statusbar.
- To have some general purpose utilities, such as popups, etc.
- To provide a nice API / RPC system to be able to control the system via scripting / hooks / etc.
- That’s roughly all.

If you provide a few amounts of nice features, and composability tools, then your users can build _more_ by composing
your system with others. For instance, [tmux] has the `display-popup` (`popup`) command that takes a command to run
and displays it in a floating window. That’s great, because now I can do something like `tmux ls -F '#S'` to list all
running sessions, pipe it to my favorite fuzzy picker (`fzf`, `sk`, etc.), and redirect the result to
`xargs tmux switch-client -t`. Something like:

```sh
popup -E "tmux ls -F '#S' | sk | xargs tmux switch-client -t"
```

Make a keybinding for that, and here you go, you have a customized session picker!

On the other side, Zellij ships with its “session manager” — which, honestly looks terrible to me.

<img
  src=https://zellij.dev/img/zellij-session-manager-animated.gif
	width=800
/>

That thing is not something you have control on, and the Zellij CLI Actions[³] won’t help you either.

> Note: we are also in the right to ask ourselves why there is a list of commands (that you can bind to), and a list of
> CLI actions. The former is richer than the latter and it means that the commands you bind to your keys and the
> commands you run on the CLI do not share the same naming / they are not from the same set.

So you’re stuck with a “native” session picker, that could have been pushed to the users. Again, with tmux, it’s a
one-liner, and it uses my favorite fuzzy picker.

## It gets worse

Lately, I have decided to head over the Zellij news[⁴] section to read about recent updates, and as mentioned at the
very top of this article, I was not disappointed. The latest update features some new enhancements, which goes against
the concept of minimalism and even against the initial scope of the project:

- A “welcome screen”, which is just something someone would create with a single script.
- Strider, a file picker. I was so surprised to find that in a terminal multiplexer. I guess you don’t have to use it,
  but the fact it’s there sounds very weird. Yet another thing to maintain in a larger and larger codebase. Hm.
- They introduce `zpipe` to pipe results into plugins. That’s interesting, but that’s also a new burden to maintain
  because of using plugins in the first place, instead of going the composability route. I’m not convinced. With tmux,
  I can simply send the result of a command to a specific session, window, pane. All of that just looks like a
  justification for the extra complexity added by plugins. But at least it makes sense once you have decided to use
  plugins?

## But it can be for you

However, I don’t want to bash on Zellij. I think this very article I’m writing should be my reflection about why Zellij
exists, why it’s not for me (anymore?) and why it could be for you. See, tmux is an old, battle-tested piece of software
that will do exactly what it’s supposed to. If you want more, well, then you have to build it yourself. Because tmux
composes well, it’s an enjoyable process for me, but it might not be for you. Maybe you don’t want to have to learn a
fuzzy picker and the default, native session manager of Zellij will be a breeze for you. Maybe the file picker they have
made will be of great help for you.

I could summarize my thoughts as follows:

- If you are like me and like minimalism, UNIX philosophy and well-scoped softwares, then Zellij is definitly not a good
  pick, and you will be better off it; go to tmux, a TWM, etc.
- If you don’t care about all that philosophy and responsability stuff, then go for it! You’ll probably enjoy it for
  being more user friendly!

[newsboat]: https://newsboat.org/index.html
[Zellij]: https://zellij.dev
[tmux]: https://github.com/tmux/tmux/wiki
[kakoune]: https://kakoune.org
[¹]: https://zellij.dev/news/welcome-screen-pipes-filepicker/
[²]: https://github.com/zellij-org/zellij/blob/main/CHANGELOG.md#0120---2021-05-27
[³]: https://zellij.dev/documentation/cli-actions#cli-actions
[⁴]: https://zellij.dev/news/
