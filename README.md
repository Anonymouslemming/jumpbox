# Jumpbox ansible roles

## Overview

This set of playbooks will turn a general Ubuntu LTS (14.04) host into a jump box. This is a box that acts as an SSH gateway for users to your network, and also provides an OpenVPN server (with CA) to allow non-SSH access. 

As a general rule, you should *not* run any other unneeded services on a host like this, and you should use something more secure than passwords for authentication.

The approach implemented here requires that users have SSH keys present on the server, and OpenVPN authentication is done using keys generated by the easy-rsa CA solution.

There are two ansible roles here:

   * jumpbox - sets up SSH and creates users and groups
   * vpn_with_ca - OpenVPN with Easy RSA certificate authority
   
The firewall component is currently part of the vpn_with_ca role, but may be split out into a separate role soon.

---

## Requirements
The Ansible controller should be running the latest version from https://github.com/ansible/ansible and the target must be an Ubuntu 14.04 LTS host. If there is any interest, I'll modify this to work with Centos 7 as well.

You should also have one existing user who is able to ssh to the host using a key to authenticate, and be able to sudo to root. This role will disable login for root over SSH and also disable password logins unless you make some changes.

---

## Basic Usage
This assumes that all variables and templates are already exactly as you would like them. **This is not likely to be true!** For more information about customization and options availalble, *please* read the rest of ths README. 

#### Host to deploy to 
You need to either edit your ansible inventory and create a new group named `jumpbox` that includes all hosts that you want to run this against *or* you need to edit `jumpbox.yml` and change the `jumpbox` in the hosts line to the name of the host that you wish to use.

#### Deployment
Simply run `ansible-playbook jumpbox.yml` and watch the output. 

#### Generating individual client ovpn config files
To generate an ovpn config for a client, you'll use the `create_vpn_key.yml` playbook. You need to supply the name of the VPN host that you want to use and also the name of the client to create on the commandline:

```ansible-playbook --extra-vars '{"vpnhost": "<yourvpnhost>", "client": "<clienttocreate>"}' create_vpn_key.yml```

For example, if I wanted to create an ovpn config with on a host named jumpbox for my ipad, I might use 

```ansible-playbook --extra-vars '{"vpnhost": "jumpbox", "client": "anonymouslemming-ipad"}' create_vpn_key.yml```

Once you've run this, you'll find the device key, certificate, csr and ovpn config under `/etc/openvpn/easy-rsa/keys` prefixed with the client value you set above. In my example, I would have 
   * anonymouslemming-ipad.crt
   * anonymouslemming-ipad.csr
   * anonymouslemming-ipad.key
   * anonymouslemming-ipad.ovpn

You can then provide whichever of these files is needed for your OpenVPN client application to your users.

---

## Variable files and templates 
### jumbox
#### variables - roles/jumpbox/vars/main.yml
There are two sections here - unixgroups and unixusers. These are each a list of dictionaries of values that will be used when creating these users and groups.

The idea here is that instead of manually creating users on the box in the future, you simply rerun the playbook. This will create any new users, and you'll have consistent uid and gids for each object.

#### templates - roles/jumpbox/templates
##### etc_ssh_sshd_config.j2
This is the SSH server config that will be deployed to the jump box. It's close to the Ubuntu default, with the following changes:
   * PermitRootLogin no
   * PasswordAuthentication no
   * LoginGraceTime 30
   * MaxStartups 3:50:10
   
If you still want root ssh access via key after running this playbook, change 'PermitRootLogin no' to 'PermitRootLogin without-password'

### vpn_with_ca
#### variables - roles/vpn_with_ca/vars/main.yml
Here, you'll define your VPN network (not your private network - the network range to be used by OpenVPN) and your key details that will be used for both the OpenVPN server and also individual device or user vpn keys. 

#### templates - roles/vpn_with_ca/templates
##### etc_openvpn_server.conf.j2
This is the OpenVPN server config and will be deployed to /etc/openvpn/server.conf. Sensible defailts have been used and you shouldn't need to edit this for basic usage. 

Ansible will use some information from your host when populating this template, specifically the nameservers and search entries from the ansible_dns section. If you're having problem with DNS lookups after deploying this, check that these are populated by running `ansible -m setup <yourvpnhost>` and look for the ansible_dns section.
	
##### etc_openvpn_easy-rsa_vars.j2
Deployed to /etc/openvpn/easy-rsa/vars and used by all Easy RSA scripts. The variables from roles/vpn_with_ca/vars/main.yml are used here.

##### etc_iptables_rules.v4.j2
We'll be using the iptables-persistent package to load and save rules. This template 
   * sets up masquerading for VPN connections
   * allows established connections 
   * allows all connections from jump box's network 
   * allows traffic tcp/22 and udp/1194 from any location
   
##### etc_openvpn_sample.ovpn.j2
Last, but not least, we have the template used to create ovpn files once the vpn is deployed. This will not be copied to the server, but instead used each time you use the `create_vpn_key.yml` playbook. 

There is one change that is likely to be needed here, and that's the `remote` line. This should be changed to something that clients can reach from outside your network. 




