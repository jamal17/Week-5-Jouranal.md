#How Linux Works: What Every Superuser Should Know Journal Notes from Chapters 6 & 7



Chapter 6. How User Space Starts

The point at which the kernel starts the first user space operation, init, it is important - not only because this is where the memory 
and the CPU are finally ready to run the normal system, but because this is where we can see how the rest of the system accumulates.

In this point the user space is much more modular. It is easy to find out what happen into the user startup space and playback. And it 
is easy to change the user's startup because doing so does not require low-level programming.

User space starts in roughly this order:

1.   init

2.   Essential low-level services such as udevd and syslogd

3.   Network configuration

4.   Mid- and high-level services (cron, printing, and so on)

5.   Login prompts, GUIs, and other high-level applications

 Introduction to init

init is a user space program like just any other program on the Linux system.

Its main purpose is to start and stop basic service operations on the system.

newer versions have more responsibilities.

There are three major implementations of init in Linux distributions:

System V init. A traditional sequenced init.

systemd. The standard for init. 

Upstart. The init on Ubuntu installations.

There are various other versions of init as well, especially on embedded platforms. 

For example, Android has its own init. 

The BSDs also have their version of init.

There are many different implementations of init.

Other limitations are that we can only start a fixed set of services as specified by the boot sequence: when you connect new devices, 
or need a service that is not already running, there is no uniform way to format the new components with init. 

Their implementations are quite different, though:

systemd is goal oriented. define a target that we want to achieve, when we want to reach the target. systemd
satisfies the dependencies and resolves the target. systemd can also defer the start of a service until it is absolutely needed.

Upstart is reactionary. 

It receives events and, based on those events, runs jobs that can in turn produce more events, causing Upstart to run more jobs, 
and so on.

The systemd and Upstart init systems also offer a more advanced way to start and track services. 

In traditional init systems, service expected to start themselves from scripts. A script runs a daemon program

we need to use ps or some other mechanism specific to the service to find the PID. 

In contrast, Upstart and systemd can manage individual deamons from the start, giving the user more power and insight into
exactly what is running on the system.

Because the new init systems are not script-centric, configuring services for them also tends to be easier. 

Finally, systemd and Upstart both offer some level of on-demand services.

Both systemd and Upstart offer some System V backward compatibility. For example, both support the concept of
runlevels.

System V Runlevels

At any given time on a Linux system, a certain base set of processes (such as crond and udevd) is running. In System V init, this 
state of the machine is called its runlevel, which is denoted by a number from 0 through 6.

A system spends most of its time in a single runlevel

when we shut down the machine, init switches to a different runlevel in order to terminate the system services in
an orderly fashion and to tell the kernel to stop.

We can check our system’s runlevel with the who -r command.

$ who -r

run-level

2  2015-09-06 08:37

the output tells us that the current runlevel is 2, and the date and time that the runlevel was established.

Runlevels serve various purposes, but the most common one is to distinguish between system startup, shutdown,
single-user mode, and console mode states. 

For example, Fedora-based systems traditionally used runlevels 2 through 4 for the text console; a runlevel of 5
means that the system will start a GUI login.

Runlevels becoming a past. 

Identifying Your init

Before any proceeding, we need to know our system’s version of init. 

To check our system as follows:

If the system has /usr/lib/systemd and /etc/systemd directories, you have systemd. 

If you have an /etc/init directory that contains several .conf files, we are probably running Upstart.

If not of any above is true, and we have an /etc/inittab file, we are probably running System V init. 

systemd

The systemd init is one of the newest init implementations on Linux. 

There are so many systemd features that it can be very difficult to know where to start learning the basics.

This is will happen when system runs at boot time:

1.   systemd loads its configuration.

2.   systemd determines its boot goal, which is usually named default.target.

3.   systemd determines all of the dependencies of the default boot goal, dependencies of these dependencies, and so on.

4.   systemd activates the dependencies and the boot goal.

5.   After boot, systemd can react to system events (such as uevents) and activate additional components.

 

Units and Unit Types

One of the most interesting things about systemd is that it does not just operate processes and services; it can
also mount filesystems, monitor network sockets, run timers, and more.

Each type of capability is called a unit type, and each specific capability is called a unit.


These are a few of the unit types that perform the boot-time tasks required in any Unix system:

Service units. Control the traditional service daemons on a Unix system.

Mount units. Control the attachment of filesystems to the system.

Target units. Control other units, usually by grouping them.

The default boot goal is usually a target unit that groups together many service and mount units as dependencies. 

As a result, it’s easy to get a partial picture of what’s going to happen when you boot, and you can even
create a dependency tree diagram with the systemctl dotcommand. 

systemd Dependencies

Boot-time and operational dependencies are more complicated than they may seem at first because strict
dependencies are too inflexible. 

For example, imagine a scenario in which want to display a login prompt after starting a database server, so define
a dependency from the login prompt to the database server.

Unix boot-time tasks are fairly fault tolerant and can often fail without causing serious problems for standard
services. 

For example, if a data disk for a system was removed but its /etc/fstab entry remained, the
initial file-system mount would fail. 

To accommodate the need for flexibility and fault tolerance, systemd offers a myriad of dependency types
and styles. We’ll label them by their keyword syntax.

Let’s first look at the basic types:

Requires Strict dependencies. 

When activating a unit with a Requires dependency unit, systemd attempts to activate the dependency unit. If the dependency unit 
fails, systemd deactivates the dependent unit.

Wants. Dependencies for activation only. Upon activating a unit, systemd activates the unit’s Wants dependencies,
but it doesn’t care if those dependencies fail.

