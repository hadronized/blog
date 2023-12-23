I have been using [Kakoune] for a long time, and I want to talk about the User Experience (UX). When it comes down to
editors / productivity platforms, people tend to confuse UX with User Interface (UI). Both are related, of course, but
having a great UX doesn’t mean the UI is “great” and having a great UI doesn’t mean the UX is “great”: it’s all a matter
of perspective.

In this blog entry, I want to talk about UX more specifically, because I do think it’s more important than UI. I will
try to show that the default UX of Kakoune is incredible and that you can **very quickly** create a super expressive
and pragmatic programmer environment. It will cover:

- Surrounding pairs.
- Grepping around
- Pickers.
- System clipboard.
- Bonus: some tools I’ve been making.

> Once again, I’d like to greet and thank [@Taupiqueur] for sharing his thoughts and joy about Kakoune.

# What I mean with UI and UX

**UI** means “anything interfacing the user to the system.” It’s both the visual depiction of the service (the menu, the
colors, the fonts, etc.) and the way you interact with the system (with the keyboard, by clicking on buttons, when a
system event happens, etc.).

**UX** means “how the user experiences the system.” For instance, something that is not UI at all but enhances the UX
is having a way to filter data in a system with high volume of events with a tag. An even better UX is having fuzzy
filtering with any tags. Etc.

There are many possible UI implementations for a given UX item: implementing filtering can be done with a select box in
a GUI, but the UX is not great because the user is presented with a set of finite choices, and if there are many, it’s
pretty annoying to scroll down the list to find the one we want. Even with this bad UX design, most of select box
implementations (e.g. web) allow to press keys to jump to the entry in the select options — most of the time, you don’t
see what you type -> bad UX for the user. Another possible UI implementation would be to use a free text box that would
filter based on its content — it could even be live for an even better UX. Etc. etc.

Now, would you prefer a nice looking GUI with the select box, or a blander UI but with the fuzzy search box? Clearly,
in terms of UX, the second option is much better. But now, imagine a GUI with the fuzzy search box. It would be pretty
similar to the blander UI in terms of UX, but it would look (much) better… which is likely to enhance the UX as well!

# Small disgression: TUI vs. GUI

> TUI: Text User Interface, which is a program mimicking traditional GUIs in the terminal.

> GUI: Graphics User Interface, the typical window-based applications you run on your machine.

Before jumping to the Kakoune content, I just want to disgress slightly to talk about something I often see something
that itches me a bit: many individuals seem to say that a TUI is often written in a way that optimizes UX and GUI the
UI, and hence, oftentimes, using a TUI feels much better. I agree with this (this is the reason why I’m using editors
and tools inside my terminal instead of a dedicated GUI, even though I think the UI is worse, for many reasons: no
pixel-perfect alignment of things; no direct integration of the application at the OS level, it has to go through the
terminal and shell; etc. etc.).

However, is there anything forcing a GUI not to provide the same kind of interactions as a TUI? I don’t think so. If
you make your TUI keyboard-oriented, everything you do is just listening to keyboard events provided by whatever
terminal protocol / library you use… which could be done exactly the same way in a GUI.

I do think that we should be able to make a GUI as good as a TUI in terms of UX, and some programs did it. For instance,
[emacs] can run both as a TUI and a GUI (and today, people recommend actually using the GUI). I think that could be the
topic of another blog article. Let’s go back to the original matter.

# Kakoune UX

Kakoune, upon installation, already provides a lot of good things in terms of UX. But as you get more productive with
it, you will face some problems. For instance, the first one I came across (pretty quickly being honest, coming from
[Helix]) was surrounding: adding, deleting and replacing surrounding delimiters. Where you need a plugin to do that in
the Vim world, in Kakoune, it’s another topic. Some plugins exist to do exactly that, but honestly, read along.

