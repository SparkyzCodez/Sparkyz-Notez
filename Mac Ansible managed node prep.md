Mac Ansible managed node prep.md

## Fast Setup To Manage Mac Nodes With Ansible

These basic steps have been tested on MacOS 14, 15, and 26 (Sononma, Sequoia, and Tahoe respectively).

The steps to do from the Desktop and local Terminal session are:
- enable remote logins
- enable root usage
- enable root login via ssh (temporarily)
- install Python

### A Few Notes Before We Start
These instructions assume you are comfortable in Unix/Linux shells and running commands at the command line.

Login to your Mac as a non-priveleged but sudo permitted user. On MacOS that just means login as a user that is an administrator. We'll need sudo permissions elevation for a few of the steps.

The default shell for root is /bin/sh. Don't change that. The default shell for other users is /bin/zsh. That will almost always work like bash for our purpose. You may want to change that to bash for your Ansible control login account if you plan to run more advanced shell scripts via Ansible. zsh is a great shell but sometimes we rely on specific shell quirks without even realizing it.

We'll be using root with Ansible only long enough to bootstrap a non-priveleged but sudo allowed user account that Ansible will use for all initial connections.

MacOS does not have a default installation of Python, but Ansible must have a supported version of Python to function. Without it even ansible.builtin.ping will fail. The only Ansible module that can function without Python is the ansible.builtin.raw module, but it's extremely limited.



### Configuration Steps


#### First we'll enable remote connections via SSH. This is the default connection method for Ansible.

Go to system settings -> General -> Sharing (scroll to the bottom) -> enable Remote Login

Now open the Terminal application (open a shell). We'll do everything else in a shell session.

And real quick, just in case you're a beginer let's just make sure you have sudo rights. Do this:
`sudo whoami`

The correct answer is: root


#### Now we enable the root account
I imagine there are ways to manage a Mac with Ansible without enabling root, but I'm comming at this from a more typical Unix/Linux admin perspective. root accounts are usually avaible for admin tasks, but access to the root accounts is usually (hopefully) strictly limited. We are going to allow root access on your Mac just long enough to bootstrap an unprivileged Ansible account.

Don't run the following command with sudo. It will validate that you are authorized to do this.

Enable the root account at the command line:
`dsenableroot`

First you type in your user account password. Then you'll set a root uesr password.

The process will look like this:
```
sparky@lab83 ~ % dsenableroot
username = sparky
user password:
root password:
verify root password:

dsenableroot:: ***Successfully enabled root user.
```


#### Allow root login in the sshd configuration
We're going to edit the system sshd configuration file. Make a backup copy. If you mess up the configuration then the ssh daemon will not start.

I use nano as my preferred text editor. Use whatever you like as long as it's a plain text editor.

Allow root login with only a password like this:
`sudo nano -w /etc/ssh/sshd_config`


I search for the #PermitRootLogin line in the file, duplicate it, and change the new line to `PermitRootLogin yes`. This is a snippet of what mine looks like. As I mentioned before, this is not a great security practice, but this is just until we can deploy an Ansible user account. As a general practice, if we allow remote connecctions to the root account at all they should be key based only, i.e., prohibit-password. The best practice is to disble remote root connections completely.
```
#PermitRootLogin prohibit-password
PermitRootLogin yes              
```

One nice thing about Mac is that the ssh daemon will detect the change in our configuration and implement it without restarting anything.

At this point you should be able to connect to your managed system via ssh. You may log out of your desktop.


