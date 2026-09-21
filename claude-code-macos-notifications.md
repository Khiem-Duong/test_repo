# Claude Code: native macOS notifications when a prompt finishes

This setup shows a macOS notification (with a sound) whenever Claude Code
finishes a response or is waiting for your input, but only if VS Code is
**not** the frontmost app. If you are already looking at VS Code, nothing
pops up.

It works for Claude Code inside VS Code. The notification appears under
its own name ("Claude Code") in Notification Center, so you can tune its
style and sound separately in System Settings.

Everything lives in `~/.claude/settings.json`, not in VS Code settings.

## How it works

- Claude Code has **hooks**: shell commands it runs on certain events.
- The `Stop` event fires when Claude finishes a response.
- The `Notification` event fires when Claude needs your attention
  (permission prompt, question, idle waiting).
- Each hook checks which app is frontmost via AppleScript. If it is not
  VS Code (`Code`), it sends a notification through `terminal-notifier`.
- `terminal-notifier` is wrapped in a copy of its own app bundle renamed
  to "Claude Code" with a custom bundle ID. That is what makes the
  notification show "Claude Code" as the sender instead of
  "terminal-notifier".

## Step 1: install terminal-notifier

```bash
brew install terminal-notifier
```

## Step 2: create the "Claude Code Notifier" app bundle

Copy the original app bundle, rename it, change its identity, then
re-sign it (macOS refuses to run a modified bundle with a stale
signature).

```bash
mkdir -p ~/Applications
SRC="$(brew --prefix terminal-notifier)/terminal-notifier.app"
DST="$HOME/Applications/Claude Code Notifier.app"

cp -R "$SRC" "$DST"

/usr/libexec/PlistBuddy -c "Set :CFBundleIdentifier com.yourname.claude-code-notifier" "$DST/Contents/Info.plist"
/usr/libexec/PlistBuddy -c "Set :CFBundleName Claude Code" "$DST/Contents/Info.plist"

codesign --force --deep --sign - "$DST"
```

Replace `com.yourname` with anything you like; it only needs to be
unique on your Mac.

Send one test notification so macOS registers the app in
System Settings > Notifications:

```bash
"$HOME/Applications/Claude Code Notifier.app/Contents/MacOS/terminal-notifier" \
  -title "Claude Code" -message "Test" -sound Glass
```

If nothing appears, open System Settings > Notifications, find
"Claude Code", and set it to Alerts or Banners.

## Step 3: add the hooks to ~/.claude/settings.json

Open `~/.claude/settings.json` (create it if it does not exist). Add a
`hooks` key at the top level. If you already have a `hooks` key, merge
the `Stop` and `Notification` arrays into it.

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "front=$(osascript -e 'tell application \"System Events\" to get name of first application process whose frontmost is true' 2>/dev/null); if [ \"$front\" != \"Code\" ]; then \"$HOME/Applications/Claude Code Notifier.app/Contents/MacOS/terminal-notifier\" -title \"Claude Code\" -message \"Prompt finished\" -sound Glass >/dev/null 2>&1; fi; true"
          }
        ]
      }
    ],
    "Notification": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "front=$(osascript -e 'tell application \"System Events\" to get name of first application process whose frontmost is true' 2>/dev/null); if [ \"$front\" != \"Code\" ]; then \"$HOME/Applications/Claude Code Notifier.app/Contents/MacOS/terminal-notifier\" -title \"Claude Code\" -message \"Claude Code is waiting for your reply\" -sound Glass >/dev/null 2>&1; fi; true"
          }
        ]
      }
    ]
  }
}
```

Restart Claude Code (or start a new session) so it picks up the hooks.

## Step 4: allow VS Code to control System Events

The first time the hook runs, macOS will ask whether VS Code (or the
terminal running Claude Code) may control "System Events". Click Allow.
If you missed the prompt, go to
System Settings > Privacy & Security > Automation and enable
System Events for Code.

Without this permission the `osascript` call returns nothing, `front`
stays empty, and the notification fires every time, even when VS Code
is in front. Still works, just noisier.

## Customizing

- **Sound**: replace `Glass` with any name from
  `/System/Library/Sounds` (Ping, Pop, Purr, Submarine, ...), or drop
  `-sound Glass` for silent notifications.
- **Message text**: edit the `-message "..."` strings.
- **Different editor / terminal**: change `"Code"` in `[ "$front" != "Code" ]`
  to the process name of your app, for example `"Cursor"`, `"iTerm2"`
  or `"Terminal"`. You can check the name with:
  `osascript -e 'tell application "System Events" to get name of first application process whose frontmost is true'`
- **Always notify** (even when the editor is in front): replace the
  whole command with just the `terminal-notifier` call.
- **Click to focus**: add `-activate com.microsoft.VSCode` to the
  `terminal-notifier` call so clicking the notification brings VS Code
  to the front.

## Removing

Delete the `Stop` and `Notification` entries from `~/.claude/settings.json`,
then `rm -rf "$HOME/Applications/Claude Code Notifier.app"` and
`brew uninstall terminal-notifier`.
