# Ansible Role: NVM


Installs NVM & Node.js on Debian/Ubuntu and RHEL/CentOS systems


Ansible weirdness with SSH and (non)interactive shells makes working with NVM and Ansible a bit problematic. This [stack overflow](https://stackoverflow.com/questions/22256884/not-possible-to-source-bashrc-with-ansible) post explains some of the things other people have done to get around this particular issue.

## Where other roles fall short
Other Ansible roles that install NVM and/or Node.js fall short in a few areas.

1. They use the apt-get or yum packages managers to install Node.js. This often means that the Node.js package is older than what is currently available via the Node.js repo. In some cases, those packages may not be a LTS release and if you need multiple Node.js versions running on the same host, you're out of luck.

1. They will often install NVM and Node.js as `root` user (`sudo su` or `become: true`). This can add to the headache of permissions related to NPM plugin management as well as how Node functions with nvm in addition to being an unneeded privilege escalation security risk

1. You cannot run ad hoc nvm, npm, node, bash or shell commands


## Where this role differs from other roles

1. You can install NVM via wget, curl or git
1. You can use NVM just like you would via your [command line](https://github.com/creationix/nvm#usage) in your own Ansible tasks and playbooks
1. You can install whatever **version** or **versions** of Node.js you want
1. Doesn't install NVM or Node.js as root
1. Can run arbitrary nvm, npm, node, bash or shell commands potentially eliminating the need for a separate Node Ansible role all together

## Installation
1. Clone this repo into your roles folder
1. Point the `roles_path` variable to the roles folder i.e. `roles_path = ../ansible-roles/` in your `ansible.cfg` file
1. Include role in your playbook

---

## :warning: WARNING!
DO NOT RUN THIS ROLE AS ROOT!

There are a few reasons for this,
1. It is an unneeded privilege escalation security risk, **it is highly unlikely that you need to run every task in every role as `root_user`**. If, for whatever reason, you do need to run everything as `root_user`, reconsider what the role is doing and why it needs root access for everything.

1. This role installs nvm in the same context/shell/session as you would run NodeJS. You don't run NodeJS as `root`

1. Ansible will change the context of the login shell to `root` and nvm will be installed in the `root_user` home directory e.g `/root/.bashrc`. This means if your primary user is **vagrant**, **ec2-user**, **ubuntu** etc. the role **WILL NOT WORK AS EXPECTED!**

BAD :thumbsdown:
```yaml
- hosts: all
  become: yes           # THIS RUNS ALL TASKS, FOR ALL HOSTS, AS ROOT_USER
  become_method: sudo   # THIS RUNS ALL TASKS, FOR ALL HOSTS, AS ROOT_USER

  roles:
    - role: ansible-role-nvm
      nodejs_version: "8.16.0"
      nvm_commands:
       - "nvm exec default npm install"

    - role: some-other-role
      ...

```

BETTER :thumbsup:

```yaml
- hosts: all

  roles:
    - role: ansible-role-nvm
      nodejs_version: "8.16.0"
      nvm_commands:
       - "nvm exec default npm install"

    - role: some-other-role
      ...
      become: yes             # THIS SCOPES ALL TASKS, ONLY FOR THE SOME-OTHER-ROLE, AS ROOT_USER
      become_method: sudo     # THIS SCOPES ALL TASKS, ONLY FOR THE SOME-OTHER-ROLE, AS ROOT_USER

```

BEST :metal:

```yaml
- hosts: all

  roles:
    - role: ansible-role-nvm
      nodejs_version: "8.16.0"
      nvm_commands:
       - "nvm exec default npm install"
      become_user: ec2-user   # THIS INSTALLS NVM IN THE CONTEXT OF THE EC2-USER/DEFAULT USER

    - role: some-other-role
      ...
      become: yes             # THIS SCOPES ALL TASKS, ONLY FOR THE SOME-OTHER-ROLE, AS ROOT_USER
      become_method: sudo     # THIS SCOPES ALL TASKS, ONLY FOR THE SOME-OTHER-ROLE, AS ROOT_USER

```

See [Issues](#issues) below for further details

---


## Example Playbooks


#### Super Simple
Include the role as is and it will install latest LTS version of Node.js
``` yaml
- hosts: all

  roles:
    - role: ansible-role-nvm
```

#### Simple
Include the role and specify the specific version of Node.js you want to install
``` yaml
- hosts: all

  roles:
    - role: ansible-role-nvm
      nodejs_version: "8.15.0"

```
#### More Complex
This example shows how you might set up multiple environments (Dev/Prod) with different options. The Prod setup takes advantage of the `nvm_commands` option to install, build and run the application. The role supports and takes advantage of Ansible variable syntax e.g. `{{ variable_name }}`.
``` yaml
- hosts: dev

  vars_files:
    - vars/dev.yml

  roles:
    - role: ansible-role-nvm
      nodejs_version: "{{ config.dev.nodejs.version }}"


- hosts: prod
  vars_files:
    - vars/prod.yml

  roles:
    - role: ansible-role-nvm
      nvm_install: "curl"
      nvm_dir: "/usr/local/nvm"
      nvm_commands:
       - "nvm install {{ config.prod.client-1.nodejs.version }}"
       - "nvm alias default {{ config.prod.client-1.nodejs.version }}"
       - "nvm exec default npm install"
       - "nvm exec default npm run prod"

```

## Installing/Running/Maintaining or Upgrading multiple versions of Node.js on the same host

By default, the **first** Node.js version instantiated in your Playbook will automatically be aliased as the "default" version regardless of whatever version you install afterwards or how many times you run the role. It is important to declare which version is expected to be the "default" version.

There are two pre-existing NVM aliases `default` (current "active" version of Node.js) and `system` (the base OS version of Node.js).

*Aliasing is a very powerful feature of NVM and it is a **recommended best practice** for managing your environment*.

#### Multi Install
``` yaml

- hosts: host-1

  roles:
    # Services
    - role: ansible-role-nvm
      nodejs_version: "8.15.0"    # <= This will be the "default" version of Node.js

    # Application
    - role: ansible-role-nvm
      nodejs_version: "10.15.0"

```

#### Multi Install w/ default
``` yaml

- hosts: host-2

  roles:
    # Services
    - role: ansible-role-nvm
      nodejs_version: "8.15.0"    

    # Application
    - role: ansible-role-nvm
      default: true
      nodejs_version: "10.15.0" # <= This is now the "default" version of Node.js

```

<a name='#nvm-commands'></a>
## Notes on NVM commands
**NVM commands are a very powerful feature of this role** which takes advantage of the groundwork NVM has set up. Leveraging `nvm_commands` could potentially eliminate the need for a specific Node role to manage your Node applications altogether.

There is a difference between `nvm run` and `nvm exec` commands. `nvm run` is functionally equivalent to `node server.js` or `node server` where you are invoking a JavaScript file

`nvm exec` executes in a sub process context and is functionally equivalent to `npm run server` where `server` is a key name in the scripts section in the `package.json` file
``` json
{
  "name": "my_application",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "preserver": "npm run dbService &",
    "server": "nodemon ./bin/www",
    "build": "node build/build.js",
    "dbService": "nodemon ./data-service/server.js --ignore node_modules/"
  },
  "dependencies": {
    "..."
  }
}
```

OR

`nvm exec` can execute some arbitrary script file .e.g. `nvm exec hello-world.py`

e.g hello-world.py

```python
#!/usr/bin/env python
print('hello-world')
```
*:warning: You must include a script header for this to work properly*



`nvm_commands` make it very easy to set up a Node Application and Node API layer running on different version of Node.js on the same host

``` yaml

- hosts: host-1
  roles:
    # Services
    # WHAT'S HAPPENING?
    # 1. Run the services JavaScript file with Node version 8.15.0
    # WARNING: This is aliased as the default version of Node.js At this point !!
    # Therefore We need to explicitly specify the version we're using because
    # the default Node.js version changes in Application section below
    - role: ansible-role-nvm
      nodejs_version: "8.15.0"
      nvm_commands:
        - "nvm exec 8.15.0 npm run services"

    # Application
    # WHAT'S HAPPENING?
    # 1. Set the default version of Node.js to version 10.15.0
    # 2. Install package dependencies with npm
    # 3. Set the environment to Production, run the build JavaScript file
    # 4. Then run the production deploy script
    - role: ansible-role-nvm
      nodejs_version: "10.15.0"
      nvm_commands:
       - "nvm alias webapp {{ nodejs_version }}" # <= Changes the default NVM version (supports Ansible variable syntax)
       - "nvm exec webapp npm install" # install app dependences
       - "NODE_ENV=production nvm run webapp build" # invoke Node.js directly to run the production build script
       - "nvm exec webapp npm run prod" # invoke npm to run the production script in your package.json file

```
Another example

``` yaml

- hosts: host-2
  roles:
    # Services
    # WHAT'S HAPPENING?
    # 1. Create an Alias for version 8.15.0 entitled service-default (Supports Ansible variable syntax)
    # 2. Run the services script
    #
    # ** It is recommended that you alias your Node.js versions and reference them accordingly **
    - role: ansible-role-nvm
      nodejs_version: "8.15.0"
      nvm_commands:
        - "nvm alias service-default {{ nodejs_version }}" # <= (Supports Ansible variable syntax)
        - "nvm exec service-default npm run services" # run the services script in your package.json file


    # Application - No separate Node.js Ansible Role Needed
    # WHAT'S HAPPENING?
    # 1. Install version 10.15.0 of Node.js
    # 1. Set the default version of Node.js to version 10.15.0
    # 2. Run the test.js script file invoking Node.js directly
    # 3. Then run the production deploy bash script
    - role: ansible-role-nvm
      nodejs_version: "10.15.0"
      nvm_commands:
       - "nvm alias default 10.15.0" # <= Changes the default NVM version
       - "nvm exec default node test.js" # invoke Node.js directly to run the test script
       - "nvm exec ./deploy.sh" # run an arbitrary bash script

```

**Whatever command line arguments you use to start your application, or command scripts you've declared in your package.json file can be placed inside the `nvm_commands: []` section of this role.**





## Caveats

1. By default, the **first** version listed in your Playbook, on the **first** run, will automatically be aliased as the "default" version of Node.js regardless of whatever version you install afterwards or however many times you run the role. **First one in/installed is always the default.** As a result, if you expect a Node.js version declared later in the playbook to be set as default use `default: true` or explicitly set it in the `nvm_commands` list like `- "nvm alias default <YOUR_VERSION>"`

1. If you have `default: true` explicitly declared as a role variable **AND** `- "nvm alias default <SOME_OTHER_VERSION>"` as part of your `nvm_commands` the version with `default: true` will **ALWAYS** be executed **first**. This is because we need Node.js to be available before doing anything else.  

1. NVM is stateless in that if you have multiple versions of Node.js installed on a machine, you may have to run `nvm use <VERSION>` as part of your script to run the Node.js version you want/expect. However, it is higly recommended that you alias your versions accordingly and reference them that way. See the examples above.

<a name='#issues'></a>
## Issues


#### `"nvm: command not found" error`

This is often the result of running the role in another user context then the nvm and node user context will run inside the machine. If you add `become: true` to all the roles in your playbook to get around errors those roles throw due to permission issues, then this role will install nvm under the `ROOT_USER` (usually `/root/.bashrc`). **It is more than likely that you will want to run nvm and node as a default user e.g. vagrant, ec2-user, ubuntu etc.** If, for whatever reason, you cannot remove the `become: true` for everything, you can get around the `become: true` issue by specifying `become_user: ec2-user` for this role alone. See [bash: nvm command not found
](https://github.com/morgangraphics/ansible-role-nvm/issues/16) for a detailed explanation of the issue


#### `"cannot find /usr/bin/python" error`

It is due to OS's that run Python 3 by default (e.g. Fedora). You will need to specify the Ansible python interpreter variable in the inventory file or via the command line

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

---


## Role Variables

Available variables are listed below, along with default values see [defaults/main.yml]( defaults/main.yml)

The Node.js version to install. The latest "lts" version is the default and works on most supported OSes.

    nodejs_version: "lts"

Convenience method for installing NVM bash autocomplete (`nvm <TAB>`) when a user has to maintain a server or workstation manually

    autocomplete: false


Set default version of Node when maintaining/installing multiple versions

    default: false

> NVM will automatically alias the first run/installed version as "default" which is more than likely what people will use this role  for, however, this will allow for installation/upgrade of multiple versions on an existing machine


List of [NVM commands to run](https://github.com/creationix/nvm#usage). Default is an empty list.

    nvm_commands: []

NVM Installation type. Options are wget, curl and git

    nvm_install: "wget"

NVM Installation directory.

    nvm_dir: ""

> *NVM will, by default, install the `.nvm` directory in the home directory of the user e.g. `/home/vagrant/.nvm`. You can override the installation directory by changing this variable e.g. `/opt/nvm` to put it into a global space (not tied to a specific user account) if you wanted. This variable will respect Ansible substitution variables e.g. `{{ansible_env.HOME}}`*

NVM Profile location Options are .bashrc, .cshrc, .tcshrc, .zshrc

nvm_profile: ".bashrc"


> *The location of the SHELL profile that will source the nvm command from. There are two potential contexts to consider, globally, meaning everyone who logs in will have access to nvm (which may or may not what you really want) e.g **/etc/bash.bashrc**, **/etc/profile**, etc.*

> **OR**

> *On a per user basis tied to a specific user account e.g. /home/vagrant/.bashrc.*
>  *This role will create the appropriate profile file if it doesn't already exist.*

> *If you specify nvm_profile: "/home/node-user/.bashrc" explicity and the node-user is not a real  user on the box, then nvm will not work as you expect. become_user and nvm_profile path are  symbiotic*
>
> :warning: **PLEASE BE AWARE OF THE LIMITATIONS OF EXPLICITLY DECLARING .profile OR .bash_profile FILES ON UBUNTU SYSTEMS**
>
>  https://askubuntu.com/a/969923 Explains in detail
>
>  https://kb.iu.edu/d/abdy Shows options for each shell type
>
>  NVM Profile location Options are:
  >
  >  **BASH**: .bashrc
  >
  >  **CSH**: /etc/csh.cshrc, .cshrc
  >
  >  **TSCH**: /etc/csh.cshrc, .tcshrc, .cshrc
  >
  >  **ZSH**: .zshrc


NVM source location i.e. you host your own fork of [NVM](https://github.com/creationix/nvm)

    nvm_source: ""


NVM version to install

    nvm_version: "0.35.0"


Uninstall NVM, will remove .nvm directory and clean up `{{ nvm_profile }}` file

    uninstall: False




## Dependencies

None.

## Change Log

**1.4.0**
* Code Linting, Indempotency updates for CI/CD testing
* Addressed version check as reported by [@Jamesking56](https://github.com/morgangraphics/ansible-role-nvm/issues/18)

**1.3.0**
* Addressed `nvm: command not found error` bug as reported by [@eyekelly](https://github.com/morgangraphics/ansible-role-nvm/issues/16).
* Updated documentation in greater detail about user context/session/shells to guard against `nvm: command not found error`.
* Updated default variable explanations
* Reworked documentation and examples surrounding `nvm_commands`
* NVM version bump

**1.2.2**
* NVM version bump

**1.2.1**
* Documentation updates for clarity

**1.2.0**
* Addresses issues [#8 Add default: True role variable to ensure NVM default alias is set correctly](https://github.com/morgangraphics/ansible-role-nvm/issues/8), [#9 Git functionality has changed according to the NVM documentation](https://github.com/morgangraphics/ansible-role-nvm/issues/9), [#10 NVM has an Autocomplete functionality. Add `autocomplete: True` Ansible variable to role](https://github.com/morgangraphics/ansible-role-nvm/issues/10), [#11 Update documentation to highlight updating a default version](https://github.com/morgangraphics/ansible-role-nvm/issues/11), and [#12 Add remove: True variable to uninstall NVM ](https://github.com/morgangraphics/ansible-role-nvm/issues/12) as discussed with [@DanHulton](https://github.com/morgangraphics/ansible-role-nvm/pull/7) to address multiple version of Node.js running on the same host.
* Expanded documentation with examples about how powerful `nvm_commands: []` can be

**1.1.2**
* Issue reported/PR supplied by [@DanHulton](https://github.com/morgangraphics/ansible-role-nvm/pull/5), Documentation updates,

**1.1.1**
* Documentation updates

**1.1.0**
* Issue reported by [@magick93](https://github.com/morgangraphics/ansible-role-nvm/issues/3), Bumped default version of NVM script, Documentation updates

**1.0.2**
* Issue reported by [@swoodford](https://github.com/morgangraphics/ansible-role-nvm/issues/1), Bumped default version of NVM script, Documentation updates

## License

MIT / BSD

## Author Information

dm00000 via MORGANGRAPHICS, INC

This role borrows heavily from [Jeff Geerling's](https://www.jeffgeerling.com/) Node.js role, author of [Ansible for DevOps](https://www.ansiblefordevops.com/).
