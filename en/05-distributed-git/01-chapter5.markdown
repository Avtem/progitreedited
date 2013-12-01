# Distributed Git #

Now that you have a remote Git repository where developers can share their code, and you’re familiar with running basic Git commands on a local repository, I’ll cover how to use Git with remote repositories.

In this chapter, you’ll see how to work with Git in a distributed environment as a contributor and as an integrator. You’ll learn how to contribute code to a project in a way that makes it as easy on yourself and the project maintainer as possible. You’ll also learn how to work with other developers to successfully maintain a project.

## Distributed Workflows ##

Unlike CVSs, the distributed nature of Git allows far more flexibility in how developers collaborate on projects. In centralized systems, every developer interacts more or less equally in a central repository. In Git, however, every developer can both contribute code to other repositories and maintain a public repository on which others can base their changes and can contribute to. This opens a vast range of workflow possibilities for your project and/or your team, so I’ll cover a few common patterns that take advantage of this flexibility. I’ll go over the strengths and possible weaknesses of each design. Choose a single one to use, or mix and match features from each.

### Centralized Workflow ###

In centralized systems, there’s generally a single collaboration model — the centralized workflow. One central shared repository contains the official code, and all developers synchronize their changes to it (see Figure 5-1).

Insert 18333fig0501.png
Figure 5-1. Centralized workflow.

This means that if two developers checkout code from the central repository and both make changes, the first developer to check in their changes can do so without having to worry about merging. The second developer must merge in their changes without overwriting the first developer’s changes.

If you have a small team or are already comfortable with a centralized workflow, you can continue using that workflow with Git. Simply set up a single repository, and give everyone on your team push access. Git won’t let users overwrite each other. If one developer clones, makes changes, and then pushes their changes before another developer pushes conflicting changes, the Git server will reject the second developer’s changes. They will be told that they’re trying to push non-fast-forward changes and that they won’t be able to do so until they fetch and merge the first developers changes.
This workflow is attractive because it’s a style that many are familiar and comfortable with.

### Integration-Manager Workflow ###

Because Git allows multiple remote repositories, it’s possible to have a workflow where each developer has their own public repository that they push to, and read access to everyone else’s. There’s also a blessed repository containing the "official" project code. To contribute to that project, clone the blessed repository into a public repository and push your changes to it. Then, send a request to the integration manager to pull your changes. They add your repository as a remote, test your changes in their own repository, merge them into a branch in their repository, and then push to the blessed repository. The process works as follows (see Figure 5-2):

1. A developer clones the blessed repository into a private repository and makes changes there.
2. The developer pushes their changes to their public repository.
3. The developer sends the integration manager an e-mail message asking them to pull changes.
4. The integration manager adds the developer’s public repository as a remote and merges it locally.
5. The integration manager pushes merged changes to the blessed repository.

Insert 18333fig0502.png
Figure 5-2. Integration-manager workflow.

This is a very common workflow with sites like GitHub, where it’s easy to fork a project and push your changes into your fork for everyone to see. One of the main advantages of this approach is that you can continue to work in your local repository, and the maintainer of the main repository can pull in your changes at any time. Contributors don’t have to wait for the project maintainer to incorporate their changes. Each party can work at their own pace.

### Dictator and Lieutenants Workflow ###

This is a variant of a multiple-repository workflow. It’s generally used by huge projects with hundreds of collaborators. One famous example is the Linux kernel. Various integration managers are in charge of certain parts of the repository — they’re called lieutenants. Above all the lieutenants in the chain of command is one integration manager known as the benevolent dictator. The benevolent dictator’s repository serves as the reference repository from which all the collaborators need to pull. The process works like this (see Figure 5-3):

1. Regular developers work on their topic branch and rebase their changes on top of dictator’s master branch.
2. Lieutenants merge the developers’ topic branches into the lieutenants’ master branches.
3. The dictator merges the lieutenants’ master branches into the dictator’s master branch.
4. The dictator pushes his master branch to the reference repository so the other developers can rebase on it.

Insert 18333fig0503.png
Figure 5-3. Benevolent dictator workflow.

This kind of workflow isn’t common but can be useful in very big projects or in highly hierarchical environments, since it allows the project leader (the dictator) to delegate much of the work and collect large subsets of code at multiple points before integrating them.

These are some commonly used workflows that are possible with a distributed system like Git, but many other variations are possible to suit your particular real-world workflow. Now that you can (I hope) determine which workflow may work for you, I’ll cover some more specific examples of how to accomplish the main roles that make up the different flows.

## Contributing to a Project ##

You know what the different workflows are, and you should have a pretty good grasp of fundamental Git usage. In this section, you’ll learn a few common patterns for contributing to a project.

The main difficulty with describing this process is the huge number of variations on how it’s done. Because Git is very flexible, people can and do work together in many ways, and it’s problematic to describe how you should contribute to a project. Every project is a bit different. Some of the variables involved are number of active contributors, chosen workflow, and commit access.

The first variable is the number of active contributors. How many users are actively contributing code to this project, and how often? In many instances, you’ll have two or three developers making a few commits a day, or possibly fewer for dormant projects. In really large projects, the number of developers could be in the thousands, with dozens or even hundreds of patches coming in each day. The number is important because with more and more developers, you run into more issues with making sure changes apply cleanly or can be easily merged. Changes you submit may be rendered obsolete or severely broken by changes that are merged in while you were working or while your changes were waiting to be approved or applied. How can you keep your code consistently up to date and your patches valid?

The next variable is the workflow in use for the project. Is it centralized, with each developer having equal write access to the main codeline? Does the project have an integration manager who checks all the patches? Are all the patches peer-reviewed and approved? Are you involved in that process? Is a lieutenant system in place, and do you have to submit your changes to them first?

The next issue is your commit access. Contributing to a project is much different if you have write access to the project than if you don’t. If you don’t have write access, how does the project prefer to accept contributed work? Does it even have a policy? How much work are you contributing at a time? How often do you contribute?

All these questions can affect how you contribute effectively to a project and what workflows are preferred or available. I’ll cover aspects of each of these in a series of use cases, moving from simple to more complex. You should be able to construct the specific workflows you need in practice from these examples.

### Commit Guidelines ###

Before you start looking at specific use cases, here’s a quick note about commit messages. Having good guideline sfor creating commits and sticking to them makes working with Git and collaborating with others a lot easier. The Git project provides a document that lays out a number of good tips for creating commits from which to submit patches — it’s in the Git source code in the `Documentation/SubmittingPatches` file.

