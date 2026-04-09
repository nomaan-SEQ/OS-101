# 00 — Introduction to Operating Systems

## What This Section Covers

Before diving into processes, memory, or file systems, you need the big picture. This section answers the foundational questions: **What is an operating system? Why does it exist? How is it structured?**

Think of this as the map before the journey. Every topic in this repo — from scheduling algorithms to container internals — connects back to the concepts introduced here.

## Why This Matters

- **For interviews:** You'll be asked "What does an OS do?" or "Explain the difference between user mode and kernel mode." These are warmup questions, but a weak answer signals shallow understanding.
- **For daily work:** When your app crashes with a segfault, when Docker containers seem "magical," when you wonder why a syscall is slow — the answers start here.

## Prerequisites

None. This is where it all begins.

## Reading Order

| # | File | What You'll Learn |
|---|------|-------------------|
| 1 | [01-what-is-an-os.md](./01-what-is-an-os.md) | The definition, the two main jobs of an OS, and the abstraction mindset |
| 2 | [02-os-history-and-evolution.md](./02-os-history-and-evolution.md) | How we got here — batch systems to modern OSes (context for *why* things are the way they are) |
| 3 | [03-os-architecture-monolithic-micro-hybrid.md](./03-os-architecture-monolithic-micro-hybrid.md) | The three major ways to structure an OS kernel |
| 4 | [04-user-mode-vs-kernel-mode.md](./04-user-mode-vs-kernel-mode.md) | The most important boundary in computing — and why it exists |
| 5 | [05-how-os-boots.md](./05-how-os-boots.md) | From pressing the power button to a running OS |

## Key Interview Questions This Section Prepares You For

- What is an operating system and what are its primary functions?
- What is the difference between user mode and kernel mode?
- Compare monolithic kernel vs microkernel architectures.
- What happens when you press the power button on a computer?
- Why do we need an operating system at all?
