<!-- omit in toc -->
# [PoC] Receptor

Detailed instructions for this repository (in Japanese):

- [Receptor (1)： Receptor はじめの一歩 | kurokobo.com](https://blog.kurokobo.com/archives/4605)
- [Receptor (2)： Kubernetes 上でのワークの実行 | kurokobo.com](https://blog.kurokobo.com/archives/4755)
- [Receptor (3)： 暗号化と認証、ファイアウォール、電子署名 | kurokobo.com](https://blog.kurokobo.com/archives/4791)

<!-- omit in toc -->
## Table of Contents

- [References](#references)
- [Demo 01: Connecting nodes and invoking commands](#demo-01-connecting-nodes-and-invoking-commands)
- [Demo 02: Invoking commands with parameters and payloads](#demo-02-invoking-commands-with-parameters-and-payloads)
- [Demo 03: Resiliency](#demo-03-resiliency)
  - [Case 01](#case-01)
  - [Case 02](#case-02)
  - [Case 03](#case-03)
  - [Case 04](#case-04)
  - [Case 05](#case-05)
- [Demo 04: Complex topology](#demo-04-complex-topology)
- [Demo 05: Kubernetes work](#demo-05-kubernetes-work)
  - [Case 01: Simple command](#case-01-simple-command)
  - [Case 02: Parameters and payloads](#case-02-parameters-and-payloads)
  - [Case 03: Customized pod](#case-03-customized-pod)
  - [Case 04: Use "InCluster" authmethod](#case-04-use-incluster-authmethod)
- [Demo 06: Encryption, firewall, work signing](#demo-06-encryption-firewall-work-signing)
  - [Case 01: Allowed peers](#case-01-allowed-peers)
  - [Case 02-1: Encryption / Below-the-mesh encryption](#case-02-1-encryption--below-the-mesh-encryption)
  - [Case 02-2: Encryption / Above-the-mesh encryption](#case-02-2-encryption--above-the-mesh-encryption)
  - [Case 03: Firewall](#case-03-firewall)
  - [Case 04: Work signing](#case-04-work-signing)

## References

- [Introduction - Receptor documentation](https://receptor.readthedocs.io/en/latest/)
- [ansible/receptor](https://github.com/ansible/receptor)

## Demo 01: Connecting nodes and invoking commands

Assets: [📂 01_simple-command](01_simple-command)

```bash
cd 01_simple-command
docker compose up -d
```

```bash
$ docker compose logs -f node01
...
node01  | DEBUG 2022/11/25 19:51:38 Running TCP peer connection node02.example.internal:27199
node01  | WARNING 2022/11/25 19:51:38 Backend connection failed (will retry): dial tcp 172.18.0.4:27199: connect: connection refused
node01  | INFO 2022/11/25 19:51:38 Running control service control
node01  | INFO 2022/11/25 19:51:38 Initialization complete
node01  | DEBUG 2022/11/25 19:51:43 Sending initial connection message
node01  | INFO 2022/11/25 19:51:43 Connection established with relayer01
node01  | DEBUG 2022/11/25 19:51:43 Stopping initial updates
node01  | DEBUG 2022/11/25 19:51:43 Re-calculating routing table
node01  | DEBUG 2022/11/25 19:51:43 Sending routing update INHzPHRD. Connections: relayer01(1.00)
...
node01  | DEBUG 2022/11/25 19:51:43 Received routing update iArMKNrx from relayer01 via relayer01
node01  | DEBUG 2022/11/25 19:51:43 Received routing update f650xBgb from executor01 via relayer01
...
node01  | INFO 2022/11/25 19:51:43 Known Connections:
node01  | INFO 2022/11/25 19:51:43    executor01: relayer01(1.00) 
node01  | INFO 2022/11/25 19:51:43    controller01: relayer01(1.00) 
node01  | INFO 2022/11/25 19:51:43    relayer01: executor01(1.00) controller01(1.00) 
node01  | INFO 2022/11/25 19:51:43 Routing Table:
node01  | INFO 2022/11/25 19:51:43    relayer01 via relayer01
node01  | INFO 2022/11/25 19:51:43    executor01 via relayer01
...
```

```bash
docker compose exec -it node01 bash
```

```bash
[root@node01 /]# receptorctl version
receptorctl  1.3.0
receptor     1.3.0

[root@node01 /]# receptorctl status
Node ID: controller01
Version: 1.3.0
System CPU Count: 4
System Memory MiB: 7762

Connection   Cost
relayer01    1

Known Node   Known Connections
controller01 relayer01: 1 
executor01   relayer01: 1 
relayer01    controller01: 1 executor01: 1 

Route        Via
executor01   relayer01
relayer01    relayer01

Node         Service   Type       Last Seen             Tags
controller01 control   Stream     2022-11-25 19:54:13   {'type': 'Control Service'}
executor01   control   Stream     2022-11-25 19:53:43   {'type': 'Control Service'}

Node         Work Types
controller01 gather-hostname
executor01   gather-hostname
```

```bash
[root@node01 /]# receptorctl work submit \
  --node executor01 \
  --no-payload \
  gather-hostname
Result: Job Started
Unit ID: xfaqU3sC

[root@node01 /]# receptorctl work list
{
    "xfaqU3sC": {
        "Detail": "exit status 0",
        "ExtraData": {
            "Expiration": "0001-01-01T00:00:00Z",
            "LocalCancelled": false,
            "LocalReleased": false,
            "RemoteNode": "executor01",
            "RemoteParams": {},
            "RemoteStarted": true,
            "RemoteUnitID": "wDNTsHQO",
            "RemoteWorkType": "gather-hostname",
            "SignWork": false,
            "TLSClient": ""
        },
        "State": 2,
        "StateName": "Succeeded",
        "StdoutSize": 24,
        "WorkType": "remote"
    }
}

[root@node01 /]# receptorctl work results xfaqU3sC
node03.example.internal

[root@node01 /]# receptorctl work release xfaqU3sC
Released:
(xfaqU3sC, released)
 
[root@node01 /]# receptorctl work list
{}
```

```bash
[root@node01 /]# receptorctl work submit \
  --node executor01 \
  --rm \
  --follow \
  --no-payload \
  gather-hostname
node03.example.internal
(hwhVXaJo, released)
```

## Demo 02: Invoking commands with parameters and payloads

Assets: [📂 02_command-payload](./02_command-payload)

```bash
cd 02_command-payload
docker compose up -d
docker compose exec -it node01 bash
```

```bash
[root@node01 /]# receptorctl work submit \
  --node executor01 \
  --rm \
  --follow \
  --no-payload \
  --param "params=/etc/hostname /etc/os-release" \
  cat-files
/etc/hostname:1:node03.example.internal
/etc/os-release:1:NAME="CentOS Stream"
/etc/os-release:2:VERSION="9"
/etc/os-release:3:ID="centos"
/etc/os-release:4:ID_LIKE="rhel fedora"
/etc/os-release:5:VERSION_ID="9"
/etc/os-release:6:PLATFORM_ID="platform:el9"
/etc/os-release:7:PRETTY_NAME="CentOS Stream 9"
/etc/os-release:8:ANSI_COLOR="0;31"
/etc/os-release:9:LOGO="fedora-logo-icon"
/etc/os-release:10:CPE_NAME="cpe:/o:centos:centos:9"
/etc/os-release:11:HOME_URL="https://centos.org/"
/etc/os-release:12:BUG_REPORT_URL="https://bugzilla.redhat.com/"
/etc/os-release:13:REDHAT_SUPPORT_PRODUCT="Red Hat Enterprise Linux 9"
/etc/os-release:14:REDHAT_SUPPORT_PRODUCT_VERSION="CentOS Stream"
(PPLqDD0a, released)
```

```bash
[root@node01 /]# echo "Hello Receptor!" > /tmp/input.txt
[root@node01 /]# receptorctl work submit \
  --node executor01 \
  --rm \
  --follow \
  --payload /tmp/input.txt \
  echo-reply
Reply from node03.example.internal: HELLO RECEPTOR!
(vfhNJ5UQ, released)

[root@node01 /]# echo "Hello Receptor!" | receptorctl work submit \
  --node executor01 \
  --rm \
  --follow \
  --payload - \
  echo-reply
Reply from node03.example.internal: HELLO RECEPTOR!
(NX9JQHC9, released)

[root@node01 /]# receptorctl work submit \
  --node executor01 \
  --rm \
  --follow \
  --payload-literal "Hello Receptor!" \
  echo-reply
Reply from node03.example.internal: HELLO RECEPTOR!
(Xs3Q5LDU, released)
```

## Demo 03: Resiliency

Assets: [📂 03_command-resiliency](03_command-resiliency)

```bash
cd 03_command-resiliency
docker compose up -d
```

### Case 01

```bash
[root@node01 /]# echo "Hello Receptor!" | receptorctl work submit \
  --node executor01 \
  --payload - \
  echo-reply
Result: Job Started
Unit ID: dcyJmLNy
 
[root@node01 /]# receptorctl work list --unit_id dcyJmLNy
{
    "dcyJmLNy": {
        "Detail": "exit status 0",
        "ExtraData": {
            "Expiration": "0001-01-01T00:00:00Z",
            "LocalCancelled": false,
            "LocalReleased": false,
            "RemoteNode": "executor01",
            "RemoteParams": {},
            "RemoteStarted": true,
            "RemoteUnitID": "gfxFWsHC",
            "RemoteWorkType": "echo-reply",
            "SignWork": false,
            "TLSClient": ""
        },
        "State": 2,
        "StateName": "Succeeded",
        "StdoutSize": 52,
        "WorkType": "remote"
    }
}
```

```bash
[root@node01 /]# cat /tmp/receptor/controller01/dcyJmLNy/status
{
  "State": 2,
  "Detail": "exit status 0",
  "StdoutSize": 52,
  "WorkType": "remote",
  "ExtraData": {
    "RemoteNode": "executor01",
    "RemoteWorkType": "echo-reply",
    "RemoteParams": {},
    "RemoteUnitID": "gfxFWsHC",
    "RemoteStarted": true,
    "LocalCancelled": false,
    "LocalReleased": false,
    "SignWork": false,
    "TLSClient": "",
    "Expiration": "0001-01-01T00:00:00Z"
  }
}
 
[root@node01 /]# cat /tmp/receptor/controller01/dcyJmLNy/stdin
Hello Receptor!
 
[root@node01 /]# cat /tmp/receptor/controller01/dcyJmLNy/stdout
Reply from node04.example.internal: HELLO RECEPTOR!
```

```bash
[root@node04 /]# cat /tmp/receptor/executor01/gfxFWsHC/status
{
  "State": 2,
  "Detail": "exit status 0",
  "StdoutSize": 52,
  "WorkType": "echo-reply",
  "ExtraData": null
}
 
[root@node04 /]# cat /tmp/receptor/executor01/gfxFWsHC/stdin
Hello Receptor!
 
[root@node04 /]# cat /tmp/receptor/executor01/gfxFWsHC/stdout
Reply from node04.example.internal: HELLO RECEPTOR!
```

```bash
[root@node01 /]# receptorctl work release --all
Released:
(dcyJmLNy, released)
 
[root@node01 /]# ls -l /tmp/receptor/controller01/
total 0
```

### Case 02

```bash
[root@node01 /]# receptorctl work submit \
  --node executor01 \
  --no-payload \
  echo-sleep
Result: Job Started
Unit ID: uocGYk24
```

```bash
$ docker compose stop node02
[+] Running 1/1
 ⠿ Container node02  Stopped  0.1s
```

```bash
$ docker compose logs -f node01
...
node01  | WARNING 2022/11/25 21:37:38 Backend connection exited (will retry)
...
node01  | DEBUG 2022/11/25 21:37:38 Re-calculating routing table
node01  | INFO 2022/11/25 21:37:38 Known Connections:
node01  | INFO 2022/11/25 21:37:38    controller01: relayer02(2.00) 
node01  | INFO 2022/11/25 21:37:38    relayer02: controller01(2.00) executor01(2.00) 
node01  | INFO 2022/11/25 21:37:38    executor01: relayer02(2.00) 
node01  | INFO 2022/11/25 21:37:38    relayer01: 
node01  | INFO 2022/11/25 21:37:38 Routing Table:
node01  | INFO 2022/11/25 21:37:38    relayer02 via relayer02
node01  | INFO 2022/11/25 21:37:38    executor01 via relayer02
...
node01  | WARNING 2022/11/25 21:37:43 Backend connection failed (will retry): dial tcp: lookup node02.example.internal on 127.0.0.11:53: no such host
...
node01  | WARNING 2022/11/25 21:37:51 Backend connection failed (will retry): dial tcp: lookup node02.example.internal on 127.0.0.11:53: no such host
...
```

```bash
[root@node01 /]# receptorctl status
...
Connection   Cost
relayer02    2
 
Known Node   Known Connections
controller01 relayer02: 2 
executor01   relayer02: 2 
relayer01    
relayer02    controller01: 2 executor01: 2 
 
Route        Via
executor01   relayer02
relayer02    relayer02
...

[root@node01 /]# receptorctl work list --unit_id uocGYk24
{
    "uocGYk24": {
        ...
        "StateName": "Succeeded",
        ...
    }
}

[root@node01 /]# receptorctl work results uocGYk24
1: 21:37:32
2: 21:37:37
...
11: 21:38:23
12: 21:38:28
```

```bash
$ docker compose start node02
[+] Running 1/1
 ⠿ Container node02  Started  0.3s
```

```bash
[root@node01 /]# receptorctl status
...
Connection   Cost
relayer02    2
relayer01    1
 
Known Node   Known Connections
controller01 relayer01: 1 relayer02: 2 
executor01   relayer01: 1 relayer02: 2 
relayer01    controller01: 1 executor01: 1 
relayer02    controller01: 2 executor01: 2 
 
Route        Via
executor01   relayer01
relayer01    relayer01
relayer02    relayer02
...
```

```bash
[root@node01 /]# receptorctl work release --all
Released:
(uocGYk24, released)
```

### Case 03

```bash
[root@node01 /]# receptorctl work submit \
  --node executor01 \
  --no-payload \
  echo-sleep
Result: Job Started
Unit ID: GD8cOc0c
```

```bash
$ docker compose stop node02
[+] Running 1/1
 ⠿ Container node02  Stopped  0.1s
 
$ docker compose stop node03
[+] Running 1/1
 ⠿ Container node03  Stopped  0.1s
```

```bash
[root@node01 /]# receptorctl status
...
Known Node   Known Connections
controller01 
executor01   relayer02: 2 
relayer01    
relayer02    executor01: 2 
...
```

```bash
[root@node01 /]# receptorctl work list --unit_id GD8cOc0c
{
    "GD8cOc0c": {
        "Detail": "Running: PID 83",
        "ExtraData": {
            ...
            "RemoteUnitID": "2Uf3Cbjj",
            ...
        },
        "State": 1,
        "StateName": "Running",
        "StdoutSize": 24,
        ...
    }
}
 
[root@node01 /]# cat /tmp/receptor/controller01/GD8cOc0c/status
{
  "State": 1,
  "Detail":"Running: PID 83",
  "StdoutSize":24,
  ...
  "ExtraData": {
    ...
    "RemoteUnitID": "2Uf3Cbjj",
    ...
  }
}
 
[root@node01 /]# cat /tmp/receptor/controller01/GD8cOc0c/stdout 
1: 11:31:18
2: 11:31:23
```

```bash
[root@node04 /]# cat /tmp/receptor/executor01/2Uf3Cbjj/status
{
  "State": 2,
  "Detail": "exit status 0",
  "StdoutSize": 147,
  ...
}
 
[root@node04 /]# cat /tmp/receptor/executor01/2Uf3Cbjj/stdout 
1: 11:31:18
2: 11:31:23
...
11: 11:32:09
12: 11:32:14
```

```bash
$ docker compose start node02
[+] Running 1/1
 ⠿ Container node02  Started  0.3s
```

```bash
[root@node01 /]# receptorctl status
...
Connection   Cost
relayer01    1
 
Known Node   Known Connections
controller01 relayer01: 1 
executor01   relayer01: 1 
relayer01    controller01: 1 executor01: 1 
relayer02    
 
Route        Via
executor01   relayer01
relayer01    relayer01
...
 
[root@node01 /]# receptorctl work list --unit_id GD8cOc0c
{
    "GD8cOc0c": {
        "Detail": "exit status 0",
        ...
        "StateName": "Succeeded",
        "StdoutSize": 147,
        ...
    }
}
 
[root@node01 /]# receptorctl work results GD8cOc0c
1: 11:31:18
2: 11:31:23
...
11: 11:32:09
12: 11:32:14
```

```bash
$ docker compose start node03
[+] Running 1/1
 ⠿ Container node03  Started  0.3s
```

```bash
[root@node01 /]# receptorctl work release --all
Released:
(GD8cOc0c, released)
```

### Case 04

```bash
[root@node01 /]# receptorctl work submit \
  --node executor01 \
  --no-payload \
  echo-sleep
Result: Job Started
Unit ID: zna1urHX
```

```bash
$ docker compose stop node01
[+] Running 1/1
 ⠿ Container node01  Stopped  0.1s
```

```bash
[root@node04 /]# cat /tmp/receptor/executor01/nJO9PdbS/status
{
  "State": 2,
  "Detail": "exit status 0",
  "StdoutSize": 147,
  ...
}
```

```bash
$ docker compose start node01
[+] Running 1/1
 ⠿ Container node02  Started  0.3s
```

```bash
[root@node01 /]# receptorctl work list
{
    "zna1urHX": {
        "Detail": "exit status 0",
        "ExtraData": {
            ...
            "RemoteUnitID": "nJO9PdbS",
            ...
        },
        "State": 2,
        "StateName": "Succeeded",
        "StdoutSize": 147,
        ...
    }
}
```

```bash
[root@node01 /]# receptorctl work release --all
Released:
(zna1urHX, released)
```

### Case 05

```bash
[root@node01 /]# receptorctl work submit \
  --node executor01 \
  --no-payload \
  echo-sleep
Result: Job Started
Unit ID: 5lIdpCah
```

```bash
$ docker compose stop node04
[+] Running 1/1
 ⠿ Container node01  Stopped  0.1s
```

```bash
[root@node01 /]# receptorctl work list
{
    "5lIdpCah": {
        "Detail": "Running: PID 182",
        "ExtraData": {
            ...
            "RemoteUnitID": "62ESTRMU",
            ...
        },
        "State": 1,
        "StateName": "Running",
        "StdoutSize": 96,
        ...
    }
}
```

```bash
$ docker compose start node04
[+] Running 1/1
 ⠿ Container node04  Started  0.3s
```

```bash
[root@node01 /]# receptorctl work list
{
    "5lIdpCah": {
        "Detail": "Running: PID 182",
        "ExtraData": {
            ...
            "RemoteUnitID": "62ESTRMU",
            ...
        },
        "State": 1,
        "StateName": "Running",
        "StdoutSize": 96,
        ...
    }
}
```

```bash
[root@node04 /]# cat /tmp/receptor/executor01/62ESTRMU/status
{
  "State": 1,
  "Detail": "Running: PID 182",
  ...
}
```

```bash
[root@node01 /]# receptorctl work release --all
Released:
(5lIdpCah, released)
```

## Demo 04: Complex topology

Assets: [📂 04_complex-topology](04_complex-topology)

```bash
cd 04_complex-topology
docker compose up -d
docker compose exec -it wc01 bash
```

```bash
[root@wc01 /]# receptorctl status
Node ID: west-controller01
Version: 1.3.0
System CPU Count: 4
System Memory MiB: 7762
 
Connection        Cost
west-relayer03    1
 
Known Node        Known Connections
controller01      east-relayer01: 1 east-relayer02: 1 west-relayer01: 1 west-relayer02: 1 
east-executor01   east-relayer01: 1 east-relayer02: 1 
east-relayer01    controller01: 1 east-executor01: 1 
east-relayer02    controller01: 1 east-executor01: 1 
west-controller01 west-relayer03: 1 
west-executor01   west-relayer03: 1 
west-executor02   west-relayer01: 1 west-relayer02: 1 
west-relayer01    controller01: 1 west-executor02: 1 west-relayer03: 1 
west-relayer02    controller01: 1 west-executor02: 1 west-relayer03: 1 
west-relayer03    west-controller01: 1 west-executor01: 1 west-relayer01: 1 west-relayer02: 1 
 
Route             Via
controller01      west-relayer03
east-executor01   west-relayer03
east-relayer01    west-relayer03
east-relayer02    west-relayer03
west-executor01   west-relayer03
west-executor02   west-relayer03
west-relayer01    west-relayer03
west-relayer02    west-relayer03
west-relayer03    west-relayer03
 
Node              Service   Type       Last Seen             Tags
west-controller01 control   Stream     2022-11-26 20:12:30   {'type': 'Control Service'}
east-executor01   control   Stream     2022-11-26 20:11:35   {'type': 'Control Service'}
west-executor01   control   Stream     2022-11-26 20:11:36   {'type': 'Control Service'}
west-executor02   control   Stream     2022-11-26 20:11:36   {'type': 'Control Service'}
controller01      control   Stream     2022-11-26 20:11:36   {'type': 'Control Service'}
 
Node              Work Types
east-executor01   gather-hostname
west-executor01   gather-hostname
west-executor02   gather-hostname
```

```bash
[root@wc01 /]# receptorctl work submit \
  --node east-executor01 \
  --rm \
  --follow \
  --no-payload \
  gather-hostname
ee01.example.internal
(7W9RAP7A, released)
```

## Demo 05: Kubernetes work

Assets: [📂 05_kubernetes](05_kubernetes)

```bash
cd 05_kubernetes
cp /etc/rancher/k3s/k3s.yaml .kube/config
sed -i 's/127\.0\.0\.1/192.168.0.219/g' .kube/config
docker compose up -d
```

```bash
kubectl apply -f manifests/namespace.yml
```

### Case 01: Simple command

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

### Case 02: Parameters and payloads

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

### Case 03: Customized pod

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

### Case 04: Use "InCluster" authmethod

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

## Demo 06: Encryption, firewall, work signing

Assets: [📂 06_encryption-firewall](06_encryption-firewall)

```bash
RECEPTOR_IMAGE="quay.io/ansible/receptor:v1.3.0"
NODES="
c01.example.internal:controller01
c02.example.internal:controller02
c03.example.internal:controller03
c04.example.internal:controller04
r01.example.internal:relayer01
e01.example.internal:executor01
e02.example.internal:executor02
e03.example.internal:executor03
"

# Generate Root CA Cert
docker run --rm --volume "${PWD}/certs:/tmp/certs" ${RECEPTOR_IMAGE} \
  receptor --cert-init commonname="Receptor Example CA" bits=2048 outcert=/tmp/certs/ca.crt outkey=/tmp/certs/ca.key

# Generate Server Certs
for item in ${NODES}; do
  node=(${item//:/ })
  echo "Generating new certificate for ${node[0]} (${node[1]})"
  docker run --rm --volume "${PWD}/certs:/tmp/certs" ${RECEPTOR_IMAGE} \
    receptor --cert-makereq bits=2048 commonname="${node[1]} example cert" dnsname=${node[0]} nodeid=${node[1]} outreq=/tmp/certs/${node[0]}.csr outkey=/tmp/certs/${node[0]}.key
  docker run --rm --volume "${PWD}/certs:/tmp/certs" ${RECEPTOR_IMAGE} \
    receptor --cert-signreq req=/tmp/certs/${node[0]}.csr cacert=/tmp/certs/ca.crt cakey=/tmp/certs/ca.key outcert=/tmp/certs/${node[0]}.crt verify=true
  docker run --rm --volume "${PWD}/certs:/tmp/certs" ${RECEPTOR_IMAGE} \
    rm /tmp/certs/${node[0]}.csr
done
```

```bash
openssl genrsa -out certs/signworkprivate.pem 2048
openssl rsa -in certs/signworkprivate.pem -pubout -out certs/signworkpublic.pem

openssl genrsa -out certs/signworkprivate.c02.pem 2048
openssl rsa -in certs/signworkprivate.c02.pem -pubout -out certs/signworkpublic.c02.pem
```

```bash
$ ls -l certs/
total 80
-rw-------. 1 root root 1212 Dec  9 15:58 c01.example.internal.crt
-rw-------. 1 root root 1675 Dec  9 15:58 c01.example.internal.key
-rw-------. 1 root root 1212 Dec  9 15:58 c02.example.internal.crt
-rw-------. 1 root root 1675 Dec  9 15:58 c02.example.internal.key
-rw-------. 1 root root 1212 Dec  9 15:58 c03.example.internal.crt
-rw-------. 1 root root 1679 Dec  9 15:58 c03.example.internal.key
-rw-------. 1 root root 1212 Dec  9 15:58 c04.example.internal.crt
-rw-------. 1 root root 1679 Dec  9 15:58 c04.example.internal.key
-rw-------. 1 root root 1139 Dec  9 15:58 ca.crt
-rw-------. 1 root root 1679 Dec  9 15:58 ca.key
-rw-------. 1 root root 1208 Dec  9 15:58 e01.example.internal.crt
-rw-------. 1 root root 1679 Dec  9 15:58 e01.example.internal.key
-rw-------. 1 root root 1208 Dec  9 15:58 e02.example.internal.crt
-rw-------. 1 root root 1679 Dec  9 15:58 e02.example.internal.key
-rw-------. 1 root root 1208 Dec  9 15:58 e03.example.internal.crt
-rw-------. 1 root root 1679 Dec  9 15:58 e03.example.internal.key
-rw-------. 1 root root 1204 Dec  9 15:58 r01.example.internal.crt
-rw-------. 1 root root 1679 Dec  9 15:58 r01.example.internal.key
-rw-------. 1 kuro kuro 1675 Dec  9 16:05 signworkprivate.c02.pem
-rw-------. 1 kuro kuro 1675 Dec  9 16:05 signworkprivate.pem
-rw-rw-r--. 1 kuro kuro  451 Dec  9 16:05 signworkpublic.c02.pem
-rw-rw-r--. 1 kuro kuro  451 Dec  9 16:05 signworkpublic.pem
```

### Case 01: Allowed peers

```bash
$ docker compose logs c04
c04  | DEBUG 2022/12/09 06:18:49 Running TCP peer connection r01.example.internal:7323
c04  | DEBUG 2022/12/09 06:18:49 Sending initial connection message
...
c04  | WARNING 2022/12/09 06:18:54 Backend connection exited (will retry)
...
c04  | DEBUG 2022/12/09 06:18:54 Re-calculating routing table
c04  | INFO 2022/12/09 06:18:54 Known Connections:
c04  | INFO 2022/12/09 06:18:54    controller04: 
c04  | INFO 2022/12/09 06:18:54    relayer01: 
c04  | INFO 2022/12/09 06:18:54 Routing Table:
...
```

```bash
$ docker compose logs r01
...
r01  | DEBUG 2022/12/09 06:18:49 Listening on TCP [::]:7323
...
r01  | ERROR 2022/12/09 06:18:49 Backend receiving error read tcp 172.21.0.4:7323->172.21.0.5:54814: use of closed network connection
r01  | ERROR 2022/12/09 06:18:49 Backend error: rejected connection with node controller04 because it is not in the allowed peers list
...
```

```bash
[root@c01 /]# receptorctl status
...
Known Node   Known Connections
controller01 relayer01: 1 
controller02 relayer01: 1 
controller03 relayer01: 1 
executor01   relayer01: 1 
executor02   relayer01: 1 
executor03   relayer01: 1 
relayer01    controller01: 1 controller02: 1 controller03: 1 executor01: 1 executor02: 1 executor03: 1 
...
```

```bash
[root@c04 /]# receptorctl status
...
Known Node   Known Connections
controller04 
relayer01    

Node         Service   Type       Last Seen             Tags
controller04 control   Stream     2022-12-09 06:20:57   {'type': 'Control Service'}
```

### Case 02-1: Encryption / Below-the-mesh encryption

```bash
$ sudo openssl x509 -text -in certs/ca.crt -noout
Certificate:
    Data:
        ...
        Issuer: CN = Receptor Example CA
        Validity
            Not Before: Dec  9 05:58:46 2022 GMT
            Not After : Dec  9 05:58:46 2032 GMT
        Subject: CN = Receptor Example CA
        ...
```

```bash
$ CERT="certs/c01.example.internal.crt"
$ sudo openssl x509 -text -in ${CERT} -noout
Certificate:
    Data:
        ...
        Issuer: CN = Receptor Example CA
        Validity
            Not Before: Dec  9 05:58:47 2022 GMT
            Not After : Dec  9 05:58:47 2023 GMT
        Subject: CN = controller01 example cert
        ...
        X509v3 extensions:
            X509v3 Subject Alternative Name: 
                DNS:c01.example.internal, othername:<unsupported>
    ...

$ OFFSET=$(sudo openssl asn1parse -in ${CERT} | grep -A 1 "X509v3 Subject Alternative Name" | tail -n 1 | sed 's/:.*//g')
$ sudo openssl asn1parse -in ${CERT} -strparse ${OFFSET}
   ...
   26:d=2  hl=2 l=   9 prim: OBJECT            :1.3.6.1.4.1.2312.19.1
   ...
   39:d=3  hl=2 l=  12 prim: UTF8STRING        :controller01
```

```bash
docker compose up -d
```

```bash
[root@c01 /]# receptorctl status
...
Known Node   Known Connections
controller01 relayer01: 1 
controller02 relayer01: 1 
controller03 relayer01: 1 
executor01   relayer01: 1 
executor02   relayer01: 1 
executor03   relayer01: 1 
relayer01    controller01: 1 controller02: 1 controller03: 1 executor01: 1 executor02: 1 executor03: 1 

Route        Via
controller02 relayer01
controller03 relayer01
executor01   relayer01
executor02   relayer01
executor03   relayer01
relayer01    relayer01
...
```

### Case 02-2: Encryption / Above-the-mesh encryption

```bash
[root@c01 /]# receptorctl status
...
Node         Service   Type       Last Seen             Tags
executor03   control   StreamTLS  2022-12-09 06:28:04   {'type': 'Control Service'}
controller01 control   Stream     2022-12-09 06:28:07   {'type': 'Control Service'}
controller02 control   Stream     2022-12-09 06:27:09   {'type': 'Control Service'}
executor01   control   Stream     2022-12-09 06:27:10   {'type': 'Control Service'}
controller03 control   Stream     2022-12-09 06:27:10   {'type': 'Control Service'}
executor02   control   Stream     2022-12-09 06:27:10   {'type': 'Control Service'}
```

```bash
[root@c01 /]# echo "Hello Receptor!" | receptorctl work submit \
  --node executor01 \
  --rm \
  --follow \
  --payload - \
  echo-reply
Reply from e01.example.internal: HELLO RECEPTOR!
(gCobCpTe, released)
```

```bash
[root@c01 /]# echo "Hello Receptor!" | receptorctl work submit \
  --node executor03 \
  --rm \
  --follow \
  --payload - \
  echo-reply
ERROR: Remote unit failed: TLS error connecting to remote service: CRYPTO_ERROR (0x12a): insecure connection to secure service
(fi4rBON8, released)
```

```bash
[root@c01 /]# echo "Hello Receptor!" | receptorctl work submit \
  --node executor03 \
  --rm \
  --follow \
  --payload - \
  --tls-client example-tls-client \
  echo-reply
Reply from e03.example.internal: HELLO RECEPTOR!
(Pc9E1HSG, released)
```

### Case 03: Firewall

```bash
[root@c01 /]# echo "Hello Receptor!" | receptorctl work submit \
  --node executor01 \
  --rm \
  --follow \
  --payload - \
  echo-reply
Reply from e01.example.internal: HELLO RECEPTOR!
(0eyPn5a8, released)
```

```bash
[root@c02 /]# echo "Hello Receptor!" | receptorctl work submit \
  --node executor01 \
  --rm \
  --follow \
  --payload - \
  echo-reply
Reply from e01.example.internal: HELLO RECEPTOR!
(p2qLHTyg, released)
```

```bash
[root@c03 /]# echo "Hello Receptor!" | receptorctl work submit \
  --node executor01 \
  --rm \
  --follow \
  --payload - \
  echo-reply
^C
(8B0hLBzU, released)
```

```bash
[root@c03 /]# echo "Hello Receptor!" | receptorctl work submit \
  --node executor01 \
  --payload - \
  echo-reply
^C

[root@c03 /]# receptorctl work list
{
    "gtAwdBBS": {
        "Detail": "Starting Worker",
        ...
        "State": 0,
        "StateName": "Pending",
        ...
    }
}

[root@c03 /]# receptorctl work release --all
Released:
(gtAwdBBS, released)
```

```bash
[root@c01 /]# echo "Hello Receptor!" | receptorctl work submit \
  --node executor01 \
  --rm \
  --follow \
  --payload - \
  echo-reply
Reply from e01.example.internal: HELLO RECEPTOR!
(cXaxadUZ, released)

[root@c01 /]# echo "Hello Receptor!" | receptorctl work submit \
  --node executor02 \
  --rm \
  --follow \
  --payload - \
  echo-reply
Reply from e02.example.internal: HELLO RECEPTOR!
(kdnrbl9s, released)
```

```bash
[root@c02 /]# echo "Hello Receptor!" | receptorctl work submit \
  --node executor01 \
  --rm \
  --follow \
  --payload - \
  echo-reply
Reply from e01.example.internal: HELLO RECEPTOR!
(R0kTKm5W, released)

[root@c02 /]# echo "Hello Receptor!" | receptorctl work submit \
  --node executor02 \
  --rm \
  --follow \
  --payload - \
  echo-reply
^C
```

```bash
$ docker compose logs c02
...
c02  | WARNING 2022/12/09 06:38:56 Received unreachable message from executor02
...
```

```bash
[root@c02 /]# receptorctl work release --all
Released:
(xFD0u0uh, released)
```

### Case 04: Work signing

```bash
[root@c01 /]# receptorctl status
...
Node         Work Types
executor02   echo-reply
executor01   echo-reply
executor03   echo-reply

Node         Secure Work Types
executor01   echo-verified-reply
executor03   echo-verified-reply
```

```bash
[root@c01 /]# echo "Hello Receptor!" | receptorctl work submit \
  --node executor01 \
  --rm \
  --follow \
  --payload - \
  echo-verified-reply
ERROR: Remote error: ERROR: could not parse response: ERROR: could not verify signature: signature is empty
```

```bash
[root@c01 /]# echo "Hello Receptor!" | receptorctl work submit \
  --node executor01 \
  --rm \
  --follow \
  --payload - \
  --signwork \
  echo-verified-reply
Reply from e01.example.internal for verified work: HELLO RECEPTOR!
(PvvdSwrU, released)
```

```bash
[root@c02 /]# echo "Hello Receptor!" | receptorctl work submit \
  --node executor01 \
  --rm \
  --follow \
  --payload - \
  --signwork \
  echo-verified-reply
ERROR: Remote error: ERROR: could not parse response: ERROR: could not verify signature: crypto/rsa: verification error
```

```bash
[root@c01 /]# echo "Hello Receptor!" | receptorctl work submit \
  --node executor03 \
  --rm \
  --follow \
  --payload - \
  echo-verified-reply
ERROR: Remote unit failed: TLS error connecting to remote service: CRYPTO_ERROR (0x12a): insecure connection to secure service
(GWtzT1HE, released)

[root@c01 /]# echo "Hello Receptor!" | receptorctl work submit \
  --node executor03 \
  --rm \
  --follow \
  --payload - \
  --tls-client example-tls-client \
  echo-verified-reply
ERROR: Remote error: ERROR: could not parse response: ERROR: could not verify signature: signature is empty

[root@c01 /]# echo "Hello Receptor!" | receptorctl work submit \
  --node executor03 \
  --rm \
  --follow \
  --payload - \
  --tls-client example-tls-client \
  --signwork \
  echo-verified-reply
Reply from e03.example.internal for verified work: HELLO RECEPTOR!
(PA6EyrXs, released)
```