First, don’t include any whitespace errors in your commit. Git provides an easy way to check for this. Before you commit, run `git diff --check`, which lists possible whitespace errors. Here’s an example, where I’ve replaced the red terminal color with `X`s.

	$ git diff --check
	lib/simplegit.rb:5: trailing whitespace.
	+    @git_dir = File.expand_path(git_dir)XX
	lib/simplegit.rb:7: trailing whitespace.
	+ XXXXXXXXXXX
	lib/simplegit.rb:26: trailing whitespace.
	+    def command(git_cmd)XXXX

If you run this before committing, you’ll see if your commit contains whitespace issues that may annoy other developers.

Next, try to make each commit logically self-contained. If you can, try to make your changes digestible. Don’t spend a whole weekend working on five different issues and then submit all your changes as one massive commit on Monday. Even if you don’t commit during the weekend, use the staging area on Monday to split your changes into at least one commit per issue, with a useful message per commit. If some of the changes modify the same file, try to use `git add --patch` to partially stage files (covered in detail in Chapter 6). The project snapshot at the tip of the branch is identical whether you make one commit or five, as long as all the changes are added at some point, so try to make things easier on your fellow developers when they have to review your changes. This approach also makes it easier to pull out or revert a commit later. Chapter 6 describes a number of useful Git tricks for rewriting history and interactively staging files. Use these tools to help craft a clean and understandable history.

The last thing to keep in mind is the commit message. Getting in the habit of creating quality commit messages makes collaborating with other developers a lot easier. As a general rule, your messages should start with a single line that’s no more than about 50 characters long that describes the commit concisely, followed by a blank line, followed by a more detailed explanation. The Git project requires that the more detailed explanation include your motivation for the change and contrast its implementation with previous behavior. This is a good guideline to follow. It’s also a good idea to use the imperative present tense in these messages. In other words, use commands. Instead of "I added tests for" or "Adding tests for," use "Add tests for."
Here’s a template originally written by Tim Pope at tpope.net:

	Short (50 chars or less) summary of changes

	More detailed explanatory text, if necessary.  Wrap it to about 72
	characters or so.  In some contexts, the first line is treated as the
	subject of an email and the rest of the text as the body.  The blank
	line separating the summary from the body is critical (unless you omit
	the body entirely); tools like rebase can get confused if you run the
	two together.

	Further paragraphs come after blank lines.

	 - Bullet points are okay, too

	 - Typically a hyphen or asterisk is used for the bullet, preceded by a
	   single space, with blank lines in between, but conventions vary here

If all your commit messages look like this, things will be a lot easier for you and the developers you work with. The Git project has well-formatted commit messages. I encourage you to run `git log --no-merges` in your copy of the Git repository to see what a nicely formatted project-commit history looks like.

In the following examples, and throughout most of this book, for the sake of brevity I don’t format messages nicely like this. Instead, I use the `-m` option to `git commit`. Do as I say, not as I do.

### Private Small Team ###

The simplest setup you’re likely to encounter is a private project with one or two other developers. By private, I mean closed source — not visible to the outside world. You and the other developers all have push access to the repository.

In this environment, you follow a workflow similar to what you might do when using Subversion or another centralized system. You still get the advantages of things like offline committing and vastly simpler branching and merging, but the workflow can be very similar. The main difference is that merges happen on the clients rather than on the server at commit time.
Let’s see what it might look like when two developers start to work together with a shared repository. The first developer, John, clones the repository, makes a change, and commits locally. (I’m replacing the protocol messages with `...` in these examples to shorten them.)

	# John's Machine
	$ git clone john@githost:simplegit.git
	Initialized empty Git repository in /home/john/simplegit/.git/
	...
	$ cd simplegit/
	$ vim lib/simplegit.rb
	$ git commit -am 'removed invalid default value'
	[master 738ee87] removed invalid default value
	 1 file changed, 1 insertion(+), 1 deletion(-)

The second developer, Jessica, does the same thing, clones the repository, and commits a change.

	# Jessica's Machine
	$ git clone jessica@githost:simplegit.git
	Initialized empty Git repository in /home/jessica/simplegit/.git/
	...
	$ cd simplegit/
	$ vim TODO
	$ git commit -am 'add reset task'
	[master fbff5bc] add reset task
	 1 file changed, 1 insertion(+), 0 deletions(-)

Now, Jessica pushes her changes up to the server.

	# Jessica's Machine
	$ git push origin master
	...
	To jessica@githost:simplegit.git
	   1edee6b..fbff5bc  master -> master

John tries to push his change up, too.

	# John's Machine
	$ git push origin master
	To john@githost:simplegit.git
	 ! [rejected]        master -> master (non-fast forward)
	error: failed to push some refs to 'john@githost:simplegit.git'

John isn’t allowed to push because Jessica has pushed in the meantime. This is especially important to understand if you’re used to Subversion, because you’ll notice that the two developers didn’t edit the same file. Although Subversion automatically does a merge on the server if different files are edited, in Git you must merge the commits locally. John has to fetch Jessica’s changes and merge them in before he will be allowed to push.

	$ git fetch origin
	...
	From john@githost:simplegit
	 + 049d078...fbff5bc master     -> origin/master

At this point, John’s local repository looks something like Figure 5-4.

Insert 18333fig0504.png
Figure 5-4. John’s initial repository.

John has a reference to the changes Jessica pushed, but he has to merge them into his own changes before he’s allowed to push.

	$ git merge origin/master
	Merge made by recursive.
	 TODO |    1 +
	 1 file changed, 1 insertion(+), 0 deletions(-)

The merge goes smoothly. John’s commit history now looks like Figure 5-5.

Insert 18333fig0505.png
Figure 5-5. John’s repository after merging origin/master.

Now, John can test his code to make sure it still works properly, and then he can push his new merged changes to the server.

	$ git push origin master
	...
	To john@githost:simplegit.git
	   fbff5bc..72bbc59  master -> master

Finally, John’s commit history looks like Figure 5-6.

Insert 18333fig0506.png
Figure 5-6. John’s history after pushing to the origin server.

In the meantime, Jessica has been working on a topic branch. She’s created a topic branch called `issue54` and made three commits on that branch. She hasn’t fetched John’s changes yet, so her commit history looks like Figure 5-7.

Insert 18333fig0507.png
Figure 5-7. Jessica’s initial commit history.

Jessica wants to sync up with John, so she fetches.

	# Jessica's Machine
	$ git fetch origin
	...
	From jessica@githost:simplegit
	   fbff5bc..72bbc59  master     -> origin/master

That pulls down the changes John has pushed in the meantime. Jessica’s history now looks like Figure 5-8.

Insert 18333fig0508.png
Figure 5-8. Jessica’s history after fetching John’s changes.

