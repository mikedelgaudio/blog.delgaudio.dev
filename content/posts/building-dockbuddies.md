---
title: "Building DockBuddies — Pixel-Art Droids That Watch Your Copilot Sessions"
date: 2026-05-27T22:00:00-04:00
draft: false
tags: ["swift", "macos", "github-copilot", "pixel-art", "side-project"]
categories: "projects"
cover:
  image: "/images/covers/building-dockbuddies.jpg"
  alt: "DockBuddies pixel-art dock robots demo"
---

I've been spending a lot of time lately with [GitHub Copilot CLI](https://githubnext.com/projects/copilot-cli) — spinning up agents to help me write code, run tests, explore repos, the usual. One thing that kept bugging me was that I'd have three or four sessions going across different terminal tabs and completely lose track of what each one was doing. Is that agent still editing files? Did my test run finish? Which tab was working on the auth module again?

So I built [DockBuddies](https://github.com/mikedelgaudio/DockBuddies) — a macOS menu bar app that puts cute little pixel-art robots right above your dock, one per active Copilot session, each showing what it's up to in real time.

<p align="center">
  <img src="https://raw.githubusercontent.com/mikedelgaudio/DockBuddies/main/Resources/demo.gif" alt="DockBuddies Demo" width="480" />
</p>

## The actual problem (and the excuse to build something fun)

Let's be honest — I didn't _need_ a pixel-art status monitor. I could have just checked my terminal tabs. But there's something satisfying about glancing at the bottom of your screen and seeing a little orange droid bouncing with "EDITING" above its head while the green one next to it says "RUNNING TESTS." You immediately know what's happening without switching context.

Double-clicking a buddy snaps you straight to the terminal tab running that session. That part I actually use constantly.

## What I really wanted to learn

More than anything, this was an educational project. I wanted to understand how GitHub Copilot CLI works under the hood — specifically, how to detect and monitor active sessions without any official API. There's no `copilot list-sessions` command, so I had to figure it out myself.

Here's what I found by poking around `~/.copilot/`:

### 1. Lock files for session detection

Each active Copilot CLI session creates a lock file at `~/.copilot/session-state/<session-id>/inuse.<pid>.lock`. The filename embeds the PID, so you can check if the process is still alive with a simple `kill(pid, 0)` call. If the PID is dead, the session is stale — ignore it.

### 2. SQLite for session metadata

Copilot CLI stores session metadata in `~/.copilot/session-store.db`. Querying it gives you the repo name, branch, working directory, a summary of what the session is doing, and more. Standard SQLite — you can just open it with any SQLite library.

### 3. Event streams for real-time status

Each session writes an `events.jsonl` file with structured events — tool executions, user messages, assistant responses. Reading the tail of this file tells you what the agent is _actively_ doing right now: editing a file, running a search, thinking, etc.

Combining all three gives you a surprisingly complete picture of every active session without touching any private APIs.

## The pixel art was the fun part

I wanted the agents to feel alive, not just be status indicators. Each buddy is a 16×16 pixel sprite drawn entirely in code — no image assets. There are four color palettes (orange, green, red, teal) and each robot has idle animations:

- **Bounce** — a gentle sine wave with per-agent phase offsets so they don't move in lockstep
- **Blink** — random intervals between 2.5 and 5 seconds, with a 150ms eye-close
- **Antenna glow** — the little antenna tips pulse brighter and dimmer

These are tiny details, but they make a big difference. The droids feel like they're actually doing something, which makes the whole experience of monitoring sessions feel less like watching paint dry.

## The tricky part: focusing the right terminal tab

Detecting sessions was straightforward compared to the terminal-switching feature. When you double-click a buddy, it needs to activate the correct terminal app _and_ switch to the exact tab running that session.

The approach:

1. Walk up the process tree from the Copilot PID using `sysctl` to find the parent terminal process
2. Activate the terminal app with `NSRunningApplication`
3. For **Ghostty** (my daily driver): match the session's TTY against Ghostty's child processes to identify the right tab index, then send `Cmd+<number>` via `CGEvent`
4. For **Terminal.app** and **iTerm2**: AppleScript to find and select the matching tab

Getting this to work reliably across seven different terminal emulators was probably the most time-consuming part of the entire project. Each terminal has its own quirks around tab management and scriptability. But it's also the feature that makes DockBuddies genuinely useful day-to-day instead of just a novelty.

## What I took away from this

Building DockBuddies taught me a few things I didn't expect:

- **Copilot CLI's local storage is well-structured.** Lock files, SQLite, and JSONL event streams are clean, predictable formats. Even without an official API, you can build reliable tooling on top of it.
- **SwiftUI's `Canvas` is great for pixel art.** Drawing filled rectangles on a canvas is fast enough for real-time animation at small sprite sizes. No need for SpriteKit or Metal.
- **Menu bar apps on macOS are deceptively simple.** No dock icon, no main window — just an `NSStatusItem` and a transparent overlay panel. The boilerplate is minimal once you know the pattern.

This was never meant to be a production tool or anything with a big user base. It was a weekend project that stretched into a week because I kept adding "just one more thing." But that's the best kind of project — the kind where you learn something real while building something that makes you smile every time you see a little robot bouncing on your dock.

If you want to try it out or dig into the code, it's all on [GitHub](https://github.com/mikedelgaudio/DockBuddies). Works on macOS Sonoma and later. And if you have Copilot CLI running, you'll see your first buddy pop up within a few seconds of launching the app.
