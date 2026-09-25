Ansible with old OSes via pyenv and python virtual environments.md


## Prerequisites
Before we go any further let's get a few prerequisites out of the way.

### Fully Configured Ansible Environment
First, you must be completely setup with pyenv, have an appropriate version of Python installed and active, and an appropriate version of Ansible installed to your virtual environment. Does this sound familiar to you? If not then please review my setup document on my Git Sparkyz Notez site.  
[Install Specific Versions of Ansible With Virtual Environments - HowTo.md](https://github.com/SparkyzCodez/Sparkyz-Notez/blob/main/Install%20Specific%20Versions%20of%20Ansible%20With%20Virtual%20Environments%20-%20HowTo.md)

You're not using a root account on your control node are you? You know better. You should be using a typical, limited permissions account on your control node for all Ansible operations. I'll share instructions on deploying a low permissions account to each of your controlled nodes. With Ansible we have the ability to elevate permissions with sudo. You shold only elevate permissions (aka *become*) on a task by task basis. Very few playbooks need root permissions for all their tasks.

### Create Disabled ansible.cfg File
**My current setup has a number of different combinations of Python and Ansible. Yours may too. If you have more than one virutal environment then this next step is important.**

Switch into your **lowest** Ansible version environment. Were going to create a default configuration file and it's really important we use the lowest common denominater version because old versions of Ansible will not understand new additoins and newer formatting items.

My lowest common denominator setup is:
- Python 3.10.21
- Ansible core 2.12.10 (community version 5.10)

Both of those are antiques but I have a few sad, obsolete Red Hat 6 (RH6) servers that must be managed. This is the setup that still works for that RH6 environent and any other systems that simply cannot install Python3 without extraordinary steps.

Ansible uses a fairly complex precedence heirarchy to find its configuration file. As of this writing these are the precedence rules. [ref to Ansible docs](https://docs.ansible.com/projects/ansible/latest/reference_appendices/general_precedence.html#configuration-settings)

- ANSIBLE_CONFIG (environment variable if set)
- ansible.cfg (in the current directory)
- ~/.ansible.cfg (in the home directory)
- /etc/ansible/ansible.cfg (This one will absolutely NOT work if you're using Python virtual environments.)

I don't use the environment variable version simply out of personal choice because I switch between shells. I just don't want to manage environment variables across sh, bash, and zsh.

I use git version control on all my Ansible playbooks, configs, roles, etc. I choose to create my config file inside that repo then use a symlink to my control node home directory. My repo local directory is `~/maclab`. (I am using a Mac as my control node to develop these instructions.)

Here's how to create an Ansible config file with all the options included but disabled. We'll enable those that we need later.  
`ansible-config init --disabled > maclab/Ansible/ansible.cfg`

Then I create a `.ansible.cfg` symlink in my home directory. Note the leading dot in the file name.  
`ln -s ~/maclab/Ansible/ansible.cfg ~/.ansible.cfg`

### You Will Need sshpass
If you already have key based authentication to all your root accounts on all your hosts then you may not need sshpass. You can skip the `--ask-pass` parameter in the ad hoc Ansible command. I'm coming at this process as if all the servers are unconfigured with the exception of a common root password and python installed.

From a security perspective, the only significant risk of sshpass is leaving passwords in your shell history. Beause we are telling Ansible to prompt us we do not have that risk.

## First Tests And Some Fixups
At this point you will need your inventory file and for the inventory location to be specified in your ansible.cfg file. I use INI format and I name my file *inventory.ini* because I feel like there are already too many Linux/Unix files named *hosts*.

You may also use an environment variable instead of hard coding it in your config file. It's a completely valid choice that I just don't happen to use.

And one last thing I ran into. I reused a bunch of IP addresses and had old entries in my known_hosts file. Now is good time to clean it up before we start testing.

If you have host key checking enabled (you probably do) then you may choose to add the host keys from each node manually. That is tedious. When I bulk add hosts with Ansible I disable host key checking so the keys are added to my *known_hosts* file automatically. Manually set this environment variable to accomplish that.  
`export ANSIBLE_HOST_KEY_CHECKING=False`

It would be a poor security practice to make this permanent. Just set it, add your hosts, then restart your shell to clear it.

**I am using Ansible 2.12 for the initial tests because I have obsolete systems. By default Red Hat 6 and 7 use Python 2.6 and 2.7 respectively. Red Hat 7 can install an old version of Python 3 from repositories. Not so for Red Hat 6. I'll switch to Ansible 2.21 later on in this process so we can enjoy a new set of errors to work through.**

Now let's do our first connection to all our hosts.  
`ansible --user root --ask-pass --module-name ansible.builtin.ping all`

A quick walk through of our options and why I chose them:
- `--user root` I don't have any accounts on my guest hosts yet. I'm connecting as root
- `--ask-pass` I don't have key based login to my guests from my control node account. I need the root password here, but not for long.
- `--module-name ansible.builtin.ping` We're pinging. I'm being verbose just for clarity. Feel free to `-m ping`.

This process assumes the systems are not configured for use with Ansible beyond knowing a root account login and password, and having Python installed. The RH6 systems have Python 2.7. Everything else has Python3. But we still have some fix-up items to get this to show all green pings.

### Allow Obsolete SSH Cipher and Key Exchange Algorithms
The first problem I had was obsolete ciphers on RH6 for SSH connections. Here's the error:  
`no matching host key type found. Their offer: ssh-rsa,ssh-dss"`

The word **offer** tells you that this is a cipher handshake mismatch. My control node does not currently allow the older cipher but RH6 is unable to offer anything newer. I am choosing to allow my control node to allow connections to RSA cipher SSH systems. While I could manually build and install a newer version of SSH on the RH6 systems, I am choosing to just use my fully patched RH6 systems as they are.

In a produciton environment be sure to check with your cyber-sec team and your network policies. Weakening ciphers in network traffic needs to be a deliberate and documented step.

I allowed RSA only in my ansible control node's user account. Add the following to your `~/.ssh/config` file. (Be sure your file permissions are 600, read-write only for the user.) Also, add this to the bottom of your config file so that any global settings aren't overriden. Limit the scope as much as you can. Here's my fix in my file for my lab 172.17.1.0/24 network.  
```
Host 172.17.1.*
    PubkeyAcceptedKeyTypes +ssh-rsa
    HostKeyAlgorithms +ssh-rsa
```

### Can't Find Python On Guest
Some systems may throw an error about not being able to find Python. In my experience this is rare on Linux systems. Not so for FreeBSD Unix. It happens on versions 14 and 15, and probably older versions as well. FreeBSD doesn't install Python by default. Even when you do install Python, it doesn't install to the typical location that Linux would use. Instead it goes to a more Unix typical location of `/usr/local/bin/python3`.

Here's a snippet of a typical error indicating this problem:  
```
"module_stdout": "/bin/sh: /usr/bin/python: not found\r\n",
"msg": "The module failed to execute correctly, you probably need to set the interpreter.\nSee stdout/stderr for the exact error"
```

We have a few aproaches to take here. First, it is possible to specify the Python interpretter location in your inventory for the affected hosts. It's not a bad aproach if you want to keep all special case tweaks on your control node and leave your guest system unchanged. You could do something like this in your inventory:  
`lab22 ansible_python_interpreter=/usr/local/bin/python3`

I use a different fix. I simply add a symlink to the guests that are affected. I don't like custom tweaks on guests in general, but this one is really simple and won't negatively affect anything else on the guest. This is all it takes on my systems:  
`ln -s /usr/local/bin/python3 /usr/bin/python`

**At this point I switched to Ansible 2.21 running with 3.14.7. These are the latest non-development versions as of this writing.  
Special note is that this version of Ansible requires a minimum of Python 3.9 on each managed host.**

### Warnings Galore!
Newer versions of Ansible throw tons of warnings. Some warnings are just annoyances. In my opinion, if a warning is thrown on every managed host, that warning is not very useful.

This one is particularly annoying to me:  
`...future installation of another Python interpreter could cause a different interpreter to be discovered`

This happens on nodes that otherwise shown SUCCESS. It's basically saying that if you install another Python it might be used instead. Uh, OK. This is one to suppress as you move into production.

Warnings that are accompanied by FAILED! are useful.

Here's a warning you're likely to see:  
`No python interpreters found for host 'xxx' (tried ['python3.14', 'python3.13', 'python3.12', 'python3.11', 'python3.10', 'python3.9', '/usr/bin/python3', 'python3'])`

This warning will be accompanied by all kinds of failure information. It may mention LOCALE problems, or from future import problems, or deserialization problems. The actual root problem is that it can't find a supported version of Python 3 on the managed host. When you see any errors like these you should first check that the system has a supported verion of Python3 installed. In some cases just updating your managed node will clear the problem. In other cases you may need to pull in additional repositories with newer/bleeding edge packages. Or you might consider using an slightly older Ansible/Python combination on your control node. (Ansible 2.16 with Python 3.12 is very flexible and works with nearly anything except managed hosts with Python 2.x.)

As I mentioned previously, Red Hat 6 has no real path to Python 3 from the usual (and now frozen) repositories. Rather than trying to fix the RH6 system I just use an older version of Ansible.

Red Hat 7 and 8, SUSE 15, and other older distributions, even when fully patched from basic repos, do not have a new enough Python. Rather than go to extraordinary means, I just use a slightly old version of Ansible. This is also why I'm a proponent of using Python venv (virtual environments) to switch control node Ansible/Python combinations.

### Other Errors?
The preceding problems are by no means the only ones you could encounter. In my experience they are the most common, but there are a lot of different system combinations in the wild.

Be methodical. First determine if the problem is on your control node or on the guest node. Use the verbose flag on your command to dig into the errors. Setup a baseline test node and get it working. Then tackle the more complex production systems once you know your control node is functional.


### All Errors Are Cleared - Green OK Pings From All Nodes
Now it's time to restart your shell to clear any environment variable tweaks you enabled. Ping everything one more time. Are you still all Green and OK? Good. You're now ready to deploy a limited permissions Ansible access user to all your nodes and add them to your sudo files. We never want to user root accounts again, but that's for a different set of instructions. I'll share my basic ansible user deployment playbook and approach on my Git Sparky'z Notez site.