Jessica thinks her topic branch is ready, but she wants to know where she has to merge her changes so that she can push. She runs `git log` to find out.

	$ git log --no-merges origin/master ^issue54
	commit 738ee872852dfaa9d6634e0dea7a324040193016
	Author: John Smith <jsmith@example.com>
	Date:   Fri May 29 16:01:27 2009 -0700

	    removed invalid default value

Now, Jessica can merge her topic branch into her master branch, merge John’s changes (`origin/master`) into her `master` branch, and then push to the server again. First, she switches back to her master branch to integrate all this work.

	$ git checkout master
	Switched to branch "master"
	Your branch is behind 'origin/master' by 2 commits, and can be fast-forwarded.

She can merge either `origin/master` or `issue54` first. They’re both upstream, so the order doesn’t matter. The end result will be identical no matter which order she chooses. Only the history will be different. She chooses to merge in `issue54` first.

	$ git merge issue54
	Updating fbff5bc..4af4298
	Fast forward
	 README           |    1 +
	 lib/simplegit.rb |    6 +++++-
	 2 files changed, 6 insertions(+), 1 deletion(-)

No problems occur. As you can see, it was a simple fast-forward. Now Jessica merges in John’s changes (`origin/master`).

	$ git merge origin/master
	Auto-merging lib/simplegit.rb
	Merge made by recursive.
	 lib/simplegit.rb |    2 +-
	 1 file changed, 1 insertion(+), 1 deletion(-)

Everything merges cleanly, and Jessica’s history looks like Figure 5-9.

Insert 18333fig0509.png
Figure 5-9. Jessica’s history after merging John’s changes.

Now `origin/master` is reachable from Jessica’s `master` branch, so she should be able to successfully push (assuming John hasn’t pushed again in the meantime).

	$ git push origin master
	...
	To jessica@githost:simplegit.git
	   72bbc59..8059c15  master -> master

Each developer has committed a few times and merged each other’s changes successfully (see Figure 5-10).

Insert 18333fig0510.png
Figure 5-10. Jessica’s history after pushing all changes back to the server.

That’s one of the simplest workflows. Work for a while, generally in a topic branch, and merge into your master branch when the topic branch is ready to be integrated. When you want to share that work, fetch and merge `origin/master` if it has changed, and finally push to the `master` branch on the server. The general sequence is something like that shown in Figure 5-11.

Insert 18333fig0511.png
Figure 5-11. General sequence of events for a simple multiple-developer Git workflow.

### Private Managed Team ###

This next scenario looks at contributor roles in a larger private group. You’ll learn how to work in an environment where small groups collaborate on features and then those team-based contributions are integrated by somebody else.

Let’s say that John and Jessica are working together on one feature, while Jessica and Josie are working on a second. In this case, the company is using a type of integration-manager workflow where the changes of the individual groups are integrated only by certain engineers, and the `master` branch of the main repository can be updated only by those engineers. In this scenario, all changes are done in team-based branches and pulled together by the integrators later.

Let’s follow Jessica’s workflow as she works on her two features, collaborating in parallel with two different developers. Assuming she already clones her repository, she decides to work on `featureA` first. She creates a new branch for the feature and does some work on it there.

	# Jessica's Machine
	$ git checkout -b featureA
	Switched to a new branch "featureA"
	$ vim lib/simplegit.rb
	$ git commit -am 'add limit to log function'
	[featureA 3300904] add limit to log function
	 1 file changed, 1 insertion(+), 1 deletion(-)

At this point, she needs to share her changes with John, so she pushes her `featureA` branch commits to the Git server. Jessica doesn’t have push access to the `master` branch — only the integrators do — so she has to push to `featureA` in order to collaborate with John.

	$ git push origin featureA
	...
	To jessica@githost:simplegit.git
	 * [new branch]      featureA -> featureA

Jessica e-mails John to tell him that she’s pushed some changes into a branch named `featureA` on the Git server and he can look at it now. While she waits for feedback from John, Jessica decides to start working on `featureB` with Josie. To begin, she starts a new feature branch, `featureB`, basing it off the server’s `master` branch.

	# Jessica's Machine
	$ git fetch origin
	$ git checkout -b featureB origin/master
	Switched to a new branch "featureB"

Now, Jessica makes a couple of commits on the `featureB` branch.

	$ vim lib/simplegit.rb
	$ git commit -am 'made the ls-tree function recursive'
	[featureB e5b0fdc] made the ls-tree function recursive
	 1 file changed, 1 insertion(+), 1 deletion(-)
	$ vim lib/simplegit.rb
	$ git commit -am 'add ls-files'
	[featureB 8512791] add ls-files
	 1 file changed, 5 insertions(+), 0 deletions(-)

Jessica’s repository looks like Figure 5-12.

Insert 18333fig0512.png
Figure 5-12. Jessica’s initial commit history.

Jessica’s ready to push her work, but she gets an e-mail from Josie saying that she (Josie) created a branch with some initial changes in it called `featureBee` that she already pushed to the Git server. Jessica first needs to merge those changes into her `featureB` branch before she can push it to the Git server. She then fetches Josie’s changes by running `git fetch`.

	$ git fetch origin
	...
	From jessica@githost:simplegit
	 * [new branch]      featureBee -> origin/featureBee

Jessica can now merge this into the changes she did with `git merge`.

	$ git merge origin/featureBee
	Auto-merging lib/simplegit.rb
	Merge made by recursive.
	 lib/simplegit.rb |    4 ++++
	 1 file changed, 4 insertions(+), 0 deletions(-)

There’s a bit of a problem here. She needs to push the merged changes in her `featureB` branch to the `featureBee` branch on the server. She can do so by specifying the local branch followed by a colon (:), followed by the remote branch to the `git push` command.

	$ git push origin featureB:featureBee
	...
	To jessica@githost:simplegit.git
	   fba9af8..cd685d1  featureB -> featureBee

This is called a _refspec_. See Chapter 9 for a more detailed discussion of Git refspecs.

Next, John e-mails Jessica to say he’s pushed some changes to the `featureA` branch and asks her to verify them. She runs `git fetch` to pull those changes.

	$ git fetch origin
	...
	From jessica@githost:simplegit
	   3300904..aad881d  featureA   -> origin/featureA

She can then see what has changed by running `git log`.

	$ git log origin/featureA ^featureA
	commit aad881d154acdaeb2b6b18ea0e827ed8a6d671e6
	Author: John Smith <jsmith@example.com>
	Date:   Fri May 29 19:57:33 2009 -0700

	    changed log output to 30 from 25

