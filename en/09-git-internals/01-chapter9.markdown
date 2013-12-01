# Git Internals #

You may have skipped to this chapter from a previous chapter, or maybe you came here after reading the rest of the book. In either case, this is where I’ll go over the inner workings and implementation details of Git. I found that learning this information is critical for understanding how to use Git, but others have argued that this information can be confusing and unnecessarily complex for beginners. Thus, I’ve made this discussion the last chapter in the book so you could read it at any point in your learning process. It’s up to you to decide.

Now that you’re here, let’s get started. First, if it isn’t yet clear, Git is fundamentally a content-addressable filesystem with a VCS on top of it. You’ll learn more about what this means in a bit.

In the early days of Git (mostly pre-1.5), the user interface to the VCS was much more complex because it emphasized the filesystem rather than presenting a polished VCS. In the last few years, the UI has been refined until it’s as clean and easy to use as any VCS out there. But, Git’s reputation still suffers from its early complex and difficult to learn UI.

The content-addressable filesystem layer is amazingly cool, so I’ll cover that first. Then, you’ll learn about the transport protocols and the repository maintenance tasks that you may eventually have to deal with.

## Plumbing and Porcelain ##

This book covers how to use 30 or so Git commands such as `git checkout`, `git branch`, `git remote`, and so on. But because Git was initially a toolkit for a VCS rather than a complete user-friendly VCS, it also contains a bunch of commands that do low-level work. These commands were designed to be chained together UNIX style or called from scripts, and are generally referred to as "plumbing" commands. The more user-friendly commands are called "porcelain" commands.

The book’s first eight chapters deal almost exclusively with porcelain commands. This chapter, however, deals mostly with the lower-level plumbing commands because they illustrate the inner workings of Git and help demonstrate how and why Git does what it does. These commands aren’t meant to be used directly on the command line, but rather to be used as building blocks for new tools and custom scripts.

When you run `git init`, Git creates a `.git` directory in the current directory. The `.git` directory is where almost everything that Git stores and manipulates is located. If you want to back up or clone your repository, copying `.git` includes nearly everything you need. This entire chapter basically deals with the stuff in this directory.

A fresh `.git` directory contains

	$ ls
	HEAD
	branches/
	config
	description
	hooks/
	index
	info/
	objects/
	refs/

You may see some other files, but this is what you see by default. The `branches` directory isn’t used by newer Git versions, and the `description` file is only used by the GitWeb program, so I won’t cover them. The `config` file contains your project-specific configuration options, and the `info` directory keeps a global exclude file containing ignore patterns that you don’t want in a .gitignore file. The `hooks` directory contains client- or server-side hook scripts, which were discussed in detail in Chapter 7.

This leaves the `HEAD` and `index` files, and the `objects` and `refs` directories. These are the core parts of Git. The `objects` directory stores the database content, the `refs` directory stores pointers to branches, the `HEAD` file points to the branch you’re currently working on, and the `index` file is where Git stores staging area information. I’ll now describe each of these sections in detail.

## Git Objects ##

Git is a content-addressable filesystem. Great. What does that mean?
It means that at the core of Git is a simple key-value data store. You can insert any kind of content into it, and Git will give you back a key that you can use to retrieve the content at any time. Git stores the content in the data store as objects, which I'll describe in great detail below.

The first kind of object you should be aware of is the `blob` object. This is the object type that Git uses to store files and directories.
To demonstrate, use the plumbing command `git hash-object`, which takes some data, stores it in your `.git` directory as a blob object, and gives you back the key associated with the object. First, initialize a new Git repository and examine what’s in the `objects` directory.

	$ cd /tmp
	$ mkdir test
	$ cd test
	$ git init
	Initialized empty Git repository in /tmp/test/.git/
	$ find .git/objects
	.git/objects
	.git/objects/info
	.git/objects/pack
	$ find .git/objects -type f

Git has initialized the `objects` directory and created empty `pack` and `info` directories in it. Now, store some text in your Git database.

	$ echo 'test content' | git hash-object -w --stdin
	d670460b4b4aece5915caf5c68d12f560a9fe3e4

The `-w` option tells `git hash-object` to store the object. Otherwise, the command simply says what the key would be. The `--stdin` option tells the command to read the content from stdin. If you don’t specify this, the command expects a filename. The output from the command is a 40-character SHA-1 hash. This is the checksum of the content you’re storing, plus a header, which you’ll learn about in a bit. This command inserted a blob object into your data store which you can see by running

	$ find .git/objects -type f
	.git/objects/d6/70460b4b4aece5915caf5c68d12f560a9fe3e4

You see a file in a subdirectory a couple of levels under the `objects` directory. This is how Git stores the blob object — as a single file per object whose name is the SHA-1 hash of the content and its header. The subdirectory name is the first 2 characters of the SHA-1 hash, and the filename is the remaining 38 characters. The content of the file is what you entered on stdin above.

Conversely, you can extract the object content with the `git cat-file` command. This command is sort of a Swiss army knife for inspecting Git objects. Passing `-p` to this command displays the content in a way that makes sense for the object type. For example, if the object is a blob containing a file, its content would be shown as follows:

	$ git cat-file -p d670460b4b4aece5915caf5c68d12f560a9fe3e4
	test content

Other object types, which you'll learn about below, are displayed differently.

By manually running Git plumbing commands you can do some simple version control. First, create a new file and save its contents in your Git database.

	$ echo 'version 1' > test.txt
	$ git hash-object -w test.txt
	83baae61804e65cc73a7201a7252750c76066a30

Then, write some new contents to the file, and save it again.

	$ echo 'version 2' > test.txt
	$ git hash-object -w test.txt
	1f7a7a472abf3dd9643fd615f6da379c4acb3e3a

Your database now contains blob objects for all three versions of the file.

	$ find .git/objects -type f
	.git/objects/1f/7a7a472abf3dd9643fd615f6da379c4acb3e3a
	.git/objects/83/baae61804e65cc73a7201a7252750c76066a30
	.git/objects/d6/70460b4b4aece5915caf5c68d12f560a9fe3e4

Now, revert the file to the version containing `version 1`.

	$ git cat-file -p 83baae61804e65cc73a7201a7252750c76066a30 > test.txt
	$ cat test.txt
	version 1

or the version containing `version 2`.

	$ git cat-file -p 1f7a7a472abf3dd9643fd615f6da379c4acb3e3a > test.txt
	$ cat test.txt
	version 2

Git can tell you the object type of any object, given its SHA-1 hash, when you run `git cat-file -t`.

	$ git cat-file -t 1f7a7a472abf3dd9643fd615f6da379c4acb3e3a
	blob

Notice that the blob object doesn’t contain the filename of the file whose contents the object contains. Filenames are stored in tree objects, which I present next.
 
### Tree Objects ###

A tree object contains one or more entries, each of which contains the SHA-1 hash of a blob or of another tree object, along with the mode, object type, and file or directory name. For example, a tree object in the simplegit project could contain the following:

	$ git cat-file -p 7894138638f9404098a56befa8d6a4fc2598c087
	100644 blob a906cb2a4a904a152e80877d4088654daad0c859      README
	100644 blob 8f94139338f9404f26296befa88755fc2598c289      Rakefile
	040000 tree 99f1a6d12cb4b6f19c8655fca46c3ecf317074e0      lib

You might notice the similarity between Git tree and blob objects, and a UNIX filesystem. The tree objects are like directories while the blob objects corresponding more or less to files.

