# Git Branching #

Nearly every VCS has some form of support for branching. Branching is when you diverge from one line of development and start working on another line. In many VCS tools, this is a somewhat expensive process, often requiring creating a new copy of your source code directory, which can take a long time and use up a lot of disk space for large projects.

Some people refer to Git’s branching model as its “killer feature”, and it certainly sets Git apart in the VCS community. Why is it so special? Git branches are incredibly lightweight, making branching operations nearly instantaneous. Unlike many other VCSs, Git encourages frequent branching and merging, even multiple times a day. Understanding and mastering branching gives you a powerful and unique tool and can literally change the way that you develop.

## What a Branch Is ##

To really understand the way Git does branching, step back and examine how Git stores its data. As you may remember from Chapter 1, Git doesn’t store a series of changesets or diffs, but instead a series of snapshots. Each snapshot is represented in the Git repository by a commit object. Each commit object contains, among other things, zero or more pointers to the commits that precede it. The first commit object in a repository doesn’t contain any pointers — since it’s the first commit there’s nothing to point to. A normal commit contains just one pointer, which points to the commit that comes before it. There’s a special kind of commit, called a merge, that contains multiple pointers because it merges two or more branches together.

In addition to these pointers, a commit object also stores information about the author and committer of the commit, a message summarizing the commit, and the size of the commit object itself. Finally, Git computes the SHA-1 hash of the commit object. Since this hash is guaranteed to be different for each commit object (can you see why?) the hash serves as a unique identifier for each commit. For the rest of this chapter you only need to pay attention to commit objects since they’re all that’s necessary for understanding branching. I’ll go into more details about the other kinds of objects in Chapter 9.

To visualize this, assume that you have a directory containing three files which you stage and commit into an empty Git repository.

	$ git add README test.rb LICENSE
	$ git commit -m 'initial commit of my project'

When you run `git commit`, Git creates a commit object with an SHA-1 hash that starts with 98ca9. Conceptually, your Git repository looks something like Figure 3-1.

Insert 18333fig0301.png
Figure 3-1. Single commit repository data.

If you make some changes then stage and commit again, the new commit object stores a pointer to the preceding commit object. After two more commits, your repository might look something like Figure 3-2.

Insert 18333fig0302.png
Figure 3-2. Git object data for multiple commits.

A branch in Git is simply a named pointer to a commit. The default branch name in Git is `master`. As you make commits, Git automatically moves `master` to point to the last commit you made, as shown in Figure 3-3.

Insert 18333fig0303.png
Figure 3-3. Branch pointing into the commit history.

A repository with just one branch isn’t very interesting. What happens when you create a new branch? This simply creates a new named pointer — nothing more, nothing less. To illustrate, let’s say you create a new branch called `testing` by running `git branch`.

	$ git branch testing

This creates a new pointer called `testing`. It initially points to the same commit as the branch you’re currently on (see Figure 3-4).

Insert 18333fig0304.png
Figure 3-4. Multiple branches pointing into the commit’s data history.

How does Git know what branch you’re currently on? Git maintains a special pointer, called `HEAD`, that always points to the branch you’re currently on. Note that this is a lot different than the concept of HEAD in other VCSs you may be used to, such as Subversion or CVS. The `git branch` command only creates a new branch — it doesn’t switch to that branch by changing what `HEAD` points to (see Figure 3-5). So, at this point, you’re still on `master`. 

Insert 18333fig0305.png
Figure 3-5. HEAD file pointing to the branch you’re on.

To switch to a different branch, run `git checkout`. Let’s switch to the new `testing` branch.

	$ git checkout testing

This changes HEAD to point to the `testing` branch (see Figure 3-6).

Insert 18333fig0306.png
Figure 3-6. HEAD points to a different branch when you switch branches.

That's nice but now there are three pointers to the same commit `f30ab`. What’s the point of this? Well, let’s do another commit.

	$ vim test.rb
	$ git commit -a -m 'made a change'

Figure 3-7 illustrates the result.

Insert 18333fig0307.png
Figure 3-7. The branch that HEAD points to moves forward with each commit.

This is interesting, because now your `testing` branch pointer has moved forward to point to the new commit, but your `master` branch pointer didn’t change. Let’s switch back to the `master` branch.

	$ git checkout master

Figure 3-8 shows the result.

Insert 18333fig0308.png
Figure 3-8. HEAD moves to another branch on a checkout.