Finally, she merges John’s changes into her own `featureA` branch.

	$ git checkout featureA
	Switched to branch "featureA"
	$ git merge origin/featureA
	Updating 3300904..aad881d
	Fast forward
	 lib/simplegit.rb |   10 +++++++++-
	1 file changed, 9 insertions(+), 1 deletion(-)

Jessica wants to tweak something, so she commits again and then pushes this commit back to the server.

	$ git commit -am 'small tweak'
	[featureA 774b3ed] small tweak
	 1 files changed, 1 insertions(+), 1 deletions(-)
	$ git push origin featureA
	...
	To jessica@githost:simplegit.git
	   3300904..774b3ed  featureA -> featureA

Jessica’s commit history now looks something like Figure 5-13.

Insert 18333fig0513.png
Figure 5-13. Jessica’s history after committing on a feature branch.

Jessica, Josie, and John inform the integrator that the `featureA` and `featureBee` branches on the server are ready for integration into the mainline. After the integrator incorporates these branches into the mainline, when Jessica runs `git fetch` this brings down the new merge commits into her repository, making her repository look like Figure 5-14.

Insert 18333fig0514.png
Figure 5-14. Jessica’s repository after merging both her topic branches.

Many groups switch to Git because of this ability to have multiple teams working in parallel, merging different branches late in the process. The ability of smaller subgroups of a team to collaborate via remote branches without necessarily having to involve or impede the entire team is a huge benefit of Git. The sequence for the workflow you saw here is something like Figure 5-15.

Insert 18333fig0515.png
Figure 5-15. Basic sequence of this managed-team workflow.

### Public Small Project ###

Contributing to public projects is a bit different. Because you don’t have permission to directly update branches in the project, you have to supply your changes to the maintainers some other way. This first example describes contributing via forking on Git hosting sites that support easy forking. The repo.or.cz and GitHub hosting sites both support this, and many project maintainers expect this style of contribution. The next section deals with projects that prefer to accept contributed patches via e-mail.

Start by cloning the main repository, creating a topic branch for the patch you’re planning to contribute, and start doing your work there. This looks basically like

	$ git clone (url)
	$ cd project
	$ git checkout -b featureA
	$ (work)
	$ git commit
	$ (work)
	$ git commit

You may want to run `git rebase -i` to squash your changes down to a single commit, or rearrange the changes in the commits to make the patch easier for the maintainer to review. See Chapter 6 for more information about interactive rebasing.

When you’re finished working in your topic branch and you’re ready to contribute it back to the maintainer, go to the original project page and click the "Fork" button, creating your own writable fork of the project. Add this new repository URL as a second remote, in this case named `myfork`.

	$ git remote add myfork (url)

Push your changes to the fork. It’s easiest to push the branch you’re working on to your fork, rather than merging into your master branch and pushing that. The reason is that if the changes aren’t accepted or are cherry picked, you don’t have to rewind your master branch. If the maintainers merge, rebase, or cherry-pick your work, you’ll eventually get it back by pulling from their repository anyhow.

	$ git push myfork featureA

After you push your changes to your fork, notify the maintainer. This is often called a pull request, and either generate it via the website — GitHub has a "pull request" button that automatically sends a message the maintainer — or run the `git request-pull` command and e-mail the output to the project maintainer manually.

The `git request-pull` command takes the base branch into which you want your topic branch pulled and the Git repository URL to pull from, and outputs a summary of all the changes you’re asking to be pulled. For instance, if Jessica wants to send John a pull request, and she’s done two commits on the topic branch she just pushed, she runs

	$ git request-pull origin/master myfork
	The following changes since commit 1edee6b1d61823a2de3b09c160d7080b8d1b3a40:
	  John Smith (1):
	        added a new function

	are available in the git repository at:

	  git://githost/simplegit.git featureA

	Jessica Smith (2):
	      add limit to log function
	      change log output to 30 from 25

	 lib/simplegit.rb |   10 +++++++++-
	 1 file changed, 9 insertions(+), 1 deletion(-)

She sends the output to the maintainer. It tells them where the changes were branched from, summarizes the commits, and tells where to pull from.

On a project for which you’re not the maintainer, it’s generally easier to have your `master` branch always track `origin/master` and to do your work in topic branches that you can easily discard if your changes are rejected. Isolating work into topic branches also makes it easier for you to rebase your changes if the tip of the main repository has moved in the meantime and your commits no longer apply cleanly. For example, to submit a second topic branch to the project, don’t continue working on the topic branch you just pushed. Start over from the main repository’s `master` branch.

	$ git checkout -b featureB origin/master
	$ (work)
	$ git commit
	$ git push myfork featureB
	$ (email maintainer)
	$ git fetch origin

Now, each of your topics is contained within a silo — similar to a patch queue — that you can rewrite, rebase, and modify without the topics interfering or interdepending on each other. Figure 5-16 shows this.

Insert 18333fig0516.png
Figure 5-16. Initial commit history with featureB work.

Let’s say the project maintainer has pulled in a bunch of other patches and tried your first branch, but it no longer cleanly merges. In this case, try to rebase that branch on top of `origin/master`, resolve the conflicts for the maintainer, and then resubmit your changes:

	$ git checkout featureA
	$ git rebase origin/master
	$ git push -f myfork featureA

This rewrites your history to now look like Figure 5-17.

Insert 18333fig0517.png
Figure 5-17. Commit history after featureA work.

Because you rebased the branch, you have to specify the `-f` option on your push command in order to replace the `featureA` branch on the server with a commit that isn’t a descendant of it. An alternative would be to push this branch to a different branch on the server (perhaps called `featureAv2`).

Let’s look at one more possible scenario: the maintainer has looked at the changes in your second branch and likes the concept but would like you to change an implementation detail. You’ll also take this opportunity to move the changes to be based off the project’s current `master` branch. You start a new branch based off the current `origin/master` branch, squash the `featureB` changes there, resolve any conflicts, make the implementation change, and then push that as a new branch.

	$ git checkout -b featureBv2 origin/master
	$ git merge --no-commit --squash featureB
	$ (change implementation)
	$ git commit
	$ git push myfork featureBv2

The `--squash` option takes all the changes on the merged branch and squashes them into one non-merge commit on top of the branch you’re on. The `--no-commit` option tells Git not to automatically record a commit. This allows you to incorporate all the changes from another branch and then make more changes before recording the new commit.

Now send the maintainer a message saying that you’ve made the requested changes and they can find those changes in your `featureBv2` branch (see Figure 5-18).

Insert 18333fig0518.png
Figure 5-18. Commit history after featureBv2 work.

### Public Large Project ###

