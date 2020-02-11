Salt-on-Kube provides an easy way to deploy H/A **Kubernetes Cluster** using Salt.

## Features

- Cloud-provider **agnostic**
- Support **high-available** clusters
- Use the power of **`Saltstack`**
- Made for **`systemd`** based Linux systems
- **Layer 3** networking by default (**`Calico`**)
- **`CoreDNS`** as internal DNS resolver
- Highly **Composable** (CNI, CRI, Add-ons)
- Integrated **add-ons** (Helm, CoreDNS, MetalLB, Dashboard, Nginx-Ingress, ...)
- **RBAC** & **TLS** by default
- Support **IPv6**

## Getting started

### I. Generate CA and TLS certificates using CfSSL

Let's clone the git repo on Salt-master and create CA & certificates on the `k8s-certs/` directory using **`CfSSL`** tools:

```bash
git clone https://github.com/valentin2105/Kubernetes-Saltstack.git /srv/salt
ln -s /srv/salt/pillar /srv/pillar

wget -q --show-progress --https-only --timestamping \
   https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 \
   https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64

chmod +x cfssl_linux-amd64 cfssljson_linux-amd64
sudo mv cfssl_linux-amd64 /usr/local/bin/cfssl
sudo mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
```

##### IMPORTANT Point

Because we generate our own CA and certificates for the cluster, 

You MUST put **every hostnames and IPs of the Kubernetes cluster** (master & workers) in the `certs/kubernetes-csr.json` (**`hosts`** field). 

You can also modify the `certs/*json` files to match your cluster-name / country. (optional)  

You can use **either public or private names**, but they must be registered somewhere (DNS provider, internal DNS server, `/etc/hosts` file) or use **IP records instead of names**.

```bash
cd /srv/salt/k8s-certs
cfssl gencert -initca ca-csr.json | cfssljson -bare ca

# !!!!!!!!!
# Don't forget to edit kubernetes-csr.json before this point !
# !!!!!!!!!

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes

chown salt: /srv/salt/k8s-certs/ -R
```

After that, edit the `pillar/cluster_config.sls` to tweak your future Kubernetes cluster :

```yaml
kubernetes:
  version: v1.16.1
  domain: cluster.local

  master:
    count: 1
    hostname: <ValidHostname-or-IP>
    ipaddr: 10.240.0.10

    etcd:
      version: v3.3.12
    encryption-key: '0Wh+uekJUj3SzaKt+BcHUEJX/9Vo2PLGiCoIsND9GyY='

  pki:
    enable: false
    host: master01.domain.tld
    wildcard: '*.domain.tld'

  worker:
    runtime:
      provider: docker
      docker:
        version: 18.09.9
        data-dir: /dockerFS
    networking:
      cni-version: v0.7.1
      provider: calico
      calico:
        version: v3.9.0
        cni-version: v3.9.0
        calicoctl-version: v3.9.0
        controller-version: 3.9-release
        as-number: 64512
        token: hu0daeHais3a--CHANGEME--hu0daeHais3a
        ipv4:
          range: 192.168.0.0/16
          nat: true
          ip-in-ip: true
        ipv6:
          enable: false
          nat: true
          interface: eth0
          range: fd80:24e2:f998:72d6::/64

  global:
    clusterIP-range: 10.32.0.0/16
    helm-version: v2.14.3
    dashboard-version: v2.0.0-beta4
    coredns-version: 1.6.4 
    admin-token: Haim8kay1rar--CHANGEME--Haim8kay11ra
    kubelet-token: ahT1eipae1wi--CHANGEME--ahT1eipa1e1w
    metallb: 
      enable: false
      version: v0.8.1
      protocol: layer2
      addresses: 10.100.0.0/24
    nginx-ingress:
      enable: false 
      version: 0.26.1
      service-type: LoadBalancer
    cert-manager:
      enable: false
      version: v0.11.0
```

###### Don't forget to change Master's hostname & Tokens  using `pwgen` for example !

If you want to enable IPv6 on pod's side, you need to change `kubernetes.worker.networking.calico.ipv6.enable` to `true`.

### II. Cluster deployment

To deploy your Kubernetes cluster using this formula, you first need to setup your Saltstack master/Minion.

