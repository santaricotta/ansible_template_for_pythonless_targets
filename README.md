# introduction

This repository contains 2 playbook templates for you to copy and alter, with each containing multiple task examples. 
* `playbook_template_raw.yml` - Use this playbook as a template for the simplest of bash commands that you want to run on a group of hosts. 
    * It only needs small adjustments explained below (and in the playbook file). 
    * This playbook does not require Python on the target host. 
* `playbook_template.yml` - Use this playbook as a template for more complex operations that require Ansible's native modules.
    * This playbook brings its own Python to the target host and deletes it at the end of its execution.
* everything in this README marked with (WIP) is a work in progress and may not even exist yet. 

## prerequisites

### ansible installation and preparation

* You only need to do this once.
* This will create a virtual Python environment, install Ansible into it (without touching your system Python packages)

```
apt install python3-venv build-essential
mkdir -p ~/bin/venvs
cd ~/bin/venvs
python3 -m venv ansible
source ansible/bin/activate
# Your prompt has changed to indicate that you are now in a virtual Python environment.
python3 -m pip install --upgrade pip
python3 -m pip install ansible boto3 botocore
ansible --version
```

# files, directories and their purpose

## ansible.cfg

* Do not adjust this file, unless you know what you are doing. Its settings disable key checking and enable retrying on failed hosts.

## inventory.aws_ec2.yml

This is a dynamic inventory file for use with AWS' EC2 instances. In this file, you may need to adjust:

* the value of `regions`. It can be a list of multiple regions, not just `eu-west-1`
* the value of `filters`. In my case: `tag:environment:` This limits the scope of influence of this playbook. 

If you want to target a specific single EC2 instance, you can replace the tag filter with one by `instance-id` or even `name` (outside of this howto's scope). 

## playbook templates

You only need to use one of these. The non-raw template takes 5 extra seconds to install Python before and remove Python after the main tasks execution. 

### playbook_template_raw.yml

* Make a copy of this file and rename it to YOUR_PLAYBOOK_NAME_raw.yml (This is just an example name, do not actually rename it to that.)
* In the file, change:
    * `remote_user:` value to your username
    * `name:`, `raw:` and `changed_when:`
        * If you don't know how to adjust `changed_when:`, just comment out both the `register:` and `changed_when:` lines

### playbook_template.yml

* Make a copy of this file or rename it to YOUR_PLAYBOOK_NAME.yml (This is just an example name, do not actually rename it to that.)
* This file is a true playbook (a book of multiple plays). It contains 3 separate plays, each containing its own set of tasks:
    * The first play **Set up Python on pythonless host** only sets up Python. Do not touch this part. 
    * The second play **Ansible tasks to perform on hosts** is where you should add your tasks. You only need to alter the `tasks:` section. 
    * The third play **Remove Python** undoes what the first play did. 
* In production, replace the `python_url:` value with a copy hosted on your own web server or S3 bucket or face Github's "403 rate limit exceeded".
    * You can also comment out the 3rd play of this playbook and leave Python on the system for future use. 

## examples (directory) (WIP)

* Examples of playbooks with specific tasks to help you compose your own.
* Examples of inventory files with different filters and cloud providers. 

# how to run this

* Adjust `inventory.aws_ec2.yml` file according to instructions above
* Prepare `YOUR_PLAYBOOK_NAME.yml` file according to instructions above 
* Activate your Python virtual environment where ansible is installed: `source ~/bin/venvs/ansible/bin/activate`. Your shell prompt changes. 
* Set up your `default` credentials in `~/.aws/credentials`. The inventory file expects the profile to be called `default`. 
* Check which hosts would be touched by this playbook: `ansible-inventory -i inventory.aws_ec2.yml --list`
* (optional) Do a verbose dry run (tasks will only be simulated on the target host and therefore many will fail if they expect to read their result): `ansible-playbook -i inventory.aws_ec2.yml YOUR_PLAYBOOK_NAME.yml -vvC`
* Do a real run but with **step by step confirmations** (cautious mode): `ansible-playbook -i inventory.aws_ec2.yml YOUR_PLAYBOOK_NAME.yml --step`
* If successful in `development` environment, change the environment in the inventory file to `stress` and do a normal run without `--step`: `ansible-playbook -i inventory.aws_ec2.yml YOUR_PLAYBOOK_NAME.yml`
    * To speed things up, you can change the default concurrency from running on 10 machines at a time to running on 40 by appending `-f 40`
* For various reasons, some target machines may fail to connect or perform the tasks. To retry only on those failed hosts: `ansible-playbook -i inventory.aws_ec2.yml YOUR_PLAYBOOK_NAME.yml --limit @/home/username/path/to/your/current/directory/YOUR_PLAYBOOK_NAME.retry`
* If it still fails on some, investigate these leftover hosts in the `YOUR_PLAYBOOK_NAME.retry` file and delete the file afterwards. 
* Deactivate your Python virtual environment: `deactivate` - your shell prompt will return to normal. 

# how to run this in an emergency situation that requires quick action

This is a shortened process with speed optimisations. Only use where emergency requires it. Not recommended otherwise! Speed optimisations in **bold**.

* Adjust `inventory.aws_ec2.yml` file according to instructions above AND **remove the whole `filters:` section**. This will ensure the playbook runs on everything, regardless of `environment` tag value. 
* Prepare `YOUR_PLAYBOOK_NAME.yml` file according to instructions above AND **under all occurrences of `hosts: all`, add `strategy: free` at the same indentation level**. This ensures tasks run without waiting for other hosts to complete the same tasks. (It is therefore incompatible with the `--step` parameter we use in non-emergency situations.)
* Activate your Python virtual environment where ansible is installed: `source ~/bin/venvs/ansible/bin/activate`. Your shell prompt changes. 
* Set up your `default` credentials in `~/.aws/credentials`. The inventory file expects the profile to be called `default`. 
* Run your playbook with "ansible-playbook -i inventory.aws_ec2.yml YOUR_PLAYBOOK_NAME.yml -f 100". **The `-f 100` spawns 100 forks and runs this on 100 target hosts concurrently instead of the default 10.** If your playbook is long and complex, you may not have enough RAM for that -> use `-f 40` instead. 
* For various reasons, some target machines may fail to connect or perform the tasks. To retry only on those failed hosts: `ansible-playbook -i inventory.aws_ec2.yml YOUR_PLAYBOOK_NAME.yml --limit @/home/username/path/to/your/current/directory/YOUR_PLAYBOOK_NAME.retry`
* If it still fails on some, investigate these leftover hosts in the `YOUR_PLAYBOOK_NAME.retry` file and delete the file afterwards. 
* Deactivate your Python virtual environment: `deactivate` - your shell prompt will return to normal. 