Requisite. Units that must already be active. Before activating a unit with a Requisite dependency, systemd first
checks the status of the dependency. If the dependency has not been activated, systemd fails on activation of the unit with the 
dependency.

Conflicts. Negative dependencies. When activating a unit with a Conflict dependency, systemd automatically deactivates
the dependency if it is active. Simultaneous activation of two conflicting units fails.

Ordering

None of the dependency syntax that we’ve seen so far explicitly specifies order. 

To activate units in a particular order:

Before. The current unit will activate before the listed unit(s). For example, if Before=bar.target appears
in foo.target, systemd activates foo.target before bar.target.

After. The current unit activates after the listed unit(s).

Conditional Dependencies

alot of dependency condition keywords operate on various operation system states
rather than systemd units. 

For example:

ConditionPathExists=p: True if the (file) path p exists in the system.

ConditionPathIsDirectory=p: True if p is a directory.

ConditionFileNotEmpty=p: True if p is a file and it’s not zero-length.

The unit will not activate if a conditional dependency in a unit is false when systemd tries to activate the unit,.

If we activate a unit that has a condition dependency as well as some other unit dependencies, systemd attempts to activate the unit 
dependencies regardless of whether the condition is true or false.

Other dependencies are primarily variations on the preceding. 

For example, the RequiresOverridable dependency is just like Requires when running normally, but it acts like a Wants
dependency if a unit is manually activated. 

.

 systemd Configuration

The systemd configuration files are spread among many directories across the system, so you typically won’t find
the files for all the units on a system in one place. 

That said, there are two main directories for systemd configuration: the system unit directory (globally configured, 
usually/usr/lib/systemd/system) and a system configuration directory (local definitions, usually /etc/systemd/system).

So when given the choice between modifying something in /usr and /etc, always change /etc.

Variables and Specifiers

The sshd.service unit file also shows use of variables—specifically, the $OPTIONS and $MAINPID environment variables that are
passed in by systemd. $OPTIONS are options that you can pass to sshd when you activate the unit with systemctl,
and $MAINPID is the tracked process of the service .

systemd Operation

we’ll interact with systemd primarily through the systemctl command, which allows us to activate and deactivate services, list status, 
reload the configuration, and much more.

systemctl list-units

The output format is typical of a Unix information-listing command. 

For example, the header and the line for media.mount would look like this:

UNIT                   LOAD   ACTIVESUB      JOBDESCRIPTION

media.mount      loaded active mounted        MediaDirectory

This command produces a lot of output.

Adding Units to systemd

You should normally put our own unit files in the system configuration directory /etc/systemd/system 

Because it’s easy to create target units that don’t do anything and don’t interfere with anything, we should try it. 

Here’s how to create two targets, one with a dependency on the other:

1.   Create a unit file named test1.target:

2.   Create a test2.target file with a dependency on test1.target:

3.   Activate the test2.target unit (remember that the dependency in test2.target causes systemd to activate test1.target when
you do this):  systemctl start test2.target

4.   Verify that both units are active: $ systemctl status test1.target test2.target

 

Removing Units

To remove a unit, we follow these steps:

1.   Deactivate the unit if necessary: # systemctl stop unit

2.   If the unit has an [Install] section, disable the unit to remove any dependent symbolic links: # systemctl disable unit

3.   Remove the unit file, if you like.

 

systemd Process Tracking and Synchronization

systemd wants a reasonable amount of information and control over every process that it starts. 

main problem that it faces is that a service can start in different ways; it may fork new instances of itself or
even daemonize and detach itself from the original process.

We use the Type option in our service unit file to indicate its startup behavior. 

There are two basic startup styles:

Type=simple The service process doesn’t fork.

Type=forking The service forks, and systemd expects the original service process to terminate.

However, some Type startup styles can indicate that the service itself will notify systemd when it is ready:

Type=notify The service sends a notification specific to systemd (with the sd_notify() function call) when it’s ready.

Type=dbus The service registers itself on the D-bus (Desktop Bus) when it’s ready.

systemd On-Demand and Resource-Parallelized Startup

One of systemd’s most significant features is its ability to delay a unit startup until it is absolutely needed. 

The setup typically works like this:

1.   we create a systemd unit (call it Unit A) for the system service that you’d like to provide, as normal.

2.   we identify a system resource such as a network port/socket, file, or device that Unit A uses to offer its services.

3.   we create another systemd unit, Unit R, to represent that resource. 

These units have special types such as socket units, path units, and device units.

Operationally, it goes like this:

1.   Upon activation of Unit R, systemd monitors the resource.

2.   When anything tries to access the resource, systemd blocks the resource, and the input to the resource is buffered.

3.   systemd activates Unit A.

4.   When the service from Unit A is ready, it takes control of the resource, reads the buffered input, and runs normally.

There are a few concerns:

we must make sure that our resource unit covers every resource that the service provides.

we need to make sure our resource unit is tied to the service unit that it represents. 

 

systemd System V Compatibility

One feature that sets systemd apart from other newer-generation init systems is that it tries to do a more complete
job of tracking services started by System V–compatible init scripts. It works like this:

1.   First, systemd activates runlevel<N>.target, where N is the runlevel.

2.   For each symbolic link in /etc/rc<N>.d, systemd identifies the script in /etc/init.d.

3.   systemd associates the script name with a service unit (for example, /etc/init.d/foo would be foo.service).

4.   systemd activates the service unit and runs the script with either a start or stop argument, based on its name in rc<N>.d.

