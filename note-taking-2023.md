TL;DR I was wrong about [org-mode]. I know, this is a bit click-bait, but really, I made wrong assumptions and I think I
need to explain my thoughts now.

# Note taking, task management, journaling… how?

As an experienced Software Engineer, I know that I need to multi-task and work on many different projects in parallel,
with different chronicity. Some new tasks can be done only a couple of minutes after discovering / creating them, and
some others will span on days, weeks if not months. That leads to having many parallel things on-going, and since I want
to do my job as correctly as possible… I take notes. I take notes about pretty much _everything_, and I’ve been using
note-taking apps more and more.

Years ago, as I discovered [Doom Emacs], I also discovered [org-mode]. I was blown away by the features brought by
[Doom Emacs], and a bit puzzled with other things. For instance, the agenda (I rarely care about ETA for my personal
notes; that’s something that is tracked at the company-wide level, for instance in the infamous Jira or whatever you’re
using for project management in your team — and I just don’t care on my spare-time). Also, I was a bit doubtful about
having notes laid as plain text (yes, [org-mode] formatted, but still text). What it means is that, there is no database
nor state. Everytime you want to get a list of your `TODO`, [org-mode] has to go through all your `.org` files and
generates the list for you. Back then, I found that a bit weird, but today, I have changed my mind.

# Zettelkasten and other tools

The way I take notes, whatever I do, is to minimize the note files I create. For instance, at work, I have many notes
about commands to run with a specific product. Instead of having a note `That product`, I tend to create smaller notes,
more scoped, like `That product - deploy`, `That product - how to configure`, etc. That allows two important things:

- Notes are shorter, easier to read and maintain.
- Looking up for information and fuzzy searching is easier.

Yes, _fuzzy searching_. Whatever the tool I use, I often need to search for notes, and either I search by note names, or
by content.

A couple of months ago, I wrote [mind], a rewrite of [mind.nvim]. I designed that system to be a _tree editor_, in which
nodes are either “document node” containing some metadata and a path to a file, or a “link node”, with metadata and URL.
Those tools were very useful to me, and I’ve been using them for a long time.

However, as I moved more and more into the UNIX world with [kakoune] and [kak-tree-sitter], I realized that I should
maybe revisit the way I think about notes. For instance, what I often need is to list the on-going tasks I’m doing,
and updating them. Oftentimes (if not all the time), tasks emerged from a context, like a discussion with a colleague,
some notes / brainstorming with myself I’m doing / a meeting / etc. And [mind], [Notion] — or whatever that is using
an external state — are not very suited. The problem relies in _interruption_. You’re in the editor of your choice,
and then you have to switch to another window, look for the task / notes, use some keybindings to change the task to
switch it from `TODO` to `WIP` or `WIP` to `DONE` or `WONTDO` or whatever.

As I was working more and more with [mind], I realized this kind of workflow was starting to annoy me.

# Composing

You start to see the pattern with my blog posts: I enjoy the UNIX world _a lot_. And even though [mind] is perfectly
fine regarding that (I was even able to capture notes by using [tmux]’’s `display-popup` with [mind]), I wanted to
switch to something… easier, yet still UNIX.

Then I started to think again about [org-mode]. See, [org-mode] is eventually just a format specification. The Emacs
mode is the reference implementation, with major modes, keybindings etc. but if Emacs can edit and do stuff to `.org`
files, then the concept is transposable to other tools. The question is: what is the core of [org-mode]? The idea
behind [org-mode] is that there is no state: the notes themselves are the state. There is no need for an external tool.
Any tool can open an `.org` file and edit it. Yes, convenience is better to navigate `.org` notes, but that’s just a
matter of tooling. If you want to open and edit an `.org` file using your favorite editor, you should be able to.

Then, for the rest, [org-mode] has a couple of interesting and useful concepts:

- You write notes in text files, including metadata. So tasks management is also done via `.org` files.
- Listing tasks, for instance, can be performed pretty easily by grepping headings. For instance, `* TODO` can be used
  at the beginning of a line to start a TODO item, that will appear in the listing. That looks arrogantly innocent, but
  it’s really powerful, and I’ll explain why just below.