Selections in Kakoune — which are so much more than _“iT’s JUsT LikE muLtIcURsOrs oR a nOOb WaY Of DoINg RegEX in
vIM”_ — change the Vim interpretation of _appending_ and _preppending_. In [Vim], `i` will start inserting on the cursor,
whereas `a` will start inserting after the cursor. In Kakoune, `i` inserts **at the beginning of each selection** and
`a` inserts **at the end of each selection**. That’s already a better UX, and it allows to do many things out of the
box.

For instance, since we have the power to insert at the beginning and at the end of each selection… then pressing `i'`
should insert a quote at the beginning of the selection… and `a'` will do the same at the end! So you can already have
a somewhat working surrounding add operation by typing `i'<esc>a'`!

> If you are adventurous, you can read the documentation of `<a-;>`, which allows to leave insert mode to normal mode
> for a single command, and come back to insert mode. You can use this to replicate what we did above without the
> `<esc>` key: `i'<a-;>a'`. Magic.

It’s a bit annoying to have to type all that, though, right? So instead, we could make a command! Commands in Kakoune
are really simple, but they require reading a bit about them. I recommend the following reads if you want to dig in a
bit:

- `:doc commands` to know about all the commands. Search for `^define-command`.
- `:doc execeval`, which explains what `evaluate-commands` and `execute-keys` do. Especially, you will want to read the
  part of `-draft`.

So let’s wrap that sequence of keys in a command. Instead of using `i` and `a`, we are going to use `P` and `p`, which,
as the name implies, _paste_ from the default register. `p` **pastes at the end of the each selection** and `P` **pastes
at the beginning of each selection**.

## Surrounding pairs

What is great is that the default register, `"`, is selection-aware: its content will be different from one selection to
another. Said otherwise, there is one `"` register for each selection. Hence, we can write this:

```
define-command -override my-surround-add -params 2 %{
  evaluate-commands -draft -save-regs '"' %{
    set-register '"' %arg{1}
    execute-keys -draft P
    set-register '"' %arg{2}
    execute-keys -draft p
  }
}
```

And here you go. You can now invoke `:my-surround-add ( )<ret>` to add parenthesis around your selections, for instance.

Kakoune has the concept of user modes, which is a nice feature allowing to declare a keyset (that will be displayed
by the help in the bottom right of your screen) when entered.

```
declare-user-mode my-surround-add
```

We can make one with a bunch of mappings in that user mode:

```
map global my-surround-add (   ':my-surround-add ( )<ret>'         -docstring 'surround with parenthesis'
map global my-surround-add )   ':my-surround-add ( )<ret>'         -docstring 'surround with parenthesis'
map global my-surround-add [   ':my-surround-add [ ]<ret>'         -docstring 'surround with brackets'
map global my-surround-add ]   ':my-surround-add [ ]<ret>'         -docstring 'surround with brackets'
map global my-surround-add {   ':my-surround-add { }<ret>'         -docstring 'surround with curly brackets'
map global my-surround-add }   ':my-surround-add { }<ret>'         -docstring 'surround with curly brackets'
map global my-surround-add <   ':my-surround-add < ><ret>'         -docstring 'surround with angle brackets'
map global my-surround-add >   ':my-surround-add < ><ret>'         -docstring 'surround with angle brackets'
map global my-surround-add "'" ":my-surround-add ""'"" ""'""<ret>" -docstring 'surround with quotes'
map global my-surround-add '"' ":my-surround-add '""' '""'<ret>"   -docstring 'surround with double quotes'
map global my-surround-add *   ':my-surround-add * *<ret>'         -docstring 'surround with asteriks'
map global my-surround-add _   ':my-surround-add _ _<ret>'         -docstring 'surround with undescores'
```

Obviously, that’s not all; we would need to delete delimiters and to replace them. Deleting is actually even more
straightforward. Kakoune has some native mappings to select everything _inside_ or _around_ a set of delimiters:

- `<a-i>` to select _inside_.
- `<a-a>` to select _outside_.

So, pressing `<a-a>(` (or `<a-a>)`, same thing) will select everything around the cursor up to the next pair of
parenthesis.

> Note: I have personally remapped that to `mi` and `ma`, but that collides with the default meaning of the `m` key in
> Kakoune, so I will use the native mapping here instead.

