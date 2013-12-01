# Git on the Server #

At this point, you should be able to run most of the day-to-day Git commands you’ll need. However, in order to do any collaboration with Git, you’ll need to access a remote Git repository. Although you can technically push changes to and pull changes from individuals’ repositories, doing so is discouraged because that can fairly easily complicate what they’re working on if you’re not careful. Furthermore, collaborators want to be able to access each other’s repository even if their computer is offline. So, having everyone share a repository is a better solution. Therefore, the preferred method for collaborating is to set up a repository that everyone has access to, and push to and pull from it. We’ll refer to this repository as a "Git server" but since it generally takes a tiny amount of resources to host a Git repository, you’ll rarely need to use a dedicated server as a Git server.

Running a Git server is simple. First, you choose which protocols you want your server to support. The first section of this chapter covers the available protocols and their pros and cons. The next sections explain some typical configurations using those protocols. Last, I’ll go over a few hosted options, if you don’t mind hosting your code on someone else’s server and don’t want to go through the hassle of setting up and maintaining your own server.

If you have no interest in running your own server, skip to the last section of this chapter to see some options for setting up a hosted account. Then, move on to the next chapter, where I discuss the ins and outs of working in a distributed source control environment.

A remote repository is generally a _bare repository_ — a Git repository that has no working directory. Because the repository is only used as a collaboration point, there’s no reason to have a working directory. The Git data is all that’s necessary. In the simplest terms, a bare repository is a project’s `.git` directory and nothing else.

## The Protocols ##

Git can use four network protocols to transfer data: Local, Secure Shell (SSH), Git, and HTTP. Here I’ll discuss what they are and when you’d want (or not want) to use them.

It’s important to note that with the exception of HTTP, all of these protocols require Git to be installed on the server.

### Local Protocol ###

The most basic protocol is the _local_ protocol, in which the remote repository is in a directory on your local computer. This is often used if everyone on your team has access to a shared filesystem, such as an NFS mount, or in the less likely case that everyone logs in to the same computer. This isn’t a good idea because placing everyone’s code repository on the same disk makes a catastrophic loss much more likely.

If you have a shared filesystem mounted locally then you can clone, push to, and pull from a repository located on it. To clone from a local repository or to add one as a remote to an existing project, use the path to the repository in place of the URL. For example, to clone a local repository, run something like

	$ git clone /opt/git/project.git

or

	$ git clone file:///opt/git/project.git

Git operates slightly differently if you explicitly specify `file://` at the beginning of the URL than if you just give a local path. If you just specify the path, Git tries to use hardlinks or directly copy the files it needs. If you specify `file://`, Git uses the same technique that it normally uses to transfer data over a network, which is generally a lot less efficient. The main reason to specify the `file://` prefix is if you want a clean copy of the repository with no extraneous references or objects — like after an import from another version-control system or something similar (see Chapter 9). I’ll use a normal path here because this is almost always faster.

To add a local repository to an existing Git project, run something like

	$ git remote add local_proj /opt/git/project.git

Then, push to and pull from that remote as though you were transferring over a network.

#### The Pros ####

The pros of file-based repositories are that they’re simple and use existing file permissions. If you already have a shared filesystem to which your whole team has access, setting up a repository is very easy. Stick the bare repository somewhere where everyone has access and set the read/write permissions as you would for any other shared directory. I’ll discuss how to export a bare repository for this purpose in the next section, “Getting Git on a Server.”

This is also a nice option for quickly pulling from someone else’s working repository. If you and a co-worker are working on the same project and you want to pull from their repository, running a command like

	$ git pull /home/john/project

is often easier than your co-worker first pushing to a remote server and then you pulling from there.

#### The Cons ####

The con of this method is that shared access is generally more difficult to set up and reach from multiple locations than basic network access. To push from your laptop when you’re at home, you’d have to mount the remote disk containing the repository, which can be difficult and slow compared to the other network-based protocols.

It’s also important to mention that this isn’t necessarily the fastest option. A local repository is fast only if you have fast access to the repository. Accessing a repository via NFS is often slower than accessing the repository over SSH on the same remote server.

### The SSH Protocol ###

Probably the most common transport protocol for Git is SSH. This is because SSH access to servers is already set up almost everywhere — and if it isn’t, it’s easy to do. SSH is also the only network-based protocol that you can use for both read and write access. The other two network protocols (HTTP and Git) are generally read-only, so even if they’re available for the unwashed masses, you still need SSH for write access. SSH is also an authenticated network protocol and, because it’s ubiquitous, it’s generally easy to set up and use.

To clone a Git repository over SSH, specify an ssh:// URL.

	$ git clone ssh://user@server/project.git

Or, use the shorter SCP-like syntax for the SSH protocol.

	$ git clone user@server:project.git

If you don’t specify a user, Git substitutes your current username.

#### The Pros ####

The pros of using SSH are many. You have no choice if you want authenticated write access to a repository over the network. Second, SSH is relatively easy to set up — SSH daemons are commonplace, many network admins have experience with them, and many OS distributions include them and have tools to manage them. Also, access over SSH is secure — all data transfer is encrypted and authenticated. Last, like the Git and local protocols, SSH is efficient, compressing the data before transferring it.

#### The Cons ####

The negative aspect of SSH is that you can’t provide anonymous read-only access to your repository using it. Users must have SSH access to your server to access your repository, even in read-only mode, which makes SSH access inappropriate for open source projects. If you’re using it only within your corporate network, SSH may be the only protocol you need to deal with. To allow anonymous read-only access to your projects, you’ll have to set up SSH for you to push over but something else for others to pull over.

### The Git Protocol ###

Next is the Git protocol. This requires a special daemon that comes packaged with Git. It listens on a dedicated port (9418) that provides a service similar to SSH, but with absolutely no authentication. In order for a repository to be served using the Git protocol, you must create the `git-export-daemon-ok` file — the daemon won’t serve a repository that doesn't contain that file. Other than that there’s no security. Either the Git repository is available for everyone to clone or it isn’t available to anyone. This means that you generally don’t allow push access using this protocol. You can enable push access but, given the lack of authentication, if you do so, anyone on the internet who knows your project’s URL could push to your project. Suffice it to say that this is rarely a good thing.

#### The Pros ####

The Git protocol is the fastest Git repository transfer protocol available. If you’re serving a lot of traffic for a public project or serving a very large project that doesn’t require user authentication for read access, it’s likely that you’ll want to set up a Git daemon to serve your project. It uses the same data-transfer mechanism as the SSH protocol but without the encryption and authentication overhead.

