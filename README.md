# Add External IP/DNS to SSL certs as SAN to a K8S cluster already deployed by kubeadm

In my case an external IP/DNS was/were not added to the SSL APISERVER certificates when launching kubeadm init as the following flag was not included `--apiserver-cert-extra-sans=IP1,IP2,DNS1,DN2`

[kubeadm init \| Kubernetes](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/)


On one of my previous post about deploying a K8S cluster

[GitHub - IvanGDR/Deploy-K8S-cluster-using-containerd-and-Calico-network-add-on](https://github.com/IvanGDR/Deploy-K8S-cluster-using-containerd-and-Calico-network-add-on)

including the following with the external IP may avoid all this hassle:

```
$ sudo kubeadm init --pod-network-cidr 192.168.0.0/16 --kubernetes-version 1.23.0 --apiserver-cert-extra-sans=IP1,IP2,DNS1,DN2`
```

Therefore, just in case this is needed, I am sharing this cookbook to sort that issue out.


The aim is to  be able to access my remote K8S cluster from my local machine as I am receiving the following error:

```
Unable to connect to the server: x509: certificate is valid for <IPs and/or DNSs list>. , not <IPs and/or DNSs list>
```


The current .kube/config in my local machine moved from  my remote K8S cluster which includes my external or public IP already, is this:

```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LSxx...LSxx==
    server: https://34.290.81.180:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: LSxx...LSxx==
    client-key-data: LSxx..LSxx==
```

The error I am reciving is because the current certificate do not recognise the external IP (34.290.81.180), so this needs to be included for this to work.

#### Previous steps:

Make sure from the local machine we can ping the public IP or public DNS of the K8S cluster so we can proceed to include these in the API server's certificate


Pinging public IP:

```
ping 34.290.81.180 
```
```
PING 34.290.81.180 (34.290.81.180): 56 data bytes
64 bytes from 34.290.81.180: icmp_seq=0 ttl=26 time=161.485 ms
64 bytes from 34.290.81.180 icmp_seq=1 ttl=26 time=155.756 ms
64 bytes from 34.290.81.180: icmp_seq=2 ttl=26 time=155.915 ms
64 bytes from 34.290.81.180: icmp_seq=3 ttl=26 time=163.209 ms
^C
--- 34.290.81.180 ping statistics ---
4 packets transmitted, 4 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 155.756/159.091/163.209/3.313 ms
```


Pinging public hostname:

```
$ ping ec2-34-290-81-180.us-west-2.compute.amazonaws.com
```
```
PING ec2-34-290-81-180.us-west-2.compute.amazonaws.com (34.290.81.180): 56 data bytes
64 bytes from 34.290.81.180: icmp_seq=0 ttl=26 time=155.731 ms
64 bytes from 34.290.81.180: icmp_seq=1 ttl=26 time=155.681 ms
64 bytes from 34.290.81.180: icmp_seq=2 ttl=26 time=156.432 ms
64 bytes from 34.290.81.180: icmp_seq=3 ttl=26 time=156.372 ms
^C
--- ec2-34-290-81-180.us-west-2.compute.amazonaws.com ping statistics ---
4 packets transmitted, 4 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 155.681/156.054/156.432/0.349 ms
```


In order to be as least intrussive as possible  the generated ConfigMap named `kubeadm-config` located in the kube-system namespace is going to be used.

### Updating API server's certificate

As mentioned the cluster was deployed using kubeadm and the certificate just included the internal IP of the cluster  therefore we can use kubeadm to update the API server's certificates to include the extrenal IP as part of the certificate SAN list. 

I am saving  a copy of this config file in my home directory using the name `kubeadm.yaml`.

```
$ kubectl -n kube-system get configmap kubeadm-config -o jsonpath='{.data.ClusterConfiguration}' > kubeadm.yaml
```
>Note: make sure the file you will edit, contains just the lines and indentation as below file. You can also use the more generic command `kubectl -n kube-system get configmap kubeadm-config -o yaml > kubeadm.yaml` to obtain these details, if this is the case, please stick to this recommendation.

```
apiServer:
  extraArgs:
    authorization-mode: Node,RBAC
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.k8s.io
kind: ClusterConfiguration
kubernetesVersion: v1.25.0
networking:
  dnsDomain: cluster.local
  podSubnet: 192.168.0.0/16
  serviceSubnet: 10.96.0.0/12
scheduler: {}
```

Above file does not have any additional SANs listed (IPs and/or DNSs). Proceeed  to add the SANs as needed. In my case I am adding a public IP and a public DNS.

The information for my cluster is this
```
public hostname: ec2-34-290-81-180.us-west-2.compute.amazonaws.com
public ip: 34.290.81.180
```

Eventually the edited and saved file should look like this:
```
$ cat kubeadm.yaml
```
```
apiServer:
  certSANs:
  - "34.290.81.180"
  - "ec2-34-290-81-180.us-west-2.compute.amazonaws.com"
  extraArgs:
    authorization-mode: Node,RBAC
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.k8s.io
kind: ClusterConfiguration
kubernetesVersion: v1.25.0
networking:
  dnsDomain: cluster.local
  podSubnet: 192.168.0.0/16
  serviceSubnet: 10.96.0.0/12
scheduler: {}
```

At this stage,  move (recommended) or remove the existing API server certificate and key because if kubeadm find these files it won't generate new ones..

These are two files, `apiserver.crt` and `apiserver.key` and are located here:
```
$ etc/kubernetes/pki$ ls -la
```
```
-rw-r--r-- 1 root root 1155 Nov  3 17:06 apiserver-etcd-client.crt
-rw------- 1 root root 1675 Nov  3 17:06 apiserver-etcd-client.key
-rw-r--r-- 1 root root 1164 Nov  3 17:06 apiserver-kubelet-client.crt
-rw------- 1 root root 1679 Nov  3 17:06 apiserver-kubelet-client.key
-rw-r--r-- 1 root root 1298 Nov  3 17:06 apiserver.crt
-rw------- 1 root root 1679 Nov  3 17:06 apiserver.key
-rw-r--r-- 1 root root 1099 Aug 22 08:36 ca.crt
-rw------- 1 root root 1675 Aug 22 08:36 ca.key
drwxr-xr-x 2 root root 4096 Aug 22 08:36 etcd
-rw-r--r-- 1 root root 1115 Aug 22 08:36 front-proxy-ca.crt
-rw------- 1 root root 1675 Aug 22 08:36 front-proxy-ca.key
-rw-r--r-- 1 root root 1119 Nov  3 17:06 front-proxy-client.crt
-rw------- 1 root root 1675 Nov  3 17:06 front-proxy-client.key
-rw------- 1 root root 1675 Aug 22 08:36 sa.key
-rw------- 1 root root  451 Aug 22 08:36 sa.pub
```


Move these files as you wish. In my case I am moving these to my home directory:
```
$ sudo mv /etc/kubernetes/pki/apiserver.* ~
```

Execute kubeadm init to generate a new certificate, this time using the kubeadm.yaml configuration file produced on the previous steps:

first,
```
$ sudo su
```
then,
```
root@automaton# kubeadm init phase certs apiserver --config kubeadm.yaml
```
```
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [ip-172-31-23-144 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 172.31.23.144]
```

Finally, we need to restart the API server:

a) Identify APISERVER pod id

```
$ sudo crictl pods

POD ID              CREATED             STATE               NAME                                       NAMESPACE           ATTEMPT             RUNTIME
209a76a9aac39       7 weeks ago         Ready               kube-apiserver-ip-172-31-23-144            kube-system         0                   (default)
d578cfdc632b2       7 weeks ago         Ready               kube-controller-manager-ip-172-31-23-144   kube-system         0                   (default)
bdfb4d176551a       7 weeks ago         Ready               kube-scheduler-ip-172-31-23-144            kube-system         0                   (default)
84becafce7df6       7 weeks ago         Ready               etcd-ip-172-31-23-144                      kube-system         0                   (default)
9634077211684       7 weeks ago         Ready               kube-proxy-79kbg                           kube-system         0                   (default)
612f2feafc5bf       4 months ago        Ready               calico-node-92c7l                          kube-system         0                   (default)
```

In my case the pod id is the firts on the list `209a76a9aac39`

b) stop APISERVER pod

```
$ sudo crictl stopp 209a76a9aac39
```
```
WARN[0000] runtime connect using default endpoints: [unix:///var/run/dockershim.sock unix:///run/containerd/containerd.sock unix:///run/crio/crio.sock unix:///var/run/cri-dockerd.sock]. As the default settings are now deprecated, you should set the endpoint instead.
ERRO[0000] unable to determine runtime API version: rpc error: code = Unavailable desc = connection error: desc = "transport: Error while dialing dial unix /var/run/dockershim.sock: connect: no such file or directory"
Stopped sandbox 209a76a9aac39
```

c) Remove APISERVER  pod 

```
$ sudo crictl rmp 209a76a9aac39
```
```
WARN[0000] runtime connect using default endpoints: [unix:///var/run/dockershim.sock unix:///run/containerd/containerd.sock unix:///run/crio/crio.sock unix:///var/run/cri-dockerd.sock]. As the default settings are now deprecated, you should set the endpoint instead.
ERRO[0000] unable to determine runtime API version: rpc error: code = Unavailable desc = connection error: desc = "transport: Error while dialing dial unix /var/run/dockershim.sock: connect: no such file or directory"
Removed sandbox 209a76a9aac39
```


The Kubelet will automatically restart the container, which will pick up the new certificates. Ignore the warn and error shown.


### Additional steps


##### Verify certificate and check if SAN have been added

```
$ openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text
```
```
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 1731399278931230259 (0x18072a7889103e33)
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN = kubernetes
        Validity
            Not Before: Aug 22 08:36:52 2022 GMT
            Not After : Dec 28 10:21:06 2023 GMT
        Subject: CN = kube-apiserver
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                RSA Public-Key: (2048 bit)
                Modulus:
                    00:b4:2e:54:44:fa:67:5a:84:a4:91:b5:b7:9e:57:
                    ...
                    2f:8e:af:14:e7:38:a0:ce:56:b0:29:33:26:0e:95:
                    eb:2d
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage:
                TLS Web Server Authentication
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Authority Key Identifier:
                keyid:03:A0:EC:8B:11:F7:3E:0B:21:3D:7B:66:E0:05:36:2A:EA:C2:2E:C5

            X509v3 Subject Alternative Name:
                DNS:ec2-34-209-81-108.us-west-2.compute.amazonaws.com, DNS:ip-172-31-23-144, DNS:kubernetes, DNS:kubernetes.default, DNS:kubernetes.default.svc, DNS:kubernetes.default.svc.cluster.local, IP Address:10.96.0.1, IP Address:172.31.23.144, IP Address:34.209.81.108
    Signature Algorithm: sha256WithRSAEncryption
         69:16:23:9e:1f:3f:b6:fd:16:0d:af:b3:cc:32:3f:51:0c:0b:
         ...
         d7:d1:d3:c5:b1:d9:16:fd:0c:30:0f:3d:5b:8f:94:fe:b2:19:
         06:82:eb:e2
-----BEGIN CERTIFICATE-----
MIIDzTCCArWgAwIBAgIIGAcqeIkQPjMwDQYJKoZIhvcNAQELBQAwFTETMBEGA1UE
...
AxMKa3ViZXJuZXRlczAeFw0yMjA4MjIwODM2NTJaFw0yMzEyMjgxMDIxMDZaMBkx
-----END CERTIFICATE-----
```

##### Refresh /.kube/config file for local and remotre kubectl access (in master node):
```
$ sudo mv $HOME/.kube/config $HOME/.kube/backup_config_old
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

##### Move the ./kube/config file to your local machine 

The new config file once moved and copied to  your local machine, needs to be placed in ./kube directory, usually in your home path. Also this needs to be slighly edited to point to the external IP of your kubernetes cluster.

This is the .kube/config file in my k8s controlplane (note it is using the internal IP, which is fine):

```
~/.kube$ cat config
```
```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0...LS0tCg==
    server: https://172.31.23.144:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: LS0...LS0tCg==
    client-key-data: LS0...LS0tCg==
```

This is the same file as above in my local machine but pointing to the public IP address (note the server IP manually changed to `34.290.81.180`):
```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0...LS0tCg==
    server: https://34.290.81.180:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: LS0...LS0tCg==
    client-key-data: LS0...LS0tCg==
```

##### Final Steps

If your local machine with kubectl is just managing one cluster and you are using .kube/config as the configuration file name, then export the KUBECONFIG env variable as follows:
`$ export KUBECONFIG="${HOME}/.kube/config"`

At this stage you should access your remote K8S cluster, otherwise check the contexts available:
`$ kubectl config get-contexts`


As I already have another cluster being managed by my local kubectl and this already is using a configuration file named .kube/config, for this new configuration file I am using this name kube/config_2 so my current files are then:
```
$~/.kube > ls -la
```
```
-rw — — — — 1 ivan ivan 230 Jul 10 20:19 config
-rw — — — — 1 ivan ivan 136 Jul 10 20:20 config_2
```

therefore I am exporting the KUBECONFIG env variable as follows:

`$ export KUBECONFIG="${HOME}/.kube/config:${HOME}/.kube/config_2"`

You may include this in your .bashrc or .zshrc for a permanent change, if so, source the changes and check the contexts again. Happy remote K8S access!!!!!!
