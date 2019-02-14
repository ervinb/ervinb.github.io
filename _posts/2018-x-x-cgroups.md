The CI work is done by build servers, which are split up with virtual machines. This allows us to serve as many customers, as we have virtual environments spinned up, with a single machine. The most popular case is to have 4 customers served with a single build server.

Now, that the machine is shared, its resources are shared as well. The virtual environments compete for machine resources like CPU, RAM, disk and network. Imagine riding a double seater bicycle with a slower person: you ride faster, so the person behind you will get exhausted much faster than if he'd ride alone.
Jobs which require a lot of resources starve others.

Naturally we strive to have a fair environment for all jobs, while allowing the physical machine to be maximally utilized, plus we give almost bare metal performance. This is where our "faster than other CI services" argument sprouts from.

Current state

There are two major branches in our current platform offering:
1. LXC, used for the standard platform
2. KVM, used for all Docker platforms, including Semaphore 2.0

KVM will be the future, due to it's ease of management, but LXC has an upper hand when it comes to performance. This is due to difference how they're implemented:
 LXC is a containerization technology (from which Docker was born), having direct access to the hosts hardware
 KVM is a virtualization, which completely simulates a computer, so virtual disk, virtual network card, virtual everything.
This "virtualize everything" aspect of KVM is what slows it down. Which is also our saviour when it comes to limiting resources: every piece of the virtualized hardware inside a KVM instance can be adjusted. But this doesn't mean that the resources a shared fairly between multiple instances. Each one will grab whatever it can. Everything is in the hand of the Linux Kernel Scheduler.

LXC instances on the other hand have direct access to the hosts resources, which is behind the superior performance. But given its unrestricted nature, there are no limits on it. Again, in the hands of the Linux Kernel Scheduler.

CGgroups

And this is where CGroups (control groups) come into play. It has the ability to limit resource usage on a process level. So you don't even have to touch the implementation of different platforms. It's also part of Linux out of the box.
LXC ships with it's own cgroups config, and it's a very important component of running containers. We had an attempt in taming LXC but it bit us hard, and we haven't touched it since.

The point is to use CGroups outside of the context of virtual machines and containers, and use it like it's meant to: on a process level. At the end of the day, KVM machines and LXC containers are just processes on the host machine.

Drawing the line

Let's see the technicalities behind CGroups and how we can apply it to our use-case.

CGroups is actually a file system, which is mounted on the host. It creates directories and files for resources (cpu, memory, network), with which they can be controlled.

# subset of CPU controllers
/sys/fs/cgroup/cpu/cpu.cfs_quota_us
/sys/fs/cgroup/cpu/cpu.cfs_period_us
...
Adding values to these files configures the controller and sets the limits. I will get into more details about these and how they limit the CPU resource later.

Processes are connected to rules, by adding their PID to a special file
/sys/fs/cgroup/cpu/tasks
Only one PID can be added at a time.

The very cool thing about the cgroup file system, is that it supports nested groups, and creating such a group is very simple.
$ sudo mkdir -p /sys/fs/cgroup/cpu/semaphore/job/1

The other cool thing is that these newly created directories automatically have all the possible control files for the given resource.
$ ls /sys/fs/cgroup/cpu/semaphore/job/1/
cgroup.clone_children  cgroup.procs  cpu.cfs_period_us  cpu.cfs_quota_us  cpu.shares  cpu.stat  notify_on_release  tasks

So it's simple as creating a new directory, setting the values in the control files, adding a your PID to 'tasks' and you're good to go. But not so fast.

Rule inheritance

What makes cgroups so flexible, compared to other specialized tools like 'cpulimit', is that once you add a PID to the 'tasks' file in your group, all it's children will follow those rules too. Their PIDs will automatically be added to the 'tasks' file.

This is trivial when you want to limit a process you've just started: start the program, fetch its PID, add it to your cgroup and you're good.
But it's a different beast when you want to apply rules to an already existing process tree - started VMs and containers are such, already existing process trees.

Let's see how to do it on an LXC container process.

1. Bootstrap our cgroup for controlling CPU usage

# create cgroup
$ sudo mkdir -p /sys/fs/cgroup/cpu/semaphore/job/1

# the directory is filled automatically
$ ls /sys/fs/cgroup/cpu/semaphore/job/1/
cgroup.clone_children  cgroup.procs  cpu.cfs_period_us  cpu.cfs_quota_us  cpu.shares  cpu.stat  notify_on_release  tasks

# limit cgroup to 10% CPU
$ sudo bash -c 'echo "100000" > /sys/fs/cgroup/cpu/semaphore/job/1/cpu.cfs_quota_us'
$ sudo bash -c 'echo "1000000" > /sys/fs/cgroup/cpu/semaphore/job/1/cpu.cfs_period_us'

2. Examine the process tree on an LXC container.
19677 root       20   0 33392  4024  2844 S  0.0  0.0  0:00.16 │  └─ /sbin/init
 ...
 6252 root       20   0 61396  5512  4848 S  0.0  0.0  0:00.00 │     ├─ /usr/sbin/sshd -D
 8902 root       20   0  103M  6588  5612 S  0.0  0.0  0:00.00 │     │  └─ sshd: runner [priv]
 8913 1002       20   0  103M  3120  2148 S  0.0  0.0  0:00.00 │     │     └─ sshd: runner@pts/2
 8914 1002       20   0 25400  8212  3384 S  0.0  0.0  0:00.02 │     │        └─ -bash
