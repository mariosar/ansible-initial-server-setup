# Ansible Initial Server Setup

This ansible playbook contains tasks you would normally run when setting up a new Ubuntu Server.

Ubuntu Server version: 18.04.3 LTS

Tasks this playbook will handle:

**USER_MANAGEMENT**:
- Create a sudo user with password
- Change the default root password

**SSH CONFIG**:
- Modify the standard ssh port to a custom port
- Prevent root login
- Add a specified public key to authorized_keys of new sudo user (for ssh without password)
- Prevent password authentication

**COMMON**:
- Update package cache / Upgrade Packages
- Install UFW, Fail2Ban

**SECURITY**:
- UFW: Deny all incoming, but allow SSH on custom port
- Fail2Ban: Enable Fail2Ban Jail for SSHD + SSHD-DDOS

### Instructions:

1. Enter your server IP addresses in inventory files based on environment (staging / production).
2. Edit group_vars/all/vars and enter your desired SUDO_USER and modify the SSH port (or keep standard port).
3. Create ansible vault in group_vars/staging/vault and group_vars/production/vault. Each vault should contain values for that environment in YAML. The values you need to include are:
  - vault_root_password (the new root password)
  - vault_sudo_password (the desired sudo password for your sudo user)
  - vault_sha512_secret_salt (a hashing salt to encrypt the passwords)

See [ansible-vault](https://docs.ansible.com/ansible/latest/user_guide/playbooks_vault.html) if you're not sure how to create the vaults.

### Running your playbook:

The first time running your playbook the default port is 22 and the user is root. We haven't yet done anything on our servers.

```
ansible-playbook -i inventory -e ansible_port=22 -u root site.yml --ask-vault-pass

# replace inventory with either production or development, depending on which environment you are targetting.
```

The second time we run our playbook, if we desire to make changes like modifying the ssh port or adding an additional sudo user, modifying the existing sudo user password, etc, we run:

```
ansible-playbook -i inventory -e ansible_port=PORT -u USER site.yml --ask-vault-pass --ask-become-pass

# replace inventory with either production or development, depending on which environment you are targetting.
```

Hope you find this useful and feel free to use it and modify to fit your needs.