Many larger projects have established procedures for accepting patches. You’ll need to check the specific rules for each project because they will differ. However, many larger public projects accept patches via a developer mailing list, so I’ll go over an example of that now.

The workflow is similar to the previous use case. Create topic branches for each patch series you work on. The difference is how you submit the patches to the project. Instead of forking the project and pushing to your own writable version, generate e-mail versions of each patch series and e-mail them to the developer mailing list.

	$ git checkout -b topicA
	$ (work)
	$ git commit
	$ (work)
	$ git commit

Now you have two commits that you want to send to the mailing list. Use `git format-patch` to generate the mbox-formatted files that you e-mail to the list. It turns each commit into an e-mail message with the first line of the commit message as the subject and the rest of the commit message, plus the patch that the commit introduces, as the body. The nice thing about this is that applying a patch from an e-mail generated with `git format-patch` preserves all the commit information properly, as you’ll see in the next section when you apply these patches.

	$ git format-patch -M origin/master
	0001-add-limit-to-log-function.patch
	0002-changed-log-output-to-30-from-25.patch

The `git format-patch` command outputs the names of the patch files it creates. The `-M` switch tells Git to look for renames. The files end up looking like

	$ cat 0001-add-limit-to-log-function.patch
	From 330090432754092d704da8e76ca5c05c198e71a8 Mon Sep 17 00:00:00 2001
	From: Jessica Smith <jessica@example.com>
	Date: Sun, 6 Apr 2008 10:17:23 -0700
	Subject: [PATCH 1/2] add limit to log function

	Limit log functionality to the first 20

	---
	 lib/simplegit.rb |    2 +-
	 1 file changed, 1 insertion(+), 1 deletion(-)

	diff --git a/lib/simplegit.rb b/lib/simplegit.rb
	index 76f47bc..f9815f1 100644
	--- a/lib/simplegit.rb
	+++ b/lib/simplegit.rb
	@@ -14,7 +14,7 @@ class SimpleGit
	   end

	   def log(treeish = 'master')
	-    command("git log #{treeish}")
	+    command("git log -n 20 #{treeish}")
	   end

	   def ls_tree(treeish = 'master')
	--
	1.6.2.rc1.20.g8c5b.dirty

You can also edit these patch files to add more information for the e-mail list that you don’t want to show up in the commit message. If you add text between the `--` line and the beginning of the patch (the `lib/simplegit.rb` line), then developers can read it in the email message; but applying the patch excludes it.

To e-mail this to an email list, either paste the file into your e-mail program or send it via a command-line email program. Pasting the text often causes formatting issues, especially with "smarter" clients that don’t preserve newlines and other whitespace appropriately. Luckily, Git provides a tool to help you send properly formatted patches via IMAP, which may be easier. I’ll demonstrate how to send a patch via Gmail, which happens to be the e-mail client many developers use. Read detailed instructions for a number of mail programs at the end of the aforementioned `Documentation/SubmittingPatches` file in the Git source code.

First, set up the imap section in your `~/.gitconfig` file. Set each value separately with a series of `git config` commands, or add them manually by editing the file with your favorite text editor. But in the end, your config file should look something like

	[imap]
	  folder = "[Gmail]/Drafts"
	  host = imaps://imap.gmail.com
	  user = user@gmail.com
	  pass = p4ssw0rd
	  port = 993
	  sslverify = false

If your IMAP server doesn’t use SSL, the last two lines aren’t necessary, and the host value will start with `imap://` instead of `imaps://`.
When your config file is set up, use `git send-email` to place the patch series in the Drafts folder of the specified IMAP server.

	$ git send-email *.patch
	0001-added-limit-to-log-function.patch
	0002-changed-log-output-to-30-from-25.patch
	Who should the emails appear to be from? [Jessica Smith <jessica@example.com>]
	Emails will be sent from: Jessica Smith <jessica@example.com>
	Who should the emails be sent to? jessica@example.com
	Message-ID to be used as In-Reply-To for the first email? y

Then, Git produces a bunch of log information that looks something like this for each patch you’re sending.

	(mbox) Adding cc: Jessica Smith <jessica@example.com> from
	  \line 'From: Jessica Smith <jessica@example.com>'
	OK. Log says:
	Sendmail: /usr/sbin/sendmail -i jessica@example.com
	From: Jessica Smith <jessica@example.com>
	To: jessica@example.com
	Subject: [PATCH 1/2] added limit to log function
	Date: Sat, 30 May 2009 13:29:15 -0700
	Message-Id: <1243715356-61726-1-git-send-email-jessica@example.com>
	X-Mailer: git-send-email 1.6.2.rc1.20.g8c5b.dirty
	In-Reply-To: <y>
	References: <y>

	Result: OK

At this point, you should be able to go to your Drafts folder, change the `To` header to the mailing list you’re sending the patch to, possibly CC the maintainer or person responsible for that section, and send the patch off.

### Summary ###

This section covered a number of common workflows for dealing with several very different types of Git project collaboration styles you’re likely to encounter and introduced a couple of new tools to help you manage this process. Next, you’ll see how to work the other side of the coin: maintaining a Git project. You’ll learn how to be a benevolent dictator or integration manager.

## Maintaining a Project ##

In addition to knowing how to effectively contribute to a project, you’ll likely need to know how to maintain one. This can consist of accepting and applying patches generated via `git format-patch` and e-mailed to you, or integrating changes in remote branches for repositories you’ve added as remotes to your project. Whether you maintain a canonical repository or want to help by verifying or approving patches, you need to know how to accept changes in a way that’s clearest to other contributors and sustainable by you over the long run.

### Working in Topic Branches ###

When you integrate new changes, it’s generally a good idea to try doing it in a topic branch — a temporary branch specifically made to try out that new idea. This way, it’s easy to work on a patch separately and switch to another branch until you have time to come back to it if the patch isn’t working. If you create a simple branch name based on the theme of the change, such as `ruby_client` or something similarly descriptive, you can easily remember what the branch is for. The maintainer of the Git project tends to namespace these branches as well, such as `sc/ruby_client`, where `sc` is short for the person who contributed the changes.
As I mentioned, create a branch based off your master branch like this.

	$ git branch sc/ruby_client master

Or, to also switch to it immediately, use the `checkout -b` command.

	$ git checkout -b sc/ruby_client master

Now you’re ready to add your changes into this topic branch and determine if you want to merge it into branches that you’re planning on keeping for a while.

### Applying Patches from E-mail ###

If you receive a patch over e-mail to integrate into your project, apply the patch in a topic branch to evaluate it. There are two ways to apply an e-mailed patch: with `git apply` or with `git am`.

#### Applying a Patch with apply ####

