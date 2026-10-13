# sshcast

Subscribe to event streams on a remote box over a single persistent SSH
connection. Producers on the remote side stream newline-delimited JSON;
a local daemon keeps the connection and every subscription alive; readers
consume per-topic append-only logs through byte cursors. Reads never touch
the network, so they are always instant and never block.

This is composition, not invention. The pieces are all existing standards:

- **OpenSSH connection multiplexing** (`ControlMaster`) for one TCP
  connection carrying many independent channels
- **autossh** (optional) or `ServerAliveInterval` for self-healing tunnels
- **NDJSON** (newline-delimited JSON) as the event framing
- `journalctl -o json`, `tail -F`, `inotifywait` as producer primitives
- append-only logs plus byte-offset cursors for consumers (the same shape
  as a write-ahead log)

What sshcast adds is the small glue nobody packaged: named topics, a
registry mapping topics to remote producer commands, a supervisor that
re-opens dead channels, and cursor-based reads with named consumers.

## Install

```sh
git clone https://github.com/rayhe/sshcast ~/repos/sshcast
export PATH="$HOME/repos/sshcast/bin:$PATH"
mkdir -p ~/.config/sshcast
cp ~/repos/sshcast/examples/topics ~/.config/sshcast/topics
```

If your host needs extra SSH options (a bastion ProxyCommand, a
non-default config file), set `SSHCAST_SSH_FLAGS`, e.g.
`export SSHCAST_SSH_FLAGS="-F ~/.ssh/bastion-config"`.

Copy the example producers (or write your own) to `~/.sshcast-watchers/`
on each remote host and make them executable.

## Use

```sh
sshcast up jump                      # open the persistent connection
sshcast topics jump                  # list registered topics + remote watchers
sshcast subscribe jump heartbeat     # keep this subscription alive
sshcast status jump                  # mux + subscription state
sshcast read jump heartbeat          # print new events, advance cursor
sshcast read jump heartbeat --peek   # print without advancing
sshcast read jump heartbeat --as deploybot   # separate cursor per consumer
sshcast read jump heartbeat --follow # print new, then follow the log
sshcast unsubscribe jump heartbeat   # stop (log retained)
sshcast prune jump heartbeat --keep 5000
sshcast down jump                     # close everything
```

Keep the supervisor running with systemd:

```sh
cp ~/repos/sshcast/systemd/sshcast@.service ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable --now sshcast@jump.service
```

## The topic registry

`~/.config/sshcast/topics`, one per line:

```
<host> <topic> <remote-command...>
jump heartbeat   .sshcast-watchers/heartbeat --interval 5
jump sysjournal  .sshcast-watchers/sysjournal
jump deploylog   .sshcast-watchers/filewatch /var/log/deploy.log
```

`<host>` is any SSH destination: a `~/.ssh/config` alias or
`user@hostname`. The remote command runs with the multiplexed connection,
so subscribing costs no new handshake.

## Producer contract

A producer is any executable on the remote host that:

1. prints newline-delimited JSON, one event per line, to stdout
2. recommends (not requires) fields `id` (monotonic string) and `ts`
   (ISO-8601)
3. accepts an optional `--since <id>` argument to skip already-seen events

No agent to install on the remote side. A producer can be a one-liner:

```sh
journalctl -f -o json --no-pager
```

## How it works

```
remote host                          local machine
-----------                          -------------
~/.sshcast-watchers/heartbeat --\
~/.sshcast-watchers/sysjournal ---+-> ssh channels (muxed over 1 TCP conn)
~/.sshcast-watchers/filewatch  --/              |
                                                v
                              sshcast daemon: one `ssh` per subscription,
                              stdout appended to per-topic .ndjson logs,
                              dead channels re-opened every 10 s
                                                |
                                                v
                              ~/.local/share/sshcast/<host>/topics/<t>.ndjson
                                                |
                                                v
                              sshcast read: byte-cursor file I/O only.
                              Never blocks, never touches the network.
```

Reads emit only complete lines: a trailing partial line is held back and
the cursor advances past complete lines only, so a writer mid-line never
yields truncated JSON. If a log is truncated under a reader, the cursor
self-heals to the new EOF instead of seeking past it.

## Prior art

- `tail -F` over SSH, tmux, autossh: the primitives. sshcast wires them
  into named, supervised subscriptions.
- [mssh](https://github.com/duanery/mssh): persistent remote shells with
  streaming output. sshcast is topics and logs, not shells.
- Kafka / NATS / log shippers (vector, filebeat): the same
  produce-append-consume shape at much larger scale and operational cost.
  sshcast is for one human watching a few boxes, with SSH as the only
  infrastructure.

## License

MIT. See LICENSE.
