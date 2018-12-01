# Ansible Role: NVM


Installs NVM & Node.js on Debian/Ubuntu and RHEL/CentOS

Ansible weirdness with SSH and (non)interactive shells makes working with NVM and Ansible a bit problematic. This [stack overflow](https://stackoverflow.com/questions/22256884/not-possible-to-source-bashrc-with-ansible) post explains some of the things other people have done to get around this issue.

## Were other roles fall short
Other Ansible roles that install NVM and/or Node.js fall short in a few areas.

1. They use the apt-get or yum packages to install Node.js. This often means that the Node.js package is older than what is currently available via the Node repo. In some cases, those packages may not be a LTS release and if you need multiple node versions, you're out of luck.

1. They will often install NVM and Node.js as `root` user (`sudo su` or `become: true`) to install NVM. This can add to the headache of NPM plugin management in addition to being an unneeded privilege escalation security risk

1. You cannot run ad hoc NVM commands


## Where this role differs

1. You can install NVM via wget, curl or git
1. You can use NVM just like you would via your [command line](https://github.com/creationix/nvm#usage)
1. You can install whatever version of Node.js you want
1. Doesn't install NVM or Node.js as root

## Installation
1. Clone this repo into your roles folder
1. Point the `roles_path` variable to the roles folder i.e. `roles_path = ../ansible-roles/` in your `ansible.cfg` file
1. Include role in your playbook


## Example Playbooks

Playbooks are set up as an 'either/or' situation in regards to `nodejs_version` and
`nvm_commands`. It is one or the other, it cannot be both.

1. If you want to just install NVM, include the `nodejs_version` as part of the role (see simple).
1. If you use `nvm_commands` you will have to add the `nvm install VERSION` explicitly (see more complex).

#### Simple
``` yaml
- hosts: all
  roles:
    - role: ansible-role-nvm
      nodejs_version: "4.8.0"

```
#### More Complex
``` yaml
- hosts: dev
  vars_files:
    - vars/dev.yml
  roles:
    - role: ansible-role-nvm
      nodejs_version: " {{ config.dev.version }}"


- hosts: prod
  vars_files:
    - vars/prod.yml
  roles:
    - role: ansible-role-nvm
      nvm_install: "curl"
      nvm_dir: "/usr/local/nvm"
      nvm_commands:
       - "nvm install {{ config.prod.client-1.nodejs }}"
       - "nvm alias default {{ config.prod.client-1.nodejs }}"
       - "nvm run default app.js"

```


## Notes

If you specify `nodejs_version` and `nvm_commands`, `nodejs_version` will be ignored.
If you do not explicitly specify the `nvm install VERSION` as part of the `nvm_commands`
Node.js will not be installed and any subsequent commands WILL NOT WORK as expected.


If you are getting a "cannot find /usr/bin/python" error. It is due to OS's that run Python 3 by default (i.e. Fedora). You will need to specify the Ansible python interpreter variable in the inventory file or via the command line

```
[fedora1]
192.168.0.1 ansible_python_interpreter=/usr/bin/python3


[fedora2]
192.168.0.2

[fedora2:vars]
ansible_python_interpreter=/usr/bin/python3

```
or
```
ansible-playbook my-playbook.yml -e "ansible_python_interpreter=/usr/bin/python3"
```







## Role Variables

Available variables are listed below, along with default values see [defaults/main.yml]( defaults/main.yml)

The Node.js version to install. The latest "lts" version is the default and works on most supported OSes.

    nodejs_version: "lts"

NVM version to install

    nvm_version: "0.33.11"

List of NVM commands to run. Default is an empty list.

    nvm_commands: []

NVM Installation type. Options are wget, curl and git

    nvm_install: "wget"

NVM Installation directory

    nvm_dir: "{{ansible_env.HOME}}/.nvm"

NVM Profile location Options are .profile, .bashrc, .bash_profile, .zshrc

    nvm_profile: ".bashrc"

NVM source location i.e. you host your own fork of [NVM](https://github.com/creationix/nvm)

    nvm_source: ""

<!--
    nodejs_install_npm_user: "{{ ansible_ssh_user }}"

The user for whom the npm packages will be installed can be set here, this defaults to `ansible_user`.

    npm_config_prefix: "/usr/local/lib/npm"

The global installation directory. This should be writeable by the `nodejs_install_npm_user`.

    npm_config_unsafe_perm: "false"

Set to true to suppress the UID/GID switching when running package scripts. If set explicitly to false, then installing as a non-root user will fail.

    nodejs_npm_global_packages: []

A list of npm packages with a `name` and (optional) `version` to be installed globally. For example:

    nodejs_npm_global_packages:
      # Install a specific version of a package.
      - name: jslint
        version: 0.9.3
      # Install the latest stable release of a package.
      - name: node-sass
      # This shorthand syntax also works (same as previous example).
      - node-sass


    nodejs_package_json_path: ""

Set a path pointing to a particular `package.json` (e.g. `"/var/www/app/package.json"`). This will install all of the defined packages globally using Ansible's `npm` module.

-->

## Dependencies

None.



## License

MIT / BSD

## Author Information

dm00000 via MORGANGRAPHICS, INC

This role borrows heavily from [Jeff Geerling's](https://www.jeffgeerling.com/) Node.js role, author of [Ansible for DevOps](https://www.ansiblefordevops.com/).