If you received the patch from someone who generated it with `git diff` or the Unix `diff` command, apply the patch with `git apply`. Assuming you saved the patch in `/tmp/patch-ruby-client.patch`, apply the patch like this.

	$ git apply /tmp/patch-ruby-client.patch

This modifies files in your working directory. It’s almost identical to running a `patch -p1` command to apply the patch, although `git apply` accepts fewer fuzzy matches than patch. It also handles file adds, deletes, and renames if they’re described in the `git diff` format, which `patch` won’t do. Finally, `git apply` follows an "apply all or abort all" model where either everything is applied or nothing is, whereas `patch` can partially apply patchfiles, leaving your working directory in a weird state. `git apply` is overall much more paranoid than `patch`. It won’t create a commit for you. After running it, you must manually stage and commit the changes.

You can also run `git apply --check` to see if a patch applies cleanly before you try actually applying it.

	$ git apply --check 0001-seeing-if-this-helps-the-gem.patch
	error: patch failed: ticgit.gemspec:1
	error: ticgit.gemspec: patch does not apply

If there’s no output, then the patch should apply cleanly. This command also exits with a non-zero status if the check fails, so you can use it in scripts.

#### Applying a Patch with am ####

If the contributor is a Git user and was good enough to use the `git format-patch` command to generate their patch, then your job is easier because the patch contains author information and a commit message. If you can, encourage your contributors to use `git format-patch` instead of `diff` to generate patches.

To apply a patch generated by `git format-patch`, use `git am`. Technically, `git am` is built to read an mbox file, which is a simple, plain-text format for storing one or more e-mail messages in one text file. Such a file might look something like this:

	From 330090432754092d704da8e76ca5c05c198e71a8 Mon Sep 17 00:00:00 2001
	From: Jessica Smith <jessica@example.com>
	Date: Sun, 6 Apr 2008 10:17:23 -0700
	Subject: [PATCH 1/2] add limit to log function

	Limit log functionality to the first 20

This is the beginning of the output of the `git format-patch` command that you saw in the previous section. This is also in valid mbox e-mail format. If someone has e-mailed you the patch properly using `git send-email`, and you save the patch in mbox format, then point `git am` to that mbox file, and `git am` will start applying all the patches it sees. If you run a mail client that can save several e-mails in mbox format, you can save an entire patch series in one file and then use `git am` to apply the patches one at a time.

However, if someone uploaded a patch file generated via `git format-patch` to a ticketing system or something similar, save the file locally and then pass its filename to `git am` to apply it.

	$ git am 0001-limit-log-function.patch
	Applying: add limit to log function

This shows that the patch went in cleanly and `git am` automatically created a new commit. The author information is taken from the e-mail’s `From` and `Date` headers, and the message of the commit is taken from the `Subject` and body (before the patch) of the e-mail. For example, if the patch from the mbox example showed above were applied, the resulting commit would look something like

	$ git log --pretty=fuller -1
	commit 6c5e70b984a60b3cecd395edd5b48a7575bf58e0
	Author:     Jessica Smith <jessica@example.com>
	AuthorDate: Sun Apr 6 10:17:23 2008 -0700
	Commit:     Scott Chacon <schacon@gmail.com>
	CommitDate: Thu Apr 9 09:19:06 2009 -0700

	   add limit to log function

	   Limit log functionality to the first 20

The `Commit` information indicates the person who applied the patch and the time it was applied. The `Author` information is the individual who originally created the patch and when it was created.

But it’s possible that the patch doesn’t apply cleanly. Perhaps your main branch has diverged too far from the branch the patch was built from, or the patch depends on another patch you haven’t applied yet. In that case, `git am` will fail and ask what you want to do.

	$ git am 0001-seeing-if-this-helps-the-gem.patch
	Applying: seeing if this helps the gem
	error: patch failed: ticgit.gemspec:1
	error: ticgit.gemspec: patch does not apply
	Patch failed at 0001.
	When you have resolved this problem run "git am --resolved".
	If you would prefer to skip this patch, instead run "git am --skip".
	To restore the original branch and stop patching run "git am --abort".

This command puts conflict markers in any files it has issues with, much like a conflicted merge or rebase operation. Solve this issue much the same way. Edit the file to resolve the conflict, stage the new file, and then run `git am --resolved` to continue to the next patch.

	$ (fix the file)
	$ git add ticgit.gemspec
	$ git am --resolved
	Applying: seeing if this helps the gem

For Git to try a bit more intelligently to resolve the conflict, include the `-3` option, which makes Git attempt a three-way merge. This option isn’t on by default because it doesn’t work if the commit the patch says it was based on isn’t in your repository. If you do have that commit — if the patch was based on a public commit — then the `-3` option is generally much smarter about resolving a conflicting patch.

	$ git am -3 0001-seeing-if-this-helps-the-gem.patch
	Applying: seeing if this helps the gem
	error: patch failed: ticgit.gemspec:1
	error: ticgit.gemspec: patch does not apply
	Using index info to reconstruct a base tree...
	Falling back to patching base and 3-way merge...
	No changes -- Patch already applied.

In this case, I was trying to apply a patch I’d already applied. Without the `-3` option, it looks like a conflict.

If you’re applying a number of patches from an mbox file, you can also run `git am` in interactive mode, which stops at each patch it finds and asks if you want to apply it.

	$ git am -3 -i mbox
	Commit Body is:
	--------------------------
	seeing if this helps the gem
	--------------------------
	Apply? [y]es/[n]o/[e]dit/[v]iew patch/[a]ccept all

This is nice if you have a number of saved patches, because you can view the patch first if you don’t remember what it is, or choose not apply the patch if it’s already been applied.

When all the patches for your topic are applied and committed into your branch, choose whether and how to integrate them into a longer-lived branch.

### Checking Out Remote Branches ###

If you received a contribution from a user who set up their own Git repository, pushed a number of changes into it, and then sent you the URL to the repository and the name of the remote branch the changes are in, add the repository as a remote and merge locally.

For instance, if Jessica sends you an e-mail saying that she has a great new feature in the `ruby-client` branch of her repository, test it by adding the remote branch and checking it out locally.

	$ git remote add jessica git://github.com/jessica/myproject.git
	$ git fetch jessica
	$ git checkout -b rubyclient jessica/ruby-client

If she e-mails you again later with another branch containing another great feature, you can fetch and checkout the branch because you already have the remote set up.

This is most useful if you’re working with a person on a regular basis. If someone only has a single patch to contribute once in a while, then accepting it over e-mail may be less time consuming than requiring everyone to run their own Git server and you having to continually add and remove remotes to get a few patches. You’re also unlikely to want hundreds of remotes, each for someone who contributes only a patch or two. However, scripts and hosted services may make this easier. It depends largely on how you and your contributors collaborate.