Notice that entry for the `lib` directory isn’t a blob but a pointer to another tree object. Examining its contents shows the contents of the `lib` directory.

	$ git cat-file -p 99f1a6d12cb4b6f19c8655fca46c3ecf317074e0
	100644 blob 47c6340d6459e05787f644c2447d2595f5d3a54b      simplegit.rb

The mode value in a tree object isn’t the same as a Unix file protection mode value, although they look similar. There are only three modes valid for files (blobs) — `100644` for a normal file, `100755` for an executable file, and `120000` for a symbolic link. Other modes are used for directories (trees).

Conceptually, the data that Git is storing looks something like Figure 9-1.

Insert 18333fig0901.png
Figure 9-1. Simple version of the Git data model.

You can create your own tree object by using Git plumbing commands. Git normally writes a tree object by examining the state of your staging area and creating a tree object that reflects its structure. So, to create a tree object, first set up a staging area by adding some files to it. To create a staging area with a single entry — the first version of your `test.txt` file — use the plumbing command `git update-index`. Use this command to artificially add the earlier version of the `test.txt` file to an empty staging area. You must pass the `--add` option because the file doesn’t yet exist in your staging area, and `--cacheinfo` because the file you’re adding isn’t in your directory but is in your database. Then, specify the mode, SHA-1 hash, and filename.

	$ git update-index --add --cacheinfo 100644 \
	  83baae61804e65cc73a7201a7252750c76066a30 test.txt

Now, use the `git write-tree` command to write the staging area out to a tree object. No `-w` option is needed — running `git write-tree` automatically creates a tree object from the state of the staging area if that tree doesn’t yet exist.

	$ git write-tree
	d8329fc1cc938780ffdd9f94e0d364e0ea74f579
	$ git cat-file -p d8329fc1cc938780ffdd9f94e0d364e0ea74f579
	100644 blob 83baae61804e65cc73a7201a7252750c76066a30      test.txt

You can also verify that this is a tree object.

	$ git cat-file -t d8329fc1cc938780ffdd9f94e0d364e0ea74f579
	tree

Now create a new tree object with the second version of test.txt and a new file as well.

	$ echo 'new file' > new.txt
	$ git update-index test.txt
	$ git update-index --add new.txt

Your staging area now has the new version of `test.txt` as well as the new file `new.txt`. Create a tree object that reflects the current state of the staging area and see what it looks like.

	$ git write-tree
	0155eb4229851634a0f03eb265b69f5a2d56f341
	$ git cat-file -p 0155eb4229851634a0f03eb265b69f5a2d56f341
	100644 blob fa49b077972391ad58037050f2a75f74e3671e92      new.txt
	100644 blob 1f7a7a472abf3dd9643fd615f6da379c4acb3e3a      test.txt

Notice that this tree contains both file entries and also that the `test.txt` SHA-1 hash is the "version 2" SHA-1 hash from earlier (`1f7a7a`). Just for fun, add the first tree as a subdirectory into this one. Read trees into your staging area by running `git read-tree`. In this case, read an existing tree into your staging area as a subtree by using the `--prefix` option to `git read-tree`.

	$ git read-tree --prefix=bak d8329fc1cc938780ffdd9f94e0d364e0ea74f579
	$ git write-tree
	3c4e9cd789d88d8d89c1073707c3585e41b0e614
	$ git cat-file -p 3c4e9cd789d88d8d89c1073707c3585e41b0e614
	040000 tree d8329fc1cc938780ffdd9f94e0d364e0ea74f579      bak
	100644 blob fa49b077972391ad58037050f2a75f74e3671e92      new.txt
	100644 blob 1f7a7a472abf3dd9643fd615f6da379c4acb3e3a      test.txt

If you created a working directory from the new tree you just wrote, you would get the two files in the top level of the working directory and a subdirectory named `bak` that contained the first version of the test.txt file. You can think of the objects that Git contains now as being like Figure 9-2.

Insert 18333fig0902.png
Figure 9-2. The content structure of your current Git data.

### Commit Objects ###

You created three trees that contain the SHA-1 hashes of the blob objects representing the files you want to track. Each tree is said to point to a `snapshot` of the contents of the index at the time you created the tree. But the earlier problem remains — you must remember all three SHA-1 hashes in order to retrieve the snapshots. You also don’t have any information about who saved the snapshots, when they were saved, or why they were saved. This is the basic information that the commit object stores.

To create a commit object, run `git commit-tree` and specify the SHA-1 hash of a single tree and the commit objects, if any, that directly precede it. The first commit in a repository won’t have any preceding commits. Start with the first tree you wrote.

	$ echo 'first commit' | git commit-tree d8329f
	fdf4fc3344e67ab068f836878b6c4951e3b15f3d

Now look at your new commit object by running `git cat-file`.

	$ git cat-file -p fdf4fc3
	tree d8329fc1cc938780ffdd9f94e0d364e0ea74f579
	author Scott Chacon <schacon@gmail.com> 1243040974 -0700
	committer Scott Chacon <schacon@gmail.com> 1243040974 -0700

	first commit

The format of a commit object is simple. It specifies the SHA-1 hash for the top-level tree of the project at that point — the author/committer information pulled from your `user.name` and `user.email` configuration settings, combined with the current timestamp, a blank line, and then the commit message.

Next, create the other two commit objects, each referencing the commit that came directly before it.

	$ echo 'second commit' | git commit-tree 0155eb -p fdf4fc3
	cac0cab538b970a37ea1e769cbbde608743bc96d
	$ echo 'third commit'  | git commit-tree 3c4e9c -p cac0cab
	1a410efbd13591db07496601ebc7a059dd55cfe9

Each of the three commit objects points to one of the three snapshot trees objects you created. Oddly enough, you have a real Git history now that you can view with the `git log` command, if you pass it the last commit object’s SHA-1 hash.

	$ git log --stat 1a410e
	commit 1a410efbd13591db07496601ebc7a059dd55cfe9
	Author: Scott Chacon <schacon@gmail.com>
	Date:   Fri May 22 18:15:24 2009 -0700

	    third commit

	 bak/test.txt |    1 +
	 1 file changed, 1 insertion(+), 0 deletions(-)

	commit cac0cab538b970a37ea1e769cbbde608743bc96d
	Author: Scott Chacon <schacon@gmail.com>
	Date:   Fri May 22 18:14:29 2009 -0700

	    second commit

	 new.txt  |    1 +
	 test.txt |    2 +-
	 2 files changed, 2 insertions(+), 1 deletions(-)

	commit fdf4fc3344e67ab068f836878b6c4951e3b15f3d
	Author: Scott Chacon <schacon@gmail.com>
	Date:   Fri May 22 18:09:34 2009 -0700

	    first commit

	 test.txt |    1 +
	 1 file changed, 1 insertion(+), 0 deletions(-)

Amazing. You’ve just performed the low-level operations to build a Git commit history without using any of the Git porcelain commands. This is essentially what Git does when you run the `git add` and `git commit` commands. Git stores blobs representing the files that have changed, updates the staging area, writes out trees, and writes commit objects that reference the top-level trees and the commits that came immediately before them. These three main Git objects — the blob, the tree, and the commit — are initially stored as separate files in your `.git/objects` directory. Here are all the objects in the example `.git/objects` directory now, with comments saying what they store:

	$ find .git/objects -type f
	.git/objects/01/55eb4229851634a0f03eb265b69f5a2d56f341 # tree 2
	.git/objects/1a/410efbd13591db07496601ebc7a059dd55cfe9 # commit 3
	.git/objects/1f/7a7a472abf3dd9643fd615f6da379c4acb3e3a # test.txt v2
	.git/objects/3c/4e9cd789d88d8d89c1073707c3585e41b0e614 # tree 3
	.git/objects/83/baae61804e65cc73a7201a7252750c76066a30 # test.txt v1
	.git/objects/ca/c0cab538b970a37ea1e769cbbde608743bc96d # commit 2
	.git/objects/d6/70460b4b4aece5915caf5c68d12f560a9fe3e4 # 'test content'
	.git/objects/d8/329fc1cc938780ffdd9f94e0d364e0ea74f579 # tree 1
	.git/objects/fa/49b077972391ad58037050f2a75f74e3671e92 # new.txt
	.git/objects/fd/f4fc3344e67ab068f836878b6c4951e3b15f3d # commit 1

