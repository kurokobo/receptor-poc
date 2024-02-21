<!-- omit in toc -->
# Demo 06: Encryption, firewall, work signing

<!-- omit in toc -->
## Table of Contents

- [Scenario](#scenario)
- [Preparation: Generate Keys and Certs](#preparation-generate-keys-and-certs)
- [Add peers on-the-fly](#add-peers-on-the-fly)

## Scenario

Initially, we have a `controller01` and a `relayer01` connected to the mesh.

We want to add an `executor01` to the mesh. Receptor is up and running on `executor01`, but it is not connected to the mesh yet.

The `relayer01` has pre-defined works to update its configuration on-the-fly:

- `receptor-gather-config`
  - Invokes `cat /etc/receptor/receptor.conf`.
- `receptor-patch-config`
  - Appends the content of the payload to `/etc/receptor/receptor.conf`.
- `receptor-reload-config`
  - Invokes `receptorctl reload`.

On `controller01`, there is a patch file `/tmp/c01-patch-r01.yml` that contains the configuration for `relayer01` to connect `executor01`.

In this scenario, we will invoke `receptorctl` on `controller01` to demonstrate how to add `executor01` to the mesh on-the-fly.

## Preparation: Generate Keys and Certs

```bash
RECEPTOR_IMAGE="quay.io/ansible/receptor:v1.4.4"
NODES="
c01.example.internal:controller01
r01.example.internal:relayer01
e01.example.internal:executor01
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

# Generate Work Signing Keys
openssl genrsa -out certs/signworkprivate.pem 2048
openssl rsa -in certs/signworkprivate.pem -pubout -out certs/signworkpublic.pem
```

```bash
$ ls -l certs/
total 40
-rw-------. 1 root root 1212 Feb 21 22:51 c01.example.internal.crt
-rw-------. 1 root root 1679 Feb 21 22:51 c01.example.internal.key
-rw-------. 1 root root 1139 Feb 21 22:51 ca.crt
-rw-------. 1 root root 1679 Feb 21 22:51 ca.key
-rw-------. 1 root root 1208 Feb 21 22:51 e01.example.internal.crt
-rw-------. 1 root root 1675 Feb 21 22:51 e01.example.internal.key
-rw-------. 1 root root 1204 Feb 21 22:51 r01.example.internal.crt
-rw-------. 1 root root 1679 Feb 21 22:51 r01.example.internal.key
-rw-------. 1 kuro kuro 1679 Feb 21 22:51 signworkprivate.pem
-rw-rw-r--. 1 kuro kuro  451 Feb 21 22:51 signworkpublic.pem
```

## Add peers on-the-fly

Get bash shell on controller01.

```bash
$ docker compose up -d
$ docker compose exec -it c01 bash
[root@c01 /]# 
```

Ensure only contoller01 and relayer01 are connected to the mesh.

```bash
[root@c01 /]# receptorctl status
Node ID: controller01
Version: 1.4.4
System CPU Count: 4
System Memory MiB: 7762

Connection   Cost
relayer01    1

Known Node   Known Connections
controller01 relayer01: 1 
relayer01    controller01: 1 

Route        Via
relayer01    relayer01

Node         Service   Type       Last Seen             Tags
controller01 control   Stream     2024-02-21 13:51:45   {'type': 'Control Service'}
relayer01    control   Stream     2024-02-21 13:51:42   {'type': 'Control Service'}

Node         Secure Work Types
controller01 receptor-gather-config
relayer01    receptor-gather-config, receptor-patch-config, receptor-reload-config
```

Gather current configuration on relayer01.

```bash
[root@c01 /]# receptorctl work submit --node relayer01 --rm --follow --no-payload --signwork receptor-gather-config
---
- node:
    id: relayer01

- log-level:
    level: debug

- control-service:
    service: control
    filename: /tmp/receptor.sock

- tls-server:
    name: tls-server
    cert: /etc/receptor/certs/r01.example.internal.crt
    key: /etc/receptor/certs/r01.example.internal.key
    requireclientcert: true
    clientcas: /etc/receptor/certs/ca.crt

- tls-client:
    name: tls-client
    rootcas: /etc/receptor/certs/ca.crt
    cert: /etc/receptor/certs/r01.example.internal.crt
    key: /etc/receptor/certs/r01.example.internal.key

- tcp-listener:
    port: 7323
    tls: tls-server

- work-verification:
    publickey: /etc/receptor/certs/signworkpublic.pem

- work-command:
    worktype: receptor-gather-config
    command: cat
    params: /etc/receptor/receptor.conf
    verifysignature: true

- work-command:
    worktype: receptor-patch-config
    command: bash
    params: "-c 'while read -r; do echo \"${REPLY}\" >> /etc/receptor/receptor.conf; done'"
    verifysignature: true

- work-command:
    worktype: receptor-reload-config
    command: receptorctl
    params: reload
    verifysignature: true
(nGq4gE6n, released)
```

Ensure the patch for relayer01 is on `/tmp/c01-patch-r01.yml`.

```bash
[root@c01 /]# cat /tmp/c01-patch-r01.yml

- tcp-peer:
    address: e01.example.internal:7323
    tls: tls-client
```

Send the patch to relayer01.

```bash
[root@c01 /]# cat /tmp/c01-patch-r01.yml | receptorctl work submit --node relayer01 --rm --follow --payload - --signwork receptor-patch-config
(LTTA7gmp, released)
```

Ensure configuration file is updated on relayer01.

```bash
[root@c01 /]# receptorctl work submit --node relayer01 --rm --follow --no-payload --signwork receptor-gather-config
---
- node:
    id: relayer01

- log-level:
    level: debug

- control-service:
    service: control
    filename: /tmp/receptor.sock

- tls-server:
    name: tls-server
    cert: /etc/receptor/certs/r01.example.internal.crt
    key: /etc/receptor/certs/r01.example.internal.key
    requireclientcert: true
    clientcas: /etc/receptor/certs/ca.crt

- tls-client:
    name: tls-client
    rootcas: /etc/receptor/certs/ca.crt
    cert: /etc/receptor/certs/r01.example.internal.crt
    key: /etc/receptor/certs/r01.example.internal.key

- tcp-listener:
    port: 7323
    tls: tls-server

- work-verification:
    publickey: /etc/receptor/certs/signworkpublic.pem

- work-command:
    worktype: receptor-gather-config
    command: cat
    params: /etc/receptor/receptor.conf
    verifysignature: true

- work-command:
    worktype: receptor-patch-config
    command: bash
    params: "-c 'while read -r; do echo \"${REPLY}\" >> /etc/receptor/receptor.conf; done'"
    verifysignature: true

- work-command:
    worktype: receptor-reload-config
    command: receptorctl
    params: reload
    verifysignature: true

- tcp-peer:
    address: e01.example.internal:7323
    tls: tls-client
(Har53eAQ, released)
```

Trigger `receptorctl reload` on relayer01.

```bash
[root@c01 /]# receptorctl work submit --node relayer01 --rm --follow --no-payload --signwork receptor-reload-config
Reload successful
(TecINTBr, released)
```

Ensure relayer01 is connected to executor01 and executor01 is available on the mesh.

```bash
[root@c01 /]# receptorctl status
Node ID: controller01
Version: 1.4.4
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
controller01 control   Stream     2024-02-21 13:54:42   {'type': 'Control Service'}
relayer01    control   Stream     2024-02-21 13:54:42   {'type': 'Control Service'}
executor01   control   Stream     2024-02-21 13:54:42   {'type': 'Control Service'}

Node         Secure Work Types
controller01 receptor-gather-config
relayer01    receptor-gather-config, receptor-patch-config, receptor-reload-config
executor01   please-introduce-yourself
```

Ensure the work on executor01 is executable.

```bash
[root@c01 /]# receptorctl work submit --node executor01 --rm --follow --no-payload --signwork please-introduce-yourself
Hi, Im executor01, Im just now joined the mesh!
(y7Myp0Hp, released)
```
