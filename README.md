vrnetlab - VR Network Lab
-------------------------
Run your favourite virtual routers in docker for convenient labbing,
development and testing.

vrnetlab been developed for the TeraStream project at Deutsche Telekom to
automate CI test related to network provisioning.

It supports:

 * Cisco XRv (not XRv90000)
 * Juniper vMX
 * Nokia VSR

Usage
-----
You have to build the virtual router docker images yourself since the license
agreements of commercial virtual routers do not allow me to distribute the
images. See the README files of the respective virtual router types for more
details.

Let's assume you've built the `xrv` router.

Start two virtual routers:
```
docker run -d --name vr1 --privileged vr-xrv:5.3.3.51U
docker run -d --name vr2 --privileged vr-xrv:5.3.3.51U
```
I'm calling them vr1 and vr2. Note that I'm using XRv 5.3.3.51U - you should
fill in your XRv version in the image tag.

It takes a few minutes for XRv to start but once up you should be able to SSH
into each virtual router. You can get the IP address using docker inspect:
```
root@host# docker inspect --format '{{.NetworkSettings.IPAddress}}' vr1
172.17.0.98
```
Now SSH to that address and login with the default credentials of
vrnetlab/VR-netlab9:
```
root@host# ssh -l vrnetlab 172.17.0.98
The authenticity of host '172.17.0.98 (172.17.0.98)' can't be established.
RSA key fingerprint is e0:61:28:ba:12:77:59:5e:96:cc:58:e2:36:55:00:fa.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '172.17.0.98' (RSA) to the list of known hosts.

IMPORTANT:  READ CAREFULLY
Welcome to the Demo Version of Cisco IOS XRv (the "Software").
The Software is subject to and governed by the terms and conditions
of the End User License Agreement and the Supplemental End User
License Agreement accompanying the product, made available at the
time of your order, or posted on the Cisco website at
www.cisco.com/go/terms (collectively, the "Agreement").
As set forth more fully in the Agreement, use of the Software is
strictly limited to internal use in a non-production environment
solely for demonstration and evaluation purposes.  Downloading,
installing, or using the Software constitutes acceptance of the
Agreement, and you are binding yourself and the business entity
that you represent to the Agreement.  If you do not agree to all
of the terms of the Agreement, then Cisco is unwilling to license
the Software to you and (a) you may not download, install or use the
Software, and (b) you may return the Software as more fully set forth
in the Agreement.


Please login with any configured user/password, or cisco/cisco


vrnetlab@172.17.0.98's password:


RP/0/0/CPU0:ios#show version
Mon Jul 18 09:04:45.261 UTC

Cisco IOS XR Software, Version 5.3.3.51U[Default]
...
```

You can also login via NETCONF:
```
root@kvm-infra:/home/kll/vrnetlab# ssh -l vrnetlab 172.17.0.98 -p 830 -s netconf
vrnetlab@172.17.0.98's password:
<hello xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
 <capabilities>
  <capability>urn:ietf:params:netconf:base:1.1</capability>
  <capability>urn:ietf:params:xml:ns:yang:ietf-netconf-monitoring</capability>
  <capability>urn:ietf:params:netconf:capability:candidate:1.0</capability>
  <capability>urn:ietf:params:netconf:capability:rollback-on-error:1.0</capability>
  <capability>urn:ietf:params:netconf:capability:validate:1.1</capability>
  <capability>urn:ietf:params:netconf:capability:confirmed-commit:1.1</capability>
  <capability>http://cisco.com/ns/yang/Cisco-IOS-XR-aaa-lib-cfg?module=Cisco-IOS-XR-aaa-lib-cfg&amp;revision=2015-08-27</capability>
  <capability>http://cisco.com/ns/yang/Cisco-IOS-XR-aaa-locald-admin-cfg?module=Cisco-IOS-XR-aaa-locald-admin-cfg&amp;revision=2015-08-27</capability>
  <capability>http://cisco.com/ns/yang/Cisco-IOS-XR-aaa-locald-cfg?module=Cisco-IOS-XR-aaa-locald-cfg&amp;revision=2015-08-27</capability>
  <capability>http://cisco.com/ns/yang/Cisco-IOS-XR-aaa-locald-oper?module=Cisco-IOS-XR-aaa-locald-oper&amp;revision=2015-08-27</capability>
  <capability>http://cisco.com/ns/yang/Cisco-IOS-XR-bundlemgr-cfg?module=Cisco-IOS-XR-bundlemgr-cfg&amp;revision=2015-08-27</capability>
...
```

