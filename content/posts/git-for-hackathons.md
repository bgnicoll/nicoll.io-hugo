+++
author = "Brandon Nicoll"
date = 2015-04-15T00:39:16Z
description = ""
draft = false
slug = "git-for-hackathons"
title = "Git for Hackathons"

+++

In a couple weeks, I will be participating in [The Nerdery's Overnight Website Challenge](http://chi2015.overnightwebsitechallenge.com/teams/191) and I have been designated as the team's DevOps. We've picked Git/GitHub for source control and I've been asked to give a quick demonstration on how we can be a more effective team by using it. Many sources online are targeted for individual developers working on personal projects or teams of developers collaborating on big projects (and rightly so). However, source control during a hackathon will need to be simple, straightforward, and just plain work. It should be a helpful tool for sharing code and files quickly instead of a time sink. There won't be time for build masters, branching strategies, peer reviews, or merge conflicts. So how will all this be managed and with a team who is largely unfamiliar or uninterested in Git/source control in general? I have no idea, but feel free to drop me a line if you do. The best way to teach something is to learn it, [right](http://en.wikipedia.org/wiki/Docendo_discimus)?

Having relatively little real-world experience with Git, I'm leaning heavily on the well-written Git and GitHub documentation for this article, plus numerous contributions by an awesome community while trying not to just create a huge link-storm.

### About Git
Git is a source control system originally authored by Linus Torvalds for work on the Linux kernel.[^n] Git is a [distributed version control system](http://git-scm.com/book/en/v2/Getting-Started-About-Version-Control#Distributed-Version-Control-Systems), which means every developer's machine has the database that contains the full history of the project's changes. Whether working alone or with a team, it's typically a good idea to have a server host the repository, which would be called a "[remote](http://git-scm.com/book/en/v2/Git-Basics-Working-with-Remotes)" in Git terms. On top of a bunch of other cool features, GitHub boils down to a remote server.

### Installing Git
[Just go here.](https://help.github.com/articles/set-up-git/) Most(all?) of my team members have Macs, except [me](http://en.wikipedia.org/wiki/Fight_the_Power); I'm certainly not qualified to explain how to install Git on a Mac. GitHub is though.

### Administrators
Version control at your day job is going to be a different story than on a team during a hackathon. If your team is like mine, they mostly don't care about version control until it doesn't do what they want, then they get frustrated and curse the day computers were invented. If you can, try to do some prep work to make it as easy as possible for those sad individuals that don't study open source distributed version control systems in their spare time.

If you're planning on using GitHub during your event, a word of warning. I've been told that Internet connections at hackathons can be unpredictable and that a pragmatic approach would be to prepare for loss of connection. Considering having a spare machine or someone's laptop prepared to take over as a Git server, just in case.

##### Remotes
If you do need to switch which server is hosting the master branch, [this page on remotes](http://git-scm.com/book/en/v2/Git-Basics-Working-with-Remotes) will be useful. It would be a good idea to instruct your teammates to add the possible remotes before the event. It's likely if you do this ahead of time on a physical machine, its IP address will change once you plug in to the event's network. Of course, if you bring your own, complete LAN, this won't be a problem, but if it becomes one, [this guide](https://help.github.com/articles/changing-a-remote-s-url/) will help update your team members.

##### Bare
When creating a repo that will be shared by team members, [that repo should be cloned or initialized as bare](http://www.gitguys.com/topics/shared-repositories-should-be-bare-repositories/). Bare means that no working directory will be created; if you will need to work on the same machine as the shared repo lives, do a normal clone from the bare repo. There is a [difference](http://stackoverflow.com/questions/2199897/how-to-convert-a-normal-git-repository-to-a-bare-one) between converting a normal repo to a bare one and cloning into a new bare repo, but for a hackathon, it probably won't be important.
[Putting the bare repository on a server](http://git-scm.com/book/en/v2/Git-on-the-Server-Getting-Git-on-a-Server#Putting-the-Bare-Repository-on-a-Server)

##### Users And Keys
Asking fellow team members to create SSH keys for the possible remotes and/or GitHub ahead of time will be a time saver during the event; they won't need to worry about remembering or entering credentials on pushes. [The Git documentation](http://git-scm.com/book/en/v2/Git-on-the-Server-Getting-Git-on-a-Server#Small-Setups) explains that you can just create one "git" user on a server and have collaborators send you their public keys. With this method, the commit user data is retained and associated with the original committer, plus you won't need to create multiple temporary user accounts that will likely be unneeded after the event.
[Generating SSH keys](https://help.github.com/articles/generating-ssh-keys/)<br/>
[Generating your SSH public key](http://git-scm.com/book/en/v2/Git-on-the-Server-Generating-Your-SSH-Public-Key)


### General Information On Workflows
[This guide](http://rogerdudler.github.io/git-guide/) does a ridiculously good job of defining a simple, basic workflow with easy to understand commands.


### Developers
As a developer, you'll want to get the latest version of the code before attempting to push your commits to the server. The recommended way of doing this would be with the [pull](http://git-scm.com/docs/git-pull) command because it executes both a [fetch](http://git-scm.com/docs/git-fetch) and [merge](http://git-scm.com/docs/git-merge) at the same time. This means you'll get the latest code committed by your colleagues and Git will attempt to auto-merge your changes in with any differences since you last fetched. If there is a conflict, Git will provide a list of the conflicted files. You will need to bring up each conflict in your text editor and resolve them before being allowed to push your commits to origin.
[Resolving a merge conflict from the command line](https://help.github.com/articles/resolving-a-merge-conflict-from-the-command-line/)<br/>
[Dealing with merge conflicts](http://www.git-tower.com/learn/ebook/command-line/advanced-topics/merge-conflicts)<br/>
[Resolving conflicts](http://githowto.com/resolving_conflicts)<br/>
[Stupid Git tricks](http://webchick.net/stupid-git-tricks)


If you're planning on using Sublime Text, you may want to give [this tutorial](https://scotch.io/tutorials/using-git-inside-of-sublime-text-to-improve-workflow) a read. It explains how to install and use the [Sublime Text Git](https://github.com/kemayo/sublime-text-git) package.

### Designers
As a designer, you'll want a way to get your designs in front of the developers quickly and the developers will want a way to see your designs and changes in design(if any). GitHub has this problem solved in a spectacular way; If you view a commit that contains images you'll see [this](https://github.com/cameronmcefee/Image-Diff-View-Modes/commit/8e95f70c9c47168305970e91021072673d7cdad8). 

[Kaleidoscope](http://www.kaleidoscopeapp.com/) is a really cool looking diff tool that claims to be able to show diffs between image versions and boasts Git integration. 

### Other Links
[Git cheat sheet](https://scotch.io/bar-talk/git-cheat-sheet)<br/>
[Demo of the Bisect functionality](http://webchick.net/node/99) <br/>
[Avoiding garbage commits when interrupted](http://24ways.org/2014/dealing-with-emergencies-in-git/)

##### References
[^n]: http://en.wikipedia.org/wiki/Git_%28software%29