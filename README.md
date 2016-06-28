# !Under Construction!

## Hadoop 2.7.2 Tutorial

Being interested in distributed programming and Hadoop, I decided to setup my own VM cluster using the latest version (2.7.2) on my Windows desktop.  It's actually not so difficult so I will outline all the tasks here.  Even though the desktop is Windows 10, the VMs run (Arch) Linux so it helps if you are/have been a Linux user.  The actual Linux distribution you use is not that important, but I chose Arch since it is a *small* install and that makes for small VM size.

At any point in this guide you can attempt to blindly follow the steps, but it's best if you read relevant material along the way.  It may take you a bit longer to get setup, but you will be more confident if you have some understanding of the intermediate steps.

## Single Machine, Hadoop 2.7.2 VM Cluster Installation Tutorial

### Install Hyper-V

```
Control Panel > Programs > Programs and Features > Turn Windows features on or off > Hyper-V
```

If for some reason you are unable to check the Hyper-V box, [you may need to enable virtualization from the BIOS](https://blogs.technet.microsoft.com/canitpro/2014/03/10/step-by-step-enabling-hyper-v-for-use-on-windows-8-1/) or do some other trouble shooting.

### Create a Linux+Hadoop Template VM

The majority of Linux and Hadoop installations can be done once and cloned.  This is nice for later when you want to show how easy it is to add new DataNodes to your VM cluster.

##### Choose and download a Linux Distro ISO

1. Start downloading a Linux ISO.  Again, the specific distro doesn't matter, you simply want to be able to install
  1. Git
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

1. Startup and login to all three VMs.
2. In each VM, write their hostname into `/etc/hostname`


Each VM will need it's own IP address which can generally be done very easily using DHCP reserved IPs.  This is generally easy to do using your router's configuration.  You likely need a wired connect to reach the router's configuration UI.


2. Use a web browser to navigate to your router's configuration UI.  If you don't know the IP:
  1. Open a command prompt (Windows Key + type "cmd" + enter)
  2. `ipconfig`
  3. It's the IP associated to "Default Gateway".
3. In the configuration there is generall a "Gateway" tab or "View Connections" that show the list of machines currently connected to your router.  Wherever this exists, you should see connections for your three VMs (possibly called "unknown" or something similar).  