To connect two virtual routers with each other we can use the `tcpbridge`
container. Let's say we want to connect Gi0/0/0/0 of vr1 and vr2 with each
other, we would do:
```
docker run -d --name tcpbridge --link vr1:vr1 --link vr2:vr2 tcpbridge --p2p vr1/1-vr2/1
```

Configure a link network on vr1 and vr2 and you should be able to ping!
```
P/0/0/CPU0:ios(config)#inte GigabitEthernet 0/0/0/0
RP/0/0/CPU0:ios(config-if)#no shutdown
RP/0/0/CPU0:ios(config-if)#ipv4 address 192.168.1.2/24
RP/0/0/CPU0:ios(config-if)#commit
Mon Jul 18 09:13:24.196 UTC
RP/0/0/CPU0:Jul 18 09:13:24.216 : ifmgr[227]: %PKT_INFRA-LINK-3-UPDOWN : Interface GigabitEthernet0/0/0/0, changed state to Down
RP/0/0/CPU0:ios(config-if)#dRP/0/0/CPU0:Jul 18 09:13:24.256 : ifmgr[227]: %PKT_INFRA-LINK-3-UPDOWN : Interface GigabitEthernet0/0/0/0, changed state to Up
o ping 192.168.1.1
Mon Jul 18 09:13:26.896 UTC
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.1.1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/1 ms
```
(obviously I configured the other end too!)

All of the NICs of the virtual routers are exposed via TCP ports by KVM. TCP
port 10001 maps to the first NIC of the virtual router, which in the case of an
XR router is GigabitEthernet 0/0/0/0. By simply connecting two of these TCP
sockets together we can bridge the traffic between those two NICs and this is
exactly what tcpbridge is for. Use the `--p2p` argument to specify the links.
The format is X/Y-Z/N where X is the name of the first router and Y is the port
on that router. Z is the second router and N is the port on the second router.

To set up more than one p2p link, simply add more mapping separated by space
and don't forget to link the virtual routers:
```
docker run -d --name tcpbridge --link vr1:vr1 --link vr2:vr2 --link vr3:vr3 tcpbridge --p2p vr1/1-vr2/1 vr1/2-vr3/1
```

The containers expose port 22 for SSH, port 830 for NETCONF and port 5000 is
mapped to the virtual serial device (use telnet). All the NICs of the virtual
routers are exposed via TCP ports in the range 10001-10099.


Virtual routers
---------------
There are a number of virtual routers available on the market:

 * Cisco XRv
 * Juniper VRR
 * Juniper vMX
 * Nokia VSR

All of the above are released as a qcow2 or vmdk file (which can easily be
converted into qcow2) making them easy to spin up on a Linux machine. Once spun
up there are a few tasks one normally wants to perform:

 * set an IP address on a management interface
 * start SSH / NETCONF daemon (and generate crypto keys)
 * create initial user so we can login

There might be more things to the list but this is the bare minimum which makes
the router remotely reachable and thus we can configure the rest from the
normal provisioning system.

vrnetlab aims to make this process as simple and convenient as possible so that
it may be used both by humans and automated systems to spin up virtual routers.
In addition, there are scripts to help you generate topologies.

The virtual machines are packaged up in docker container. Since we need to
start KVM the docker containers have to be run with `--privileged` which
effectively defeats the security features of docker. Our use of docker is
essentially reduced to being a packaging format but a rather good one at that.

The assignment of a management IP address is handed over to docker, so you can
use whatever docker IPAM plugin you want. Overall the network setup of the
virtual routers are kind of shoe-horned into the world of docker networking.
I'm not sure this is a good idea but it seems to work for now and it was fun
putting it together ;)

It's possible to remotely control a docker engine and tell it to start/stop
containers. It's not entirely uncommon to run the CI system in a VM and letting
it remotely control another docker engine can give us some flexibility in where
the CI runner is executed vs where the virtual routers are running.

