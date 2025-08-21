# Using Tailscale as a VPN solution to connect an LKE-E cluster to an on premise subnet.

As part of a PoC, I had to come up with a way to connect an application install in an LKE-E (Linode Kubernetes Engine, Enterprise) to some existing machines which were going to stay on premise. Longer term I expect VPN gateways to be available, but for now I was looking for a solution that would provide this feature with minimum fuss and also offer a number of features such as robust access and user controls to provide the requisite level of security. 

I should also note that I have a very restrictive firewall in place (literally block everything apart from the /32 address of my laptop from which I am connecting to it from). I was curious to see if there was a solution which would work in this case as well.

## Picture

                         ┌──────────────────────────┐
                         │   London Datacentre 🇬🇧   │
                         │     (Kubernetes)🔒 VPN   │
                         ├──────────────────────────┤
                         │Node1:📦 172.237.101.204  │
                         │Node2:📦 172.237.101.106  │
                         │Node3:📦 172.237.101.33   │
                         └──────────────────────────┘
                                   │
                                   │  (WAN / Interconnect)
                                   │
                         ┌──────────────────────────┐
                         │ Amsterdam Datacentre🇳🇱   │
                         │  (Virtual Machines)      │
                         ├──────────────────────────┤
                         │🖥️VM 1: 172.235.175.230 🔒│
                         |             |            |
                         │🖥️VM 2: 172.235.168.115   │
                         └──────────────────────────┘


## Setup

The setup I configured was as follows:

* London

LKE-E cluster which consisted of three nodes which had very restrictive firewalls on the inbound traffic:

| action | addresses.ipv4 | addresses.ipv6 | description | label            | ports   | protocol |
|--------|----------------|----------------|-------------|------------------|---------|----------|
| ACCEPT | 10.0.0.0/8     |                |             | inbound-vpc-tcp  | 1-65535 | TCP      |
| ACCEPT | 10.0.0.0/8     |                |             | inbound-vpc-udp  | 1-65535 | UDP      |
| ACCEPT | 10.0.0.0/8     |                |             | inbound-vpc-icmp |         | ICMP     |


```bash
$ k get nodes -o wide
NAME                           STATUS   ROLES    AGE   VERSION   INTERNAL-IP   EXTERNAL-IP       OS-IMAGE             KERNEL-VERSION       CONTAINER-RUNTIME
lke509158-734007-6kdxs-7t6x9   Ready    <none>   21h   v1.31.8   10.0.0.3      172.237.101.204   Ubuntu 22.04.5 LTS   5.15.0-130-generic   containerd://1.7.24
lke509158-734007-6kdxs-m882m   Ready    <none>   21h   v1.31.8   10.0.0.4      172.237.101.106   Ubuntu 22.04.5 LTS   5.15.0-130-generic   containerd://1.7.24
lke509158-734007-6kdxs-rq6ln   Ready    <none>   21h   v1.31.8   10.0.0.2      172.237.101.33    Ubuntu 22.04.5 LTS   5.15.0-130-generic   containerd://1.7.24
```

* Amsterdam

A pair of linodes, one of which I would add into my VPN solution and another which was going to pretend to be a subnet (well at least a subnet of a single /32 machine to test) which did not have any VPN software installed on it. This should simulate a bunch of legacy servers which we did not want to have to install any new software or make any changes to.

```bash
ubuntu-ams-tailscale-vm       │ nl-ams │ g6-standard-4 │ linode/ubuntu24.04 │ 172.235.175.230

ams-ubuntu-subnet             │ nl-ams │ g6-standard-2 │ linode/ubuntu24.04 │ 172.235.168.115
```

So the machine ending in 230 would be part of the VPN and the machine ending in 115 has zero VPN software added to it.

## Guides I am following

I am using the tailscale sidecar pattern in K8s which you can find the docs for here:

https://tailscale.com/kb/1185/kubernetes

I am also installing onto the ubuntu VM which is part of the quick start here:

https://tailscale.com/kb/1031/install-linux

## Tailscale admin user

Tailscale uses 3rd party services to authenticate users identity, for this I setup a new google user in a stand alone manner so that I could be authenticated as the first user and therefore would become the tailscale administrator. You will need to be a tailscale administrator to do things like create subnet routers, so please make sure you are an Admin in the tailscale capacacity before you carry on with the guide.

## Install client on the VM in Amsterdam