If you follow all the internal pointers, you get an object graph looking something like Figure 9-3.

Insert 18333fig0903.png
Figure 9-3. All the objects in your Git directory.

### Object Storage ###

I mentioned earlier that a header is stored with the content when creating a blob object. Let’s take a minute to look at how Git stores its objects. You’ll see how to store a blob object — in this case, the string "what is up, doc?" — interactively in the Ruby scripting language. Start up interactive Ruby mode with the `irb` command.

	$ irb
	>> content = "what is up, doc?"
	=> "what is up, doc?"

Git constructs a header that starts with the type of the object, in this case `blob`. Then, it adds a space followed by the size of the content, and finally a null byte.

	>> header = "blob #{content.length}\0"
	=> "blob 16\000"

Git concatenates the header and the original content into what I’ll call the new content. It then calculates the SHA-1 hash of the new content. You can calculate the SHA-1 hash of a string in Ruby by including the SHA1 digest library with the `require` statement and then calling `Digest::SHA1.hexdigest()`, passing the string to hash.

	>> store = header + content
	=> "blob 16\000what is up, doc?"
	>> require 'digest/sha1'
	=> true
	>> sha1 = Digest::SHA1.hexdigest(store)
	=> "bd9dbf5aae1a3862dd1526723246b20206e5fc37"

Git compresses the new content with zlib, which you can also do in Ruby. First, require the zlib library and then run `Zlib::Deflate.deflate()` on the new content.

	>> require 'zlib'
	=> true
	>> zlib_content = Zlib::Deflate.deflate(store)
	=> "x\234K\312\311OR04c(\317H,Q\310,V(-\320QH\311O\266\a\000_\034\a\235"

Finally, write your zlib-compressed new content to a Git object. Determine the path of the object you want to write out (the first two characters of the SHA-1 hash as the subdirectory name, and the last 38 characters as the filename in that subdirectory). In Ruby, use the `FileUtils.mkdir_p()` function to create the subdirectory if it doesn’t exist. Then, open the file with `File.open()` and write out the previously zlib-compressed new content to the file with a `write()` call on the resulting file handle.

	>> path = '.git/objects/' + sha1[0,2] + '/' + sha1[2,38]
	=> ".git/objects/bd/9dbf5aae1a3862dd1526723246b20206e5fc37"
	>> require 'fileutils'
	=> true
	>> FileUtils.mkdir_p(File.dirname(path))
	=> ".git/objects/bd"
	>> File.open(path, 'w') { |f| f.write zlib_content }
	=> 32

That’s it! You’ve created a valid Git blob object. All Git objects are stored the same way, just with different types. Instead of the string `blob`, the header will begin with `commit` or `tree`. Also, although the blob object content can be nearly anything, the commit and tree object content have specific requirements.

## Git References ##

You can run something like `git log 1a410e` to look through your whole history, but you still have to remember that `1a410e` is the last commit in order to know where to start to see all those objects. Git also allows creating a file with a specific name whose content helps you remember a specific commit. The SHA-1 hash of the commit is the content of the file.

In Git, these are called "references" or "refs". The files that contain the SHA-1 hashes are in the `.git/refs` directory. In the current project, this directory contains no files, but it has a simple structure.

	$ find .git/refs
	.git/refs
	.git/refs/heads
	.git/refs/tags
	$ find .git/refs -type f

To create a new reference to help remember where your latest commit is, you can technically do something as simple as

	$ echo "1a410efbd13591db07496601ebc7a059dd55cfe9" > .git/refs/heads/master

This created a head reference, because the SHA-1 hash you put in the file points to the head of a series of commit objects. The name of the reference is the final component in the filename — in this case `master`.

Now, use `master` instead of the SHA-1 hash in your Git commands.

	$ git log --pretty=oneline  master
	1a410efbd13591db07496601ebc7a059dd55cfe9 third commit
	cac0cab538b970a37ea1e769cbbde608743bc96d second commit
	fdf4fc3344e67ab068f836878b6c4951e3b15f3d first commit

You aren’t encouraged to directly edit reference files. Instead Git provides a safer command called `git update-ref` to update a reference.

	$ git update-ref refs/heads/master 1a410efbd13591db07496601ebc7a059dd55cfe9

That’s basically what a branch in Git is — a simple pointer or reference to the head of a list of commits. To create a branch back at the second commit, run

	$ git update-ref refs/heads/test cac0ca

Your branch will contain only work starting from that commit.

	$ git log --pretty=oneline test
	cac0cab538b970a37ea1e769cbbde608743bc96d second commit
	fdf4fc3344e67ab068f836878b6c4951e3b15f3d first commit

Now, your Git database conceptually looks something like Figure 9-4.

Insert 18333fig0904.png
Figure 9-4. Git directory objects with branch head references included.

When you run commands like `git branch (branchname)`, Git basically runs `git update-ref` to add the SHA-1 hash of the last commit of the branch you’re on into whatever new reference you want to create.

### The HEAD ###

The question now is, when you run `git branch (branchname)`, how does Git know the SHA-1 hash of the last commit? The answer is the HEAD file, which is a symbolic reference to the branch you’re currently on. By symbolic reference, I mean that unlike a normal reference, it doesn’t contain a SHA-1 hash but rather the name of another reference. If you look in this file, you’ll normally see something like

	$ cat .git/HEAD
	ref: refs/heads/master

If you run `git checkout test`, Git updates `.git/HEAD` to look like

	$ cat .git/HEAD
	ref: refs/heads/test

When you run `git commit`, Git creates the commit object, specifying the SHA-1 hash of the reference HEAD points to as the parent commit object.

You can also manually edit this file, but `git symbolic-ref` is a safer command for doing this. You can see the value of HEAD using the following command:

	$ git symbolic-ref HEAD
	refs/heads/master

You can also set the value of HEAD.

	$ git symbolic-ref HEAD refs/heads/test
	$ cat .git/HEAD
	ref: refs/heads/test

You can’t set a symbolic reference outside of the refs style.

	$ git symbolic-ref HEAD test
	fatal: Refusing to point HEAD outside of refs/

### Tags ###

I’ve just gone over Git’s three main object types, but there’s also a fourth type — the tag object. The tag object is very much like a commit object. It contains the username of the person doing the tagging, a date, a message, and a pointer to the object being tagged. The main difference is that a tag object points to a commit rather than a tree. It’s like a branch reference, but it never moves. It always points to the same commit but lets you use a friendlier name to refer to the commit.

As discussed in Chapter 2, there are two types of tags: annotated and lightweight. You make a lightweight tag by running something like

	$ git update-ref refs/tags/v1.0 cac0cab538b970a37ea1e769cbbde608743bc96d

That’s all a lightweight tag is — a branch that never moves. An annotated tag is more complex, however. If you create an annotated tag, Git creates a tag object and then creates a reference to point to the tag object rather than directly to the commit. You can see this by creating an annotated tag (`-a` specifies an annotated tag).

	$ git tag -a v1.1 1a410efbd13591db07496601ebc7a059dd55cfe9 -m 'test tag'