#### The Cons ####

The downside of the Git protocol is the lack of authentication. It’s generally undesirable for the Git protocol to be the only way to access your project. Generally, you’ll pair it with SSH access for the few developers who have push (write) access, leaving everyone else to use the Git protocol for read-only access.
It’s also probably the most difficult protocol to set up. It requires running a daemon — I’ll describe setting one up in the “Gitosis” section of this chapter — and requires `xinetd` configuration or the like, which isn’t always a walk in the park. It also requires your firewall to allow access to port 9418, which isn’t a standard port that corporate firewalls allow.

### The HTTP Protocol ###

Last, there’s the HTTP protocol. (Since HTTPS works the same as HTTP as far as Git’s concerned, I won’t include it here). The beauty of the HTTP protocol is the simplicity of setting it up. Simply put the bare Git repository under your HTTP document root and set up a specific `post-update` hook, and you’re done (See Chapter 7 for details on Git hooks). At that point, anyone who can access the web server can also clone your repository. To allow read-only access to your repository over HTTP, do something like

	$ cd /var/www/htdocs/
	$ git clone --bare /path/to/git_project gitproject.git
	$ cd gitproject.git
	$ mv hooks/post-update.sample hooks/post-update
	$ chmod a+x hooks/post-update

That’s all. The `post-update` hook that comes with Git by default runs the appropriate command (`git update-server-info`) to make HTTP fetching and cloning work properly. This command is run when you push to this repository over SSH. Then, other people can clone via something like

	$ git clone http://example.com/gitproject.git

In this particular case, I’m using the `/var/www/htdocs` path that’s common for Apache setups, but you can use any path — just put the bare repository in the path. The Git data is served as static files (see Chapter 9 for details about exactly how).

It’s possible to make Git push over HTTP as well, although that technique isn’t as widely used and requires setting up a complex WebDAV environment. Because of these issues, I won’t cover it in this book. If you’re interested in using the HTTP-push protocols, read about preparing a repository using these protocols at `http://www.kernel.org/pub/software/scm/git/docs/howto/setup-git-server-over-http.txt`. One nice thing about Git pushing over HTTP is that you can use any WebDAV server, without any specific Git requirements. This means you can use this approach if your web-hosting provider supports WebDAV for updating your web site.

#### The Pros ####

The upside of using the HTTP protocol is that it’s so easy to set up. Running the handful of required commands gives you a simple way to give the world read-only access to your Git repository. The HTTP protocol also isn’t very resource intensive on your server. Because Git repositories are static files, a normal Apache server can serve thousands of files per second on average. It’s difficult to overload even a small server.

Another nice thing is that HTTP is such a commonly used protocol that corporate firewalls are often set up to pass HTTP traffic.

#### The Cons ####

The downside of serving your repository over HTTP is that it’s relatively inefficient in terms of network resources. It generally takes a lot longer to clone or fetch from the repository, and there’s a lot more network overhead and data transfer volume over HTTP than with any of the other network protocols. For more information about the differences in efficiency between the HTTP protocol and the other protocols, see Chapter 9.

## Getting Git on a Server ##

In order to initially set up a Git server, you have to start with a new bare repository — a repository that doesn’t contain a working directory. This is generally straightforward to do.
By convention, bare repository directories end in `.git`. To clone a repository to create a new bare repository, run `git clone --bare`.

	$ git clone --bare my_project my_project.git
	Initialized empty Git repository in /opt/projects/my_project.git/

The output for this command is a little confusing. Since `git clone` is basically a `git init` then a `git fetch`, you see some output from the `git init` part, which creates an empty directory. The actual object transfer generates no output. You should now have a copy of the Git repository in your `my_project.git` directory.

This is roughly equivalent to

	$ cp -Rf my_project/.git my_project.git

There are a couple of minor differences but, as far as you’re concerned, this is close to the same thing. It copies just the Git repository, without a working directory, into a new directory.

### Putting the Bare Repository on a Server ###

Now that you have a bare copy of your repository, all you need to do is put it on a server and set up your access protocols. Let’s say you’ve set up a server called `git.example.com` that you have SSH access to, and you want to store all your Git repositories under the `/opt/git` directory. Set up your new repository by copying your bare repository over.

	$ scp -r my_project.git user@git.example.com:/opt/git

At this point, other users who have SSH and read access to the `/opt/git` directory on the server can clone your repository by running

	$ git clone user@git.example.com:/opt/git/my_project.git

If a user SSHs into a server and has write access to the `/opt/git/my_project.git` directory, they will also be able to push to the repository. As an aside, Git automatically adds group write permissions to a repository if you run `git init --shared`.

	$ ssh user@git.example.com
	$ cd /opt/git/my_project.git
	$ git init --bare --shared

You see how easy it is to take a Git repository, create a bare version, and place it on a server to which you and your collaborators have SSH access. Now you’re ready to collaborate.

It’s important to note that this is literally all you need to do to run a useful Git server to which several people have access — just add accounts on a server that users can SSH to, and stick a bare repository somewhere that all those users have read and write access to. You’re ready to go — nothing else needed.

In the next few sections, you’ll see how to expand to more sophisticated setups. This discussion will include not having to create user accounts for each user, adding public read access to repositories, setting up web UIs, using the Gitosis tool, and more. However, keep in mind that to collaborate with a couple of people on a private project, all you _need_ is Git, an SSH server, and a bare repository.

### Small Setups ###

If you’re a small outfit or are just trying out Git in your organization and have only a few developers, setting up a Git server is easy. The most complicated task is user management. If you want some repositories to be read-only to certain users and read/write to others, access and permissions can be a bit difficult to configure.

#### SSH Access ####

If you already have a server to which all your developers have SSH access, it’s generally easiest to set up your first repository there, because you have to do almost no work (as I covered in the last section). If you want more complex access controls on your repositories, use normal filesystem permissions.

To place your repositories on a server that doesn’t already have accounts for everyone on your team, you must set up SSH access for them. I assume that you can already access the server using SSH yourself.

There are a few ways to give access to everyone on your team. The first is to set up accounts for everybody, which is straightforward but can be cumbersome. You may not want to run `adduser` and set temporary passwords for every user.

A second method is to create a single 'git' user, ask every user who needs write access to send you their SSH public key, and add that key to the new 'git' user’s `~/.ssh/authorized_keys` file. At that point, everyone will be able to access that machine as the 'git' user. This doesn’t affect the commit data in any way — the SSH user you connect as doesn’t affect user that Git sees as doing the commits.