This is the crucial part: cgroup rules are applied to the defined PID, its new direct descendants, and the descendants of those.

If we apply our cgroup to "/sbin/init" (PID#19677), we'd expect that the rules all applied throughout the descendants, but that's not the case. Notice the new word in the previous sentence. Once the rule is applied to a PID, only new children will respect it.

So in this particular case, to apply the rule to "/usr/sbin/sshd -D" (PID#6252), we have to restart the SSH service inside the container, so that it gets a new PID. Once we do that, it will inherit a cgroup applied to its parent "/sbin/init" (PID#19677), and it will get a new PID: 11003.

There's another catch: once the SSH service is restarted and it's given a new PID, its children are untouched - their PID doesn't change (8902, 8913, 8914). To address this, we have to disconnect from the session, and reconnect. After this they will get new PIDs: 12240, 12251, 12265.

Below is a full example of this.

0. Apply the rule to '/sbin/init'
  sudo bash -c 'echo "19677" > /sys/fs/cgroup/cpu/semaphore/job/1/tasks'

1. SSH into the container and restart the SSH service (sshd gets new PID)
  # sshd PID: 6252
  # sshd subproc PIDS: 8902, 8913
  # bash subproc PID: 8914
  # no CPU limit

  $ ssh runner@10.0.3.181 -p29920 -o StrictHostKeyChecking=no
  $ runner@LXC_trusty_1809c-124220b9:~$ sudo service ssh restart

  # sshd PID: 11003
  # sshd subproc PIDS: 8902, 8913
  # bash subproc PID: 8914

  # At this point the rule applies only to PID 11003+, subprocs are unaffected

2. Exit the session (important!) and reconnect. Otherwise the nested bash PID (8914)
will remain the same, and the new rule will not apply.
  $ runner@LXC_trusty_1809c-124220b9:~$ exit
  $ ssh runner@10.0.3.181 -p29920 -o StrictHostKeyChecking=no

  # sshd PID: 11003
  # sshd subproc PIDS: 12240, 12251
  # bash subproc PID: 12265

  # We have new PIDs, the rule now applies to them

3. Check CPU usage
  $ runner@LXC_trusty_1809c-124220b9:~$ yes > /dev/null

4. Clean up
  $ sudo lxc-destroy -n cont -f
  $ sudo rmdir /sys/fs/cgroup/cpu/semaphore/job/1

Now that this is out of the way, let's see how to limit resources in Semaphore jobs and some proposals.

Limiting resources for Semaphore jobs

The exhausting example above actually shows the technicalities of cgroup applying rules to Semaphore jobs, and here we will see the reasoning behind it.

When a job arrives to the build server, it already has a virtual instance waiting for it -- an existing process tree. The software which runs the jobs (job_runner), connects to these waiting instances over SSH.
So to limit resources which are available for the job, we have to apply our cgroup to the master sshd process, and reconnect to it. After this, all commands started inside the job will have to obey our cgroup.

This is quite flexible, so rules could be applied lower or higher on the process tree, but the most resource hungry things by far are the user commands running in jobs, and the above approach will limit them.

Setting CPU limits

CPU is by far the most exhausted hardware resource on our platform, so levelling the field would be very beneficial to our users.

# soft limit
cpu.shares

# hard limits
cpu.cfs_quota_us
cpu.cfs_period_us

Soft limit means that the process is free to use the resource as much as it'd like to, until there's a race for it. So as soon as CPU resources become scarce, the limit will kick in.
This will assure that other processes are not starved.
By default there are 1024 "cpu.shares", which incorporates all cores at once - a CPU.

Hard limits are active right from the start.
"cpu.cfs_period_us" describes the base unit, during which the controlling is enforced. So to do this on a per second basis, use 1M usec.
"cpu.cfs_quota_us" tells it how much of that time can be used by the process. So if we want to limit the process to have access to only 10% of the performance, we'd set 1M * 0.1 = 100k.

Some real world examples:

1. Standard platform: 1 build server, allows 4 concurrent jobs to be executed
- calculate the soft limit, which will allow the job to maximally take advantage of the machine when it's not under load; this allows fluctuating performance and prevents starvation.

1024 [max CPU shares] / 4 [max jobs] = 256 shares per job

$ sudo bash -c 'echo "256" > /sys/fs/cgroup/cpu/semaphore/job/1/cpu.shares'

2. Least powerful platform for S2: hard limit CPU usage to 0.5 cores per job
- calculate how many cores can be delegated per job

48 [vCores] / 96 [max jobs] = 0.5 vCores per job

- set hard limit 

To calculate quota for this particular case, we need to divide the base period with the number of jobs, so each job has an equal time-share.

1000000 [cpu.cfs_period_us] / 96 [max jobs] = 10416

$ sudo bash -c 'echo "10416" > /sys/fs/cgroup/cpu/semaphore/job/1/cpu.cfs_quota_us'
$ sudo bash -c 'echo "1000000" > /sys/fs/cgroup/cpu/semaphore/job/1/cpu.cfs_period_us'

More info on: http://man7.org/linux/man-pages/man7/cgroups.7.html