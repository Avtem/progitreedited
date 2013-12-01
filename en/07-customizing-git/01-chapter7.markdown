# Customizing Git #

So far, I’ve covered the basics of how Git works and how to use it, and I’ve introduced a number of Git tools that help you use it efficiently. In this chapter, I’ll go through some steps that you can perform to make Git operate in a more personalized fashion. I’ll introduce several important configuration settings and the hooks system. With these, it’s easy to get Git to work exactly the way you, your company, or your group needs it to.

## Git Configuration ##

As you briefly saw in the Chapter 1, you modify Git configuration settings with the `git config` command. One of the first things you did was to set your name and e-mail address.

	$ git config --global user.name "John Doe"
	$ git config --global user.email johndoe@example.com

Now you’ll learn a few of the more interesting settings.

You saw some simple Git configuration details in Chapter 1, but I’ll go over them again quickly here. Git uses a collection of configuration files for storing settings. The first place Git looks for these settings is in the `/etc/gitconfig` file, which contains settings that apply to every user on the system and in all repositories. If you run `git config --system`, it reads and writes from this file.

The next place Git looks is the `~/.gitconfig` file, which is user specific. Git reads and writes to this file when you run `git config --global`.

Finally, Git looks for configuration settings in the `.git/config` file in the repository you’re currently using. These settings are specific to that single repository.

Each level overwrites settings in the previous level, so settings in `.git/config` trump those in `/etc/gitconfig`, for instance. You can also set these settings manually by editing the file and inserting the correct setting, but it’s generally easier to run `git config`.

### Basic Client Configuration ###

The configuration settings recognized by Git fall into two categories: client side and server side. The majority of the settings are client side — configuring only your preferences. Although tons of settings are available, I’ll only cover the few that either are commonly used or can significantly affect your workflow. Many settings are useful only in edge cases that I won’t go over. To see a list of all the settings the version of Git you’re running recognizes, run

	$ git config --help

The manual page for `git config` lists all the available settings in quite a bit of detail.

#### core.editor ####

By default, Git uses whatever you’ve set in your shell EDITOR variable or else falls back to the Vi editor to create and edit commit and tag messages. To modify that default, use the `core.editor` setting.

	$ git config --global core.editor emacs

Now, no matter what’s set in your shell EDITOR variable, Git will fire up Emacs to edit messages.

#### commit.template ####

If you set commit.template to the path of a file on your system, Git will use the contents of that file as the default message when you commit. For instance, suppose you create the template file `$HOME/.gitmessage.txt` that looks like

	subject line

	what happened

	[ticket: X]

To tell Git to use the contents of this file as the default message that appears in your editor when you run `git commit`, set the `commit.template` configuration setting.

	$ git config --global commit.template $HOME/.gitmessage.txt
	$ git commit

Then, your editor buffer will look something like this when you run `git commit`.

	subject line

	what happened

	[ticket: X]
	# Please enter the commit message for your changes. Lines starting
	# with '#' will be ignored, and an empty message aborts the commit.
	# On branch master
	# Changes to be committed:
	#   (use "git reset HEAD <file>..." to unstage)
	#
	# modified:   lib/test.rb
	#
	~
	~
	".git/COMMIT_EDITMSG" 14L, 297C

If you have a commit-message policy in place, then creating a template that satisfies that policy and configuring Git to use it by default can help increase the chances of that policy being followed.

#### core.pager ####

The core.pager setting determines what pager Git uses to page output from commands such as `git log` and `git diff`. You can set it to `more` or to your favorite pager (the default is `less`), or you can turn it off by setting it to a blank string.

	$ git config --global core.pager ''

If you do that, Git won’t page the output of any command, no matter how long the output is.

#### user.signingkey ####

If you’re making signed annotated tags (as discussed in Chapter 2), setting your GPG signing key as a configuration setting makes things easier. Set your GPG signing key like so.

	$ git config --global user.signingkey <gpg-key-id>

Now you can sign tags without having to specify your key every time when you run `git tag`.

	$ git tag -s <tag-name>

#### core.excludesfile ####

Put patterns in your project’s `.gitignore` file so that Git ignores untracked files that match the patterns, as discussed in Chapter 2. However, if you want to place these patterns in a file outside of your project, tell Git the location of that file with the `core.excludesfile` setting. Simply set it to the path of a file with content similar to what’s in `.gitignore`.

#### help.autocorrect ####

If you mistype a command in Git, you see something like

	$ git com
	git: 'com' is not a git-command. See 'git --help'.

	Did you mean this?
	     commit

If you set `help.autocorrect` to 1, Git will automatically run the suggested command, but only if there’s just one matched possibility.

### Colors in Git ###

Git can color what it sends to your screen, which can make it easier to understand the output. A number of settings can help you set the colors to your liking.

#### color.ui ####

Git automatically colors most of its output. You can get very specific about what you want colored and how. To turn on default coloring, set `color.ui` to true.

	$ git config --global color.ui true

With that setting, Git colors its output if the output goes to a screen. Other possible settings are false, which never colors the output, and always, which colors all the time, even if you’re redirecting Git commands to a file or piping them to another command.

You’ll rarely set `color.ui = always`. In most scenarios, if you want color in your redirected output, instead pass the `--color` flag to the Git command to force it to use colors. The `color.ui = true` setting is almost always what you’ll want.

#### `color.*` ####

To be more specific about which commands produce colors and how, Git contains verb-specific color settings. Each of these can be set to `true`, `false`, or `always`.

	color.branch
	color.diff
	color.interactive
	color.status

In addition, each of these has subsettings to set specific colors for parts of the output. For example, to set the meta information in diff output to blue foreground, black background, and bold text, run

	$ git config --global color.diff.meta "blue black bold"

You can set the color to any of the following values: `normal`, `black`, `red`, `green`, `yellow`, `blue`, `magenta`, `cyan`, or `white`. If you want an attribute like `bold` in the previous example, you can choose from `bold`, `dim`, `ul`, `blink`, and `reverse`.

See the `git config` manpage for all the subsettings you can configure.

### External Merge and Diff Tools ###

Although you’ve been using Git’s internal implementation of diff, you can use an external tool instead. You can also use a graphical merge conflict-resolution tool instead of having to resolve conflicts manually. I’ll demonstrate setting up the Perforce Visual Merge Tool (P4Merge) to do your diffs and merge resolutions because it’s a nice graphical tool, it’s free, and it works on all major platforms. I’ll use Mac and Linux path names in the examples.