That did two things. It moved the HEAD pointer back to point to the `master` branch, and it quickly reverted the files in your working directory back to the snapshot that `master` points to. This ability to quickly revert back to a previous snapshot is something that seems almost miraculous at first. In other VCSs, switching between branches can take a noticeable amount of time. In Git, it’s almost instantaneous. I show how this is done so quickly in later chapters.

Changes you make from this point forward will diverge from what’s on the `testing` branch. Git essentially rewinds the work you’ve done on your testing branch temporarily so you can go in a different direction.

Let’s make a few changes and commit again.

	$ vim test.rb
	$ git commit -a -m 'made other changes'

Now your project history has diverged (see Figure 3-9). You created and switched to a branch, did some work on it, and then switched back to your other branch and did other work. All this work is isolated in separate branches: you can switch back and forth between the branches and merge them together when you’re ready. You did all that with simple `git branch` and `git checkout` commands.

Insert 18333fig0309.png
Figure 3-9. The branch histories have diverged.

Because a branch in Git is actually a simple file that contains the 40 character SHA-1 hash of the commit it points to, branches are cheap to create and destroy. Creating a new branch is as quick and simple as writing 41 bytes to a file (40 characters and a newline). Also, because Git records the parent commit(s) when you commit, it’s easy for Git to find the snapshots it needs to do merging. 

Let’s see how to do some branching and merging.

## Basic Branching and Merging ##

Let’s go through a simple example of branching and merging that might be similar to what you’d do in the real world. Follow these steps:

1. Do work on the master branch for your company’s web site.
2. Create a branch for a new story you’re working on.
3. Do some work in the new story branch.

At this stage, you receive a call that the web site has a critical issue and you need to develop a hotfix. You do the following:

1. Revert back to the master branch.
2. Create a branch to add the hotfix.
3. After it’s tested, merge the hotfix branch into the master branch, and put the new code into production.
4. Switch back to your new story branch and continue working.

### Basic Branching ###

First, let’s say you’re working on a project and have a couple of commits already in the master branch (see Figure 3-10).

Insert 18333fig0310.png
Figure 3-10. A short and simple commit history.

You’ve decided that you’re going to work on issue #53 in the issue-tracking system your company uses. To do this, create a new branch to work in. To create the branch `iss53` and switch to it at the same time, run `git checkout` with the `-b` switch.

	$ git checkout -b iss53
	Switched to a new branch "iss53"

This is shorthand for

	$ git branch iss53
	$ git checkout iss53

Figure 3-11 illustrates the result.

Insert 18333fig0311.png
Figure 3-11. Creating a new branch pointer.

Work on your web site and do some commits. Doing so moves the `iss53` branch forward since this is your current branch, as shown in Figure 3-12.

	$ vim index.html
	$ git commit -a -m 'added a new footer [issue 53]'

Insert 18333fig0312.png
Figure 3-12. The iss53 branch has moved forward with your work.

Now you get a call that there’s an issue with the web site, and you need to fix it immediately. At first you’re worried because the changes in the `iss53` branch aren’t finished yet. With Git, you don’t have to deploy your fix along with your `iss53` changes, and you don’t have to revert those changes before you can work on your fix. All you have to do is switch back to your `master` branch.

However, before you do that, note that if your working directory or staging area has uncommitted changes that conflict with content on the branch you’re checking out, Git won’t let you switch branches. You’ll see an error message like the following:

	$ git checkout master
    error: Your local changes to the following files would be overwritten by checkout:

followed by a list of the files containing uncommitted changes.

It’s best to have a clean working state when you switch branches. There are ways to get around this that I’ll cover later. For now, you’ve committed all your changes, so you can switch back to your `master` branch.

	$ git checkout master
	Switched to branch "master"

At this point, your working directory is exactly the way it was before you started working on issue #53 so you can concentrate on your hotfix. This is an important point to remember: Git restores your working directory from the snapshot of the commit pointed to by the branch you check out. It adds, removes, and modifies files automatically to make this happen.

Next, you have a hotfix to make. Create a `hotfix` branch to contain the fix, create a fix, and then commit it (see Figure 3-13).

	$ git checkout -b hotfix
	Switched to a new branch "hotfix"
	$ vim index.html
	$ git commit -a -m 'fixed the broken email address'
	[hotfix]: created 3a0874c: "fixed the broken email address"
	 1 file changed, 0 insertions(+), 1 deletion(-)

Insert 18333fig0313.png
Figure 3-13. hotfix branch based back at your master branch point.