Here’s the object SHA-1 hash it created.

	$ cat .git/refs/tags/v1.1
	9585191f37f7b0fb9444f35a9bf50de191beadc2

Now, run `git cat-file` on that SHA-1 hash.

	$ git cat-file -p 9585191f37f7b0fb9444f35a9bf50de191beadc2
	object 1a410efbd13591db07496601ebc7a059dd55cfe9
	type commit
	tag v1.1
	tagger Scott Chacon <schacon@gmail.com> Sat May 23 16:48:58 2009 -0700

	test tag

Notice that the object entry points to the commit SHA-1 hash that you tagged. Also notice that it doesn’t need to point to a commit. You can tag any Git object. In the Git source code, for example, the maintainer has added his GPG public key as a blob object and then tagged it. You can view the public key by running

	$ git cat-file blob junio-gpg-pub

in the Git source code repository. The Linux kernel repository also has a non-commit-pointing tag object. The first tag created points to the initial tree of the import of the source code.

### Remotes ###

The third type of reference is a remote reference. If you add a remote and push to it, Git stores in the `refs/remotes` directory the SHA-1 hash of the most recent commit from each branch you pushed to that remote. For instance, add a remote called `origin` and push your `master` branch to it.

	$ git remote add origin git@github.com:schacon/simplegit-progit.git
	$ git push origin master
	Counting objects: 11, done.
	Compressing objects: 100% (5/5), done.
	Writing objects: 100% (7/7), 716 bytes, done.
	Total 7 (delta 2), reused 4 (delta 1)
	To git@github.com:schacon/simplegit-progit.git
	   a11bef0..ca82a6d  master -> master

The value in the `refs/remotes/origin/master` file then contains the SHA-1 hash of the `master` branch on the remote `origin` server.

	$ cat .git/refs/remotes/origin/master
	ca82a6dff817ec66f44342007202690a93763949

Remote references differ from branches (`refs/heads` references) mainly in that remote references can’t be checked out. Git moves them around as bookmarks to the last known state of where those branches were on remote servers.

## Packfiles ##

Let’s go back to the objects database for your test Git repository. At this point, you have 11 objects — 4 blobs, 3 trees, 3 commits, and 1 tag.

	$ find .git/objects -type f
	.git/objects/01/55eb4229851634a0f03eb265b69f5a2d56f341 # tree 2
	.git/objects/1a/410efbd13591db07496601ebc7a059dd55cfe9 # commit 3
	.git/objects/1f/7a7a472abf3dd9643fd615f6da379c4acb3e3a # test.txt v2
	.git/objects/3c/4e9cd789d88d8d89c1073707c3585e41b0e614 # tree 3
	.git/objects/83/baae61804e65cc73a7201a7252750c76066a30 # test.txt v1
	.git/objects/95/85191f37f7b0fb9444f35a9bf50de191beadc2 # tag
	.git/objects/ca/c0cab538b970a37ea1e769cbbde608743bc96d # commit 2
	.git/objects/d6/70460b4b4aece5915caf5c68d12f560a9fe3e4 # 'test content'
	.git/objects/d8/329fc1cc938780ffdd9f94e0d364e0ea74f579 # tree 1
	.git/objects/fa/49b077972391ad58037050f2a75f74e3671e92 # new.txt
	.git/objects/fd/f4fc3344e67ab068f836878b6c4951e3b15f3d # commit 1

Git compresses the contents of these files with zlib, and these files aren’t very big anyway, so they collectively occupy only 925 bytes. Add some larger content to the repository to demonstrate an interesting feature of Git. Add the `repo.rb` file from the Grit library you worked with earlier — about a 12K file.

	$ curl https://raw.github.com/mojombo/grit/master/lib/grit/repo.rb > repo.rb
	$ git add repo.rb
	$ git commit -m 'added repo.rb'
	[master 484a592] added repo.rb
	 3 files changed, 459 insertions(+), 2 deletions(-)
	 delete mode 100644 bak/test.txt
	 create mode 100644 repo.rb
	 rewrite test.txt (100%)

If you look at the resulting tree, you see the SHA-1 hash of the blob object holding the contents of `repo.rb`.

	$ git cat-file -p master^{tree}
	100644 blob fa49b077972391ad58037050f2a75f74e3671e92      new.txt
	100644 blob 9bc1dc421dcd51b4ac296e3e5b6e2a99cf44391e      repo.rb
	100644 blob e3f094f522629ae358806b17daf78246c27c007b      test.txt

Then check how big that object is on disk.

	$ du -b .git/objects/9b/c1dc421dcd51b4ac296e3e5b6e2a99cf44391e
	4102	.git/objects/9b/c1dc421dcd51b4ac296e3e5b6e2a99cf44391e

Now, modify that file a little, and see what happens.

	$ echo '# testing' >> repo.rb 
	$ git commit -am 'modified repo a bit'
	[master ab1afef] modified repo a bit
	 1 file changed, 1 insertion(+), 0 deletions(-)

Checking the tree created by that commit shows something interesting.

	$ git cat-file -p master^{tree}
	100644 blob fa49b077972391ad58037050f2a75f74e3671e92      new.txt
	100644 blob 05408d195263d853f09dca71d55116663690c27c      repo.rb
	100644 blob e3f094f522629ae358806b17daf78246c27c007b      test.txt

The blob containing `repo.rb` is now different, which means that although you added only a single line to the end of a 400-line file, Git stored that new content in a completely new object.

	$ du -b .git/objects/05/408d195263d853f09dca71d55116663690c27c
	4109	.git/objects/05/408d195263d853f09dca71d55116663690c27c

You now have two nearly identical 4K objects on your disk. Wouldn’t it be nice if Git could store one of them in full but then store the second object as only the difference between it and the first?

It turns out that this is exactly what Git does. The initial format in which Git saves objects is called loose object format. However, occasionally Git packs several loose objects into a single file called a packfile in order to save space and be more efficient. Git does this if you have too many loose objects around, if you push to a remote server, or if you run the `git gc` command manually. Let's see what happens when you run `git gc`.

	$ git gc
	Counting objects: 17, done.
	Delta compression using 2 threads.
	Compressing objects: 100% (13/13), done.
	Writing objects: 100% (17/17), done.
	Total 17 (delta 1), reused 10 (delta 0)

If you look in your objects directory, you’ll find that most of your objects are gone, and a new pair has appeared.

	$ find .git/objects -type f
	.git/objects/71/08f7ecb345ee9d0084193f147cdad4d2998293
	.git/objects/d6/70460b4b4aece5915caf5c68d12f560a9fe3e4
	.git/objects/info/packs
	.git/objects/pack/pack-7a16e4488ae40c7d2bc56ea2bd43e25212a66c45.idx
	.git/objects/pack/pack-7a16e4488ae40c7d2bc56ea2bd43e25212a66c45.pack

The objects that remain are the blobs that aren’t pointed to by any commit — in this case, the "what is up, doc?" and the "test content" blobs you created earlier. Because you never added them to any commits, they’re considered dangling and aren’t included in your new packfile.

The two new files are your new packfile and an index. The packfile is a single file containing all the objects that were removed from your objects directory. The index contains offsets into that packfile so Git can quickly seek to a specific object. What’s cool is that although the objects on disk before you ran `git gc` were collectively about 8K in size, the new packfile is only 4K. You’ve halved your disk usage by packing your objects.

