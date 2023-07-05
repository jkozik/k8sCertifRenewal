# k8sCertifRenewal
Unable to connect to the server: x509: certificate has expired or is not yet valid

Every year my cluster access is blocked because my certificates expire.  Here's the steps I follow to renew them.

## Verify certificates expired
From the root login on the master node, verify that the certificates are expired.
```
[root@kmaster ~]# kubeadm certs check-expiration
[check-expiration] Reading configuration from the cluster...
[check-expiration] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[check-expiration] Error reading configuration from the Cluster. Falling back to default configuration

CERTIFICATE                EXPIRES                  RESIDUAL TIME   CERTIFICATE AUTHORITY   EXTERNALLY MANAGED
admin.conf                 Jun 18, 2023 22:36 UTC   <invalid>                               no
apiserver                  Jun 18, 2023 22:36 UTC   <invalid>       ca                      no
apiserver-etcd-client      Jun 18, 2023 22:36 UTC   <invalid>       etcd-ca                 no
apiserver-kubelet-client   Jun 18, 2023 22:36 UTC   <invalid>       ca                      no
controller-manager.conf    Jun 18, 2023 22:36 UTC   <invalid>                               no
etcd-healthcheck-client    Jun 18, 2023 22:36 UTC   <invalid>       etcd-ca                 no
etcd-peer                  Jun 18, 2023 22:36 UTC   <invalid>       etcd-ca                 no
etcd-server                Jun 18, 2023 22:36 UTC   <invalid>       etcd-ca                 no
front-proxy-client         Jun 18, 2023 22:36 UTC   <invalid>       front-proxy-ca          no
scheduler.conf             Jun 18, 2023 22:36 UTC   <invalid>                               no

CERTIFICATE AUTHORITY   EXPIRES                  RESIDUAL TIME   EXTERNALLY MANAGED
ca                      May 31, 2031 22:15 UTC   7y              no
etcd-ca                 May 31, 2031 22:15 UTC   7y              no
front-proxy-ca          May 31, 2031 22:15 UTC   7y              no
[root@kmaster ~]# kubectl version
Client Version: version.Info{Major:"1", Minor:"21", GitVersion:"v1.21.1", GitCommit:"5e58841cce77d4bc13713ad2b91fa0d961e69192", GitTreeState:"clean", BuildDate:"2021-05-12T14:18:45Z", GoVersion:"go1.16.4", Compiler:"gc", Platform:"linux/amd64"}
```
## Renew Cerificates
```
[root@kmaster ~]#  kubeadm certs renew all
[renew] Reading configuration from the cluster...
[renew] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[renew] Error reading configuration from the Cluster. Falling back to default configuration

certificate embedded in the kubeconfig file for the admin to use and for kubeadm itself renewed
certificate for serving the Kubernetes API renewed
certificate the apiserver uses to access etcd renewed
certificate for the API server to connect to kubelet renewed
certificate embedded in the kubeconfig file for the controller manager to use renewed
certificate for liveness probes to healthcheck etcd renewed
certificate for etcd nodes to communicate with each other renewed
certificate for serving etcd renewed
certificate for the front proxy client renewed
certificate embedded in the kubeconfig file for the scheduler manager to use renewed

Done renewing certificates. You must restart the kube-apiserver, kube-controller-manager, kube-scheduler and etcd, so that they can use the new certificates.
```
## Restart kubelet and verify renewal
```
[root@kmaster ~]# systemctl restart kubelet
[root@kmaster ~]# kubeadm certs check-expiration
[check-expiration] Reading configuration from the cluster...
[check-expiration] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'

CERTIFICATE                EXPIRES                  RESIDUAL TIME   CERTIFICATE AUTHORITY   EXTERNALLY MANAGED
admin.conf                 Jul 03, 2024 21:07 UTC   364d                                    no
apiserver                  Jul 03, 2024 21:07 UTC   364d            ca                      no
apiserver-etcd-client      Jul 03, 2024 21:07 UTC   364d            etcd-ca                 no
apiserver-kubelet-client   Jul 03, 2024 21:07 UTC   364d            ca                      no
controller-manager.conf    Jul 03, 2024 21:07 UTC   364d                                    no
etcd-healthcheck-client    Jul 03, 2024 21:07 UTC   364d            etcd-ca                 no
etcd-peer                  Jul 03, 2024 21:07 UTC   364d            etcd-ca                 no
etcd-server                Jul 03, 2024 21:07 UTC   364d            etcd-ca                 no
front-proxy-client         Jul 03, 2024 21:07 UTC   364d            front-proxy-ca          no
scheduler.conf             Jul 03, 2024 21:07 UTC   364d                                    no

CERTIFICATE AUTHORITY   EXPIRES                  RESIDUAL TIME   EXTERNALLY MANAGED
ca                      May 31, 2031 22:15 UTC   7y              no
etcd-ca                 May 31, 2031 22:15 UTC   7y              no
front-proxy-ca          May 31, 2031 22:15 UTC   7y              no
```
## install new admin.conf 
To run the kubectl command, the new admin.conf file needs to be distributed.  First to the root login on the master node... 
```
[root@kmaster ~]# cd /etc/kubernetes/
[root@kmaster kubernetes]# ls
admin.conf  controller-manager.conf  kubelet.conf  manifests  pki  scheduler.conf
[root@kmaster kubernetes]# vi admin.conf
[root@kmaster kubernetes]# pwd
/etc/kubernetes
[root@kmaster kubernetes]# cp admin.conf /root/.kube/config^C
[root@kmaster kubernetes]# mkdir /root/.kube/config
[root@kmaster kubernetes]# cp admin.conf /root/.kube/c
cache/  config/
[root@kmaster kubernetes]# cp admin.conf /root/.kube/config
[root@kmaster kubernetes]# kubectl get nodes
error: error loading config file "/root/.kube/config": read /root/.kube/config: is a directory
[root@kmaster kubernetes]# cd /root/.kube
[root@kmaster .kube]# ls
cache  config
[root@kmaster .kube]# cd config
[root@kmaster config]# ls
admin.conf
[root@kmaster config]# mv admin.conf ..
[root@kmaster config]# cd ..
[root@kmaster .kube]# ls
admin.conf  cache  config
[root@kmaster .kube]# rm config
rm: cannot remove ‘config’: Is a directory
[root@kmaster .kube]# rmdir config
[root@kmaster .kube]# cp admin.conf config
[root@kmaster .kube]# kubectl get nodes
NAME       STATUS   ROLES                  AGE     VERSION
kmaster    Ready    control-plane,master   2y31d   v1.21.1
kworker1   Ready    <none>                 2y31d   v1.22.2
kworker2   Ready    <none>                 2y30d   v1.21.1
kworker3   Ready    <none>                 709d    v1.21.3
```
... and then to my client login off the cluster.
```
[root@kmaster .kube]# scp admin.conf 192.168.101.152:/home/jkozik
root@192.168.101.152's password:
admin.conf                                                                            100% 5595   108.3KB/s   00:00
[root@kmaster .kube]# exit
logout
[jkozik@kmaster ~]$ exit
logout
Connection to 192.168.100.172 closed.
[jkozik@dell2 ~]$ ls
admin.conf
admin.conf062122
alpine-standard-3.11.6-x86_64.iso
bck
bin
CentOS-7-x86_64-Minimal-1804.iso
Desktop
desktop.ovpn
Dockerfile
Documents
Downloads
example-ingress.yaml
gitpat
grafana.values
index.html
index.html.1
jkozik@192.168.100.121
k8sSkScw.com
kernel-devel-3.10.0-1160.24.1.el7.x86_64.rpm
kernel-devel-3.10.0-1160.25.1.el7.x86_64.rpm
kernel-headers-3.10.0-1160.25.1.el7.x86_64.rpm
kernel-plus-headers-3.10.0-1160.25.1.el7.centos.plus.x86_64.rpm
kernel-plus-headers-3.10.0-1160.25.1.el7.centos.plus.x86_64.rpm.1
local-minikube-docker
logs
metallb
Music
nginx
nwcom-ingress-service.yaml
nwcom.yaml
old
Oracle_VM_VirtualBox_Extension_Pack-5.2.16.vbox-extpack
Pictures
prometheus-example-app.yaml
prometheus-grafana-ingress.yaml
prometheus.values
Public
pvc.yaml
skaffold
Templates
thinclient_drives
vagrant-home
VBoxGuestAdditions_5.2.16.iso
Videos
VirtualBox VMs
x011222.tmp
x072121.tmp
x.gz
x.tmp
xx.tmp
y.tmp
Zabbix5SetupNotes
[jkozik@dell2 ~]$ ls ad*
admin.conf  admin.conf062122
[jkozik@dell2 ~]$ ls .kube
cache  config  http-cache  old
[jkozik@dell2 ~]$ cd .
[jkozik@dell2 ~]$ cd .kube
[jkozik@dell2 .kube]$ mv config admin.conf070423
[jkozik@dell2 .kube]$ mv ../admin.conf config
[jkozik@dell2 .kube]$ cd
[jkozik@dell2 ~]$ kubectl get nodes
error: error loading config file "/home/jkozik/.kube/config": open /home/jkozik/.kube/config: permission denied
[jkozik@dell2 ~]$ cd -
/home/jkozik/.kube
[jkozik@dell2 .kube]$ ls -lasth
total 40K
   0 drwxr-xr-x.  5 jkozik jkozik  110 Jul  4 16:17 .
8.0K drwxr-xr-x. 44 jkozik jkozik 4.0K Jul  4 16:17 ..
8.0K -rw-------.  1 root   root   5.5K Jul  4 16:16 config
 16K drwxr-x---.  3 jkozik jkozik  12K Jul  4 13:17 http-cache
8.0K -rw-------.  1 jkozik jkozik 5.5K Jun 21  2022 admin.conf070423
   0 drwxrwxr-x.  2 jkozik jkozik   28 Jun 21  2022 old
   0 drwxr-x---.  4 jkozik jkozik   47 Aug 25  2021 cache
[jkozik@dell2 .kube]$ su -
Password:
Last login: Sun Feb  5 15:31:46 CST 2023 on pts/6
[root@dell2 ~]# cd ~jkozik/.kube
[root@dell2 .kube]# chown jkozik config
[root@dell2 .kube]# chgrp jkozik config
[root@dell2 .kube]# ls -lasth
total 40K
   0 drwxr-xr-x.  5 jkozik jkozik  110 Jul  4 16:17 .
8.0K drwxr-xr-x. 44 jkozik jkozik 4.0K Jul  4 16:17 ..
8.0K -rw-------.  1 jkozik jkozik 5.5K Jul  4 16:16 config
 16K drwxr-x---.  3 jkozik jkozik  12K Jul  4 13:17 http-cache
8.0K -rw-------.  1 jkozik jkozik 5.5K Jun 21  2022 admin.conf070423
   0 drwxrwxr-x.  2 jkozik jkozik   28 Jun 21  2022 old
   0 drwxr-x---.  4 jkozik jkozik   47 Aug 25  2021 cache
[root@dell2 .kube]# exit
logout
[jkozik@dell2 .kube]$ kubectl get nodes
NAME       STATUS   ROLES                  AGE     VERSION
kmaster    Ready    control-plane,master   2y31d   v1.21.1
kworker1   Ready    <none>                 2y31d   v1.22.2
kworker2   Ready    <none>                 2y30d   v1.21.1
kworker3   Ready    <none>                 709d    v1.21.3
[jkozik@dell2 .kube]$ kubectl top nodes
NAME       CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
kmaster    329m         16%    1329Mi          76%
kworker1   130m         13%    1248Mi          71%
kworker2   125m         12%    878Mi           50%
kworker3   238m         7%     2397Mi          64%
```
# Links
K8S: Unable to connect to the server: x509: certificate has expired or is not yet valid
https://plainenglish.io/blog/k8s-unable-to-connect-to-the-server-x509-certificate-has-expired-or-is-not-yet-valid-bb3c3429be04

Kubernetes: expired certificate
https://stackoverflow.com/questions/49885636/kubernetes-expired-certificate