libvirt can also be remotely controlled so it could potentially be used to the
same effect. However, unlike libvirt, docker also has a registry concept which
greatly simplifies the distribution of the virtual routers. It's already neatly
packaged up into a container image and now we can pull that image through a
single command. With libvirt we would need to distribute the VM image and
launch scripts as individual files.

The launch script differ from router to router. For example, it's possible to
feed a Cisco XR router a bootup config via a virtual CD-ROM drive so we can use
that to enable SSH/NETCONF and create a user. Nokia VSR however does not, so we
need to tell KVM to emulate a serial device and then have the launch script
access that virtual serial port via telnet to do the initial config.

The intention is to keep the arguments to each virtual router type as similar
as possible so that a test orchestrator or similar need minimal knowledge about
the different router types.

System requirements
-------------------
CPU:

 * sros: 1 core
 * vmx: 5 cores
 * xrv: 1 core

RAM:

 * sros: 4GB
 * vmx: 8GB
 * xrv: 4GB

Disk space depends on what image you are using but here are some rough numbers:

 * sros: ~600MB
 * vmx: ~5GB
 * xrv: ~1.5GB

Docker healtcheck
-----------------
Docker v1.12 includes a healtcheck feature that would be really sweet to use to
tell if the router has been bootstrapped or not.

FUAQ - Frequently or Unfrequently Asked Questions
-------------------------------------------------
##### Q: Why don't you ship pre-built docker images?
A: I don't think Cisco, Juniper or Nokia would allow me to distribute their virtual
   router images and since one of the main points of vrnetlab is to have a self
   contained docker image I don't see any other way than for you to build your
   own image based on vrnetlab but where you get to download the router image
   yourself.

##### Q: Why don't you ship docker images where I can provide the image through a volume?
A: I don't like the concept as it means you have to ship around an extra file.
   If it's a self-contained image then all you have to do is push it to your
   docker registry and then ask a box in your swarm cluster to spin it up!

##### Q: Do you plan to support classic IOS?
A: Hell to the no! ;)

##### Q: How do I connect a vrnetlab router with a normal docker container?
A: I'm not entirely sure. For now you have to live with only communicating
between vrnetlab routers. There's https://github.com/TOGoS/TUN2UDP and I
suppose the same idea could be used to bridge the TCP-socket NICs used by
vrnetlab to a tun device, but if all this should happen inside a docker
container or if we should rely on setting this up on the docker host (using
something similar to pipework) is not entirely clear to me. I'll probably work
on it.

##### Q: How does this relate to GNS3, UNetLab and VIRL?
A: It was a long time since I used GNS3 and I have only briefly looked at
UNetLab and VIRL but from what I know or can see, these are all more targeted
towards interactive labbing. You get a pretty UI and similar whereas vrnetlab
is controlled in a completely programmatic fashion which makes them good at
different things. vrnetlab is superb for CI and programmatic testing where the
others probably target labs run by humans.

Building with GitLab CI
-----------------------
vrnetlab ships with a .gitlab-ci.yml config file so if you happen to be using
GitLab CI you can use this file to let your CI infrastructure build the docker
images and push them to your registry. The CI config and makefiles are written
in a generic manner and the specifics are controlled through environment
variables so to use with GitLab CI you simply need to add three env vars:

 * DOCKER_USER - the username to authenticate to the docker registry with
 * DOCKER_PASSWORD - the password to authenticate to the docker registry with
 * DOCKER_REGISTRY - the URL to the docker registry, like reg.example.com:5000

Next you need to add the actual virtual router images to the git repository.
You can create a separate branch where you add the images as to avoid potential
git merge issues.
```
git checkout -b images
git add xrv/iosxrv-k9-demo-6.0.0.vmdk
git commit -a -m "Added Cisco XRv 6.0.0 image"
git push your-git-repo images
```
Now CI should build the images and push to wherever $DOCKER_REGISTRY points.

When new changes are commited to the upstream repo/master you can just rebase
your branch on top of that:
```
git checkout master
git pull origin master
git checkout images
git rebase master
git push --force your-git-repo images
```
Note that you have to force push since you've rewritten git history.