Another way to do it is to have your SSH server authenticate from an LDAP server or some other centralized authentication source that you may already have set up. As long as each user can get shell access on the Git server, any SSH authentication mechanism you can think of should work.

## Generating Your SSH Key Pair ##

Many Git servers authenticate using SSH key pairs. Each user on your system must generate a key pair. The process for doing so is similar across systems using SSH.
First, make sure the user doesn’t already have a key pair. By default, a user’s SSH keys are stored in their `~/.ssh` directory. Check for SSH keys by listing the contents of that directory.

	$ cd ~/.ssh
	$ ls
	authorized_keys2  id_dsa       known_hosts
	config            id_dsa.pub

Look for a pair of files usually named `something` and `something.pub`, where `something` is usually `id_dsa` or `id_rsa`. The `.pub` file contains the public key, and the other file contains the private key. If you don’t see these files (or if you don’t even see a `.ssh` directory), the user can create them by running `ssh-keygen`, which is provided with the OpenSSH package on Linux/Mac systems and comes with the MSysGit package on Windows.

	$ ssh-keygen
	Generating public/private rsa key pair.
	Enter file in which to save the key (/Users/schacon/.ssh/id_rsa):
	Enter passphrase (empty for no passphrase):
	Enter same passphrase again:
	Your identification has been saved in /Users/schacon/.ssh/id_rsa.
	Your public key has been saved in /Users/schacon/.ssh/id_rsa.pub.
	The key fingerprint is:
	43:c5:5b:5f:b1:f1:50:43:ad:20:a6:92:6a:1f:9a:3a schacon@agadorlaptop.local

First, `ssh-keygen` confirms where to save the key (`.ssh/id_rsa`), and then asks twice for a passphrase, which can be left empty if the user doesn’t want to type a passphrase when using the key pair.

Now, each user that does this has to send their public key (the `.pub` file) to whoever is administering the Git server (assuming you’re using an SSH server setup that requires public keys). Public keys look something like

	$ cat ~/.ssh/id_rsa.pub
	ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEAklOUpkDHrfHY17SbrmTIpNLTGK9Tjom/BWDSU
	GPl+nafzlHDTYW7hdI4yZ5ew18JH4JW9jbhUFrviQzM7xlELEVf4h9lFX5QVkbPppSwg0cda3
	Pbv7kOdJ/MTyBlWXFCR+HAo3FXRitBqxiX1nKhXpHAZsMciLq8V6RjsNAQwdsdMFvSlVK/7XA
	t3FaoJoAsncM1Q9x5+3V0Ww68/eIFmb1zuUFljQJKprrX88XypNDvjYNby6vw/Pb0rwert/En
	mZ+AW4OZPnTPI89ZPmVMLuayrD2cE86Z/il8b+gw3r3+1nKatmIkjn2so1d01QraTlMqVSsbx
	NrRFi9wrf+M7Q== schacon@agadorlaptop.local

As the name implies, public keys are indeed public, so there’s no danger including them in email messages. You could even print them on the front page of your local newspaper and nothing bad would happen.
For a more in-depth tutorial on creating SSH keys on multiple operating systems, see the GitHub guide on SSH keys at `http://github.com/guides/providing-your-ssh-key`.

## Setting Up the Server ##

Let’s walk through setting up SSH access on the server. In this example, you’ll use the `authorized_keys` method for authenticating your users. I also assume you’re running a standard Linux distribution, like Ubuntu. First, create a 'git' user and an `.ssh` directory for that user.

	$ sudo adduser git
	$ su git
	$ cd
	$ mkdir .ssh

Next, add the SSH public keys for everyone who’ll be connecting using the 'git' user account to the `authorized_keys` file for the 'git' user. Let’s assume you’ve received a few public keys by e-mail and saved them in temporary files. Again, the public keys look something like

	$ cat /tmp/id_rsa.john.pub
	ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCB007n/ww+ouN4gSLKssMxXnBOvf9LGt4L
	ojG6rs6hPB09j9R/T17/x4lhJA0F3FR1rP6kYBRsWj2aThGw6HXLm9/5zytK6Ztg3RPKK+4k
	Yjh6541NYsnEAZuXz0jTTyAUfrtU3Z5E003C4oxOj6H0rfIF1kKI9MAQLMdpGW1GYEIgS9Ez
	Sdfd8AcCIicTDWbqLAcU4UpkaX8KyGlLwsNuuGztobF8m72ALC/nLF6JLtPofwFBlgc+myiv
	O7TCUSBdLQlgMVOFq1I2uPWQOkOWQAHukEOmfjy2jctxSDBQ220ymjaNsHT4kgtZg2AYYgPq
	dAv8JggJICUvax2T9va5 gsg-keypair

Just append them to the 'git' user’s `authorized_keys` file.

	$ cat /tmp/id_rsa.john.pub >> ~/.ssh/authorized_keys
	$ cat /tmp/id_rsa.josie.pub >> ~/.ssh/authorized_keys
	$ cat /tmp/id_rsa.jessica.pub >> ~/.ssh/authorized_keys

Now, create an empty repository by running `git init --bare`, which initializes a repository without a working directory.

	$ cd /opt/git
	$ mkdir project.git
	$ cd project.git
	$ git --bare init

Then, John, Josie, or Jessica can push the first version of their project into the new repository by adding it as a remote and pushing to it.

Note that someone must login to the server and create a bare repository every time you want to add a project. Let’s use `gitserver` as the hostname of the Git server. If you created a DNS entry for `gitserver`, then use the following commands:

	# on Johns computer
	$ cd myproject
	$ git init
	$ git add .
	$ git commit -m 'initial commit'
	$ git remote add origin git@gitserver:/opt/git/project.git
	$ git push origin master

At this point, the others can clone it and push changes back just as easily.

	# on others' computers
	$ git clone git@gitserver:/opt/git/project.git
	$ cd project
	$ vim README
	$ git commit -am 'fix for the README file'
	$ git push origin master

This method quickly provisions a read/write Git server for a handful of developers.

As an extra precaution, you can easily restrict the 'git' user to only running Git commands by using a limited shell called `git-shell` that comes with Git. If you set this as the 'git' user’s login shell, then the 'git' user doesn’t get normal shell access on the server. To do this, specify `git-shell` instead of bash or csh as your user’s login shell. To do so, you’ll likely have to edit your `/etc/passwd` file.

	$ sudo vim /etc/passwd

You should find a line that looks something like

	git:x:1000:1000::/home/git:/bin/sh

Change `/bin/sh` to `/usr/bin/git-shell` (or run `which git-shell` to see where `git-shell` is installed and use that path). The edited line should look something like

	git:x:1000:1000::/home/git:/usr/bin/git-shell

