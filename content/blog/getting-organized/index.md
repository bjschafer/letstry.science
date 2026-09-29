+++
title = "Getting organized?"
date = 2026-09-28
draft = true

description = """\
    In this very belated and barely anticipated follow-up to
    [_In Search of Organization_](/blog/in-search-of-organization), I share some
    about what I've done that's worked for both work and personal organization.\
    \
    ...as well as what still needs some improvement.\
"""

[taxonomies]
tags = [
    "organization",
    ]
+++

## Introduction

Once upon a time, in [_In Search of Organization_](../in-search-of-organization), I wrote about my ideal organization system. Interestingly, this was well before I was diagnosed with ADHD. I know roughly what I was thinking, and I was looking _entirely_ for a technical solution to a problem that also requires some amount of mental discipline and effort.
That's not to say though, that having the right tools doesn't make that a whole lot easier!

{{< toc />}}

## Work

For work my holy grail really is the fully converged system. The ability to capture notes and tasks/todos in the same place and manage them in one pane is absolutely killer.

### Practice

The big thing here has been habit stacking. When I'm logging off for the night, I hit a macro
key on my Stream Deck that minimizes my chat window before locking my screen. This way, when I
log in the next morning, I have a chance to cook before getting bombarded by messages from all
the folks over in Europe.

When I first log on for the day, I always sit and drink my coffee. I paired that habit with popping open Obsidian, opening a new daily, and running through some (dopamine-inducing) checkboxes. Simultaneously, I hit another button on my Stream Deck to launch what I call the "Gardener" (more about that below in ##Tooling).

(TODO fill in more details about daily note template)

### Tooling

## Personal

Here, I'm still not fully set and still iterating quite a bit. My needs are quite a bit different from work, I've found out. So trying to find one system that fits all of it is a fool's errand. Whereas at work, I'm pretty much always seated at the same computer, at home
I often just have my phone handy and nothing else.

However, being able to work from multiple devices is a big plus if I can add something from my phone and flesh it out at the computer. Additionally I value collaboration more than at work -- it's handy to be able to put tasks up for both my wife and I to see and grab.

I'm also less keen on fully converged here. It's enough to have a shared Obsidian vault with my wife for writing things down or planning trips, and an entirely separate tool for keeping track of things that need to be done.

### Practice

My biggest challenges here are:

1. A lot of tasks are very open ended. "Go look at the downspout." "Figure out why the hose is leaking."
2. Things that I don't *see* one way or another end up forgotten.
3. I need to reduce friction in capturing a task as much as possible.

With these sorts of challenges in mind, I set out to build something to help handle tasks.

### Tooling

For my three challenges, I built three pieces:

1. Open ended tasks are called "Looks". They're time-bucketed to 15 minutes, and completing a task just asks you to write what you've learned. I can then automatically or manually take that look and turn it into more tasks if need be.
2. A recurring review (1) pings me nightly to do it, and (2) pulls up things stuck on the backlog and asks me if they're still worth doing.
3. The front screen lets you just type, and deal with cleaning it up separately.
