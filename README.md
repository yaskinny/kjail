# kjail
---
This script is usefull when You are controlling K8s from a bastion host. It will create a read-only jailed environment for each user and a user in your kubernetes.  


```BASH
$ sudo kjail -h
kjail creates new user for kubernetes which is jailed in SSH access to just run kubectl commands.
if You would like, You can rename this script to kubectl-kjail and use it as a kubernetes plugin like
`kubectl kjail ...`


Options:
-u , --username , $KJAIL_USERNAME <username> (REQUIRED)
  username to create jail and kubeconfig for it.
  This name will be used as username in kubernetes certificate.(certificate common name/CN)

-p, --password , $KJAIL_PASSWORD <password>
  Password to set for new user

-b , --base , $KJAIL_BASEDIR <full path> (REQUIRED)
  Full path to a directory which does not exist. For each user this must be different.

--no-read-only , $KJAIL_NO_READ_ONLY
  If It's not set, a read-only jail environment will be created for user.(It is a read-only mount of writable directory)
  Read-only environment will make some problems for kubectl which is in client-side and is not important, You can ignore them.

-g, --groups , $KJAIL_GROUPS <groups>
  Comma seperated names of groups to set for new user.
  These names will be used in kubernetes certificate.(certificate organization/O)

--ssh-pubkey , --pubkey , $KJAIL_SSH_PUBKEY <path>
  Path to a public key to add in authorized keys for new user.

--output-dir , $KJAIL_OUTPUT <path>
  Path to a directory to save output files.
  Directory shouldn't exists.

--ssh-config , $KJAIL_SSH_CONFIG
  If this command is used, SSH config will be written to `/etc/ssh/sshd_config.d/<username>.conf`.
  Otherwise config will be written to `<output dir>/sshd.conf`

-h , --help
  shows this message


Examples:
kjail --username yaser --base /jails/yaser --password pswd123 --ssh-config

env KJAIL_USERNAME=yaser KJAIL_BASEDIR=/jails/yaser KJAIL_PASSWORD='pswd123' KJAIL_SSH_CONFIG=yes kjail
```

---
Example:
```BASH

$ (ls /jails || echo does not exist) 2>/dev/null
does not exist

$ kjail --username kjail-yaskinny --password QQQ333 --base /jails/yaskinny
[INFO] public key is not set. No public key will be added to user authorized keys.
[INFO] no kubernetes group is set for new user
[INFO] output dir is not set
[INFO] ssh config is not set. output will be written to file /root/kjail-yaskinny-kjail/ssh.conf
[INFO] read-only jail will be created
[INFO] csr for user kjail-yaskinny found in kubernetes
SSH config for new user is in /root/kjail-yaskinny-kjail/ssh.conf file.
Don't forget to reload SSH service to pickup new changes after moving SSH config to appropriate place.
All related files to new user are written to files inside of /root/kjail-yaskinny-kjail directory.
Don't leave these files on file system.

$ tree /root/kjail-yaskinny-kjail
/root/kjail-yaskinny-kjail
├── config              # Kubeconfig file for new user
├── csr.yaml            # CSR kubernetes object
├── kjail-yaskinny.crt  # User certificate signed by kubernetes CA
├── kjail-yaskinny.csr  # CSR we made from user's private key
├── kjail-yaskinny.key  # User's new private key
└── ssh.conf            # SSH config for this user

$ cat /root/kjail-yaskinny-kjail/ssh.conf
Match User kjail-yaskinny
  ChrootDirectory /jails/yaskinny-ro
  # Don't forget to set authentication options
  # AuthenticationMethods THIS
  # PasswordAuthentication THIS
  # PubkeyAuthentication THIS
  # And other options like TCP forwarding, X11 forwarding and ...

$ ls -1 /jails/
/jails/yaskinny       # writable directory
/jails/yaskinny-ro    # read-only
$ tree -L 2 /jails/yaskinny
/jails/yaskinny
├── bin -> usr/bin
├── dev
│   ├── null
│   ├── random
│   ├── tty
│   ├── urandom
│   └── zero
├── etc
│   ├── group
│   ├── nsswitch.conf
│   ├── passwd
│   └── terminfo
├── home
│   └── kjail-yaskinny
├── lib
│   ├── libnsl-2.31.so
│   ├── libnsl.so.1
│   ├── libnss_compat-2.31.so
│   ├── libnss_compat.so.2
│   ├── libnss_dns-2.31.so
│   ├── libnss_dns.so.2
│   ├── libnss_files-2.31.so
│   ├── libnss_files.so.2
│   ├── libnss_hesiod-2.31.so
│   ├── libnss_hesiod.so.2
│   ├── libnss_nis-2.31.so
│   ├── libnss_nis.so.2
│   ├── libnss_nisplus-2.31.so
│   ├── libnss_nisplus.so.2
│   ├── libnss_systemd.so.2
│   ├── terminfo
│   └── x86_64-linux-gnu
├── lib64
│   └── ld-linux-x86-64.so.2
├── sbin -> usr/sbin
└── usr
    ├── bin
    ├── sbin
    └── share

$ cat /root/kjail-yaskinny-kjail/ssh.conf | tee -a /etc/ssh/sshd_config
Match User kjail-yaskinny
  ChrootDirectory /jails/yaskinny-ro
  # Don't forget to set authentication options
  # AuthenticationMethods THIS
  # PasswordAuthentication THIS
  # PubkeyAuthentication THIS
  # And other options like TCP forwarding, X11 forwarding and ...

$ systemctl reload ssh
$ ssh kjail-yaskinny@bastion

# error reason is RO file-system.
# since I didn't make any kind of RBAC for new user, I can not get list of nodes
kjail-yaskinny@bastion:~$ kubectl get no
I0301 13:57:21.470127   35448 request.go:665] Waited for 1.138593145s due to client-side throttling, not priority and fairness, request: GET:https://192.168.1.51:6443/apis/storage.k8s.io/v1beta1?timeout=32s
Error from server (Forbidden): nodes is forbidden: User "kjail-yaskinny" cannot list resource "nodes" in API group "" at the cluster scope
```



# TODOs
- improve jailed shell
- add option for user to add cmds other than just bash and kubectl to jailed user
- improve error handling 
