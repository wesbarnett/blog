---
title: "snap-pac and snap-sync updates"
description: "Updates on a couple of projects"
layout: post
toc: true
comments: true
hide: false
search_exclude: false
categories: [linux,bash,snapper,btrfs]
---

I've made some updates to a couple of projects that I wrote a few years ago. It's been
quite some time since I've changed anything with them.

Thanks to all who contributed with issues and pull requests in both these projects!

## snap-pac

[snap-pac](https://github.com/wesbarnett/snap-pac) is a set of pacman hooks that take
pre/post btrfs snapshots via snapper. It's just a way to make it easy to rollback the
system after an upgrade on Arch Linux.

Not much changes here other than removing some old information. The hooks and script
remain the same.

[Here is the most recent
release](https://github.com/wesbarnett/snap-pac/releases/tag/2.3.3).

## snap-sync

[snap-sync](https://github.com/wesbarnett/snap-sync) is a bash script that helps use
snapper for incremental btrfs backups.

All of the improvements came as ideas from other users by them filing issues or PR's.
You can see their contributions on Github.

Here I added a quiet option via the `-q` flag so that notifications don't pop up if the
user doesn't want them. I also changed it so that old snapshots used in calculating the
incremental change are no longer kept by default. They can be kept with the `-k` flag
though.

Additionally I added support for `sudo` when performing a backup remotely via ssh and
added the optional dependency `pv` to show a progress bar during the backup.

Lastly, a user has set it up so the package [can now be installed on
Fedora](https://copr.fedorainfracloud.org/coprs/peoinas/snap-sync/).

[Here is the most recent
release](https://github.com/wesbarnett/snap-sync/releases/tag/0.7).

There are a couple of outstanding issues that I'm still working on. One is when there
are multiple mount points for the same subvolume. [Here we want the user to specify a
flag to indicate which mount point to
use.](https://github.com/wesbarnett/snap-sync/pull/99)

Additionally [there's some discussion
ongoing](https://github.com/wesbarnett/snap-sync/pull/97) about what should be listed as
the dependencies of the project.