Run your tests to make sure the hotfix fixes the bug, and merge it back into your `master` branch, which will then be ready to deploy to production. Do this with the `git merge` command.

	$ git checkout master
	$ git merge hotfix
	Updating f42c576..3a0874c
	Fast forward
	 README |    1 -
	 1 file changed, 0 insertions(+), 1 deletion(-)

Notice the phrase "Fast forward" in that output. Because the commit pointed to by the branch you merged in (C4) was directly upstream of the commit you’re on (C2), Git simply moves the `master` branch pointer forward (to C4). To phrase that another way, when you try to merge one commit (C4) with a commit (C2) that can be reached by following the first (C4) commit’s history, Git simplifies things by moving the pointer forward because there’s no divergent work to merge together — this is called a "fast forward".

Your change is now in the snapshot of the commit pointed to by the `master` branch, and you can deploy your change (see Figure 3-14).

Insert 18333fig0314.png
Figure 3-14. Your master branch points to the same place as your hotfix branch after the merge.

After your super-important fix is deployed, you’re ready to switch back to the work you were doing before you were interrupted. However, first delete the `hotfix` branch, because you no longer need it — the `master` branch points to the same place. Delete it by running `git branch -d`.

	$ git branch -d hotfix
	Deleted branch hotfix (was 3a0874c).

Now switch back to `iss53`, your work-in-progress branch and continue working on it. Commit when you’re finished. (see Figure 3-15).

	$ git checkout iss53
	Switched to branch "iss53"
	$ vim index.html
	$ git commit -a -m 'finished the new footer [issue 53]'
	[iss53]: created ad82d7a: "finished the new footer [issue 53]"
	 1 file changed, 1 insertion(+), 0 deletions(-)

Insert 18333fig0315.png
Figure 3-15. Your iss53 branch can move forward independently.

It’s worth noting here that the work you did on what was your `hotfix` branch (now in C4) is not contained in the commits on your `iss53` branch (C3 and C5). If you need to include that work on your `iss53` branch, merge your `master` branch into your `iss53` branch now by running `git merge master`, or wait to integrate those changes until you decide to merge the `iss53` branch back into `master` later.

### Basic Merging ###

Suppose you’ve decided that your `iss53` work is complete and ready to be merged into your `master` branch. In order to do that, merge in your `iss53` branch, much like you merged in your `hotfix` branch earlier. All you have to do is check out the branch you wish to merge into and then run `git merge`.

	$ git checkout master
	$ git merge iss53
	Merge made by recursive.
	 README |    1 +
	 1 file changed, 1 insertion(+), 0 deletions(-)

This looks different than the `hotfix` merge you did earlier. In this case, your development history has diverged from some older common ancestor (C2). Because the commit (C4) on the branch you’re on isn’t a direct ancestor of the latest commit (C5) in the branch you’re merging in, Git has work to do. In this case, Git does a simple three-way merge, using the two snapshots pointed to by the branch tips (C4 and C5) and their common ancestor (C2). Figure 3-16 highlights the three snapshots that Git uses to do its merge in this case.

Insert 18333fig0316.png
Figure 3-16. Git automatically identifies the best common-ancestor merge base for branch merging.

Instead of just moving the branch pointer forward, Git creates a new snapshot that results from this three-way merge and automatically creates a new commit object (C6) that points to it (see Figure 3-17). This is referred to as a merge commit and is special in that it has more than one parent.

It’s worth pointing out that Git determines the best common ancestor to use for its merge base. This is different than CVS or Subversion (before version 1.5), where the developer doing the merge has to figure out the best merge base. This makes merging a heck of a lot easier in Git than in these other systems.

Insert 18333fig0317.png
Figure 3-17. Git automatically creates a new commit object that contains the merged work.

Now that your work is merged in, you have no further need for the `iss53` branch. Delete it and then manually close the ticket in your ticket-tracking system:

	$ git branch -d iss53

### Basic Merge Conflicts ###

Occasionally, merging doesn’t go smoothly. If you changed the same part of the same file in different ways in the two branches you’re merging, Git won’t be able to merge them automatically. For example, if your fix for issue #53 modified the same part of `index.html` that you modified in the `hotfix` branch, you’ll get a merge conflict that looks something like

	$ git merge iss53
	Auto-merging index.html
	CONFLICT (content): Merge conflict in index.html
	Automatic merge failed; fix conflicts and then commit the result.

