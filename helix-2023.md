Today is the 1st of January, 2023. I think it’s the right moment to write another blog article about editors and 
productivity platforms. If you haven’t read my previous iterations in that series, here are some links you might want 
to have a look at (not required for this article, though):

- [My thoughts about editors in 2020](https://phaazon.net/blog/editors-in-2020).
- [My thoughts about editors in 2021](https://phaazon.net/blog/editors-in-2021).
- [Neovim plugins stability](https://phaazon.net/blog/neovim-plugins-stability).
- [Development environments](https://phaazon.net/blog/development-environments).

My take on all of that has evolved a bit over the last months / weeks. A couple of things happened and I think it’s
time for an update.

# My views on productivity platforms

If you have read my articles about development environment, you already know what I’m talking about here. If not, 
let me do a little summary. Basically, as a software engineer, I need to use _tools_ to solve problems. Those 
problems are daily little issues while working on a project:

- Organizing my work.
- Searching for stuff, like a file name, some documents, discovering a code base, etc..
- Editing code in an efficient way.
- Building and running programs.
- Testing programs.
- Debugging programs.
- Versioning my code.
- Recording notes, journaling, creating meeting summaries, etc..
- Manipulating program output.
- Composing such output.
- Viewing read-only data and grepping into it.
- Testing an API by issuing HTTP / gRPC / whatever calls to target endpoints.
- Using various systems such as `bazel`, `helm`, `kubernetes`, etc. and composing them with the rest.
- Etc. etc.

Whatever the tools you are using, those problems will roughly be the same. Maybe you don’t have them all, but you
will come across a couple of them on a daily basis. Let’s start talking about what most people use: IDEs.

## IDEs

An [IDE (Integrated Development Environment)](https://en.wikipedia.org/wiki/Integrated_development_environment)
is a software that provides a solution to a subset of the problems I mentioned above. For instance, **IntelliJ 
IDEA** has a solution to edit your code, go to definitions, implementations, lookup files, classes, symbols;
version your code and write Git commit messages; it has a debugger; it has a way to build and test your code; it
has a terminal so you can run random CLI commands; etc. It doesn’t solve all the problems, but clearly it solves
a subset of them. With programs like that, most of the time, you install it, and you’re good to go without 
configuring anything. **JetBrains** editors are famous (and loved!) for this reason. You want to work with Java?
Install **IDEA**. Of course, you will still find plugins to customize the experience, but the _vanilla_ editor is
most of the time enough.

Then you have “customizable” IDEs, such as **VS Code**. Such softwares will often require you to download plugins
because the default experience is unlikely to have support for your language, even though it should be enough for
most people. **VS Code** is clearly the most famous one and it has a plugin for everything you might need (or not 
know you need!). I could have merged the two IDE sections into one, but I do make a difference in my mind because
of how **JetBrains** are advanced and ready-to-be-used. **VS Code**, whether you like it or not, is an important 
piece of software. Microsoft has made a big change with it, since most developers would agree it’s a good editor
and environment to develop in, and it helped introducing things like 
[LSP](https://microsoft.github.io/language-server-protocol/) and 
[DAP](https://microsoft.github.io/debug-adapter-protocol/). You cannot trash-talk **VS Code** in a non-joking way,
they contributed too much and we all use their work.

## The rest

And then, you get into the “do one thing and do it right” way of working, but it’s more complicated than that.
Historically, you would find tools such as **Vim**, **Neovim**, the **git** CLI, **fzf**, terminals, shells, TUI
applications, etc. However, as time passes, there is a trend: people tend to transform those “unit tools” into
IDEs. The terminology doesn’t really matter (whether you want to call that IDE or your own neologism). What matters
is that such tools are not minimal nor “do one thing and do it right” anymore. **Neovim**, for instance, is now
more a Lua interpreter in disguise of an editor, allowing to build via the plugin ecosystem, than a minimal editor.
I was told that I was wrong thinking that Lua wasn’t part of the equation since the beginning, so yeah, I was a bit
on the wrong track from the start. 

It’s the same trend as it has been in the **Vim** ecosystem for so many years. Just look at plugins like 
[vim-fugitive](https://github.com/tpope/vim-fugitive) or [nerdtree](https://github.com/preservim/nerdtree). In 
**Neovim**, you have plugins like [mason.nvim](https://github.com/williamboman/mason.nvim) (which is
basically yet another package manager), [lazy.nvim](https://github.com/folke/lazy.nvim) (same thing but different),
[nvim-tree.lua](https://github.com/nvim-tree/nvim-tree.lua), a file explorer, 
[neogit](https://github.com/TimUntersberger/neogit), a Git implementation in Lua, and the list goes on.

So, yes, I also contribute to that “plugin-based” ecosystem with [hop.nvim](https://github.com/phaazon/hop.nvim)
and [mind.nvim](https://github.com/phaazon/mind.nvim), but I’ve been thinking about all that quite a lot. That adds
up to what I discuss in my article about configuration vs. scripting (basically, I think configuration should 
remain data, not code — which what scripting is about). Quoting something I said on the **Neovim** Matrix room, _“where
people see power, I see weakness”_. A scripting language brings lots of powerful things, such as extensibility, but
it also brings bugs, hence unstability, and a Turing-complete language, preventing the host (i.e. **Neovim** or even
plugins) to easily use the configuration to discover options and data, without having to standardize an API. The 
most infuriating point here to me is colorscheme. They are just scripts. Some of them are even stateful (they cache
things in `~/.cache/nvim`). So “applying a colorscheme” is not simply just switching highlights mapping: it requires
to _execute code_, which makes it super hard to reason about the colorscheme. What’s a colorscheme when you know it
can run an HTTP command?

I’ve been seeing a _lot_ of new plugins, since I am the author and maintainer of 
[This Week in Neovim](https://this-week-in-neovim.org). And recently, I saw a video from 
[TJ DeVries](https://github.com/tjdevries): 
[Effective Neovim: Instant IDE](https://www.youtube.com/watch?v=stqUbv-5u2s). And that video confirms what I said
above **Neovim** becoming an IDE. But it also made me realize something else; TJ uses a plugin as support for 
explaining how to easily turn your favorite editor into an IDE: 
[kickstart.nvim](https://github.com/nvim-lua/kickstart.nvim).

That plugin is basically a single Lua `init.lua` script which serves as a starting init script for your configuration.
You just install it and it downloads a bunch of stuff for you, and calls all the required Lua code to set up correctly
the LSP implementation, [Tree-sitter](https://tree-sitter.github.io/tree-sitter/), downloading managers (yes, plural,
because you need one for plugins and one for LSPs, debuggers, linters, etc.), installing various plugins for 
completion, Git integration, surrounding pairs, etc. And then, I wondered: _“Newcomers are expected to run a
script that can download pretty much anything from the Internet… or write pretty much most of what it does — which is
a lot — on their own?!”_ And the thing is that, my own Lua configuration, which is not based on 
`kickstart.nvim` since I created years ago, has been completely obsolete and I often need to go back to it to remove
or update things, especially regarding LSP, completion, snippets and all.

# Why I think it’s bad

## Do one thing and do it right?

Most people from the community I talked to disagree with my point of view regarding **Neovim**. For them, plugins like
[mason.nvim](https://github.com/williamboman/mason.nvim) are amazing, because they close the gap between their 
editor and the tools required by their editor to work correctly (Mason downloads LSPs / linters / DAP 
adapters / etc.). I used Mason too, but eventually stopped using it when it started downloading a version of
[rust-analyzer](https://rust-analyzer.github.io/) that was released **years ago** (that was a bug in Mason, I 
guess?). I came to the conclusion that I was depending on something doing HTTP calls to download tools that, in 
theory, could be used by other tools on my machines, and that I could also download myself very easily. In the case
of `rust-analyzer`:

```sh
rustup component add rust-analyzer
```

Worse… some of those tools are actually packaged in my package manager (`pacman`), so I’m basically using a tool 
(Mason) that is doing the same thing as a package manager. As if we did not have enough package managers already.

## Too high and wrong expectations

I then continued thinking about all those plugins (among some I use or even have created, like Mind!). Why should
I use them in **Neovim**? I’ve never been a file explorer guy that much, but I know about 
[nvim-tree.lua](https://github.com/nvim-tree/nvim-tree.lua) and… and why do we have to have that in our editor? I
remember the state of mind I was in when I wrote my article about 
[Doom Emacs](https://github.com/doomemacs/doomemacs), which completely changed the way I think about software 
development. **Emacs** doesn’t belong to the “minimal editors”, nor the IDEs directly… it’s a different beast on its own,
but if I had to compare it to something else, I wouldn’t say **VS Code**, or **Neovim**, or anything else. I would
say _“My terminal with all of the commands I run, including an editor, a git implementation, etc.”_ Wait a minute.
Why are we trying to push all those features as plugins into **Neovim**, again?

The reason I didn’t stick around **Emacs** was basically because of its runtime choice, and ultimately, its Lisp 
ecosystem. The **Emacs** community is one of the best I have been talking to, and they have really, really talented 
people working on insanely complex topics, like turning Elisp code into native code (via C), including Emacs itself.
But even with all those optimizations, the editor was still feeling too slow, and has a lot of background that you
can feel. All the abstraction layers, all the Lisp macros (oh no), etc.

Eventually, I went back to thinking about that sentence… that haunting sentence… _“Do one thing, and do it right._”
That sentence has a lot of meaning and I think people have been tearing and bending it to align with their 
conviction, completely ignoring their biases. I read people from the Emacs community stating that yes, Emacs still
applies to that sentence, because _“It’s just a Lisp interpreter.”_ But in the end, the experience users have highly
depends on the plugins, which is the same situation as with **Neovim**. And they have to configure their editor using
a Turing-complete language that might introduce bugs, complex statements (who loves to set up a colorscheme by
conditionally importing / requiring a Lua file located you-have-no-clue-where on your file system?)

Why are we trying to push all those features as plugins into **Neovim**, again? Why am I not trying to focus on using 
tools with a narrower scope, but ensure that the tool remains stable, powerful to use and compose well with the rest?

# Exploring new paths

Lately, I have discovered [Helix](https://helix-editor.com/), a _“post-modern modal editor”_. The first thing I
noticed was that it’s different than **Neovim** in terms of motions. In **Neovim**, in normal mode, you start with a
_verb_, like deleting is `d`, and then you type a motion, like a word is `w`. So typing `dw` on your keyboard will 
delete a word. In **Helix**, it’s reversed. You first select with `w`, and then you apply the verb you want. 
So `wd`. At first, I thought is was a neglible difference. Then I realized how more powerful the [Helix]’ way ways.
Since you “see” the selections, you can select first and then decide what to do, or even change your mind and extend
a bit the selection. You have this nice visual feedback.

And then comes all the good stuff. **Helix** comes with those included features, **requiring zero configuration**:

- An LSP native implementation (the editor is written in Rust, so is the LSP implementation then), with all the 
  features you expect, like preview popups, documentation, signatures, go-to-def, references, implementations, rename,
  code actions, etc.
- Tree-sitter support, including highlights, indents, text-objects, etc.
- DAP support, both the feature and its UI.
- Multi-cursors.
- Surrounding pairs.
- Git gutters.
- Fuzzy finders for buffers, symbols, jump lists, diagnostics, project-wise / global grepping, etc.
- Etc.

And it has no plugin support for now (they plan on adding it at some point, but not for now). And that made me realize:
my editing experience in that editor — even though it took a couple of days to adjust the muscle memory — has been 
flawless. So yes, I miss my `hop.nvim` plugin, but I realized I could hack around by using 
[Kitty hints](https://sw.kovidgoyal.net/kitty/kittens/hints/) until that kind of motions is built-in.

However, I’m just talking about editing, here. I still have that reflex of pressing `SPC g g` to bring up a Git prompt
in my editor to start committing… but **Helix** doesn’t have one. So I’m splitting my terminal into another window and
I use the `git` CLI. And it’s fine. Now, the way I think about it is that I could probably invest time into learning
[lazygit](https://github.com/jesseduffield/lazygit) or anything else.

The point is that my editor is now minimal again. My configuration (which is public, you can find it 
[here](https://github.com/phaazon/config/blob/master/helix/config.toml)) is mostly about 
[bépo](https://bepo.fr/wiki/Accueil) remappings and some very minimal aesthetics changes. The configuration is data
(it’s just a TOML file), and I don’t have to worry about stability anymore since I’m not using any plugin. The fact
that I have an amazing editing experience (even better, honestly, due to the selection then verb principle, 
multi-cursors, out-of-the-box LSP and Treesitter experience, including completion) is just the perfect fit for what
I want.

# Yes, but

But there’s a catch. See, **Helix** is about editing. If you like to have a file explorer in your editor, the way I
would recommend looking at **Helix** is that _you cannot have it in Helix and you should probably use a proper,
standalone file explorer, or consider another alternative like Neovim_. If there is something you would like to add
to **Helix**, you have to open a PR and write some Rust code. You cannot extend it on your own, as it doesn’t have
plugin.

To me, that’s great, because it means the scope is under the responsibility of the **Helix** Team. And I love that. I 
love it because it’s _easy_ to think about the features of my editor. It’s easy because I don’t have to keep 
fearing something break because of an incompatibility between a plugin and the version of the editor I’m running
(or even two plugins between each other).

And honestly, contributing using a statically and strongly typed language (Rust) feels so much _sounder_ to me than
using something like Lua. You can benefit from all the tests of the code base, use the API without any ABI 
conversion in between, and catch bugs at compile time instead of waiting for them to crawl up at runtime.

# So what does it mean for “productivity platforms?”

My view on that hasn’t changed since my last articles. I still think that the terminal needs to be revamped and go
into a direction similar to Emacs. I have started a project a couple of months ago that tries to explore that. 
Basically, I’m making something that is not a terminal, a shell nor editor. It’s “a platform”, with primitives like
tree views, item lists, read-only / read-write buffers, virtual text, popups, command outputs, cells, etc. And it
comes with no way to edit text or file explorers, or anything.

Then, applications targetting that platform can use all those primitives to compose and now the features are
emergent. A text  editor then uses the buffer, virtual text and popup primitives, for instance. A file explorer would 
use the tree view primitive. An LSP client could be a daemon that attaches to edit buffers. Etc. etc.

That’s the dream tool I would love to see, and I still think that Emacs is the closest thing to that, but it comes
with too much legacy to me. And it’s unlikely that my experiment will ever be mature enough to be usable or even 
used. But you know, I like experimenting. A cool project to play with is [nushell](https://www.nushell.sh/). It’s far
from being that dream platform of mine, but it has some very interesting ideas for composing commands that I think
are worth mentioning.

And no, I don’t want **Helix** to become such a platform. Nor **Emacs** to get rid of its legacy and become it. Nor
**Neovim**. I want to keep playing with **Helix** and using it for what it is (and shines for!): editing code. If my
dream platform ever exists, whether it’s mine or someone else’s, I will probably move away from **Helix** to whatever
that platform provides. But such a change would require a standardization, such as `stdout` and `stdin`, but with all
those primitives I mentioned. And I’m not sure whether such a thing will or can exist.

# And Helix? Is it good? 

I’m not going to give my _complete_ opinion on **Helix** just yet. I have been using it for a couple of days only,
and at the time writing, it still has a lot of missing parts / experimental ones. For instance, its DAP support is
experimental, so I can’t judge. What I plan to do is to stick to it and move away from “making my own IDE in an 
editor” and instead enjoying composing tools on the CLI. Then, when I have enough hindsight, I will give a fair review
of **Helix**.

# What about `mind.nvim`, `hop.nvim` and This Week in Neovim?

So, about `mind.nvim`, I plan on rewriting the plugin as a standalone tool so that I can use it whatever the editor.
It will probably do things like `git` when you run `git commit` (opening `$EDITOR`), but I’m still not sure exactly
how I’m going to make it. Maybe I’ll get in touch with people from [Charm](https://charm.sh/) and rewrite it using some 
of their work? I still haven’t thought about it, it’s too early.

About `hop.nvim`, I plan on continuing maintaining it and fixing bugs, even though I haven’t been very active around `Hop`
lately. The reason is mainly a lack of spare-time.

As for **This Week in Neovim**… I honestly do not know. I discussed the project with some people from the **Neovim** core 
team, and I’m a bit stuck. On one side, the community has received it pretty well, given the amount of upvotes I have
on each week release on Reddit; the comments; the appreciation issues on GitHub, etc. I know people have been
enjoying my work, and I’m happy they do.

On the other side, the core team doesn’t seem to have noticed it that much, and none of their members approached me
to talk about it. So I’m not sure what to think. The community enjoys TWiN a lot; the core team doesn’t really care.
Then I need to think about _exhaustion_: I’m really tired of maintaining TWiN. 

See, the idea is to communicate, every week, about what has happened in the **Neovim** world, whether it’s core or 
plugins. What I had initially in mind was to _bootstrap_ the couple of first releases and let people adopt and
contribute to it. On the 2nd of January 2023 is released **TWiN #25**, which means that I’m currently on a 
25 weeks streak. What it basically tells is that, every week (most of the time Sundays), I skim Reddit, GitHub 
projects, man pages, etc. to get as much information as I can, and create a really big PR containing the weekly 
update. That PR is merged and available on the very next day (Monday) for every neovimmers to enjoy reading on Monday 
morning with a nice cup of coffee, tea or whatever you like for breakfast.

So every week, one person (me) spends hours skimming many projects, while what I thought would happen was that
many plugin authors would contribute once every two months a very small text to explain their new plugins / change.
The difference is massive: on one side, you have a single developer doing a big amount of work… every week. On the
other side, you would have many developers doing a very small amount of work every time they release something… which
is clearly not every week (and even then?).

I think I have enough distance with the project to admit I failed marketing my idea. Someone once told me that I was
basically doing free advertisement for plugin authors, which is actually true. People mention they would like to 
donate to contribute and ensure that I keep doing what I do, but I don’t want money — hosting costs me 10€/month and the
domain name is 10€/year, I can sustain that on my own. What I need is contributions. It wouldn’t cost much for a plugin author 
to open a PR to [twin-contents](https://github.com/phaazon/this-week-in-neovim-contents/pulls) and add their
update to the upcoming week. There’s a few regular contributors, writing good PRs I rarely
need to modify. But most of the weeks are contribution-free, and it saddens me even more when I see the reaction of
plugin authors on Reddit, like _“Oh yeah my plugin made it to TWiN!”_ Every time I read that, I think _“Great, next
time maybe they will be pushing the update themselves to help me?”_ And most of the time, they don’t.

So, people started to mention that I should slow down or I will burn out. And I’m honestly pretty fed up with this
_read-only_ relationship: people consume / read; they rarely contribute, even when they could contribute their own
update for their own plugins! I don’t filter out _anything_, as long as it’s about **Neovim** and doesn’t convey any bad
speech (you know the deal). TWiN is about new plugins, updates of existing plugins, tips of the week, blog articles, 
youtube videos, etc. Anything **Neovim** related produced by a member of the community. And even with the exposure of 
TWiN, people still do not contribute. Even after the big refactoring to ease contribution [I announced on 
Reddit](https://www.reddit.com/r/neovim/comments/yeo89j/twin_gets_a_new_collaboration_process_to_ease/). So yes, it’s
a personal failure to market my idea regarding TWiN, and I’m not sure what the next steps are.

Nevertheless, we all learn from mistakes and it’s important to understand them. I will collect my thougths and decide
what to do next. For the time being, I wish you all a Happy New Year, lots of great things, and a tremedous amount of
happy hacking, in your favorite editor, IDE or whether you pipe `echo` commands directly at the end of files!

Keep the vibes!

> [Discuss here](https://www.reddit.com/r/programming/comments/1007yq5/my_thoughts_about_editors_in_2022/)
