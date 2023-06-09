I want to talk about Kakoune in this blog article, and more specifically, its UNIX design. See, in my 
[previous](https://github.com/helix-editor/helix/blob/master/LICENSE) blog post, I explained why I love
[Helix](https://helix-editor.com/), and why I love [Kakoune](https://kakoune.org/). The design philosophy of **Kakoune**
really is excellent, but I want to put more relief now. Especially, I will write this blog article along the line of
[kak-tree-sitter](https://github.com/phaazon/kak-tree-sitter), which is a UNIX server and daemon I’ve been writing to
add support of [tree-sitter](https://tree-sitter.github.io/tree-sitter/).

# Kakoune and the shell

Because Kakoune doesn’t have a plugin interface, you are limited to what is called _kakspeak_, which is basically what
you can type in the command line (the `:` line). Contrary to other editors like Vim, everything you type in that `:`
line can be laid out in a file (a `.kak`) and be interpreted directly. Another cool aspect about that is that you can
just select some part of a `.kak` file, and simply type `:^r.<cr>` – in Kakoune, the `.` is the selection register. That
will execute the Kakoune command.

This property is really good, but still, you are limited in what you can do. Among the command you will be running:

- `declare-user-mode`, to declare your own mode (yes, like insert mode, normal mode, etc.). Yes it’s a default / core
  feature and yes it’s excellent.
- `set-face` to declare / update highlighting groups.
- `execute-keys` and `evaluate-commands` to execute keys as if the user typed them, in sandboxed / isolated 
  environments. This is incredibly powerful, as you can run some commands that creates, updates registers, move the 
  cursor around, etc. but since it’s sandboxed, as soon as the command is done, you get back to the state you were in
  before issuing the commands.
- And more.

There is nothing to write plugins… except one thing.

## String expansions

Kakoune has this concept of _expanding strings_. It’s the same feature as in any other editor: some strings can contain
special keywords and identifiers to replace their content with what they hold. For instance, in your shell, it’s very
likely that this string will contain your username: `"$USER"`, or this will be the current year `"$(date +%Y)"`.
However, this will remain verbatim: `'$(date +%Y)'`. The reason is that, in a shell, `"` expands while `'` doesn’t.

Kakoune has the same mechanism, but the syntax is different, and Kakoune has different _source_ of expansions. For
instance, `%opt{foo}` will be replaced why the `foo` option, that can be set with `set-option`. `%val{timestamp}`
contains the current timestamp of the buffer, etc. etc.

There is one interesting expansion that Kakoune has: `%sh{}`. This is _shell expansion_. It will execute its content
inside a thin shell. For instance, try open Kakoune and enter in the command line, something like `:echo %sh{date +%Y}`.
You can see where we are going here. We can use that to run arbitrary shell commands.

Kakoune is monothreaded, so when you run a command in a shell, it blocks until the shell command finishes. On its own,
it’s not that bad. It forces us to call short-living shell commands.

This mechanism allows us to run external programs, but it doesn’t tell us how we can call back Kakoune.

# Kakoune and UNIX sockets

Kakoune is monothreaded, but it’s concurrent. It listens on a UNIX socket Kakoune commands that any external programs
can send, via the `kak -p` interface.

> I initially tried to send content to the UNIX socket programmatically, but it wasn’t planned for that, and hence hit
> issues while doing so. It hurts to say that but you have to spawn a programm running `kak -p`; forget about writing
> directly into the UNIX socket for now.

So, with this mechanism, we can:

1. Run a short-living program via `%sh{}` expansion to talk to, for instance, a server, quickly accepting our request
  and treating asynchronously.
2. Then, once the request is handled, send back the response to Kakoune by running a `kak -p` program, using the UNIX
  socket.
  
The cool thing is that Kakoune follows a server/client architecture and has a _session identifier_ (you must pass it
to `kak -p`). You can retrieve that value with `%val{session}` — and inside `%sh{}` expansion, it’s available as an
environment variable `$kak_session`.

And here you have it. The formula I’ve been using successfully to add **tree-sitter** support to Kakoune. Whenever we
need to highlight the buffer again, simply craft a small request to send to the `kak-tree-sitter` local server and
immediately get control back (so that the UI doesn’t freeze). Then, at some point, the highlight request is computed and
arrives via `kak -p` in Kakoune.

But you might have a question… how do I deal with the buffer content? Indeed, the only thing we can do with with `%sh{}`
is starting a program in a shell. We could do something like laying the buffer content on the shell invocation but 
that’s seriously limiting (especially on giant buffers). We could put the content of the buffer in an environment 
variable, but it would suffer from the same problem (and damn it’s so dirty). What else?

# The final part of the recipe: FIFOs

UNIX systems have this incredible thing called _FIFOs_. I FIFO — also named _pipe_ — is a special kind of file. It lives
on your filesystem (so it’s located at a given path), a bit like UNIX socket. However:

- Its content _never_ exists on the filesystem. It’s all in-memory.
- It’s a streaming primitive, so you cannot just write into it and assume it’s stored somewhere.
- Think of it as a buffer between two entities: a _reader_ and a _writer_.
- When a reader reads from it, it blocks until data is available to read.
- When a writer writes to it, it blocks until a reader is available to read.

So it implements a rendez-vous buffer between a single reader and a single writer. And yes, the `|` in your shell is
using something that behind the scene. The thing is: you can create your own, and you don’t even need to write any code.
Enter the `mkfifo` program. For instance, you can try it out on your own by creating a FIFO (let’s call it `rdv`),
reading its content with `cat` first (you’ll see the `cat` process freeze, so you will need another terminal session!)
and then write content to it, for instance with `echo`:

```sh
mkfifo /tmp/rdv

# in shell 1
cat /tmp/rdv

# in shell 2
echo "Hello, world!" > /tmp/rdv
```

As seen as the `echo` starts writing, you can see `cat` return the result.

> Now replace `cat` with `tail -f` 😏.

Anyway, Kakoune has a mechanism where it scans `%sh{}` blocks and it sees `$kak_command_fifo` and/or
`$kak_response_fifo`, it will create those FIFOs for us (and manage their lifetimes). Because we are in the shell, we
can write Kakoune commands to execute in `$kak_command_fifo`, which will be executed as soon as you’re done writing to
the FIFO, and you can read `$kak_response_fifo` from, for instance, an external program, to get more data from Kakoune.

This is the exact mechanism that is used to stream buffer content between Kakoune and `kak-tree-sitter`. Here’s the
Kakoune commands used to highlight a buffer:

```sh
# Send a single request to highlight the current buffer.
define-command kak-tree-sitter-highlight-buffer -docstring 'Highlight the current buffer' %{
  nop %sh{
    echo "evaluate-commands -no-hooks -verbatim write $kak_response_fifo" > $kak_command_fifo
    kak-tree-sitter -s $kak_session -c $kak_client -r "{\"type\":\"highlight\",\"buffer\":\"$kak_bufname\",\"lang\":\"$kak_opt_filetype\",\"timestamp\":$kak_timestamp,\"payload\":\"$kak_response_fifo\"}"
  }
}
```

I won’t explain the protocol into two much details, but the important part is that I start a shell to run
`kak-tree-sitter … -r`, format the request (it’s just plain JSON), and ask `kak-tree-sitter -r` to read from 
`$kak_response_fifo`. You can see in the previous line that we write to it (the `write $kak_response_fifo` basically
means to write the content of the current buffer to `$kak_response_fifo`, which is a file — everything is a file in
UNIX!).

`kak-tree-sitter -r` is made in a way so that it exits quickly. It will read from the FIFO file and forward the request
to the `kak-tree-sitter` server, which, in turn, will move the request to a different thread so that it can quickly
mark the request done. The whole thing is then synchronous, but extremely fast. The only bottleneck we have here (and
mind it, it’s important) is **that we must read from the FIFO synchronously in the shell `kak-tree-sitter -r`**.

# How’s it going?

I’ve been running with this setup for weeks now, since I started `kak-tree-sitter` around the end of April, 2023. The
code you read above runs in a couple of Kakoune hooks (the same `kak-lsp` is using or very similar), which waits for a
_idle_ time after the buffer is edited (in practice, 50ms after the last edit operation). And it’s already fast enough.

However, there is a nice optimization that we could do here. See, the only reason to start a shell is to format the 
request to send to `kak-tree-sitter`, the server. We can completely by-pass it. Instead, we could:

- Create the FIFO in `kak-tree-sitter`, when a new Kakoune session is created.
- Write the requests to the FIFO handled by `kak-tree-sitter`. No shell is needed to write to files!
- The rest is the same, since the server sends back highlighting Kakoune commands via `kak -p`.

This on its own should greatly enhance performance, which is already pretty good, even with the shell. I plan on working
on this enhancement in the upcoming days.

# Alright, but what’s wrong then?

Well, there is one thing. See, in order to support **tree-sitter**, I had to write this `kak-tree-sitter` server, and
write buffer contents to FIFO files. `kak-lsp` is doing the same. All of that is synchronous, and even if it’s fast,
it’s still two sequential locks of the buffer, copy of the buffer, etc. On my side, I’m currently exploring the
`%val{history}` expanded variable, which contains the textedit operations (so that I don’t have to stream a full 
buffer and diff in `kak-tree-sitter` anymore, but instead just small chunks of updates).

However, the problem still holds. The design of Kakoune is nice when you’re working alone with your small integration,
but when you have two big ones (LSP and tree-sitter are not small environments), I think Kakoune shows its limit.

We – [@krobelus](https://github.com/krobelus) and I – discussed that matter. A middleware program or a change in how
Kakoune interfaces with external programs, especially for buffer streaming, would be greatly appreciated. But it’s not
currently the case. And even if we decide to come up with a middleware – that could act as a buffer cache / map, for
instance – we would still be writing buffer contents to different FIFOs.

Another thing that I’ve been thinking about a lot is… is this the right approach? Something like **tree-sitter** _seems_
to be ideal as an embedded library, directly inside the code of your editor. Especially since Kakoune is all about
selections, being able to have semantic selections by default seems like something pretty good. **tree-sitter** requires
some runtime resources (relocatable objects like `.so` / `.dylib`, queries, etc.)… and it’s the same thing with the
regular Kakoune highlighters (by default, Kakoune doesn’t have any; they come bundled up with your distribution).

I’m not entirely sure the approach scales very well. Of course, I still think the design is excellent, and it prevents
too much maintenance on Kakoune, which is also a good thing. For instance, this runtime resource problem is something
I’m solving in `kak-tree-sitter` (and easing with a controller tool called `ktsctl`, which can download, compile and
install grammars and queries for you), but still. The current version of `kak-tree-sitter` only supports semantic
highlighting, not text-objects just yet (but it shouldn’t be too hard to add). The state of the project is not completly
ready for people to jump in (the wiki is not written), but if you tag along on Matrix, I can help you get started.

Please consider `kak-tree-sitter` as an experimental project, because I’m not sure this is the right approach, even
though it’s probably the only one for Kakoune for now (I don’t think [@mawww](https://github.com/mawww) plans on
integrating it into the core of Kakoune).

Keep the vibes!