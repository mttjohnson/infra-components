# Claude Code Remote Control — Quick Reference

Running `claude remote-control` inside a `screen` session so it survives SSH
disconnects and can be accessed from the Claude mobile app or claude.ai/code.

## Start a session (one-liner)

```bash
screen -dmS claude-rc claude remote-control
```

Then attach to grab the session URL or QR code:

```bash
screen -r claude-rc
```

Press **spacebar** inside Claude to display the QR code for your phone.

To detach and leave it running: `Ctrl+A` then `D`.

## Start a session (interactive)

If you'd rather watch it start up:

```bash
screen -S claude-rc
claude remote-control
```

Then `Ctrl+A`, `D` to detach.

## Reconnecting later

```bash
screen -r claude-rc            # reattach
screen -d -r claude-rc         # force-detach any stale attachment, then reattach
```

## Useful commands

```bash
screen -ls                     # list all screen sessions
screen -X -S claude-rc quit    # kill the screen session entirely
```

Inside Claude:

- `/rename homelab` — give the session a friendly name so it's easy to find
  in the mobile app's session list
- `/mobile` — show a QR code linking to the iOS / Android app download
- `/config` — toggle push notifications and other settings
- `/status` — check auth method and subscription
- `/exit` — end the Claude session (screen will keep running, just empty)

## Troubleshooting

`screen -ls` shows `(Attached)` but you can't connect:

```bash
screen -d -r claude-rc
```