Now, the 'git' user can only use the SSH connection to push and pull Git repositories and can’t login to the machine. If they try, they’ll see a login rejection like

	$ ssh git@gitserver
	fatal: What do you think I am? A shell?
	Connection to gitserver closed.

## Public Access ##

What if you want to provide anonymous read access to your project? Perhaps instead of hosting an internal private project, you want to host an open source project. Or maybe you have a bunch of automated build servers or continuous integration servers that change a lot, and you don’t want to have to generate SSH keys all the time — you just want to allow simple anonymous read access.

Probably the simplest way for smaller setups to accomplish this is to run a static web server with its document root where your Git repository is, with the `post-update` hook that I mentioned in the first section of this chapter enabled. Let’s work from the previous example. Say you have your repository in the `/opt/git` directory, and an Apache server is running on your server. Again, you can use any web server for this but, as an example, I’ll show a basic Apache configuration that should give you an idea of what you might need.

First, you need to enable the hook.

	$ cd project.git
	$ mv hooks/post-update.sample hooks/post-update
	$ chmod a+x hooks/post-update

What does this `post-update` hook do? It looks basically like

	$ cat .git/hooks/post-update
	#!/bin/sh
	exec git-update-server-info

This means that when you push to the server via SSH, Git runs `git-update-server-info` to update the repository so that others can pull over HTTP.

Next, add a VirtualHost entry to your Apache configuration with the root directory of your Git projects as the DocumentRoot. Here, I’re assuming that you have DNS set up to direct `*.gitserver` to the server you’re using to run all this.

	<VirtualHost *:80>
	    ServerName git.gitserver
	    DocumentRoot /opt/git
	    <Directory /opt/git/>
	        Order allow, deny
	        allow from all
	    </Directory>
	</VirtualHost>

You’ll also need to set the Unix group of the `/opt/git` directory to `www-data` so your web server has read-access to the repository. That’s the group the Apache instance running the CGI script will (by default) be running as.

	$ chgrp -R www-data /opt/git

After restarting Apache, you should be able to clone your repository by specifying the URL for your project.

	$ git clone http://git.gitserver/project.git

Using this method you can set up HTTP-based read access to any of your Git projects in a few minutes. Another simple option for public unauthenticated access is to start a Git daemon. I’ll cover this option in the next section, if you prefer that route.

## GitWeb ##

Now that you’ve configured read/write and read-only access to your project, you can set up a simple web-based repository visualizer. Git comes with a CGI script called GitWeb that’s commonly used for this. See GitWeb in use at sites like `http://git.kernel.org` (see Figure 4-1).

Insert 18333fig0401.png
Figure 4-1. The GitWeb web-based user interface.

To check out what GitWeb would look like for your project, Git comes with a command to start a temporary web server if you have a lightweight web server already installed on your system, such as `lighttpd` or `webrick`. `lighttpd` is often installed on Linux machines, so you may be able to get it to run by running `git instaweb` in your project directory. If you’re running a Mac, OS X comes preinstalled with Ruby, so `webrick` may be your best bet. To tell `instaweb` to use something other than `lighttpd`, run it with the `--httpd` option.

	$ git instaweb --httpd=webrick
	[2009-02-21 10:02:21] INFO  WEBrick 1.3.1
	[2009-02-21 10:02:21] INFO  ruby 1.8.6 (2008-03-03) [universal-darwin9.0]

This starts a web server on port 1234 and then automatically starts a web browser that opens on the GitWeb page. This is all you have to do. When you’re done and want to shut down the server, run the same command with the `--stop` option.

	$ git instaweb --httpd=webrick --stop

To run the web interface all the time, set up the CGI script that creates the output so that it’s run by your normal web server. Some Linux distributions have a `gitweb` package that you may be able to install via `apt` or `yum`, so you may want to try that first. I’ll walk though installing GitWeb manually very quickly. First, get the Git source code, which includes GitWeb, and generate the custom CGI script.

	$ git clone git://git.kernel.org/pub/scm/git/git.git
	$ cd git/
	$ make GITWEB_PROJECTROOT="/opt/git" \
	        prefix=/usr gitweb
	$ sudo cp -Rf gitweb /var/www/

Notice that you have to tell the `make` command where to find your Git repositories by assigning a value to the `GITWEB_PROJECTROOT` environment variable. Now, make Apache use CGI to run that script. To do this, add a VirtualHost section.

	<VirtualHost *:80>
	    ServerName gitserver
	    DocumentRoot /var/www/gitweb
	    <Directory /var/www/gitweb>
	        Options ExecCGI +FollowSymLinks +SymLinksIfOwnerMatch
	        AllowOverride All
	        order allow,deny
	        Allow from all
	        AddHandler cgi-script cgi
	        DirectoryIndex gitweb.cgi
	    </Directory>
	</VirtualHost>

Again, GitWeb can be run by any CGI capable web server. If you prefer something else, it shouldn’t be difficult to set up. At this point, you should be able to visit `http://gitserver/` to view your repositories online, and use `http://git.gitserver` to clone and fetch your repositories over HTTP.

## Gitosis ##

Keeping all users’ public keys in the `authorized_keys` file for limiting access works well only up to a point. When you have hundreds of users, this technique becomes painful.  Making a change requires logging in to the Git server. Plus, there’s no access control — everyone whose public key is in `authorized_keys` has read and write access to every project.

At this point, you may consider turning to a widely used software project called Gitosis. This is basically a set of scripts that helps manage the `authorized_keys` file and implements simple access controls. The really interesting part is that the UI for this tool isn’t a web interface, but rather a special Git repository. You set up user access and control in that repository so when you push it, Gitosis reconfigures the Git server based on what’s in the repository. This is very clever.

Installing Gitosis isn’t the simplest task ever, but it’s not too difficult. It’s easiest to run Gitosis on a Linux server — these examples use a stock Ubuntu 8.10 server.

Gitosis requires the Python setuptools package, which Ubuntu calls python-setuptools.

	$ apt-get install python-setuptools

Next, clone and install Gitosis from the Gitosis project’s main site.

	$ git clone git://eagain.net/gitosis.git
	$ cd gitosis
	$ sudo python setup.py install

That installs several executables for Gitosis to use. Gitosis expects its repositories to be under `/home/git`. But you’ve already set up your repositories in `/opt/git`. Instead of reconfiguring everything, create a symlink.

	$ ln -s /opt/git /home/git/repositories

