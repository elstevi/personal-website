<!--
.. title: linux phonehome
.. slug: linux-phonehome
.. date: 2018-03-27 23:05:29 UTC-07:00
.. tags: hax, linux, ssh
.. category: 
.. link: 
.. description: 
.. type: text
-->

Often when deploying a new machine, I have it reverse-tunnel into my jump server so that I can always get to it via ssh, despite mixed firewall environments. I am used to doing this with FreeBSD systems, but when doing it on Linux I often forget the systemd unit file syntax.

<!-- TEASER_END -->

<h2>Generate ssh keypair on new system</h2>
As myself:
```
ssh-keygen
```
Make sure you do not use a paraphrase, at least for this key. You may want to name it id_rsa_phonehome so that your regular ssh key on this machine can use a paraphrase.
Copy the newly generated public key to the clipboard, you will need it in the next step.
```
cat ~/.ssh/id_rsa_phonehome.pub
```

<h2>Add user, public key on jump server</h2>
This new machine is called 'redrocket'. I am going to create a new user on this machine with no privileges called 'sdouglas-redrocket' and set a random password. My ssh servers only accept private/public keypairs, no password authentication.
```
adduser sdouglas-redrocket
su sdouglas-redrocket
cd ~/
mkdir -p .ssh
```
Paste your key into <code>~/.ssh/authorized_keys</code>.

<h2>Edit ssh configuration on jump server</h2>
Add the following snipit to reduce the privilege of the user:
```
Match User sdouglas-redrocket 
   AllowTcpForwarding yes
   X11Forwarding no
   PermitTunnel no
   GatewayPorts yes
   AllowAgentForwarding no
   PermitOpen localhost:4030
   ForceCommand echo 'This account has no shell access'
```
4030 is the port on the jump host that the reverse tunnel will bind to. In my case, I will just ssh into that port from the jump host, and instantly be connected to the ssh daemon on that machine, wherever that is.
Restart sshd:
```
service sshd restart
```

<h2>Install autossh on new system</h2>
Autossh is a nice little script that creates more resilient ssh connections. If a connection gets disconnected, it will attempt to reconnect until it succeeds.
Installation on your platform will vary, but here are a few examples:
Debian/Ubuntu:
```sudo apt-get install autossh```
Centos/RHEL/Fedora:
```sudo yum install autossh```

<h2>Systemd unit file</h2>
<code>/etc/systemd/system/phonehome.service</code>:
```
[Unit]
Description=phonehome

[Service]
ExecStart=/usr/bin/su sdouglas -c '/usr/bin/autossh -i ~/.ssh/id_rsa_phonehome -M 0 -N -f -p 5022 -R 4030:localhost:22 sdouglas-redrocket@example.com.com'

[Install]
WantedBy=multi-user.target
```
'sdouglas' is the user on the new workstation that I want this to run on. The '-i' flag should point ssh to the private keypair you created. '-p 5022' is the port my jump servers ssh server runs on. '-R 4030:localhost:22' instructs ssh to create a reverse tunnel, where a local port on the new system (22) will be reachable via a remote port on the jump server (4030). 'sdouglas-redrocket' is the user I created for this machine on the jump server. 'example.com' is the host where the jumpserver is reachable.
```
systemctl enable phonehome
```

After you reboot your system, you should be able to ssh into your new system from your jump server:
```
sdouglas@rshell--> ssh -p 4030 sdouglas@localhost
The authenticity of host '[localhost]:4030 ([127.0.0.1]:4030)' can't be established.
ECDSA key fingerprint is SHA256:...
No matching host key fingerprint found in DNS.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '[localhost]:4030' (ECDSA) to the list of known hosts.
sdouglas@localhost's password: 
Last login: Mon Jun 19 22:42:50 2017 from localhost
[sdouglas@redrocket ~]$
```
