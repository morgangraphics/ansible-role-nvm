# Dev Notes

## Working with venv

```bash

```

## ansible-lint

When running ansible-lint as part of the `molecule lint` command it is looking for the 
ansible roles in the wrong place. 

```bash
export ANSIBLE_ROLES_PATH=${MOLECULE_PROJECT_DIRECTORY}/..
```

in the `lint: |` key of the molecule,yml file addresses the issue. [#3055](https://github.com/ansible-community/molecule/issues/3055)


## Testing locally

```bash
ansible-playbook molecule/default/verify.yml -i molecule/default/inventory --extra-vars=active_playbook=autocomplete -vvv
```