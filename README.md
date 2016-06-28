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
The configuration files that reside on each VM are identical, we only need to install Hadoop and the configurations on the template VM, and then make sure to setup the IPs/hostnames so that VMs can communicate with one another.

##### Get Hadoop

1. In your home folder `wget http://www.apache.org/dyn/closer.cgi/hadoop/common/hadoop-2.7.2/hadoop-2.7.2.tar.gz`
2. `tar -xf hadoop-2.7.2.tar.gz`
3. `mv hadoop-2.7.2 haddop`
4. `
4. `cd hadoop`
5. `mkdir 

##### Configure Hadoop

This repo comprises the configuration files for the cluster I set up, which for you is `~/hadoop/etc/hadoop`.  To keep things simple, I am going to assume that you are going to setup the same three VM cluster having the same hostnames.  Feel free to poke around the configs and change names if you like, but if you are following this guide, odds are you are also new to Hadoop and it may benefit you to have the VMs suggestively named.

*Important Files in `etc/hadoop`*:

These are the suggestively named, core configuration files.  I currently don't understand everything that can be done through them, but what you see in this repo are the minimal configurations to get a cluster up and running.

1. `core-site-xml`: (important) sets location of HDFS filesystem.  Note that this value `hdfs://namenode:9000` targets our `namenode` VM as the NameNode.
2. `hdfs-site.xml`: (
3. `mapred-site.xml`
4. `yarn-site.xml`
