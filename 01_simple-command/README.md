# Demo 01: Connecting nodes and invoking commands

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
[root@node01 /]# receptorctl ping relayer01
Reply from relayer01 in 317.614µs
Reply from relayer01 in 352.845µs
Reply from relayer01 in 315.205µs
Reply from relayer01 in 254.051µs

[root@node01 /]# receptorctl ping executor01
Reply from executor01 in 451.259µs
Reply from executor01 in 539.444µs
Reply from executor01 in 587.212µs
Reply from executor01 in 588.532µs
```

```bash
[root@node01 /]# receptorctl traceroute executor01
0: controller01 in 73.859µs
1: relayer01 in 429.427µs
2: executor01 in 694.976µs
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
