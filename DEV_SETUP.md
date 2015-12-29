# Quick developer's guide for working with GPDB

### Credits
This guide was developed in collaboration with Navneet Potti (@navsan) and
Nabarun Nag (@nabarunnag).

## Who should read this document?
Any new developer who wants to develop code for GPDB. This guide targets
the free-lance developer who typically has a laptop and wants to develop
GPDB code on it. In other words, such a typical developer does not necessarily
have 24x7 access to a cluster, and need a miminal stand-alone development
enviroment.

The instructions here were first written and verified on OSX El Capitan,
v.10.11.2. In the future, we hope to add to the list below.

| **OS**        | **Date Tested**    | **Comments**                           |
| :------------ |:-------------------| --------------------------------------:|
| OSX v.10.11.2 | 2015-12-29         | Vagrant v. 1.8.1; VirtualBox v. 5.0.12 |

## 1: Setup VirtualBox and Vagrant
You need to have both VirtualBox and Vagrant setup. If you don't have these
setup already, then head over to https://www.virtualbox.org/wiki/Downloads
and http://www.vagrantup.com/downloads to download and then install these
packages.


##2: Clone GPDB code from github

Go to the directory in your machine where you want to check out the GPDB code
and clone the GPDB code, by typing the following into a terminal window.

```shell
git clone git clone https://github.com/greenplum-db/gpdb.git
```

##3: Setup and start the virtual machine
Next go to the gpdb/vagrant directory and start up a virtual machine using the
Vagrant file there; i.e. from the terminal window, issue the following command:

```shell
cd gpdb/build
vagrant up
```

