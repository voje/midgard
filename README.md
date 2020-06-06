# Midgard
A raspberry kubernetes home project.   
Kubernetes: k3s.   
Hardware:   
* 2x rpi B
* rpi B+
* cheap tp-link router (provides Wifi access for the RPIs)

## Raw timeline notes

### Preparing the RPIs

I got excited about this project when I saw this video:   
https://www.youtube.com/watch?v=IafVCHkJbtI&t=2039s   
I want a Minecraft server on a raspberry k3s too!   
Also this is a great exercise for managing a cluster.   

Let's get to it...

I've built a simple rack for my 3 RPIs and wired them up to the wireless router.   
First of all, we want static IPs and sensible hostnames:
```
192.168.0.21 loki
192.168.0.22 freyr
192.168.0.23 odin
```
(Avengers ruined Thor for me; Freyr will do.)

Useful for setting up a static IP (`/etc/netplan/`):   
```yml
# network configuration capabilities, write a file
# /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg with the following:
# network: {config: disabled}
network:
    version: 2
    ethernets:
        eth0:
            dhcp4: false
            addresses:
                - 192.168.0.23/24
            gateway4: 192.168.0.1
            nameservers:
                addresses: [8.8.8.8, 1.1.1.1]

```

### Installing k3s
Following Rancher's k3s quickstart:   
https://rancher.com/docs/k3s/latest/en/quick-start/

Installing the server on `odin`.   
```bash
curl -sfL https://get.k3s.io | sh -
k3s kubectl get nodes
WARN[2020-06-06T14:31:24.032996884Z] Unable to read /etc/rancher/k3s/k3s.yaml, please start server with --write-kubeconfig-mode to modify kube config permissions 
error: error loading config file "/etc/rancher/k3s/k3s.yaml": open /etc/rancher/k3s/k3s.yaml: permission denied
```

Stopping k3s as a service, starting manually with -v 3 (log level), to see what's going on.   
```bash
sudo systemctl stop k3s
k3s server -v 3

...

INFO[2020-06-06T14:44:05.264786240Z] k3s is up and running                        
ERRO[2020-06-06T14:44:05.265807271Z] Failed to find memory cgroup, you may need to add "cgroup_memory=1 cgroup_enable=memory" to your linux cmdline (/boot/cmdline.txt on a Raspberry Pi) 
FATA[2020-06-06T14:44:05.266459921Z] failed to find memory cgroup, you may need to add "cgroup_memory=1 cgroup_enable=memory" to your linux cmdline (/boot/cmdline.txt on a Raspberry Pi) 

```
Friendly log output is telling us to enable `cgroup memory` - containers do need that.   

```bash
cat /proc/cgroups 
#subsys_name	hierarchy	num_cgroups	enabled
cpuset	10	1	1
cpu	5	89	1
cpuacct	5	89	1
blkio	2	89	1
memory	0	100	0
devices	9	89	1
freezer	6	2	1
net_cls	3	1	1
perf_event	7	1	1
net_prio	3	1	1
pids	8	98	1
rdma	4	1	1
```
Yup, cgroups memory is not enabled.   
Let's change the settings and reboot.   
```bash
echo "cgroup_memory=1 enable_cgroup_memory" | sudo tee -a /boot/cmdline.txt
sudo reboot now
```
Didn't have the desired effect. tsk...
