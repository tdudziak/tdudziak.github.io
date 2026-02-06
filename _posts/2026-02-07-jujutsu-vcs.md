---
layout: default
title: Jujutsu Version Control System
summary: Jujutsu is a strong backend-compatible Git replacement, particularly for workflows involving AI coding agents.
         This is a short overview of what I find compelling about it.
---

# Jujutsu Version Control System

I've been using Git via its default CLI client for most of my professional career.
I've tried a few command-line or graphical alternatives but always kept coming back.
There's a strong muscle memory associated with daily-used CLI tools that is not worth fighting against when the benefits are marginal.
[Jujutsu](https://www.jj-vcs.dev) (a.k.a. `jj`) is the first Git alternative that really stuck for me and I thought I'd try to write down what really makes it worth it, and why it's
particularly useful if you're using a local AI coding agent like [Codex CLI](https://developers.openai.com/codex/cli/) or [Claude Code](https://claude.com/product/claude-code).

`jj` has lots of features, many of which I'm not really using right now.
This isn't a comprehensive review or an introductory tutorial, just a short summary of what I believe are its main selling points.

## Basic concept

With `jj` you're cloning, pushing, and pulling from an ordinary Git repository.
Almost everything is compatible on the backend, with only a little bit of [extra commit metadata](https://docs.jj-vcs.dev/latest/git-compatibility/#format-mapping-details) attached.
Unless you write a blog post about it, your coworkers will probably never know that you're a massive weirdo using an alternative VCS.

There is no index ("staging area") and no equivalent to `git stash` either.
Instead, certain revisions are marked as mutable.
You can move between them with `jj edit` and the changes you make to the working copy will automatically change the corresponding revision.
This might introduce conflicts in revisions descended from the one you're editing, but that's not a big deal because dealing with conflicts in `jj` is a lot nicer.
You can edit the commit message at any time and the commit is only finalized once it becomes immutable.
This typically happens when you tag it or push it to the main branch upstream, although this behavior is [highly customisable](https://docs.jj-vcs.dev/latest/config/#set-of-immutable-commits).

This makes `jj` conceptually simpler than Git but the downside is that the content-dependent commit ID that you'd use to identify a revision is not stable over time.
For that purpose, every `jj` commit also has a _change ID_, which is randomly generated when it is created and doesn't change depending on the contents.
Change IDs are numbers formatted in "reverse hexadecimal", i.e. using letters `z-k` as digits instead of `0-f`.
You will never confuse the two IDs when looking at the output of `jj log` and, when passing IDs as command arguments, you don't need to specify which kind of ID you mean.

## My typical workflow

Unlike seemingly most people, I've always liked the Git staging area and I initially missed it in `jj`.
Having to stage changes helps to avoid accidentally committing debug code, temporary "TODO" comments, or little stupid notes I write to myself.

In `jj`, I end up using `jj new` and `jj squash` a lot.
The former creates an empty commit right after the current one, the latter moves all the changes from current to the parent commit.
This acts a bit like Git's staging area but with slightly different default behavior---all changes are squashed (a bit like `git add -A`) unless you perform an "interactive squash" with `jj squash -i`, which lets you select changes at a file- or line-level.

You can have multiple "levels" of staging or even different changes in parallel branches.
You can even name them with `jj describe`.
Once you're done, you can squash together your changes to the number of commits you want to push to the Git repository and move a `jj` bookmark (which corresponds to a Git branch) with `jj bookmark set`.

## What does it have to do with AI coding agents?

Not really that much, except I found the "multi-level staging area" workflow useful when editing code together with Codex CLI.
You can always do another `jj new` for the agent and squash it after reviewing.
Or apply your own manual changes on top that the agent will not see or be able to touch.

Another somewhat common scenario is when the AI agent's output is bad enough that, instead of getting it to fix the current state, you think about reverting it and either implementing the change yourself or trying code generation again with more precise prompting.
In that case, you can simply keep this change in parallel in an ephemeral unnamed branch and either discard or use it later.

Another useful piece of `jj` functionality is the ability to use multiple _workspaces_, a feature similar to [Git worktrees](https://git-scm.com/docs/git-worktree).
With `jj workspace add` you can check out a working copy in a different directory.
All workspaces have names and the commit they're editing is marked in the output of `jj log`.

This allows multiple "local collaborators" (human or AI) to edit different commits in the same repository at the same time.
A background edit might even happen on an ancestor revision of the one you're working on, in which case you'll need to run `jj workspace update-stale` for the changes to take effect.
This might create conflicts if you and the agent modified the same parts of the codebase in parallel.

## Conflict resolution

In Git, conflicts can be created as a result of a merge or rebase operation.
Git CLI will then put you into a special state and you need to either resolve the conflict right away or rage-quit and cancel the whole operation.

In `jj` pretty much any editing operation can create a conflict, since you might be editing a revision with many descendants.
Those descendants can include merges or revisions checked out as a working copy of another workspace.
The conflicted commit will be marked with a red scary "conflict" label in the output of `jj log` but nothing forces you to deal with this situation immediately.
You can continue editing the same revision you've been working on, or move to a different one.

To resolve the conflict, you `jj edit` a conflicted revision and your working copy will contain code with ASCII conflict markers.
Once those conflict markers are removed, `jj` will consider the conflict resolved.
You can also use `jj new` and a "staging area" workflow described above, allowing you to review your resolution before you apply it.
You can even resolve one part of the conflict first and then edit an ancestral revision, preventing the other part from materializing in the first place.
It's really neat and pleasant compared to the Git CLI way of doing things, probably my favorite part.

## Everything else

There's a lot more to `jj`: arbitrary "undo" of pretty much any operation, much more reasonable expression language for specifying sets of revisions, lots of commands I didn't mention.
The best resource is [its official documentation](https://www.jj-vcs.dev/latest/).