Gitosis is going to manage your SSH keys, so remove the current `authorized_keys` file, and let Gitosis control it automatically. For now, move it out of the way.

	$ mv /home/git/.ssh/authorized_keys /home/git/.ssh/ak.bak

Next, restore the normal shell for the 'git' user if you’ve changed the shell to `git-shell`. Users still won’t be able to log in, but Gitosis will take care of that. So, change the line in your `/etc/passwd` file

	git:x:1000:1000::/home/git:/usr/bin/git-shell

back to

	git:x:1000:1000::/home/git:/bin/sh

Now, it’s time to initialize Gitosis. Run the `gitosis-init` command with the public key of the user you want to manage Gitosis as input.

	$ sudo -H -u git gitosis-init < /tmp/id_dsa.pub
	Initialized empty Git repository in /opt/git/gitosis-admin.git/
	Reinitialized existing Git repository in /opt/git/gitosis-admin.git/

This lets the user whose public key you gave modify the main Git repository that controls the Gitosis setup. Next, set the execute bit on the `post-update` script in your new control repository.

	$ sudo chmod 755 /opt/git/gitosis-admin.git/hooks/post-update

You’re ready to roll. If you’re set up correctly, try to SSH into your Git server as the user whose public key you added when initializing Gitosis. You should see something like

	$ ssh git@gitserver
	PTY allocation request failed on channel 0
	fatal: unrecognized command 'gitosis-serve schacon@quaternion'
	  Connection to gitserver closed.

That means Gitosis recognized you but blocked you because you’re not trying to run any Git commands. So, let’s run an actual Git command to clone the Gitosis repository.

	# on your local computer
	$ git clone git@gitserver:gitosis-admin.git

Now you have a directory named `gitosis-admin`, which has two parts.

	$ cd gitosis-admin
	$ find .
	./gitosis.conf
	./keydir
	./keydir/scott.pub

`gitosis.conf` is the control file that specifies users, repositories, and permissions. The `keydir` directory is where you store the public keys of all the users who need access to your repositories — one file per user. The name of the file in `keydir` (`scott.pub` above) will be different for you — Gitosis uses the name from the description at the end of the public key that was imported with the `gitosis-init` script.

If you look at `gitosis.conf`, it should only specify information about the `gitosis-admin` project that you just cloned.

	$ cat gitosis.conf
	[gitosis]

	[group gitosis-admin]
	writable = gitosis-admin
	members = scott

It shows that 'scott' — the user whose public key you used to initialize Gitosis — is the only user with access to the `gitosis-admin` project.

Now, let’s add a new project. Add a new section called `mobile` which lists the developers on your mobile team and projects that they need access to. Because 'scott' is the only user in the system right now, add him as the only member, and create a new project called `iphone_project` to start on.

	[group mobile]
	writable = iphone_project
	members = scott

Whenever you make changes to the `gitosis-admin` project, commit the changes and push to the Git server in order for the changes to take effect.

	$ git commit -am 'add iphone_project and mobile group'
	[master]: created 8962da8: "changed name"
	 1 files changed, 4 insertions(+), 0 deletions(-)
	$ git push
	Counting objects: 5, done.
	Compressing objects: 100% (2/2), done.
	Writing objects: 100% (3/3), 272 bytes, done.
	Total 3 (delta 1), reused 0 (delta 0)
	To git@gitserver:/opt/git/gitosis-admin.git
	   fb27aec..8962da8  master -> master

Make your first push to the new `iphone_project` project by adding your Git server as a remote to your local version of the project and then pushing to the remote. You no longer have to manually create a bare repository for new projects on the server — Gitosis creates them automatically when it sees the first push.

	$ git remote add origin git@gitserver:iphone_project.git
	$ git push origin master
	Initialized empty Git repository in /opt/git/iphone_project.git/
	Counting objects: 3, done.
	Writing objects: 100% (3/3), 230 bytes, done.
	Total 3 (delta 0), reused 0 (delta 0)
	To git@gitserver:iphone_project.git
	 * [new branch]      master -> master

Notice that you don’t need to specify the path (in fact, doing so won’t work), just a colon and then the name of the project — Gitosis finds the path for you.

You want to work on this project with your friends, so you’ll have to re-add their public keys. But instead of appending them manually to the `~/.ssh/authorized_keys` file on your server, add them, one key per file, into the `keydir` directory. How you name the keys determines how you refer to the users in `gitosis.conf`. Let’s re-add the public keys for John, Josie, and Jessica.

	$ cp /tmp/id_rsa.john.pub keydir/john.pub
	$ cp /tmp/id_rsa.josie.pub keydir/josie.pub
	$ cp /tmp/id_rsa.jessica.pub keydir/jessica.pub

Now, add them all to your 'mobile' team so they have read and write access to `iphone_project`.

	[group mobile]
	writable = iphone_project
	members = scott john josie jessica

After you commit and push that change, all four users will have read and write access to that project.

Gitosis has simple access controls as well. For John to have only read access to `iphone_project`, do this instead.

	[group mobile]
	writable = iphone_project
	members = scott josie jessica

	[group mobile_ro]
	readonly = iphone_project
	members = john

Now John can clone the project and get updates, but Gitosis won’t allow him to push to the project. Create as many of these groups as you want, each containing different users and projects. You can also specify another group as a member (using `@` as prefix), to include all of the members automatically.

	[group mobile_committers]
	members = scott josie jessica

	[group mobile]
	writable  = iphone_project
	members   = @mobile_committers

	[group mobile_2]
	writable  = another_iphone_project
	members   = @mobile_committers john

If you have any problems, add `loglevel=DEBUG` in the `[gitosis]` section. If you’ve lost push access by pushing a messed-up configuration, manually fix the file on the server under `/home/git/.gitosis.conf` — the file from which Gitosis reads its configuration. A push to the project takes the `gitosis.conf` file you just pushed and sticks it there. If you edit that file manually, it retains your modifications until the next successful push to the `gitosis-admin` project.

## Gitolite ##

This section serves as a quick introduction to Gitolite, and provides basic installation and setup instructions.  It can’t, however, replace the enormous amount of [documentation][gltoc] that comes with Gitolite .  There may also be occasional changes to this section itself, so you may also want to look at the latest version of the documentation [here][gldpg].

[gldpg]: http://sitaramc.github.com/gitolite/progit.html
[gltoc]: http://sitaramc.github.com/gitolite/master-toc.html

Gitolite is an authorization layer on top of Git, relying on `sshd` or `httpd` for authentication.  (Recap: authentication is identifying who the user is, authorization is deciding what he is allowed to do).

