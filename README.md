# ansible Playbook for adding the inovia users to the machines

You have to run this playbook with the --ask-vault-pass.
Like this:
```
ansible-playbook playbooks/inovia_users/inovia_users.yml --ask-vault-pass
```
## what this playbook does.
This playbook adds users and groups to the inovia servers; handles SUDO rights on the
inovia server; copies the users ssh keys to all servers; sets up the secure sshd
settings that dissalows root to login through ssh and, and only enable users ssh keys;
sets message of the day; and it also sets the root password to the servers.
All this is handled in three diffrent roles.

users role:
handles adding of users and groups, sudo access, and motd message

secure_sshd_access role:
handles the sshd login settings.

rootpwd role:
handles the root password.

This playbook sets upp all the users on inovia instead of the old
security playbooks with all the diffrent roles.

Now there is one role named users. with only one roles/users/tasks/main.yml
and only one roles/users/vars/main.yml with the list of all the users.

ssh_keys are added to roles/users/tasks/files/

there is also the role to secure the sshd access to not allow root login
its located in the rolse/secure_sshd_access

## adding a user
edit roles/users/vars/main.yml
and add the following as below example
```bash
users:
- name: firstname.lastname
  user_groups:
    - 'staff'
    - 'developers'
  ssh_keys:
    - files/firstname.lastname
```
make sure the groups you add a user to exists as well. There is a list of groups
in the file defining the available groups to exist on all machines.

- 'staff' or - 'contractors'
Every user that is a person must be part of the "staff", or "contractors" group.
staff is everyone employed at inovia, and contractors are people working for inovia but not
under inovia employment.

- functional
Functional accounts are service accounts that need a home folder set up, and that are
not always set up by software.

- developers or - operations or - machinelearning..
Every user must be part of developers or operations or machinelearning or some other group.
A user under operations is in the operations team, and gets SUDO root privledges
on all inovia hosts. (see more under sudo privledges to give developers privledges on specific host.)
Other users are developers or something else. But every user has a group part from staff or contractor.

functional users are users such as jenkins, confluence, etc that need a home folder.

- functional, service accounts set up by software.
For users that do not need a home folder, service accounts under uid 1000, these are
usually set up by the software that needs them. Sometimes that software creates
users with a homefolder however. And if that user is not in the users list that
account risks to be deleted!!! This is a problem we found for the user "kube"..
So there is a list called users_to_keep
Be sure to add to users_to_keep so we do not delete these service accounts.
NEVER add a person at inovia to users_to_keep or an account that is used to log in to
the inovia servers.

recap
The default groups a user tied to a person must be a part of are:
- 'contractors' or
- 'staff'
A person also belongs to developers or operations.
- 'developers',
- 'operations' or
- 'functional'

## functional users
There are users that are functional users only.
example is user "jenkins"
these users belong to the group - functional

## adding a user with sudo privledges.
to add a user with sudo privledges on 2 hosts:
```bash
users:
- name: firstname.lastname
  user_groups:
    - 'staff'
    - 'operations'
  sudo:
    - 'ilxdemo'
    - 'ilxdemo2'
  ssh_keys:
    - files/firstname.lastname
```
sudo can also be 'ALL' this will give the same privledges to the user as if he/she
was part of the operations group. (they get sudo all by default.)

## deleting a user
users not in the vars list will be deleted if found on a host.
The playbook checks the contents of /home and deletes all accounts that exist on
the host but does not exist in this playbook.

## motd or message of the day.
The playbook also places a /etc/motd file on every host with text for every user that logs in.
This file can be changed in roles/users/templates/motd.j2

## sshd access on special machines.
The playbook allows for machine specific sshd configurations. For example the ilxsftp server
in use by SEB has some special users set up that access the sftp server with username and password.

To add special settings for some machines, you create a template file and append
the hostname to the file. For example: ilxsftp01 has a special template file in:
playbooks/inovia_users/roles/secure_sshd_access/templates/ilxsftp01.sshd_config.j2

## root password set
To set the root password on the inoviahosts you need some prerequisites.
You need to generate the hash of the password first.

Then you insert hash is inserted in the roles/rootpwd/tasks/main.yml that look like this: (the example hash is of course not the current one)
```
- name: Update Root user's Password
  user:
    name: root
    update_password: always
    password: $6$rounds=65600N$719l0dAzw3hcH1hdHsosLYXYZCQ9jHWDbItwr6yQ0MXCW00
```

This file (roles/rootpwd/tasks/main.yml) is however encrypted! you can only edit this
file with ansible-vault.
```
ansible-vault edit ansible/playbooks/inovia_users/roles/rootpwd/tasks/main.yml
```

But step by step to change this is:

Install passlib on your mac so you can generate hashed passwords
```
pip install passlib
```

use passlib to genrate a hashed password:
```
python -c "from passlib.hash import sha512_crypt; import getpass; print sha512_crypt.encrypt(getpass.getpass())"
```
(Use passwords that are at least 16chars long BUT only consists of A-Z, a-z, and 0-9 chars.
  If you use chars that are special the mapping of keys on the keyboard of mac and vmware in the console
  can get fucked up.)

Now replace the hashed string in main.yml with your new one.
```
ansible-vault edit ansible/playbooks/inovia_users/roles/rootpwd/tasks/main.yml
```
(uses vi to edit)

And importantly! You have to update the vaultfile where Operations stores all passwords.
Update the inovia vault file with the password! Not the hashed one..
If you dont do that, we have no record of what root password we have on the system.

ansible-vault edit ansible/etc/passwords/passwords.txt

Update the password under the text:
# current root password updated by inovia_users playbook on all inovia linux servers