5.   systemd attempts to associate any processes from the script with the service unit.

 

systemd Auxiliary Programs

When we are starting out with systemd, we may notice the exceptionally large number of programs in /lib/systemd.

These are primarily support programs for units. 

For example, udevd is part of systemd, 

Upstart

The Upstart version of init revolves around jobs and events. 

Jobs are startup and runtime actions for Upstart to perform (such as system services and configuration), and
events are messages that Upstart receives from itself or other processes (such as udevd). Upstart works by starting jobs in response 
to  events.

How this works, we consider the udev job for starting the udevd daemon. Its configuration file is typically /etc/init/udev.conf, which 
includes the following:

start on virtual-filesystems

stop on runlevel [06] 

Upstart Initialization Procedure

Upon startup, Upstart does the following:

1.   Loads its configuration and the job configuration files in /etc/init.

2.   Emits the startup event.

3.   Runs jobs configured to start upon receiving the startup event.

4.   These initial jobs emit their own events, triggering more jobs and events.
 
Upon finishing all jobs associated with a normal startup, Upstart continues to monitor and react to events during the entire system 
uptime.

Most Upstart installations run like this:

1.   The most significant job that Upstart runs in response to the startup event is mountall. This job attaches all
necessary local and virtual filesystems to the currently running system so that everything else can run.

2.   The mountall job emits a number of events, including filesystem, virtual-filesystems, local-filesystems, remote-filesystems,
and all-swaps, among others. These events indicate that the important filesystems on the system are now attached and ready.

3.   In response to these events, Upstart starts a number of essential service jobs.

For example, udev starts in response to the virtual-filesystems event, and dbus starts in response to the local-filesystems event.

4.   Among the essential service jobs, Upstart starts the network-interfaces job, usually in response to the local-filesystems event
and udevd being ready.

5.   The network-interfaces job emits the static-network-up event.

6.   Upstart runs the rc-sysinit job in response to the filesystem and static-network-up events. 

Upstart Jobs

Each file in the Upstart /etc/init configuration directory corresponds to a job, and the main configuration file for each job has 
a .conf extension.


For example, /etc/init/mountall.conf defines the mountall job.

There are two primary kinds of Upstart jobs:

Task jobs. These are jobs with a clear end. 

For example, mountall is a task job because it terminates when finished mountin  g filesystems.

Service jobs. These jobs have no defined stop. Servers (daemons) such as udevd, database servers, and web servers are all service jobs.

Viewing Jobs

You can view Upstart jobs and job status with the initctl command. To get an overview of what’s happening on your system, run:

$ initctl list

You’ll get a lot of output, so let’s just look at two sample jobs that might appear in a typical listing. Here’s a simple example of a 
task job status:

Job State Transitions

There are many job states, but there’s a set way to move between them. For example, here’s how a typical job starts:

1.   All jobs begin in the stop/waiting status.

2.   When a user or a system event starts a job, the job’s goal changes from stop to start.

3.   Upstart changes the job’s state from waiting to starting, so the status is now start/starting.

4.   Upstart emits a starting job event.

5.   The job performs whatever it needs to do for the starting state.

6.   Upstart changes the job’s state from starting to pre-start and emits the pre-start job event.

7.   The job works its way through several more states until it hits the running state.

8.   Upstart emits a started job event.

 

Upstart Configuration

One for the task job mountall and the other for the service job tty1. 

Like all Upstart configuration files, the configuration files are in /etc/init, and they are named mountall.conf and tty1.conf. 

The configuration files are organized into smaller pieces called stanzas. 

Each stanza starts with a leading keyword, such as description or start.

To get started, open the mountall.conf file on your system. 

Look for a line like this in the first stanza:

description  "Mount filesystems on boot"

This stanza gives a short text description of the job.

Next you’ll see a few stanzas describing how the mountall job starts:

start on startup

stop on starting rcS

Here, the first line tells Upstart to start the job upon receiving the startup event (the initial event that Upstart emits). The 
second  line tells Upstart to terminate the job upon receiving the rcS event, when the system goes into single-user mode.

The next two lines tell Upstart how the mountall job behaves:

expect daemon task

A Service Job: tty1

The service job tty1 is much simpler; it controls a virtual console login prompt. Its entire configuration file, tty1.conf, looks like 
this: start on stopped rc RUNLEVEL=[2345] and (not-container or container CONTAINER=lxc or container CONTAINER=lxc-libvirt) stop on 
runlevel [!2345] respawn exec /sbin/getty -8 38400 tty1

The most complicated part of this job is actually when it starts, but for now, ignore the container lines and concentrate on this 
portion: start on stopped rc RUNLEVEL=[2345]

This part tells Upstart to activate the job upon receiving a stopped rc event from Upstart when the rc task job has run and 
terminated. 

Process Tracking and the Upstart expect Stanza

Because Upstart tracks processes in jobs once they’ve started (so that it can terminate and restart them
efficiently), it wants to know which processes are relevant to each job. 

This can be a difficult task, because in the traditional Unix startup scheme, processes fork from others
during startup to become daemons, and the main process for a job may start after one or two forks. 

You tell Upstart how a job behaves with the expect stanza. 

There are four basic possibilities:

Noexpectstanza the main job process does not fork. Track the main process.

expect fork the process forks once. Track the forked process.

expect daemon the process forks twice. Track the second fork.

expect stop the job’s main process will raise a SIGSTOP signal to indicate that it is ready. 

For Upstart and other modern versions of init, such as systemd, the ideal case is the first one
(no expect stanza), because the main job process doesn’t have to include any of its own startup and shutdown mechanics. 