Gitolite allows specifying permissions not just by repository, but also by branch or tag names within each repository.  That is, you can specify that certain people (or groups of people) can only push certain "refs" (branches or tags) but not others.

### Installing ###

Installing Gitolite is very easy, even if you don’t read the extensive documentation that comes with it.  You need an account on a Unix server of some kind.  You do not need root access, assuming Git, Perl, and an OpenSSH compatible SSH server are already installed.  In the examples below, I’ll use the `git` account on a host called `gitserver`.

Gitolite is somewhat unusual as far as "server" software goes — access is via SSH, and so every user on the Git server is a potential "gitolite host".  I’ll describe the simplest installation method in this article. For the other methods please see the documentation.

To begin, create a user called `git` on your Git server and login as this user.  Copy your SSH public key from your workstation, naming it `<yourname>.pub` (I’ll use `scott.pub` in the examples).  Then, run these commands:

	$ git clone git://github.com/sitaramc/gitolite
	$ gitolite/install -ln
	# assumes $HOME/bin exists and is in your $PATH
	$ gitolite setup -pk $HOME/scott.pub

That last command creates a new Git repository called `gitolite-admin` on the Git server.

Finally, back on your workstation, run `git clone git@gitserver:gitolite-admin`. You’re done!  Gitolite has now been installed on the server, and you now have a brand new repository called `gitolite-admin` in your workstation. Administer your Gitolite setup by making changes to this repository and pushing.

### Customizing the Install ###

While the default quick installation works for most people, there are some ways to customize it.  You can make some changes by editing `gitolite.conf`, but if that isn’t sufficient, there’s documentation on customizing Gitolite.

### Config File and Access Control Rules ###

Once the install is done, switch to the `gitolite-admin` clone you just made, and poke around to see what you have.

	$ cd ~/gitolite-admin/
	$ ls
	conf/  keydir/
	$ find conf keydir -type f
	conf/gitolite.conf
	keydir/scott.pub
	$ cat conf/gitolite.conf

	repo gitolite-admin
	    RW+                 = scott

	repo testing
	    RW+                 = @all

Notice that "scott" (the name of the public key in the `gitolite setup` command you used earlier) has read-write permissions on the `gitolite-admin` repository as well as a public key file of the same name.

Adding users is easy.  To add a user called "alice", obtain her public key, name it `alice.pub`, and put it in your `keydir` directory in the `gitolite-admin` repository you just made.  Add, commit, and push the change, and "alice" has been added.

The config file syntax for Gitolite is well documented, so I’ll only mention some highlights here.

You can group users or repositories  for convenience.  The group names are just like macros. When defining them, it doesn’t even matter whether they’re projects or users. That distinction is only made when you *use* the "macro".

	@oss_repos      = linux perl rakudo git gitolite
	@secret_repos   = fenestra pear

	@admins         = scott
	@interns        = ashok
	@engineers      = sitaram dilbert wally alice
	@staff          = @admins @engineers @interns

You can control permissions at the "ref" level.  In the following example, interns can only push the "int" branch.  Engineers can push any branch whose name starts with "eng-", and tags that start with "rc" followed by a digit. Admins can do anything (including rewind) to any ref.

	repo @oss_repos
	    RW  int$                = @interns
	    RW  eng-                = @engineers
	    RW  refs/tags/rc[0-9]   = @engineers
	    RW+                     = @admins

The expression after the `RW` or `RW+` is a regular expression (regex) that the refname (ref) being pushed is matched against.  So I call it a "refex"!  Of course, a refex can be far more powerful than shown here, so don’t overdo it if you’re not comfortable with Perl regexes.

Also, as you probably guessed, Gitolite prefixes the refex with `refs/heads/` as a syntactic convenience if the refex doesn’t begin with `refs/`.

An important feature of the config file’s syntax is that all the rules for a repository need not be in one place. Keep all the common stuff together, like the rules for all `oss_repos` shown above, then add specific rules for specific cases later on, like

	repo gitolite
	    RW+                     = sitaram

That rule will just get added to the ruleset for the `gitolite` repository.

At this point you might be wondering how the access control rules are actually applied, so I’ll go over that briefly.

There are two levels of access control in Gitolite.  The first is at the repository level. If you have read (or write) access to *any* ref in the repository, then you have read (or write) access to the repository.  This is the only access control that Gitosis had.

The second level, applicable only to "write" access, is by branch or tag within a repository.  The username, the access being attempted (`W` or `+`), and the refname being updated are known.  The access rules are checked in order of appearance in the config file, looking for a match for this combination (but remember that the refname is regex-matched, not merely string-matched).  If a match is found, the push succeeds.  A fallthrough results in access being denied.

### Advanced Access Control with "deny" rules ###

So far, you’ve only seen `R`, `RW`, or `RW+` permissions.  However, Gitolite allows another permission: `-`, standing for "deny".  This gives a lot more power, at the expense of some complexity, because now fallthrough is not the *only* way for access to be denied, so the *order of the rules now matters*!

Let’s say, in the situation above, I want engineers to be able to rewind any branch *except* `master` and `integ`.  Here’s how to do that:

	    RW  master integ    = @engineers
	    -   master integ    = @engineers
	    RW+                 = @engineers

Again, simply follow the rules top down until you hit a match for your access mode, or a deny.  Non-rewind push to `master` or `integ` is allowed by the first rule.  A rewind push to those refs doesn’t match the first rule, drops down to the second, and is therefore denied.  Any push (rewind or non-rewind) to refs other than `master` or `integ` won’t match the first two rules anyway, and the third rule allows it.

### Restricting pushes by files changed ###

In addition to restricting the branches a user can push changes to, you can also restrict what files they’re allowed to touch.  For example, perhaps `Makefile` shouldn’t be changed by just anyone, because a lot of things depend on it or would break if the changes are not done *just right*. Tell Gitolite.

    repo foo
        RW                      =   @junior_devs @senior_devs

        -   VREF/NAME/Makefile  =   @junior_devs

Users migrating from older Gitolite versions should note that this feature’s behavior has changed significantly. Please see the migration guide for details.

### Personal Branches ###

Gitolite also has a feature called "personal branches" (or rather, "personal branch namespaces") that can be very useful in a corporate environment.

A lot of code exchange in the Git world happens by "please pull" requests. In a corporate environment, however, unauthenticated access to a server isn’t allowed, and developer workstations can’t do authentication, so you have to push to a central server and then ask someone to pull from there.

This would normally cause the same branch name clutter as in a CVCS. Plus setting up permissions on the central server becomes a chore for the admin.

