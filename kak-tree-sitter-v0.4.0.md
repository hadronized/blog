# kak-tree-sitter v0.4.0 announcement and reflections

> Currently, [Kakoune] doesn’t support writing to files with a timeout, which could help.
> [kak-tree-sitter] is a UNIX server bridging [Kakoune] with [tree-sitter], supporting semantic highlighting,
> text-objects, indent guidelines and more!

I want to use the opportunity of releasing [kak-tree-sitter] in version `v0.4.0` to talk about what’s new but also
explain a bit the new protocol used to make [Kakoune] communicate with the server and vice versa. I think it’s a pretty
fun (and performant!) way of making things, and it can give people ideas.

## What’s new?

This new version adds a bunch of things to [kak-tree-sitter]. Among the interesting things:

- Add support for many languages by default in the shipped `config.toml`.
- Fix PID detection to make it easier to restart / fix the server in case of problems.
- Allow to specif the log level of the server.
- `C-c` support to stop the server if started as standalone.
- Remove `tokio` and replace with `mio`. It’s easier, faster and lighter.
-
- ##
- And obviously, rewrite the internals to use a new protocol.
hannel is not
efore trying to quit the editor).

> Currently, [Kakoune] doesn’t support writing to files with a timeout, which could help.
[Full changelog here](https://github.com/phaazon/kak-tree-sitter/releases/tag/kak-tree-sitter-v0.4.0).


##
Additionnaly, you get `v0.4.1`, which also brings some interesting changes:

- `%opt{kts_lang}` is now used to know how to highlight a buffer instead of `%opt{filetype}` directly. A hook
  automatically calls `kak-tree-sitter-set-lang` (which you can override) to set `%opt{kts_lang}` before talking to the
  server. By default, it simply forwards `%opt{filetype}`. The goal here is to allow people to provide their own
  filetype detection by overriding `kak-tree-sitter-set-lang`.
- Some more internal fixes.

## New protanneocol?hannel is not
efore trying to quit the editor).

> Currently, [Kakoune] doesn’t support writing to files with a timeout, which could help.

So yes, the big improvement is the introduction of a new protocol of communication between [Kakoune] and

##
[kak-tree-sitter]. Before `v0.4.0`, the way things worked was:

1. The user typically starts the server from their `kakrc`, with a line such as
  `eval %sh{ kak-tree-sitter --kakoune --daemon --session $kak_session }`.
2. This blocks the editor while the server is starting. Because `--daemon` is passed, once the server is ready, it’s
  going to daemonize so that its main process exits and Kakoune gets control back.
3. The `--kakoune` argument does one important thing: it asks the server to send some data back to the editor, once the
  server is initialized. The data is sent back via the UNIX socket of Kakoune (`kak -p`). The session is known via
  `--session $kak_session`.
4. The data is basically a bunch of options, commands and hooks that automatically calls back to the server when a
  buffer is open with a supported `filetype`. Each command are wrapped in a `%sh{}` block, and uses the
  `kak-tree-sitter -r` interface to format request as JSON.
5. For highlighting requests, the FIFO interface of Kakoune (i.e. `kak_command_fifo` and `kak_response_fifo`) is used
  to stream buffer content via `write`.

What was the issue with that? Well, all those `%sh{}` implied to open a shell everytime we wanted to perform a request
to the server. And for highligthing, that happens _very often_ (by default, the trigger hook runs for `NormalIdle` and
`InsertIdle`, but you can easily imagin that it runs almost on all keypress modifying the window or the buffer). That’s
a lot. I quickly came to the realization that I wanted something… faster and with less layers.

See, this `kak_command_fifo` and `kak_response_fifo` are pretty interesting FIFO files. The idea is that Kakoune
handles their lifetime, and they work when you do shell expansions (i.e. `%sh{}`). You can write to the
`$kak_command_fifo` Kakoune commands to be executed. This allows to execute Kakoune commands interleaved with shell
commands. You can then use `$kak_response_fifo` to write back result from Kakoune. I used to write the content of
buffers to `$kak_response_fifo` and pass the path to this FIFO in the highlight request so that the server just had
to read from the FIFO and provide the highlights back to Kakoune asynchronously.

FIFOs are wonderful because you can think about them as a 1-1 blocking rendez-vous point between a producer and a
consumer. If a consumer arrives, it blocks until a writer arrives and starts writing, and a writer won’t write until a
consumer arrives. FIFO do not live on your disk, so writing to them (i.e. `write` syscall) is super fast and can be
tought as memory-to-memory communication. It’s honestly pretty good.

However, we still have to open a shell to send the request to the server with this approach… so I needed something else.

## More FIFOs… MOOOOOORE

The idea is actually pretty simple: we can use FIFOs to send request, in the end. If we have one FIFO for each Kakoune
session, we know that we should only have one writer (i.e. the Kakoune session), and we have a single consumer (the
KTS server). Inside Kakoune, writing to files can be done with the `echo -to-file` and `write` command, and we don’t
need a shell for that!

Now the important thing is that Kakoune doesn’t know the path to the FIFO to write requests to, so we still need a
way to get that information. This is done via the `--kakoune` flag. The data that is sent back to the editor contains
options (e.g. `%opt{kts_cmd_fifo_path}`) so that we know where to write requests.

The installed hook, in the `kak-tree-sitter` group, will contain only a few options, mostly set in the `global` scope.
The main hook is run whenever a new window is open on a given buffer. A FIFO request is made to KTS with the filetype
(to be more correct, `%opt{kts_lang}`) and if KTS has the runtime files for this language, it will send back more data
to Kakoune, that will be set for the `buffer` scope. This code will contain more hooks (e.g. `NormalIdle` and
`InsertIdle`) that will make more FIFO requests to, for instance, highlight the buffer.

The content of the buffer is streamed via a FIFO, too — `%opt{kts_buf_fifo_path}`. Using this pair of FIFOs, we can
send requests to the server and stream buffer content by just using `echo -to-file` and `write` Kakoune commands.

> Note: currently, buffers must be entirely read by [kak-tree-sitter] because the diffing algorithm is done by
> [tree-sitter-highlight]. Partial updates using `%val{history}` and `%val{uncommitted_modifications}` are planned for
> a later release.

The only drawback to this approach is that, because we are writing to FIFO files… if one side of the channel is not
there, the other side will block and hang. That could lead to [Kakoune] freezing (for instance, if you kill the server
before trying to quit the editor).

> Currently, [Kakoune] doesn’t support writing to files with a timeout, which could help.

[kak-tree-sitter]: https://github.com/phaazon/kak-tree-sitter/
[Kakoune]: https://kakoune.org/
[tree-sitter]: https://tree-sitter.github.io/tree-sitter/
[tree-sitter-highlight]: https://crates.io/crates/tree-sitter-highlight