You can use [Salt-Bootstrap](https://docs.saltstack.com/en/stage/topics/tutorials/salt_bootstrap.html) or [Salt-Cloud](https://docs.saltstack.com/en/latest/topics/cloud/) to enhance the process. 

The configuration is done to use the Salt-master as the Kubernetes master. 

You can have them as different nodes if needed but the `post_install/script.sh` require `kubectl` and access to the `pillar` files.

#### The recommended configuration is :

- one or three Kubernetes-master (Salt-master & minion)

- one or more Kubernetes-workers (Salt-minion)

The Minion's roles are matched with `Salt Grains` (kind of inventory), so you need to define theses grains on your servers :

If you want a small cluster, a master can be a worker too. 

```bash
# Kubernetes masters
cat << EOF > /etc/salt/grains
role: k8s-master
EOF

# Kubernetes workers
cat << EOF > /etc/salt/grains
role: k8s-worker
EOF

# Kubernetes master & workers
cat << EOF > /etc/salt/grains
role: 
  - k8s-master
  - k8s-worker
EOF

service salt-minion restart 
```

After that, you can apply your configuration with a (`highstate`) :

```bash
# Apply Kubernetes master configurations :
~ salt -G 'role:k8s-master' state.highstate 

~ kubectl get componentstatuses
NAME                 STATUS    MESSAGE              ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-0               Healthy   {"health": "true"}
etcd-1               Healthy   {"health": "true"}
etcd-2               Healthy   {"health": "true"}

# Apply Kubernetes worker configurations :
~ salt -G 'role:k8s-worker' state.highstate

~ kubectl get nodes
NAME           STATUS   ROLES    AGE     VERSION   OS-IMAGE                       KERNEL-VERSION           CONTAINER-RUNTIME
k8s-worker01   Ready    <none>   3h56m   v1.16.1   Ubuntu 18.04.3 LTS             4.15.0-58-generic        docker://18.9.9
k8s-worker02   Ready    <none>   3h56m   v1.16.1   Ubuntu 18.04.3 LTS             4.15.0-58-generic        docker://18.9.9
k8s-worker03   Ready    <none>   91m     v1.16.1   Debian GNU/Linux 10 (buster)   4.19.0-6-cloud-amd64     docker://18.9.9
k8s-worker04   Ready    <none>   67m     v1.16.1   Fedora 30 (Cloud Edition)      5.2.18-200.fc30.x86_64   docker://18.9.9

# Deploy Calico and Add-ons :
~  /opt/kubernetes/post_install/setup.sh

~# kubectl get pod --all-namespaces
cert-manager           pod/cert-manager-55c44f98f-vmpcm                     1/1     Running     
cert-manager           pod/cert-manager-cainjector-576978ffc8-w7m5j         1/1     Running     
cert-manager           pod/cert-manager-webhook-c67fbc858-tjcvm             1/1     Running     
default                pod/debug-85d7f9799-dtc6c                            1/1     Running
kube-system            pod/calico-kube-controllers-5979855b8-vdpvw          1/1     Running
kube-system            pod/calico-node-h7n58                                1/1     Running
kube-system            pod/calico-node-jl4fc                                1/1     Running
kube-system            pod/calico-node-tv5cq                                1/1     Running
kube-system            pod/calico-node-xxbgh                                1/1     Running
kube-system            pod/coredns-7c7c6c44bf-4lxn4                         1/1     Running
kube-system            pod/coredns-7c7c6c44bf-t9g7v                         1/1     Running
kube-system            pod/tiller-deploy-6966cf57d8-jpf5k                   1/1     Running
kubernetes-dashboard   pod/dashboard-metrics-scraper-566cddb686-mf8xn       1/1     Running
kubernetes-dashboard   pod/kubernetes-dashboard-7b5bf5d559-25cdb            1/1     Running
metallb-system         pod/controller-6bcfdfd677-g9s6f                      1/1     Running
metallb-system         pod/speaker-bmx5p                                    1/1     Running
metallb-system         pod/speaker-g8cqr                                    1/1     Running
metallb-system         pod/speaker-mklzd                                    1/1     Running
metallb-system         pod/speaker-xmhkm                                    1/1     Running
nginx-ingress          pod/nginx-ingress-controller-5dcb7b4488-b68zj        1/1     Running
nginx-ingress          pod/nginx-ingress-controller-5dcb7b4488-n7kwc        1/1     Running
nginx-ingress          pod/nginx-ingress-default-backend-659bd647bd-5l2km   1/1     Running
```

### III. Add nodes afterwards 

If you want add a node on your Kubernetes cluster, just add the new **Hostname**  and *IPs* on `kubernetes-csr.json` and run theses commands to regenerate your cluster certificates :

```bash
cd /srv/salt/k8s-certs

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes

# Reload k8s components on Master and Workers.
salt -G 'role:k8s-master' state.highstate
salt -G 'role:k8s-worker' state.highstate
```

The `highstate` configure automatically new workers (if it match the k8s-worker role in Grains).

- Tested on Debian, Ubuntu and Fedora.
- You can easily upgrade software version on your cluster by changing values in `pillar/cluster_config.sls` and apply a `highstate`.
- This configuration use ECDSA certificates (you can switch to `rsa` in `certs/*.json`).
- You can change IPv4 IPPool, enable IPv6, change IPv6 IPPool, enable IPv6 NAT (for no-public networks), change BGP AS number, Enable IPinIP (to allow routes sharing between subnets).
- If you use `salt-ssh` or `salt-cloud` you can quickly scale new workers.

## Support me on Patreon
Help me out for a couple of :beers:!

https://www.patreon.com/ValentinOuvrard
