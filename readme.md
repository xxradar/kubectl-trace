# A quick braindump on how I got started w/ kubectl-trace (bpftrace)
For all information head out to https://github.com/iovisor/bpftrace and https://github.com/iovisor/kubectl-trace

## Get your hands on a K8S cluster
I'm using a self-managed kubeadm cluster using Ubuntu and the latest version of K8S.

### Verify cluster access
Always good practice to verify you have access to the correct cluster and your nodes are in a `ready` state
```
export KUBECONFIG=$PWD/kubeconfig.yaml 
kubectl get nodes
```

## Installing kubectl-trace
### Install 'krew'
Check out this page https://krew.sigs.k8s.io/docs/user-guide/setup/install/ to find more information in the 'krew' plugin for 'kubectl'
```
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/krew.tar.gz" &&
  tar zxvf krew.tar.gz &&
  KREW=./krew-"${OS}_${ARCH}" &&
  "$KREW" install krew
)

export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"

kubectl krew
```
### Installing kubectl-trace
```
kubectl krew install trace
```

## Setup a demo app
```
git clone https://github.com/xxradar/app_routable_demo.git
cd ./app_routable_demo
./setup.sh
watch kubectl get po -n app-routable-demo
```

Set a helper variable for easy cli commands.<br>
Make sure you select a worker node to monitor the sample app ...
```
$ kubectl get po -n app-routable-demo -o wide
...
```
```
ex. export NODE=demo-poolebpftracedemo-8ryo2
```

## Capturing pods connections
### Clone the bpftrace repo
To get some bpfprobe examples, clone this repo
```
git clone https://github.com/iovisor/bpftrace.git
```

Setup a bpfprobe (ex. monitoring TCP connect events)
```
$ kubectl trace run $NODE -f ./bpftrace/tools/tcpconnect.bt
trace 2b25bf44-ae69-11eb-89a6-061a9c98df32 created
```
```
$ kubectl get po
NAME                                                       READY   STATUS      RESTARTS   AGE
kubectl-trace-2b25bf44-ae69-11eb-89a6-061a9c98df32-pczj7   1/1     Running   0          41s
```
```
$ kubectl attach -it kubectl-trace-2b25bf44-ae69-11eb-89a6-061a9c98df32-pczj7
If you don't see a command prompt, try pressing enter.
13:11:15 7962     kubelet          10.11.2.126                             56608  10.11.2.126                             6443
13:11:16 7962     kubelet          10.11.2.126                             56610  10.11.2.126                             6443
13:11:17 7962     kubelet          10.11.2.126                             56612  10.11.2.126                             6443
13:11:18 7962     kubelet          127.0.0.1                               51342  127.0.0.1                               10259
13:11:18 7962     kubelet          10.11.2.126                             56616  10.11.2.126                             6443
13:11:19 7962     kubelet          10.11.2.126                             56618  10.11.2.126                             6443
13:11:20 7962     kubelet          10.11.2.126                             56620  10.11.2.126                             6443
13:11:21 7962     kubelet          127.0.0.1                               35742  127.0.0.1                               10257
13:11:21 7962     kubelet          10.11.2.126                             56624  10.11.2.126                             6443
13:11:22 7962     kubelet          127.0.0.1                               41082  127.0.0.1                               2381
13:11:22 7962     kubelet          10.11.2.126                             56628  10.11.2.126                             6443
13:11:23 7962     kubelet          10.11.2.126                             56630  10.11.2.126                             6443
...
```
You should see a list of connections being initiated by the pods on the K8S node.<br>
Let's run this complicated command to run some commands on the node we're running the probe.<br>
Open a seperate shell (set the `NODE` variable)
```
$ kubectl run -it --rm --image xxradar/hackon --overrides='{"apiVersion": "v1", "spec": {"nodeSelector": { "kubernetes.io/hostname": "'$NODE'" }}}' demo -- bash
curl www.radarhack.com
...
nc www.radarhack.com 80
...
```
analyse the ouptut of the probe ...
```
...
13:57:24 6395     kubelet          10.11.2.83                              48882  192.168.19.130                          8080
13:57:24 6395     kubelet          10.11.2.83                              52170  192.168.19.130                          8181
13:57:24 13173    curl             192.168.19.148                          36206  198.199.124.250                         80
13:57:25 20108    nginx            192.168.19.133                          50962  10.100.232.133                          80
...
13:57:41 20108    nginx            192.168.19.133                          46684  10.109.222.35                           80
13:57:41 13442    nc               192.168.19.148                          36730  198.199.124.250                         80
13:57:41 20108    nginx            192.168.19.133                          51486  10.100.232.133                          80

```