Gitolite lets you define a "personal" or "scratch" namespace prefix for each developer (for example, `refs/personal/<devname>/*`). See the documentation for details.

### "Wildcard" repositories ###

Gitolite allows specifying repositories using wildcards (actually Perl regexes), like, for example `assignments/s[0-9][0-9]/a[0-9][0-9]`.  It also allows assigning the permission mode (`C`) which permits creating repositories based on such wild cards, automatically assigning ownership to the specific user who created the repository, and allowing them to hand out `R` and `RW` permissions to other users to collaborate. Again, please see the documentation for details.

### Other Features ###

I’ll round off this discussion with a sample of other features, all of which, and many others, are described in great detail in the documentation.

**Logging**: Gitolite logs all successful accesses.  If you’re somewhat relaxed about giving people rewind permissions (`RW+`) and some kid blows away `master`, the log file is a life saver by easily and quickly showing the SHA-1 hash that got hosed.

**Access rights reporting**: Another convenient feature is what happens when you try to just ssh to the server.  Gitolite shows what repositories you have access to, and what access you have.  Here’s an example.

        hello scott, this is git@git running gitolite3 v3.01-18-g9609868 on git 1.7.4.4

             R     anu-wsd
             R     entrans
             R  W  git-notes
             R  W  gitolite
             R  W  gitolite-admin
             R     indic_web_input
             R     shreelipi_converter

**Delegation**: In really large installations, Gitolite can delegate responsibility for groups of repositories to various administrators to manage independently.  This reduces the load on the main administrator, and makes him less of a bottleneck.

**Mirroring**: Gitolite can help maintain multiple mirrors, and switch between them easily if the primary server goes down.

## Git Daemon ##

For public unauthenticated read access to your projects, you’ll want to move past the HTTP protocol and start using the Git protocol. The main reason is speed. The Git protocol is far more efficient and faster than the HTTP protocol.

Again, this is for unauthenticated read-only access. Running this on a server outside your firewall should only be done for projects that are publicly visible to the world. If the server you’re running it on is inside your firewall, you might use it for projects that a large number of people or computers (continuous integration or build servers) have read-only access to, when you don’t want to have to add an SSH key for each user.

In any case, the Git protocol is relatively easy to set up. Basically, run this command to start a Git daemon.

	git daemon --reuseaddr --base-path=/opt/git/ /opt/git/

`--reuseaddr` allows the server to restart without waiting for old connections to time out, `--base-path` allows cloning without specifying a project’s entire path, and the path at the end tells the Git daemon where to look for repositories to export. If you’re behind a firewall, you must also open access to port 9418 on the box you’re setting this up on.

You can daemonize this process a number of ways, depending on your operating system. On an Ubuntu machine, use an Upstart script. So, put this script in '/etc/event.d/local-git-daemon'.

	start on startup
	stop on shutdown
	exec /usr/bin/git daemon \
	    --user=git --group=git \
	    --reuseaddr \
	    --base-path=/opt/git/ \
	    /opt/git/
	respawn

For security reasons, you’re strongly encouraged to run this daemon as a user with read-only permissions to the repositories. Do this by creating a new user 'git-ro' and running the daemon as this user.  For the sake of simplicity I’ll simply run it as the same 'git' user that Gitosis is running as.

When you restart your server, your Git daemon will start automatically and respawn if it goes down. To start the daemon without having to reboot, run

	initctl start local-git-daemon

On other systems, you may want to use `xinetd`, a script in your `sysvinit` system, or something else — as long as you get the `git` command daemonized and set up to restart automatically.

Next, tell your Gitosis server which repositories to allow unauthenticated Git server-based access to. If you add a section for each repository, specify the repository your Git daemon should allow reading from. To allow Git protocol access for the `iphone_project`, add this to the end of the `gitosis.conf` file:

	[repo iphone_project]
	daemon = yes

When that’s committed and pushed, your running daemon should start serving requests for the project to anyone who has access to port 9418 on your Git server.

If you decide not to use Gitosis, but you want to set up a Git daemon, run the following commands on each project the Git daemon should serve:

	$ cd /path/to/project.git
	$ touch git-daemon-export-ok

The presence of the `git-daemon-export-ok` file tells Git that it’s OK to serve this project without authentication.

Gitosis can also control which projects GitWeb shows. First, add something like the following to `/etc/gitweb.conf`:

	$projects_list = "/home/git/gitosis/projects.list";
	$projectroot = "/home/git/repositories";
	$export_ok = "git-daemon-export-ok";
	@git_base_url_list = ('git://gitserver');

Control which projects GitWeb lets users browse by adding or removing a `gitweb` setting in the Gitosis configuration file. For instance, for `iphone_project` to show up on GitWeb, make the `repo` setting look like

	[repo iphone_project]
	daemon = yes
	gitweb = yes

Now if you commit and push the project, GitWeb will automatically start showing the `iphone_project`.

## Hosted Git ##

If you don’t want to go through all of the work involved in setting up your own Git server, you have several options for hosting your Git projects on an external dedicated hosting site. Doing so offers a number of advantages: a hosting site is generally quick to set up and easy to start projects on, and no server maintenance or monitoring is involved. Even if you set up your own server internally, you may still want to use a public hosting site for your open source projects — it’s generally easier for the open source community to collaborate.

These days you have a huge number of hosting options to choose from, each with different advantages and disadvantages. To see an up-to-date list, check out the following page:

	https://git.wiki.kernel.org/index.php/GitHosting

Because I can’t cover all of them, I’ll use this section to walk through setting up an account and creating a new project at GitHub, which is where I work. This will give you an idea of what’s involved.

GitHub is by far the largest open source Git hosting site and it’s also one of the very few that offers both public and private hosting options so you can keep your open source and private commercial code in the same place. In fact, I used GitHub to privately collaborate on this book.

### GitHub ###

GitHub is slightly different than most code-hosting sites in the way it uses namespaces in projects. Instead of being primarily based on the project, GitHub is user-centric. That means when I host my `grit` project on GitHub, you won’t find it at `github.com/grit` but instead at `github.com/schacon/grit`. There’s no standard name for any project, which allows a project to move from one user to another seamlessly if the first author abandons the project.

GitHub is also a commercial company that charges for accounts for maintaining private repositories, but anyone can quickly get a free account to host as many open source projects as they want. I’ll quickly go over how that’s done.

### Setting Up a User Account ###

The first thing to do is to create a free user account. Visit the Pricing and Signup page at `http://github.com/plans` and click the "Create a free account" button on the right hand side of the page (see Figure 4-2). You’re taken to the signup page.

