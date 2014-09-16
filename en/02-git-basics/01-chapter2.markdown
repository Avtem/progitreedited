# Git Basics #

If you only read one chapter in this book, this should be the one. This chapter covers every basic command you need to do the vast majority of the things you’ll eventually spend your time doing with Git. By the end of the chapter, you should be able to configure and initialize a repository, begin and stop tracking files, and stage and commit changes. I’ll also show how to set up Git to ignore certain files and file patterns, how to undo mistakes quickly and easily, how to browse the history of your project and view changes between commits, and how to push and pull from remote repositories.

## Creating a Git Repository ##

Create a Git repository using either of two methods. The first takes an existing project not managed by Git and puts it under Git control. The second clones an existing Git repository.

### Initializing a Repository in an Existing Directory ###

To start managing an existing project, go to the project’s top-level directory and run

	$ git init

This creates a new directory named `.git` that contains all necessary repository files — a Git repository skeleton. At this point, Git isn’t managing, or "tracking", anything in your project yet. (See *Chapter 9* for more information about exactly what files are contained in the `.git` directory you just created.)

To start managing existing files, tell Git to manage those files. You can accomplish that with a few `git add` commands that specify the files you want to manage.

	$ git add *.c
	$ git add README

Next, put a copy of the managed files into the Git repository by committing the files.

	$ git commit -m 'initial project version'

I’ll go over what these commands do in just a minute. Before I do, keep in mind that managing a file and tracking a file mean the same thing. A file is managed, or tracked, when Git is keeping track of the changes to it.

At this point, you have a Git repository containing tracked files and an initial commit.

### Cloning an Existing Repository ###

To get a copy of an existing Git repository — for example, a project you’d like to contribute to — the command is `git clone`. If you’re familiar with other VCSs, such as Subversion, you’ll notice that the command does a `clone` and not a `checkout`. This is an important distinction — `git clone` receives a copy of nearly everything contained in the repository you’re cloning from. Every version of every file for the entire history of the project is transferred when you run `git clone`. In fact, if the disk holding your official project repository gets corrupted, you can use any of the clones of the repository on any client to restore the server back to the state it was in when the clones were done (you may lose some server-side hooks and such, but all the versioned data would be there — see *Chapter 4* for more details).

You clone a repository by running `git clone [url]`. For example, to clone the Ruby Git library called Grit, run

	$ git clone git://github.com/schacon/grit.git

This creates a working directory named `grit`, initializes a `.git` directory inside it, pulls everything from the repository you’re cloning from, and checks out a working copy of the latest version into the directory named `grit`. Inside `grit` you’ll see the files that make up the project, ready to be worked on. To clone the repository into a working directory named something other than `grit`, specify the directory name you want as a command-line option.

	$ git clone git://github.com/schacon/grit.git mygrit

This command does the same thing as the previous one, but the new working directory is called `mygrit`.

Git supports a number of different transfer protocols. The previous example uses the `git` protocol, but you may also use `http(s)` or `ssh`. *Chapter 4* introduces all of the available options for accessing a remote Git repository, along with their pros and cons.

## Recording Changes to the Repository ##

You now have a bona fide Git repository and a working directory containing the files in that project. After you’ve made enough changes to reach a state you want to record,  commit a snapshot of your working directory into the repository.

This is an easy process to understand. However, there’s an intermediate step that you might find puzzling at first.  You don’t just directly commit a snapshot from your working directory into the Git repository. Instead, using the `git add` command, first add the files that you want to be part of the next commit into the staging area. Think of the staging area as standing between your working directory and the Git repository. Files in the staging area are called *tracked* files because these are the files that Git is keeping track of.

What about the *untracked* files? It’s not hard to imagine that your working directory might contain what I call “throw away” files that are created as part of your development work. Examples of such files are temporary editor files, object files and libraries, and executable files. There’s no point in having Git track them because you can always recreate them. You also have no intention of committing them into your Git repository.

When you first clone a repository, all of the files in your working directory will be tracked because you just checked them out from a repository managed by Git. These files are obviously managed by Git.

Files are also *unmodified*, *modified*, or *staged*. An unmodified file hasn’t changed since your last commit. As you edit files, Git sees them as modified, because you’ve changed them since your last commit. You *stage* these modified files then commit all your staged changes, and the cycle repeats. This lifecycle is illustrated in Figure 2-1.

Insert 18333fig0201.png
Figure 2-1. The lifecycle of the status of your files.

### Checking the Status of Your Files ###

The command that shows the status of files is `git status`. If you run this command directly after a clone, you see something like

	$ git status
	# On branch master
	nothing to commit (working directory clean)

This means you have a clean working directory — in other words, all tracked files are unmodified. This means that files in the staging area are identical to the files in the working directory and in the repository. Git also doesn’t see any untracked files, or they would be listed here. Finally, the command shows which branch you’re on. In this chapter that is always `master`, which is the default. I won’t go into branches here but I go over them in detail in the next chapter.

Let’s say you add a new file to your project — a simple `README` file. If the file didn’t exist before, and you run `git status`, the file appears as untracked.

	$ vim README
	$ git status
	# On branch master
	# Untracked files:
	#   (use "git add <file>..." to include in what will be committed)
	#
	#	README
	nothing added to commit but untracked files present (use "git add" to track)

You can see that `README` is untracked, because it’s in the “Untracked files” section in the `git status` output. Git won’t start including it in your commit snapshots until you explicitly tell it to do so. This is so you don’t accidentally include throw away files in commits. You do want to start including README, so start tracking it by adding it to the staging area by using the `git add` command, as shown below.

### Tracking New Files ###

To begin tracking a new file, use `git add`. So, to begin tracking the `README` file, run

	$ git add README

If you run `git status` again, `README` appears in a different section in the output.

	$ git status
	# On branch master
	# Changes to be committed:
	#   (use "git reset HEAD <file>..." to unstage)
	#
	#	new file:   README
	#

`README` is now under the “Changes to be committed” heading because it’s now staged. If you commit at this point, the version of the file at the time you ran `git add` is what will be in the snapshot.

You may recall that when you ran `git init` earlier, you then ran `git add (files)` — that was to begin tracking existing files in your directory. The path name given with the `git add` command can be either for a file or a directory. If it’s a directory, the command stages all the files in that directory recursively.

### Staging Modified Files ###

Let’s change a file that was already staged. If you change a previously staged file called `benchmarks.rb` and then run `git status` again, you’ll see something like

	$ git status
	# On branch master
	# Changes to be committed:
	#   (use "git reset HEAD <file>..." to unstage)
	#
	#	new file:   README
	#
	# Changes not staged for commit:
	#   (use "git add <file>..." to update what will be committed)
	#
	#	modified:   benchmarks.rb
	#

