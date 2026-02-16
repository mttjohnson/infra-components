# mttjohnson's Ansible Collection

Experimental ansible pieces

Namespace: mttjohnson.infra

Installing this collection may be possible with Ansible Galaxy like this:
```bash
# public repo main branch
ansible-galaxy collection install git+https://github.com/mttjohnson/infra-components.git#/ansible/,main
# private repo specific commit pinning
ansible-galaxy collection install git@github.com:mttjohnson/infra-components.git#/ansible/,38e9adb955f5db3780e4c07270d1c23787199b1b
# private repo tagged version or branch name
ansible-galaxy collection install git@github.com:mttjohnson/infra-components.git#/ansible/,v0.1.0
# path to repo at a local disk path
ansible-galaxy collection install git+file:///Users/mattjohnson/projects/infra-components#/ansible/
```

https://docs.ansible.com/ansible/latest/dev_guide/developing_collections_distributing.html

Rather than trying to extract a copy of a repo or distribution archive it may be simpler to symlink a directory in the current repo so that upstream contributions can be made directly to the existing checked out repo.

This command could be run from an implementation ansible directory to symlink the collection:
```bash
ln -fs "/Users/mattjohnson/projects/infra-components/ansible/" "collections/ansible_collections/mttjohnson/infra"
```
