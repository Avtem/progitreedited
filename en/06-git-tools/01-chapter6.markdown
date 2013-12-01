# Git Tools #

By now, you’ve learned most of the day-to-day commands and workflows that you need to manage and maintain a Git source code repository. You’ve learned the basic tasks of tracking and committing files, and you’ve harnessed the power of the staging area, lightweight topic branching, and merging.

Now you’ll explore a number of very powerful things that Git can do that you may not necessarily use on a day-to-day basis but that you may need one day.

## Commit Selection ##

Git allows you to specify specific commits or a range of commits in several ways.

### Single Commit ###

You can obviously refer to a commit by its SHA-1 hash, but there are also more human-friendly ways. This section outlines the various ways you can refer to a single commit.

### Short SHA-1 Hash ###

Git is smart enough to figure out what commit you’re referring to if you only provide the first few characters of its SHA-1 hash, as long as the partial hash is at least four characters long and unambiguous — that is, only one object in the current repository begins with that partial SHA-1 hash.

For example, to see a specific commit, suppose you run `git log` to identify the commit where you made a certain change.

	$ git log
	commit 734713bc047d87bf7eac9674765ae793478c50d3
	Author: Scott Chacon <schacon@gmail.com>
	Date:   Fri Jan 2 18:32:33 2009 -0800

	    fixed refs handling, added gc auto, updated tests

	commit d921970aadf03b3cf0e71becdaab3147ba71cdef
	Merge: 1c002dd... 35cfb2b...
	Author: Scott Chacon <schacon@gmail.com>
	Date:   Thu Dec 11 15:08:43 2008 -0800

	    Merge commit 'phedders/rdocs'

	commit 1c002dd4b536e7479fe34593e72e6c6c1819e53b
	Author: Scott Chacon <schacon@gmail.com>
	Date:   Thu Dec 11 14:58:32 2008 -0800

	    added some blame and merge stuff

In this case, let’s say `1c002dd....` is the commit you’re interested in. For `git show` to show that commit, the following commands are equivalent (assuming the shorter versions are unambiguous):

	$ git show 1c002dd4b536e7479fe34593e72e6c6c1819e53b
	$ git show 1c002dd4b536e7479f
	$ git show 1c002d

Git can recognize a short, unique representation for your SHA-1 hashes. If you pass `--abbrev-commit` to `git log`, the output will contain short unique hash values. This option defaults to showing seven character hash values but makes the hashes longer if necessary to keep them unambiguous.

	$ git log --abbrev-commit --pretty=oneline
	ca82a6d changed the version number
	085bb3b removed unnecessary test code
	a11bef0 first commit

Generally, eight to ten characters are more than enough for a SHA-1 hash to be unique within a project. One of the largest Git projects, the Linux kernel, is beginning to need 12 characters out of the possible 40 for SHA-1 hashes to stay unique.

### A SHORT NOTE ABOUT SHA-1 HASHES ###

You might become concerned at some point that you will, by random happenstance, attempt to place two different objects into your repository that hash to the same SHA-1 hash. What then?

If you do happen to commit an object that hashes to the same SHA-1 hash as an object already in your repository, Git will see that the new object is already in your Git repository and assume there’s no need to write it again. If you try to check out that object again, you’ll always get the contents of the first object.