Insert 18333fig0402.png
Figure 4-2. The GitHub plan page.

Here you must choose a username that isn’t yet taken and enter an e-mail address and password that will be associated with the account (see Figure 4-3).

Insert 18333fig0403.png
Figure 4-3. The GitHub user signup form.

If you have your public SSH key available, this is a good time to add it. I described how to generate a new SSH key pair earlier in the "Simple Setups" section. Take the contents of the public key of that pair, and paste it into the SSH Public Key text box. Clicking the "explain ssh keys" link takes you to detailed instructions on how to create SSH key pairs on all major operating systems.
Clicking the "I agree, sign me up" button takes you to your new user dashboard (see Figure 4-4).

Insert 18333fig0404.png
Figure 4-4. The GitHub user dashboard.

Next, create a new repository.

### Creating a New Repository ###

Start by clicking the "create a new one" link next to "Your Repositories" on the user dashboard. You’re taken to the "Create a New Repository" form (see Figure 4-5).

Insert 18333fig0405.png
Figure 4-5. Creating a new repository on GitHub.

All you really have to do is provide a project name, but you can also add a description. When that’s done, click the "Create Repository" button. Now you have a new repository on GitHub (see Figure 4-6).

Insert 18333fig0406.png
Figure 4-6. GitHub project header information.

Since you have no code there yet, GitHub will show instructions for how to create a brand-new Git project, push an existing project, or import a project from a public Subversion repository (see Figure 4-7).

Insert 18333fig0407.png
Figure 4-7. Instructions for a new repository.

These instructions are similar to what I’ve already gone over. To initialize a project that isn’t already a Git project, run

	$ git init
	$ git add .
	$ git commit -m 'initial commit'

When you have a Git repository locally, add GitHub as a remote and push your master branch.

	$ git remote add origin git@github.com:testinguser/iphone_project.git
	$ git push origin master

Now your project is hosted on GitHub, and you can give the URL to anyone you want to share your project with. In this case, it’s `http://github.com/testinguser/iphone_project`. You can also see from the header on each of your project’s pages that you have two Git URLs (see Figure 4-8).

Insert 18333fig0408.png
Figure 4-8. Project header with a public URL and a private URL.

The Public Clone URL is a public, read-only Git URL that anyone can use to clone the project. Feel free to give out that URL and post it wherever you want.

The Your Clone URL is a read/write SSH-based URL that you use to read or write only if you connect with the SSH private key associated with the public key you uploaded for your account. When other users visit this project page, they won’t see that URL — only the public one.

### Importing from Subversion ###

If you have an existing public Subversion project to import into Git, GitHub can often do most of the work for you. At the bottom of the instructions page is a link to Subversion import. If you click on it, you see a form with information about the import process and a text box where you paste the URL of your public Subversion project (see Figure 4-9).

Insert 18333fig0409.png
Figure 4-9. Subversion importing interface.

If your project is very large, nonstandard, or private, this process probably won’t work. In Chapter 7, you’ll learn how to manually import more complicated projects.

### Adding Collaborators ###

Let’s add the rest of the team. If John, Josie, and Jessica all sign up for accounts on GitHub, and you want to give them push access to your repository, add them to your project as collaborators. Doing so will allow pushes to your repository from accounts with their public keys to work.

Click the "edit" button in the project header or the "Admin" tab at the top of the project to reach the "Admin" page of your GitHub project (see Figure 4-10).

Insert 18333fig0410.png
Figure 4-10. GitHub administration page.

To give another user write access to your project, click the “Add another collaborator” link. A new text box appears into which you type a GitHub username. As you type, a helper pops up, showing possible matches with other GitHub accounts. When you find the correct user, click the "Add" button to add that user as a collaborator to your project (see Figure 4-11).

Insert 18333fig0411.png
Figure 4-11. Adding a collaborator to your project.

When you’re finished adding collaborators, you should see a collaborator list in the "Repository Collaborators" box (see Figure 4-12).

Insert 18333fig0412.png
Figure 4-12. A list of collaborators on your project.

To revoke access to an individual, click the "revoke" link, and their push access will be removed. For future projects, you can also copy collaborator groups by copying the permissions of an existing project.

### Your Project ###

After you push your project or import it from Subversion, you have a main project page that looks something like Figure 4-13.

Insert 18333fig0413.png
Figure 4-13. A GitHub main project page.

When people visit your project, they see this page. It contains tabs to different aspects of your projects. The "Commits" tab shows a list of commits in reverse chronological order, similar to the output of the `git log` command. The "Network" tab shows all the people who have forked your project and contributed back. The "Downloads" tab allows you to upload project binaries and link to tarballs and zipped versions of any tagged points in your project. The "Wiki" tab provides a wiki where you write documentation or other information about your project. The "Graphs" tab has contribution visualizations and statistics about your project. The main "Source" tab that you land on shows your project’s main directory listing and automatically renders the README file below it, if you have one. This tab also shows a box with the latest commit information.

### Forking Projects ###

To contribute to an existing project to which you don’t have push access, GitHub encourages forking the project. When you land on a project page that looks interesting and you want to hack on it, click the "fork" button in the project header to have GitHub copy that project to your account so you can push to it after making changes.

This way, projects don’t have to worry about adding users as collaborators to give them write access. If people fork a project and push to it after making changes, then the project maintainer can pull those changes by adding the forks as remotes and merging the work they contain.

To fork a project, visit the project page (in this case, mojombo/chronic) and click the "fork" button in the header (see Figure 4-14).

Insert 18333fig0414.png
Figure 4-14. Get a writable copy of any repository by clicking the "fork" button.

After a few seconds, you’re taken to your new project page, which indicates that this project is a fork of another one (see Figure 4-15).

Insert 18333fig0415.png
Figure 4-15. Your fork of a project.

### GitHub Summary ###

That’s all I’ll mention about GitHub, but it’s important to note how quickly you can do all this. You can create an account, add a new project, and push to it in a matter of minutes. If your project is open source, you also get a huge community of developers who now have visibility into your project and may well fork it and help contribute to it. At the very least, this may be a way to get up and running so you can try using Git to maintain it.

## Summary ##

There are several ways to get a remote Git repository up and running to collaborate with others or share your work.

Running your own server gives you complete control and allows you to run the server within your own firewall. But, such a server generally requires a significant amount of time to set up and maintain. Placing your data on a hosted server is easy to set up and maintain. However, you have to be allowed to keep your code on someone else’s server, which some organizations don’t allow.

It should be fairly straightforward to determine which solution or combination of solutions is appropriate for you and your organization.