- [org-mode] has the concept of a _capture_ buffer. The idea is simple: you open it to add stuff right off your mind,
  and you come back to it later to triage its content. This is a fantastic concept, because very often, I need to take
  notes about things I have to do, but I have no structure yet. For instance,
  `* TODO talk to Bob about service-epsilon`. Later on, I might create a note `Epsilon` and move the `TODO` item into
  the note. Since there is no state, the `TODO` will still appears in my todo-list.
- Another important thing is _archiving_. You are encouraged to create many small notes, even for task management. At
  work, I currently have a note called `Back from holiday Oct 2023`, which contains many `TODO` items taken from the
  backlog I read on Slack (basically, stuff my colleagues did and that I want to catch up with). At some point, I will
  be done with all of that, so instead of keeping `DONE` notes around (that will show up in my `DONE` listing), I can
  just archive that note. It will not appear anymore in my fuzzy search results.

All of that is really exciting, because it makes note taking easy and seamless; it doesn’t require any state, and you
can pick the tools you want to implement the workflow.

# It’s not a tool; it’s a workflow

Yes, that’s what I think is the most important thing here. I do not want to use [org-mode] for now. I like Markdown.
Even though I agree `.org` files are more powerful, Markdown, at least for now, is more than enough to me. However, I
wanted to augment it to do things slightly similar to [org-mode].

And so I did it. I created in my [kakoune] a couple of commands to:

- Open a capture buffer located at the same place on my disk. It defaults to `~/notes/capture.md`.
- Quickly capture a line via [kakoune]’’s user prompt and append it to the capture buffer (super useful for todo items).
- Create a new note in `~/notes/notes/`.
- Archive a note into `~/notes/archives/`.
- List / fuzzy search notes.
- List / fuzzy search archives.
- Open a daily journal in `~/notes/journal/<year>/<month>/<day>.md`. It’s super easy with [kakoune] since you have
  `%sh{}` expansion. Just use the `date` command you’re done!
- List daily journals.
- Open task list by:
  - `TODO`, `WIP`, `DONE` or `WONTDO` items.
  - Those items are put into a special buffers on which I have set `buffer` keybindings to jump to the place where the
    item is located. For instance, if I list `TODO` items, pressing `<ret>` on a line will open the appropriate file and
    move my cursor to the right line and column where the `TODO` item is defined.
  - Labels, like `:learn:` or `:hard:` or whatever.
- Local (`buffer`) keybindings to switch tasks to different status.
- A special convenience I created: Markdown files containing a special `<!-- github_project: <user>/<project> -->`,
  usually at the end of the file, allows me to press a keybinding on issue numbers (like `#123`) to automatically open
  them in my browser.
- Synchronize notes with a single keybinding, which is running some `git` commands to rebase / push.
- And many more similar features.

> I plan on releasing the `.kak` file containing all of that if people are interested.

The great thing is that most of the features, like fuzzy searching, are implemented via external tools. I use `fd` and
`rg` fuzzy search todo items. There is nothing to do in [kakoune] besides gluing everything together.

This setup has been my daily driver for months now and I really enjoy it. Being able to take notes and just slide in a
`- TODO I need to check that` right in the middle of the file and having it appear in a listing later is a really
powerful aspect of this workflow… and [org-mode] was doing that all along ever since. Eh, it almost makes me nostalgic
of my time using the Emacs plugin!

I hope that gave me some (not so fresh) hindsight about note taking and task management (and journaling!). Have fun
hacking around!

[org-mode]: https://orgmode.org/
[Doom Emacs]: https://github.com/doomemacs/doomemacs
[mind]: https://github.com/phaazon/mind
[mind.nvim]: https://github.com/phaazon/mind.nvim
[kakoune]: https://kakoune.org
[kak-tree-sitter]: https://github.com/phaazon/kak-tree-sitter
[Notion]: https://www.notion.so/
[tmux]: https://github.com/tmux/tmux