Git didn’t automatically create a new merge commit. Instead, you have to first manually resolve the conflict. To see which files are unmerged at any point after a merge conflict, run `git status`.

	[master*]$ git status
	index.html: needs merge
	# On branch master
	# Changes not staged for commit:
	#   (use "git add <file>..." to update what will be committed)
	#   (use "git checkout -- <file>..." to discard changes in working directory)
	#
	#	unmerged:   index.html
	#

Anything that has merge conflicts that haven’t been resolved appears as unmerged. Git adds standard conflict-resolution markers in the files that have conflicts, so you can open them and manually find and resolve those conflicts. In this case, `index.html` contains a section that looks something like

	<<<<<<< HEAD:index.html
	<div id="footer">contact : email.support@github.com</div>
	=======
	<div id="footer">
	  please contact us at support@github.com
	</div>
	>>>>>>> iss53:index.html

This means the version pointed to by HEAD (your `master` branch, because that was your current branch when you ran the merge command) is in the top part of that output (everything above the `=======`), while the version in your `iss53` branch is in the bottom part. In order to resolve the conflict, either choose one version or the other, or merge the contents yourself. For instance, you might resolve this conflict by removing the the entire block marked by the `<<<<<<<` and `>>>>>>>` lines in `index.html` on your `master` branch and replacing it with this.

	<div id="footer">
	please contact us at email.support@github.com
	</div>

This resolution contains a little from each possibility. After you’ve resolved a conflicted file, run `git add` on it. Staging the file marks the merge conflict as resolved.
To use a graphical tool to resolve these issues, run `git mergetool`, which fires up an appropriate visual merge tool and walks you through the conflicts.

	$ git mergetool
	merge tool candidates: kdiff3 tkdiff xxdiff meld gvimdiff opendiff emerge vimdiff
	Merging the files: index.html

	Normal merge conflict for 'index.html':
	  {local}: modified
	  {remote}: modified
	Hit return to start merge resolution tool (opendiff):

To use a merge tool other than the default, enter the name of the tool you’d rather use from the “merge tool candidates” list. In Chapter 7, I’ll discuss how to change this default.

After you exit the merge tool, Git asks if the merge was successful. If you say yes, Git stages the file to mark it as resolved.

Run `git status` again to verify that all conflicts have been resolved.

	$ git status
	# On branch master
	# Changes to be committed:
	#   (use "git reset HEAD <file>..." to unstage)
	#
	#	modified:   index.html
	#

If everything that had conflicts has been staged, run `git commit` to finalize the merge commit. The commit message by default looks something like

	Merge branch 'iss53'

	Conflicts:
	  index.html
	#
	# It looks like you may be committing a MERGE.
	# If this is not correct, please remove the file
	# .git/MERGE_HEAD
	# and try again.
	#

Modify that message with details about how you resolved the merge if you think it would be helpful to others looking at this merge in the future — why you did what you did, if it’s not obvious.

## Branch Management ##

Now that you’ve created, merged, and deleted some branches, let’s look at some branch management techniques that will come in handy when you begin using branches all the time.

The `git branch` command does more than just create and delete branches. If you run it with no arguments, it shows a list of the branches in your repository.

	$ git branch
	  iss53
	* master
	  testing

Notice the `*` character in front of `master`: it indicates your current branch. This means that if you commit now, the commit will go on the `master` branch. The `master` pointer will be moved forward to point to the new commit. To see the last commit on each branch, run `git branch -v`.

	$ git branch -v
	  iss53   93b412c fix javascript issue
	* master  7a98805 Merge branch 'iss53'
	  testing 782fd34 add scott to the author list in the readmes

Another useful option for examining the state of your branches is to filter the output from `git branch -v` to show which branches need to be merged into your current branch, and which branches have already been merged. The `--merged` and `--no-merged` options to `git branch` do this. To see which branches are already merged into your current branch, run `git branch --merged`.

	$ git branch --merged
	  iss53
	* master

Because you already merged in `iss53` earlier, you see it in the output. Branches without the `*` in front are generally safe to delete. Since you’ve already merged their contents into another branch, there’s no reason not to delete them.

To see all the branches that contain work you haven’t yet merged in, run `git branch --no-merged`.

	$ git branch --no-merged
	  testing

This shows any branches containing work that isn’t merged in yet. Trying to delete such a branch with `git branch -d` will fail.

	$ git branch -d testing
	error: The branch 'testing' is not an ancestor of your current HEAD.
	If you are sure you want to delete it, run 'git branch -D testing'.

If you really do want to delete the branch and lose the work it contains, force the deletion with `-D`, as the helpful message points out.

## Branching Workflows ##

