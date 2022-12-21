<!-- omit in toc -->
# Demo 03: Resiliency

<!-- omit in toc -->
## Table of Contents

- [Preparation](#preparation)
- [Case 01: Normal behavior](#case-01-normal-behavior)
- [Case 02: Redundancy issue](#case-02-redundancy-issue)
- [Case 03: Unreachable](#case-03-unreachable)
- [Case 04: Node issue (Controller)](#case-04-node-issue-controller)
- [Case 04: Node issue (Executor)](#case-04-node-issue-executor)

## Preparation

```bash
cd 03_command-resiliency
docker compose up -d
```

```bash
[root@node01 /]# receptorctl status
...
Connection   Cost
relayer01    1
relayer02    2

...
Route        Via
executor01   relayer01
relayer01    relayer01
relayer02    relayer02

[root@node01 /]# receptorctl traceroute executor01
0: controller01 in 70.855µs
1: relayer01 in 344.99µs
2: executor01 in 330.43µs
```

## Case 01: Normal behavior

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

## Case 02: Redundancy issue

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

[root@node01 /]# receptorctl traceroute executor01
0: controller01 in 93.198µs
1: relayer02 in 308.564µs
2: executor01 in 336.695µs

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

[root@node01 /]# receptorctl traceroute executor01
0: controller01 in 58.766µs
1: relayer01 in 305.701µs
2: executor01 in 349.324µs
```

```bash
[root@node01 /]# receptorctl work release --all
Released:
(uocGYk24, released)
```

## Case 03: Unreachable

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

## Case 04: Node issue (Controller)

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

## Case 04: Node issue (Executor)

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
