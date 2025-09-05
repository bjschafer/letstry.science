---
title: In Search of Organization
date: 2020-11-17T15:56:42-06:00
draft: false

toc: true

taxonomies:
    tags:
      - organization
      - taskmanagement
      - blog
      - rfc
---

I've long struggled with finding an organizational, note taking, and task management system that works for me. I feel like I've tried dang near everything out there, but nothing really "sticks". In this blog post, I focus _mostly_ on technological solutions. While not a hard requirement in my mind, the convenience of not having to keep a physical thing on my person. Plus, my handwriting is atrocious!

A further problem I have is the divide between work and personal. I have different requirements for each (although my requirements for a work organizational system are more stringent); my job encourages us to not use most cloud-hosted/third-party options without prior approval. It also seems like a lot of options that would otherwise work well for work are iOS/Mac exclusive; I don't have a company-provided Apple device.

Some of this boils down to discipline -- some would say I need to just pick something and stick with it. I don't _disagree_, but I also don't exactly agree -- I need to find something that really works with me and meshes with my mindbrain. Others might suggest something like GTD, Inbox Zero, Bullet Journaling, etc. Those are all great concepts, and ones I've explored. However they tend to gloss over implementation details, and there's the rub.

## Categorization

Broadly, I divide systems into 4 different category:

1. ToDo lists
2. Notetaking
3. Semiconverged
4. Fully converged

### ToDo lists

These apps and systems are generally focused on tracking tasks and only tasks. They may support adding notes to tasks but this is not their primary focus.

Examples include:

* Microsoft Todo
* Todoist
* Remember the Milk
* Outlook tasks
* GTD
* Taskwarrior
* Todo.txt
* Kanban (Trello, Kanboard, Taiga...)

### Notetaking

In contrast to ToDo lists, notetaking apps and systems provide means to journal or take notes but don't provide any special todo functionality (or extremely rudimentary functionality).

Examples include:

* OneNote
* Evernote
* Roam/Obsidian/Foam
* Text files
* Journaling/note taking on paper

### Semiconverged

Semiconverged systems provide _some_ integration between note taking and task management, but it's limited in some way.

Examples include:

* Bullet Journaling
* "Classic" OneNote (<=2016)

### Fully converged

What I would refer to as "the holy grail" of organizational systems, fully converged systems provide a means to take notes and todos in a unified fashion.

* Org mode
* Vimwiki + Taskwiki

## Thoughts

As you might surmise, my ultimate organizational system is one in which notes and tasks flow together. Others may call this fluent notes; regardless of how you refer to it, I believe it's the key to successfully organizing your life. That being said, one of the newer trends seems to be around "semantic notetaking", or "life wikis", or some such -- this is things like Roam and Obsidian. While I do believe those are useful and certainly have their place, they do not provide task management functionality.

### Requirements

#### Must haves

* Cross platform
* Combines notes and todos a la orgmode
  * While taking notes can set anything as a task
	* From task view can easily view associated notes
* Sync capable
  * Ideally using git or some other standard
		* Dropbox/Nextcloud?
	* Mobile accessible
* Free form organization of content
	* Easily searchable
	* Easy to reorganize
	* Tags, not (necessarily) categories

#### Nice to haves

* Notate task metadata inline
	* Due date, priority, dependencies
* Ideally textual format
* CLI accessible
* As easy as pen and paper to get started
* Minimal UI

### In search of a solution

As I mentioned, I have tried a lot of different systems. The one that most closely matches with my natural way of thinking is Org mode. It's plain text, notes and tasks mesh perfectly, and it provides enough "smart" features (agenda, tables, code blocks) to make it worthwhile. My primary problem with it, however, is its container: Emacs.

I've been using Vim for as long as I've been using Linux; I'm the guy who, when forced to use other editors/IDEs, looks for and installs a Vim mode plugin. Now, I'm not so extreme as to think the very idea of Emacs anathema. I was willing to give it a try, using Spacemacs in eVIl mode to get me started. Org mode itself is fantastic; unbeatable. But Emacs itself feels bloated, has a lot of weird configuration hiccups that are "not quite" like Vim, and overall just feels wrong.

I have looked at some of the other implementations of Org mode outside Emacs. The problem being, they generally do not include the tight task/agenda integration. They generally just parse the syntax and render it.

I was using Vimwiki + Taskwiki to integrate Vim and Taskwarrior. I got off that boat eventually, though. Setting up your own Taskwarrior server in any sort of modern (read: containerized) platform is nigh on impossible. Plus, the integration was weird and buggy. Concealed metadata that you'd accidentally skip into in a motion; buffers getting de-synced from the task list. Nice in principal, but didn't work out.

So then the thought came around to, "why don't I roll my own?" I know what I want, I could probably write it. Thing of it is, I'd likely as not end up reimplementing Org mode, a task tthat many programmers smarter than me seem to have tried and failed at.

So thus, I find myself once again pining for a solution.