Many traditional service daemons already include debugging-style options that tell the main process to not fork.


Examples include the Secure Shell daemon, sshd, and its -D option. A look at the /etc/init/ssh.conf startup
stanzas reveals a simple configuration to start sshd, prevent rapid respawning, and eliminate spurious output to stderr:

respawn

respawn limit 10 5 umask 022 # 'sshd -D' leaks stderr and confuses things in conjunction with 'console log'

console none

--snip--

exec /usr/sbin/sshd -D

Among jobs that require an expect stanza, expect fork is the most common. For example, here’s the startup portion of 
the /etc/init/cron.conf file:

expect fork

respawn

exec cron

A simple job startup like this usually indicates a well-behaved, stable daemon.

Upstart Operation

To start an Upstart job, use initctl start:

$ initctl start job

To stop a job, use initctl stop:

$ initctl stop job

To restart a job:

$ initctl restart job

If you need to emit an event to Upstart, you can do it manually with:

$ initctl emit event

You can also add environment variables to the emitted event by adding key=value parameters after event.

 

Upstart Logs

There are two basic kinds of logs in Upstart: service job logs, and diagnostic messages that Upstart itself produces. 

Service job logs record the standard output and standard error of the scripts. 

Upstart Runlevels and System V Compatibility

Here’s a more detailed overview of how it works on Ubuntu systems:

1.   The rc-sysinit job runs, usually after getting the filesystem and static-network-up events. Before it runs, there is no runlevel.

2.   The rc-sysinit job determines which runlevel to enter. Usually, the run-level is the default, but it can also parse an 
older /etc/inittab file or take the runlevel from a kernel parameter (in /proc/cmdline).

3.   The rc-sysinit job runs telinit to switch the runlevel. The command emits a runlevel event, specifying the runlevel in 
the RUNLEVEL environment variable.

4.   Upstart receives the runlevel event. A number of jobs are configuredto start on the runlevel event paired with a certain 
runlevel, and Upstart sets these in motion.

5.   One of the runlevel-activated task jobs, rc, is responsible for running the System V start. In order to do so, the rc job
runs /etc/init.d/rc, just as System V init would (see 6.6 System V init).

6.   Once the rc job terminates, Upstart can start a number of other jobs upon receiving the stopped rc event (such as the tty1 job
in A Service Job: tty1).

System V init

The System V init implementation on Linux dates to the early days of Linux; its core idea is to support an orderly
bootup to different runlevels with a carefully sequenced process startup. 

Though System V is now uncommon on most desktop installations.

There are two major components to a typical System V init installation: a central configuration file and a large
set of boot scripts augmented by a symbolic link farm. The configuration file /etc/inittab is where it all starts. 

If you have System V init, look for a line like the following in yourinittab file: id:5:initdefault:

This indicates that the default runlevel is 5.

All lines in inittab take the following form, with four fields separated by colons in this order:

A unique identifier (a short string, such as id in the previous example)

The applicable runlevel number(s)

The action that init should take (default runlevel to 5 in the previous example)

A command to execute (optional)

To see how commands work in an inittab file, consider this line:

l5:5:wait:/etc/rc.d/rc 5 respawn

You’re likely to see something like this in an inittab file:

1:2345:respawn:/sbin/mingetty tty ctrlaltdel

The ctrlaltdel action controls what the system does when you press CTRLALT-DEL on a virtual console. 

On most systems, this is some sort of reboot command, using the shutdown command. 

sysinit

The sysinit action is the first thing that init should run when starting, before entering any runlevels.

System V init: Startup Command Sequence

In System V init starts system services, just before it lets us to log in. Recall this inittab line from earlier:

l5:5:wait:/etc/rc.d/rc 5

The System V init Link Farm

The contents of the rc*.d directories are actually symbolic links to files in yet another directory, init.d. If your goal is to 
interact with, add, delete, or modify services in the rc*.d directories, you need to understand these symbolic links. 

A long listing of a directory such as rc5.d reveals a structure like this:

lrwxrwxrwx . . . S10sysklogd -> ../init.d/sysklogd

lrwxrwxrwx . . . S12kerneld -> ../init.d/kerneld

lrwxrwxrwx . . . S15netstd_init -> ../init.d/netstd_init

lrwxrwxrwx . . . S18netbase -> ../init.d/netbase

--snip--

lrwxrwxrwx . . . S99httpd -> ../init.d/http

Starting and Stopping Services

To start and stop services by hand, use the script in the init.d directory. 

For example, one way to start the httpd web server program manually is to run init.d/httpd start. Similarly, to kill a running 
service, you can use the stop argument (httpd stop, for instance).

Modifying the Boot Sequence

Changing the boot sequence in
System V init is normally done by modifying the link farm. 

The most common change is to
prevent one of the commands in the init.d directory from running
in a particular runlevel.

One of the best ways to do it is to
add an underscore (_) at the beginning of the link name, like this:

                $
mv S99httpd _S99httpd

 

run-parts

The mechanism that System V init
uses to run the init.d scripts has found its way into many
Linux systems, regardless of whether they use System V init. It’s a utility
called run-parts, and the only thing it does is run a bunch of executable
programs in each directory, in predictable order. 

The default behavior is to run all programs
in a directory, but we often have the option to select certain programs and
ignore others. 

In some distributions, we don’t
need much control over the programs that run. 

For example, Fedora ships with a
very simple run-parts utility.

 

Controlling System V init

