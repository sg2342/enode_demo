# Motivation
demonstrate lifecycle (create, start, update, stop) of an erlang OTP node that
runs as bhyve vm with the minimal FreeBSD kernel, user-land and init provided
by [bhyve_enode](https://github.com/sg2342/bhyve_enode).


# Prerequisites

- FreeBSD source in /usr/src
- packages git, doas, rebar3
- erlang runtime with libcrypto statically linked into the erlang crypto driver

# Step By Step Instructions

## 0.0.1 release
```bash
rebar3 as prod tar
```


## bhyve enode archive
```sh
git clone https://github.com/sg2342/bhyve_enode.git ../bhyve_enode
doas ../bhyve_enode/build.sh
```


## image with 0.0.1 release
```sh
truncate -s 300m _build/enode_demo.img
mkdir _build/mnt
MD=$(doas mdconfig -a -t vnode -f _build/enode_demo.img)
doas newfs -n /dev/"$MD"
doas mount -o noatime /dev/"$MD" _build/mnt
doas tar -C _build/mnt/ -xf ../bhyve_enode/_build/bhyve_enode.txz
doas tar -C _build/mnt/root/ -xf _build/prod/rel/enode_demo/enode_demo-0.0.1.tar.gz
doas umount _build/mnt
doas mdconfig -du "$MD"
```


## network setup
```sh
BR=$(doas ifconfig bridge create)
doas ifconfig "$BR" name testbr
doas ifconfig testbr inet 192.168.23.254/24 up
doas ifconfig testbr inet6 fd12:3456:789a:1::ffff/64
doas valectl -h vale23:testbr
```


## bhyve vm
### destroy and bhyveload with network config in kernel environment
```sh
doas bhyvectl --vm=enode_demo --destroy
doas bhyveload -m 512 -e autoboot_delay=0 -e console=comconsole \
 -e hint.uart.0.flags=0x10 -e hint.uart.0.irq=4 -e hint.uart.0.port=0x3F8 \
 -e vfs.root.mountfrom=ufs:/dev/vtbd0 \
 -e minit.ip4.iface.0='vtnet0 192.168.23.1/24' \
 -e minit.ip4.route.0='default 192.168.23.254' \
 -e minit.ip6.iface.0='vtnet0 fd12:3456:789a:1::1/64' \
 -e minit.ip6.route.0='default fd12:3456:789a:1::ffff' \
 -l /boot/userboot_4th.so \
 -d _build/enode_demo.img enode_demo
```
### start vm
```sh
doas bhyve -P -A -H -u  -l com1,stdio -m 512 -c 2 -s 17,lpc \
 -s 16,hostbridge -s 8,virtio-blk,_build/enode_demo.img \
 -s 9,virtio-net,vale23:ed enode_demo
```


## create ssh key and connect
```sh
ssh-keygen -t ed25519 -N '' -f _build/ssh_admin_key
ssh_opts="-i _build/ssh_admin_key -o UserKnownHostsFile=_build/known_hosts -o IdentitiesOnly=yes"
ssh $ssh_opts admin@192.168.23.1
```

## or connect via IPv6
```sh
ssh $ssh_opts admin@fd12:3456:789a:1::1
```

## 0.0.2 release
### get 0.0.2 source and update ssh_maint_ep to 0.1.1
```sh
git checkout 0.0.2
rebar3 clean && rebar3 upgrade ssh_maint_ep
```
### create release archive incl relup
```sh
rebar3 as prod release

rebar3 as prod relup -n enode_demo -v 0.0.2 -u 0.0.1

rebar3 as prod tar -n enode_demo -v 0.0.2
```


## deploy 0.0.2
### transfer release archive to bhyve vm
```sh
echo put _build/prod/rel/enode_demo/enode_demo-0.0.2.tar.gz releases/enode_demo.tar.gz |
sftp -b - $ssh_opts admin@192.168.23.1
```
### update running release to 0.0.2 and make permanent
```sh
ssh $ssh_opts admin@192.168.23.1 'release_handler:unpack_release("enode_demo").'
ssh $ssh_opts admin@192.168.23.1 'release_handler:install_release("0.0.2").'
ssh $ssh_opts admin@192.168.23.1 'release_handler:make_permanent("0.0.2").'
```


## stop vm
```sh
ssh $ssh_opts admin@192.168.23.1 'init:stop(15).'
```