How does Git do this? When Git packs objects, it looks for files that are named and sized similarly, and stores just the differences from one version of the file to the next. You can examine the packfile to see what Git did to save space. The `git verify-pack` plumbing command allows you to see what was packed.

	$ git verify-pack -v \
	  .git/objects/pack/pack-7a16e4488ae40c7d2bc56ea2bd43e25212a66c45.idx
	0155eb4229851634a0f03eb265b69f5a2d56f341 tree   71 76 5400
	05408d195263d853f09dca71d55116663690c27c blob   12908 3478 874
	09f01cea547666f58d6a8d809583841a7c6f0130 tree   106 107 5086
	1a410efbd13591db07496601ebc7a059dd55cfe9 commit 225 151 322
	1f7a7a472abf3dd9643fd615f6da379c4acb3e3a blob   10 19 5381
	3c4e9cd789d88d8d89c1073707c3585e41b0e614 tree   101 105 5211
	484a59275031909e19aadb7c92262719cfcdf19a commit 226 153 169
	83baae61804e65cc73a7201a7252750c76066a30 blob   10 19 5362
	9585191f37f7b0fb9444f35a9bf50de191beadc2 tag    136 127 5476
	9bc1dc421dcd51b4ac296e3e5b6e2a99cf44391e blob   7 18 5193 1 \
	  05408d195263d853f09dca71d55116663690c27c
	ab1afef80fac8e34258ff41fc1b867c702daa24b commit 232 157 12
	cac0cab538b970a37ea1e769cbbde608743bc96d commit 226 154 473
	d8329fc1cc938780ffdd9f94e0d364e0ea74f579 tree   36 46 5316
	e3f094f522629ae358806b17daf78246c27c007b blob   1486 734 4352
	f8f51d7d8a1760462eca26eebafde32087499533 tree   106 107 749
	fa49b077972391ad58037050f2a75f74e3671e92 blob   9 18 856
	fdf4fc3344e67ab068f836878b6c4951e3b15f3d commit 177 122 627
	chain length = 1: 1 object
	pack-7a16e4488ae40c7d2bc56ea2bd43e25212a66c45.pack: ok

Here, the `9bc1d` blob, which, if you remember, was the first version of `repo.rb`, is referencing the `05408` blob, which was the second version of the file. The third column in the output is the size of the object’s content, so you can see that the content of `05408` takes up 12K, but that of `9bc1d` only takes up 7 bytes. What’s also interesting is that the second version of the file is the one that’s stored intact, whereas the original version is stored as a delta. This is because you’re most likely to need faster access to the most recent version.

The really nice thing about this technique is that Git can automatically repack objects at any time, always trying to save more space. You can also repack at any time by running `git gc` by hand.

## The Refspec ##

Throughout this book, you’ve used simple mappings from remote branches to local references but they can be more complex.
Suppose you add a remote like

	$ git remote add origin git@github.com:schacon/simplegit-progit.git

