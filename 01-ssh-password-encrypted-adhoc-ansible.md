<!-- omit in toc -->
# Ssh password from encrypted file in ad-hoc ansible command

- [Context](#context)
- [Goal](#goal)
- [Environment](#environment)
- [Solution](#solution)
  - [Folder structure](#folder-structure)
  - [hosts.yaml](#hostsyaml)
  - [password.yaml](#passwordyaml)
- [Gotchas](#gotchas)
  - [EDITOR for ansible-vault](#editor-for-ansible-vault)
  - [hosts.yaml structure](#hostsyaml-structure)
  - [ansible_password variable](#ansible_password-variable)
- [Troubleshooting](#troubleshooting)
  - [Using ansible-inventory](#using-ansible-inventory)
  - [Using ansible -vvvv](#using-ansible--vvvv)
  - [Using ansible-vault](#using-ansible-vault)
- [Result](#result)
- [References](#references)

## Context

I have VPS and couple of VMs running Ubuntu 18.04 that I experiment e.g. deploying WordPress, mail server, Django apps. Fresh VPS has only root ssh access, so each time I have to create non-root user with non-empty password, add to sudo group, set up ssh key access for non-root user and then disable root access, configure firewall, install app(s). In this doc I consider only accessing remote host with ansible using ssh password authentication.

## Goal

Perform ad-hoc command with ansible on remote host using ssh password authentication. Ssh password is picked from encrypted file.

## Environment

I run ansible on Ubuntu 18.04 inside WSL (Windows Subsystem for Linux).

Ansible 2.9.2 installed with pip3 (rather than ubuntu package).

(Remote) hosts run Ubuntu Server 18.04.3, have OpenSSH server installed and ssh password authentication enabled (default).

## Solution

### Folder structure

```bash
.
├── group_vars
│   └── all
│       └── password.yaml
└── hosts.yaml
```

- ssh user name and password are part of inventory
- ssh user name is in inventory file `hosts.yaml`
- ssh password is in `password.yaml`
- `.yaml` extension makes it clear what format to expect
- `all` is folder rather than file, so it is possible to have passwords encrypted and other variables (not shown here) in plain text (unencrypted)
- `group_vars` must be in the same folder as inventory `hosts.yaml` file
- you may name files `hosts.yaml` and `password.yaml` differently, but `group_vars` and `all` are reserved names for ansible

### hosts.yaml

```yaml
all:
  hosts:
    ubuntu-18.04-macos:
      ansible_host: 192.168.1.100

    ubuntu-18.04-win10:
      ansible_host: 192.168.1.200

  vars:
    ansible_user: alice
```

- `all` is reserved group name
- `hosts` is reserved name for hosts list
- `ubuntu-18.04-macos` and `ubuntu-18.04-win10` are aliases in this case as my hosts doesn't have DNS names, only IP addresses
- `vars` is reserved name for group variables. Each variable under `vars:` is applied to each host in `all` group. Note indent of `vars:` is the same as `hosts:`'s
- `ansible_user` is user name on (remote) host (prefix ansible is confusing to me)

### password.yaml

```yaml
$ANSIBLE_VAULT;1.1;AES256
36363933643661353936323563393861366236643337623835323830616563393765343461313266<more digits>
```

- file is encrypted and starts with header (first line) followed by lines of digits. \<more digits\> is a placeholder for the rest of my file.
- if current folder is the one where `hosts.yaml` is located, command
  ```EDITOR=nano ansible-vault create group_vars/all/password.yaml```  
  will create file, let you enter variable and when you exit editor (`nano` in this example) writes encrypted file to disk.
- content of file in editor should be as follows:

  ```yaml
  ansible_password: @lice$passw0rd
  ```

  @lice$passw0rd is placeholder you should replace with your password. Note space after colon.

## Gotchas

### EDITOR for ansible-vault

I'm using Visual Studio Code as text editor and initially started `ansible-vault` from WSL bash console like this:

```bash
EDITOR=code ansible-vault create group_vars/all/password.yaml
```

This resulted in encrypted file that had no variables. As I didn't know how output of e.g. `ansible-vault view` should look like I spent quite some time before understood I have no variable `ansible_password`.

### hosts.yaml structure

Most examples are in INI file format. I didn't quite get what are mandatory elements for YAML format. In particular I created `hosts.yaml` with `hosts:`. This lead to `vars:` being indented relative to `hosts:`, while they must have equal indents.

### ansible_password variable

I created variable `ansible_pass`. If it where `ansible_ssh_pass` it would work. So always consult reference about variable names.

## Troubleshooting

All commands run from WSL console with folder containing `hosts.yaml` being a current folder.

### Using ansible-inventory

```bash
ansible-inventory --ask-vault-pass -i hosts.yaml --host ubuntu-18.04-win10
```

Must print JSON with three variables, password in plain text.

### Using ansible -vvvv

Until I found that encrypted file is empty logging in with ssh succeeded, while logging in with ansible failed. `-vvvv` switch to the rescue.

```bash
ansible ubuntu-18.04-win10 -vvvv --ask-vault-pass -i hosts.yaml -a "echo $TERM"
```

This printed ssh command line and it had password authentication disabled. This hinted me that there may be no password provided.

### Using ansible-vault

To check that encrypted file has variable(s) you expect, use

```bash
ansible-vault view group_vars/all/password.yaml
```

You must see variables and plain text (unencrypted) values.

## Result

```bash
ansible ubuntu-18.04-win10 --ask-vault-pass -i hosts.yaml -a "echo $TERM"
Vault password:
ubuntu-18.04-win10 | CHANGED | rc=0 >>
xterm-256color
```

## References

- [ansible ssh connection variables](https://docs.ansible.com/ansible/latest/plugins/connection/ssh.html#ssh-connection)