Now that you have the basics of branching and merging down, I’ll cover some common workflows that Git’s lightweight branching makes possible, so you can decide whether to incorporate them into the way you work.

### Long-Running Branches ###

Because Git uses a simple three-way merge, merging from one branch into another multiple times over a long period is generally easy to do. This means you can have several branches always open to use for different stages of your development cycle. Merge them early and often.

Many Git developers have a workflow that embraces this approach, having only entirely stable code in their `master` branch — possibly only code that has been or will be released. They have another parallel branch named `develop` to test stability — it isn’t necessarily always stable, but whenever it gets to a stable state, it can be merged into `master`. It’s used to pull in topic branches, which are short-lived branches, like your earlier `iss53` branch, when they’re ready. This approach requires topic branches pass all tests before going into `master`.

In reality, this approach simply moves branch pointers along the line of commits you’re making. The stable branches are farther to the left in your commit history, and the bleeding-edge branches are farther to the right (see Figure 3-18).

Insert 18333fig0318.png
Figure 3-18. More stable branches are generally farther to the left in commit history.

It’s generally easier to think about a collection of silos, where sets of commits graduate to a more stable silo when they’re fully tested (see Figure 3-19).

Insert 18333fig0319.png
Figure 3-19. It may be helpful to think of your branches as silos.

Keep doing this for multiple stability levels. Some larger projects also have a `proposed` branch containing branches that may not be ready to go into the `next` or `master` branch. The idea is that your branches are at various levels of stability. When they reach a more stable level, merge them into the branch above.
Multiple long-running branches aren’t necessary, but they’re often helpful, especially when dealing with very large or complex projects.

### Topic Branches ###

Topic branches, however, are useful in projects of any size. Again, a topic branch is a short-lived branch for a single particular purpose. This is something you’ve likely never done with a VCS before because it’s generally too expensive to create and merge branches. But in Git it’s common to do so several times a day.

You saw this in the last section with the `iss53` and `hotfix` branches. You did a few commits in them and deleted them after merging them into your main branch. This technique allows you to context-switch quickly and completely — because your work is separated into silos where all the changes have to do with a specific topic, it’s easier to see how the code changes over time. You can keep the changes in the topic branches for minutes, days, or months, and merge them in when they’re ready, regardless of the order in which they were created or modified.

Consider an example of doing some work on `master` (C0 and C1), creating the `iss91` branch off of `master` for an issue, working on `iss91` for a bit (C2 and C3), creating the `iss91v2` branch off of `iss91` to try another way of handling the same issue (C4, C5, and C6), going back to `iss91` to try a few more clever ideas (C7 and C8), going back to `master` and working there for a while (C9, C10, and C11), and then creating `dumbidea` off of `master` to do some work that you’re not sure is a good idea (C12 and C13). Your commit history will look something like Figure 3-20.

Insert 18333fig0320.png
Figure 3-20. Your commit history with multiple topic branches.

Now, let’s say you decide you like the second solution to your issue best (`iss91v2`). You showed the `dumbidea` branch to your coworkers, and they thought it was pure genius. Throw away the original `iss91` branch (losing commits C7 and C8) and merge in the other two (C2 and C3). Finally, merge `dumbidea` (C12 and C13) and `iss91v2` (C4, C5, and C6) onto `master` to be your new production version (C14). Your history then looks like Figure 3-21.

Insert 18333fig0321.png
Figure 3-21. Your history after merging in dumbidea and iss91v2.

It’s important to remember when you’re doing all this that these branches are on your local computer. When you’re branching and merging, everything is being done only in your Git repository — no communication with a remote server is happening.

## Remote Branches ##

Remote branches are branches in remote repositories. They act like local branches that you yourself can’t move but rather Git adjusts automatically whenever you communicate with a remote repository whose branches have changed since the last time you communicated with it. Remote branches act as bookmarks to remind you where the branches on remote repositories were the last time you communicated with them.

A remote branch name takes the form `(remote)/(branch)`. For instance, to refer to what the `master` branch on the `origin` remote looked like the last time you communicated with it, use the name `origin/master`. Let’s say you’re working on issue #53 with a partner and they created a branch named `iss53` that they pushed to the `origin` server. To refer to the branch on the server, use the name `origin/iss53`.

This may be a bit confusing, so let’s look at an example. Let’s say you have a Git server on your network named `git.ourcompany.com`. If you clone from this, Git automatically creates the name `origin` in your Git repository, copies all the data from `git.ourcompany.com` into your Git repository, and creates a pointer named `origin/master` in your repository pointing to where the `master` branch was in the remote repository when you did the clone. You can’t move `origin/master` yourself. Git also creates your own `master` branch starting at the same place as origin’s `master` branch, so you have something to work from (see Figure 3-22).

