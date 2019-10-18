Ubuntu Server 18.04.3 LTS

Instructions

1. Update server IP in inventory
2. RUN: ansible-playbook -i inventory init.yml
3. RUN: ansible-playbook -i inventory -e ansible_port=PORT --ask-become-pass server_setup.yml

mkdir -p group_vars/all && touch $_/vars $_/vault

ansible-vault view --vault-id dev@~/.vault-pass hello.yml

```
# group_vars/all
---
sudo_user: USER
ssh_port: PORT
public_key_path: PATH_TO_PUB_KEY
```

```
# group_vars/vault
---
root_password: ROOT_PASS
sudo_password: SUDO_PASS
```
```
ansible-vault encrypt group_vars/all/vault
```
Run your playbook with the public and encrypted inventory vars
```
ansible-playbook -i inventory -e ansible_port=22 -u root site.yml --ask-vault-pass
ansible-playbook -i inventory -e ansible_port=PORT -u USER site.yml --ask-vault-pass --ask-become-pass
```

# Getting started with ansible

This is a basic tutorial for initial server configuration. We'll be using Ansible to automate managing users and setting some sensible defaults for SSH.

### playbook

Playbook takes its name from a football playbook, where a professional football team keeps schematics of their plays. Each individual play is a drawing or schematic where each individual player starts and where they should end up after executing the play. A collection of plays make a playbook, and a collection of tasks make a play. A task is a little instruction that performs something on the host or hosts - oftentimes, we use [Ansible modules](https://docs.ansible.com/ansible/latest/user_guide/modules.html) to perform tasks. A role is a task and a task is a role, they're basically interchangeable for the most part so don't get confused with the terminology.

##### YAML

Playbooks are defined in [YAML](https://docs.ansible.com/ansible/latest/reference_appendices/YAMLSyntax.html). It is very easy to learn, but you have to stick to the syntax. Follow along in the examples and you should pick it up right away or click on the link provided for a more in-depth primer.

##### First playbook

So for starters, in order to begin drafting your 'play', the very first step is to define which remote machines we want to execute our play on, because we need to have a target for our actions. This is defined as the first parameter in our YAML playbook file containing our play.
```
---
- hosts: webservers
  remote_user: YOUR_USER
  become: yes
  become_user: postgres
```
Here our target is 'webservers', which is just an alias representing one or more hosts. Just below that we've included the remote_user, which is the user that we'd like Ansible to SSH to the host as. After connection is established we can immediately tell Ansible to escalate privileges on our target by providing `become: yes` and even tell it who to 'become' by providing, for example: `become_user: postgres`. There are many ways to escalate privilege, for the the whole play, for specific tasks in the play, and Ansible gives you this flexibility. For more on [privilege escalation](https://docs.ansible.com/ansible/latest/user_guide/become.html) click on the link.

***Note:*** if there is privilege escalation in your playbook and your remote server requires a password for sudo, then you can provide it at the start of running your playbook from the command line by adding the option `--ask-become-pass`. The first thing Ansible will do, before running your playbook, is ask for your sudo password to be used throughout your playbook where privilege escalation is required.

So now we've tackled defining the target of our play (the remote machines) inside our playbook. We've gone over the basics of defining which user Ansible will connect to our target as, using `remote_user`. Furthermore, we've gone over the basics of privilege escalation in the target. Now for the fun part... executing actual tasks.

##### tasks

Tasks are instructions that we'd like executed on our target (host). A task is essentially just a module that gets executed on the remote machines with the arguments we provide. Ansible provides many modules, and you can even develop your own modules. For a quick reference of what is available checkout the [module directory](https://docs.ansible.com/ansible/latest/modules/modules_by_category.html).

Modules should be idempotent, which is just a fancy word to say, "running our module once should produce the same result as running it additional times on the host." This makes it safe to re-run your module multiple times (which will happen) because there is no danger that the second time you run the same task it will produce different and unexpected results, leading to inconsistency in our system. We're always trying to achieve the same consistent result, whether we're running our play once, twice, or several times.

If each task achieves a result, than a good way before executing a module is to check, "has the result of this module been achieved?" If yes, then continue because there is nothing to be done, if not, then execute the module and give me the desired result.

```
# Example task in our play inside our playbook

tasks:
  - name: Change bob password
    user:
      name: bob
      password: "{{ password | password_hash('sha512') }}"
```
The above task has a name 'Change root password'. The name can be anything you like and just describes what the task is doing. Following we're using the `user` module and nested inside that are two key value pairs, used as arguments: the name of the user and the password. The `user` module will make changes to user accounts in our target machines. This task will change the password for this particular user. Easy to understand right?! That's because Ansible uses YAML and not only is it easy to write, but also easy to read and follow along on what is happening.

You may wonder, what is that inside the quotes on the password argument. We're using variables now and injecting them inside at runtime. Variables will be covered shortly, but essentially we're getting the password variable and immediately hashing it to protect it.

Tasks can be inlined right there in our play inside our playbook. For a simple play this is fine, but you'll want to extract out tasks into roles. We'll go over that as well.

### Handlers

There are certain tasks we want executed on our target that naturally follow other tasks. Like, for example, restarting Apache server. We don't want to restart Apache server just for kicks, so this task by itself is meaningless. We should only restart Apache server *after* we've performed a task that modifies an Apache configuration of some kind. At the same time, we also don't want to restart Apache, if we've executed a task to change a configuration, but because the desired configuration is already existant in our system, no change is actually made. 

So what we really want is a *handler*, that gets executed after a task *if* it has made a change to the system. Ansible has a way to do that.

```
handlers:
  - name: restart apache
    service:
      name: apache
      state: restarted
```
So far we have only defined a handler.

Ansible has a way of noticing when a task has produced a change in our target system, and if this is the case, we can notify our handlers, which are follow up tasks that need to be executed because of the change. Take the example above, we've defined a handler named 'restart apache', that restarts our Apache process using the [service](https://docs.ansible.com/ansible/latest/modules/service_module.html) module. By itself, it does nothing. But under tasks we can list handlers within a `notify` block which is basically like registering a callback to be executed *in the event the task has produced a change in our system*.
```
tasks:
  - name: update website apache conf file
    template:
      src: my_site.conf.j2
      dest: /etc/apache2/sites-available/my_site.conf
    notify:
      - restart apache
```
The above is using the `template` module which you can read more about [here](https://docs.ansible.com/ansible/latest/modules/template_module.html). It is taking a template and moving it over to the `src` in our target machines. If the file isn't already there, or else the file is different than our template, copy our template over to that destination. If we run this task and it produces a change (the file didn't exist yet or was different than our template), Ansible will "notify" our handlers and they will be executed.

***IMPORTANT CAVEAT:*** You may have several tasks "notifying" the same handler within a play. It does not matter how many actually "notify", the handler will only be executed a single time, after the completion of the block of tasks inside the play.

For more on handlers click [here](https://docs.ansible.com/ansible/latest/user_guide/playbooks_intro.html#handlers-running-operations-on-change)

[Ansible-Pull](https://docs.ansible.com/ansible/latest/user_guide/playbooks_intro.html#handlers-running-operations-on-change)
> How cool would it be, instead of sending out instructions you'd have your targets check-in to a specific respository for instructions.

[Ansible-Link](https://docs.ansible.com/ansible/latest/user_guide/playbooks_intro.html#linting-playbooks)
> You can use ansible-lint to run a detail check of your playbooks before you execute them.

### inventory

The inventory is the source of truth and Ansible is an automation tool. We specify in our inventory the variables for where we want our playbook to run, and how we want it to run.

For the initial inventory specifying our server IP addresses I'll use INI format
```
# Create an INI file called 'inventory'

[webservers]
webserver1_IP

[dbservers]
dbserver1_IP

[infra:children]
webservers
dbservers
```
Here we specified the WHAT. What will we be interacting with? We've created categories. We have webservers category and db servers category + a group called infra which is a grouping of groups. So it includes both webservers and db servers.

If you have variables that apply to a particular server or group you can include them right there inline.
```
[webservers]
webserver1_IP ansible_user=joe
```
Or you can apply a group of vars to the entire group
```
[webservers]
webserver1_IP

[webservers:vars]
ansible_user=joe
another_var=another_value
```

How to run an ansible playbook?
```
ansible-playbook -i inventory -k your_playbook.yml
```
Easy as that. The -k flag is telling Ansible you'd like to specify your SSH password at the prompt to initiate the connection.

### roles

Roles are groupings of tasks, handlers, variables, templates and files, that are related in accomplishing a particular goal. It is a solution to keeping all your tasks inside of a play in a large file playbook. If you needed to reuse those same tasks to accomplish the same goal elsewhere, well now you can resuse the role. The role, therefore, is a means of keeping things organized while at the same time allowing for you to reuse those tasks elsewhere in the same playbook or in other playbooks.

Roles follow a strict directory structure. Ansible will look for folder named after the role you are trying to use inside the `roles` directory. Inside each role you can have directories: tasks, handlers, files, templates, vars, defaults, meta, each containing a main.yml file that has the relevant data.

To use a role in your play you simply include it under the roles option:
```
---
- hosts: webservers
  roles:
    - common
    - user_management
    - firewall_config
```
Including roles in this way will make any tasks provided in those roles be added to your play. Handlers, vars, and defaults will also be added to the play. Any file or template included will also be available in your play.

I hope you can the usefulness of extracting out certain behavior grouped around a broader task so you can not only cleanup your play, but also purposefully reuse the role in other plays or playbooks.

### A few notes on playbook execution

There is a logical order for execution of tasks in your play. And here it is:

- pre_tasks
- any handlers that have been notified
- each role will execute in turn
- any tasks defined in the play
- any handlers that have been notified (again)
- post_tasks
- any handlers that have been notified (again)

### Security and Sensitive Information

We can see the clear value here for DevOps. We create a playbook, simply a recipe of commands, that we'd like executed on the host or hosts that will modify their state to where we'd like for them to be. That may mean, creating certain SUDO users, enabling a firewall, setting up Fail2Ban, preventing Root Login, etc. These are all repetitive steps we normally undertake on a fresh server. Let's save time and automate this process.

We can include variables in our inventory and other files that tweak our commands. But what about sensitive information like database passwords, root login passwords, privilege-escalation, etc. `ansible-vault` to the rescue. It's a handy utility for encrypting and decrypting files and strings.

Ansible 2.4 released support for *vault-ids*. Vault ids allow for you to encrypt several files with unique passwords. Prior to this release, each ansible run supported only a single password, therefore limiting you to encrypting all your files with the *same* password. Now you can reference several several vault ids when you run your playbooks.

Each password is stored inside a plain text file. You can reference the text file (password) to ansible-vault create, view, and edit a file that was encrypted using that password. In order to help you identify which password you used to encrypt the file you can use a label.

Ex:
```
ansible-vault create --vault-id LABEL@prompt encrypted_file.yml
```
When you go on to specify the identity file to decrypt an encrypted file, the label is a helpful hint and identify files with the same label as the encrypted file will be tried first. If this fails, then additional identity files will be tried in order they were entered in the command line, until the file is decrypted or it fails.

> The password with the same label as the encrypted data will be tried first, after that each vault secret will be tried in the order they were provided on the command line.
> Where the encrypted data doesn’t have a label, or the label doesn’t match any of the provided labels, the passwords will be tried in the order they are specified.

Example encrypting a single string
```
ansible-vault encrypt_string --vault-id LABEL@prompt 'secretpass' --name 'ansible_password'
```
It will print out the key (ansible_password) and the encrypted password (secretpass) in a nice format so we can copy paste it into our YAML file. In order to do the encryption `ansible-vault` will ask for a password. You can have it prompt you for a password, or you can save a password in your home directory in a plain text file and instead of your_user@prompt you'd type your_user@<path to password file>.

An INI inventory doesn't support this. But we can place this value inside a 'vars' file inside a directory 'host_vars' or 'group_vars', both are the standard in Ansible and vars files within those directories will be automatically picked up.

Now to execute our playbook and decrypt the encrypted information, we'll always need to provide the password we used for encryption. Like I said before, the password can live in a text file in your home directory and you can reference it, or you can simply request that Ansible prompt you for the password when executing the playbook. Both methods are shown below.
```
ansible-playbook -i inventory --vault-id my_user@prompt first_playbook.yml
ansible-playbook -i inventory --vault-id my_user@~/my-ansible-vault-pw-file first_playbook.yml
```
> Vault content can only be decrypted with the password that was used to encrypt it. If you want to stop using one password and move to a new one, you can update and re-encrypt existing vault content with ansible-vault rekey myfile, then provide the old password and the new password. Copies of vault content still encrypted with the old password can still be decrypted with old password.

I like to keep the main inventory file for organizing and grouping my hosts in INI format. It just helps me to visualize and quickly see what hosts pertain to what groups in a flattened out way (without the nesting of Yaml). The variables though, are much easier to manage and visualize in YAML inside `group_vars` and `host_vars` directories.

Valid file extensions are ‘.yml’, ‘.yaml’, ‘.json’, or no file extension, and the actual filename inside group_vars and host_vars would take the group_name or host_name, respectively. You could also use directories named after your group_name within YAML files inside. These files would all be loaded for that group, as well, and I find for me this to be the approach that makes the most sense.

Here is an example:
```
# ./inventory

[webservers]
1.1.1.1
2.2.2.2.


# ./group_vars/webservers/vars

ansible_user: YOUR_USER
ansible_port: YOUR_PORT
ansible_password: YOUR_PASSWORD
```

### Variables are merged and "flattened" when you run your playbook, and naturally there is an order of precedence for how this is done, since you may have variables defined at the global level, the group level, the sub group level, or the host level. The question is: how does Ansible flatten and merge variables?

That is easy to answer. The order of precendence always starts with the widest scope to the narrowest. Vars defined at the host level are overwritten by the same variable (if they exist) at the group_level and the same happens again if we have define the same variable at the host level.

I lied. The above makes sense and generally works, but there are many places in Ansible where you can define a variable. Not always is the precedence clear, therefore, if you ever get tripped up, go to this [link](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#ansible-variable-precedence), which breaks down the many places where variables can be defined and how to they overwritten.

Here are some standard Ansible variables that define the connection when you run your playbook.

- ansible_host: either an alias or an IP
- ansible_port: (default 22) or whatever you picked
- ansible_user: which user to connect as
- ansible_password: the password to use to authenticate to the host (if you're not using ssh key)

There are a number of other variables, but these are the principal ones. Consult the documentation for more.

For privilege escalation variables:

- ansible_become: forces privilege escalation
- ansible_become_method: defines the privilege escalation method (sudo, su)
- ansible_become_user: defines the user you become through privilege escalation
- ansible_become_password: defines the password that the host will require to permit the privilege escalation (sudo password)

One that might trip you up is not setting the `ansible_python_interpreter` variable. You need to set this for systems that use multiple versions of Python or where the Python path may not be in the obvious /usr/bin/python path. Since we're on Ubuntu, and I know the latest version is using Python3, I know the path is /usr/bin/python3 and will include this as a global variable.