To install tailscale on the VM, the command is 

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```
When this has completed a weblink will be displayed which you can open to register your VM with the tailscale identity solution. In this case I was able to login and see my machine registered once I had authenticated. You can see logs and follow what is happening using `journalctl -u tailscaled` on the machine, for example the login prompt was:

```bash
Aug 20 13:51:48 localhost tailscaled[3035]: control: Generating a new nodekey.
Aug 20 13:51:48 localhost tailscaled[3035]: control: RegisterReq: onode= node=[mTs/r] fup=false nks=false
Aug 20 13:51:48 localhost tailscaled[3035]: control: controlhttp: forcing port 443 dial due to recent noise dial
Aug 20 13:51:48 localhost tailscaled[3035]: control: RegisterReq: got response; nodeKeyExpired=false, machineAuthorized=false; authURL=true
Aug 20 13:51:48 localhost tailscaled[3035]: control: AuthURL is https://login.tailscale.com/a/abc1234
Aug 20 13:51:48 localhost tailscaled[3035]: control: doLogin(regen=false, hasUrl=true)
Aug 20 13:51:48 localhost tailscaled[3035]: control: RegisterReq: onode= node=[mTs/r] fup=true nks=false
Aug 20 13:51:48 localhost tailscaled[3035]: Received auth URL: https://login.tailsc...
```

## Add a subnet router

I followed this guide here https://tailscale.com/kb/1019/subnets for the subnet router install, the basic steps were:

1. Enable IP forwarding on the machine:

```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```

and then add the subnet to be advertised using:

```bash
sudo tailscale set --advertise-routes=172.235.168.115/32
```

Note you will now need to go into the tailscale admin console (or use the api) as an admin to approve the subnet route. You can find this in the machine page of the tailscale console and you will need to click on the machine and it should look like the below:

![Approve Subnet](./approve-subnet.png "Approve Subnet")

## Install the Sidecar Proxy with an application in kubbernetes

I am using an LKE-E cluster, there are a number of patterns which tailscale supports, the one I want to use is the proxy sidecar which will expose an application (deployment) directly to the tailscale wireguard connectivity.

To do this I am going to follow the guide linked above, this broadly boils down to the following steps:

1. Create an auth key.

![Generate Auth Key](./generate-auth-key.png "Generate Auth Key")

This should be setup as per the doc as reusable and ephemeral.

2. Once you have the key, you will need to create a secret and then allow a tailscale service account to access this scecret. The first part is to create the secret:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tailscale-auth
stringData:
  TS_AUTHKEY: tskey-auth-redacted-secret
```
and then once you have created the secret, you can creata a tailscale service account with the permissions needed by tailscale to use it:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: tailscale
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["secrets"]
  # Create can not be restricted to a resource name.
  verbs: ["create"]
- apiGroups: [""] # "" indicates the core API group
  resourceNames: ["tailscale-auth"]
  resources: ["secrets"]
  verbs: ["get", "update", "patch"]
- apiGroups: [""] # "" indicates the core API group
  resources: ["events"]
  verbs: ["get", "create", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: tailscale
subjects:
- kind: ServiceAccount
  name: "tailscale"
roleRef:
  kind: Role
  name: tailscale
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tailscale
```

At this point, we are ready to use tailscale and we will just need to add a sidecar to the application that we want to deply.

## Application to deploy 

There are a couple of patterns, I am using the privileged mode and I have a couple of different applications I am testing with. The first application is a simple hello app that will return the pod name when it is called via http:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-app
  template:
    metadata:
      labels:
        app: hello-app
    spec:
      serviceAccountName: "tailscale"
      containers:
        - name: hello-container
          image: hashicorp/http-echo:latest
          args:
            - "-listen=:8080"
            - "-text=Hello from $(HOSTNAME)"
          ports:
            - containerPort: 8080
          env:
            - name: HOSTNAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
        - name: ts-sidecar
          imagePullPolicy: Always
          image: "ghcr.io/tailscale/tailscale:latest"
          env:
          # Store the state in a k8s secret
            - name: TS_KUBE_SECRET
              value: "tailscale-auth"
            - name: TS_USERSPACE
              value: "false"
            - name: TS_DEBUG_FIREWALL_MODE
              value: auto
            - name: TS_AUTHKEY
              valueFrom:
                secretKeyRef:
                  name: tailscale-auth
                  key: TS_AUTHKEY
                  optional: true
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_UID
              valueFrom:
                fieldRef:
                  fieldPath: metadata.uid
          securityContext:
            privileged: true
```
The second is a net-tools pod which I can exec to and then ping out to other machines from:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-app
  template:
    metadata:
      labels:
        app: hello-app
    spec:
      serviceAccountName: "tailscale"
      containers:
        - name: net-tools-container
          image: ghcr.io/rtsp/docker-net-tools:1.4.35
          command: ["/bin/sh", "-c"]
          args: ["sleep infinity"]
        - name: ts-sidecar
          imagePullPolicy: Always
          image: "ghcr.io/tailscale/tailscale:latest"
          env:
          # Store the state in a k8s secret
            - name: TS_KUBE_SECRET
              value: "tailscale-auth"
            - name: TS_USERSPACE
              value: "false"
            - name: TS_DEBUG_FIREWALL_MODE
              value: auto
            - name: TS_AUTHKEY
              valueFrom:
                secretKeyRef:
                  name: tailscale-auth
                  key: TS_AUTHKEY
                  optional: true
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_UID
              valueFrom:
                fieldRef:
                  fieldPath: metadata.uid
          securityContext:
            privileged: true
```

## Results

When we call the tailscale address of the kubernetes pod from the remote VM running the tailscale install, we can curl the pod address and we see that the VM will happily talk to the pod in the K8s cluster with no issues:

```bash
# curl -k  http://hello-deployment-c8f7dfbdf-mr2lm.tail0c959d.ts.net:8080
Hello from hello-deployment-c8f7dfbdf-mr2lm
```

For the reverse scenario, if I deply the net-tools pod and exec into this, I can ping the remote subnet routed through the tailscale VM:

```bash
$ k exec -ti hello-deployment-5c58cb876d-xw9vd -- bash
Defaulted container "net-tools-container" out of: net-tools-container, ts-sidecar
10:15:49 hello-deployment-5c58cb876d-xw9vd:/# ping 172.235.168.115
PING 172.235.168.115 (172.235.168.115) 56(84) bytes of data.
64 bytes from 172.235.168.115: icmp_seq=1 ttl=44 time=6.28 ms
64 bytes from 172.235.168.115: icmp_seq=2 ttl=44 time=6.35 ms
```

## Conclusion

As part of a PoC I am stopping at this point, in production you should do a lot more now, for example setting up access control lists and configure user roles. Generally the system an users and roles should have a lot more hardening than has been done here. For a PoC these are out of scope of this document.