The other advantage of this approach is that you get the history of the commits as well. Although you may have legitimate merge issues, you know where in your history their changes are based. A proper three-way merge is the default rather than having to supply the `-3` option and hope the patch was generated off a public commit to which you have access.

If you aren’t working with a person consistently but still want to pull from them in this way, provide the URL of the remote repository to the `git pull` command. This does a one-time pull and doesn’t save the URL as a remote reference.

	$ git pull git://github.com/onetimeguy/project.git
	From git://github.com/onetimeguy/project
	 * branch            HEAD       -> FETCH_HEAD
	Merge made by recursive.

### Determining What Is Introduced ###

Now you have a topic branch that contains contributed changes. At this point, determine what you’d like to do with it. This section revisits a couple of commands to show how to review exactly what you’ll be introducing if you merge the changes into your main branch.

It’s often helpful to review all the commits in this branch but not in your master branch. Exclude commits in the master branch by adding the `--not` option before the branch name. For example, if your contributor sends two patches and you create a branch called `contrib` where you apply those patches, run

	$ git log contrib --not master
	commit 5b6235bd297351589efc4d73316f0a68d484f118
	Author: Scott Chacon <schacon@gmail.com>
	Date:   Fri Oct 24 09:53:59 2008 -0700

	    seeing if this helps the gem

	commit 7482e0d16d04bea79d0dba8988cc78df655f16a0
	Author: Scott Chacon <schacon@gmail.com>
	Date:   Mon Oct 22 19:38:36 2008 -0700

	    updated the gemspec to hopefully work better

To see what changes each commit introduces, remember to pass the `-p` option to `git log` to append the diff introduced in each commit.

To see a full diff of what would happen if you were to merge this topic branch with another branch, you may have to use a weird trick to get the correct results. You may think to run

	$ git diff master

This command gives you a diff, but it may be misleading. If your `master` branch has moved forward since you created the topic branch from it, then you’ll get strange looking results. This happens because Git directly compares the snapshot of the last commit of the topic branch you’re on and the snapshot of the last commit on the `master` branch. For example, if you’ve added a line in a file on the `master` branch, a direct comparison of the snapshots will look like the topic branch is going to remove that line.

If `master` is a direct ancestor of your topic branch, this isn’t a problem. But if the two histories have diverged, the diff will look like you’re adding all the new stuff in your topic branch and removing everything unique to the `master` branch.

What you really want to see are the changes added to the topic branch — the changes you’ll introduce if you merge this branch with master. Do that by comparing the last commit on your topic branch with the first common ancestor it has with the master branch.

Technically, you do that by explicitly figuring out the common ancestor and then running diff on it.

	$ git merge-base contrib master
	36c7dba2c95e6bbb78dfa822519ecfec6e1ca649
	$ git diff 36c7db

However, that isn’t convenient, so Git provides a shorthand for doing the same thing: the triple-dot syntax. In a `git diff` command, put three periods after another branch to do a `diff` between the last commit of the branch you’re on and its common ancestor with the other branch.

	$ git diff master...contrib

This command shows only the changes your current topic branch has introduced since its common ancestor with master. That’s a very useful syntax to remember.

### Integrating Contributed Changes ###

When all the changes in your topic branch are ready to be integrated into a more mainline branch, the question is how to do it. Furthermore, what overall workflow do you want to use to maintain your project? You have a number of choices, so I’ll cover a few of them.

#### Merging Workflows ####

One simple workflow merges your changes into your `master` branch. In this scenario, you have a `master` branch that contains basically stable code. When you’ve verified the changes in a topic branch that you created or that someone else has contributed, merge the topic branch into your master branch, delete the topic branch, and then continue the process.  If you have a repository with changes in two branches named `ruby_client` and `php_client` that looks like Figure 5-19, and you merge `ruby_client` first and then `php_client` next, then your history will end up looking like Figure 5-20.

Insert 18333fig0519.png
Figure 5-19. History with several topic branches.

Insert 18333fig0520.png
Figure 5-20. After a topic branch merge.

That’s probably the simplest workflow, but it’s problematic if you’re dealing with larger repositories or projects.

If you have more developers or a larger project, you’ll probably want to use at least a two-phase merge cycle. In this scenario, you have two long-running branches, `master` and `develop`. `master` is updated only when a very stable release is ready and all new code is integrated into the `develop` branch. You regularly push both of these branches to the public repository. Each time you have a new topic branch to merge in (Figure 5-21), merge it into `develop` (Figure 5-22). Then, when you tag a release, fast-forward `master` to wherever the now-stable `develop` branch is (Figure 5-23).

Insert 18333fig0521.png
Figure 5-21. Before a topic branch merge.

Insert 18333fig0522.png
Figure 5-22. After a topic branch merge.

Insert 18333fig0523.png
Figure 5-23. After a topic branch release.

This way, when somebody clones your project’s repository, they can either checkout `master` to build the latest stable version and keep `master` up to date, or they can check out `develop`, which is the more cutting-edge stuff.
You can also continue this concept, having a branch for integration where all the changes are merged together. Then, when the codebase on that branch is stable and passes any tests, merge it into a `develop` branch. When that has proven itself stable for a while, fast-forward your `master` branch.

#### Large-Merging Workflows ####

The Git project has four long-running branches: `master`, `next`, `pu` (proposed updates) for new work, and `maint` for maintenance backports. When new changes are introduced by contributors, they’re collected into topic branches in the maintainer’s repository in a manner similar to what I’ve described (see Figure 5-24). At this point, the maintainer evaluates the topics to determine whether they’re safe and ready for consumption or whether they need more work. If they’re safe, they’re merged into `next`, and that branch is pushed so everyone can try the topics integrated together.

Insert 18333fig0524.png
Figure 5-24. Managing a complex series of parallel contributed topic branches.

If the topic branches still need work, they’re merged into `pu` instead. When it’s determined that they’re totally stable, the topics are re-merged into `master` and are then rebuilt from the topic branches that were in `next` but didn’t yet graduate to `master`. This means `master` almost always moves forward, `next` is rebased occasionally, and `pu` is rebased even more often (see Figure 5-25).

Insert 18333fig0525.png
Figure 5-25. Merging contributed topic branches into long-term integration branches.

When a topic branch has finally been merged into `master`, it’s removed from the repository. The Git project also has a `maint` branch that’s forked off from the last release to provide backported patches in case a maintenance release is required. Thus, when you clone the Git repository, you have four branches that you can check out to evaluate the project in different stages of development, depending on how cutting edge you want to be or how you want to contribute. The maintainer has a structured workflow to help vet new contributions.