#### Install system wide Python
(We'll show two methods to do this task. One is done on each OS Desktop. The other is done from an ssh session command line as root.)
We need to install Python now. We have a lot of choices of how we approach this. Our main issue is that Apple doesn't have a native way to intall it. It's not included  with Xcode command line tools. That means we'll have to us a non-native method.

For system wide use we really only have two options. One is Homebrew but with tweaks to allow system wide use of the installed packages. Homebrew is great but it requires we install the Xcode command line developer tools and then we need to tweak it's configuration to allow all system users access to Python. I'm not an expert in this setup at all. I also don't want all the bloat.

I'm opting for a direct download from [python.org](https://www.python.org/downloads/macos/) and downloading a Mac installer. We also need to download an appropriate version of Python, which will probably not be the latest version.

Here are some of the considerations in choosing the correct version:
- it should be capable of running on all MacOS versions you support
- it must be capable of operating with all versions of Ansible you expect to use

should = a good ideal
must = mandatory

I prefer to find a common version that will run on all my supported MacOS versions. As of this writing Apple still support MacOS 14 and up. It is possible to install different versions of Python on your different MacOS nodes, but I like the consistency of using the same version everywhere. I'm not talking about developers though. They'll probably override all of this with pyenv or some other virtual environment anyway.

We **must** choose a version of Python that will work will all the versions of Ansible you may use. I have some obsolete systems that I have to manage. As a consequence I need to use Ansible 2.12. Python 3.10 is the highest version of Python that was officially supported. The most recent version of Ansible (2.21 as of this writing) is fully supported on Python versions 3.9 through 3.14. Givin those contstraints, I am choosing Python 3.10. How did I find this information? Here: [Python Ansible Support Matrix](https://docs.ansible.com/projects/ansible/latest/reference_appendices/release_and_maintenance.html#ansible-core-support-matrix)

I'm going with Python 3.10.11. There are newer 3.10.x releases, but they don't have native installers for Mac. I don't want to build a newer version, so this is what I'm using.

##### OS Desktop method
Download the installer then from your Downloads directory run the package. (It's not a dmg.) Agree to all the license acknowledgements. Then you'll be prompted for your password. Again, you must be a local system administrator. That should do it. I tested this on MacOS verions 14 through 26.

Do a quick test. Connect via ssh as root then type `python3 --version`.
Your result should look like this:  
```
lab83:~ root# python3 --version
Python 3.10.11
```

##### Alternate command line method
If you open a shell to many Mac hosts and want to send the same command to all of them in ssh sessions, then this is the way to do it. Simple. Fast. Scriptable.

Note: This could work with Ansible raw commands too. If you write a script just be sure to use syntax compatible with *sh* or invoke a better shell. root accounts on Mac do not use bash, zsh, or any other more advanced shell by default.

Open an ssh session to the host Mac as root. (You may use any user with sudo priveleges. Be sure to prepend `sudo`.)

Get the installer:  
`curl -O https://www.python.org/ftp/python/3.10.11/python-3.10.11-macos11.pkg`

Run the installer, installing to the default file system location (base path /):  
`installer -pkg python-3.10.11-macos11.pkg -target /`

Do the test:  
`python3 --version`

You can delete the installer package now if you like.

#### Time to test
The way I verify that everything is working the way I expect is to do an Ansible ping. This is what it looks like:
```
(ans2.12) kbrown@MacBookPro maclab % ansible -o --user root --ask-pass --module-name ansible.builtin.ping lab83
SSH password:
lab83 | SUCCESS => {"ansible_facts": {"discovered_interpreter_python": "/usr/bin/python3"},"changed": false,"ping": "pong","warnings": ["Platform darwin on host lab83 is using the discovered Python interpreter at /usr/bin/python3, but future installation of another Python interpreter could change the meaning of that path. See https://docs.ansible.com/ansible-core/2.12/reference_appendices/interpreter_discovery.html for more information."]}
```

In that scenario I used Ansible 2.12 running on Python 3.10.21 against a MacOS 26 system running Python 3.10.11. It's a very old Ansible working with a very new MacOS running an older version of Python. You can mix and match however you need it.

I hope this was helpful.

## Did you make it to the end? Here's a bonus. Find my **generic-MacOS-python_install_raw.yml** playbook https://github.com/SparkyzCodez/Sparkyz-Notez . It will install Python3 to all your Macs at the same time. It does not need Python because it uses the ansible.builtin.raw module. It's well documented and should answer all your questions.

Cheers,
Sparky