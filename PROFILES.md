# Ansible Role: NVM : Profiles

Go Back to ansible-role-nvm [README.md](README.md)

This role can install NVM both locally and at a system wide, global level. However, the key difference between the two file paths lies in their execution based on the shell type: Interactive or Non-Interactive (Login).


> You will have to use `nvm use <YOUR_NODE_VERSION_NAME>` in your tasks if mmultiple versions are installed on the system or [see the examples about aliasing](README.md#nvm_commands)

## Examples

  ```yaml
  nvm_profile: ".bashrc"
  ```

  Will install NVM at a user level and be accessible via an interactive shell

  ```yaml
  nvm_profile: ".profile"
  ```

  Will install NVM at a user level and be accessible via a login (non-interactive) shell


  ```yaml
  nvm_profile: "/etc/bashrc"
  nvm_dir: "/opt/.nvm"
  ```

  **OR (depending on \*nix distribution)** 

 ```yaml
  nvm_profile: "/etc/bash.bashrc"
  nvm_dir: "/opt/.nvm"
  ```

  Will install NVM at a global level, is accessible to everyone on the system, and be accessible via an interactive shell

```yaml
  nvm_profile: "/etc/profile"
nvm_dir: "/opt/.nvm"
  ```

  Will install NVM at a global level, is accessible to everyone on the system, and be accessible via a login (non-interactive) shell




## Where to put the NVM command in a Global context


### Most Usecases

If you want the command to be available at a global level, avalable to anyone at any time, it is recommended to add it to the global bashrc file:

  ```yaml
  nvm_profile: "/etc/bashrc"
  nvm_dir: "/opt/.nvm"
  ```

  **OR (depending on \*nix distribution)** 

 ```yaml
  nvm_profile: "/etc/bash.bashrc"
  nvm_dir: "/opt/.nvm"
  ```

### Login shell Only

If you want the command to be available at a global level only for login shells, it is recommended to add it to the global profile file:

  ```yaml
  nvm_profile: "/etc/profile"
  nvm_dir: "/opt/.nvm"
  ```


## More info



A thorough explination of interactive and non-interactive shells can be read at this [ask buntu](https://askubuntu.com/questions/247738/why-is-etc-profile-not-invoked-for-non-login-shells) page

> ⚠️ When installing NVM in a global context, `nvm_dir` variable must also be declared 


*This role will create the appropriate profile file if it doesn't already exist.*

*If you specify nvm_profile: "/home/node-user/.bashrc" explicity and the node-user is not a real  user on the box, then nvm will not work as expected. `become`, `become_user` and `nvm_profile` path are symbiotic*

⚠️ **PLEASE BE AWARE OF THE LIMITATIONS OF EXPLICITLY DECLARING .profile OR .bash_profile FILES ON UBUNTU SYSTEMS**

 [https://askubuntu.com/a/969923](https://askubuntu.com/a/969923) Explains in detail

  [https://kb.iu.edu/d/abdy](https://kb.iu.edu/d/abdy) Shows options for each shell type

  NVM Profile location Options are:

  **BASH**: .bashrc

  **CSH**: /etc/csh.cshrc, .cshrc

  **TSCH**: /etc/csh.cshrc, .tcshrc, .cshrc

  **ZSH**: .zshrc