Download P4Merge here.

	http://www.perforce.com/perforce/downloads/component.html

To begin, create external wrapper scripts to run your commands. I’ll use the Mac path for the executable. On other systems, it will be where the `p4merge` binary is installed. Create a merge wrapper script named `extMerge` that calls `p4merge` with all the necessary arguments.

	$ cat /usr/local/bin/extMerge
	#!/bin/sh
	/Applications/p4merge.app/Contents/MacOS/p4merge $*

The diff wrapper checks to make sure seven arguments are provided and passes two of them to your merge script. By default, Git passes the following arguments to the diff program:

	path old-file old-hex old-mode new-file new-hex new-mode

Because you only want the `old-file` and `new-file` arguments, the wrapper script only passes these arguments.

	$ cat /usr/local/bin/extDiff
	#!/bin/sh
	[ $# -eq 7 ] && /usr/local/bin/extMerge "$2" "$5"

Make sure these tools are executable.

	$ sudo chmod +x /usr/local/bin/extMerge
	$ sudo chmod +x /usr/local/bin/extDiff

Now set up your config file so that Git uses your custom merge and diff tools. Doing so requires a number of custom settings: `merge.tool` to tell Git what strategy to use, `mergetool.*.cmd` to specify how to run the command, `mergetool.trustExitCode` to tell Git if the exit code of the program indicates a successful merge resolution, and `diff.external` to tell Git what command to run to do diffs. So, either run the following four config commands:

	$ git config --global merge.tool extMerge
	$ git config --global mergetool.extMerge.cmd \
	    'extMerge "$BASE" "$LOCAL" "$REMOTE" "$MERGED"'
	$ git config --global mergetool.trustExitCode false
	$ git config --global diff.external extDiff

or add these lines to `~/.gitconfig`:

	[merge]
	  tool = extMerge
	[mergetool "extMerge"]
	  cmd = extMerge \"$BASE\" \"$LOCAL\" \"$REMOTE\" \"$MERGED\"
	  trustExitCode = false
	[diff]
	  external = extDiff

After all this is done, if you run diff commands such as

	$ git diff 32d1776b1^ 32d1776b1

instead of producing the diff output on the command line, Git fires up P4Merge, which looks something like Figure 7-1.

Insert 18333fig0701.png
Figure 7-1. P4Merge.

If you try to merge two branches and subsequently have conflicts, run `git mergetool`. It starts P4Merge to the conflicts.

The nice thing about this wrapper setup is that you can change your diff and merge tools easily. For example, to change your `extDiff` and `extMerge` tools to run the KDiff3 tool instead, all you have to do is edit your `extMerge` file.

	$ cat /usr/local/bin/extMerge
	#!/bin/sh
	/Applications/kdiff3.app/Contents/MacOS/kdiff3 $*

Now, Git will use the KDiff3 tool for diff viewing and merge conflict resolution.

Git comes preset to use a number of merge-resolution tools without you having to do any special configuration. You can set your merge tool to kdiff3, opendiff, tkdiff, meld, xxdiff, emerge, vimdiff, or gvimdiff. If you’re not interested in using kdiff3 for diff but rather want to use it just for merge resolution, and the kdiff3 command is in your path, then run

	$ git config --global merge.tool kdiff3

If you run this instead of setting up the `extMerge` and `extDiff` files, Git will use kdiff3 for merge resolution and the normal Git diff tool for diffs.

### Formatting and Whitespace ###

Formatting and whitespace issues are some of the more frustrating and subtle problems that many developers encounter when collaborating, especially when using multiple platforms. It’s very easy for patches or other collaborative work to include subtle text editor introduced whitespace changes or conflicting line-ending markers. Git has a few configuration settings to help with these issues.

#### core.autocrlf ####

If any of the people collaborating on a project are using Windows, you’ll probably run into line-ending issues at some point. This is because Windows uses both a carriage-return character (CR) and a linefeed character (LF) at the end of lines in files, whereas Mac and Linux systems use only LF. This is a subtle but incredibly annoying fact of cross-platform work.

Git can handle this by automatically converting CRLF line endings into LF when you commit, and vice versa when it checks out code into your working directory. You can turn on this ability with the `core.autocrlf` setting. If you’re on a Windows machine, set it to `true`. This converts LF endings into CRLF when you check out code.

	$ git config --global core.autocrlf true

If you’re on a Linux or Mac system that uses LF line endings, then you don’t want Git to automatically do any conversion when you check out files; however, if a file with CRLF endings accidentally gets introduced, then you may want Git to fix it. You can tell Git to convert CRLF to LF on commit but not the other way around by setting `core.autocrlf` to input.

	$ git config --global core.autocrlf input

This setup results in CRLF endings in Windows checkouts but LF endings on Mac and Linux systems and in the  Git repository.

If you’re a Windows programmer doing a Windows-only project, then you can turn off conversions all together, saving CRLF line endings in the Git repository, by setting the config setting to `false`.

	$ git config --global core.autocrlf false

#### core.whitespace ####

Git comes preset to detect and fix four primary whitespace issues — two are enabled by default and can be turned off, and two aren’t enabled by default but can be turned on.

The two that are turned on by default are `trailing-space`, which looks for spaces at the end of lines, and `space-before-tab`, which looks for spaces before tabs at the beginning of lines.

The two that are disabled by default but can be turned on are `indent-with-non-tab`, which looks for lines that begin with eight or more spaces instead of tabs, and `cr-at-eol`, which tells Git that carriage returns at the end of lines are allowed.

Tell Git which of these you want enabled by setting `core.whitespace` to the settings you want turned on or off, separated by commas. Disable settings by either leaving them out of the setting string or prepending a `-` in front of the setting. For example, if you want all but `cr-at-eol` to be set, run

	$ git config --global core.whitespace \
	    trailing-space,space-before-tab,indent-with-non-tab

Git will detect these issues when you run `git diff` and try to warn you by coloring them to give you a chance to fix them before you commit. It will also use these settings to help when you apply patches with `git apply`. When you’re applying patches, you can tell Git to warn you if it’s applying patches with the specified whitespace issues.

	$ git apply --whitespace=warn <patch>

Or, you can have Git try to automatically fix the issue before applying the patch.

	$ git apply --whitespace=fix <patch>

These options apply to `git rebase` as well. If you have commits with whitespace issues but haven’t yet pushed upstream, run `git rebase --whitespace=fix` to have Git automatically fix whitespace issues as it’s rewriting the patches.

### Server Configuration ###

Not nearly as many configuration settings are available for the server side of Git, but there are a few interesting ones to take note of.

#### receive.fsckObjects ####

Although Git can make sure each object still matches its SHA-1 hash and points to valid objects during a push, it doesn’t do that by default. This is a relatively expensive operation and may add a lot of time to each push, depending on the size of the repository. For Git to check object consistency on every push, set `receive.fsckObjects` to true:

	$ git config --system receive.fsckObjects true

Now, Git will check the integrity of your repository before each push is accepted to make sure faulty clients aren’t sending corrupt data.

#### receive.denyNonFastForwards ####

If you rebase commits that you’ve already pushed and then try to push again, or otherwise try to push a commit to a remote branch that doesn’t contain the commit that the remote branch currently points to, the push will be denied. This is generally good policy but in the case of the rebase, you may determine that you know what you’re doing and can force-update the remote branch with a `-f` flag to your `git push` command.

To disable the ability to force-update remote branches to non-fast-forward references, set `receive.denyNonFastForwards` to true:

	$ git config --system receive.denyNonFastForwards true

The other way to do this is via server-side receive hooks, which I’ll cover in a bit. That approach lets you do more complex things, like deny non-fast-forwards to a certain subset of users.

#### receive.denyDeletes ####

One of the workarounds to the `denyNonFastForwards` policy is for the user to delete the branch and then push it back with the new reference. Setting `receive.denyDeletes` to true

	$ git config --system receive.denyDeletes true

denies branch and tag deletion over a push across the board — no user can do it. To remove remote branches, you must remove the ref files from the server manually. There are also more interesting ways to do this on a per-user basis via ACLs, as you’ll learn at the end of this chapter.

## Git Attributes ##

Rather than applying to everything in a Git repository, some of these settings can also be specified for a specific path, so that Git applies those settings only for a subdirectory or subset of files. These path-specific settings are called Git attributes and appear either in a `.gitattributes` file in the root of your project or in the `.git/info/attributes` file if you don’t want the attributes file committed with your project.

Using attributes, you can do things like specify separate merge strategies for individual files or directories, tell Git how to diff non-text files, or have Git filter content before you check it into or out. In this section, you’ll learn about some of the attributes you can set and see a few examples of using this feature in practice.

### Binary Files ###

One cool trick for using Git attributes is telling Git which files are binary (in case Git can’t figure it out) and giving Git special instructions for handling those files. For instance, some text files may be machine generated and not diffable, whereas some binary files can be diffed. You’ll see how to tell Git which is which.

#### Identifying Binary Files ####

Some files look like text files but for all intents and purposes should be treated as binary. For instance, Xcode projects on the Mac contain a file whose name ends in `.pbxproj`, which is basically a JSON (plain text javascript data format) file created by the Xcode IDE to record your build settings and other values. Although it’s technically a text file, because it only contains ASCII characters, you don’t want to treat it as such because it’s really a lightweight database. You can’t merge the contents if two people makes changes to their own copies, and diffs generally aren’t helpful. The file is meant to be consumed by a machine. In essence, you want to treat it like a binary file.

To tell Git to treat all `pbxproj` files as binary data, add the following line to your `.gitattributes` file:

	*.pbxproj -crlf -diff

Now Git won’t try to convert or fix CRLF issues. Nor will it produce a diff showing changes in `.pbxproj` files when you run `git show` or `git diff`. You can also use the built-in macro `binary` that means `-crlf -diff`.

	*.pbxproj binary

#### Diffing Binary Files ####

You can use Git attributes to meaningfully diff binary files. You do this by telling Git how to convert your binary data into a text format that can be compared using a normal diff tool.

##### MS Word files #####

One of the most annoying problems known to humanity is version-controlling Microsoft Word documents. Everyone knows that Word is the most horrific word processor around but, oddly, everyone uses it. To version-control Word documents, you can stick them in a Git repository and commit every once in a while. But, what good does that do? If you run `git diff` normally, you only see something like

	$ git diff
	diff --git a/chapter1.doc b/chapter1.doc
	index 88839c4..4afcb7c 100644
	Binary files a/chapter1.doc and b/chapter1.doc differ

You can’t directly compare two Word files unless you check them out and scan them manually, right? It turns out there's an easy way to do this — using Git attributes. Put the following line in your `.gitattributes` file:

	*.doc diff=word

This tells Git that any file whose name matches this pattern (.doc) should use the "word" filter when to display a diff. What is the "word" filter? You have to create it yourself. Here you configure Git to use the `strings` program to convert Word documents into readable text files, which it can then diff properly.

	$ git config diff.word.textconv strings

This command adds a section to your `.git/config` file that looks like

	[diff "word"]
		textconv = strings

Side note: There are different kinds of `.doc` files. Some use an UTF-16 encoding or other "codepages" and `strings` won’t find anything useful in there. Your mileage may vary.

Now Git knows that if it tries to do diff two snapshots, and any of the files in the snapshots end in `.doc`, it should run those files through the "word" filter, which is defined as the `strings` program. This effectively makes nice text-based versions of your Word files before attempting to diff them. Because this is a pretty cool and not widely known feature, I’ll go over a few examples.

Here’s one example. I put a MS Word version of Chapter 1 of this book into Git, added some text to a paragraph, and saved the document. Then, I ran `git diff` to see what changed.

	$ git diff
	diff --git a/chapter1.doc b/chapter1.doc
	index c1c8a0a..b93c9e4 100644
	--- a/chapter1.doc
	+++ b/chapter1.doc
	@@ -8,7 +8,8 @@ re going to cover Version Control Systems (VCS) and Git basics
	 re going to cover how to get it and set it up for the first time if you don
	 t already have it on your system.
	 In Chapter Two we will go over basic Git usage - how to use Git for the 80%
	-s going on, modify stuff and contribute changes. If the book spontaneously
	+s going on, modify stuff and contribute changes. If the book spontaneously
	+Let's see if this works.

Git successfully and succinctly told me that I added the string "Let’s see if this works", which is correct. This solution isn’t perfect — it adds a bunch of random stuff at the end — but it certainly works. If you can find or write a Word-to-plain-text converter that works better, then the Git community will be grateful. However, `strings` is available on most Mac and Linux systems, so this approach may be a good first try.

##### OpenDocument Text files #####

The same approach that I used for MS Word files (`*.doc`) can be used for OpenDocument Text files (`*.odt`) created by OpenOffice.org.

Add the following line to your `.gitattributes` file:

	*.odt diff=odt

Now, add the `odt` diff filter in `.git/config`.

	[diff "odt"]
		binary = true
		textconv = /usr/local/bin/odt-to-txt

OpenDocument files are actually zipped directories containing multiple files (the content in XML format, stylesheets, images, etc.). You’ll need to write a script to extract the content as plain text. Create the file `/usr/local/bin/odt-to-txt` (it can go in any directory) with the following content:

	#! /usr/bin/env perl
	# Simplistic OpenDocument Text (.odt) to plain text converter.
	# Author: Philipp Kempgen

	if (! defined($ARGV[0])) {
		print STDERR "No filename given!\n";
		print STDERR "Usage: $0 filename\n";
		exit 1;
	}

	my $content = '';
	open my $fh, '-|', 'unzip', '-qq', '-p', $ARGV[0], 'content.xml' or die $!;
	{
		local $/ = undef;  # slurp mode
		$content = <$fh>;
	}
	close $fh;
	$_ = $content;
	s/<text:span\b[^>]*>//g;           # remove spans
	s/<text:h\b[^>]*>/\n\n*****  /g;   # headers
	s/<text:list-item\b[^>]*>\s*<text:p\b[^>]*>/\n    --  /g;  # list items
	s/<text:list\b[^>]*>/\n\n/g;       # lists
	s/<text:p\b[^>]*>/\n  /g;          # paragraphs
	s/<[^>]+>//g;                      # remove all XML tags
	s/\n{2,}/\n\n/g;                   # remove multiple blank lines
	s/\A\n+//;                         # remove leading blank lines
	print "\n", $_, "\n\n";

And make it executable.

	chmod +x /usr/local/bin/odt-to-txt

Now `git diff` will be able to tell you what changed in `.odt` files.


##### Image files #####

Another interesting problem you can solve this way involves diffing image files. One way to diff PNG format files is to run them through a filter that extracts their EXIF information — metadata that’s recorded with most image formats. If you install the `exiftool` program, you can use it to see the metadata in text format, so at least the diff will show a textual representation of any changes.

	$ echo '*.png diff=exif' >> .gitattributes
	$ git config diff.exif.textconv exiftool

If you replace an image in your project and run `git diff`, you see something like

	diff --git a/image.png b/image.png
	index 88839c4..4afcb7c 100644
	--- a/image.png
	+++ b/image.png
	@@ -1,12 +1,12 @@
	 ExifTool Version Number         : 7.74
	-File Size                       : 70 kB
	-File Modification Date/Time     : 2009:04:17 10:12:35-07:00
	+File Size                       : 94 kB
	+File Modification Date/Time     : 2009:04:21 07:02:43-07:00
	 File Type                       : PNG
	 MIME Type                       : image/png
	-Image Width                     : 1058
	-Image Height                    : 889
	+Image Width                     : 1056
	+Image Height                    : 827
	 Bit Depth                       : 8
	 Color Type                      : RGB with Alpha

You can easily see that the file size and image dimensions have both changed.

### Keyword Expansion ###

SVN- or CVS-style keyword expansion is an often requested feature. The main problem with this in Git is that you can’t modify a file with information about a commit after you’ve committed, because Git checksums the file first. However, you can inject text into a file when it’s checked out and remove it again before it’s staged. Git attributes offer two ways to do this.

First, you can inject the SHA-1 hash of a blob into an `$Id$` field in the file automatically. If you set this attribute on a file, then the next time you check out the file, Git will replace that field with the SHA-1 hash of the blob. It’s important to notice that it isn’t the SHA-1 hash of the commit, but of the blob itself.

	$ echo '*.txt ident' >> .gitattributes
	$ echo '$Id$' > test.txt

The next time you check out this file, Git injects the SHA-1 hash of the blob.

	$ rm test.txt
	$ git checkout -- test.txt
	$ cat test.txt
	$Id: 42812b7653c7b88933f8a9d6cad0ca16714b9bb3 $

However, that result is of limited use. If you’ve used keyword substitution in CVS or Subversion, you can include a datestamp. The SHA-1 hash isn’t all that helpful, because it’s completely random so you can’t tell if one SHA-1 hash is older or newer than another.

It turns out that you can write your own filters for doing substitutions in files when they’re committed or checked out. These are the "clean" and "smudge" filters. In the `.gitattributes` file, set a filter for particular paths and then create scripts that process files just before they’re checked out ("smudge", see Figure 7-2) or just before they’re committed ("clean", see Figure 7-3). These filters can be set up to do all sorts of fun things.

Insert 18333fig0702.png
Figure 7-2. The “smudge” filter is run on checkout.

Insert 18333fig0703.png
Figure 7-3. The “clean” filter is run when files are staged.

The original commit message for this feature shows a simple example of running all your C source code through the `indent` program before committing. You can set it up by setting the filter attribute in `.gitattributes` to filter `*.c` files with the "indent" filter.

	*.c     filter=indent

Then, tell Git what the "indent" filter does on smudge and clean.

	$ git config --global filter.indent.clean indent
	$ git config --global filter.indent.smudge cat

In this case, when you commit files whose name matches `*.c`, Git will run them through the indent program before it commits them and then run them through the `cat` program before it checks them back out. The `cat` program is basically a no-op: it spits out the same data that it gets in. This combination effectively filters all C source code files through `indent` before committing.

Another interesting example does `$Date$` keyword expansion, RCS style. To do this properly, you need a small script that takes a filename, figures out the last commit date for this project containing the file, and then inserts the date into the file. Here’s a small Ruby script that does that.

	#! /usr/bin/env ruby
	data = STDIN.read
	last_date = `git log --pretty=format:"%ad" -1`
	puts data.gsub('$Date$', '$Date: ' + last_date.to_s + '$')

The script gets the latest commit date from the `git log` command, sticks the date into any `$Date$` string it finds in stdin, and prints the results. This should be simple to do in whatever language you’re most comfortable in. You can name this script `expand_date` and put it in your path. Now, you need to set up a filter (call it `dater`) and tell Git to use your `expand_date` filter to smudge the files on checkout. You’ll use a Perl expression to clean that up on commit.

	$ git config filter.dater.smudge expand_date
	$ git config filter.dater.clean 'perl -pe "s/\\\$Date[^\\\$]*\\\$/\\\$Date\\\$/"'

This Perl snippet strips out anything it sees in a `$Date$` string to get back to where you started. Now that your filter is ready, test it by creating a file containing a `$Date$` keyword and then setting up a Git attribute for that file that invokes the new filter.

	$ echo '# $Date$' > date_test.txt
	$ echo 'date*.txt filter=dater' >> .gitattributes

If you commit those changes and check out the file again, you see the keyword properly substituted.

	$ git add date_test.txt .gitattributes
	$ git commit -m "Testing date expansion in Git"
	$ rm date_test.txt
	$ git checkout date_test.txt
	$ cat date_test.txt
	# $Date: Tue Apr 21 07:26:52 2009 -0700$

You can see how powerful this technique can be for customized applications. You have to be careful, though, because the `.gitattributes` file is committed and passed around with the project but the driver (in this case, `dater`) isn’t. So, this won’t work everywhere. When you design these filters, they should be able to fail without damaging the project.

### Exporting Your Repository ###

Git attributes also allow some interesting things to happen when exporting an archive of your project.

#### export-ignore ####

You can tell Git not to export certain files or directories when generating an archive. If there’s anything that you don’t want to include in an archive but that you do want checked into your project, configure the `export-ignore` attribute to bypass what you don’t want included.

For example, say you have some test files in a `test/` subdirectory, and it doesn’t make sense to include them in the export of your project. Add the following line to your Git attributes file:

	test/ export-ignore

Now, when you run `git archive` to create a tarball of your project, that directory won’t be included in the archive.

#### export-subst ####

You can also do some simple keyword substitution in your archives. Git lets you put the string `$Format:$` in any file with any of the `--pretty=format` formatting shortcodes, many of which you saw in Chapter 2. For instance, to include a file named `LAST_COMMIT` containing a project’s last commit date, with the last commit date automatically injected into it when you run `git archive`, set up `LAST_COMMIT` to look like

	$ echo 'Last commit date: $Format:%cd$' > LAST_COMMIT
	$ echo "LAST_COMMIT export-subst" >> .gitattributes
	$ git add LAST_COMMIT .gitattributes
	$ git commit -am 'adding LAST_COMMIT file for archives'

When you run `git archive`, the contents of `LAST_COMMIT` in the archive file look like

	$ cat LAST_COMMIT
	Last commit date: $Format:Tue Apr 21 08:38:48 2009 -0700$

### Merge Strategies ###

You can also use Git attributes to select different merge strategies for specific files in your project. One very useful setting is for Git to not try to merge specific files when they have conflicts, but rather to use your side of the merge over someone else’s.

This is helpful if a branch in your project has diverged, but you want to be able to merge changes back in from it, ignoring certain files. Say you have a database settings file called `database.xml` that’s different in two branches, and you want to merge in your other branch without messing up the file. Set up an attribute like

	database.xml merge=ours

If you merge the other branch, instead of having merge conflicts with `database.xml`, you see something like

	$ git merge topic
	Auto-merging database.xml
	Merge made by recursive.

In this case, `database.xml` remains unchanged.

## Git Hooks ##

Like many other VCSs, Git can fire off custom scripts when certain important actions occur. These are called hooks. There are two groups of hooks: client side and server side. The client-side hooks are for client operations such as committing and merging. The server-side hooks are for server operations such as receiving pushed commits. You can use these hooks for all sorts of purposes. You’ll learn about a few of them here.

### Installing a Hook ###

The hooks are all stored in the `.git/hooks` subdirectory. By default, Git populates this directory with a bunch of example scripts, many of which are useful by themselves. But the examples also document the input values of each script. Most of the examples are written as shell scripts, but there’s some Perl thrown in. Any properly named executable script will work fine — you can write them in Ruby, Python, or what have you. The example hook files end with `.sample` — you’ll need to rename them in order for them to be executed. I’ll cover most of the major hook filenames here.

### Client-Side Hooks ###

There are a lot of client-side hooks. This section splits them into committing-workflow scripts, e-mail-workflow scripts, and everything else.

#### Committing-Workflow Hooks ####

The first four hooks have to do with the commit process. Git runs the `pre-commit` hook first, before you even enter a commit message. This hook can be used to inspect the snapshot that’s about to be committed to see if you’ve forgotten something, to make sure tests have run, or to examine whatever you need to inspect in the code. A non-zero exit status from this hook aborts the commit, although you can override this check with the `--no-verify` option to `git commit`. Some other things you can do are check for code style (run lint or something equivalent), check for trailing whitespace (the default hook does exactly that), or check for appropriate documentation on new methods.

Git runs the `prepare-commit-msg` hook before the commit message editor is fired up but after the default message is created. It lets you edit the default message before the committer sees it. This hook takes a few options: the path to the file that holds the commit message so far, the type of commit, and the commit SHA-1 hash if this is an amended commit. This hook generally isn’t useful for normal commits. Rather, it’s good for commits where the default message is auto-generated, such as commit messages generated from templates, merge commits, squashed commits, and amended commits. You may use it in conjunction with a commit template to programmatically insert text into the commit message.

The `commit-msg` hook takes one parameter, which is the path to a temporary file that contains the current commit message. If this script exits with a non-zero value, Git aborts the commit process. This hook is useful for validating your project state or commit message before allowing a commit to take place. In the last section of this chapter, I’ll demonstrate using this hook to check that your commit message conforms to a required pattern.

After the entire commit process is completed, the `post-commit` hook runs. It doesn’t take any parameters, but you can easily get the SHA-1 hash of the last commit by running `git log -1 HEAD`. Generally, this script is used for notifications or something similar.

The committing-workflow client-side scripts can be used in just about any workflow. They’re often used to enforce certain policies, although it’s important to note that these scripts aren’t transferred during a clone. You can enforce a policy on the server side to reject pushes of commits that don’t conform to some policy, but it’s entirely up to the developer to use these scripts on the client side. These scripts help developers, and they must be set up and maintained by them, although the scripts can be overridden or modified at any time.

#### E-mail Workflow Hooks ####

You can set up three client-side hooks for an e-mail-based workflow. They’re all invoked by the `git am` command, so if you aren’t using that command in your workflow, you can safely skip to the next section. If you accept patches over e-mail prepared by `git format-patch`, then some of these hooks may be helpful.

The first hook that Git runs run is `applypatch-msg`. It takes a single argument — the name of the temporary file that contains the proposed commit message. Git aborts the patch if this script exits with a non-zero value. Use this hook to make sure a commit message is properly formatted or to modify the message by having the script edit it in place.

The next hook that runs when applying patches via `git am` is `pre-applypatch`. It takes no arguments and is run after the patch is applied, so you can use it to inspect the snapshot before committing. You can run tests or otherwise inspect the working directory with this script. If something is missing or the tests don’t pass, exiting with a non-zero value also aborts the `git am` command without committing the patch.

The last hook that runs during a `git am` operation is `post-applypatch`. You can use it to notify a group or the author of the patch that you’ve applied the patch. You can’t stop the patching process with this script.

#### Other Client Hooks ####

The `pre-rebase` hook runs before you rebase anything and can halt the process by exiting with a non-zero value. Use this hook to disallow rebasing any commits that have already been pushed. The example `pre-rebase` hook that Git installs does this, although it assumes that `next` is the name of the branch you publish. You’ll likely need to change that to the name of your stable published branch.

After you run a successful `git checkout`, the `post-checkout` hook runs. Use it to set up your working directory properly for your project environment. This may mean moving in large binary files that you don’t want source controlled, auto-generating documentation, or something along those lines.

Finally, the `post-merge` hook runs after a successful `git merge` command. Use it to restore data in the working tree that Git can’t track, such as permissions data. This hook can likewise validate files out of Git control that you may want copied in when the working tree changes.

### Server-Side Hooks ###

In addition to the client-side hooks, you can use a couple of important server-side hooks to enforce nearly any kind of policy. These scripts run before and after pushes to the server. The pre hooks can exit with non-zero values at any time to reject the push as well as send an error message back to the client. You can set up a push policy that’s as complex as you wish.

#### pre-receive and post-receive ####

The first script that runs when handling a push from a client is `pre-receive`. It takes a list from stdin of the references that are being pushed. If it exits with a non-zero value, none of the pushes are accepted. Use this hook to do things like make sure none of the updated references are non-fast-forwards or to check that the user doing the pushing has create, delete, or push access.

The `post-receive` hook runs after the entire push process is complete and can be used to update other services or notify users. It takes the same data from stdin as the `pre-receive` hook. Examples include e-mailing a list, notifying a continuous integration server, or updating a ticket-tracking system. You can even parse the commit messages to see if any tickets need to be opened, modified, or closed. This script can’t stop the push process, but the client doesn’t disconnect until the push has completed. So be careful using this hook to do anything that may take a long time.

#### update ####

The `update` script is very similar to the `pre-receive` script, except that `update` is run once for each branch the push is trying to update. If the push goes to multiple branches, `pre-receive` runs only once, whereas `update` runs once for each branch being pushed to. Instead of reading from stdin, this script takes three arguments — the name of the reference (branch), the SHA-1 hash that reference pointed to before the push, and the SHA-1 hash the user is trying to push. If the update script exits with a non-zero value, only that reference is rejected. Other references can still be updated.

## An Example Git-Enforced Policy ##

In this section, you’ll use what you’ve learned to establish a Git workflow that checks for a prescribed commit message format, enforces fast-forward-only pushes, and allows only certain users to modify certain subdirectories in a project. You’ll build client scripts that tell the developer if their push will be rejected, and server scripts that actually enforce the policies.

I used Ruby to write the scripts, both because it’s my preferred scripting language and because I feel it’s the most pseudocode-looking of the scripting languages. Thus, you should be able to roughly follow the code even if you don’t use Ruby. However, any language will work fine. All the sample hook scripts distributed with Git are written in either Perl or Bash, so you can also see plenty of examples of hooks in those languages by looking at the samples.

### Server-Side Hook ###

All the server-side work will go into the `update` script in your hooks directory. This script runs once per branch being pushed and takes the reference being pushed to, the old revision where that branch was, and the new revision being pushed. You can also retrieve the username of the user doing the pushing if the push is being sent over SSH. If you’ve allowed everyone to connect as the same user (like "git") via public-key authentication, you may have to create a shell wrapper for that user that determines which remote user is connecting based on the public key, and set an environment variable containing that username. Here I assume the name of the connecting user is in the `$USER` environment variable, so the `update` script begins by gathering all the information it needs.

	#!/usr/bin/env ruby

	$refname = ARGV[0]
	$oldrev  = ARGV[1]
	$newrev  = ARGV[2]
	$user    = ENV['USER']

	puts "Enforcing Policies... \n(#{$refname}) (#{$oldrev[0,6]}) (#{$newrev[0,6]})"

Yes, I’m using global variables. Don’t judge me — it’s easier to demonstrate in this manner.

#### Enforcing a Specific Commit-Message Format ####

Your first challenge is to enforce the policy that each commit message must adhere to a particular format. As a sample specification, assume that each message has to include a string that looks like "ref: 1234" because you want each commit to link to a specific ticket in your ticketing system. You must look at each commit being pushed, check if a string in that format is in the commit message, and, if not, exit with a non-zero value so the push is rejected.

You can get a list of the SHA-1 hashes of all the commits that are being pushed by passing the value of `$newrev` and `$oldrev` to a Git plumbing command called `git rev-list`. This is similar to the `git log` command, but by default it outputs only SHA-1 hashes and nothing else. So, to get a list of all the commit SHA-1 hashes introduced between one commit’s SHA-1 hash and another, run something like

	$ git rev-list 538c33..d14fc7
	d14fc7c847ab946ec39590d87783c69b031bdfb7
	9f585da4401b0a3999e84113824d15245c13f0be
	234071a1be950e2a8d078e6141f5cd20c1e61ad3
	dfa04c9ef3d5197182f13fb5b9b1fb7717d2222a
	17716ec0f1ff5c77eff40b7fe912f9f6cfd0e475

Take that output, loop through each of those commit SHA-1 hashes, grab the commit message for each commit, and test that message against a regular expression that looks for the required pattern.

Next, get the commit message from each of these commits in order to test them. To get the complete commit data, use the `git cat-file` plumbing command. I’ll go over all these plumbing commands in detail in Chapter 9 but for now, here’s what that command gives you.

	$ git cat-file commit ca82a6
	tree cfda3bf379e4f8dba8717dee55aab78aef7f4daf
	parent 085bb3bcb608e1e8451d4b2432f8ecbe6306e7e7
	author Scott Chacon <schacon@gmail.com> 1205815931 -0700
	committer Scott Chacon <schacon@gmail.com> 1240030591 -0700

	changed the version number

A simple way to get the commit message when you have the commit’s SHA-1 hash is to go to the first blank line and take everything after that. The `sed` command on Unix systems can do this.

	$ git cat-file commit ca82a6 | sed '1,/^$/d'
	changed the version number

Use that incantation to select the commit message from each commit in the push and exit if anything doesn’t match. To exit the script and reject the push, exit with a non-zero value. The whole method looks like this:

	$regex = /\[ref: (\d+)\]/

	# enforced custom commit message format
	def check_message_format
	  missed_revs = `git rev-list #{$oldrev}..#{$newrev}`.split("\n")
	  missed_revs.each do |rev|
	    message = `git cat-file commit #{rev} | sed '1,/^$/d'`
	    if !$regex.match(message)
	      puts "[POLICY] Your message is not formatted correctly"
	      exit 1
	    end
	  end
	end
	check_message_format

Putting that in your `update` script will reject updates that contain commits with messages that don’t adhere to your rule.

#### Enforcing a User-Based ACL System ####

Suppose you want to add a mechanism that uses an access control list (ACL) that specifies which users are allowed to push changes to certain parts of your projects. Some users have full access, and others only have access to push changes to certain subdirectories or specific files. To implement this, put those rules in the file `~/.git/acl`. The `update` hook looks at those rules, sees what files are in all the commits being pushed, and determines whether the user doing the push has permission to update all those files.

The first thing to do is create the ACL. Here you’ll use a format very much like the CVS ACL syntax. It uses a series of lines, where the first field is `avail` or `unavail`, the next field is a comma-delimited list of the users to which the rule applies, and the last field is the path to which the rule applies (blank means open access). All of these fields are delimited by a vertical bar (`|`) character.

In this case, there are a couple of administrators, some documentation writers with access to the `doc` directory, and one developer who only has access to the `lib` and `tests` directories. Your ACL file looks like

	avail|nickh,pjhyett,defunkt,tpw
	avail|usinclair,cdickens,ebronte|doc
	avail|schacon|lib
	avail|schacon|tests

Begin by reading this data into a structure. In this case, to keep the example simple, I only enforce the `avail` directives. Here’s a method that contains an associative array where the key is the user name and the value is an array of paths to which the user has write access.

	def get_acl_access_data(acl_file)
	  # read in ACL data
	  acl_file = File.read(acl_file).split("\n").reject { |line| line == '' }
	  access = {}
	  acl_file.each do |line|
	    avail, users, path = line.split('|')
	    next unless avail == 'avail'
	    users.split(',').each do |user|
	      access[user] ||= []
	      access[user] << path
	    end
	  end
	  access
	end

In the ACL file you looked at earlier, the `get_acl_access_data` method returns a data structure that looks like

	{"defunkt"=>[nil],
	 "tpw"=>[nil],
	 "nickh"=>[nil],
	 "pjhyett"=>[nil],
	 "schacon"=>["lib", "tests"],
	 "cdickens"=>["doc"],
	 "usinclair"=>["doc"],
	 "ebronte"=>["doc"]}

Now that you have the permissions sorted out, determine what paths the commits being pushed modify to make sure the user doing the pushing has access to all of them.

You can pretty easily see what files have been modified in a single commit with the `--name-only` option to the `git log` command (mentioned briefly in Chapter 2).

	$ git log -1 --name-only --pretty=format:'' 9f585d

	README
	lib/test.rb

If you use the ACL structure returned from the `get_acl_access_data` method and check it against the files listed in each of the commits, you can determine whether the user has access to push all of their commits.

	# only allows certain users to modify certain subdirectories in a project
	def check_directory_perms
	  access = get_acl_access_data('acl')

	  # see if anyone is trying to push something they can't
	  new_commits = `git rev-list #{$oldrev}..#{$newrev}`.split("\n")
	  new_commits.each do |rev|
	    files_modified = `git log -1 --name-only --pretty=format:'' #{rev}`.split("\n")
	    files_modified.each do |path|
	      next if path.size == 0
	      has_file_access = false
	      access[$user].each do |access_path|
	        if !access_path || # user has access to everything
	          (path.index(access_path) == 0) # access to this path
	          has_file_access = true
	        end
	      end
	      if !has_file_access
	        puts "[POLICY] You do not have access to push to #{path}"
	        exit 1
	      end
	    end
	  end
	end

	check_directory_perms

Most of that should be easy to follow. Get a list of new commits being pushed to your server with `git rev-list`. Then, for each of those commits, find which files are modified and make sure the user who’s doing the push has access to all the paths being modified. One Rubyism that may not be clear is `path.index(access_path) == 0`, which is true if path begins with `access_path`. This ensures that `access_path` is not just in one of the allowed paths, but appears at the beginning of each accessed path.

Now your users can’t push any commits with badly formed messages or with modified files outside of their designated paths.

#### Enforcing Fast-Forward-Only Pushes ####

The only thing left is to set the `receive.denyDeletes` and `receive.denyNonFastForwards` settings to enforce fast-forward-only pushes. Enforcing this with a hook will also work, and you can modify the hook to apply only for certain users or whatever else you come up with later.

The logic for this script is to see if any commits are reachable from the older revision that aren’t reachable from the newer one. If there are none, then it was a fast-forward push. Otherwise, deny the push.

	# enforces fast-forward only pushes
	def check_fast_forward
	  missed_refs = `git rev-list #{$newrev}..#{$oldrev}`
	  missed_ref_count = missed_refs.split("\n").size
	  if missed_ref_count > 0
	    puts "[POLICY] Cannot push a non fast-forward reference"
	    exit 1
	  end
	end

	check_fast_forward

Everything is ready. If you run `chmod u+x .git/hooks/update`, which is the file which should contain all this code, and then try to push a non-fast-forward reference, you’ll get something like

	$ git push -f origin master
	Counting objects: 5, done.
	Compressing objects: 100% (3/3), done.
	Writing objects: 100% (3/3), 323 bytes, done.
	Total 3 (delta 1), reused 0 (delta 0)
	Unpacking objects: 100% (3/3), done.
	Enforcing Policies...
	(refs/heads/master) (8338c5) (c5b616)
	[POLICY] Cannot push a non fast-forward reference
	error: hooks/update exited with error code 1
	error: hook declined to update refs/heads/master
	To git@gitserver:project.git
	 ! [remote rejected] master -> master (hook declined)
	error: failed to push some refs to 'git@gitserver:project.git'

There are a couple of interesting things here. When the hook starts running you see

	Enforcing Policies...
	(refs/heads/master) (8338c5) (c5b616)

You sent that to stdout at the very beginning of your update script. It’s important to note that anything your script sends to stdout will be received by the client.

The next thing you’ll notice is the error message.

	[POLICY] Cannot push a non fast-forward reference
	error: hooks/update exited with error code 1
	error: hook declined to update refs/heads/master

The first line was printed by your script. The other two lines are from Git saying that the update script exited with a non-zero value, and that’s what canceled the push. Lastly, you’ll see

	To git@gitserver:project.git
	 ! [remote rejected] master -> master (hook declined)
	error: failed to push some refs to 'git@gitserver:project.git'

You’ll see a remote rejected message for each reference that your hook declined, saying that the push was declined specifically because of a hook failure.

Furthermore, if the ref marker doesn’t appear in any of your commits, you’ll see an error message.

	[POLICY] Your message is not formatted correctly

Or, if someone tries to edit a file they don’t have access to and then push a commit containing the file, the result will look similar. For instance, if a documentation author tries to push a commit modifying something in the `lib` directory, they see

	[POLICY] You do not have access to push to lib/test.rb

That’s all. From now on, as long as that `update` script is executable, your repository will never be rewound, will never contain a commit message that doesn't conform to your requirements, and your users will be unable to change files to which they’ve been refused access.

### Client-Side Hooks ###

The downside to this approach is the whining that will inevitably result when users’ pushes are rejected. Having carefully crafted work rejected at the last minute can be extremely frustrating and confusing. Furthermore, they will have to edit their commit history to correct the violation, which isn’t always for the faint of heart.

The answer to this dilemma is to provide client-side hooks that users can use to be notified when they do something that the server is likely to reject. That way, they can correct any problems before committing and before those issues become more difficult to fix. Because hooks aren’t transferred when a project is cloned, you must distribute these scripts some other way and then have your users copy them to their `.git/hooks` directory and make them executable. You can distribute these hooks within the project or in a separate project, but there’s no way to set them up automatically.

To begin, check your commit message just before each commit is recorded, so you know the server won’t reject your change due to a badly formatted commit message. To do this, add the `commit-msg` hook. It reads the message from the file passed as the first argument and compares the contents of the file that to the required pattern. Git aborts the commit if there’s no match.

	#!/usr/bin/env ruby
	message_file = ARGV[0]
	message = File.read(message_file)

	$regex = /\[ref: (\d+)\]/

	if !$regex.match(message)
	  puts "[POLICY] Your message is not formatted correctly"
	  exit 1
	end

If that script is in place (in `.git/hooks/commit-msg`) and executable, and you commit with a message that isn’t properly formatted, you see

	$ git commit -am 'test'
	[POLICY] Your message is not formatted correctly

No commit was completed. However, if your message contains the proper pattern, Git allows the commit.

	$ git commit -am 'test [ref: 132]'
	[master e05c914] test [ref: 132]
	 1 file changed, 1 insertion(+), 0 deletions(-)

Next, make sure you aren’t modifying files that are outside your ACL scope. If your project’s `.git` directory contains a copy of the ACL file you used previously, then the following `pre-commit` script will enforce this constraint:

	#!/usr/bin/env ruby

	$user    = ENV['USER']

	# [ insert acl_access_data method from above ]

	# only allows certain users to modify certain subdirectories in a project
	def check_directory_perms
	  access = get_acl_access_data('.git/acl')

	  files_modified = `git diff-index --cached --name-only HEAD`.split("\n")
	  files_modified.each do |path|
	    next if path.size == 0
	    has_file_access = false
	    access[$user].each do |access_path|
	    if !access_path || (path.index(access_path) == 0)
	      has_file_access = true
	    end
	    if !has_file_access
	      puts "[POLICY] You do not have access to push to #{path}"
	      exit 1
	    end
	  end
	end

	check_directory_perms

This is roughly the same script as the server-side version, but with two important differences. First, the ACL file is in a different place because this script runs from your working directory, not from your Git directory. You have to change the path to the ACL file from this

	access = get_acl_access_data('acl')

to this.

	access = get_acl_access_data('.git/acl')

The other important difference is the way you get a listing of the files that have changed. Because the server-side method looks at the log of commits, and, at this point, the commit hasn’t been recorded yet, you must get your file listing from the staging area instead. Instead of

	files_modified = `git log -1 --name-only --pretty=format:'' #{ref}`

use

	files_modified = `git diff-index --cached --name-only HEAD`

But, those are the only differences. Otherwise, the script works the same way. One caveat is that the script expects you to be running locally as the same user you connect as on the remote machine. If that’s not true, set the `$user` variable manually.

The last thing to do is check that you’re not trying to push non-fast-forwarded references, but that’s a bit less common. To get a non-fast-forward reference, you either have to rebase past a commit you’ve already pushed or try pushing a different local branch to the same remote branch.

Because the server will tell you that you can’t push a non-fast-forward reference anyway, and the hook prevents forced pushes, the only thing you can try to catch is accidental rebasing commits that have already been pushed.

Here’s an example pre-rebase script that checks for that. It gets a list of all the commits you’re about to rewrite and checks whether they exist in any of your remote references. If it sees one that’s reachable from one of your remote references, it aborts the rebase.

	#!/usr/bin/env ruby

	base_branch = ARGV[0]
	if ARGV[1]
	  topic_branch = ARGV[1]
	else
	  topic_branch = "HEAD"
	end

	target_shas = `git rev-list #{base_branch}..#{topic_branch}`.split("\n")
	remote_refs = `git branch -r`.split("\n").map { |r| r.strip }

	target_shas.each do |sha|
	  remote_refs.each do |remote_ref|
	    shas_pushed = `git rev-list ^#{sha}^@ refs/remotes/#{remote_ref}`
	    if shas_pushed.split("\n").include?(sha)
	      puts "[POLICY] Commit #{sha} has already been pushed to #{remote_ref}"
	      exit 1
	    end
	  end
	endG
This script uses a syntax that wasn’t covered in the Revision Selection section of Chapter 6. Get a list of commits that have already been pushed by running

	git rev-list ^#{sha}^@ refs/remotes/#{remote_ref}

The `SHA^@` syntax resolves to all the parents of that commit. You’re looking for any commit that’s reachable from the last commit on the remote that isn’t reachable from any parent of any of the SHA-1 hashes you’re trying to push — meaning it’s a fast-forward commit.

The main drawback to this approach is that it can be very slow and is often unnecessary. If you don’t try to force the push with `-f`, the server will warn you and not accept the push anyway. However, it’s an interesting exercise and can, in theory, help avoid a rebase that you might later have to go back to fix.

## Summary ##

I’ve covered most of the major ways you can customize your Git client and server to best fit your workflows. You’ve learned about all sorts of configuration settings, file-based attributes, and event hooks, and you’ve built an example policy-enforcing script. You should now be able to make Git fit nearly any workflow you can dream up.
