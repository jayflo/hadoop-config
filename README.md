## Hadoop 2.7.2 Single Host, multiple VM Cluster Setup

Being interested in distributed programming and Hadoop, I decided to setup my own VM cluster using the latest version (2.7.2) on my Windows desktop.  It's actually not so difficult so I will outline all the tasks here.  Even though the desktop is Windows 10, the VMs run (Arch) Linux so it helps if you are/have been a Linux user.  The actual Linux distribution you use is not that important, but I chose Arch since it is a *small* install and that makes for small VM size.

At any point in this guide you can attempt to blindly follow the steps, but it's best if you read relevant material along the way.  It may take you a bit longer to get setup, but you will be more confident if you have some understanding of the intermediate steps.

### Install Hyper-V

```
Control Panel > Programs > Programs and Features > Turn Windows features on or off > Hyper-V
```

If for some reason you are unable to check the Hyper-V box, [you may need to enable virtualization from the BIOS](https://blogs.technet.microsoft.com/canitpro/2014/03/10/step-by-step-enabling-hyper-v-for-use-on-windows-8-1/) or do some other trouble shooting.

### Create a Linux+Hadoop Template VM

The majority of Linux and Hadoop installations can be done once and cloned.  This is nice for later when you want to show how easy it is to add new DataNodes to your VM cluster.

##### Choose and download a Linux Distro ISO

1. Start downloading a Linux ISO.  Again, the specific distro doesn't matter, you simply want to be able to install
  1. wget
  2. Java
  3. SSH
  I prefer Arch Linux simple because the base install is relatively small (approx 4Gb per VM).  However, Arch does not have a graphical installer and if you are not used to command line installations it may seem daunting.  If you've never done it before, you should consider it a rite of passage and give it a shot.

##### Create an External Switch in Hyper-V

1. Open Hyper-V
2. In the menu on the right hand side:
  1. Virtual Switch Manager
  2. Selected "External"
  3. Create Virtual Switch
  4. Pick a name
  5. External Network > Select your ethernet card
  6. Ok/Apply

##### Create Template VM

1. Open Hyper-V
2. In the menu on the right hand side
  1. New > Virtual Machine
  2. Choose a name, e.g. `ArchHadoop-Template`
  3. Choose a different storage location if desired
  4. Generation 1
  5. Memory ... something small
  6. Connection ... the virtual switch created above
  7. Create a virtual hard disk, change size if desired
  8. Install an operating system from a bootable CD/DVD-ROM > Image File > the ISO downloaded above
  9. Finish

##### Install Linux

1. Open Hyper-V
2. Select your VM from the list
3. Start/Connect ... you should boot into the installer
4. Do the installation as normal
5. Create a user that has sudo rights.

*Arch Linux*

1. See the [Beginner's Guide](https://wiki.archlinux.org/index.php/beginners'_guide)
2. See the [Hyper-V Guide](https://wiki.archlinux.org/index.php/Hyper-V) (this also contains steps for creating a Hyper-V VM)
3. **Special Notes**
  1.  The Hyper-V Guide tells you how you need to format your VM's hard drive.  The `cgdisk` tool is easiest to use.  This is needed near the beginning of the installation.
  2.  Near the middle/end you will install a bootloader.  The Hyper-V Guide also explains how you should do this, e.g. use `grub`.
  3.  You do not need to set a `hostname` in `/etc/hostname`

##### Install Hadoop Dependencies

1. Install Java (Arch: `sudo pacman -S jdk8-openjdk`)
2. Install `wget` (Arch: `sudo pacman -S wget`)
3. Install ssh (Arch: `sudo pacman -S openssh`, `sudo systemctl enable sshd.socket`)
4. [Setup passwordless login for ssh](http://stackoverflow.com/a/16651742/2705308)

### Install Hadoop

Now we are getting to the good stuff.  If you made it to this point but you have no idea what Hadoop is, [you should probably read some of the documentation](http://hadoop.apache.org/docs/stable/index.html).  Some good pages to read are:

1. [HDFS Architecture](http://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HdfsDesign.html)
2. [HDFS User Guide](http://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HdfsUserGuide.html)
3. [Yarn Architecture](http://hadoop.apache.org/docs/stable/hadoop-yarn/hadoop-yarn-site/YARN.html)
4. [MapReduce Tutorial](http://hadoop.apache.org/docs/stable/hadoop-mapreduce-client/hadoop-mapreduce-client-core/MapReduceTutorial.html) (this one can wait a little bit)

[Here is the general guide for Hadoop's cluster setup](http://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/ClusterSetup.html#Hadoop_Cluster_Setup) that we will apply to our VMs.

##### The General Idea

That major players in a Hadoop cluster are
  1. NameNode: the HDFS master node that manages all HDFS metadata
  2. DataNode(s): HDFS slaves that store and serve up (chunks of) files
  3. ResourceManager: the Yarn master node responsible for arbitrating cluster resources
  4. NodeManager: the Yarn slave monitiring a node's resources and reporting to the ResourceManager
I'm not sure if you can have all these different processes running on a one or two nodes (VMs), but in this tutorial we are going to create three:
  1. `namenode`: will be the NameNode and a DataNode.
  2. `resourcemanager`: will be the ResourceManager and a DataNode
  3. `datanode1`: will be a dedicated DataNode.
The configuration files that reside on each VM are identical, we only need to install Hadoop and the configurations on the template VM, and then make sure to setup the IPs/hostnames so that VMs can communicate with one another.  Setting up VM hostnames and IPs will be done after the Hadoop installation.

##### Get Hadoop

1. In your home folder `wget http://www.apache.org/dyn/closer.cgi/hadoop/common/hadoop-2.7.2/hadoop-2.7.2.tar.gz`
2. `tar -xf hadoop-2.7.2.tar.gz`
3. `mv hadoop-2.7.2 haddop`
4. `mkdir -p hadoop-hdfs/{name,data}`
5. `cd hadoop`

##### Configure Hadoop

This repo comprises the configuration files for the cluster I set up, which for you is `~/hadoop/etc/hadoop`.  To keep things simple, I am going to assume that you are going to setup the same three VM cluster having the same hostnames.  Feel free to poke around the configs and change names if you like, but if you are following this guide, odds are you are also new to Hadoop and it may benefit you to have the VMs suggestively named.

*Important Files in `etc/hadoop`*:

These are the suggestively named, core configuration files.  I currently don't understand everything that can be done through them, but what you see in this repo are the minimal configurations to get a cluster up and running.

If you want to start with this repo rather than copy/paste, do:

1. `cd etc`
2. `rm -rf hadoop`
3. `git clone https://github.com/jayflo/hadoop-config.git hadoop`

Now...

1. `core-site-xml`: (no changes required) sets location of HDFS filesystem.  Note that this value `hdfs://namenode:9000` targets our `namenode` VM as the NameNode.
2. `hdfs-site.xml`: change the file paths to point to your home directory.  These file paths tell the NameNode and DataNodes where to store data.
3. `mapred-site.xml`: (no changes required) tell Yarn to use mapreduce
4. `yarn-site.xml`: (no changes required) specifies the hostname of the ResourceManager.
5. `hadoop-env.sh, mapred-env.sh, yarn-env.sh`: change the `JAVA_HOME` file path to point to your installation of Java.  These files setup the environment for Hadoop tasks.
6. `slaves`: (no changes required) a list of hostnames to use as DataNodes.  This is not used by the Java code, but utilized by helper scripts to spin up processes throughout the cluster.

That's it!  Our Linux+Hadoop template is complete. Logout and shutdown your VM.

### `/etc/hosts`

After the VMs are cloned, each one will be given a hostname (`namenode`, `resourcemanager` and `datanode1`) and a reserved IP address on your local network.  The association `ip address --> hostname` for machines on your network is stored in the OSs `/etc/hosts` file (on Windows `C:\Windows\System32\drivers\etc\hosts`) and will be the same for each VM in the cluster.  If you already know what three addresses you will assign these VMs, you can edit the `/etc/hosts` file as follows:

```
::1           localhost
127.0.0.1     localhost
<ip_1>        namenode
<ip_2>        resourcemanager
<ip_3>        datanode1
```

However, if you are not yet sure what the IP addresses should be, you can wait to edit this file till later.  You will simply need to `ssh` to each machine to change it.

### Clone the VM

*Export*

1. Open Hyper-V
2. Right Click > Export
3. Pick a location

*Import x 3*

1. Open Hyper-V
2. In the right hand menu > Import Virtual Machine
3. Browse to the folder chosen in Export above.
4. Next, Next
5. Copy the virtual machine (create a new unique ID)
6. Store the machine in a different location: enter the same path for all three text boxes
7. When choosing where to store the virtual hard drive, enter the path from the previous step + `\Virtual Hard Disks\`.
8. Finish
9. When it's complete, rename the new VM to `namenode`.

Repeate 1-9 two more times but called the next VMs `resourcemanager` and `datanode1`.

### Setup Hostnames and IPs

*Hostnames*

1. Startup and login to all three VMs `namenode`, `resourcemanager` and `datanode1`.
2. In each VM, write their respective name (`namenode`, `resourcemanager` or `datanode1`) hostname into `/etc/hostname`.  The file should contain only this name.
3. Restart the VM.

Each VM will need it's own IP address which can generally be done very easily using DHCP reserved IPs.  In general this is easy to do using your router's configuration UI.  You likely need a wired connect to reach the router's IP.

1. Use a web browser to navigate to your router's configuration UI.  If you don't know the IP:
  1. Open a command prompt (Windows Key + "cmd" + enter)
  2. `ipconfig`
  3. It's the IP associated to "Default Gateway".
2. Usually there are two ways IPs are reserved:
  1. There is a menu where you can specify a hostname/mac-address and the IP you want to reserve for that hostname/mac-address.
  2. There is a menu listing currently connected devices by hostname/mac-address and you can "edit" the connection providing an IP.
  The IP address you choose doesn't really matter as long as it's valid.  If you like to keep things "in order," your cluster IPs should start after all other IPs.  This way, if you want to add more machines in the future, the next spot won't be taken.  Otherwise, you can simply start right after the router's IP.
3. If you didn't already edit the `/etc/hosts` file, now you must.  You can visit each VM or `ssh` freely between them to make the edits.  The form of the `/etc/hosts` is shown above.
4. Shut down all the VMs.
5. Edit the `C:\Windows\System32\drivers\etc\hosts` file of the host (your Windows desktop) to include the IP --> VM hostname mappings.  You don't need to include the `localhost` lines, but it wouldn't hurt anything if you did.
5. Open another cmd prompt on the host (your Windows desktop) and do:

  ```
  ipconfig /release
  ipconfig /all
  ipconfig /flushdns
  ipconfig /renew
  ```
6.  Restart your computer.

### Spin up Hadoop

Time to reap the benefits.  Everything should now be in place, we simply need to spin up the cluster and start all the HDFS/Yarn processes.

1. Open Hyper-V
2. Connect/Start cluster VMs `namenode`, `resourcemanager` and `datanode1`
3. On `namenode` run:
  1. `cd ~/hadoop`
  2. `bin/hdfs namenode -format`
  3. `sbin/start-dfs.sh` (it should start 5 things)
4. On `resourcemanager` run:
  1. `sbin/start-yarn.sh`

If everything is working correctly, all of the following should be true:

1. The `jps` (`man jps` for more info) command should have the following output:
  1. `namenode` --> `NodeManager`, `NameNode`, `SecondaryNameNode`
  2. `resourcemanager` --> `ResourceManager`, `NodeManager`
  3. `datanode1` --> `NodeManager`
2. You should be able to visit the NameNode Web UI from your host's browser at URL `namenode:50070`
3. You should be able to visit the ResourceManager Web UI from your host's browser at URL `resourcemanager:8088`
4. `cd ~/hadoop && grep -iR error logs` should return nothing.

If everything isn't working correctly then you are in for some more work.  Be certain to look at the logs `~/hadoop/logs` to see if you can find specific errors which you can Google about.  From my own searching on the web, more often than not the problem will be a network connectivity issue requiring firewall modifications (i.e. the nodes are unable to connect to one another).  For these you are on your own...feel free to open up and issue though I promise nothing!

### Contributing/Changes

I made this a repo so that anyone can help make changes and keep it up to date/provide more explanation where needed.  If you'd like to add some info, submit a PR and we can include it.  Aside from adding more detail to this tutorial, it would be nice to populate the "core" xml configuration files with more options+descriptions (commented out so they are not "on" by default).  This will provide a quick way for other Hadoop hopefuls to learn about common cluster settings.  See these pages for comprehensive lists of configurations and their default values:

1. [core-site.xml](http://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/core-default.xml)
2. [hdfs-site.xml](http://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/hdfs-default.xml)
3. [mapred-site.xml](http://hadoop.apache.org/docs/stable/hadoop-mapreduce-client/hadoop-mapreduce-client-core/mapred-default.xml)
4. [yarn-site.xml](http://hadoop.apache.org/docs/stable/hadoop-yarn/hadoop-yarn-common/yarn-default.xml)

And of course, read as much as you can about Hadoop from the general/API documentation.

### The future

Next I will work on setting up IntelliJ for Hadoop development so that one can easily test their mapreduce jobs on the cluster created in this tutorial.  I will create another repo containing the project code as well as a comprehensive README for setting up the project and executing jobs.  When it is finished, I'll link it here.

Good luck!