This adds a section to your `.git/config` file, specifying the name of the remote (`origin`), the URL of the remote repository, and the refspec for fetching.

	[remote "origin"]
	       url = git@github.com:schacon/simplegit-progit.git
	       fetch = +refs/heads/*:refs/remotes/origin/*

The format of the refspec is an optional `+`, followed by `<src>:<dst>`, where `<src>` is the pattern for references on the remote side and `<dst>` is where those references will be written locally. The `+` tells Git to update the reference even if it isn’t a fast-forward.

In the default case that’s automatically occurs when you run `git remote add`, Git fetches all the references under `refs/heads/` on the remote server and writes them to `refs/remotes/origin/` locally. So, if there’s a `master` branch on the remote server, you can access the log of that branch locally via

	$ git log origin/master
	$ git log remotes/origin/master
	$ git log refs/remotes/origin/master

They’re all equivalent, because Git expands each of them to `refs/remotes/origin/master`.

If you want Git instead to pull only the `master` branch, rather than every branch on the remote server, change the fetch line to

	fetch = +refs/heads/master:refs/remotes/origin/master

This is just the default refspec for `git fetch` for that remote. To fetch a branch only once, specify the full refspec on the command line. For example, to pull the `master` branch on the remote server to `origin/mymaster` locally, run

	$ git fetch origin master:refs/remotes/origin/mymaster

You can also specify multiple refspecs. On the command line, pull down several branches like so.

	$ git fetch origin master:refs/remotes/origin/mymaster \
	   topic:refs/remotes/origin/topic
	From git@github.com:schacon/simplegit
	 ! [rejected]        master     -> origin/mymaster  (non fast forward)
	 * [new branch]      topic      -> origin/topic

In this case, the pull of the `master` branch was rejected because it wasn’t a fast-forward reference. You can override that by including the `+` in front of the refspec.

You can also specify multiple refspecs for fetching your configuration file. To always fetch the `master` and `experiment` branches, add two lines.

	[remote "origin"]
	       url = git@github.com:schacon/simplegit-progit.git
	       fetch = +refs/heads/master:refs/remotes/origin/master
	       fetch = +refs/heads/experiment:refs/remotes/origin/experiment

You can’t use partial globs in the pattern, so the following would be invalid:

	fetch = +refs/heads/qa*:refs/remotes/origin/qa*

However, you can use namespacing to accomplish something similar. If you have a QA team that pushes a series of branches, and you want to pull the `master` branch and any of the QA team’s branches but nothing else, create a config section containing

	[remote "origin"]
	       url = git@github.com:schacon/simplegit-progit.git
	       fetch = +refs/heads/master:refs/remotes/origin/master
	       fetch = +refs/heads/qa/*:refs/remotes/origin/qa/*

If you have a complex workflow that includes a QA team pushing branches, developers pushing branches, and integration teams pushing and collaborating on remote branches, namespace them easily this way.

### Pushing Refspecs ###

It’s nice that you can fetch namespaced references that way, but how does the QA team get their branches into a `qa/` namespace in the first place? They accomplish that by using refspecs when they push.

If the QA team wants to push their `master` branch to `qa/master` on the remote server, they run

	$ git push origin master:refs/heads/qa/master

If they want Git to do that automatically each time they run `git push origin`, they add a `push` value to their config file.

	[remote "origin"]
	       url = git@github.com:schacon/simplegit-progit.git
	       fetch = +refs/heads/*:refs/remotes/origin/*
	       push = refs/heads/master:refs/heads/qa/master

Again, this will cause `git push origin` to push the local `master` branch to the remote `qa/master` branch by default.

### Deleting References ###

You can also use the refspec to delete references from the remote server by running something like

	$ git push origin :topic

Because the refspec is `<src>:<dst>`, leaving off the `<src>` part says to make the topic branch on the remote nothing, which deletes it.

## Transfer Protocols ##

Git can transfer data between repositories in two primary ways: over HTTP and via the so-called smart protocols used by the `file://`, `ssh://`, and `git://` transports. This section quickly covers how these two styles operate.

### The Dumb Protocol ###

Git transport over HTTP is often referred to as a dumb protocol because it doesn’t run any Git code on the remote server during the transport process. Fetching is simply a series of HTTP GET requests, where the client can replicate the layout of the Git repository that appears on the server. Let’s follow the `http-fetch` process for the simplegit library.

	$ git clone http://github.com/schacon/simplegit-progit.git

The first thing this command does is pull down the `info/refs` file. This file is updated by the `update-server-info` command, which for the HTTP transport to work properly needs to be enabled as a `post-receive` hook:

	=> GET info/refs
	ca82a6dff817ec66f44342007202690a93763949     refs/heads/master

Now you have a list of the remote references and SHA-1 hashes. Next, Git looks for what the HEAD reference points to so it knows what to check out when the transfer is finished.

	=> GET HEAD
	ref: refs/heads/master

At this point, Git is ready to start the repository walking process. Because the starting point is the `ca82a6` commit object found in the `info/refs` file, Git starts by fetching that object.

	=> GET objects/ca/82a6dff817ec66f44342007202690a93763949
	(179 bytes of binary data)

The object that’s returned is in loose format on the server, and Git fetched it using a static HTTP GET request. Git can zlib-uncompress it, strip off the header, and look at the commit content.

	$ git cat-file -p ca82a6dff817ec66f44342007202690a93763949
	tree cfda3bf379e4f8dba8717dee55aab78aef7f4daf
	parent 085bb3bcb608e1e8451d4b2432f8ecbe6306e7e7
	author Scott Chacon <schacon@gmail.com> 1205815931 -0700
	committer Scott Chacon <schacon@gmail.com> 1240030591 -0700

	changed the version number

Next, Git has two more objects to retrieve — `cfda3b`, which is the tree object that the commit just retrieved points to, and `085bb3`, which is the parent commit.

	=> GET objects/08/5bb3bcb608e1e8451d4b2432f8ecbe6306e7e7
	(179 bytes of data)

That gives the next commit object. Git grabs the tree object.

	=> GET objects/cf/da3bf379e4f8dba8717dee55aab78aef7f4daf
	(404 - Not Found)

Oops — it looks like that tree object isn’t in loose format on the server, so a 404 response is returned. There are a couple of reasons for this. The object could be in an alternate repository, or it could be in a packfile in this repository. Git checks for any listed alternates first.

	=> GET objects/info/http-alternates
	(empty file)

If this comes back with a list of alternate URLs, Git checks for loose files and packfiles in each one. This is a nice mechanism for projects that are forks of one another to share objects. However, because in this case no alternates are listed, your object must be in a packfile. To see what packfiles are available on this server, Git needs to retrieve the `objects/info/packs` file, which contains a listing of all the packfiles (also generated by `update-server-info`).

	=> GET objects/info/packs
	P pack-816a9b2334da9953e530f27bcac22082a9f5b835.pack

There’s only one packfile on the server, so the object is obviously there, but Git checks the index file to make sure. This is also useful if there are multiple packfiles on the server, so Git can see which packfile contains the object it needs.

	=> GET objects/pack/pack-816a9b2334da9953e530f27bcac22082a9f5b835.idx
	(4k of binary data)

Now that Git has read the packfile index, Git can search the index for the SHA-1 it’s looking for. Remember, the index contains hashes of the objects contained in the packfile and the offsets to those objects. The object is there, so Git can go ahead and get the whole packfile.

	=> GET objects/pack/pack-816a9b2334da9953e530f27bcac22082a9f5b835.pack
	(13k of binary data)

Git found the tree object, and continues walking the commits. They’re all also within the packfile just downloaded, so Git doesn’t have to make any more requests to the server. Git checks out a working copy of the `master` branch that was pointed to by the HEAD reference it downloaded at the beginning.

The entire output of this process looks like

	$ git clone http://github.com/schacon/simplegit-progit.git
	Initialized empty Git repository in /private/tmp/simplegit-progit/.git/
	got ca82a6dff817ec66f44342007202690a93763949
	walk ca82a6dff817ec66f44342007202690a93763949
	got 085bb3bcb608e1e8451d4b2432f8ecbe6306e7e7
	Getting alternates list for http://github.com/schacon/simplegit-progit.git
	Getting pack list for http://github.com/schacon/simplegit-progit.git
	Getting index for pack 816a9b2334da9953e530f27bcac22082a9f5b835
	Getting pack 816a9b2334da9953e530f27bcac22082a9f5b835
	 which contains cfda3bf379e4f8dba8717dee55aab78aef7f4daf
	walk 085bb3bcb608e1e8451d4b2432f8ecbe6306e7e7
	walk a11bef06a3f659402fe7563abf99ad00de2209e6

### The Smart Protocol ###

The HTTP method is simple but inefficient. It's more common to use smart protocols to transfer Git data. These protocols require a Git process to be running on the remote end. The process can read server Git data and figure out what the client has or needs so that it can return just what’s needed. There are actually two sets of processes for transferring data — a pair for uploading data and a pair for downloading data.

#### Uploading Data ####

To upload data to a remote process, Git uses the `send-pack` and `receive-pack` processes. The `send-pack` process runs on the client and connects to a `receive-pack` process on the remote end.

For example, say you run `git push origin master`, and `origin` is defined as a URL that uses the SSH protocol. Git fires up the `send-pack` process, which initiates a connection over SSH to the remote server. `send-pack` tries to run a command on the remote server via an SSH call that looks something like

	$ ssh -x git@github.com "git-receive-pack 'schacon/simplegit-progit.git'"
	005bca82a6dff817ec66f4437202690a93763949 refs/heads/master report-status delete-refs
	003e085bb3bcb608e1e84b2432f8ecbe6306e7e7 refs/heads/topic
	0000

The `git-receive-pack` command immediately responds with one line for each reference currently in the remote repository — in this case, just the `master` branch and its SHA-1 hash. The first line also includes a list of the server’s capabilities (here, `report-status` and `delete-refs`).

Each line starts with a 4-byte hex value specifying the length of the rest of the line. The first line starts with 005b in hex, which is 91 in decimal, meaning that 91 bytes remain on that line. The next line starts with 003e, which is 62, so Git reads the remaining 62 bytes. The next line is 0000, meaning the server is done listing references.

Now that the `send-pack` process knows the server’s state,  it determines which commits the client has that the server doesn’t. The `send-pack` process tells the `receive-pack` process about each remote reference that this push will update. For instance, if you’re updating the `master` branch and adding an `experiment` branch, the `send-pack` response may look something like

	0085ca82a6dff817ec66f44342007202690a93763949  15027957951b64cf874c3557a0f3547bd83b3ff6 refs/heads/master report-status
	00670000000000000000000000000000000000000000 cdfdb42577e2506715f8cfeacdbabc092bf63e8d refs/heads/experiment
	0000

The SHA-1 hash of all '0's means that nothing was there before, because you’re adding the `experiment` reference. If you were deleting a reference, you would see the opposite — all '0's on the right side.

For each reference you’re updating Git sends a line containing the old SHA-1 hash, the new SHA-1 hash, and the reference that’s being updated. The first line also shows the client’s capabilities. Next, the client uploads a packfile containing all the objects the server doesn’t have yet. Finally, the server responds with a success (or failure) indication.

	000Aunpack ok

#### Downloading Data ####

When Git downloads data, the `fetch-pack` and `upload-pack` processes are run. The client initiates a `fetch-pack` process that connects to an `upload-pack` process on the remote server in order to negotiate what data will be transferred.

There are different ways to initiate the `upload-pack` process on the remote server. It can start via SSH the same way as the `receive-pack` process. Or, if the remote server is running the Git daemon, you can initiate the transfer process by connecting to port 9418. The `fetch-pack` process sends data that looks like this to the daemon after connecting.

	003fgit-upload-pack schacon/simplegit-progit.git\0host=myserver.com\0

It starts with 4 bytes specifying how much data follows, then the command to run followed by a null byte, and then the server’s hostname followed by a final null byte. The Git daemon checks that the command can be run, that the repository exists, and that the repository is publicly accessible. If everything passes, the Git daemon fires up the `upload-pack` process and hands off the request to it.

If you’re doing the fetch over SSH, `fetch-pack` instead runs something like

	$ ssh -x git@github.com "git-upload-pack 'schacon/simplegit-progit.git'"

In either case, after `fetch-pack` connects, `upload-pack` sends back something like

	0088ca82a6dff817ec66f44342007202690a93763949 HEAD\0multi_ack thin-pack \
	  side-band side-band-64k ofs-delta shallow no-progress include-tag
	003fca82a6dff817ec66f44342007202690a93763949 refs/heads/master
	003e085bb3bcb608e1e8451d4b2432f8ecbe6306e7e7 refs/heads/topic
	0000

This is very similar to how `receive-pack` responds, but the capabilities are different. In addition, `upload-pack` sends back the HEAD reference so the client knows what to check out if this is a clone.

At this point, the `fetch-pack` process looks at the objects it has and responds with the objects that it needs by sending "want" and then the SHA-1 hash of the object it wants. It then sends all the objects it already has with "have" and then the SHA-1 hashes of the objects. At the end of this list, it sends "done" to tell the `upload-pack` process to begin sending the packfile of the data it needs.

	0054want ca82a6dff817ec66f44342007202690a93763949 ofs-delta
	0032have 085bb3bcb608e1e8451d4b2432f8ecbe6306e7e7
	0000
	0009done

These are very basic examples of the transfer protocols. In more complex cases, the client supports `multi_ack` or `side-band` capabilities but these examples show you the basic back and forth used by the smart protocol processes.

## Maintenance and Data Recovery ##

Occasionally, you may have to do some cleanup — compact a repository, clean up an imported repository, or recover lost work. This section will cover some of these scenarios.

### Maintenance ###

Occasionally, Git runs `git gc --auto`. Most of the time, this command does nothing. However, if there are too many loose objects (objects not in a packfile) or too many packfiles, Git launches a full-fledged garbage collection pass, which does a number of things. It gathers up all the loose objects and places them in packfiles, it consolidates packfiles into one big packfile, and it removes objects that aren’t reachable from any commit and are a few months old.

You can run auto gc manually.

	$ git gc --auto

Again, this does nothing unless you have around 7,000 loose objects or more than 50 packfiles. You can modify these limits with the `gc.auto` and `gc.autopacklimit` config settings, respectively.

The other thing `git gc` does is pack up your references into a single file. Suppose your repository contains the following branches and tags:

	$ find .git/refs -type f
	.git/refs/heads/experiment
	.git/refs/heads/master
	.git/refs/tags/v1.0
	.git/refs/tags/v1.1

If you run `git gc`, you’ll no longer have these files in the `refs` directory. For the sake of efficiency Git will move them into a file named `.git/packed-refs` that looks like

	$ cat .git/packed-refs
	# pack-refs with: peeled
	cac0cab538b970a37ea1e769cbbde608743bc96d refs/heads/experiment
	ab1afef80fac8e34258ff41fc1b867c702daa24b refs/heads/master
	cac0cab538b970a37ea1e769cbbde608743bc96d refs/tags/v1.0
	9585191f37f7b0fb9444f35a9bf50de191beadc2 refs/tags/v1.1
	^1a410efbd13591db07496601ebc7a059dd55cfe9

If you update a reference, Git doesn’t edit `.git/packed-refs` but instead writes a new file in `refs/heads`. To get the appropriate SHA-1 hash for a given reference, Git checks for that reference in the `refs` directory and then checks the `packed-refs` file as a fallback. However, if Git can’t find a reference in the `refs` directory, it’s probably in the `packed-refs` file.

Notice the last line of the file, which begins with a `^`. This means the tag directly above is an annotated tag and that line is the commit that the annotated tag points to.

### Data Recovery ###

At some point in your Git journey, you may accidentally lose a commit. Generally, this happens because you force-delete a branch that had work on it, and it turns out you wanted the branch after all. Or, you hard-reset a branch, thus abandoning commits that you still need. If this happens, how can you get your commits back?

Here’s an example that hard-resets the master branch in your test repository to an older commit and then recovers the lost commits. First, let’s review where your repository is at this point.

	$ git log --pretty=oneline
	ab1afef80fac8e34258ff41fc1b867c702daa24b modified repo a bit
	484a59275031909e19aadb7c92262719cfcdf19a added repo.rb
	1a410efbd13591db07496601ebc7a059dd55cfe9 third commit
	cac0cab538b970a37ea1e769cbbde608743bc96d second commit
	fdf4fc3344e67ab068f836878b6c4951e3b15f3d first commit

Now, move the `master` branch back to the middle commit.

	$ git reset --hard 1a410efbd13591db07496601ebc7a059dd55cfe9
	HEAD is now at 1a410ef third commit
	$ git log --pretty=oneline
	1a410efbd13591db07496601ebc7a059dd55cfe9 third commit
	cac0cab538b970a37ea1e769cbbde608743bc96d second commit
	fdf4fc3344e67ab068f836878b6c4951e3b15f3d first commit

You’ve effectively lost the top two commits. You have no branch from which those commits are reachable. You need to find the latest commit SHA-1 hash and then add a branch that points to it. The trick is finding that latest commit SHA-1 hash. You probably haven’t memorized it, right?

The quickest way is often using a tool called `git reflog`. As you’re working, Git silently records the value of HEAD every time it changes. Each time you commit or change branches, the reflog is updated. The reflog is also updated by the `git update-ref` command, which is another reason to use it instead of just writing the SHA-1 hash value to your ref files, as I mentioned earlier in the "Git References" section of this chapter. You can see where you’ve been at any time by running `git reflog`.

	$ git reflog
	1a410ef HEAD@{0}: 1a410efbd13591db07496601ebc7a059dd55cfe9: updating HEAD
	ab1afef HEAD@{1}: ab1afef80fac8e34258ff41fc1b867c702daa24b: updating HEAD

Here you can see the two commits that you checked out but there isn’t much other information here.  To see the same information in a much more useful way, run `git log -g`, which will give you a normal log output for your reflog.

	$ git log -g
	commit 1a410efbd13591db07496601ebc7a059dd55cfe9
	Reflog: HEAD@{0} (Scott Chacon <schacon@gmail.com>)
	Reflog message: updating HEAD
	Author: Scott Chacon <schacon@gmail.com>
	Date:   Fri May 22 18:22:37 2009 -0700

	    third commit

	commit ab1afef80fac8e34258ff41fc1b867c702daa24b
	Reflog: HEAD@{1} (Scott Chacon <schacon@gmail.com>)
	Reflog message: updating HEAD
	Author: Scott Chacon <schacon@gmail.com>
	Date:   Fri May 22 18:15:24 2009 -0700

	     modified repo a bit

It looks like the bottom commit is the one you lost, so you can recover it by creating a new branch at that commit. For example, start a branch named `recover-branch` at that commit (ab1afef).

	$ git branch recover-branch ab1afef
	$ git log --pretty=oneline recover-branch
	ab1afef80fac8e34258ff41fc1b867c702daa24b modified repo a bit
	484a59275031909e19aadb7c92262719cfcdf19a added repo.rb
	1a410efbd13591db07496601ebc7a059dd55cfe9 third commit
	cac0cab538b970a37ea1e769cbbde608743bc96d second commit
	fdf4fc3344e67ab068f836878b6c4951e3b15f3d first commit

Cool. Now you have a branch named `recover-branch` that’s where your `master` branch used to be, making the first two commits reachable again.
Next, suppose, for some reason, the commits you lost aren’t in the reflog — you can simulate that by removing `recover-branch` and deleting the reflog. Now the first two commits aren’t reachable by anything.

	$ git branch -D recover-branch
	$ rm -Rf .git/logs/

Because the reflog data is kept in the `.git/logs/` directory, you’ve just lost the reflog. How can you recover a commit at this point? One way is to use the `git fsck` utility, which checks the integrity of your repository. If you run `git fsck` with the `--full` option, you’ll see all the objects that aren’t pointed to by another object.

	$ git fsck --full
	dangling blob d670460b4b4aece5915caf5c68d12f560a9fe3e4
	dangling commit ab1afef80fac8e34258ff41fc1b867c702daa24b
	dangling tree aea790b9a58f6cf6f2804eeac9f0abbe9631e4c9
	dangling blob 7108f7ecb345ee9d0084193f147cdad4d2998293

In this case, you can see your missing commit after the dangling commit. You can recover it the same way, by adding a branch that points to that SHA-1 hash.

### Removing Objects ###

There are a lot of great things about Git, but one feature that can cause issues is the fact that `git clone` downloads the entire history of the project, including every version of every file. This is fine if the whole thing is source code, because Git is highly optimized to compress source code efficiently. However, if someone at any point in the history of your project added a single huge file, every clone for all time will be forced to include that file, even if it was removed from the project in the very next commit. Because the commit that added the huge file is in the commit history, the file will always be there.

This can be a huge problem when you’re importing Subversion or Perforce repositories into Git. If you find that your repository is much larger than it should be, here’s how you can find and remove large objects.

Be warned: this technique is destructive to your commit history. It rewrites every commit object starting from the earliest tree containing a large file reference. If you do this immediately after an import, before anyone has started to base work on the commit, you’re fine. Otherwise, you have to notify all contributors that they must rebase their work onto your new commits.

To demonstrate, add a large file into a test repository, remove it in the next commit, find it, and remove it permanently from the repository. First, add a large file.

	$ curl http://kernel.org/pub/software/scm/git/git-1.6.3.1.tar.bz2 > git.tbz2
	$ git add git.tbz2
	$ git commit -am 'added git tarball'
	[master 6df7640] added git tarball
	 1 file changed, 0 insertions(+), 0 deletions(-)
	 create mode 100644 git.tbz2

Oops — you didn’t want to add a huge tarball to your project. Better get rid of it.

	$ git rm git.tbz2
	rm 'git.tbz2'
	$ git commit -m 'oops - removed large tarball'
	[master da3f30d] oops - removed large tarball
	 1 files changed, 0 insertions(+), 0 deletions(-)
	 delete mode 100644 git.tbz2

Now, run `git gc` and see how much space you’re using.

	$ git gc
	Counting objects: 21, done.
	Delta compression using 2 threads.
	Compressing objects: 100% (16/16), done.
	Writing objects: 100% (21/21), done.
	Total 21 (delta 3), reused 15 (delta 1)

Run `git count-objects` to quickly see how much space you’re using:

	$ git count-objects -v
	count: 4
	size: 16
	in-pack: 21
	packs: 1
	size-pack: 2016
	prune-packable: 0
	garbage: 0

The `size-pack` entry is the size of your packfiles in kilobytes, so you’re using 2MB. Before the last commit, you were using closer to 2K. Clearly, removing the file from the previous commit didn’t remove it from your history. Every time anyone clones this repository, they’ll have to clone all 2MB just to get this tiny project, because you accidentally added a big file. Let’s get rid of it.

First you have to find it. In this case, you already know what file it is. But suppose you didn’t know this. How would you identify what was taking up so much space? If you run `git gc`, all the objects are in a packfile. You can identify the big objects by running another plumbing command called `git verify-pack` and sorting on the third field of the output, which is file size. You can also pipe the output through the `tail` command because you’re only interested in the last few largest files.

	$ git verify-pack -v .git/objects/pack/pack-3f8c0...bb.idx | sort -k 3 -n | tail -3
	e3f094f522629ae358806b17daf78246c27c007b blob   1486 734 4667
	05408d195263d853f09dca71d55116663690c27c blob   12908 3478 1189
	7a9eb2fba2b1811321254ac360970fc169ba2330 blob   2056716 2056872 5401

The big object is the 2MB object at the bottom. To find out what file it is, use the `git rev-list` command, which you used briefly in Chapter 7. If you pass `--objects` to `git rev-list`, it lists all the commit SHA-1 hashes and also the blob SHA-1 hashes with the file paths associated with them. Use this to find your blob’s name.

	$ git rev-list --objects --all | grep 7a9eb2fb
	7a9eb2fba2b1811321254ac360970fc169ba2330 git.tbz2

Now, you need to remove this file from all trees in your history. You can easily see what commits added or remoted this file.

	$ git log --pretty=oneline --branches -- git.tbz2
	da3f30d019005479c99eb4c3406225613985a1db oops - removed large tarball
	6df764092f3e7c8f5f94cbe08ee5cf42e92a0289 added git tarball

You must rewrite all the commits downstream from `6df76` to fully remove this file from your Git history. To do so, use `git filter-branch`, which you used in Chapter 6.

	$ git filter-branch --index-filter \
	   'git rm --cached --ignore-unmatch git.tbz2' -- 6df7640^..
	Rewrite 6df764092f3e7c8f5f94cbe08ee5cf42e92a0289 (1/2)rm 'git.tbz2'
	Rewrite da3f30d019005479c99eb4c3406225613985a1db (2/2)
	Ref 'refs/heads/master' was rewritten

The `--index-filter` option is similar to the `--tree-filter` option used in Chapter 6, except that instead of passing a command that modifies checked out files, you’re modifying your staging area each time. Rather than remove a specific file with something like `git rm file`, you have to remove it with `git rm --cached`. You must remove it from the index, not from your working directory. The reason for this is speed. Because Git doesn’t have to check out each revision before running your filter, the process can be much, much faster. You can accomplish the same task with the `--tree-filter` option. The `--ignore-unmatch` option to `git rm` tells it not to abort if the pattern you’re trying to remove isn’t found. Finally, tell `git filter-branch` to rewrite your history only starting from the `6df7640` commit, because you know that’s where this problem started. Otherwise, it will start from the first commit and will unnecessarily take longer.

Your history no longer contains a reference to that file. However, your reflog and the new set of refs that Git added when you ran `git filter-branch` under `.git/refs/original` still contain the references, so you have to remove them and then repack the database. You need to get rid of anything that has a pointer to those old commits before you repack.

	$ rm -Rf .git/refs/original
	$ rm -Rf .git/logs/
	$ git gc
	Counting objects: 19, done.
	Delta compression using 2 threads.
	Compressing objects: 100% (14/14), done.
	Writing objects: 100% (19/19), done.
	Total 19 (delta 3), reused 16 (delta 1)

Let’s see how much space you saved.

	$ git count-objects -v
	count: 8
	size: 2040
	in-pack: 19
	packs: 1
	size-pack: 7
	prune-packable: 0
	garbage: 0

The packed repository size is down to 7K, which is much better than 2MB. You can see from the size that the big object is still in your loose objects, so it’s not gone. But it won’t be transferred on a push or subsequent clone, which is what’s important. If you really want, you could remove the file completely by running `git prune --expire`.

## Summary ##

You should have a pretty good understanding of what Git does in the background and, to some degree, how it’s implemented. This chapter has covered a number of plumbing commands — commands that are lower level and simpler than the porcelain commands you’ve learned about in the rest of the book. Understanding how Git works at this level should make it easier to understand why it’s doing what it’s doing. You should also be able to write your own tools and custom scripts to make your specific workflow work for you.

Git as a content-addressable filesystem is a very powerful tool that you can easily use as more than just a VCS. I hope you use your new found knowledge of Git internals to implement your own cool application of this technology and feel more comfortable doing so.