The `benchmarks.rb` file appears under a section named “Changes not staged for commit” — which means that a tracked file has been modified in the working directory but the latest version has not yet been staged. To stage it, run `git add` (it’s a multipurpose command — use it to begin tracking new files, to stage files, and to do other things that you’ll learn about later). Run `git add` now to stage `benchmarks.rb`, and then run `git status` again.

	$ git add benchmarks.rb
	$ git status
	# On branch master
	# Changes to be committed:
	#   (use "git reset HEAD <file>..." to unstage)
	#
	#	new file:   README
	#	modified:   benchmarks.rb
	#

Both files are staged and will go into your next commit. At this point, suppose you remember one little change that you want to make to `benchmarks.rb` before you commit it. You make that change, and you’re ready to commit. However, run `git status` one more time.

	$ vim benchmarks.rb
	$ git status
	# On branch master
	# Changes to be committed:
	#   (use "git reset HEAD <file>..." to unstage)
	#
	#	new file:   README
	#	modified:   benchmarks.rb
	#
	# Changes not staged for commit:
	#   (use "git add <file>..." to update what will be committed)
	#
	#	modified:   benchmarks.rb
	#

What the heck? Now `benchmarks.rb` is listed as both staged and unstaged. How is that possible? It turns out that Git is seeing two versions of `benchmarks.rb`. One is the version that you last staged when you ran `git add`. The other is the current version of `benchmarks.rb` in your working directory. If you commit now, the currently staged version of `benchmarks.rb` would go into the commit, not the version in your working directory. Remember, if you modify a file after you run `git add`, you have to run it again to stage the latest version.

	$ git add benchmarks.rb
	$ git status
	# On branch master
	# Changes to be committed:
	#   (use "git reset HEAD <file>..." to unstage)
	#
	#	new file:   README
	#	modified:   benchmarks.rb
	#

### Ignoring Files ###

Often, there will be a bunch of throw away files that you don’t want Git to automatically add or even show as being untracked. These are generally automatically generated files such as log files or files produced by your build system. To make Git ignore such files, create a file named `.gitignore` and put patterns in it that match the filenames you want Git to ignore.  Here’s an example `.gitignore` file.

	*.[oa]
	*~

The first line tells Git to ignore any files ending in `.o` or `.a` — *object* and *archive* files that may be a byproduct of building your code. The second line tells Git to ignore all files that end with a tilde (`~`), which is used by many text editors for temporary files. You may also include patterns that match `log`, `tmp`, or `pid` directories, automatically generated documentation, and so on. Setting up a `.gitignore` file before you get going is generally a good idea so you don’t accidentally commit files that you really don’t want in your Git repository.

The rules for the patterns that go in a `.gitignore` file are as follows:

*	Blank lines or lines starting with `#` are ignored.
*	Standard glob patterns work.
*	End patterns with a forward slash (`/`) to specify a directory.
*	Negate a pattern by starting it with an exclamation point (`!`).

Glob patterns are like simplified shell regular expressions. An asterisk (`*`) matches zero or more characters, `[abc]` matches any character inside the square brackets (in this case `a`, `b`, or `c`), a question mark (`?`) matches any single character, and square brackets enclosing characters separated by a hyphen (`[0-9]`) match any character in a range (in this case 0 through 9).

Here’s another example `.gitignore` file.

	# a comment - this is ignored
	# no .a files
	*.a
	# but do track lib.a, even though you're ignoring .a files above
	!lib.a
	# only ignore the root TODO file, not subdir/TODO
	/TODO
	# ignore all files in the build/ directory
	build/
	# ignore doc/notes.txt, but not doc/server/arch.txt
	doc/*.txt
	# ignore all .txt files in the doc/ directory
	doc/**/*.txt

A `**/` pattern is available in Git since version 1.8.2.

### Viewing Your Staged and Unstaged Changes ###

If the output of `git status` is too vague — you want to know exactly what you changed, not just which files were changed — use the `git diff` command. I’ll cover `git diff` in more detail later but you’ll probably use it most often to answer these two questions: What have you changed but not yet staged? And what have you staged that you’re about to commit? Although `git status` answers those questions very generally, `git diff` shows the exact lines added and removed — the patch, as it were.

Let’s say you edit and stage `README` again and then edit `benchmarks.rb` without staging it. If you run `git status`, once again you see something like

	$ git status
	# On branch master
	# Changes to be committed:
	#   (use "git reset HEAD <file>..." to unstage)
	#
	#	new file:   README
	#
	# Changes not staged for commit:
	#   (use "git add <file>..." to update what will be committed)
	#
	#	modified:   benchmarks.rb
	#

To see what you’ve changed but not yet staged, run `git diff` with no other arguments.

	$ git diff
	diff --git a/benchmarks.rb b/benchmarks.rb
	index 3cb747f..da65585 100644
	--- a/benchmarks.rb
	+++ b/benchmarks.rb
	@@ -36,6 +36,10 @@ def main
	           @commit.parents[0].parents[0].parents[0]
	         end

	+        run_code(x, 'commits 1') do
	+          git.commits.size
	+        end
	+
	         run_code(x, 'commits 2') do
	           log = git.commits('master', 15)
	           log.size

That command compares what’s in your working directory with what’s in your staging area. The result shows the changes you’ve made that you haven’t staged yet.

To see what you’ve staged that will go into your next commit, run `git diff --staged`. This command compares what you’ve staged to your last commit.

	$ git diff --staged
	diff --git a/README b/README
	new file mode 100644
	index 0000000..03902a1
	--- /dev/null
	+++ b/README2
	@@ -0,0 +1,5 @@
	+grit
	+ by Tom Preston-Werner, Chris Wanstrath
	+ http://github.com/mojombo/grit
	+
	+Grit is a Ruby library for extracting information from a Git repository

It’s important to note that `git diff` by itself doesn’t show all changes made since your last commit — only changes that are still unstaged. This can be confusing, because if you’ve staged all of your changes, `git diff` shows nothing.

For another example, if you stage `benchmarks.rb` and then edit it, `git diff` shows both the staged and unstaged changes.

	$ git add benchmarks.rb
	$ echo '# test line' >> benchmarks.rb
	$ git status
	# On branch master
	#
	# Changes to be committed:
	#
	#	modified:   benchmarks.rb
	#
	# Changes not staged for commit:
	#
	#	modified:   benchmarks.rb
	#

Now, run `git diff` to see what’s still unstaged.

	$ git diff
	diff --git a/benchmarks.rb b/benchmarks.rb
	index e445e28..86b2f7c 100644
	--- a/benchmarks.rb
	+++ b/benchmarks.rb
	@@ -127,3 +127,4 @@ end
	 main()

	 ##pp Grit::GitRuby.cache_client.stats
	+# test line

