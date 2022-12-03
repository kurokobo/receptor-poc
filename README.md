<!-- omit in toc -->
# [PoC] Receptor

Detailed instructions for this repository (in Japanese):

- [Receptor (1)： Receptor はじめの一歩 | kurokobo.com](https://blog.kurokobo.com/archives/4605)

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
receptorctl  1.3.0.dev2
receptor     1.3.0.dev2

[root@node01 /]# receptorctl status
Node ID: controller01
Version: 1.3.0.dev2
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

[root@node01 /]# echo "Hello Receptor!" > /tmp/input.txt
[root@node01 /]# receptorctl work submit \
  --node executor01 \
  --rm \
  --follow \
  --payload /tmp/input.txt \
  echo-reply
Reply from node03.example.internal: HELLO RECEPTOR!
(vfhNJ5UQ, released)

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
docker compose exec -it node01 bash
```

```bash
[root@wc01 /]# receptorctl status
Node ID: west-controller01
Version: 1.3.0.dev2
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
