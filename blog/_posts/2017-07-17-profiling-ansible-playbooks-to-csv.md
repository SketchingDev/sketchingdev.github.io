---
layout: post
title:  "Profiling Ansible playbooks"
date:   2017-07-17 00:00:00
categories: ansible
---

At work we have been working on extending our [Ansible](https://www.ansible.com/) playbooks to deploy a Hadoop cluster to on-premise hardware. Although not a problem on AWS, when deployed on-premise we've faced network latency issues.
To aid in investigating the affect of the latency on our playbooks I used the following process to profile time each task took to complete and then transformed the output into a CSV format for easy interrogation.

## Instructions

As per the Ansible profile plugin’s [usage instructions](https://docs.ansible.com/ansible/devel/plugins/callback/profile_tasks.html) enable task profiling in Ansible 2.0+ by adding `profile_tasks` to the `callback_whitelist` in the `ansible.cfg` file.

```
[defaults]
callback_whitelist = profile_tasks
```

You’ll then want to configure the plugin using the following environment variables. These will ensure the output includes all the tasks and in the same order:

```bash
export PROFILE_TASKS_SORT_ORDER=none
export PROFILE_TASKS_TASK_OUTPUT_LIMIT=all
```

With the plugin now enabled and configured running the playbook will result in a summary like the one below being appended to the end of the output:

```
Friday 14 July 12:46:03 +0000 (0:00:00.832)
===============================================================================
Wait for instances ---------------------------------------------------- 317.76s
Wait for network interfaces -------------------------------------------- 60.04s
provision instances ----------------------------------------------------- 9.01s
provision cluster security group ---------------------------------------- 3.36s
provision gateway security group ---------------------------------------- 2.48s
provision edgenode security group --------------------------------------- 1.79s
provision edgenode security group --------------------------------------- 0.87s
```

The summary that's now output at the end of each run can be converted into a comma-separated values (of task name and duration) with the command:

```bash
$ ansible-playbook site.yml -i hosts | egrep '(.+) -{3,} {.+}' | awk -F ' -{3,} ' '{print $1 "," $2}'
```

Now that the output is in CSV format you can pipe it to a file and interrogate it with your favourite program e.g. using [q](http://harelba.github.io/q/) to query it with SQL-like statements.

```
wait for instances,317.76s
wait for network interfaces,60.04s
provision instances,9.01s
provision cluster security group,3.36s
provision gateway security group,2.48s
provision edgenode security group,1.79s
provision edgenode security group,0.87s
```

## Using the ‘free’ strategy

If you use the `free` strategy and intend on comparing the CSV values across multiple executions of the same playbooks you might want to use the `sort` utility to ensure the task order of the outputs are consistent.

```bash
$ ansible-playbook site.yml -i hosts | egrep '(.+) -{3,} {.+}' | awk -F ' -{3,} ' '{print $1 "," $2}' | sort -t, -k1
```
