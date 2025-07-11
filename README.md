# kubernetes
This is my first try of Talos Linux on an old Laptop

First, I obtained the Metal ISO from Talos Linux and copied it to a USB flash drive. 

There are some Prerequisites that I needed on a PC to help configure the Nodes, as you do not use SSH to log in and configure the machines; you connect via an API. 

To interface with the Talos API, I needed a CLI tool called **talosctl**. I am using a PC running Arch Linux, so I ran:

```bash
sudo pacman -S talosctl
```
I also installed **kubectl** with the command:

```bash
sudo pacman -S kubectl
```
In order for me to move forward, I needed to figure out my Kubernetes Endpoint. Since I am doing this for the first time, I thought I would keep it simple by only creating a single control plane node. That should mean I can use the control plane node directly as the Kubernetes API endpoint.

Going back to the Laptop I saw that it had booted into Talos. The Talos dashboard showed that it was in Maintenance mode and that its IP was 192.168.2.223. 

The default port for the Kubernetes API Server is 6443, and it uses HTTPS, so my endpoint would be: https://192.168.2.223:6443

So, knowing this, I could move ahead to generate the machine configurations for my first cluster. 

I ran the following Command on my Arch machine:

```bash
talosctl gen config firstcluster https://192.168.2.223:6443
```
This then created a few files for me to use:

```bash
generating PKI and tokens
created /Users/kris/controlplane.yaml
created /Users/kris/worker.yaml
created /Users/kris/talosconfig
```
The **.yaml** files are Machine Configs. Since I am not doing a standard multiple-node cluster (only using one laptop in this setup), I will need to enable workers on my control plane node. 

I can do this by editing the **controlplane.yaml**

At the bottom of the file is a commented-out line.

I removed the comment, so the line ended up looking like this:

```bash
allowSchedulingOnControlPlanes: true
```
Saved and exited the file.

Next, I need to see which disks my nodes have by running the following:

```bash
talosctl -n 192.168.2.223 get disks --insecure
```
This returned the following:

```bash
NODE   NAMESPACE   TYPE   ID      VERSION   SIZE     READ ONLY   TRANSPORT   ROTATIONAL   WWID                   MODEL              SERIAL
       runtime     Disk   loop0   2         4.1 kB   true
       runtime     Disk   loop1   2         77 MB    true
       runtime     Disk   sda     2         32 GB    false       usb         true                                USB Flash Disk
       runtime     Disk   sdb     2         1.0 TB   false       sata        true         naa.50000396f3807796   TOSHIBA MQ01ABD1
```
This means the Disk to use would be **sdb**. By default, Talos has the disk set to /dev/sda in the config files, so I need to go in and edit the line to appear like this:

```bash
    install:
        disk: /dev/sdb # The disk used for installations.
```
Okay, now that I have enabled workers on the control plane and updated the disk to use, I can now apply the config with the following:

```bash
talosctl apply-config --nodes 192.168.2.223 --insecure --file controlplane.yaml
```
After running this command on my Arch machine, it brought me back to a prompt, which is a good thing as it will only display any errors to my knowledge. I checked the laptop, and on the dashboard, I saw a lot of things happening in the logs and noted that the stage had changed to Installing. The laptop then rebooted, and the dashboard showed Stage as Booting. Once it was done booting, I moved on to the next step, which was bootstrapping. 

In order to bootstrap the cluster, I ran the following:

```bash
talosctl bootstrap --nodes 192.168.2.223 --endpoints 192.168.2.223 --talosconfig=./talosconfig
```
On the dashboard, the Stage changed from Booting to running, and that ready state is false. There was a lot of action happening in the logs, so I just continued to watch the logs until finally the ready status changed to True.

Next, I needed to download my Kubernetes client configuration. I did this via the following: 

```bash
talosctl kubeconfig ./kubeconfig --nodes 192.168.2.223 --endpoints 192.168.2.223 --talosconfig=./talosconfig
```
I then export the kubeconfig so I can call the API. Did this via the following:

```bash
export KUBECONFIG=./kubeconfig
```
I should now be able to see my nodes. I checked this via the following:

```bash
kubectl get nodes
```

This returned the following: 

```bash
NAME            STATUS   ROLES           AGE   VERSION
talos-fdi-sxc   Ready    control-plane   14m   v1.33.2
```
Now, I checked on all the pods via the following:

```bash
kubectl get pods -A
```

This returned all of the normal pods that should be running on a Control plane node. 

```bash
NAMESPACE     NAME                                    READY   STATUS    RESTARTS      AGE
kube-system   coredns-78d87fb69b-694m8                1/1     Running   0             15m
kube-system   coredns-78d87fb69b-8sl55                1/1     Running   0             15m
kube-system   kube-apiserver-talos-fdi-sxc            1/1     Running   0             15m
kube-system   kube-controller-manager-talos-fdi-sxc   1/1     Running   2 (16m ago)   15m
kube-system   kube-flannel-vmpxd                      1/1     Running   0             15m
kube-system   kube-proxy-nghtv                        1/1     Running   0             15m
kube-system   kube-scheduler-talos-fdi-sxc            1/1     Running   3 (16m ago)   15m
```

Okay, so now I wanted to deploy something that would be on a worker node. I went with nginx to keep it simple. I did this via the following:

```bash
kubectl create deployment nginx-demo --image nginx --replicas 1
```
This returned with:

```bash
Warning: would violate PodSecurity "restricted:latest": allowPrivilegeEscalation != false (container "nginx" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "nginx" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "nginx" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "nginx" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
deployment.apps/nginx-demo created
```
Seems like it was created. I checked the pods again 

```bash
kubectl get pods -A
```
It came back with the following, and I see nginx in the list:

```bash
NAMESPACE     NAME                                    READY   STATUS    RESTARTS      AGE
default       nginx-demo-87cd4cbb7-7jc2g              1/1     Running   0             22s
kube-system   coredns-78d87fb69b-694m8                1/1     Running   0             23m
kube-system   coredns-78d87fb69b-8sl55                1/1     Running   0             23m
kube-system   kube-apiserver-talos-fdi-sxc            1/1     Running   0             22m
kube-system   kube-controller-manager-talos-fdi-sxc   1/1     Running   2 (24m ago)   22m
kube-system   kube-flannel-vmpxd                      1/1     Running   0             23m
kube-system   kube-proxy-nghtv                        1/1     Running   0             23m
kube-system   kube-scheduler-talos-fdi-sxc            1/1     Running   3 (23m ago)   22m
```
I also ran the following to see just the nginx info:

```bash
kubectl get pods -o wide
```
This command, as you can see, shows a bit more info:

```bash
NAME                         READY   STATUS    RESTARTS   AGE   IP           NODE            NOMINATED NODE   READINESS GATES
nginx-demo-87cd4cbb7-7jc2g   1/1     Running   0          40s   10.244.0.4   talos-fdi-sxc   <none>           <none>
```
Now that nginx is running, I will need to expose the port. I do this via the following:

```bash
kubectl expose deployment nginx-demo --type NodePort --port 80
```
It came back with:

```bash
service/nginx-demo exposed
```
I can check if the service is created and confirm the ports being used by the service. I did this via the following:

```bash
kubectl get services
```

It returned the following:

```bash
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP        26m
nginx-demo   NodePort    10.111.164.213   <none>        80:32283/TCP   20s
```

This means I should be able to access it via a web browser via: **http://192.168.2.223:32283/**

This brought me to the welcome to nginx default page, so it worked!!!

Next thing to try: using multiple nodes via bare metal and or via Proxmox.