Insert 18333fig0322.png
Figure 3-22. A Git clone gives you your own master branch and origin/master pointing to origin’s master branch.

If you do some work on your local master branch, and, in the meantime, someone else changes the master branch on `git.ourcompany.com`, then the commit histories in the two Git repositories move forward differently. As long as you don’t contact `git.ourcompany.com`, `origin/master` in your local Git repository doesn’t move (see Figure 3-23).

Insert 18333fig0323.png
Figure 3-23. Working locally and having someone push to your remote server makes each history move forward differently.

To synchronize your work, run `git fetch origin`. This command fetches any data from the `origin` server (in this case `git.ourcompany.com`) that you don’t yet have in your Git repository, and then updates your Git repository by moving `origin/master` to its new position (see Figure 3-24).

Insert 18333fig0324.png
Figure 3-24. The git fetch command updates your remote references.

To demonstrate having multiple remote servers and what remote branches for those remote projects look like, let’s assume `git.team1.ourcompany.com` is another Git server that’s used only by one of your sprint teams. Add it as a new remote reference to the project you’re currently working on by running `git remote add` as I described in Chapter 2. Name this remote `teamone`, which will be your shortname for that remote URL (see Figure 3-25).

Insert 18333fig0325.png
Figure 3-25. Adding another server as a remote.

Now, run `git fetch teamone` to fetch everything on the remote `teamone` server that you don’t already have. Because that server only has a subset of the data on the `origin` server right now, Git fetches no data but creates a remote branch pointer called `teamone/master` in your Git repository pointing to the commit that `teamone` has as its `master` branch (see Figure 3-26).

Insert 18333fig0326.png
Figure 3-26. You get a reference to teamone’s master branch position locally.

### Pushing ###

To share a branch in your Git repository, push it to a remote Git repository that you have write access to. Keep in mind that your local branches aren’t automatically synchronized with the remote branches they came from — you have to explicitly push the branches you want to share. That way, you can keep some branches private for work you don’t want to share, and push only the branches you want to collaborate on.

If you have a branch named `serverfix` that you want to work on with others, push it the same way you pushed your first branch. Run `git push (remote) (branch)`.

	$ git push origin serverfix
	Counting objects: 20, done.
	Compressing objects: 100% (14/14), done.
	Writing objects: 100% (15/15), 1.74 KiB, done.
	Total 15 (delta 5), reused 0 (delta 0)
	To git@github.com:schacon/simplegit.git
	 * [new branch]      serverfix -> serverfix

This takes your local `serverfix` branch and pushes it to update the remote `serverfix` branch. If you don’t want the remote branch to be called `serverfix`, you could instead run `git push origin serverfix:awesomebranch` to push your local `serverfix` branch to the `awesomebranch` branch on the remote Git repository.

The next time one of your collaborators fetches from the remote server, they will get a reference called `origin/serverfix` to where the remote server’s version of `serverfix` is.

	$ git fetch origin
	remote: Counting objects: 20, done.
	remote: Compressing objects: 100% (14/14), done.
	remote: Total 15 (delta 5), reused 0 (delta 0)
	Unpacking objects: 100% (15/15), done.
	From git@github.com:schacon/simplegit
	 * [new branch]      serverfix    -> origin/serverfix

It’s important to note that when you do a fetch that copies new remote branches, you don’t automatically have local, editable copies of the branches. In other words, in this case, you don’t have a new `serverfix` branch — you only have an `origin/serverfix` pointer that you can’t modify.

To merge this work into your current working branch, run `git merge origin/serverfix`. If you want your own `serverfix` branch to work on, base it off your remote branch.

	$ git checkout -b serverfix origin/serverfix
	Branch serverfix set up to track remote branch refs/remotes/origin/serverfix.
	Switched to a new branch "serverfix"

This gives you a local branch to work on that starts where `origin/serverfix` is.

### Tracking Branches ###

Checking out a local branch from a remote branch automatically creates what is called a _tracking branch_. Tracking branches are local branches that have a direct relationship to a remote branch. If you’re on a tracking branch and type `git push`, Git automatically knows which server and branch to push to. Also, running `git pull` while on one of these branches fetches all the remote references and then automatically merges in the corresponding remote branch.