We can then simply use the previous `i` and `a` command mentioned before to start editing at the beginning and end of
the selection. `i<del>` will start insert mode at the beginning of the selection and will remove a character (left
delimiter) and `a<backspace>` will insert at the end of the selection and remove the previous character (right
delimiter). Eh, that looks like so simple it’s almost stupid. But that’s what makes Kakoune so damn good: it’s _simple_
to reason about:

```
define-command -hidden my-surround-delete-key -params 1 %{
  execute-keys -draft "<a-a>%arg{1}i<del><esc>a<backspace><esc>"
}

define-command my-surround-delete %{
  on-key %{
    my-surround-delete-key %val{key}
  }
}
```

Because Kakoune composes really well, you can already imagine that you should be able to use the previous commands and
mappings. And indeed:

```
define-command my-surround-replace %{
  on-key %{
    surround-replace-sub %val{key}
  }
}

define-command -hidden my-surround-replace-sub -params 1 %{
  on-key %{
    evaluate-commands -no-hooks -draft %{
      execute-keys "<a-a>%arg{1}"

      # select the surrounding pair and add the new one around it
      enter-user-mode my-surround-add
      execute-keys %val{key}
    }

    # delete the old one
    my-surround-delete-key %arg{1}
  }
}
```

## Revisiting grepping

Ah… who has never had the problem of trying to locate something in a codebase without _really knowing where to start_.
That happens a lot to me when working on a front-end project and taking an issue asking to fix a random page, that is
broken. Usually, LSPs don’t help to _discover_ things that are based on the final product. For instance, if you see the
checkout page broken — like a `<div>` is missing an attribute or a tag is misplaced in the DOM, no LSP will help you
locate the code that needs to be fixed. Instead, you need other tools.

What I like to do is looking at the page and looking for what I call _unique tokens_. For instance, a short sentence
that might appear only on that page, or modal. Or a header, a title, etc. Then, using that information, I usually grep
the code base. The problem is that, doing that using `grep` or `ripgrep` as CLI is pretty boring, and not very
practical. Indeed, if you get many results, you are likely to try to reduce the result set by constraing more your
regex. Once you have some files, you usually look in your terminal scrollback buffer until you find something
interesting, then open that file in your editor.

People using something like **IntelliJ** products, or **VS Code**, or some plugins with **emacs** or **vim** might have
a way to perform the search from within their editors, but again, that’s not composability: it’s extensibility, and I
explained in [a previous article] why it’s not a good design _to me_.

Kakoune, on the other side, went the _composition_ route. `grep`, `ripgrep`, etc. are all <ins>amazing</ins> tools.
Kakoune comes with a bunch of what it calls _tools_, which are basically Kakoune commands shipped with the editor. Among
those, there is one that is of interest here: the `:grep` command. The `:grep` command forwards its arguments to the
underlying `%opt{grepcmd}` (which defaults to something like `grep -RHn`). Hence, `:grep foo` will run `grep -RHn foo`
in a shell behind the scene, then the result will be output in a `*grep*` buffer. That buffer will get special
highlightings, along with some keybindings, all of that provided by the `grep.kak` tool. If tried the command, you might
have noticed that it’s basically a list of lines of the form:

```
<path>:<line>:<column>:<content-of-the-line>
```

Then, how do you think `grep.kak` implements _pressing `<ret>` jumps to the line, column and file of the line under the
cursor? The cursor can be anywhere on the line. Well, it’s simple: `execute-keys` again! `gh` will put your cursor at
the beginning of the line. Then, you can use `f:` or `t:` to jump to the next `:` — there is no need to parse anything,
we can just programmatically interact with the editor!

Here, `ghT:"py` will go the beginning of the line, select the path and yank it to the `p` register. We can then do
`2l` to move the cursor to beginning of the `<line>` number, and `T:"ly` will yank the line number to the `l` register.
`2l` again to move to the `<column>` section, ten `T:"cy` to yank the column number to the `c` register. And we have
everything we need. We can then just simply run the `edit -existing %reg{p} %reg{l} %reg{c}` command:

aoc-18.md:1:1:lol

```
define-command -override my-jump-for-current-line %{
  evaluate-commands -save-regs clp %{
    execute-keys -draft 'ghT:"py2lT:"ly2lT:"cy'
    edit -existing %reg{p} %reg{l} %reg{c}
  }
}
```

A couple of comments here:

- `-save-regs clp` saves the content of the `c`, `l` and `p` register before evaluating the commands, then restores
  those registers afterwards. That is required since we copy a bunch of data to those registers, but maybe the user is
  already using them. `-save-regs` allows us to specificy a set of registers we locally need, but we do not want those
  registers to bleed outside.
- `-draft` makes the command evaluation run in disposable context. That prevents our selections from being changed —
  `execute-keys` runs a couple of move and goto commands here, while we don’t really want the cursor to move.
- `-existing` fails if the file doesn’t exist.

The `:grep` tool implements something probably very similar to this, but it uses `%opt{grepcmd}`, which you can change
to whatever you like. I personally use:

```
set-option global grepcmd 'rg --column --smart-case --sort path'
```

## Kakoune lacks pickers… or does it?

Something that is very important having around is being able to locate files very quickly. Most editors today ship with
a way to locate files:

- Vim has _netrw_. It’s old. It’s ugly. But it’s there. It’s basically a file browser with a (very) minimal set of
  features.
- VS Code has fuzzy pickers, so you can locate a file by fuzzy searching it.
- Etc.

Kakoune has a default powerful completion engine. Pressin `:edit ` (or, for short, `:e `) and then typing something will
auto-complete the file in the current directory with a somewhat fuzzy algorithm. If you select a directory and type the
trailing `/`, it will list the content of that directory and will auto complete its children. It’s a pretty nice way to
start moving around with the vanilla editor.

However, Kakoune doesn’t ship more than this, because, well, it’s _composable_. You can use any tool you like to locate
files, and then compose them with Kakoune. I personally really like `fd` (a Rust rewrite of `find`). For instance,
`fd --type file` will locate all the files in the current directory. Piping that to a fuzzy finder would allow to
jump to any file in the current directory. And Kakoune supports that. It has several commands for that, but the one you
will be interested at first is `prompt`. It prompts the user for some text and pass the entered text to the provided
commands in the `%val{text}` variable. It supports some switches, and one of interest for us is
`-shell-script-candidates`. That switch accepts a shell command to execute, reading from its standard output
asynchronously and displaying into the completion items. For instance, try running the following command:

```
prompt -shell-script-candidates ls file: %{ info "you selected %val{text}" }
```

It runs `ls`, gets the output and allows you to fuzzy complete the result. Now consider this:

```
prompt -shell-script-candidates 'fd --type file' open: %{ edit -existing %val{text} }
```

And bam. Here you have it. Fuzzy picker in Kakoune. You can map that to a command, or wrap it in a command:

```
define-command my-file-picker %{
  prompt -shell-script-candidates 'fd --type file' open: %{ edit -existing %val{text} }
}
```

If — like many people — you want that to be run when you type `SPC f`:

```
map global user f ':my-file-picker<ret>'
```

You can do the same thing with anything, really. Some plugins like `kak-lsp` uses that to show document symbols, etc.

## System clipboard

By default, Kakoune comes up with registers you can yank to. However, oftentimes, you need to yank and paste from the
system cliboard. Kakoune doesn’t have a _“system register”_ as in Vim. Instead, as with the rest, you have to compose
some tools with Kakoune to get that feature. And it’s pretty simple. You need to know how to yank some text in CLI. Two
situations:

- On **macOS**, we use the `pbcopy` and `pbpaste` to respectively copy and paste from/to the system clipboard.
- On **Linux**, we use the `xclip` and `xsel -ob` commands to do the same.

And then, there is nothing much more to do. We can wrap those in two commands agnostic of the platform by using the
`uname` utility:

```
declare-option str extra_yank_system_clipboard_cmd %sh{
  test "$(uname)" = "Darwin" && echo 'pbcopy' || echo 'xclip'
}

declare-option str extra_paste_system_clipboard_cmd %sh{
  test "$(uname)" = "Darwin" && echo 'pbpaste' || echo 'xsel -ob'
}
```

And the actual commands, using the `!` key (to run a shell command with the current selection piped as standard input
and replace selections with its output) and `<a-!>` (which runs a shell command and ignore its output):

```
define-command extra-yank-system -docstring 'yank into the system clipboard' %{
  execute-keys -draft "<a-!>%opt{extra_yank_system_clipboard_cmd}<ret>"
}

define-command extra-paste-system -docstring 'paste from the system clipboard' %{
  execute-keys -draft "!%opt{extra_paste_system_clipboard_cmd}<ret>"
}
```

I personally map those as in Helix:

```map
map global user y ':extra-yank-system<ret>'  -docstring 'yank to system clipboard'
map global user p ':extra-paste-system<ret>' -docstring 'paste selections from system clipboard'
```

## Bonus: some tools I’ve been making

Because the UX in Kakoune is amazing, and because it composes so well, I have made a couple of binaries and Kakoune
commands to enhance the experience. So far (23rd December of 2023), I have wrote those

- [kak-tree-sitter], which provides [tree-sitter] support (so far, only semantic highlighting; semantic selections will
  come later).
- [bookmarks.kak], a super small plugin I use to create persistent bookmarked locations (and list them in a dedicated
  buffer).
- [git.kak], a super set of the (already great) Git integration in Kakoune. I for instance add a cursor-anchored overlay
  showing git blame information.
- [hop.kak], Hop brought to Kakoune. My implementation is by far _much_ simpler than what I did in [hop.nvim] because I
  read native Kakoune selection descriptions, instead of relying on an internal regex engine run on each line: it
  composes so much better.
- [notes.kak], one of my most useful plugin. It enhances Markdown with some features:
  - Lists marked `- TODO`, `- WIP`, `- DONE`, `- WONTDO`, `- IDEA`, etc. are highlighted with specific faces.
  - Todo items from the list above can be gathered in a buffer (similar to a grep buffer) to jump directly to the note
    file containing them.
  - Open the journal of the day.
  - Open notes.
  - Search journals and notes.
  - Archive and list archives.
  - And much more.
- [swiper.kak], my interpretation of the famous [swiper] emacs package; it allows to reduce a buffer by fuzzy searching
  something in it, only showing the lines that matches, but showing them all at once (by removing the lines that don’t
  match), and pressing `<ret>` jump to the line under the cursor (or `<esc>` goes back to where you were buffer). I
  also included a _reduce_ mode that helps with reducing grep buffers or any buffer that can be directly modified.

All in all, I’m really happy with my current Kakoune setup, and as time passes, I realize it’s a nice _interactive
editing platform_ so far. For instance, at work, I have started making a `k8s.kak` to highlight and run commands on
Kubernetes cluster from within Kakoune; and it works pretty well.

Have fun and keep hacking around!

[Kakoune]: https://kakoune.org
[Helix]: https://helix-editor.com
[emacs]: https://www.gnu.org/software/emacs/
[Vim]: https://www.vim.org
[@Taupiqueur]: https://github.com/alexherbo2
[a previous article]: https://phaazon.net/blog/more-hindsight-vim-helix-kakoune
[kak-tree-sitter]: https://github.com/phaazon/kak-tree-sitter
[tree-sitter]: https://tree-sitter.github.io/tree-sitter/
[bookmarks.kak]: https://github.com/phaazon/bookmarks.kak
[git.kak]: https://github.com/phaazon/git.kak
[hop.kak]: https://github.com/phaazon/hop.kak
[notes.kak]: https://github.com/phaazon/notes.kak
[swiper.kak]: https://github.com/phaazon/swiper.kak
[swiper]: https://elpa.gnu.org/packages/swiper.html