and `git diff --staged` to see what you’ve staged.

	$ git diff --staged
	diff --git a/benchmarks.rb b/benchmarks.rb
	index 3cb747f..e445e28 100644
	--- a/benchmarks.rb
	+++ b/benchmarks.rb
	@@ -36,6 +36,10 @@ def main
	          @commit.parents[0].parents[0].parents[0]
	        end

	+        run_code(x, 'commits 1') do
	+          git.commits.size
	+        end
	+
	        run_code(x, 'commits 2') do
	          log = git.commits('master', 15)
	          log.size

### Committing Your Changes ###

Now that your staging area contains what you want, commit your changes. Remember that anything that’s still unstaged — any files you’ve created or modified that you haven’t run `git add` on since you edited them — won’t go into this commit. They will remain as modified files.
In this case, the last time you ran `git status`, you saw that everything was staged, so you’re ready to commit your changes. The simplest way to commit is to run `git commit`.

	$ git commit

This launches your editor of choice. (This is set by your `$EDITOR` environment variable — usually vim or emacs, although you can configure it to be whatever you want using `git config --global core.editor`, as you saw in *Chapter 1*).

The editor displays the following text (this example is from Vim):

	# Please enter the commit message for your changes. Lines starting
	# with '#' will be ignored, and an empty message aborts the commit.
	# On branch master
	# Changes to be committed:
	#   (use "git reset HEAD <file>..." to unstage)
	#
	#       new file:   README
	#       modified:   benchmarks.rb
	~
	~
	~
	".git/COMMIT_EDITMSG" 10L, 283C

The default commit message contains an empty line on top followed by the commented out output of `git status`. You can remove these comments and type your commit message, or you can leave them in to help you remember what you’re committing. (For an even more detailed reminder of what you’ve modified, add the `-v` option to `git commit`. This also puts the `git diff` output in the editor so you can see exactly what you did). When you exit the editor, Git creates your commit with the commit message you entered but with the comments and diff stripped out.

Alternatively, include your commit message with `git commit` by adding an `-m` flag followed by the message.

	$ git commit -m "Story 182: Fix benchmarks for speed"
	[master]: created 463dc4f: "Fix benchmarks for speed"
	 2 files changed, 3 insertions(+), 0 deletions(-)
	 create mode 100644 README

You’ve created your first commit! You can see that the commit resulted in some status output: which branch you committed to (`master`), the SHA-1 hash of the commit (`463dc4f`), how many files were changed, and statistics about the number of lines inserted and deleted in the commit.

Remember that the commit records the snapshot from what’s in your staging area. Any changes you didn’t stage are still sitting in your working directory. You can stage the files and then do another commit to add them to your repository. Every time you perform a commit, you’re recording a snapshot of your project that you can revert to or compare to later.

### Skipping the Staging Step ###

Although the staging area can be amazingly useful for crafting commits exactly how you want them, having to run `git add` can be a bother if you want to stage all the modified files in your working directory. If you want to skip the separate staging step, Git provides a simple shortcut. Adding `-a` to `git commit` automatically stages every modified file before doing the commit, letting you skip the `git add` step.

	$ git status
	# On branch master
	#
	# Changes not staged for commit:
	#
	#	modified:   benchmarks.rb
	#
	$ git commit -a -m 'added new benchmarks'
	[master 83e38c7] added new benchmarks
	 1 file changed, 5 insertions(+), 0 deletions(-)

Notice how you didn’t have to run `git add benchmarks.rb` before you committed.

### Removing Files ###

To remove a file in your working directory that you haven’t staged, you can just remove it using the standard Unix `mv` command. The same is true for ignored files. However, to completely remove a file that you have staged from Git, you have to remove it from the staging area and then commit. The `git rm` command does that and also removes the file from your working directory so you don’t see it as an untracked file the next time you run `git status`.

If you simply remove the file from your working directory, it shows up in the `git status` output under the “Changes not staged for commit” area.

	$ rm grit.gemspec
	$ git status
	# On branch master
	#
	# Changes not staged for commit:
	#   (use "git add/rm <file>..." to update what will be committed)
	#
	#       deleted:    grit.gemspec
	#

Then, when you run `git rm`, Git stages the file’s removal.

	$ git rm grit.gemspec
	rm 'grit.gemspec'
	$ git status
	# On branch master
	#
	# Changes to be committed:
	#   (use "git reset HEAD <file>..." to unstage)
	#
	#       deleted:    grit.gemspec
	#

The next time you commit, the file will be gone and no longer tracked. If you modified and staged the file already, you must force the removal with the `-f` option. This is a safety feature to prevent accidental removal of files that haven’t been saved in a snapshot yet, which would prevent them from being recoverable.

Another useful thing to do is keeping the file in your working directory but removing it from your staging area. In other words, you may want Git to stop tracking it. This is particularly useful if you forgot to add something to your `.gitignore` file and accidentally staged it, like a large log file or a bunch of `.a` files. To do this, run `git rm --cached`.

	$ git rm --cached readme.txt

You can pass files, directories, and file-glob patterns to the `git rm` command. That means you can do things like

	$ git rm log/\*.log