When you clone a repository, Git automatically creates a `master` branch that tracks `origin/master`. That’s why `git push` and `git pull` work out of the box with no other arguments. However, you can set up other tracking branches — ones that don’t track branches on `origin` and don’t track the `master` branch. The simple case is the example you just saw, running `git checkout --track [branch] [remotename]/[branch]`.

	$ git checkout --track origin/serverfix
	Branch serverfix set up to track remote branch refs/remotes/origin/serverfix.
	Switched to a new branch "serverfix"

To set up a local branch with a different name than the remote branch, you can easily use the first version with a different local branch name.

	$ git checkout --track -b sf origin/serverfix
	Branch sf set up to track remote branch refs/remotes/origin/serverfix.
	Switched to a new branch "sf"

Now, your `sf` local branch will automatically push to and pull from `origin/serverfix`.

### Deleting Remote Branches ###

Suppose you’re done with a remote branch — say, you and your collaborators are finished with a feature and have merged it into your remote’s `master` branch. You delete a remote branch using the rather obtuse syntax `git push [remotename] :[branch]`. To delete your `serverfix` branch from your Git repository, run

	$ git push origin :serverfix
	To git@github.com:schacon/simplegit.git
	 - [deleted]         serverfix

Boom. No more `serverfix` branch on the remote server. You may want to dog-ear this page, because you’ll need that command, and you’ll likely forget the syntax. A way to remember this command is by recalling the `git push [remotename] [localbranch]:[remotebranch]` syntax that I went over a bit earlier. If you leave off the `[localbranch]` portion, then you’re basically saying, “Take nothing on my side and make it be `[remotebranch]`”.

## Rebasing ##

There are two main ways to integrate changes from one branch into another: `merging` and `rebasing`. You’ve already learned about merging. In this section you’ll learn what rebasing is, how to do it, why it’s a pretty amazing tool, and in what cases not to do it.

### Basic Rebasing ###

If you go back to an earlier example from the Merge section (see Figure 3-27), you see that you diverged your work and made commits on two different branches.

Insert 18333fig0327.png
Figure 3-27. Your initial diverged commit history.

The easiest way to integrate the branches, as I’ve already covered, is the `git merge` command. It performs a three-way merge between the two latest branch snapshots (C3 and C4) and the most recent common ancestor of the two (C2), creating a new commit (C5), as shown in Figure 3-28.

Insert 18333fig0328.png
Figure 3-28. Merging a branch to integrate the diverged work history.

However, there’s another way: take the change that was introduced in C3 and reapply it on top of C4. In Git, this is called _rebasing_. The `git rebase` command takes all the changes that were committed on one branch and replays them on another.

In this example, run

	$ git checkout experiment
	$ git rebase master
	First, rewinding head to replay your work on top of it...
	Applying: added staged command

This works by going to the common ancestor (C2) of the branch you’re on (`experiment`) and the branch you’re rebasing onto (`master`), getting the diffs introduced by each commit (C3) of the branch you’re on, saving those diffs to temporary files, resetting the current branch (`experiment`) to the same commit as the branch you’re rebasing onto (C4), and finally applying each change in turn to the branch you’re rebasing onto (`master`). Figure 3-29 illustrates this process.

Insert 18333fig0329.png
Figure 3-29. Rebasing the change introduced in C3 onto C4.

At this point, you can go back to the master branch and do a fast-forward merge (see Figure 3-30).

Insert 18333fig0330.png
Figure 3-30. Fast-forwarding the master branch.

The snapshot pointed to by C3' is exactly the same as the one that was pointed to by C5 in the merge example. There’s no difference in the end product of the integration, but rebasing makes for a cleaner history. If you examine the log of a rebased branch, it looks like a linear history: it appears that all the work happened in series, even when it originally happened in parallel.

Often, you’ll do this to make sure your commits apply cleanly on a remote branch — perhaps in a project to which you’re trying to contribute but that you don’t maintain. In this case, you’d do your work in a branch and then rebase your work onto `origin/master` when you were ready to submit your patches to the main project. That way, the maintainer doesn’t have to do any integration work — just a fast-forward or a clean apply.

Note that the snapshot pointed to by the final commit you end up with, whether it’s the last of the rebased commits for a rebase or the final merge commit after a merge, is the same snapshot — it’s only the history that is different. Rebasing replays changes from one line of work onto another in the order they were introduced, whereas merging takes the endpoints and merges them together.

### More Interesting Rebases ###