#### Rebasing and Cherry Picking Workflows ####

Other maintainers prefer to rebase or cherry-pick contributed changes on top of their master branch, rather than merging them, to keep a mostly linear history. When you have changes in a topic branch and have determined that you want to integrate them, move to that branch and run `git rebase` to rebuild the changes on top of your current `master` (or `develop`, and so on) branch. If that works well, fast-forward your `master` branch, and you’ll end up with a linear project history.

The other way to move introduced changes from one branch to another is to cherry-pick them. A cherry-pick in Git is like a rebase for a single commit. It takes the patch that was introduced in a commit and tries to reapply it on the branch you’re currently on. This is useful if you have a number of commits on a topic branch and you want to integrate only one of them, or if you only have one commit on a topic branch and you’d prefer to cherry-pick it rather than run `git rebase`. For example, suppose you have a project that looks like Figure 5-26.

Insert 18333fig0526.png
Figure 5-26. Example history before a cherry pick.

To pull commit `e43a6` onto your master branch, run

	$ git cherry-pick e43a6fd3e94888d76779ad79fb568ed180e5fcdf
	Finished one cherry-pick.
	[master]: created a0a41a9: "More friendly message when locking the index fails."
	 3 files changed, 17 insertions(+), 3 deletions(-)

This pulls the same change introduced in `e43a6`, but you get a new commit SHA-1 hash, because the date when you did the commit is different. Now your history looks like Figure 5-27.

Insert 18333fig0527.png
Figure 5-27. History after cherry-picking a commit on a topic branch.

Now remove your topic branch and drop the commits you didn’t want to pull in.

### Tagging Your Releases ###

When you’ve decided to create a release, you’ll probably want to create a tag as I discussed in Chapter 2 so you can re-create that release at any point going forward. If you decide to sign the tag as the maintainer, the tagging may look something like

	$ git tag -s v1.5 -m 'my signed 1.5 tag'
	You need a passphrase to unlock the secret key for
	user: "Scott Chacon <schacon@gmail.com>"
	1024-bit DSA key, ID F721C45A, created 2009-02-09

If you do sign your tags, you’ll have the problem of distributing the public PGP key you used to sign your tags. The maintainer of the Git project has solved this issue by including his public key as a blob in the repository and then adding a tag that points directly to that content. To do this, figure out which key you want by running `gpg --list-keys`.

	$ gpg --list-keys
	/Users/schacon/.gnupg/pubring.gpg
	---------------------------------
	pub   1024D/F721C45A 2009-02-09 [expires: 2010-02-09]
	uid                  Scott Chacon <schacon@gmail.com>
	sub   2048g/45D02282 2009-02-09 [expires: 2010-02-09]

Then, directly import the key into the Git database by exporting it from `gpg` and piping the key through `git hash-object`, which writes a new blob with the key’s contents into Git and gives you back the SHA-1 hash of the blob.

	$ gpg -a --export F721C45A | git hash-object -w --stdin
	659ef797d181633c87ec71ac3f9ba29fe5775b92

Now that you have the contents of your key in Git, create a tag that points directly to it by specifying the new SHA-1 hash that the `hash-object` command gave you.

	$ git tag -a maintainer-pgp-pub 659ef797d181633c87ec71ac3f9ba29fe5775b92

If you run `git push --tags`, the `maintainer-pgp-pub` tag will be shared with everyone. If anyone wants to verify a tag, they can directly import your PGP key by pulling the blob directly out of the database and importing it into GPG.

	$ git show maintainer-pgp-pub | gpg --import

They can use that key to verify all your signed tags. Also, if you include instructions in the tag message, running `git show <tag>` will give the end user more specific instructions about tag verification.

### Generating a Build Number ###

Because Git doesn’t give each commit a monotonically increasing number, like 'v123' or the equivalent, if you want a human-readable name to go with a commit, run `git describe` on that commit. This shows the name of the nearest tag with the number of commits after that tag and a partial SHA-1 hash of the commit you’re describing.

	$ git describe master
	v1.6.2-rc1-20-g8c5b85c

This way, you can export a snapshot or a build and name it something understandable to people. In fact, if you build Git from source code cloned from the Git repository, `git --version` shows something similar. If you’re describing a commit that you’ve directly tagged, it gives you the tag name.

The `git describe` command favors annotated tags (tags created with the `-a` or `-s` flag), so release tags should be created this way if you’re using `git describe` to ensure the commit is named properly. You can also use this string as the target of `git checkout` or `git show`, although it relies on the abbreviated SHA-1 hash at the end, so it may not be valid forever. For instance, the Linux kernel tag recently jumped from 8 to 10 characters to ensure SHA-1 hash object uniqueness, so older `git describe` output names were invalidated.

### Preparing a Release ###

Now you want to release a build. One of the things you’ll do is create an archive of the latest snapshot of your code for those poor souls who don’t use Git. The command to do this is `git archive`.

	$ git archive master --prefix='project/' | gzip > `git describe master`.tar.gz
	$ ls *.tar.gz
	v1.6.2-rc1-20-g8c5b85c.tar.gz

That tarball contains the latest snapshot of your project under a project directory. You also create a zip archive in much the same way, but by passing the `--format=zip` option to `git archive`.

	$ git archive master --prefix='project/' --format=zip > `git describe master`.zip

You now have a nice tarball and a zip archive of your project release that you upload to your website or e-mail to people.

### The Shortlog ###

It’s time to send email to your mailing list of people who want to know what’s happening in your project. A nice way of quickly getting a sort of changelog of what has been added to your project since your last release is to use the `git shortlog` command. It summarizes all the commits in the range you give. For example, the following shows a summary of all the commits since your last release, if your last release was named v1.0.1:

	$ git shortlog --no-merges master --not v1.0.1
	Chris Wanstrath (8):
	      Add support for annotated tags to Grit::Tag
	      Add packed-refs annotated tag support.
	      Add Grit::Commit#to_patch
	      Update version and History.txt
	      Remove stray `puts`
	      Make ls_tree ignore nils

	Tom Preston-Werner (4):
	      fix dates in history
	      dynamic version method
	      Version bump to 1.0.1
	      Regenerated gemspec for version 1.0.2

You get a clean summary of all the commits since v1.0.1, grouped by author, that you can e-mail to your list.

## Summary ##

You should feel fairly comfortable contributing to a project managed by Git as well as maintaining your own project or integrating other users’ contributions. Congratulations on being an effective Git developer! In the next chapter, you’ll learn more powerful tools and tips for dealing with complex situations, which will truly make you a Git master.