Note the backslash (`\`) in front of the `*`. This is necessary because you want Git to do filename expansion instead of your shell. On Windows using the system console, omit the backslash. This command removes all files that have the `.log` extension in the `log/` directory. Or, you can do something like

	$ git rm \*~

which removes all files that end with `~`.

### Moving Files ###

Unlike many other VCSs, Git doesn’t explicitly track file movement. If you rename a file in Git, no metadata is stored in Git showing that you renamed the file. However, Git is pretty smart about figuring that out after the fact — I’ll deal with detecting file movement a bit later.

Thus, it’s a bit confusing that Git has a `mv` command. To rename a file in Git, run something like

	$ git mv file_from file_to

which works fine. In fact, if you do this and look at the `git status` output, you’ll notice that Git sees a renamed file:

	$ git mv README.txt README
	$ git status
	# On branch master
	# Your branch is ahead of 'origin/master' by 1 commit.
	#
	# Changes to be committed:
	#   (use "git reset HEAD <file>..." to unstage)
	#
	#       renamed:    README.txt -> README
	#

However, this is equivalent to running something like

	$ mv README.txt README
	$ git rm README.txt
	$ git add README

Git recognizes the rename automatically, so it doesn’t matter if you rename a file using `git mv` or with the Unix `mv` command. The only real difference is that `git mv` is one command instead of three so it’s more convenient.

## Viewing the Commit History ##

After you’ve done several commits, or if you’ve cloned a repository with an existing commit history, you’ll probably want to look back to see what’s been committed. The most basic and powerful way to do this is the `git log` command.

These examples use a very simple project called `simplegit` that I often use for demonstrations. To get the project, run

	git clone git://github.com/schacon/simplegit-progit.git

When you run `git log` in this project, you should get output that looks something like

	$ git log
	commit ca82a6dff817ec66f44342007202690a93763949
	Author: Scott Chacon <schacon@gee-mail.com>
	Date:   Mon Mar 17 21:52:11 2008 -0700

	    changed the version number

	commit 085bb3bcb608e1e8451d4b2432f8ecbe6306e7e7
	Author: Scott Chacon <schacon@gee-mail.com>
	Date:   Sat Mar 15 16:40:33 2008 -0700

	    removed unnecessary test code

	commit a11bef06a3f659402fe7563abf99ad00de2209e6
	Author: Scott Chacon <schacon@gee-mail.com>
	Date:   Sat Mar 15 10:31:28 2008 -0700

	    first commit

By default, with no arguments, `git log` lists commits in reverse chronological order. That is, the most recent commits show up first. As you can see, this command lists each commit with its SHA-1 hash, the author’s name and e-mail, the date of the commit, and the commit message.

The `git log` command has a huge number and variety of options to express exactly what you’re looking for and how to display it. Here are some of the most-used options.

One of the more helpful options is `-p` which shows the changes introduced in each commit. You can also use `-2` along with `-p`, which limits the output to only the last two entries.

	$ git log -p -2
	commit ca82a6dff817ec66f44342007202690a93763949
	Author: Scott Chacon <schacon@gee-mail.com>
	Date:   Mon Mar 17 21:52:11 2008 -0700

	    changed the version number

	diff --git a/Rakefile b/Rakefile
	index a874b73..8f94139 100644
	--- a/Rakefile
	+++ b/Rakefile
	@@ -5,5 +5,5 @@ require 'rake/gempackagetask'
	 spec = Gem::Specification.new do |s|
	     s.name      =   "simplegit"
	-    s.version   =   "0.1.0"
	+    s.version   =   "0.1.1"
	     s.author    =   "Scott Chacon"
	     s.email     =   "schacon@gee-mail.com

	commit 085bb3bcb608e1e8451d4b2432f8ecbe6306e7e7
	Author: Scott Chacon <schacon@gee-mail.com>
	Date:   Sat Mar 15 16:40:33 2008 -0700

	    removed unnecessary test code

	diff --git a/lib/simplegit.rb b/lib/simplegit.rb
	index a0a60ae..47c6340 100644
	--- a/lib/simplegit.rb
	+++ b/lib/simplegit.rb
	@@ -18,8 +18,3 @@ class SimpleGit
	     end

	 end
	-
	-if $0 == __FILE__
	-  git = SimpleGit.new
	-  puts git.show
	-end
	\ No newline at end of file

This displays the same information but with a diff directly following each entry. This is very helpful for code review or to quickly browse what happened in a series of commits that a collaborator added.

Sometimes it’s easier to review changes by word rather than by line. There’s a `--word-diff` option that you can append to the `git log -p` command to see changes this way. Word diff format is quite useless when applied to source code, but it comes in handy when used with large text files, like a book or a dissertation. Here’s an example.

	$ git log -U1 --word-diff
	commit ca82a6dff817ec66f44342007202690a93763949
	Author: Scott Chacon <schacon@gee-mail.com>
	Date:   Mon Mar 17 21:52:11 2008 -0700

	    changed the version number

	diff --git a/Rakefile b/Rakefile
	index a874b73..8f94139 100644
	--- a/Rakefile
	+++ b/Rakefile
	@@ -7,3 +7,3 @@ spec = Gem::Specification.new do |s|
	    s.name      =   "simplegit"
	    s.version   =   [-"0.1.0"-]{+"0.1.1"+}
	    s.author    =   "Scott Chacon"

As you can see, there are no added and removed lines in this output as in a normal diff. Changes are shown inline instead. The added word is enclosed in `{+ +}` and the removed word enclosed in `[- -]`. You may also want to reduce the usual three line context in diff output to only one line, since the context is now words, not lines. Do this with the `-U1` option, as in the example above.

You can also use a series of summarizing options with `git log`. For example, to see some abbreviated stats for each commit, use the `--stat` option.

	$ git log --stat
	commit ca82a6dff817ec66f44342007202690a93763949
	Author: Scott Chacon <schacon@gee-mail.com>
	Date:   Mon Mar 17 21:52:11 2008 -0700

	    changed the version number

	 Rakefile |    2 +-
	 1 file changed, 1 insertion(+), 1 deletion(-)

	commit 085bb3bcb608e1e8451d4b2432f8ecbe6306e7e7
	Author: Scott Chacon <schacon@gee-mail.com>
	Date:   Sat Mar 15 16:40:33 2008 -0700

	    removed unnecessary test code

	 lib/simplegit.rb |    5 -----
	 1 file changed, 0 insertions(+), 5 deletions(-)

	commit a11bef06a3f659402fe7563abf99ad00de2209e6
	Author: Scott Chacon <schacon@gee-mail.com>
	Date:   Sat Mar 15 10:31:28 2008 -0700

	    first commit

	 README           |    6 ++++++
	 Rakefile         |   23 +++++++++++++++++++++++
	 lib/simplegit.rb |   25 +++++++++++++++++++++++++
	 3 files changed, 54 insertions(+), 0 deletions(-)

As you can see, the `--stat` option includes a list of modified files below each commit entry, the number of files changed, and the number of lines in those files that were inserted and deleted. It also includes a summary of the information at the end.
Another really useful option is `--pretty`. This option changes the log output format to something other than the default. A few prebuilt options are available. The `oneline` option puts each commit on a single line, which is useful if you’re looking at a lot of commits. In addition, the `short`, `full`, and `fuller` options show the output in roughly the same format, but with less or more information, respectively.

	$ git log --pretty=oneline
	ca82a6dff817ec66f44342007202690a93763949 changed the version number
	085bb3bcb608e1e8451d4b2432f8ecbe6306e7e7 removed unnecessary test code
	a11bef06a3f659402fe7563abf99ad00de2209e6 first commit

The most interesting option is `format`, which allows you to specify your own log output format. This is especially useful when you’re generating output for a program to parse — because you specify the format explicitly, you know it won’t change with updates to Git.

	$ git log --pretty=format:"%h - %an, %ar : %s"
	ca82a6d - Scott Chacon, 11 months ago : changed the version number
	085bb3b - Scott Chacon, 11 months ago : removed unnecessary test code
	a11bef0 - Scott Chacon, 11 months ago : first commit

Table 2-1 lists some of the more useful options that format accepts.

<!-- Attention to translators: this is a table declaration.
The lines must be formatted as follows
<TAB><First column text><TAB><Second column text>
-->

	Option	Description of Output
	%H	Commit hash
	%h	Abbreviated commit hash
	%T	Tree hash
	%t	Abbreviated tree hash
	%P	Parent hashes
	%p	Abbreviated parent hashes
	%an	Author name
	%ae	Author e-mail
	%ad	Author date (format respects the --date= option)
	%ar	Author date, relative
	%cn	Committer name
	%ce	Committer email
	%cd	Committer date
	%cr	Committer date, relative
	%s	Subject

You may be wondering what the difference is between _author_ and _committer_. The _author_ is the person who originally wrote the patch, whereas the _committer_ is the person who last applied the patch. So, if you send in a patch to a project and one of the core members applies the patch, both of you get credit — you as the author and the core member as the committer. I’ll cover this distinction a bit more in *Chapter 5*.

The `oneline` and `format` options are particularly useful with the `--graph` option to `git log`. This adds a nice little ASCII graph showing your branch and merge history, which you can see in your copy of the Grit project repository.

	$ git log --pretty=format:"%h %s" --graph
	* 2d3acf9 ignore errors from SIGCHLD on trap
	*  5e3ee11 Merge branch 'master' of git://github.com/dustin/grit
	|\
	| * 420eac9 Added a method for getting the current branch.
	* | 30e367c timeout code and tests
	* | 5a09431 add timeout protection to grit
	* | e1193f8 support for heads with slashes in them
	|/
	* d6016bc require time for xmlschema
	*  11d191e Merge branch 'defunkt' into local

Those are only some simple output-formatting options to `git log` — there are many more. Table 2-2 lists the options I’ve covered so far and some other common formatting options, along with how they change the output of the `git log` command.

<!-- Attention to translators: this is a table declaration.
The lines must be formatted as follows
<TAB><First column text><TAB><Second column text>
-->

	Option	Description
	-p	Show the patch introduced with each commit.
	--word-diff	Show the patch in a word diff format.
	--stat	Show statistics for files modified in each commit.
	--shortstat	Display only the changed/insertions/deletions line from the --stat command.
	--name-only	Show the list of files modified after the commit information.
	--name-status	Show the list of files affected with added/modified/deleted information as well.
	--abbrev-commit	Show only the first few characters of the SHA-1 checksum instead of all 40.
	--relative-date	Display the date in a relative format (for example, “2 weeks ago”) instead of using the full date format.
	--graph	Display an ASCII graph of the branch and merge history beside the log output.
	--pretty	Show commits in an alternate format. Options include oneline, short, full, fuller, and format (where you specify your own format).
	--oneline	A convenience option short for `--pretty=oneline --abbrev-commit`.

### Limiting Log Output ###

In addition to output-formatting options, `git log` takes a number of useful limiting options — that is, options that only show a subset of commits. You’ve seen one such option already — the `-2` option, which shows only the last two commits. In fact, you can use `-<n>`, where `n` is any integer, to show the last `n` commits. In reality, you’re unlikely to need that because Git, by default, pipes all output through a pager so you see only one page of log output at a time.

However, the time-selection options such as `--since` and `--until` are very useful. For example, this command shows the commits made in the last two weeks.

	$ git log --since=2.weeks

This command accepts lots of different formats, such as a specific date (“2008-01-15”), or a relative date, such as “2 years 1 day 3 minutes ago”.

You can also filter the output to only include commits that match some search criteria. The `--author` option filters on a specific author, and the `--grep` option searches for keywords in commit messages. (Note that to specify both `--author` and `--grep` options, add `--all-match`, otherwise the command will match commits satisfying either option.)

The last really useful option to pass to `git log` as a filter is a path. If you specify a file name, the log output only shows commits that introduced a change to that file. This is always the last option and is generally preceded by double dashes (`--`) to separate the path(s) from the options.

Table 2-3 lists these and a few other common options.

<!-- Attention to translators: this is a table declaration.
The lines must be formatted as follows
<TAB><First column text><TAB><Second column text>
-->

	Option	Description
	-(n)	Show only the last n commits
	--since, --after	Limit the commits to those made after the specified date.
	--until, --before	Limit the commits to those made before the specified date.
	--author	Only show commits in which the author entry matches the specified string.
	--committer	Only show commits in which the committer entry matches the specified string.

For example, to see which commits modifying test files in the Git source code history were committed by Junio Hamano in the month of October 2008 and were not merges, run something like

	$ git log --pretty="%h - %s" --author=gitster --since="2008-10-01" \
	   --before="2008-11-01" --no-merges -- t/
	5610e3b - Fix testcase failure when extended attribute
	acd3b9e - Enhance hold_lock_file_for_{update,append}()
	f563754 - demonstrate breakage of detached checkout wi
	d1a43f2 - reset --hard/read-tree --reset -u: remove un
	51a94af - Fix "checkout --track -b newbranch" on detac
	b0ad11e - pull: allow "git pull origin $something:$cur

Of the nearly 20,000 commits in the Git source code history, this command shows the 6 that match those criteria.

### Using a GUI to Visualize History ###

If you like using a more graphical tool to visualize your commit history, take a look at the Tcl/Tk program `gitk` that’s distributed with Git. Gitk is basically a visual `git log` tool, and it accepts nearly the same filtering options that `git log` does. If you run `gitk` in your working directory, you should see something like Figure 2-2.

Insert 18333fig0202.png
Figure 2-2. The gitk history visualizer.

You can see the commit history in the top half of the window along with a nice ancestry graph. The diff viewer in the bottom half of the window shows the changes introduced in any commit you click.

## Undoing Things ##

You can change your mind at any point and revert a change that you’ve already made. I’ll review a few basic tools for doing so. Be careful, because you can’t always undo these undos. This is one of the few areas in Git where you may lose some work.

### Changing Your Last Commit ###

One of the common reasons for reverting a change is when you commit too early and possibly forget to add some files, or you mess up your commit message. To try that commit again, run

	$ git commit --amend

If you haven’t made any changes since your last commit (for instance, you run `git commit --amend` immediately after a commit), then the snapshot in your staging area will look exactly the same as when you made your last commit so all you’ll change is your commit message.

Your text editor starts up with the message from your previous commit already in its buffer. The message you create then replaces your previous commit message.

As an example, if you make a commit and then realize you forgot to stage a file you wanted to be in this commit, run

	$ git commit -m 'initial commit'
	$ git add forgotten_file
	$ git commit --amend

After these three commands, you end up with a single commit — the second commit replaces the first.

### Unstaging a Staged File ###

The output from `git status` also describes how to undo changes to the staging area and working directory. For example, let’s say you’ve changed two files and want to commit them as two separate changes, but you accidentally type `git add *` which stages them both. How can you unstage one of the two? The `git status` command reminds you.

	$ git add *
	$ git status
	# On branch master
	# Changes to be committed:
	#   (use "git reset HEAD <file>..." to unstage)
	#
	#       modified:   README.txt
	#       modified:   benchmarks.rb
	#

Right below the “Changes to be committed” text you see "use `git reset HEAD <file>...` to unstage". So, follow those directions to unstage `benchmarks.rb`.

	$ git reset HEAD benchmarks.rb
	benchmarks.rb: locally modified
	$ git status
	# On branch master
	# Changes to be committed:
	#   (use "git reset HEAD <file>..." to unstage)
	#
	#       modified:   README.txt
	#
	# Changes not staged for commit:
	#   (use "git add <file>..." to update what will be committed)
	#   (use "git checkout -- <file>..." to discard changes in working directory)
	#
	#       modified:   benchmarks.rb
	#

The output could be clearer, but `git reset` worked. The state of `benchmarks.rb` is back to what it was before you accidentally staged it.

### Unmodifying a Modified File ###

What if you realize that you don’t want to keep your changes to `benchmarks.rb`? How can you easily unmodify it — that is, revert it back to what it looked like when you last committed it? Luckily, `git status` tells you how to do that, too. In the last example, part of the output looks like

	# Changes not staged for commit:
	#   (use "git add <file>..." to update what will be committed)
	#   (use "git checkout -- <file>..." to discard changes in working directory)
	#
	#       modified:   benchmarks.rb
	#

This shows exactly how to discard the changes you’ve made. Do what it says.

	$ git checkout -- benchmarks.rb
	$ git status
	# On branch master
	# Changes to be committed:
	#   (use "git reset HEAD <file>..." to unstage)
	#
	#       modified:   README.txt
	#

The changes were reverted. You should also realize that this is a dangerous command: any changes you made to `benchmarks.rb` are gone — you just copied an older version over it. Don’t ever use this command unless you’re absolutely certain that you don’t want the modified file. To just get it out of the way, stashing and branching, covered in the next chapter, are generally better ways to go.

Remember, almost anything you commit in Git can be recovered. Even commits on branches that are deleted or commits that are overwritten with `git commit --amend` can be recovered (see *Chapter 9* for data recovery). However, a lost uncommitted change is gone for good.

## Working with Remotes ##

To collaborate on a Git project, you need to know how to manage remote repositories. Remote repositories are versions of a project on a host other than your own. You can work on several remote repositories, each of which you could have either read-only or read/write access to. Collaborating with other developers involves configuring your project to access these remote repositories, and pushing and pulling data to and from them when you need to share work.
This kind of configuration requires knowing how to add remote repositories, remove remotes that are no longer valid, manage various remote branches and define them as being tracked or not, and more. In this section, I’ll cover these skills.

### Showing Your Remotes ###

Rather than always referring to remote repositories by long URLs, Git lets you define shortnames that you can use in their place. When you clone a Git repository, the  shortname `origin` will be defined automatically to refer to the URL from which you cloned the repository. To see which remote repositories you have configured, run `git remote`. It lists the shortnames of each remote repository you’ve accessed.

	$ git clone git://github.com/schacon/ticgit.git
	Initialized empty Git repository in /private/tmp/ticgit/.git/
	remote: Counting objects: 595, done.
	remote: Compressing objects: 100% (269/269), done.
	remote: Total 595 (delta 255), reused 589 (delta 253)
	Receiving objects: 100% (595/595), 73.31 KiB | 1 KiB/s, done.
	Resolving deltas: 100% (255/255), done.
	$ cd ticgit
	$ git remote
	origin

You can also specify `-v`, which shows the URL that the shortname will be expanded to.

	$ git remote -v
	origin  git://github.com/schacon/ticgit.git (fetch)
	origin  git://github.com/schacon/ticgit.git (push)

If you have more than one remote, `git remote` lists them all. For example, my Grit repository looks something like

	$ cd grit
	$ git remote -v
	bakkdoor  git://github.com/bakkdoor/grit.git
	cho45     git://github.com/cho45/grit.git
	defunkt   git://github.com/defunkt/grit.git
	koke      git://github.com/koke/grit.git
	origin    git@github.com:mojombo/grit.git

This means I can easily pull contributions from any of these repositories. But notice that only `origin` is an SSH URL, so it’s the only one I can push to (I’ll cover why this is true in *Chapter 4*).

### Adding Remote Repositories ###

In previous sections I showed how cloning automatically adds remote repositories. You can also add remote repositories without cloning them. Here’s how. To add a new shortname that refers to a remote Git repository to make it easier to reference the remote, run `git remote add [shortname] [url]`.

	$ git remote
	origin
	$ git remote add pb git://github.com/paulboone/ticgit.git
	$ git remote -v
	origin	git://github.com/schacon/ticgit.git
	pb	git://github.com/paulboone/ticgit.git

Now you can use the shortname `pb` on the command line in lieu of the remote’s URL. For example, to fetch all the information that Paul has but that you don’t yet have in your repository, run `git fetch pb`.

	$ git fetch pb
	remote: Counting objects: 58, done.
	remote: Compressing objects: 100% (41/41), done.
	remote: Total 44 (delta 24), reused 1 (delta 0)
	Unpacking objects: 100% (44/44), done.
	From git://github.com/paulboone/ticgit
	 * [new branch]      master     -> pb/master
	 * [new branch]      ticgit     -> pb/ticgit

You can now access Paul’s `master` branch from your repository as `pb/master` — you can merge it into one of your branches, or you can simply look at it. (I’ll go over what branches are and how to use them in much more detail in *Chapter 3*.)

### Fetching and Pulling from Your Remotes ###

As you just saw, to get the contents of a remote repository, run

	$ git fetch [remote-name]

Git pulls whatever contents of that remote repository that don’t already exist in your local repository. After this, you’ll be able to see all the branches from that remote, which you can merge or inspect at any time. 

As I said above, when you clone a repository, Git automatically adds that remote repository using the shortname `origin`. So, `git fetch origin` fetches any new work that appears in that repository since you cloned (or last fetched from) it. It’s important to note that `git fetch` pulls the data into your local repository — it doesn’t automatically merge it with any of your existing work or modify what you’re currently working on. Any merging must take place as a separate step.

If you’ve run `git clone` to create your local repository, run `git pull` to automatically fetch and then merge from the repository you cloned from. This may be an easier or more comfortable way of doing things. This will also be covered in much more detail in *Chapter 3*.

### Pushing to Your Remotes ###

When your project is ready to be shared, push it back to where you originally cloned it from. The command for this is simple: `git push [remote-name] [branch-name]`. To push your `master` branch to your `origin` server, run

	$ git push origin master

Again, running `git clone` created the `origin` and `master` shortnames automatically.

This command works only if you cloned from a repository to which you have write access and if nobody has pushed to the same repository since you made your clone. This is important. If you and someone else clone at the same time and they push to `origin` and then you push to `origin`, your push will be rejected. You’ll have to pull down their work first and merge it into your repository before you’ll be allowed to push. See *Chapter 3* for more detailed information on how to push to remote servers.

### Inspecting a Remote ###

To see more information about a particular remote, run `git remote show [remote-name]`. For example, If you run this command with the shortname `origin`, you see

	$ git remote show origin
	* remote origin
	  URL: git://github.com/schacon/ticgit.git
	  Remote branch merged with 'git pull' while on branch master
	    master
	  Tracked remote branches
	    master
	    ticgit

This lists the URL for the remote repository as well as the tracked remote branch information. The command helpfully tells you that if you’re on the `master` branch and you run `git pull`, Git will automatically merge the master branch on the remote into the master branch in your local repository. The output also lists all the remote branches you’ve pulled already.

### Renaming and Removing Remotes ###

To rename the shortname of a remote repository run `git remote rename`. For instance, if you want to rename `pb` to `paul`, run

	$ git remote rename pb paul
	$ git remote
	origin
	paul

To remove a reference to a remote repository for some reason — perhaps the server no longer exists or a contributor isn’t participating anymore — run `git remote rm`.

	$ git remote rm paul
	$ git remote
	origin

## Tagging ##

Like most VCSs, Git has the ability to assign tags to specific points in your commit history. Generally, people do this to assign release names (e.g. `v1.0`). In this section, you’ll learn how to list tags, create new tags, and what the different types of tags are.

### Listing Your Tags ###

Listing tags is straightforward. Just run `git tag`.

	$ git tag
	v0.1
	v1.3

This lists the tags in alphabetical order.

You can also search for tags matching a particular pattern. The Git source repo, for instance, contains more than 240 tags. If you’re only interested in looking at the 1.4.2 series, run

	$ git tag -l 'v1.4.2.*'
	v1.4.2.1
	v1.4.2.2
	v1.4.2.3
	v1.4.2.4

### Creating Tags ###

Git implements two types of tags: lightweight and annotated. A lightweight tag is very much like a branch that doesn’t change — it’s just a pointer to a specific commit. Annotated tags, however, are stored almost like a commit in the Git repository. They’re checksummed, contain the tagger’s name, e-mail, and date, have a tagging message, and can be signed and verified with GNU Privacy Guard (GPG). It’s generally recommended that you create annotated tags so you can add all this information. But, if you want a temporary tag or for some reason don’t need all the information in an annotated tag, lightweight tags are perfectly acceptable.

### Lightweight Tags ###

A lightweight tag is basically a commit SHA-1 hash stored in a file containing nothing else. The name of the file is the tag name. To create a lightweight tag, don’t use any options with `git tag`.

	$ git tag v1.4-lw
	$ git tag
	v0.1
	v1.3
	v1.4
	v1.4-lw
	v1.5

If you run `git show` on the tag, you see the commit information.

	$ git show v1.4-lw
	commit 15027957951b64cf874c3557a0f3547bd83b3ff6
	Merge: 4a447f7... a6b4c97...
	Author: Scott Chacon <schacon@gee-mail.com>
	Date:   Sun Feb 8 19:02:46 2009 -0800

	    Merge branch 'experiment'

### Annotated Tags ###

Another way to tag commits is with an annotated tag, which allows you to include information that doesn't appear in lightweight tags. The first thing is to include a message about the commit in the tag. Specify the `-a` and `-m` options to create an annotated tag.

	$ git tag -a v1.4 -m 'my version 1.4'
	$ git tag
	v0.1
	v1.3
	v1.4

The `-a` specifies the tag name and the `-m` specifies a message. Both are stored with the tag. If you don’t specify a message for an annotated tag, Git launches your editor so you can enter the message.

You can see the tag data along with the corresponding commit information by running the `git show` command.

	$ git show v1.4
	tag v1.4
	Tagger: Scott Chacon <schacon@gee-mail.com>
	Date:   Mon Feb 9 14:45:11 2009 -0800

	my version 1.4
	commit 15027957951b64cf874c3557a0f3547bd83b3ff6
	Merge: 4a447f7... a6b4c97...
	Author: Scott Chacon <schacon@gee-mail.com>
	Date:   Sun Feb 8 19:02:46 2009 -0800

	    Merge branch 'experiment'

That shows the tagger information, the date the commit was tagged, and the tag message before showing the commit information.

### Signed Tags ###

You can also sign your tags with GPG, assuming you have a private signing key. All you have to do is use `-s` instead of `-a`.

	$ git tag -s v1.5 -m 'my signed 1.5 tag'
	You need a passphrase to unlock the secret key for
	user: "Scott Chacon <schacon@gee-mail.com>"
	1024-bit DSA key, ID F721C45A, created 2009-02-09

If you run `git show` on that tag, you see your GPG signature attached to it.

	$ git show v1.5
	tag v1.5
	Tagger: Scott Chacon <schacon@gee-mail.com>
	Date:   Mon Feb 9 15:22:20 2009 -0800

	my signed 1.5 tag
	-----BEGIN PGP SIGNATURE-----
	Version: GnuPG v1.4.8 (Darwin)

	iEYEABECAAYFAkmQurIACgkQON3DxfchxFr5cACeIMN+ZxLKggJQf0QYiQBwgySN
	Ki0An2JeAVUCAiJ7Ox6ZEtK+NvZAj82/
	=WryJ
	-----END PGP SIGNATURE-----
	commit 15027957951b64cf874c3557a0f3547bd83b3ff6
	Merge: 4a447f7... a6b4c97...
	Author: Scott Chacon <schacon@gee-mail.com>
	Date:   Sun Feb 8 19:02:46 2009 -0800

	    Merge branch 'experiment'

### Verifying Tags ###

To verify a signed tag, run `git tag -v [tag-name]`. This uses GPG to verify the signature. You need the signer’s public key in your keyring for this to work properly.

	$ git tag -v v1.4.2.1
	object 883653babd8ee7ea23e6a5c392bb739348b1eb61
	type commit
	tag v1.4.2.1
	tagger Junio C Hamano <junkio@cox.net> 1158138501 -0700

	GIT 1.4.2.1

	Minor fixes since 1.4.2, including git-mv and git-http with alternates.
	gpg: Signature made Wed Sep 13 02:08:25 2006 PDT using DSA key ID F3119B9A
	gpg: Good signature from "Junio C Hamano <junkio@cox.net>"
	gpg:                 aka "[jpeg image of size 1513]"
	Primary key fingerprint: 3565 2A26 2040 E066 C9A7  4A7D C0C6 D9A4 F311 9B9A

If you don’t have the signer’s public key, you see something like this instead.

	gpg: Signature made Wed Sep 13 02:08:25 2006 PDT using DSA key ID F3119B9A
	gpg: Can't check signature: public key not found
	error: could not verify the tag 'v1.4.2.1'

### Tagging Later ###

You can also tag commits you made in the past. Suppose your commit history looks like

	$ git log --pretty=oneline
	15027957951b64cf874c3557a0f3547bd83b3ff6 Merge branch 'experiment'
	a6b4c97498bd301d84096da251c98a07c7723e65 beginning write support
	0d52aaab4479697da7686c15f77a3d64d9165190 one more thing
	6d52a271eda8725415634dd79daabbc4d9b6008e Merge branch 'experiment'
	0b7434d86859cc7b8c3d5e1dddfed66ff742fcbc added a commit function
	4682c3261057305bdd616e23b64b0857d832627b added a todo file
	166ae0c4d3f420721acbb115cc33848dfcc2121a started write support
	9fceb02d0ae598e95dc970b74767f19372d61af8 updated rakefile
	964f16d36dfccde844893cac5b347e7b3d44abbc commit the todo
	8a5cbc430f1a9c3d00faaeffd07798508422908a updated readme

Now, suppose you forgot to assign a `v1.2` tag, which should point to the "updated rakefile" commit. You can add it after the fact. To tag that commit, specify the commit SHA-1 hash (or part of it) at the end of the `git tag` command.

	$ git tag -a v1.2 -m 'version 1.2' 9fceb02

You can see that you created the tag and the commit you tagged.

	$ git tag
	v0.1
	v1.2
	v1.3
	v1.4
	v1.4-lw
	v1.5

	$ git show v1.2
	tag v1.2
	Tagger: Scott Chacon <schacon@gee-mail.com>
	Date:   Mon Feb 9 15:32:16 2009 -0800

	version 1.2
	commit 9fceb02d0ae598e95dc970b74767f19372d61af8
	Author: Magnus Chacon <mchacon@gee-mail.com>
	Date:   Sun Apr 27 20:43:35 2008 -0700

	    updated rakefile
	...

### Sharing Tags ###

By default, `git push` doesn’t transfer tags to remote servers. If you do want to transfer tags you have to do this explicitly, which is just like sharing remote branches — run `git push origin [tagname]`.

	$ git push origin v1.5
	Counting objects: 50, done.
	Compressing objects: 100% (38/38), done.
	Writing objects: 100% (44/44), 4.56 KiB, done.
	Total 44 (delta 18), reused 8 (delta 1)
	To git@github.com:schacon/simplegit.git
	* [new tag]         v1.5 -> v1.5

If you have a lot of tags to push at once, use the `--tags` option to `git push`.  This transfers all your tags that aren’t already on the remote server.

	$ git push origin --tags
	Counting objects: 50, done.
	Compressing objects: 100% (38/38), done.
	Writing objects: 100% (44/44), 4.56 KiB, done.
	Total 44 (delta 18), reused 8 (delta 1)
	To git@github.com:schacon/simplegit.git
	 * [new tag]         v0.1 -> v0.1
	 * [new tag]         v1.2 -> v1.2
	 * [new tag]         v1.4 -> v1.4
	 * [new tag]         v1.4-lw -> v1.4-lw
	 * [new tag]         v1.5 -> v1.5

Now, when someone else clones or pulls from the remote repository, they get all your tags as well.

## Tips and Tricks ##

Before I finish this chapter on basic Git, I want to mention a few little tips and tricks that may make your Git experience a bit more pleasant. Many people use Git without using any of these tips, and I won’t refer to them later in the book, but you should know about them.

### Auto-Completion ###

If you use the Bash shell, Git comes with a nice auto-completion script you can enable. Download the Git source code, and look in the `contrib/completion` directory. There should be a file there called `git-completion.bash`. Copy this file to your home directory, and add this to your `.bashrc` file.

	source ~/.git-completion.bash

To set up Git so that all users automatically use this script, copy it to the `/opt/local/etc/bash_completion.d` directory on Mac systems or to the `/etc/bash_completion.d/` directory on Linux systems. This is a directory of scripts that Bash automatically loads to provide auto-completion.

If you’re using Windows with Git Bash, which is the default when installing Git on Windows with msysGit, auto-completion is preconfigured.

Press the Tab key when you’re entering a Git command, and it should display a set of suggestions for you to pick from.

	$ git co<tab><tab>
	commit config

In this case, typing `git co` and then pressing the Tab key twice suggests commit and config. Adding `m<tab>` selects `git commit` automatically.

This also works with command options, which is probably more useful. For instance, if you’re trying to run `git log` and can’t remember one of the options, start typing the command and then press the Tab key to see what matches.

	$ git log --s<tab>
	--shortstat  --since=  --src-prefix=  --stat   --summary

That’s a pretty nice trick and may save you some time.

### Git Aliases ###

Git doesn’t guess a command if you only partially enter it. If you don’t want to type the entire text of each of the Git commands you commonly use, you can easily set up an alias for the commands by running `git config`. Here are a couple of examples.

	$ git config --global alias.co checkout
	$ git config --global alias.br branch
	$ git config --global alias.ci commit
	$ git config --global alias.st status

So, for example, instead of typing `git commit`, just type `git ci`. As you continue using Git, you’ll probably use other commands frequently as well so don’t hesitate to create new aliases.

This technique can also be very useful in creating commands that you think should exist but don’t. For example, to correct the usability problem you encountered with unstaging a file, add your own unstage alias.

	$ git config --global alias.unstage 'reset HEAD --'

This makes the following two commands equivalent:

	$ git unstage fileA
	$ git reset HEAD fileA

The first seems a bit clearer. It’s also common to add a `git last` command, like this.

	$ git config --global alias.last 'log -1 HEAD'

This way, you can easily see the last commit.

	$ git last
	commit 66938dae3329c7aebe598c2246a8e6af90d04646
	Author: Josh Goebel <dreamer3@example.com>
	Date:   Tue Aug 26 19:48:51 2008 +0800

	    test for current head

	    Signed-off-by: Scott Chacon <schacon@example.com>

As you can tell, Git simply replaces what you type with the alias. However, maybe you want to run a Linux command rather than a Git command. In that case, start the alias string with a `!` character. This is useful if you write your own tools that work with Git. I alias `git visual` to run `gitk`.

	$ git config --global alias.visual '!gitk'

## Summary ##

At this point, you know how to do all the basic local Git operations — creating or cloning a repository, making changes, staging and committing those changes, and viewing the history of all the changes contained in a repository. Next, we’ll cover Git’s killer feature: its branching model.