Your rebase can also replay on something other than the rebase branch. Take a commit history like Figure 3-31, for example. You created a topic branch (`server`) to add some server-side functionality to your project, and made a commit (C3). Then, you branched off that to make the client-side changes (`client`) and committed a few times (C4 and C5). Finally, you went back to your server branch and did a few more commits (C6 and C7). Finally, you made a few commits on `master` (C8 and C9).

Insert 18333fig0331.png
Figure 3-31. A history with a topic branch off another topic branch.

Suppose you decide to merge your client-side changes into your master branch for a release, but you want to hold off merging the server-side changes until they’re tested further. You can take the changes on `client` that aren’t on `server` (C4 and C5) and replay them onto `master` by running `git rebase --onto`.

	$ git rebase --onto master server client

This basically says, “Check out the `client` branch, figure out the patches from the common ancestor of the `client` and `server` branches, and then replay the patches onto `master`.” It’s a bit complex but the result, shown in Figure 3-32, is pretty cool.

Insert 18333fig0332.png
Figure 3-32. Rebasing a topic branch off another topic branch.

Now fast-forward your `master` branch (see Figure 3-33).

	$ git checkout master
	$ git merge client

Insert 18333fig0333.png
Figure 3-33. Fast-forwarding your master branch to include the client branch changes.

Let’s say you decide to do the same thing with `server`. Rebase `server` onto `master` without having to check out `master` first by running `git rebase [basebranch] [topicbranch]` — which checks out the topic branch (in this case, `server`) and replays it onto the base branch (`master`).

	$ git rebase master server

This replays your `server` work on top of your `master` work, as shown in Figure 3-34.

Insert 18333fig0334.png
Figure 3-34. Rebasing your server branch on top of your master branch.

Then, fast-forward the base branch (`master`).

	$ git checkout master
	$ git merge server

At this point, remove the `client` and `server` branches because all the work is contained in `master` so you don’t need them anymore. This leaves your history looking like Figure 3-35.

	$ git branch -d client
	$ git branch -d server

Insert 18333fig0335.png
Figure 3-35. Final commit history.

### The Perils of Rebasing ###

Ahh, but the bliss of rebasing isn’t without its drawbacks, which can be summed up in a single line:

**Do not rebase commits that you have pushed to a public repository.**

If you follow that guideline, you’ll be fine. If you don’t, people will hate you, and you’ll be scorned by friends and family.

When you rebase, you’re abandoning existing commits and creating new ones that are similar but not exactly the same. If you push commits somewhere and other people pull them and base work on them, and then you rewrite those commits with `git rebase` and push them again, your collaborators will have to re-merge their work. Things will get messy when you try to pull their work back into yours.

Let’s look at an example of how making rebasing public can cause problems. Suppose you clone from a central server and then do some work. Your commit history looks like Figure 3-36.

Insert 18333fig0336.png
Figure 3-36. Clone a repository, and base some work on it.

Now, someone else does more work that includes a branch and merge, and pushes that work to the central server. Fetch and merge the new remote branch into your work, making your history look something like Figure 3-37.

Insert 18333fig0337.png
Figure 3-37. Fetch more commits, and merge them into your work.

Next, the person who pushed the merged work decides to go back and rebase their work instead. They do a `git push --force` to overwrite the history on the server. You then fetch from that server, bringing down the new commits.

Insert 18333fig0338.png
Figure 3-38. Someone pushes rebased commits, abandoning commits you’ve based your work on.

At this point, you have to merge this work in again, even though you’ve already done so. Rebasing changes the SHA-1 hashes of these commits so to Git they look like new commits, when in fact you already have the C4 work in your history (see Figure 3-39).

Insert 18333fig0339.png
Figure 3-39. You merge in the same work again into a new merge commit.

You have to merge that work in at some point so you can keep up with the other developers in the future. After you do that, your commit history will contain both the C4 and C4' commits, which have different SHA-1 hashes but introduce the same work and have the same commit message. If you run `git log` when your history looks like this, you’ll see two commits that have the same author, date, and message, which will be confusing. Furthermore, if you push this history back to the server, you’ll reintroduce all those rebased commits to the central server, which can further confuse people.

If you treat rebasing as a way to work with commits before you push them, and if you only rebase commits that have never been available publicly, then you’ll be fine. If you rebase commits that have already been pushed publicly, and people may have based work on those commits, then you may be in for some frustrating trouble.

## Summary ##

I’ve covered basic branching and merging in Git. You should feel comfortable creating and switching to new branches, switching between branches, and merging local branches.  You should also be able to share your branches by pushing them to a shared server, working with others on shared branches, and rebasing your branches before they’re shared.