Occasionally, we’ll need to give
init a little kick to tell it to switch runlevels, to reread its configuration,
or to shut down the system. 

To control System V init, we use telinit.


For example, to switch to runlevel
3.

$ telinit
3

When we need to add or remove jobs.

The telinit command for
this is:

$ telinit
q

 

Shutting Down Your System

init controls how the system shuts
down and reboots. 

The commands to shut down the
system are the same regardless of which version of init you run. 

The proper way to shut down a Linux
machine is to use the shutdown command.

There are two basic ways to
use shutdown. 

If you halt the
system, it shuts the machine down and keeps it down. 

To make the machine halt
immediately, run this:

$ shutdown
-h now

To make the system reboot in 10 minutes,
enter:

$ shutdown
-r +10

.

The Initial RAM Filesystem

The Linux boot process is, for the
most part, straightforward. However, one component has always been somewhat
confounding: initramfs, or the intitial RAM filesystem.


The problem stems from the
availability of many kinds of storage hardware. 

Remember, the Linux kernel does not
talk to the PC BIOS or EFI interfaces to get data from disks, so to mount its
root file-system, it needs driver support for the underlying storage mechanism.


For example, if the root is on a
RAID array connected to a third-party controller, the kernel needs the driver
for that controller first. 

Unfortunately, there are so many
storage controller drivers that distributions can’t include all of them in
their kernels, so many drivers are shipped as loadable modules. 

It’s easy to see the contents of
your initial RAM filesystem because, on most modern systems, they are
simple gzip-compressed cpio archives (see the cpio(1) manual
page). 

