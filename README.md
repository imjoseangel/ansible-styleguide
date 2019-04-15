# Ansible Styleguide

## Table of Contents

  1. [Practices](#practices)
  2. [Start of Files](#start-of-files)
  3. [End of Files](#end-of-files)
  4. [Quotes](#quotes)
  5. [Environment](#environment)
  6. [Booleans](#booleans)
  7. [Key value pairs](#key-value-pairs)
  8. [Sudo](#sudo)
  9. [Hosts Declaration](#hosts-declaration)
  10. [Task Declaration](#task-declaration)
  11. [Include Declaration](#include-declaration)
  12. [Spacing](#spacing)
  13. [Variable Names](#variable-names)
  14. [Jinja Variables](#jinja-variables)
  15. [File Extension](#file-extension)
  16. [Vaults](#Vaults)
  17. [Role Names](#Role-Names)

## Practices

You should follow the [Best Practices](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html) defined by the Ansible documentation when developing playbooks.

### Why?

The Ansible developers have a good understanding of how the playbooks work and where they look for certain files. Following these practices will avoid a lot of problems.

### Why Doesn't Your Style Follow Theirs?

The playbook examples are inconsistent in style throughout the Ansible documentation; the purpose of this document is to define a consistent style that can be used throughout Ansible playbooks to create robust, readable code.

## Start of Files

You should start your playbooks with some comments explaining what the playbook's purpose does (and an example usage, if necessary), followed by `---` with no blank lines around it, then followed by the rest of the playbook.

```yaml
#bad
- name: Change s1m0n3's status
  service:
    enabled: true
    name: "s1m0ne"
    state: "{{ state }}"
  become: true

#good
# Example usage: ansible-playbook -e state=started playbook.yml
# This playbook changes the state of s1m0n3 the robot
---
- name: Change s1m0n3's status
  service:
    enabled: true
    name: "s1m0ne"
    state: "{{ state }}"
  become: true
```

### Why?

This makes it easier to quickly find out the purpose/usage of a playbook, either by opening the file or using the `head` command.

## End of Files

You should always end your files with a newline.

### Why?

This is common Unix best practice, and avoids any prompt misalignment when printing files in a terminal.

## Quotes

**We always quote strings** and prefer double quotes over single quotes. The only time you should use single quotes is when they are nested within double quotes (e.g. Jinja map reference), or when your string requires escaping characters (e.g. using "\n" to represent a newline). If you must write a huge string, we can the "folded scalar" style and omit all special quoting. The only things you should avoid quoting are booleans (e.g. true/false), numbers (e.g. 42), and things referencing the local Ansible environemnt (e.g. boolean logic or names of variables we are assigning values to).

Do NOT quote:

* *hosts*: targets (e.g. hosts: databases rather than hosts: ‘databases’)
* *include_tasks*: and *include_roles*: target file names
* task and role names
* registered variables
* number values
* boolean values
* conditional logic (when: task options)

```yaml
# bad
- name: "start robot named S1m0ne"
  service:
    name: s1m0ne
    state: started
    enabled: true
  become: true

# good
- name: start robot named S1m0ne
  service:
    name: "s1m0ne"
    state: "started"
    enabled: true
  become: true

# single quotes w/ nested double quotes
- name: start all robots
  service:
    name: "{{ item['robot_name'] }}"
    state: "started"
    enabled: true
  with_items: "{{ robots }}"
  become: true

# double quotes to escape characters
- name: print some text on two lines
  debug:
    msg: "This text is on\ntwo lines"

# folded scalar style
- name: robot infos
  debug:
    msg: Robot {{ item[robot_name] }} is {{ item[status] }} and in {{ item[az] }}
  with_items: robots

# folded scalar when the string has nested quotes already
- name: print some text
  debug:
    msg: “I haven’t the slightest idea,” said the Hatter.

# don't quote booleans/numbers
- name: download google homepage
  get_url:
    dest: "/tmp"
    timeout: 60
    url: "https://google.com"
    validate_certs: true

# variables example 1
- name: set a variable
  set_fact:
    my_var: "test"

# variables example 2
- name: print my_var
  debug:
    var: my_var
  when: ansible_os_family == "Darwin"

# variables example 3
- name: set another variable
  set_fact:
    my_second_var: "{{ my_var }}"
```

### Why?

Even though strings are the default type for YAML, syntax highlighting looks better when explicitly set types. This also helps troubleshoot malformed strings when they should be properly escaped to have the desired effect.

## Booleans

```yaml
# bad
- name: start sensu-client
  service:
    name: "sensu-client"
    state: "restarted"
    enabled: 1
  become: "yes"

# good
- name: start sensu-client
  service:
    name: "sensu-client"
    state: "restarted"
    enabled: true
  become: true
```

### Why?

There are many different ways to specify a boolean value in ansible, `True/False`, `true/false`, `yes/no`, `1/0`. While it is cute to see all those options we prefer to stick to one : `true/false`. The main reasoning behind this is that Java and JavaScript have similar designations for boolean values.

## Key value pairs

Use only one space after the colon when designating a key value pair

```yaml
# bad
- name : start sensu-client
  service:
    name    : "sensu-client"
    state   : "restarted"
    enabled : true
  become : true


# good
- name: start sensu-client
  service:
    name: "sensu-client"
    state: "restarted"
    enabled: true
  become: true
```

**Always use the map syntax,** regardless of how many pairs exist in the map.

```yaml
# bad
- name: create checks directory to make it easier to look at checks vs handlers
  file: "path=/etc/sensu/conf.d/checks state=directory mode=0755 owner=sensu group=sensu"
  become: true

- name: copy check-memory.json to /etc/sensu/conf.d
  copy: "dest=/etc/sensu/conf.d/checks/ src=checks/check-memory.json"
  become: true

# good
- name: create checks directory to make it easier to look at checks vs handlers
  file:
    group: "sensu"
    mode: "0755"
    owner: "sensu"
    path: "/etc/sensu/conf.d/checks"
    state: "directory"
  become: true

- name: copy check-memory.json to /etc/sensu/conf.d
  copy:
    dest: "/etc/sensu/conf.d/checks/"
    src: "checks/check-memory.json"
  become: true
```

Use the map syntax **for roles** too.

```yaml
# bad
  roles:
    - { role: "tomcat", tags: "tomcat" }

# good
  roles:
    role: tomcat
    tags: "tomcat"

    role: webapp
    code: "mycode"
```

### Why?

It's easier to read and it's not hard to do. It reduces changeset collisions for version control.

## Sudo

Use the new `become` syntax when designating that a task needs to be run with `sudo` privileges

```yaml
#bad
- name: template client.json to /etc/sensu/conf.d/
  template:
    dest: "/etc/sensu/conf.d/client.json"
    src: "client.json.j2"
  sudo: true

# good
- name: template client.json to /etc/sensu/conf.d/
  template:
    dest: "/etc/sensu/conf.d/client.json"
    src: "client.json.j2"
  become: true
```

### Why?

Using `sudo` was deprecated at [Ansible version 1.9.1](https://docs.ansible.com/ansible/latest/user_guide/become.html)

## Hosts Declaration

`host` sections should follow this general order:

```yaml
# host declaration
# host options in alphabetical order
# pre_tasks
# roles
# tasks

# example
- hosts: webservers
  remote_user: "centos"
  vars:
    tomcat_state: "started"
  pre_tasks:
    - name: set the timezone to America/Boise
      lineinfile:
        dest: "/etc/environment"
        line: "TZ=America/Boise"
        state: "present"
      become: true
  roles:
    - role: tomcat
      tags: "tomcat"
  tasks:
    - name: start the tomcat service
      service:
        name: "tomcat"
        state: "{{ tomcat_state }}"
```

### Why?

A proper definition for how to order these maps produces consistent and easily readable code.

## Task Declaration

A task should be defined in such a way that it follows this general order:

```yaml
# task name
# taks vars
# task map declaration (e.g. service:)
# task parameters in alphabetical order (remember to always use multi-line map syntax)
# loop operators (e.g. with_items)
# task options in alphabetical order (e.g. become, ignore_errors, register)

# example
- name: create some azure instances
  vars:
    instance_names:
      - "az1"
      - "az2"
  azure_rm_virtualmachine:
    admin_username: "azureuser"
    name: "{{ item }}"
    network_interfaces: "my_network"
    ssh_password_enabled: false
    vm_size: "Standard_DS1_v2"
  with_items: "{{ instance_names }}"
  ignore_errors: true
  register: ec2_output
  when: ansible_os_family == "Darwin"
```

### Why?

Similar to the hosts definition, having a well-defined style here helps us create consistent code.

## Include Declaration

For `include` statements, make sure to NOT quote filenames and only use blank lines between `include` statements if they are multi-line (e.g. they have vars).

```yaml
# bad
- include: "other_file.yml"

- include: second_file.yml

- include: third_file.yml tags=third

# good

- include: other_file.yml
- include: second_file.yml

- include: third_file.yml
  vars:
    myvar: "third"
```

### Why?

This tends to be the most readable way to have `include` statements in your code.

## Spacing

You should have blank lines between two host blocks, between two task blocks, and between host and include blocks. When indenting, you should use 2 spaces to represent sub-maps, and multi-line maps should start with a `-`). For a more in-depth example of how spacing (and other things) should look, consult [style.yml](style.yml).

### Why?

This produces nice looking code that is easy to read.

## Variable Names

Use `snake_case` for variable names in your playbooks.

```yaml
# bad
- name: set some facts
  set_fact:
    myBoolean: true
    myint: 20
    MY_STRING: "test"

# good
- name: set some facts
  set_fact:
    my_boolean: true
    my_int: 20
    my_string: "test"
```

### Why?

Ansible uses `snake_case` for module names so it makes sense to extend this convention to variable names.

## Jinja Variables

Use spaces around jinja variable names.

```yaml
# bad
- name: set some facts
  set_fact:
    myBoolean: "{{myoldvar}}"

# good
- name: set some facts
  set_fact:
    myBoolean: "{{ myoldvar }}"
```

### Why?

A proper definition for how to create Jinja variables produces consistent and easily readable code.

## File Extension

All Ansible Yaml files MUST have a `.yml` extension (and NOT .YML, .yaml etc).

```sh
# bad
~/tasks.yaml

# good
~/tasks.yml
```

### Why?

Ansible tooling (like ansible-galaxy init) create files with a `.yml` extension. Also, the Ansible documentation website references files with a `.yml` extension several times. Because of this, it is normal in the Ansible community to use a `.yml` extension for all Ansible Yaml files.

## Vaults

All Ansible Vault files MUST have a `.vault` extension (and NOT .yml, .YML, .yaml etc).

```sh
# bad
~/secrets.yml

# good
~/secrets.vault
```

### Why?

It is easier to control unencrypted files automatically for the specific `.vault` extension.

## Role Names

All the newly created Ansible roles will follow the name convention using dashes if necessary:

[company]-[action]-[function/technology]

```sh
# bad
lvm

# good
mycompany-setup-lvm
```

### Why?

If using roles from Ansible Galaxy, it will keep a consistency about which roles are created internally
