<!-- omit in toc -->
# Demo 05: Kubernetes work

<!-- omit in toc -->
## Table of Contents

- [Preparation](#preparation)
- [Case 01: Simple command](#case-01-simple-command)
- [Case 02: Parameters and payloads](#case-02-parameters-and-payloads)
- [Case 03: Customized pod](#case-03-customized-pod)
- [Case 04: Use "InCluster" authmethod](#case-04-use-incluster-authmethod)

## Preparation

```bash
cd 05_kubernetes
cp /etc/rancher/k3s/k3s.yaml .kube/config
sed -i 's/127\.0\.0\.1/192.168.0.219/g' .kube/config
docker compose up -d
```

```bash
kubectl apply -f manifests/namespace.yml
```

## Case 01: Simple command

```bash
[root@node01 /]# receptorctl work submit \
  --node executor01 \
  --no-payload \
  echo-sleep
Result: Job Started
Unit ID: jBLBH6pV
```

```bash
$ kubectl -n receptor get pod
NAME               READY   STATUS    RESTARTS   AGE
echo-sleep-mqnjl   1/1     Running   0          13s

$ kubectl -n receptor get pod echo-sleep-mqnjl -o yaml
apiVersion: v1
kind: Pod
metadata:
  ...
  generateName: echo-sleep-
  name: echo-sleep-mqnjl
  namespace: receptor
  ...
spec:
  containers:
  - args:
    - -c
    - 'for i in $(seq 1 12); do echo ${i}: $(date +%T); sleep 5; done'
    command:
    - bash
    image: quay.io/centos/centos:stream8
    imagePullPolicy: IfNotPresent
    name: worker
    ...
```

```bash
[root@node01 /]# ls -l /tmp/receptor/controller01/jBLBH6pV/
total 8
-rw-------. 1 root root 317 Dec  4 19:46 status
-rw-------. 1 root root   0 Dec  4 19:46 status.lock
-rw-------. 1 root root   0 Dec  4 19:45 stdin
-rw-------. 1 root root 147 Dec  4 19:46 stdout

[root@node01 /]# cat /tmp/receptor/controller01/jBLBH6pV/status
{
  "State": 2,
  "Detail": "Finished",
  "StdoutSize": 147,
  ...
  "ExtraData": {
    ...
    "RemoteUnitID": "zfr7GxnR",
    ...
  }
}
```

```bash
[root@node03 /]# ls -l /tmp/receptor/executor01/zfr7GxnR/
total 8
-rw-------. 1 root root 3292 Dec  4 19:46 status
-rw-------. 1 root root    0 Dec  4 19:46 status.lock
-rw-------. 1 root root    0 Dec  4 19:45 stdin
-rw-------. 1 root root  147 Dec  4 19:46 stdout

[root@node03 /]# cat /tmp/receptor/executor01/zfr7GxnR/status
{
  "State": 2,
  "Detail": "Finished",
  "StdoutSize": 147,
  ...
  "ExtraData": {
    "Image": "quay.io/centos/centos:stream8",
    "Command": "bash",
    "Params": "-c 'for i in $(seq 1 12); do echo ${i}: $(date +%T); sleep 5; done'",
    "KubeNamespace": "receptor",
    "KubeConfig": "apiVersion: v1\n...",
    "KubePod": "",
    "PodName": "echo-sleep-mqnjl"
  }
}
```

```bash
[root@node01 /]# receptorctl work list --unit_id jBLBH6pV
{
    "jBLBH6pV": {
        "Detail": "Finished",
        "ExtraData": {
            ...
        },
        "State": 2,
        "StateName": "Succeeded",
        "StdoutSize": 147,
        ...
    }
}

[root@node01 /]# receptorctl work results jBLBH6pV
1: 19:45:18
2: 19:45:23
...
11: 19:46:08
12: 19:46:13
```

```bash
[root@node01 /]# receptorctl work release --all
Released:
(jBLBH6pV, released)
```

## Case 02: Parameters and payloads

```bash
[root@node01 /]# receptorctl work submit \
  --node executor01 \
  --rm \
  --follow \
  --no-payload \
  --param "kube_params=/etc/hostname /etc/os-release" \
  cat-files
/etc/hostname:1:cat-files-ld5zs
/etc/os-release:1:NAME="CentOS Stream"
/etc/os-release:2:VERSION="8"
/etc/os-release:3:ID="centos"
/etc/os-release:4:ID_LIKE="rhel fedora"
/etc/os-release:5:VERSION_ID="8"
/etc/os-release:6:PLATFORM_ID="platform:el8"
/etc/os-release:7:PRETTY_NAME="CentOS Stream 8"
/etc/os-release:8:ANSI_COLOR="0;31"
/etc/os-release:9:CPE_NAME="cpe:/o:centos:centos:8"
/etc/os-release:10:HOME_URL="https://centos.org/"
/etc/os-release:11:BUG_REPORT_URL="https://bugzilla.redhat.com/"
/etc/os-release:12:REDHAT_SUPPORT_PRODUCT="Red Hat Enterprise Linux 8"
/etc/os-release:13:REDHAT_SUPPORT_PRODUCT_VERSION="CentOS Stream"
(y6UCfqYt, released)
```

```bash
$ kubectl -n receptor get pod
NAME              READY   STATUS      RESTARTS   AGE
cat-files-ld5zs   0/1     Completed   0          1s

$ kubectl -n receptor get pod -o yaml
...
- apiVersion: v1
  kind: Pod
  metadata:
    ...
    generateName: cat-files-
    name: cat-files-ld5zs
    namespace: receptor
    ...
  spec:
    containers:
    - args:
      - -n
      - -H
      - ""
      - /etc/hostname
      - /etc/os-release
      command:
      - grep
      image: quay.io/centos/centos:stream8
      imagePullPolicy: IfNotPresent
      name: worker
      ...
```

```bash
[root@node01 /]# echo "Hello Receptor!" > /tmp/input.txt
[root@node01 /]# receptorctl work submit \
  --node executor01 \
  --rm \
  --follow \
  --payload /tmp/input.txt \
  echo-reply
Reply from echo-reply-64l6z: HELLO RECEPTOR!
(mW6CH1D9, released)

[root@node01 /]# echo "Hello Receptor!" | receptorctl work submit \
  --node executor01 \
  --rm \
  --follow \
  --payload - \
  echo-reply
Reply from echo-reply-jf4bd: HELLO RECEPTOR!
(vmjsnjSd, released)

[root@node01 /]# receptorctl work submit \
  --node executor01 \
  --rm \
  --follow \
  --payload-literal "Hello Receptor!" \
  echo-reply
Reply from echo-reply-4jzxn: HELLO RECEPTOR!
(MUdlZNqN, released)
```

```bash
[root@node01 /]# receptorctl work submit \
  --node executor01 \
  --rm \
  --follow \
  --no-payload \
  --param kube_image=ubuntu:22.04 \
  --param "kube_params=/etc/hostname /etc/os-release" \
  cat-files
/etc/hostname:1:cat-files-tqwj9
/etc/os-release:1:PRETTY_NAME="Ubuntu 22.04.1 LTS"
/etc/os-release:2:NAME="Ubuntu"
/etc/os-release:3:VERSION_ID="22.04"
/etc/os-release:4:VERSION="22.04.1 LTS (Jammy Jellyfish)"
/etc/os-release:5:VERSION_CODENAME=jammy
/etc/os-release:6:ID=ubuntu
/etc/os-release:7:ID_LIKE=debian
/etc/os-release:8:HOME_URL="https://www.ubuntu.com/"
/etc/os-release:9:SUPPORT_URL="https://help.ubuntu.com/"
/etc/os-release:10:BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
/etc/os-release:11:PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
/etc/os-release:12:UBUNTU_CODENAME=jammy
(jplGpy12, released)
```

```bash
$ kubectl -n receptor get pod
NAME              READY   STATUS      RESTARTS   AGE
cat-files-tqwj9   0/1     Completed   0          1s

$ kubectl -n receptor get pod -o yaml
...
- apiVersion: v1
  kind: Pod
  metadata:
    ...
    generateName: cat-files-
    name: cat-files-vkgzj
    namespace: receptor
    ...
  spec:
    containers:
    - args:
      - -n
      - -H
      - ""
      - /etc/hostname
      - /etc/os-release
      command:
      - grep
      image: ubuntu:22.04
      imagePullPolicy: IfNotPresent
      name: worker
      ...
```

## Case 03: Customized pod

```bash
[root@node01 /]# receptorctl work submit \
  --node executor01 \
  --rm \
  --follow \
  --no-payload \
  custom-predefined-pod
/etc/hostname:1:custom-predefined-pod-rcptp
/etc/os-release:1:PRETTY_NAME="Ubuntu 22.04.1 LTS"
/etc/os-release:2:NAME="Ubuntu"
/etc/os-release:3:VERSION_ID="22.04"
/etc/os-release:4:VERSION="22.04.1 LTS (Jammy Jellyfish)"
/etc/os-release:5:VERSION_CODENAME=jammy
/etc/os-release:6:ID=ubuntu
/etc/os-release:7:ID_LIKE=debian
/etc/os-release:8:HOME_URL="https://www.ubuntu.com/"
/etc/os-release:9:SUPPORT_URL="https://help.ubuntu.com/"
/etc/os-release:10:BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
/etc/os-release:11:PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
/etc/os-release:12:UBUNTU_CODENAME=jammy
(MM4c8e9s, released)
```

```bash
$ kubectl -n receptor get pod -o yaml
...
- apiVersion: v1
  kind: Pod
  metadata:
    ...
    labels:
      app: hello-receptor
    name: custom-predefined-pod-rcptp
    namespace: receptor
    ...
  spec:
    containers:
    - args:
      - -n
      - -H
      - ""
      - /etc/hostname
      - /etc/os-release
      command:
      - grep
      image: ubuntu:22.04
      imagePullPolicy: IfNotPresent
      name: worker
      resources:
        requests:
          cpu: 10m
      ...
    initContainers:
    - args:
      - "5"
      command:
      - sleep
      image: quay.io/centos/centos:stream8
      imagePullPolicy: IfNotPresent
      name: init-worker
      ...
```

```bash
[root@node01 /]# receptorctl work submit \
  --node controller01 \
  --rm \
  --follow \
  --no-payload \
  --param secret_kube_config=@/tmp/.kube/config \
  --param secret_kube_pod=@/tmp/pod.yml \
  custom-runtime-pod
/etc/hostname:1:custom-runtime-pod-znh8k
/etc/os-release:1:PRETTY_NAME="Ubuntu 22.04.1 LTS"
/etc/os-release:2:NAME="Ubuntu"
/etc/os-release:3:VERSION_ID="22.04"
/etc/os-release:4:VERSION="22.04.1 LTS (Jammy Jellyfish)"
/etc/os-release:5:VERSION_CODENAME=jammy
/etc/os-release:6:ID=ubuntu
/etc/os-release:7:ID_LIKE=debian
/etc/os-release:8:HOME_URL="https://www.ubuntu.com/"
/etc/os-release:9:SUPPORT_URL="https://help.ubuntu.com/"
/etc/os-release:10:BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
/etc/os-release:11:PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
/etc/os-release:12:UBUNTU_CODENAME=jammy
(Jk6LcMtY, released)
```

```bash
$ kubectl -n receptor get pod -o yaml
...
- apiVersion: v1
  kind: Pod
  metadata:
    ...
    labels:
      app: hello-receptor
    name: custom-runtime-pod-98bmc
    namespace: receptor
    ...
  spec:
    containers:
    - args:
      - -n
      - -H
      - ""
      - /etc/hostname
      - /etc/os-release
      command:
      - grep
      image: ubuntu:22.04
      imagePullPolicy: IfNotPresent
      name: worker
      resources:
        requests:
          cpu: 10m
      ...
    initContainers:
    - args:
      - "5"
      command:
      - sleep
      image: quay.io/centos/centos:stream8
      imagePullPolicy: IfNotPresent
      name: init-worker
      ...
```

## Case 04: Use "InCluster" authmethod

```bash
$ kubectl apply -k .
namespace/receptor created
serviceaccount/receptor created
role.rbac.authorization.k8s.io/receptor created
rolebinding.rbac.authorization.k8s.io/receptor created
configmap/receptor-conf created
pod/node04 created

$ kubectl -n receptor get pod
NAME     READY   STATUS    RESTARTS   AGE
node04   1/1     Running   0          5s

$ kubectl -n receptor get pod node04 -o yaml
apiVersion: v1
kind: Pod
metadata:
  ...
  name: node04
  namespace: receptor
  ...
spec:
  containers:
  - command:
    - receptor
    - -c
    - /etc/receptor/receptor.conf
    image: quay.io/ansible/receptor:v1.3.0
    imagePullPolicy: IfNotPresent
    name: node04
    ...
    volumeMounts:
    - mountPath: /etc/receptor/receptor.conf
      name: receptor-conf
      subPath: receptor.conf
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: kube-api-access-b2sck
      readOnly: true
  ...
  serviceAccount: receptor
  serviceAccountName: receptor
  ...
  volumes:
  - configMap:
      defaultMode: 420
      items:
      - key: node04.yml
        path: receptor.conf
      name: receptor-conf
    name: receptor-conf
  - name: kube-api-access-b2sck
    projected:
      ...
```

```bash
[root@node01 /]# receptorctl status
...

Connection   Cost
relayer01    1

Known Node   Known Connections
controller01 relayer01: 1 
executor01   relayer01: 1 
executor02   relayer01: 1 
relayer01    controller01: 1 executor01: 1 executor02: 1 

Route        Via
executor01   relayer01
executor02   relayer01
relayer01    relayer01

Node         Service   Type       Last Seen             Tags
controller01 control   Stream     2022-12-04 22:23:48   {'type': 'Control Service'}
executor01   control   Stream     2022-12-04 22:23:19   {'type': 'Control Service'}
executor02   control   Stream     2022-12-04 22:22:55   {'type': 'Control Service'}

Node         Work Types
controller01 custom-runtime-pod
executor01   echo-sleep, cat-files, echo-reply, custom-predefined-pod
executor02   echo-reply-incluster
```

```bash
[root@node01 /]# echo "Hello Receptor!" | receptorctl work submit \
  --node executor02 \
  --rm \
  --follow \
  --payload - \
  echo-reply-incluster
Reply from echo-reply-incluster-ql9cp: HELLO RECEPTOR!
(JB484Cvb, released)
```

```bash
$ kubectl -n receptor get pod
NAME                         READY   STATUS      RESTARTS   AGE
node04                       1/1     Running     0          107s
echo-reply-incluster-ql9cp   0/1     Completed   0          2s
```

```bash
$ kubectl -n receptor get pod -o yaml
...
- apiVersion: v1
  kind: Pod
  metadata:
    ...
    generateName: echo-reply-incluster-
    name: echo-reply-incluster-ql9cp
    namespace: receptor
    ...
  spec:
    containers:
    - args:
      - -c
      - 'while read -r line; do echo Reply from $(cat /etc/hostname): ${line^^}; done'
      command:
      - bash
      image: quay.io/centos/centos:stream8
      imagePullPolicy: IfNotPresent
      name: worker
      ...
```