The last command will take a while as Vagrant works with VirtualBox to fetch
a box image for CentOS. The box image is many hundred MiBs is size, so it takes
a while. This image is fetched only once and will be stored by vagrant in a
directory (likely `~/.vagrant.d/boxes/`), so you won't incur this network IO
if you repeat the steps above. A side-effect is that vagrant has now used a
bunch of space on your machine. You can see the list of boxes that you
vagrant has downloaded using ``vagrant box list``. If you need to drop some
box images, follow the instructions posted [here](https://docs.vagrantup.com/v2/cli/box.html "vagrant manage boxes").

If you are curious about what vagrant is doing, then open the file 
vagrant/Vagrantfile. The `config.vm.box` parameter there specifies the vagrant
box image that is being feteched. Essentially you are creating an image of
CentOS on your machine that we will use below to setup and run GPDB. 

While you are viewing the Vagrantfile, a few more things to notice here are:
* The parameter `vb.memory` sets the memory to 8GB for the virtual machine.
  You could dial that number up or down depending on the memory in your machine.
  Going below 2GB is not recommended. 
* The parameter `vb.cpu` sets the number of cores that the virtual machine will
  used to 2. Again, feel free to change this number based on the machine that
  you have. 
* Notice the parameter `config.vm.synced_folder`. This requests that the code
  you checked out into the directory `gpdb` to be mounted as `/gpdb` in the
  virtual machine. More on this later below. 


Once the command above (`vagrant up`) returns, we are ready to start the
virtual machine. Type in the following into the terminal window (make sure
that you are in the directory `gpdb/vagrant`):

```shell
vagrant ssh
```

Now you are in the virtual machine shell in a guest OS that is running in
your actual machine (the host). Everything that you do in the guest machine
is going to be isolated from the host, except for any changes that you make to 
``/gpdb`` -- recall that we requested that the code we checked out show up at
this mount point in the virtual machine. From the virtual machine shell if you
type ``ls /gpdb``, you should see the gpdb code.

##4: Setup and compile GPDB

Next, issue the following commands from the guest shell. 

```shell
mkdir -p /gpdb/vagrant/install 
mkdir -p ~/gpdb_data
```

The above commands create an install directory and a data directory. The install
directory is created in the shared folder (so changes here will be visible 
from the host). The data directory is created in the local disk in the guest OS.
Changing the data directory to the shared folder (i.e. under `/gpdb`) will run
into trouble with file permissions when creating data under GPDB. So, make sure
you keep the data directory pointed to disk space that is only visible in
the guest OS.

Now, in the guest shell go to the directory that contains the GPDB code and
configure it to build, as follows:
```shell
cd /gpdb
./configure --prefix=/gpdb/vagrant/install --enable-depend --enable-debug
```

Now, let's build GPDB. In the guest shell, type in:
```shell
make
```
Building the binaries will take a while. If you are adventurous you can type in
``make -j 2`` to build in parallel using two threads (note your VM is setup
to run on two cores of the host machine). Sometimes, this parallel make
produces errors. If that happens, just re-issue the ``make`` command again,
without the ``-j`` flag.

Next, we are ready to install GPDB in the directory ``/gpdb/vagrant/install``.
To do this, in the guest shell type in:

```shell
make install
```

##5: Setup and compile GPDB
We are nearly ready to run GPDB. But before we do that we have to setup a few
environment variables, and make changes to the guest OS. Here are the steps. In
the guest shell, type in the following to create the required environment
variables: 

```shell
printf '\n# GPDB environment variables\nexport GPDB=/gpdb\n' >> ~/.bashrc
printf 'export GPHOME=/gpdb/vagrant/install\n' >> ~/.bashrc
printf 'export GPDATA=~/gpdb_data\n' >> ~/.bashrc
printf 'if [ -e $GPHOME/greenplum_path.sh ]; then\n\t' >> ~/.bashrc
printf 'source $GPHOME/greenplum_path.sh\nfi\n' >> ~/.bashrc
source ~/.bashrc
```

Then, type in the following to create the required data directories: 
```shell
mkdir -p $GPDATA/master
mkdir -p $GPDATA/segments
```

We also need to setup up ssh key, by typing (in the guest shell) the following
commands"
```shell
gpssh-exkeys -h `hostname`
hostname >> $GPDATA/hosts
```

We need to create a configuration file for GPDB. Let's setup a cluster with
just one master and two segments. Type in the following commands into the guest
shell to create a configuration file called ``gpinitsystem_config``.

```shell
GPCFG=$GPDATA/gpinitsystem_config
rm -f $GPCFG
printf "declare -a DATA_DIRECTORY=($GPDATA/segments $GPDATA/segments)" >> $GPCFG
printf "\nMASTER_HOSTNAME=`hostname`\n"                                >> $GPCFG
printf "MACHINE_LIST_FILE=$GPDATA/hosts\n"                             >> $GPCFG
printf "MASTER_DIRECTORY=$GPDATA/master\n"                             >> $GPCFG
printf "ARRAY_NAME=\"GPDB\" \n"                                        >> $GPCFG
printf "SEG_PREFIX=gpseg\n"                                            >> $GPCFG
printf "PORT_BASE=40000\n"                                             >> $GPCFG
printf "MASTER_PORT=5432\n"                                            >> $GPCFG
printf "TRUSTED_SHELL=ssh\n"                                           >> $GPCFG
printf "CHECK_POINT_SEGMENTS=8\n"                                      >> $GPCFG
printf "ENCODING=UNICODE\n"                                            >> $GPCFG
```

We also need to change the guest OS setting as per the instructions
[here](http://gpdb.docs.pivotal.io/4360/prep_os-system-params.html#topic3). 
The security limits and other settings specified there are not necessary for us,
so we can skip that. To change the guest OS setting issue the following command
in the guest shell (you could use any editor that you like, but will need 
to use sudo mode to save the changes of the file):

```shell
sudo vi /etc/sysctl.conf 
```

Then, hit `G`, followed by `O`, and paste the following lines at the end of the
file:
```shell
kernel.shmmax = 500000000
kernel.shmmni = 4096
kernel.shmall = 4000000000
kernel.sem = 250 512000 100 2048
kernel.sysrq = 1
kernel.core_uses_pid = 1
kernel.msgmnb = 65536
kernel.msgmax = 65536
kernel.msgmni = 2048
net.ipv4.tcp_syncookies = 1
net.ipv4.ip_forward = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_max_syn_backlog = 4096
net.ipv4.conf.all.arp_filter = 1
net.ipv4.ip_local_port_range = 1025 65535
net.core.netdev_max_backlog = 10000
net.core.rmem_max = 2097152
net.core.wmem_max = 2097152
vm.overcommit_memory = 2
```
Hit the escape key, then type `:wq` to finish saving the file. Verify that you
see the changes that you made by typing in `cat /etc/sysctl.conf`.


##5: Reboot the Guest OS
We will have to reboot the guest OS to allow the changes to the ``sysctl.conf`
file to take effect. In the guest shell, type in
```shell
exit
```

Then, halt the virtual machine by typing the following command (in the host
OS shell as you are there now!):

```shell
vagrant halt
```

Now, before bringing up vagrant again, we will need to make sure that we have
VirtualBox guest addtions setup. This sadly requires an extra step (which
hopefully will go away with future version of Vagrant and VirtualBox). From
the host terminal, type in the following command: 

```shell
vagrant plugin install vagrant-vbguest
```

Now bring up the virtual box again, by typing (in the host OS terminal), the
following commands:
```shell
vagrant up
vagrant ssh
```

You are back in the guest OS shell! But, now you have GPDB compiled, installed,
and ready to run. 


##6: Initialize and start GPDB
Next we initialize GPDB using the config file we created above. In the *guest*
shell type in:
```shell
$GPHOME/bin/gpinitsystem -a -c $GPDATA/gpinitsystem_config
```
Once initialized, you are now ready to run GPDB. One more configuration to set,
which you can do by typing into the guest terminal the following command:

```shell
printf '# Remember the master data directory location\n' >> ~/.bashrc
printf 'export MASTER_DATA_DIRECTORY=$GPDATA/master/gpseg-1\n' >> ~/.bashrc
source ~/.bashrc
```

Now, you are ready to start GPDB. To do that, type in the following commands
into the (guest) terminal: 

```shell
DBNAME=$USER
createdb $DBNAME
psql $DBNAME
```

You should see a psql prompt that says `vagrant=#`. At this point, you can open
up another shell from the host OS. Start another (host) terminal, and 
`cd gpdb/vagrant`, and then type in `vagrant ssh`. From this second guest
terminal, you can run GPDB commands like `gpstate`. Go ahead and try it. You
should see a report that states that you have a master and two segments. 

If you want to try out a few SQL commands, go back to the guest shell in which
you have the psql prompt, and issue the following SQL commands: 

```sql
-- Create and populate a Users table
CREATE TABLE Users (uid INTEGER PRIMARY KEY, name VARCHAR);
INSERT INTO Users 
  SELECT generate_series, md5(random()) FROM generate_series(1, 100000);

-- Create and populate a Messages table
CREATE TABLE Messages (mid INTEGER PRIMARY KEY, 
                       uid INTEGER REFERENCES Users(uid), 
                       ptime DATE, message VARCHAR);
INSERT INTO Messages 
   SELECT generate_series, 
          round(random()*100000), 
          date(now() - '1 hour'::INTERVAL * round(random()*24*30)), 
          md5(random())::text 
   FROM generate_series(1, 1000000);

-- Report the number of tuples in each table
SELECT COUNT(*) FROM Messages;
SELECT COUNT(*) FROM Users;

-- Report how many messages were posted on each day
SELECT M.ptime, COUNT(*) 
FROM Users U NATURAL JOIN Messages M 
GROUP BY M.ptime 
ORDER BY M.ptime;
```

As you can see, you created a simple warehouse database simulating users posting
messages on a social media network, and reported the number of messages that
were posted on each day. 

##7: Using GDB
If you are doing serious development, you will likely need to use a debugger.
Here is how you do that. 
* First, list the Postgres processes by typing in (a guest terminal):
`ps ax | grep postgres`. You should see a list that looks something like:
![Postgres processes](/vagrant/pictures/gpdb_processes.png)
Here the key processes are the ones that were started as 
`/gpdb/vagrant/install/bin/postgres`. The master is the process (pid 25486 
in the picture above) that has the word "master" in the `-D` parameter setting,
whereas the segment hosts have the word "gpseg" in the `-D` parameter setting.

* Start ``gdb`` from a guest terminal. Once you get a prompt in gdb, type in the
following (the pid you specify in attach will be different for you):
```gdb
set follow-fork-mode child
b ExecutorMain
attach 25486
```
Of course, you can change which function you want to break into, and change
whether you want to debug the master or the segment processes. Happy hacking!