However, you should be aware of how ridiculously unlikely this scenario is. A SHA-1 hash is 20 bytes or 160 bits. The number of randomly hashed objects needed to ensure a 50% probability of a single collision is about 2^80 (the formula for determining collision probability is `p = (n(n-1)/2) * (1/2^160))`. 2^80 is 1.2 x 10^24 or 1 million billion billion. That’s 1,200 times the number of grains of sand on earth.

Here’s an example to give you an idea of what it would take to get a SHA-1 hash collision. If all 6.5 billion humans on Earth were programming, and every second each one was producing code that was the equivalent of the entire Linux kernel history (1 million Git objects) and pushing it into one enormous Git repository, it would take 5 years until that repository contained enough objects to have a 50% probability of a single SHA-1 hash object collision. A higher probability exists that every member of your programming team will be attacked and killed by wolves in unrelated incidents on the same night.

### Branch References ###

The most straightforward way to specify a commit requires that the commit have a branch reference pointed at it. Then, you can use a branch name in any Git command that expects a commit object or SHA-1 hash. For instance, to show the last commit object on a branch, the following commands are equivalent, assuming that the `topic1` branch points to `ca82a6d`:

	$ git show ca82a6dff817ec66f44342007202690a93763949
	$ git show topic1

To see which specific SHA-1 hash a branch points to, use a Git plumbing tool called `git rev-parse`. See Chapter 9 for more information about plumbing tools. Basically, `git rev-parse` exists for lower-level operations and isn’t designed to be used in day-to-day operations. However, it can be helpful sometimes when you need to see what’s really going on. Here, run `git rev-parse` on your branch.

	$ git rev-parse topic1
	ca82a6dff817ec66f44342007202690a93763949

### RefLog Shortnames ###

Git keeps a reflog — a log of where your HEAD and branch references have been. You can see your reflog by running `git reflog`.

	$ git reflog
	734713b... HEAD@{0}: commit: fixed refs handling, added gc auto, updated
	d921970... HEAD@{1}: merge phedders/rdocs: Merge made by recursive.
	1c002dd... HEAD@{2}: commit: added some blame and merge stuff
	1c36188... HEAD@{3}: rebase -i (squash): updating HEAD
	95df984... HEAD@{4}: commit: # This is a combination of two commits.
	1c36188... HEAD@{5}: rebase -i (squash): updating HEAD
	7e05da5... HEAD@{6}: rebase -i (pick): updating HEAD

Every time your branch tip is updated for any reason, Git stores the new value in this temporary log. You can also specify older commits as well. If you want to see the fifth prior value of the HEAD of your repository, use the `@{n}` syntax that you see in the reflog output.

	$ git show HEAD@{5}

You can also use this syntax to see where a branch was some specific amount of time ago. For instance, to see where your `master` branch was yesterday, run

	$ git show master@{yesterday}

That shows you where the branch tip was yesterday. This technique only works for data that’s still in your reflog, so you can’t use it to look for commits older than a few months.

To see reflog information formatted like `git log` output, run `git log -g`.

	$ git log -g master
	commit 734713bc047d87bf7eac9674765ae793478c50d3
	Reflog: master@{0} (Scott Chacon <schacon@gmail.com>)
	Reflog message: commit: fixed refs handling, added gc auto, updated
	Author: Scott Chacon <schacon@gmail.com>
	Date:   Fri Jan 2 18:32:33 2009 -0800

	    fixed refs handling, added gc auto, updated tests

	commit d921970aadf03b3cf0e71becdaab3147ba71cdef
	Reflog: master@{1} (Scott Chacon <schacon@gmail.com>)
	Reflog message: merge phedders/rdocs: Merge made by recursive.
	Author: Scott Chacon <schacon@gmail.com>
	Date:   Thu Dec 11 15:08:43 2008 -0800

	    Merge commit 'phedders/rdocs'

It’s important to note that the reflog information is strictly local — it’s a log of what you’ve done in your repository. The references won’t be the same in someone else’s copy of the repository. Right after you initially clone a repository, you’ll have an empty reflog, since no activity has occurred yet in your repository. Running `git show HEAD@{2.months.ago}` will work only if you cloned the project at least two months ago. If you cloned it five minutes ago, you’ll see no results.

### Ancestry References ###

The other main way to specify a commit is via its ancestry. If you place a `^` at the end of a reference, Git resolves it to mean the parent of that commit.
Suppose you look at the history of your project.

	$ git log --pretty=format:'%h %s' --graph
	* 734713b fixed refs handling, added gc auto, updated tests
	*   d921970 Merge commit 'phedders/rdocs'
	|\
	| * 35cfb2b Some rdoc changes
	* | 1c002dd added some blame and merge stuff
	|/
	* 1c36188 ignore *.gem
	* 9b29157 add open3_detach to gemspec file list

Then you can see the previous commit by specifying `HEAD^`, which means "the parent of HEAD".

	$ git show HEAD^
	commit d921970aadf03b3cf0e71becdaab3147ba71cdef
	Merge: 1c002dd... 35cfb2b...
	Author: Scott Chacon <schacon@gmail.com>
	Date:   Thu Dec 11 15:08:43 2008 -0800

	    Merge commit 'phedders/rdocs'

You can also specify a number after the `^`. For example, `d921970^2` means "the second parent of d921970." This syntax is only useful for merge commits, which have more than one parent. The first parent is the branch you were on when you merged, and the second is the commit on the branch that you merged in.

	$ git show d921970^
	commit 1c002dd4b536e7479fe34593e72e6c6c1819e53b
	Author: Scott Chacon <schacon@gmail.com>
	Date:   Thu Dec 11 14:58:32 2008 -0800

	    added some blame and merge stuff

	$ git show d921970^2
	commit 35cfb2b795a55793d7cc56a6cc2060b4bb732548
	Author: Paul Hedderly <paul+git@mjr.org>
	Date:   Wed Dec 10 22:22:03 2008 +0000

	    Some rdoc changes

The other main ancestry specification is the `~`. This also refers to the first parent, so `HEAD~` and `HEAD^` are equivalent. The difference becomes apparent when you specify a number. `HEAD~2` means "the first parent of the first parent," or "the grandparent" — it traverses the first parents the number of times you specify. For example, in the history listed earlier, `HEAD~3` would be

	$ git show HEAD~3
	commit 1c3618887afb5fbcbea25b7c013f4e2114448b8d
	Author: Tom Preston-Werner <tom@mojombo.com>
	Date:   Fri Nov 7 13:47:59 2008 -0500

	    ignore *.gem

This can also be written `HEAD^^^`, which again is the first parent of the first parent of the first parent.

	$ git show HEAD^^^
	commit 1c3618887afb5fbcbea25b7c013f4e2114448b8d
	Author: Tom Preston-Werner <tom@mojombo.com>
	Date:   Fri Nov 7 13:47:59 2008 -0500

	    ignore *.gem

You can also combine these forms — get the second parent of the previous reference (assuming it was a merge commit) by using `HEAD~3^2`, and so on.

### Commit Ranges ###

Now that you can specify individual commits, let’s see how to specify ranges of commits. This is particularly useful for managing your branches — if you have a lot of branches, use range specifications to answer questions such as, "What work is on this branch that I haven’t yet merged into my main branch?"

#### Double Dot ####

The most common range specification is the double-dot syntax. This specifies a range of commits that are reachable from one commit but aren’t reachable from another. For example, say you have a commit history that looks like Figure 6-1.

Insert 18333fig0601.png
Figure 6-1. Example history for range selection.

You want to see what’s in your `experiment` branch that hasn’t yet been merged into your `master` branch. Ask Git to show a list of just those commits with `master..experiment`. That means "all commits reachable by `experiment` that aren’t reachable by `master`." For the sake of brevity and clarity in these examples, I’ll use the letters of the commit objects from the diagram in place of the actual log output.

	$ git log master..experiment
	D
	C

If, on the other hand, you want to see the opposite — all commits in `master` that aren’t in `experiment` — reverse the branch names. `experiment..master` shows you everything in `master` not reachable from `experiment`.

	$ git log experiment..master
	F
	E

This is useful for keeping the `experiment` branch up to date and for previewing what you’re about to merge in. Another very common use of this syntax is to see what you’re about to push to a remote.

	$ git log origin/master..HEAD

This command shows any commits in your current branch that aren’t in the `master` branch on your `origin` remote. If you run `git push` and your current branch is tracking `origin/master`, the commits listed by `git log origin/master..HEAD` are those that will be transferred to the server.
If you leave off one side of the specification, Git assumes the missing side is HEAD. For example, you get the same results as in the previous example by running `git log origin/master..`.

#### Multiple Points ####

The double-dot syntax is useful as a shorthand but perhaps you want to specify more than two branches to indicate your revision, such as seeing what commits are in any of several branches that aren’t in your current branch. Git does this by using either the `^` character or `--not` before any reference from which you don’t want to see reachable commits. Thus, the following three commands are equivalent:

	$ git log refA..refB
	$ git log ^refA refB
	$ git log refB --not refA

This is nice because with this syntax you can specify more than two references in your query, which you can’t do with the double-dot syntax. For instance, to see all commits reachable from `refA` or `refB`, but not from `refC`, run one of

	$ git log refA refB ^refC
	$ git log refA refB --not refC

This makes for a very powerful revision query system that should help you figure out what’s in your branches.

#### Triple Dot ####

The last major range-selection form is the triple-dot syntax, which specifies all the commits that are reachable by either of two references, but not by both. Look back at the example commit history in Figure 6-1.
To see what’s in `master` or `experiment` but not any common references, run

	$ git log master...experiment
	F
	E
	D
	C

Again, this produces normal `git log` output but shows you only the commit information for those four commits, appearing in the traditional commit date ordering.

A common option to use with `git log` in this case is `--left-right`, which shows which side of the range each commit is in. This helps make the data more useful.

	$ git log --left-right master...experiment
	< F
	< E
	> D
	> C

With these tools, you can much more easily tell Git what commit or commits to inspect.

## Interactive Staging ##

Git comes with a couple of scripts that make some command-line tasks easier. Here, you’ll see a few interactive commands that can help you easily craft commits to include only certain combinations and parts of files. These tools are very helpful if you modify a bunch of files and then decide that you want those changes to be in several focused commits rather than one big messy commit. This way, you can make sure your commits are logically separate and can be easily reviewed by the developers working with you.
If you run `git add` with the `-i` or `--interactive` option, Git goes into an interactive mode, displaying something like

	$ git add -i
	           staged     unstaged path
	  1:    unchanged        +0/-1 TODO
	  2:    unchanged        +1/-1 index.html
	  3:    unchanged        +5/-1 lib/simplegit.rb

	*** Commands ***
	  1: status     2: update      3: revert     4: add untracked
	  5: patch      6: diff        7: quit       8: help
	What now>

You can see that this command shows a much different view of your staging area — basically the same information you get from `git status` but a bit more succinct and informative. It lists the changes you’ve staged on the left and the unstaged changes on the right.

After this comes a Commands section. Here you can do a number of things, including staging files, unstaging files, staging parts of files, adding untracked files, and seeing diffs of what has been staged.

### Staging and Unstaging Files ###

If you type `2` or `u` at the `What now>` prompt, you’re prompted for which files to stage.

	What now> 2
	           staged     unstaged path
	  1:    unchanged        +0/-1 TODO
	  2:    unchanged        +1/-1 index.html
	  3:    unchanged        +5/-1 lib/simplegit.rb
	Update>>

To stage the TODO and index.html files, type their numbers.

	Update>> 1,2
	           staged     unstaged path
	* 1:    unchanged        +0/-1 TODO
	* 2:    unchanged        +1/-1 index.html
	  3:    unchanged        +5/-1 lib/simplegit.rb
	Update>>

The `*` next to each file name means the file is selected to be staged. If you just type Enter at the `Update>>` prompt, Git takes what you selected and stages it for you.

	Update>>
	updated 2 paths

	*** Commands ***
	  1: status     2: update      3: revert     4: add untracked
	  5: patch      6: diff        7: quit       8: help
	What now> 1
	           staged     unstaged path
	  1:        +0/-1      nothing TODO
	  2:        +1/-1      nothing index.html
	  3:    unchanged        +5/-1 lib/simplegit.rb

Now you see that TODO and index.html are staged and simplegit.rb is still unstaged. To unstage TODO at this point, enter the `3` or `r` (for revert) option.

	*** Commands ***
	  1: status     2: update      3: revert     4: add untracked
	  5: patch      6: diff        7: quit       8: help
	What now> 3
	           staged     unstaged path
	  1:        +0/-1      nothing TODO
	  2:        +1/-1      nothing index.html
	  3:    unchanged        +5/-1 lib/simplegit.rb
	Revert>> 1
	           staged     unstaged path
	* 1:        +0/-1      nothing TODO
	  2:        +1/-1      nothing index.html
	  3:    unchanged        +5/-1 lib/simplegit.rb
	Revert>> [enter]
	reverted one path

Looking at your Git status again, you see that you’ve unstaged TODO.

	*** Commands ***
	  1: status     2: update      3: revert     4: add untracked
	  5: patch      6: diff        7: quit       8: help
	What now> 1
	           staged     unstaged path
	  1:    unchanged        +0/-1 TODO
	  2:        +1/-1      nothing index.html
	  3:    unchanged        +5/-1 lib/simplegit.rb

To see the diff of what you’ve staged, use the `6` or `d` (for diff) command. It shows a list of staged files, and you can select the files for which you would like to see the diffs between what you’ve staged and what’s in the most recent commit. This is much like running `git diff --cached` on the command line.

	*** Commands ***
	  1: status     2: update      3: revert     4: add untracked
	  5: patch      6: diff        7: quit       8: help
	What now> 6
	           staged     unstaged path
	  1:        +1/-1      nothing index.html
	Review diff>> 1
	diff --git a/index.html b/index.html
	index 4d07108..4335f49 100644
	--- a/index.html
	+++ b/index.html
	@@ -16,7 +16,7 @@ Date Finder

	 <p id="out">...</p>

	-<div id="footer">contact : support@github.com</div>
	+<div id="footer">contact : email.support@github.com</div>

	 <script type="text/javascript">

With these basic commands, you can use the interactive add mode to manage your staging area a little more easily.

### Staging Patches ###

It’s also possible for Git to stage certain parts of files and not others. For example, if you make two changes to simplegit.rb and want to stage one of them and not the other, doing so is very easy in Git. From the interactive prompt, type `5` or `p` (for patch). Git will ask which files to partially stage. Then, for each section of the selected files, it displays hunks of the file diff and asks if you would like to stage them, one by one.

	diff --git a/lib/simplegit.rb b/lib/simplegit.rb
	index dd5ecc4..57399e0 100644
	--- a/lib/simplegit.rb
	+++ b/lib/simplegit.rb
	@@ -22,7 +22,7 @@ class SimpleGit
	   end

	   def log(treeish = 'master')
	-    command("git log -n 25 #{treeish}")
	+    command("git log -n 30 #{treeish}")
	   end

	   def blame(path)
	Stage this hunk [y,n,a,d,/,j,J,g,e,?]?

You have a lot of options at this point. Typing `?` shows a list of what you can do.

	Stage this hunk [y,n,a,d,/,j,J,g,e,?]? ?
	y - stage this hunk
	n - do not stage this hunk
	a - stage this and all the remaining hunks in the file
	d - do not stage this hunk nor any of the remaining hunks in the file
	g - select a hunk to go to
	/ - search for a hunk matching the given regex
	j - leave this hunk undecided, see next undecided hunk
	J - leave this hunk undecided, see next hunk
	k - leave this hunk undecided, see previous undecided hunk
	K - leave this hunk undecided, see previous hunk
	s - split the current hunk into smaller hunks
	e - manually edit the current hunk
	? - print help

Generally, you’ll type `y` or `n` to stage each hunk, but staging all of them in certain files or postponing a hunk decision until later can be helpful too. If you stage one part of a file and leave another part unstaged, your status output looks like

	What now> 1
	           staged     unstaged path
	  1:    unchanged        +0/-1 TODO
	  2:        +1/-1      nothing index.html
	  3:        +1/-1        +4/-0 lib/simplegit.rb

The status of simplegit.rb is interesting. It shows that a couple of lines are staged and a couple are unstaged. You’ve partially staged this file. At this point, you can exit the script and run `git commit` to commit the partially staged files.

You don’t need to be in interactive add mode to do partial-file staging. You can start the same script by running `git add -p` or `git add --patch` on the command line.

## Stashing ##

Often, when you’ve been working on part of a project, your working directory gets into a messy state, and you want to switch branches for a bit to work on something else. The problem is, you don’t want to commit half-done work just so you can return to this state later. The answer to this dilemma is the `git stash` command.

Stashing takes the dirty state of your working directory — that is, your modified tracked files and staged changes — and saves it on a stack of unfinished changes to which you can revert at any time.

### Stashing Your Work ###

To demonstrate, start working on a couple of files in a project and stage one of them. Run `git status` to see your state.

	$ git status
	# On branch master
	# Changes to be committed:
	#   (use "git reset HEAD <file>..." to unstage)
	#
	#      modified:   index.html
	#
	# Changes not staged for commit:
	#   (use "git add <file>..." to update what will be committed)
	#
	#      modified:   lib/simplegit.rb
	#

Now, you want to switch to a different branch, but you don’t want to commit what you’ve been working on yet. Instead, stash the changes. To push a new stash onto your stack, run `git stash`.

	$ git stash
	Saved working directory and index state \
	  "WIP on master: 049d078 added the index file"
	HEAD is now at 049d078 added the index file
	(To restore them type "git stash apply")

Your working directory is now clean.

	$ git status
	# On branch master
	nothing to commit (working directory clean)

At this point, you can easily start working on a different branch. Your changes are stored on your stack. To see the stashes you’ve stored, run `git stash list`.

	$ git stash list
	stash@{0}: WIP on master: 049d078 added the index file
	stash@{1}: WIP on master: c264051... Revert "added file_size"
	stash@{2}: WIP on master: 21d80a5... added number to log

In this case, you already did two stashes previously, so are now three different stashed states. Revert to the one you just stashed by using the command shown in the help output of the original stash command, `git stash apply`. To revert to one of the older stashes, name it like this: `git stash apply stash@{2}`. If you don’t specify a stash, Git tries to revert to the most recent stash.

	$ git stash apply
	# On branch master
	# Changes not staged for commit:
	#   (use "git add <file>..." to update what will be committed)
	#
	#      modified:   index.html
	#      modified:   lib/simplegit.rb
	#

You can see that Git restored the uncommitted files from when you saved the stash. In this case, you had a clean working directory when you tried to revert from the stash, and you tried to revert on the same branch you saved from. But having a clean working directory and reverting it on the same branch aren’t necessary to successfully revert a stash. You can save a stash on one branch, switch to another branch later, and try to reapply the changes. You can also have modified and uncommitted files in your working directory when you revert a stash — Git shows merge conflicts if anything no longer applies cleanly.

The changes to your files were reapplied, but any files you staged before weren’t restaged. To do that, you must run `git stash apply` with the `--index` option to try to reapply the staged changes.

	$ git stash apply --index
	# On branch master
	# Changes to be committed:
	#   (use "git reset HEAD <file>..." to unstage)
	#
	#      modified:   index.html
	#
	# Changes not staged for commit:
	#   (use "git add <file>..." to update what will be committed)
	#
	#      modified:   lib/simplegit.rb
	#

`git stash apply` only tries to revert the stashed work — the stash remains on your stack. To remove a stash, run `git stash drop` with the name of the stash to remove.

	$ git stash list
	stash@{0}: WIP on master: 049d078 added the index file
	stash@{1}: WIP on master: c264051... Revert "added file_size"
	stash@{2}: WIP on master: 21d80a5... added number to log
	$ git stash drop stash@{0}
	Dropped stash@{0} (364e91f3f268f0900bc3ee613f9f733e82aaed43)

You can also run `git stash pop` to revert the stash and then immediately remove it from your stack.

### Un-applying a Stash ###

In some cases you might want to revert stashed changes, do some work, but then stash those changes that originally came from the stash. Git does not include a `stash unapply` command, but it’s possible to achieve the same effect by simply retrieving the patch associated with a stash and applying it in reverse.

    $ git stash show -p stash@{0} | git apply -R

Again, if you don’t specify a stash, Git assumes the most recent stash.

    $ git stash show -p | git apply -R

You may want to create an alias to effectively add a `stash-unapply` option to `git`. For example

    $ git config --global alias.stash-unapply '!git stash show -p | git apply -R'
    $ git stash
    $ #... work work work
    $ git stash-unapply

### Creating a Branch from a Stash ###

If you stash some work, leave it for a while, and continue on the branch from which you stashed the work, you may have a problem reverting the work. If reverting tries to modify a file that you’ve since modified, you’ll get a merge conflict that you’ll have to try to resolve. If you want an easier way to test the stashed changes again, run `git stash branch`, which creates a new branch, checks out the commit you were on when you stashed your work, reverts your work there, and then drops the stash if it applies successfully.

	$ git stash branch testchanges
	Switched to a new branch "testchanges"
	# On branch testchanges
	# Changes to be committed:
	#   (use "git reset HEAD <file>..." to unstage)
	#
	#      modified:   index.html
	#
	# Changes not staged for commit:
	#   (use "git add <file>..." to update what will be committed)
	#
	#      modified:   lib/simplegit.rb
	#
	Dropped refs/stash@{0} (f0dfc4d5dc332d1cee34a634182e168c4efc3359)

This is a nice shortcut to easily revert stashed work to work on in a new branch.

## Rewriting History ##

Many times, when working with Git, you want to revise your commit history. One of the great things about Git is that it allows you to make decisions at the last possible moment. You can decide what files go into which commits right before you commit by only putting certain files in the staging area, you can decide that you don’t mean to be working on something yet with the stash command, and you can rewrite previous commits so they look like they happened differently. This can involve changing the order of the commits, changing messages or modifying the files in a commit, squashing together or splitting apart commits, or removing commits entirely — all before you share your work with others.

In this section, I’ll cover how to accomplish these very useful tasks so that you can make your commit history look the way you want before you share your repository.

### Changing the Last Commit ###

Changing your last commit is probably the most common rewriting of history that you’ll do. You’ll often want to change two basic things in your last commit — the commit message, or the snapshot you just recorded by adding, changing, and removing files.

If you only want to modify your last commit message, it’s very simple.

	$ git commit --amend

That drops you into your text editor, which will have your last commit message in it, ready to be modified. When you save your message and exit the editor, it writes a new commit containing the new message and makes the commit your new last commit.

If you’ve committed and then want to change the snapshot you committed by adding or changing files, possibly because you forgot to add a newly created file when you originally committed, the process works basically the same way. Stage the changes you want by running `git add` on a modified file or `git rm` on a tracked file, and the subsequent `git commit --amend` takes your current staging area and makes it the snapshot for the new commit.

Be careful with this technique because `git commit --amend` changes the SHA-1 hash of the commit. It’s like a very small rebase — don’t amend your last commit if you’ve already pushed.

### Changing Multiple Commit Messages ###

To modify a commit that’s farther back in your history, you must use more complex tools. Git doesn’t have a modify-history tool, but you can rebase to move a series of commits onto the branch they were originally based on instead of moving them to another branch. With the interactive rebase tool, you can then stop after each commit you want to modify and change the message, add files, or do whatever you wish. You run rebase interactively by running `git rebase -i`. You must indicate how far back you want to rewrite commits by telling the command which commit to rebase onto.

For example, to change any of the last three commit messages, supply the parent of the last commit to edit, which is `HEAD~2^` or `HEAD~3`, as an argument to `git rebase -i`. It may be easier to remember the `~3` because you’re trying to edit the last three commits. But keep in mind that you’re actually referring to a commit four commits back — the parent of the last commit you want to edit.

	$ git rebase -i HEAD~3

Remember again that this is a rebasing command. Every commit included in the range `HEAD~3..HEAD` will be rewritten whether or not you change the message. Don’t include any commit you’ve already pushed to a central server. Doing so will confuse other developers by providing an alternate version of the same change.

Running this command opens your text editor with a list of commits in the buffer that look something like

	pick f7f3f6d changed my name a bit
	pick 310154e updated README formatting and added blame
	pick a5f4a0d added cat-file

	# Rebase 710f0f8..a5f4a0d onto 710f0f8
	#
	# Commands:
	#  p, pick = use commit
	#  e, edit = use commit, but stop for amending
	#  s, squash = use commit, but meld into previous commit
	#
	# If you remove a line here THAT COMMIT WILL BE LOST.
	# However, if you remove everything, the rebase will be aborted.
	#

It’s important to note that these commits are listed in the opposite order from what you normally see using the `git log` command. If you run `git log`, you see something like

	$ git log --pretty=format:"%h %s" HEAD~3..HEAD
	a5f4a0d added cat-file
	310154e updated README formatting and added blame
	f7f3f6d changed my name a bit

Again, notice the reverse order. The interactive rebase creates a script that it’s going to run. It starts at the commit you specify on the command line (`HEAD~3`) and replays the changes introduced in each of these commits, from top to bottom. It lists the oldest at the top, rather than the newest, because that’s the first one it will replay.

You need to edit the script so that it stops at the commit you want to edit. To make this happen, change the word `pick` to the word `edit` in each of the commits you want the script to stop after. For example, to modify only the third commit message, change the file to look like

	edit f7f3f6d changed my name a bit
	pick 310154e updated README formatting and added blame
	pick a5f4a0d added cat-file

When you save the file and exit the editor, Git rewinds back to the last commit in that list and shows the following message:

	$ git rebase -i HEAD~3
	Stopped at 7482e0d... updated the gemspec to hopefully work better
	You can amend the commit now, with

	       git commit --amend

	Once you’re satisfied with your changes, run

	       git rebase --continue

These instructions tell you exactly what to do. Type

	$ git commit --amend

Change the commit message, and exit the editor. Then, run

	$ git rebase --continue

This command applies the other two commits automatically, and then you’re done. If you change `pick` to `edit` on more lines, you can repeat these steps for each commit you want to change. Each time Git will stop, let you amend the commit, and then continue when you’re finished.

### Reordering Commits ###

You can also use interactive rebases to entirely reorder or remove commits. To remove the "added cat-file" commit and change the order in which the other two commits appear, change the rebase script from this

	pick f7f3f6d changed my name a bit
	pick 310154e updated README formatting and added blame
	pick a5f4a0d added cat-file

to this.

	pick 310154e updated README formatting and added blame
	pick f7f3f6d changed my name a bit

When you save the file and exit the editor, Git rewinds your branch to the parent of these commits, applies `310154e` and then `f7f3f6d`, and then stops. You effectively change the order of those commits and remove the "added cat-file" commit completely.

### Squashing Commits ###

It’s also possible to take a series of commits and squash them down into a single commit with the interactive rebasing tool. The script puts helpful instructions in the rebase message.

	#
	# Commands:
	#  p, pick = use commit
	#  e, edit = use commit, but stop for amending
	#  s, squash = use commit, but meld into previous commit
	#
	# If you remove a line here THAT COMMIT WILL BE LOST.
	# However, if you remove everything, the rebase will be aborted.
	#

If, instead of `pick` or `edit`, you specify `squash`, Git applies both that change and the change directly before it, and makes you merge the commit messages together. So, to make a single commit from these three commits, make the script look like

	pick f7f3f6d changed my name a bit
	squash 310154e updated README formatting and added blame
	squash a5f4a0d added cat-file

When you save and exit the editor, Git applies all three changes and then puts you back into the editor to merge the three commit messages.

	# This is a combination of 3 commits.
	# The first commit's message is:
	changed my name a bit

	# This is the 2nd commit message:

	updated README formatting and added blame

	# This is the 3rd commit message:

	added cat-file

When you save that file, you have a single commit that includes the changes from all three previous commits.

### Splitting a Commit ###

Splitting a commit undoes a commit and then partially stages and commits as many times as the number of commits you want to end up with. For example, suppose you want to split the middle commit of your three commits. Instead of "updated README formatting and added blame", you want to split it into two commits — "updated README formatting" for the first commit, and "added blame" for the second. You do that in the `git rebase -i` script by changing the command on the commit you want to split to `edit`.

	pick f7f3f6d changed my name a bit
	edit 310154e updated README formatting and added blame
	pick a5f4a0d added cat-file

When you save the file and exit the editor, Git rewinds to the parent of the first commit in your list, applies the first commit (`f7f3f6d`), applies the second (`310154e`), and exits. Then, do a mixed reset of that commit by running `git reset HEAD^`, which effectively undoes that commit and leaves the modified files unstaged. Now you can take the changes that have been reset, and create multiple commits out of them. Simply stage and commit files until you have several commits, and run `git rebase --continue` when you’re done.

	$ git reset HEAD^
	$ git add README
	$ git commit -m 'updated README formatting'
	$ git add lib/simplegit.rb
	$ git commit -m 'added blame'
	$ git rebase --continue

Git applies the last commit (`a5f4a0d`) in the script, making your history look like

	$ git log -4 --pretty=format:"%h %s"
	1c002dd added cat-file
	9b29157 added blame
	35cfb2b updated README formatting
	f3cc40e changed my name a bit

Once again, this changes the SHA-1 hashes of all the commits in your list, so make sure no commit shows up in that list that you’ve already pushed to a shared repository.

### The Nuclear Option: filter-branch ###

There’s another history-rewriting option to rewrite a larger number of commits in some scriptable way — for instance, changing your e-mail address globally or removing a file from every commit. The command is `git filter-branch`. It can rewrite huge swaths of history, so you probably shouldn’t use it unless your project isn’t yet public and other people haven’t based work off the commits you’re about to rewrite. However, it can be very useful. You’ll learn a few of the common uses now so you can get an idea of some of the things it’s capable of.

#### Removing a File from Every Commit ####

This occurs fairly commonly. Someone accidentally commits a huge binary file after a thoughtless `git add .`, and you want to remove it in every commit. Perhaps you accidentally committed a file that contained a password, and you want to make your project open source. `git filter-branch` is the tool you probably want to use to scrub your entire history. To remove a file named `passwords.txt` from your entire commit history, run `git filter-branch --tree-filter`.

	$ git filter-branch --tree-filter 'rm -f passwords.txt' HEAD
	Rewrite 6b9b3cf04e7c5686a9cb838c3f36a8cb6a0fc2bd (21/21)
	Ref 'refs/heads/master' was rewritten

The `--tree-filter` option runs the specified command after each checkout of the project and then recommits the results. In this case, you remove `passwords.txt` from every snapshot, whether or not it exists. To remove all accidentally committed editor backup files, run something like

	$ git filter-branch --tree-filter "rm -f *~" HEAD

You’ll be able to watch Git rewriting trees and commits, and then move the branch pointer at the end. It’s generally a good idea to do this in a testing branch and then hard-reset your master branch after you’ve determined the outcome is what you really want. To run `git filter-branch` on all your branches, add `--all` to the command.

#### Making a Subdirectory the New Root ####

Suppose you’ve done an import from another source control system and have subdirectories that make no sense (`trunk`, `tags`, and so on). To make the `trunk` subdirectory be the new project root for every commit, `git filter-branch` can help you do that, too.

	$ git filter-branch --subdirectory-filter trunk HEAD
	Rewrite 856f0bf61e41a27326cdae8f09fe708d679f596f (12/12)
	Ref 'refs/heads/master' was rewritten

Now your new project root is what was in the `trunk` subdirectory. Git will also automatically remove commits that didn’t affect the subdirectory.

#### Changing E-Mail Addresses Globally ####

Another common case is that you forgot to run `git config` to set your name and e-mail address before you started working, or perhaps you want to open-source a project at work and change all your work e-mail addresses to your personal address. In any case, you can change e-mail addresses in multiple commits in a batch with `git filter-branch` as well. Be careful to change only the e-mail addresses that are yours, so use `--commit-filter`.

	$ git filter-branch --commit-filter '
	        if [ "$GIT_AUTHOR_EMAIL" = "schacon@localhost" ];
	        then
	                GIT_AUTHOR_NAME="Scott Chacon";
	                GIT_AUTHOR_EMAIL="schacon@example.com";
	                git commit-tree "$@";
	        else
	                git commit-tree "$@";
	        fi' HEAD

This goes through and rewrites every commit to have your new address. Because commits contain the SHA-1 hashes of their parents, this command changes every commit SHA-1 hash in your history, not just those having the matching e-mail address.

## Debugging with Git ##

Git also provides a couple of tools to help you debug issues in your projects. Because Git is designed to work with nearly any type of project, these tools are pretty generic, but they can often help hunt for a bug when things go wrong.

### File Annotation ###

If you track down a bug in your code and want to know when it was introduced and why, file annotation is often your best tool. It shows what commit was the last to modify each line in any file. So, if you see that a method in your code is buggy, annotate the file by running `git blame` to see when each line of the method was last edited, and by whom. This example uses the `-L` option to limit the output to lines 12 through 22.

	$ git blame -L 12,22 simplegit.rb
	^4832fe2 (Scott Chacon  2008-03-15 10:31:28 -0700 12)  def show(tree = 'master')
	^4832fe2 (Scott Chacon  2008-03-15 10:31:28 -0700 13)   command("git show #{tree}")
	^4832fe2 (Scott Chacon  2008-03-15 10:31:28 -0700 14)  end
	^4832fe2 (Scott Chacon  2008-03-15 10:31:28 -0700 15)
	9f6560e4 (Scott Chacon  2008-03-17 21:52:20 -0700 16)  def log(tree = 'master')
	79eaf55d (Scott Chacon  2008-04-06 10:15:08 -0700 17)   command("git log #{tree}")
	9f6560e4 (Scott Chacon  2008-03-17 21:52:20 -0700 18)  end
	9f6560e4 (Scott Chacon  2008-03-17 21:52:20 -0700 19)
	42cf2861 (Magnus Chacon 2008-04-13 10:45:01 -0700 20)  def blame(path)
	42cf2861 (Magnus Chacon 2008-04-13 10:45:01 -0700 21)   command("git blame #{path}")
	42cf2861 (Magnus Chacon 2008-04-13 10:45:01 -0700 22)  end

Notice that the first field in the output is the partial SHA-1 hash of the commit that last modified that line. The next two fields are values extracted from that commit—the author name and the date of that commit — so you can easily see who modified that line and when. After that come the line number and the contents of the file. Also note the `^4832fe2` commit lines, which show that those lines were in this file’s original commit. That commit is when this file was first added to this project, and those lines have been unchanged ever since. This is a tad confusing, because now you’ve seen at least three different ways that Git uses `^` to modify a commit SHA-1 hash.

Another cool thing about Git is that it doesn’t track file renames explicitly. It records the snapshots and then tries to figure out what was renamed implicitly, after the fact. One of the interesting results of this is that you can ask Git to figure out all sorts of code movement as well. If you pass `-C` to `git blame`, Git analyzes the file you’re annotating and tries to figure out where snippets of code within it originally came from if they were copied from elsewhere. Recently, I was refactoring a file named `GITServerHandler.m` into multiple files, one of which was `GITPackUpload.m`. By running `git blame` with the `-C` option, I could see where sections of the code originally came from.

	$ git blame -C -L 141,153 GITPackUpload.m
	f344f58d GITServerHandler.m (Scott 2009-01-04 141)
	f344f58d GITServerHandler.m (Scott 2009-01-04 142) - (void) gatherObjectShasFromC
	f344f58d GITServerHandler.m (Scott 2009-01-04 143) {
	70befddd GITServerHandler.m (Scott 2009-03-22 144)         //NSLog(@"GATHER COMMI
	ad11ac80 GITPackUpload.m    (Scott 2009-03-24 145)
	ad11ac80 GITPackUpload.m    (Scott 2009-03-24 146)         NSString *parentSha;
	ad11ac80 GITPackUpload.m    (Scott 2009-03-24 147)         GITCommit *commit = [g
	ad11ac80 GITPackUpload.m    (Scott 2009-03-24 148)
	ad11ac80 GITPackUpload.m    (Scott 2009-03-24 149)         //NSLog(@"GATHER COMMI
	ad11ac80 GITPackUpload.m    (Scott 2009-03-24 150)
	56ef2caf GITServerHandler.m (Scott 2009-01-05 151)         if(commit) {
	56ef2caf GITServerHandler.m (Scott 2009-01-05 152)                 [refDict setOb
	56ef2caf GITServerHandler.m (Scott 2009-01-05 153)

This is really useful. Normally, the commit you see as the original commit is from when you copied the code over, because that was the first time you touched those lines in this file. Git tells you the original commit where you wrote those lines, even if they’re from another file.

### Binary Search ###

Annotating a file helps if you know where the issue is to begin with. If you don’t know what caused the problem, and there have been dozens or hundreds of commits since when the code last worked, turn to `git bisect` for help. This command does a binary search through your commit history to help identify as quickly as possible which commit introduced a problem.

Let’s say you just pushed out a release of your code to a production environment, and you’re getting bug reports about something that wasn’t happening in your development environment. You have no idea what’s going wrong. You go back to your code, and it turns out you can reproduce the issue, but you still can’t figure out what’s wrong. You can bisect the code to find out. First, run `git bisect start` to get things going, and then use `git bisect bad` to tell the system that the current commit you’re on is broken. Then, tell Git when the last known good state was by running `git bisect good [good_commit]`.

	$ git bisect start
	$ git bisect bad
	$ git bisect good v1.0
	Bisecting: 6 revisions left to test after this
	[ecb6e1bc347ccecc5f9350d878ce677feb13d3b2] error handling on repo

Git figured out that about 12 commits came between the commit you marked as the last good commit (v1.0) and the current bad version, and it checked out the middle commit for you. At this point, run your test to see if the issue exists in this commit. If it does, then it was introduced sometime before. If it doesn’t, then the problem was introduced sometime after. If it turns out there’s no issue here, tell Git by running `git bisect good` again and continue your journey.

	$ git bisect good
	Bisecting: 3 revisions left to test after this
	[b047b02ea83310a70fd603dc8cd7a6cd13d15c04] secure this thing

Now you’re on another commit, halfway between the one you just tested and your bad commit. When you run your test again you find that this commit is broken, so tell Git by running `git bisect bad`.

	$ git bisect bad
	Bisecting: 1 revisions left to test after this
	[f71ce38690acf49c1f3c9bea38e09d82a5ce6014] drop exceptions table

This commit is fine, and now Git has all the information it needs to determine where the issue was introduced. It tells you the SHA-1 hash of the first bad commit and shows some of the commit information and which files were modified in that commit so you can figure out what may have introduced this bug.

	$ git bisect good
	b047b02ea83310a70fd603dc8cd7a6cd13d15c04 is first bad commit
	commit b047b02ea83310a70fd603dc8cd7a6cd13d15c04
	Author: PJ Hyett <pjhyett@example.com>
	Date:   Tue Jan 27 14:48:32 2009 -0800

	    secure this thing

	:040000 040000 40ee3e7821b895e52c1695092db9bdc4c61d1730
	f24d3c6ebcfc639b1a3814550e62d60b8e68a8e4 M  config

When you’re finished, run `git bisect reset` to reset your HEAD to where you were before you started, or you’ll end up in a weird state.

	$ git bisect reset

This is a powerful tool that can help you check hundreds of commits for a mysterious bug in minutes. In fact, if you have a script that returns 0 if the project code is good or non-0 if the project code is bad, you can fully automate `git bisect`. First, again tell Git the scope of the bisect by providing the known bad and good commits. Do this by including them on the `bisect start` command line, placing the known bad commit first and the known good commit second.

	$ git bisect start HEAD v1.0
	$ git bisect run test-error.sh

This automatically runs `test-error.sh` on each checked-out commit until Git finds the first broken commit. You can also run something like `make`, `make tests`, or whatever you have that runs automated tests for you.

## Submodules ##

Sometimes you work on a project that itself depends on another project. Perhaps the other project is a library that someone else developed or that you’re developing separately and using in multiple parent projects. A common issue arises in these scenarios — how to treat the two projects as separate yet still be able to use one from within the other.

Here’s an example. Suppose you’re developing a web site and creating Atom feeds. Instead of writing your own Atom-generating code, you decide to use a library. You’re likely to have to either install the library using CPAN or a Ruby gem, or copy the library source code into your own project. The issue with including the source code is that it’s difficult to do any customization of the library, and using the library is a nuisance because you need to make sure every client has that library installed. The issue with incorporating the library code into your own project is that any custom changes you make are difficult to merge when changes to the library become available from its developer.

Git addresses this issue with submodules. Submodules allow you to keep one Git repository as a subdirectory of another Git repository. This lets you clone another repository into your project and keep your commits separate.

### Starting with Submodules ###

Suppose you want to add the Rack library (a Ruby web server gateway interface) to your project, possibly making your own changes to it, but you want to continue to merge changes from its developers. The first thing to do is to clone the external repository into your subdirectory. Add external projects as submodules with the `git submodule add` command.

	$ git submodule add git://github.com/chneukirchen/rack.git rack
	Initialized empty Git repository in /opt/subtest/rack/.git/
	remote: Counting objects: 3181, done.
	remote: Compressing objects: 100% (1534/1534), done.
	remote: Total 3181 (delta 1951), reused 2623 (delta 1603)
	Receiving objects: 100% (3181/3181), 675.42 KiB | 422 KiB/s, done.
	Resolving deltas: 100% (1951/1951), done.

Now you have the Rack project in a subdirectory named `rack` within your project. You can `cd` to that subdirectory, make changes, add your own writable remote repository to push your changes to, fetch and merge from the original repository, and more. If you run `git status` right after you add the submodule, you see two things.

	$ git status
	# On branch master
	# Changes to be committed:
	#   (use "git reset HEAD <file>..." to unstage)
	#
	#      new file:   .gitmodules
	#      new file:   rack
	#

First, you notice the `.gitmodules` file. This is a configuration file that stores the mapping between the project’s URL and the local subdirectory where you put it.

	$ cat .gitmodules
	[submodule "rack"]
	      path = rack
	      url = git://github.com/chneukirchen/rack.git

If you have multiple submodules, this file will have multiple entries. It’s important to note that Git tracks this file along with your other files, like your `.gitignore` file. It’s pushed and pulled with the rest of your project. This is how other people who clone this project know where to get the submodule projects from.

The other listing in the `git status` output is the `rack` entry. If you run `git diff` on that, you see something interesting.

	$ git diff --cached rack
	diff --git a/rack b/rack
	new file mode 160000
	index 0000000..08d709f
	--- /dev/null
	+++ b/rack
	@@ -0,0 +1 @@
	+Subproject commit 08d709f78b8c5b0fbeb7821e37fa53e69afcf433

Although `rack` is a subdirectory in your working directory, Git sees it as a submodule and doesn’t track its contents when you’re not in that directory. Instead, Git records it as a particular commit from that repository. When you make changes and commit in that subdirectory, the superproject notices that the HEAD there has changed and records the exact commit you’re currently working off of. That way, when others clone this project, they can re-create the environment exactly.

This is an important point with submodules. Reference them as the exact commit they’re at. You can’t reference a submodule at `master` or some other symbolic reference.

When you commit, you see something like

	$ git commit -m 'first commit with submodule rack'
	[master 0550271] first commit with submodule rack
	 2 files changed, 4 insertions(+), 0 deletions(-)
	 create mode 100644 .gitmodules
	 create mode 160000 rack

Notice the 160000 mode for the `rack` entry. That’s a special mode in Git that means you’re recording a commit as a directory entry rather than as a subdirectory or a file.

You can treat the `rack` directory as a separate project and then update your superproject from time to time with a pointer to the latest commit in that subproject. All the Git commands work independently in the two directories.

	$ git log -1
	commit 0550271328a0038865aad6331e620cd7238601bb
	Author: Scott Chacon <schacon@gmail.com>
	Date:   Thu Apr 9 09:03:56 2009 -0700

	    first commit with submodule rack
	$ cd rack/
	$ git log -1
	commit 08d709f78b8c5b0fbeb7821e37fa53e69afcf433
	Author: Christian Neukirchen <chneukirchen@gmail.com>
	Date:   Wed Mar 25 14:49:04 2009 +0100

	    Document version change

### Cloning a Project with Submodules ###

Here you clone a project containing a submodule. When you receive such a project, you get the directories that contain submodules, but none of the files yet.

	$ git clone git://github.com/schacon/myproject.git
	Initialized empty Git repository in /opt/myproject/.git/
	remote: Counting objects: 6, done.
	remote: Compressing objects: 100% (4/4), done.
	remote: Total 6 (delta 0), reused 0 (delta 0)
	Receiving objects: 100% (6/6), done.
	$ cd myproject
	$ ls -l
	total 8
	-rw-r--r--  1 schacon  admin   3 Apr  9 09:11 README
	drwxr-xr-x  2 schacon  admin  68 Apr  9 09:11 rack
	$ ls rack/
	$

The `rack` directory is there, but empty. You must run two commands: `git submodule init` to initialize your local configuration file, and `git submodule update` to fetch all the files from that project and check out the appropriate commit listed in your superproject.

	$ git submodule init
	Submodule 'rack' (git://github.com/chneukirchen/rack.git) registered for path 'rack'
	$ git submodule update
	Initialized empty Git repository in /opt/myproject/rack/.git/
	remote: Counting objects: 3181, done.
	remote: Compressing objects: 100% (1534/1534), done.
	remote: Total 3181 (delta 1951), reused 2623 (delta 1603)
	Receiving objects: 100% (3181/3181), 675.42 KiB | 173 KiB/s, done.
	Resolving deltas: 100% (1951/1951), done.
	Submodule path 'rack': checked out '08d709f78b8c5b0fbeb7821e37fa53e69afcf433'

Now your `rack` subdirectory is in the exact state it was in when you committed earlier. If another developer makes changes to the rack code and commits, and you pull that reference and merge it, you get something a bit odd.

	$ git merge origin/master
	Updating 0550271..85a3eee
	Fast forward
	 rack |    2 +-
	 1 files changed, 1 insertions(+), 1 deletions(-)
	[master*]$ git status
	# On branch master
	# Changes not staged for commit:
	#   (use "git add <file>..." to update what will be committed)
	#   (use "git checkout -- <file>..." to discard changes in working directory)
	#
	#      modified:   rack
	#

You merged in what is basically a change to the pointer for your submodule. But this doesn’t update the code in the submodule directory, so it looks like your working directory is in a dirty state.

	$ git diff
	diff --git a/rack b/rack
	index 6c5e70b..08d709f 160000
	--- a/rack
	+++ b/rack
	@@ -1 +1 @@
	-Subproject commit 6c5e70b984a60b3cecd395edd5b48a7575bf58e0
	+Subproject commit 08d709f78b8c5b0fbeb7821e37fa53e69afcf433


	$ git submodule update
	remote: Counting objects: 5, done.
	remote: Compressing objects: 100% (3/3), done.
	remote: Total 3 (delta 1), reused 2 (delta 0)
	Unpacking objects: 100% (3/3), done.
	From git@github.com:schacon/rack
	   08d709f..6c5e70b  master     -> origin/master
	Submodule path 'rack': checked out '6c5e70b984a60b3cecd395edd5b48a7575bf58e0'

You have to do this every time you pull down a submodule change in the main project. It’s strange, but it works.

One common problem happens when a developer makes a change locally in a submodule but doesn’t push it to a public server. Then, they commit a pointer to that non-public state and push the superproject. When other developers try to run `git submodule update`, the submodule system can’t find the referenced commit because it exists only on the first developer’s system. If that happens, you see an error like

	$ git submodule update
	fatal: reference isn’t a tree: 6c5e70b984a60b3cecd395edd5b48a7575bf58e0
	Unable to checkout '6c5e70b984a60b3cecd395edd5ba7575bf58e0' in submodule path 'rack'

You have to see who last changed the submodule.

	$ git log -1 rack
	commit 85a3eee996800fcfa91e2119372dd4172bf76678
	Author: Scott Chacon <schacon@gmail.com>
	Date:   Thu Apr 9 09:19:14 2009 -0700

	    added a submodule reference I will never make public. hahahahaha!

Then, you e-mail that guy and yell at him.

### Superprojects ###

Sometimes, developers want to get a combination of a large project’s subdirectories, depending on what team they’re on. This is common if you’re coming from CVS or Subversion, where you’ve defined a module or collection of subdirectories, and you want to keep this type of workflow.

A good way to do this in Git is to make each of the subdirectories a separate Git repository and then create superproject Git repositories that contain multiple submodules. A benefit of this approach is that you can more specifically define the relationships between the projects with tags and branches in the superprojects.

### Issues with Submodules ###

Using submodules isn’t without hiccups, however. First, be careful when working in the submodule directory. When you run `git submodule update`, it checks out the specific version of the project, but not within a branch. This is called having a detached HEAD — it means HEAD points directly to a commit, not to a symbolic reference. The problem is that you generally don’t want to work in a detached HEAD environment because it’s easy to lose changes. If you run an initial `git submodule update`, commit in that submodule directory without creating a branch to work in, and then run `git submodule update` again from the superproject without committing in the meantime, Git will overwrite your changes without telling you.  Technically you won’t lose the work, but you won’t have a branch pointing to it, so it will be difficult to retrieve.

To avoid this issue, create a branch when you work in a submodule directory with `git checkout -b work` or something equivalent. When you update the submodule a second time, it will still revert your work, but at least you have a pointer to get back to.

Switching branches containing submodules can also be tricky. If you create a new branch, add a submodule, and then switch back to a branch without that submodule, you still have the submodule directory as an untracked directory.

	$ git checkout -b rack
	Switched to a new branch "rack"
	$ git submodule add git@github.com:schacon/rack.git rack
	Initialized empty Git repository in /opt/myproj/rack/.git/
	...
	Receiving objects: 100% (3184/3184), 677.42 KiB | 34 KiB/s, done.
	Resolving deltas: 100% (1952/1952), done.
	$ git commit -am 'added rack submodule'
	[rack cc49a69] added rack submodule
	 2 files changed, 4 insertions(+), 0 deletions(-)
	 create mode 100644 .gitmodules
	 create mode 160000 rack
	$ git checkout master
	Switched to branch "master"
	$ git status
	# On branch master
	# Untracked files:
	#   (use "git add <file>..." to include in what will be committed)
	#
	#      rack/

You have to either move the submodule directory out of the way or remove it, in which case you have to clone it again when you switch back. You may lose local changes or branches that you didn’t push.

The last main caveat that many people run into involves switching from subdirectories to submodules. If you’ve been tracking files in your project and you want to move them out into a submodule, be careful or Git will get angry at you. Assume that you have the `rack` files in a subdirectory of your project, and you want to switch it to a submodule. If you delete the subdirectory and then run `submodule add`, Git yells at you.

	$ rm -Rf rack/
	$ git submodule add git@github.com:schacon/rack.git rack
	'rack' already exists in the index

You have to unstage the `rack` directory first. Then, add the submodule.

	$ git rm -r rack
	$ git submodule add git@github.com:schacon/rack.git rack
	Initialized empty Git repository in /opt/testsub/rack/.git/
	remote: Counting objects: 3184, done.
	remote: Compressing objects: 100% (1465/1465), done.
	remote: Total 3184 (delta 1952), reused 2770 (delta 1675)
	Receiving objects: 100% (3184/3184), 677.42 KiB | 88 KiB/s, done.
	Resolving deltas: 100% (1952/1952), done.

Suppose you did that in a branch. If you try to switch back to a branch in which those files are still in the actual tree rather than a submodule you get

	$ git checkout master
	error: Untracked working tree file 'rack/AUTHORS' would be overwritten by merge.

You have to move the `rack` submodule directory out of the way before you can switch to a branch that doesn’t contain it.

	$ mv rack /tmp/
	$ git checkout master
	Switched to branch "master"
	$ ls
	README	rack

Then, when you switch back, you get an empty `rack` directory. You can either run `git submodule update` to reclone, or move your `/tmp/rack` directory back into the empty directory.

## Subtree Merging ##

Now that you’ve seen the difficulties of dealing with submodules, let’s look at an alternate way to solve the same problem. When Git does a merge, it looks at what it has to merge and then chooses an appropriate merging strategy. If you’re merging two branches, Git uses a _recursive_ strategy. If you’re merging more than two branches, Git picks the _octopus_ strategy. These strategies are automatically chosen for you because the recursive strategy can’t handle complex three-way merges — for example, more than one common ancestor — it can only handle merging two branches. The octopus merge can handle multiple branches but is more cautious about avoiding difficult conflicts, so it’s chosen as the default strategy for merging more than two branches.

However, there are other strategies to choose from as well. One of them is the _subtree_ merge, which you can use to deal with the subproject issue. Here you’ll see how to do the same `rack` subdirectory embedding as in the last section, but using subtree merges instead.

The idea of the subtree merge is that you have two projects, and one of the projects maps to a subdirectory of the other one, and vice versa. When you specify a subtree merge, Git is smart enough to figure out that one is a subtree of the other and merge appropriately. It’s pretty amazing.

First add the Rack application to your project. Add the Rack project as a remote reference in your own project and then check it out into its own branch.

	$ git remote add rack_remote git@github.com:schacon/rack.git
	$ git fetch rack_remote
	warning: no common commits
	remote: Counting objects: 3184, done.
	remote: Compressing objects: 100% (1465/1465), done.
	remote: Total 3184 (delta 1952), reused 2770 (delta 1675)
	Receiving objects: 100% (3184/3184), 677.42 KiB | 4 KiB/s, done.
	Resolving deltas: 100% (1952/1952), done.
	From git@github.com:schacon/rack
	 * [new branch]      build      -> rack_remote/build
	 * [new branch]      master     -> rack_remote/master
	 * [new branch]      rack-0.4   -> rack_remote/rack-0.4
	 * [new branch]      rack-0.9   -> rack_remote/rack-0.9
	$ git checkout -b rack_branch rack_remote/master
	Branch rack_branch set up to track remote branch refs/remotes/rack_remote/master.
	Switched to a new branch "rack_branch"

Now you have the root of the Rack project in your `rack_branch` branch and your own project in the `master` branch. If you check out one and then the other, you see that they have different project roots.

	$ ls
	AUTHORS	       KNOWN-ISSUES   Rakefile      contrib	       lib
	COPYING	       README         bin           example	       test
	$ git checkout master
	Switched to branch "master"
	$ ls
	README

You want to pull the Rack project into your `master` project as a subdirectory. You can do that in Git by running `git read-tree`. You’ll learn more about `read-tree` and its friends in Chapter 9, but for now know that it reads the root tree of one branch into your current staging area and working directory. You just switched back to your `master` branch, and you pull the `rack` branch into the `rack` subdirectory of your `master` branch of your main project.

	$ git read-tree --prefix=rack/ -u rack_branch

When you commit, it looks like you have all the Rack files under that subdirectory, as though you copied them in from a tarball. What gets interesting is that you can fairly easily merge changes from one of the branches to the other. So, if the Rack project is updated, pull in upstream changes by switching to that branch and pulling.

	$ git checkout rack_branch
	$ git pull

Then, merge those changes back into your master branch. Running `git merge -s subtree` works fine but Git will also merge the histories together, which you probably don’t want. To pull in the changes and pre-populate the commit message, use the `--squash` and `--no-commit` options as well as the `-s subtree` strategy option.

	$ git checkout master
	$ git merge --squash -s subtree --no-commit rack_branch
	Squash commit -- not updating HEAD
	Automatic merge went well; stopped before committing as requested

All the changes from your Rack project are merged in and ready to be committed locally. You can also do the opposite — make changes in the `rack` subdirectory of your master branch and then merge them into your `rack_branch` branch later to submit them to the maintainers or push them upstream.

To get a diff between what’s in your `rack` subdirectory and the code in your `rack_branch` branch — to see if you need to merge them — you can’t use the normal `git diff` command. Instead, run `git diff-tree` with the branch you want to compare to.

	$ git diff-tree -p rack_branch

Or, to compare what’s in your `rack` subdirectory with what was in the `master` branch on the server the last time you fetched, run

	$ git diff-tree -p rack_remote/master

## Summary ##

You’ve seen a number of advanced tools that allow more precise manipulation of your commits and staging area. When you notice issues, you should be able to easily figure out what commit introduced them, when they were committed, and who committed them. You’ve also learned a few ways to use subprojects in your project. At this point, you should be able to comfortably do most of the things in Git that you’ll need every day.