First, find the archive file by
looking at your boot loader configuration (Then use cpio to dump the
contents of the archive into a temporary directory somewhere and peruse the
results. For example:

$ mkdir
/tmp/myinitrd

$ cd
/tmp/myinitrd

$ zcat
/boot/initrd.img-3.2.0-34 | cpio -i --no-absolute-filenames

 

Emergency Booting and Single-User
Mode

When something goes wrong with the
system, the first recourse is usually to boot the system with a distribution’s
“live” image (most distributions’ installation images double as live images) or
with a dedicated rescue image such as SystemRescueCd that you can put on
removable media. Common tasks for fixing a system include the following:

Checking
filesystems after a system crash

Resetting
a forgotten root password

Fixing
problems in critical files, such as /etc/fstab and /etc/passwd

Restoring
from backups after a system crash

﻿

 

Chapter 7. System Configuration: Logging, System Time, Batch Jobs,
and Users

The subject material in this chapter covers the parts of the
system.

Configuration files that the system
libraries access to get server and user information

Server programs (sometimes
called daemons) that run when the system
boots

Configuration utilities that can be
used to tweak the server programs and configuration files

Administration utilities

The Structure of /etc

Most system configuration files on a Linux system are found
in /etc.
Historically, each program had one or more configuration files there, and
because there are so many packages on a Unix system, /etc would
accumulate files quickly.

There were two problems with this approach: 

It was hard to find configuration files on a running system. and it was difficult to
maintain a system configured this way. 



For example, if you wanted to change the system logger
configuration, we’d have to edit /etc/syslog.conf. But after your
change, an upgrade to your distribution could wipe out your customizations.

 

 System Logging

Most system programs write their diagnostic output to the syslog service.


 

The System Logger

The system logger is one of the most important parts of the
system. 

It is good to check when something goes wrong and we don’t know
where to start, we will check the system log files first. 

Here is a sample log file message:

Aug 19 17:59:48 duplex sshd[484]:
Server listening on 0.0.0.0 port 22.

 

Configuration Files

A traditional rule has a selector and an action to
show how to catch logs and where to send them, respectively. For example:

Example 7-1. syslog rules

kern.*                      
/dev/console

*.info;authpriv.none➊        /var/log/messages

authpriv.*                  
/var/log/secure,root

mail.*       
               /var/log/maillog

cron.*                      
/var/log/cron

*.emerg                     
*➋

local7.*                    
/var/log/boot.log

Facility and Priority

The selector is a pattern that matches the facility and priority of
log messages. 

The facility is a general category of message. 

The function of most facilities will be obvious from their name. 

Troubleshooting

One of the easiest ways to test the
system logger is to send a log message manually with
the logger command, as shown here:

$ logger
-p daemon.info something bad just happened

 

User Management Files

Unix systems allow multiple independent users. 

At the kernel level, users are simply numbers (user
IDs.

Usernames exist only in user space, so any program that works with
a username generally needs to be able to map the username to a user ID if it
wants to refer to a user when talking to the kernel.

 

The /etc/passwd File

The plaintext file /etc/passwd maps
usernames to user IDs. It looks something like this:

Example 7-2. A list of
users in /etc/passwd

root:x:0:0:Superuser:/root:/bin/sh

daemon:*:1:1:daemon:/usr/sbin:/bin/sh

bin:*:2:2:bin:/bin:/bin/sh

sys:*:3:3:sys:/dev:/bin/sh

nobody:*:65534:65534:nobody:/home:/bin/false

juser:x:3119:1000:J. Random
User:/home/juser:/bin/bash

beazley:x:143:1000:David
Beazley:/home/beazley:/bin/bash

Each line represents one user and
has seven fields separated by colons. 

The user ID (UID),
which is the user’s representation in the kernel. You can have two entries with
the same user ID, but doing this will confuse you, and your software may mix
them up as well. Keep the user ID unique.

The group ID (GID).
This should be one of the numbered entries in the /etc/group file.
Groups determine file permissions and little else. This group is also called
the user’s primary group.

The user’s real name (often called
the GECOS field).
You’ll sometimes find commas in this field, denoting room and telephone
numbers.

The user’s home directory.

 

Special Users

 will find a few special
users in /etc/passwd. 

The superuser (root) always has
UID 0 and GID 0.

The users that cannot log in are called pseudo-users.


The /etc/shadow File

The shadow password file (/etc/shadow) on a Linux system
normally contains user authentication information, including the encrypted
passwords and password expiration information that correspond to the users
in /etc/passwd.

 

Manipulating Users and Passwords

Regular users interact with /etc/passwd using
the passwd command. 

Changing /etc/passwd as the
Superuser

Because /etc/passwd is plaintext, the
superuser may use any text editor to make changes. 

To add a user, simply add an appropriate line and create a home
directory for the user; to delete, do the opposite. 

To edit the file, you’ll most likely want to use
the vipw program, which backs up and locks /etc/passwd while
you’re editing it as an added precaution. To edit /etc/shadow instead
of /etc/passwd,
use vipw -s. (You’ll likely never need to do this, though.)

Most organizations frown on editing passwd directly
because it’s too easy to make a mistake. 

 

Working with Groups

Groups in Unix offer a way to share
files with certain users but deny access to all others. The idea is that you
can set read or write permission bits for a group, excluding everyone else. 

This feature was once important because many users shared one
machine, but it’s become less significant in recent years as workstations are
shared less often.

The /etc/group file
defines the group IDs (such as the ones found in the /etc/passwd file). Example 7-3 is
an example.

Example 7-3. A sample
/etc/group file

root:*:0:juser

daemon:*:1:

bin:*:2:

sys:*:3:

adm:*:4:

disk:*:6:juser,beazley

nogroup:*:65534:

user:*:1000:

Like the /etc/passwd file,
each line in /etc/group is a set of fields
separated by colons. The fields in each entry are as follows, from left to
right:

The group name. This appears when you run a command like ls -l.

The group password. This is hardly ever used, nor should you use it
(use sudo instead). Use * or any other default value.

The group ID (a number). The GID must be unique within the group file.
This number goes into a user’s group field in that user’s /etc/passwd entry.

An optional list of users that
belong to the group. In addition to the users
listed here, users with the corresponding group ID in their passwd file
entries also belong to the group.

 

getty and login

getty is a program that attaches to terminals and displays a
login prompt. 

$ ps ao
args | grep getty

/sbin/getty 38400 tty1

 

Setting the Time

Unix machines depend on accurate timekeeping. 

The kernel maintains the system clock, which is the clock
that is consulted when you run commands like date. 

PC hardware has a battery-backed real-time clock (RTC).
The RTC isn’t the best clock in the world, but it’s better than nothing. The
kernel usually sets its time based on the RTC at boot time, and you can reset
the system clock to the current hardware time with hwclock. Keep your
hardware clock in Universal Coordinated Time (UTC) to avoid any trouble with
time zone or daylight savings time corrections. You can set the RTC to your
kernel’s UTC clock using this command:

# hwclock
--hctosys --utc

the kernel is even worse at keeping time than the RTC, and because
Unix machines often stay up for months or years on a single boot, they
tend to develop time drift. Time drift is the current
difference between the kernel time and the true time (as defined by an atomic
clock or another very accurate clock).

 

Kernel Time Representation and Time Zones

The kernel’s system clock represents the current time as the
number of seconds since 12:00 midnight on January 1, 1970, UTC. To see this
number now, run:

$ date +%s

To convert this number into something that humans can read, user-space
programs change it to local time and compensate for daylight savings time and
any other strange circumstances (such as living in Indiana). 

The local time zone is controlled by the file /etc/localtime.
(Don’t bother trying to look at it; it’s a binary file.)

The time zone files on your system are in /usr/share/zoneinfo.
You’ll find that this directory contains a lot of time zones and a lot of
aliases for time zones. 

To use a time zone other than the system default for just one
shell session, set the TZ environment variable to the name of a file
in /usr/share/
zoneinfo and test the change, like this:

$ export
TZ=US/Central

$ date

As with other environment
variables, you can also set the time zone for the duration of a single command
like this:

$ TZ=US/Central
date

7.5.2 Network Time

If you need to do the configuration by hand, you’ll find help on
the main NTP web page at http://www.ntp.org/, but if
you’d rather not read through the mounds of documentation there, do this:

1.    Find the
closest NTP time server from your ISP or from the ntp.org web
page.

2.    Put that time
server in /etc/ntpd.conf.

3.   
Run ntpdate server at boot time.

4.   
Run ntpd at boot time, after the ntpdate command.

If your machine doesn’t have a
permanent Internet connection, you can use a daemon like chronyd to
maintain the time during disconnections.

You can also set your hardware
clock based on the network time in order to help your system maintain time
coherency when it reboots. (Many distributions do this automatically.) To do
so, set your system time from the network
with ntpdate (or ntpd), then run the command you saw back
in Note:

$ hwclock --systohc –-utc

 

Scheduling Recurring Tasks with cron

Unix cron service runs programs repeatedly on a fixed schedule. 

You can run any program with cron at whatever times suit you. 

The program running through cron is called a cron job.


To install a cron job, you’ll create an entry line in your crontab
file, usually by running the crontab command. 

For example, the crontab entry schedules
the/home/juser/bin/spmake command daily at 9:15 AM:

15 09 * * * /home/juser/bin/spmake

The fields are as follows, in
order:

Minute (0 through 59). The cron job
above is set for minute 15.

Hour (0 through 23). The job above
is set for the ninth hour.

Day of month (1 through 31).

Month (1 through 12).

Day of week (0 through 7). The
numbers 0 and 7 are Sunday.

 

A star (*) in any field means to
match every value. The preceding example runs spmake daily because
the day of month, month, and day of week fields are all filled with stars,
which cron reads as “run this job every day, of every month, of every week.”

To run spmake only on the
14th day of each month, you would use this crontab line:

15 09 14 *
* /home/juser/bin/spmake

You can select more than one time
for each field. For example, to run the program on the 5th and the 14th day of
each month, you could enter 5,14 in the third field:

15 09 5,14 *
* /home/juser/bin/spmake

 

Installing Crontab Files

Each user can have his or her own
crontab file, which means that every system may have multiple crontabs, usually
found in /var/spool/cron/crontabs. 

Normal users can’t write to this
directory; the crontab command installs, lists, edits, and removes a
user’s crontab.

 

System Crontab Files

 Linux distributions
normally have an /etc/crontab file. 

Don’t use crontab to edit this file, because this
version has an additional field inserted before the command to run—the user
that should run the job. For example, this cron job defined in /etc/crontab runs
at 6:42 AM as the superuser.

 

The Future of cron

The cron utility is one of the oldest components of a Linux
system; it’s been around for decades (predating Linux itself), and its
configuration format hasn’t changed much for many years. 

When something gets to be this old, it becomes fodder for
replacement, and there are efforts underway to do exactly that.

 

Scheduling One-Time Tasks with at

To run a job once in the future
without using cron, use the at service. 

For example, to
run myjob at 10:30 PM, enter this command:

$ at 22:30

at> myjob

End the input with CTRL-D.
(The at utility reads the commands from the standard input.)

To check that the job has been
scheduled, use atq. To remove it, use atrm. You can also schedule
jobs days into the future by adding the date in DD.MM.YY format, for
example, at 22:30 30.09.15.

 

Understanding User IDs and User Switching

We’ve discussed how setuid programs such
as sudo and su allow you to change users, and we’ve
mentioned system components like login that control user access. 

Perhaps you’re wondering how these pieces work and what role the
kernel plays in user switching.

There are two ways to change a user ID, and the kernel handles
both. 

setuid executable, which is covered in 2.17 File Modes and
Permissions. through the setuid() family of system calls. 



There are a few different versions
of this system call to accommodate the various user IDs associated with a
process, as you’ll learn in 7.8.1 Process Ownership, Effective UID,
Real UID, and Saved UID.

The kernel has basic rules about
what a process can or can’t do, but here are the three basics:

A process running as root (userid 0) can use setuid() to
become any other user.

A process not running as root has severe restrictions on how it
may use setuid(); in most cases, it cannot.

Any process can execute a setuid program as long as it has
adequate file permissions.

 

Process Ownership, Effective UID, Real UID, and Saved UID

Our discussion of user IDs so far has been simplified. 

Every process has more than one user ID. 

We’ve described the effective user ID (euid),
which defines the access rights for a process. 

A second user ID, the real user ID (ruid),
indicates who initiated a process. 

When you run a setuid program, Linux sets the effective user ID to
the program’s owner during execution, but it keeps your original user ID in the
real user ID.

On modern systems, the difference between the effective and real
user IDs is confusing, so much so that a lot of documentation regarding process
ownership is incorrect.

Think of the effective user ID as the actor and
the real user ID as the owner. 

The real user ID defines the user that can interact with the
running process—most significantly, which user can kill and send signals to a
process. 

For example, if user A starts a new process that runs as user B
(based on setuid permissions), user A still owns the process and can kill it.

On normal Linux systems, most processes have the same effective
user ID and real user ID. By default, ps and other system diagnostic
programs show the effective user ID. 

To view both the effective and real user IDs on your system, try
this, but don’t be surprised if you find that the two user ID columns are
identical for all processes on your system:

$ ps -eo
pid,euser,ruser,comm

To create an exception just so that you can see different values
in the columns, try experimenting by creating a setuid copy of
the sleep command, running the copy for a few seconds, and then
running the preceding ps command in another window before the copy
terminates.

To add to the confusion, in addition to the real and effective
user IDs, there is also a saved user ID (which is
usually not abbreviated). A process can switch its effective user ID to the
real or saved user ID during execution. (To make things even more complicated,
Linux has yet another user ID: the file system user ID [fsuid],
which defines the user accessing the filesystem but is rarely used.)

Typical Setuid Program Behavior

The idea of the real user ID might contradict your previous
experience. 

Why don’t you have to deal with the other user IDs very
frequently? 

For example, after starting a process with sudo, if you want
to kill it, you still use sudo; you can’t kill it as your own regular
user. Shouldn’t your regular user be the real user ID in this case, giving you
the correct permissions?

The cause of this behavior is that sudo and many other
setuid programs explicitly change the effective and real
user IDs with one of the setuid() system calls. 

These programs do so because there are often unintended side
effects and access problems when all of the user IDs do not match.

 

Security Implications

Because the Linux kernel handles all user switches (and as a
result, file access permissions) through setuid programs and subsequent system
calls, systems developers and administrators must be extremely careful with two
things:

The programs that have setuid permissions

What those programs do

Because there are so many ways to break into a system, preventing
intrusion is a multifaceted affair. One of the most essential ways to keep
unwanted activity off your system is to enforce user authentication with
usernames and passwords.

User Identification and Authentication

A multiuser system must provide basic support for user security in
terms of identification and authentication. 

The identification portion of
security answers the question of who users are. 

The authentication piece asks
users to prove that they are who they
say they are. 

Finally, authorization is used to
define and limit what users are allowed to do.

 

Using Libraries for User Information

If every developer who needed to know the current username had to
write all the code you’ve just seen, the system would be a horrifyingly
disjointed, buggy, bloated, and unmaintainable mess. 

Fortunately, we can use standard libraries to perform repetitive
tasks, so all you’d normally need to do to get a username is call a
function like getpwuid() in the standard library after you have the
answer from geteuid(). 

 

PAM

Because there are many kinds of authentication scenarios, PAM
employs a number of dynamically loadable authentication modules. 

Each module performs a specific task; for example, the pam_unix.so module
can check a user’s password.

This is tricky business, to say the least. The programming
interface isn’t easy, and it’s not clear that PAM solves all of the existing
problems. 

Nevertheless, PAM support is in nearly every program that
requires authentication on a Linux system, and most distributions use PAM.
And because it works on top of the existing Unix authentication API,
integrating support into a client requires little, if any, extra work.

 

PAM Configuration

We’ll explore the basics of how PAM works by examining its
configuration. 

You’ll normally find PAM’s application configuration files in
the /etc/pam.d directory
(older systems may use a single /etc/pam.conf file). Most
installations include many files, so you may not know where to start. Some
filenames should correspond to parts of the system that you know already, such
as cron and passwd.

Because the specific configuration in these files varies
significantly between distributions, it can be difficult to find a common
example. We’ll look at an example configuration line that you might find
for chsh (the change shell program):

auth      
requisite     pam_shells.so

This line says that the user’s shell must be in /etc/shells in
order for the user to successfully authenticate with
the chsh program. Let’s see how. Each configuration line has three
fields: a function type, control argument, and module, in that order. Here’s
what they mean for this example:

Function type. The function that a user application asks PAM to perform.
Here, it’s auth, the task of authenticating the user.

Control argument. This setting controls what PAM does after success
or failure of its action for the current line (requisite in this example).
We’ll get to this shortly.

Module. The authentication module that runs for this line,
determining what the line does. Here, the pam_shells.so module checks to
see whether the user’s current shell is listed in /etc/shells.

PAM configuration is detailed on
the pam.conf(5) manual page. Let’s look at a few of the essentials.

Function Types

A user application can ask PAM to
perform one of the following four functions:

auth Authenticate a user (see if
the user is who they say they are).

account Check user account status
(whether the user is authorized to do something, for example).

session Perform something only for
the user’s current session (such as displaying a message of the day).

password Change a user’s password or
other credentials.

 

Control Arguments and Stacked Rules

One important feature of PAM is that the rules specified by its
configuration lines stack, meaning that you can apply
many rules when performing a function. 

This is why the control argument is important: The success or
failure of an action in one line can impact following lines or cause the entire
function to succeed or fail.

There are two kinds of control arguments: the simple syntax and a
more advanced syntax. 

Here are the three major simple syntax control arguments that
you’ll find in a rule:

sufficient If this rule succeeds, the
authentication is successful, and PAM does not need to look at any more rules.
If the rule fails, PAM proceeds to additional rules.

requisite If this rule succeeds, PAM
proceeds to additional rules. If the rule fails, the authentication is
unsuccessful, and PAM does not need to look at any more rules.

required If this rule succeeds, PAM
proceeds to additional rules. If the rule fails, PAM proceeds to additional
rules but will always return an unsuccessful authentication regardless of the
result of the additional rules.

Continuing with the preceding
example, here is an example stack for the chsh authentication
function:

auth      
sufficient      pam_rootok.so

auth      
requisite       pam_shells.so

auth      
sufficient      pam_unix.so

auth      
required        pam_deny.so

Module Arguments

PAM modules can take arguments
after the module name. 

You’ll often encounter this example
with the pam_unix.so module:

auth     
sufficient    pam_unix.so    nullok

The nullok argument here
says that the user can have no password (the default would be fail if the user
has no password).

 

Notes on PAM

Due to its control flow capability and module argument syntax, the
PAM configuration syntax has many features of a programming language and a
certain degree of power. We’ve only scratched the surface so far, but here are
a few more tips on PAM:

To find out which PAM modules are present on your system,
try man -k pam_ (note the underscore). It can be difficult to track
down the location of modules. Try the locate unix_pam.so command and
see where that leads you.

The manual pages contain the functions and arguments for each
module.

Many distributions automatically generate certain PAM
configuration files, so it may not be wise to change them directly in /etc/pam.d.
Read the comments in your /etc/pam.d files before
editing them; if they’re generated files, the comments will tell you where they
came from.

The /etc/pam.d/other configuration
file defines the default configuration for any application that lacks its own
configuration file. The default is often to deny everything.

There are different ways to include additional configuration files
in a PAM configuration file. The @include syntax loads an entire
configuration file, but you can also use a control argument to load only the
configuration for a particular function. The usage varies among distributions.

PAM configuration doesn’t end with module arguments. Some modules
can access additional files in /etc/security, usually to configure
per-user restrictions.

 

PAM and Passwords

Where does PAM get its information about the password encryption
scheme? Recall that there are two ways for PAM to interact with passwords:
the auth function (for verifying a password) and
the password function (for setting a password). It’s easiest to track
down the password-setting parameter. The best way is probably just
to grep it:

$ grep
password.*unix /etc/pam.d/*

The matching lines should
contain pam_unix.so and look something
like this:

password     
sufficient     pam_unix.so obscure sha512



