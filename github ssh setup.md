github ssh setup.md

A quick note: I'm switching between my three lab servers in the examples below. The are lab97 (FreeBSD), lab98 (MacOS), and lab99 (Linux). Be sure to adapt the examples to your system.

## Github SSH Key Persistent Connectivity

A little ancient background is that some years ago GitHub disabled git operations via http/https with username and password authentication. So now what do we do? We setup SSH key based authentication. The syntax for specifying your github site is a little different too. We'll talk about this below in the **Test and Enjoy The Results** section.

This is all for Linux/Unix derivatives such as Linux, FreeBSD, and Darwin/MacOS. This should all work fine whether you use BASH or ZSH. I won't be touching on native Windows methods because Windows drives me nuts with its SSH and PowerShell. You can install a git shell that will be a bit slow, but otherwise works great. You could probably do the steps below with Linux subsystems as well. You may even get CygWin to work.

### Generate Keys

It's maddeningly difficult to get timely information about what ciphers and key exchange algorithms Git currently supports. I'll just use a contemporary cipher that works with modern OSes and GitHub as of this writing. In the near future you will need to migrate to quantum resistant methods, aka post-quantum key exchange. It's a brave new world. Good luck.

Let's make some ssh keys. This works on Linux, MacOS, and FreeBSD. I'm sure it will work on nearly any \*nix derivative.  
`ssh-keygen -t ed25519 -f ~/.ssh/git-SparkyzCodez_lab98`

I don't like to use default names for the keys because I tend to have a lot of them. Use a descriptive name and save your self some grief.

I am not using a password with my keys. I'm satisfied with the security. If you do use passwords then you will need to modify some of the steps below to accomodate them.

In case you're new to ssh keys, or encryption keys in general, just know that the private key is yours alone. No matter what scenario you run into you never share that thing. Look in the text files that ssh-keygen created. It will have the basic format:
```
-----BEGIN OPENSSH PRIVATE KEY-----
xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
...
xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
-----END OPENSSH PRIVATE KEY-----
```
Obviously you can see the words PRIVATE KEY. There are some other key formats that may look a little different. Just know that you gotta keep these very safe.

In contrast, the public key is completely shareable. It only matches your private key and your private key cannot be reverse engineered from it. How confident am I? Here's the public key I'm using from the above example. Do what you like with it.

`ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIA53cVMhqM5h+PPKOA9nlDYuMNaWTxifES8wJkzP2C43 anscontrol@Sparkys-Mac.local`

### Setup Github To Accept Your Connection

This is the generalized method that will apply to all your repos. You can also do this to individual repos.

Do these steps:  
- log into your github
- go to settings by clicking your picture/avatar at the top right corner
- click on the settings option
- goto SSH and GPG keys
- click on New SSH Key
- give the new key a descriptive name and paste in your public key

You may need to login again or use your MFA to prove it's really you. This is serious business, so this is completely appropriate.

Test from your workstation like this. We'll specificy the exact key to use so your local ssh service doesn't need additional configuration yet.  
`ssh -i ~/.ssh/git-SparkyzCodez_lab98 -T git@github.com`

-i is the path to your private key
-T Disable pseudo-terminal allocation

This is a pass/fail test. If it fails then keep working on your keys until you get them right before proceeding.


### Setup Your SSH To Use The Private Key With Git

We have a number of different methods we could use. The two most common are adding your private key to your active SSH service to use automaticall and creating a git config that knows which key to use. I'm kind of mixed on which one is best. The SSH service is just easy. If you already use that for other ssh activities then it's probably the best to use.

On the other hand, configuring git to use the correct key keeps the configuration for git localized to git and no other applications. Limiting scope and keeping configuration items in one place is always a good idea.

Your system ssh, such as the .ssh/config file setup we used above, has the lowest precedence. The other methods listed below are in order of precedence, lower to higher. Each high precedence method will override the previous lower precedence method. You may use then all together.

#### System-wide SSH Service Method

First we'll show how to use the simpler system-wide SSH service. We will use the traditional methods and not more modern keychains because I don't want to install any software if you're not already using keychains.

Let's start with making sure our ssh service is running. (If not then you need to fix that first.) Sometimes there are minor variations in service name. Some systems call is SSH and others SSHD.

Linux SystemD systems:  
`systemctl status ssh`

FreeBSD systems:  
`service sshd status`

MacOS (just has to be different) systems:  
`systemsetup -getremotelogin`


This alternate test should work just on just about any \*nix system. As always, MacOS may need a bit of extra interpretation.  
`ps aux | grep ssh`


The basic procedure is that we add our private key to the running service then place a few instructions in a config file. It's (too) easy to use a `*` to apply to all hosts. We'll limit these isntructions to just affect connections to Git. You may tweak this if you some other URL. Just try to avoid overbroad wildcards. They're sloppy.

A quick note about permissions. SSH config files must only be readable by the user, and not by a group or the world. That's 600 or 0600. This will fix it:  
`chmod u=rw,go= ~/.ssh/config`

Linux, Mac, and FreeBSD (assumes to passwords on keys):  
```
Host github.com
  AddKeysToAgent yes
  IdentityFile ~/.ssh/git-SparkyzCodez_lab99
```

Now we test like we did above but without specifying the key location:
`ssh -T git@github.com`

Once again you should get the message: `You've successfully authenticated, but GitHub does not provide shell access.`

#### Git Global Key Configuration

This method relies on a Git configuration file. It does nothing for general ssh connections.

This is really easy if you want to config all your repositories to use the same key file. Just do this (assumes git is installed):
`git config --global core.sshCommand "ssh -i ~/.ssh/git-SparkyzCodez_lab99"`

Testing is easy too. Just clone my public Sparkyz Notez repository:
`git clone git@github.com:SparkyzCodez/Sparkyz-Notez.git`


#### Git Per-repository Key Configuration

You may not want a global configuration or you may want different keys for specific repositories. You may set the key used per-repository. It will override the global settings too.

Doing this takes two steps. First you need a repository. You may init a new repository or clone an existing one first. We'll focus on cloning while specifying the correct key to do so.
`git clone -c core.sshCommand="ssh -i ~/.ssh/git-SparkyzCodez_lab97" git@github.com:SparkyzCodez/Sparkyz-Notez.git`

Now cd into the repository you just cloned/created and add the ssh key to its configuration.
`git config core.sshCommand "ssh -i ~/.ssh/git-SparkyzCodez_lab97 -F /dev/null"`

-i specifies the key to use
-F /dev/null tells the command to ignore the ~/.ssh/config file

#### Git Environment Variable

If you use the `GIT_SSH_COMMAND` environment variable it will override everything else. It has the highest precedence. This especially nice if you have scripts that do git commands and need to switch between different repositories. Basically, I think it's best for temporary settings.

