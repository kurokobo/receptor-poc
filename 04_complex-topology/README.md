# Demo 04: Complex topology

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
west-relayer02    controller01: 1 west-executor02: 1 
west-relayer03    west-controller01: 1 west-executor01: 1 west-relayer01: 1 
 
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
[root@wc01 /]# receptorctl ping east-executor01
Reply from east-executor01 in 1.414839ms
Reply from east-executor01 in 1.977049ms
Reply from east-executor01 in 1.366458ms
Reply from east-executor01 in 1.9261ms

[root@wc01 /]# receptorctl traceroute east-executor01
0: west-controller01 in 96.69µs
1: west-relayer03 in 317.221µs
2: west-relayer01 in 742.736µs
3: controller01 in 839.477µs
4: east-relayer02 in 1.204882ms
5: east-executor01 in 806.648µs
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
