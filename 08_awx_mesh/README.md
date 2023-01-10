<!-- omit in toc -->
# Demo 08: Enable Automation Mesh with Hop Nodes in AWX

As of **AWX 21.10.2**.

## ⚠️ IMPORTANT NOTICE ⚠️

**This is for testing purposes only.**

**Currently, AWX does not support Automation Mesh including Hop Nodes. This is truly PoC intended to improve technical understanding of AWX and Receptor.**

<!-- omit in toc -->
## Table of Contents

- [⚠️ IMPORTANT NOTICE ⚠️](#️-important-notice-️)
- [Register instances and its peers](#register-instances-and-its-peers)
  - [Register nodes](#register-nodes)
  - [Register peers for each nodes (Optional)](#register-peers-for-each-nodes-optional)
- [Modify AWX to force to use customized `receptor.conf`](#modify-awx-to-force-to-use-customized-receptorconf)
  - [Create config map](#create-config-map)
  - [Deploy AWX with extra volumes](#deploy-awx-with-extra-volumes)
  - [Correct peers for Control Node](#correct-peers-for-control-node)
- [Deploy execution nodes](#deploy-execution-nodes)
- [Deploy hop nodes](#deploy-hop-nodes)
- [Review](#review)
- [Appendix: Windows-based Hop Nodes](#appendix-windows-based-hop-nodes)

## Register instances and its peers

### Register nodes

```bash
EXEC_NODES="
exec01.ansible.internal
exec02.ansible.internal
exec03.ansible.internal
"

HOP_NODES="
hop01.ansible.internal
hop02.ansible.internal
hop03.ansible.internal
hop04.ansible.internal
"

for exec in ${EXEC_NODES}; do
  kubectl -n awx exec -it deployment/awx -c awx-task -- awx-manage provision_instance --hostname ${exec} --node_type execution
done

for hop in ${HOP_NODES}; do
  kubectl -n awx exec -it deployment/awx -c awx-task -- awx-manage provision_instance --hostname ${hop} --node_type hop
done
```

Execution nodes are automatically added as peers for AWX itself.

```bash
$ kubectl -n awx exec -it deployment/awx -c awx-ee -- cat /etc/receptor/receptor.conf
...
- tcp-peer:
    address: exec01.ansible.internal:27199
    tls: tlsclient
- tcp-peer:
    address: exec02.ansible.internal:27199
    tls: tlsclient
- tcp-peer:
    address: exec03.ansible.internal:27199
    tls: tlsclient
```

### Register peers for each nodes (Optional)

This step is optional. Perform this step if you want to correctly display `Topology View` and `Peers` for each `Instances` in the Web UI.

Also, since `Peers` for AWX itself are reset each time after recreation of the Pod, so this step should be performed repeatedly if you want to keep the Web UI correct.

```bash
CONTROL_NODE=$(kubectl -n awx get pod --no-headers -o custom-columns=:metadata.name -l app.kubernetes.io/name=awx)
kubectl -n awx exec -it deployment/awx -c awx-task -- awx-manage register_peers ${CONTROL_NODE} --exact hop01.ansible.internal hop02.ansible.internal hop03.ansible.internal

kubectl -n awx exec -it deployment/awx -c awx-task -- awx-manage register_peers hop01.ansible.internal --peers exec01.ansible.internal exec02.ansible.internal
kubectl -n awx exec -it deployment/awx -c awx-task -- awx-manage register_peers hop02.ansible.internal --peers exec01.ansible.internal exec02.ansible.internal
kubectl -n awx exec -it deployment/awx -c awx-task -- awx-manage register_peers hop03.ansible.internal --peers hop04.ansible.internal
kubectl -n awx exec -it deployment/awx -c awx-task -- awx-manage register_peers hop04.ansible.internal --peers exec03.ansible.internal
```

Currently, modifying peers for each nodes updates database only. Feature to modify `receptor.conf` for AWX automatically during registering peers is not implemented in current AWX. This is why we have to override `receptor.conf` for AWX in the later steps.

```bash
$ kubectl -n awx exec -it deployment/awx -c awx-ee -- cat /etc/receptor/receptor.conf
...
- tcp-peer:
    address: exec01.ansible.internal:27199
    tls: tlsclient
- tcp-peer:
    address: exec02.ansible.internal:27199
    tls: tlsclient
- tcp-peer:
    address: exec03.ansible.internal:27199
    tls: tlsclient
```

## Modify AWX to force to use customized `receptor.conf`

### Create config map

```bash
$ kubectl -n awx create configmap awx-custom-receptor-config --from-file=receptor.conf
configmap/awx-custom-receptor-config created

$ kubectl -n awx get configmap awx-custom-receptor-config -o yaml
apiVersion: v1
data:
  receptor.conf: |
    ...
    - node:
        firewallrules:
          - action: reject
            tonode: /awx-[0-9a-z]{10}-[0-9a-z]{5}/
            toservice: control
    ...
    - tcp-peer:
        address: hop01.ansible.internal:27199
        tls: tlsclient
    - tcp-peer:
        address: hop02.ansible.internal:27199
        tls: tlsclient
    - tcp-peer:
        address: hop03.ansible.internal:27199
        tls: tlsclient
...
```

### Deploy AWX with extra volumes

Append `extra_volumes`, `task_extra_volume_mounts`, and `ee_extra_volume_mounts` to your `spec` of AWX.

```yaml
---
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
spec:
  ...
  extra_volumes: |
    - name: awx-dummy-receptor-config
      emptyDir: {}
    - name: awx-custom-receptor-config
      configMap:
        name: awx-custom-receptor-config

  task_extra_volume_mounts: |
    - name: awx-dummy-receptor-config
      mountPath: /etc/receptor

  ee_extra_volume_mounts: |
    - name: awx-custom-receptor-config
      mountPath: /etc/receptor/receptor.conf
      subPath: receptor.conf
```

```bash
$ kubectl -n awx exec -it deployment/awx -c awx-task -- cat /etc/receptor/receptor.conf
...
- tcp-peer:
    address: exec01.ansible.internal:27199
    tls: tlsclient
- tcp-peer:
    address: exec02.ansible.internal:27199
    tls: tlsclient
- tcp-peer:
    address: exec03.ansible.internal:27199
    tls: tlsclient

$ kubectl -n awx exec -it deployment/awx -c awx-ee -- cat /etc/receptor/receptor.conf
...
- tcp-peer:
    address: hop01.ansible.internal:27199
    tls: tlsclient
- tcp-peer:
    address: hop02.ansible.internal:27199
    tls: tlsclient
- tcp-peer:
    address: hop03.ansible.internal:27199
    tls: tlsclient
```

### Correct peers for Control Node

```bash
CONTROL_NODE=$(kubectl -n awx get pod --no-headers -o custom-columns=:metadata.name -l app.kubernetes.io/name=awx)
kubectl -n awx exec -it deployment/awx -c awx-task -- awx-manage register_peers ${CONTROL_NODE} --exact hop01.ansible.internal hop02.ansible.internal hop03.ansible.internal
```

## Deploy execution nodes

Nothing special, just download install bundle from Web UI and execute it.

## Deploy hop nodes

```bash
HOP_NODES="
hop01.ansible.internal
hop02.ansible.internal
hop03.ansible.internal
hop04.ansible.internal
"

# Copy CA key from awx-task to awx-ee
kubectl -n awx exec deployment/awx -c awx-web -- cat /etc/receptor/tls/ca/receptor-ca.key | kubectl -n awx exec -i deployment/awx -c awx-ee -- bash -c "cat > /tmp/receptor-ca.key"

# Generate certificate and key for each hop nodes and copy them to local directory
for node in ${HOP_NODES}; do
  kubectl -n awx exec deployment/awx -c awx-ee -- mkdir -p /tmp/${node}
  kubectl -n awx exec deployment/awx -c awx-ee -- receptor --cert-makereq bits=2048 commonname=${node} dnsname=${node} nodeid=${node} outreq=/tmp/${node}/receptor.csr outkey=/tmp/${node}/receptor.key
  kubectl -n awx exec deployment/awx -c awx-ee -- receptor --cert-signreq req=/tmp/${node}/receptor.csr cacert=/etc/receptor/tls/ca/receptor-ca.crt cakey=/tmp/receptor-ca.key outcert=/tmp/${node}/receptor.crt verify=true

  mkdir -p ${node}
  kubectl -n awx exec deployment/awx -c awx-ee -- bash -c "tar zcf - /tmp/${node}" | tar zxvf - --strip-components 2 -C ${node}
  kubectl -n awx exec deployment/awx -c awx-ee -- cat /etc/receptor/tls/ca/receptor-ca.crt > ${node}/receptor-ca.crt
done

# Remove key on awx-ee
kubectl -n awx exec deployment/awx -c awx-ee -- rm /tmp/receptor-ca.key
```

```bash
$ tree hop*
hop01.ansible.internal
├── receptor-ca.crt
├── receptor.crt
├── receptor.csr
└── receptor.key
hop02.ansible.internal
├── receptor-ca.crt
├── receptor.crt
├── receptor.csr
└── receptor.key
hop03.ansible.internal
├── receptor-ca.crt
├── receptor.crt
├── receptor.csr
└── receptor.key
hop04.ansible.internal
├── receptor-ca.crt
├── receptor.crt
├── receptor.csr
└── receptor.key

0 directories, 16 files
```

```bash
ansible-playbook -i inventory.yaml install_hop_nodes.yaml
```

```bash
$ ssh root@hop01.ansible.internal -- cat /etc/receptor/receptor.conf
---
- node:
    id: hop01.ansible.internal
...
- tcp-peer:
    address: exec01.ansible.internal:27199
    redial: true
    tls: tls_client
- tcp-peer:
    address: exec02.ansible.internal:27199
    redial: true
    tls: tls_client

$ ssh root@hop01.ansible.internal -- systemctl status receptor
● receptor.service - Receptor
   Loaded: loaded (/usr/lib/systemd/system/receptor.service; enabled; vendor preset: disabled)
   ...
```

## Review

```bash
kubectl -n awx exec -it deployment/awx -c awx-ee -- pip install receptorctl
kubectl -n awx exec -it deployment/awx -c awx-ee -- /home/runner/.local/bin/receptorctl --socket /var/run/receptor/receptor.sock status
```

```bash
$ kubectl -n awx exec -it deployment/awx -c awx-ee -- /home/runner/.local/bin/receptorctl --socket /var/run/receptor/receptor.sock status
...

Connection              Cost
hop01.ansible.internal  1
hop02.ansible.internal  1
hop03.ansible.internal  1

Known Node              Known Connections
awx-8597d86894-czg7l    hop01.ansible.internal: 1 hop02.ansible.internal: 1 hop03.ansible.internal: 1
exec01.ansible.internal hop01.ansible.internal: 1 hop02.ansible.internal: 1
exec02.ansible.internal hop01.ansible.internal: 1 hop02.ansible.internal: 1
exec03.ansible.internal hop04.ansible.internal: 1
hop01.ansible.internal  awx-8597d86894-czg7l: 1 exec01.ansible.internal: 1 exec02.ansible.internal: 1
hop02.ansible.internal  awx-8597d86894-czg7l: 1 exec01.ansible.internal: 1 exec02.ansible.internal: 1
hop03.ansible.internal  awx-8597d86894-czg7l: 1 hop04.ansible.internal: 1
hop04.ansible.internal  exec03.ansible.internal: 1 hop03.ansible.internal: 1

Route                   Via
exec01.ansible.internal hop01.ansible.internal
exec02.ansible.internal hop01.ansible.internal
exec03.ansible.internal hop03.ansible.internal
hop01.ansible.internal  hop01.ansible.internal
hop02.ansible.internal  hop02.ansible.internal
hop03.ansible.internal  hop03.ansible.internal
hop04.ansible.internal  hop03.ansible.internal

Node                    Service   Type       Last Seen             Tags
hop01.ansible.internal  control   StreamTLS  2022-12-20 01:07:48   {'type': 'Control Service'}
hop02.ansible.internal  control   StreamTLS  2022-12-20 01:07:51   {'type': 'Control Service'}
hop03.ansible.internal  control   StreamTLS  2022-12-20 01:07:48   {'type': 'Control Service'}
hop04.ansible.internal  control   StreamTLS  2022-12-20 01:07:46   {'type': 'Control Service'}
exec01.ansible.internal control   StreamTLS  2022-12-20 01:08:10   {'type': 'Control Service'}
exec02.ansible.internal control   StreamTLS  2022-12-20 01:08:19   {'type': 'Control Service'}
exec03.ansible.internal control   StreamTLS  2022-12-20 01:08:37   {'type': 'Control Service'}
awx-8597d86894-czg7l    control   Stream     2022-12-19 16:08:38   {'type': 'Control Service'}

Node                    Work Types
awx-8597d86894-czg7l    local, kubernetes-runtime-auth, kubernetes-incluster-auth

Node                    Secure Work Types
exec01.ansible.internal ansible-runner
exec02.ansible.internal ansible-runner
exec03.ansible.internal ansible-runner
```

## Appendix: Windows-based Hop Nodes

Build `receptor.exe` and run it with following example configuration file.

```bash
# Build EXE on Linux
GOOS=windows go build -o receptor.exe ./cmd/receptor-cl

# Build EXE on Windows
go build -o receptor.exe .\cmd\receptor-cl
```

```powershell
.\receptor.exe -c .\receptor.conf
```

```yaml
---
- node:
    id: hop01.ansible.internal
- log-level: info
- control-service:
    service: control
    tls: tls_server
- tls-server:
    name: tls_server
    cert: C:\receptor\receptor.crt
    key: C:\receptor\receptor.key
    clientcas: C:\receptor\receptor-ca.crt
    requireclientcert: true
    mintls13: False
- tls-client:
    name: tls_client
    cert: C:\receptor\receptor.crt
    key: C:\receptor\receptor.key
    rootcas: C:\receptor\receptor-ca.crt
    insecureskipverify: false
    mintls13: False
- tcp-listener:
    port: 27199
    tls: tls_server
- tcp-peer:
    address: exec01.ansible.internal:27199
    redial: true
    tls: tls_client
- tcp-peer:
    address: exec02.ansible.internal:27199
    redial: true
    tls: tls_client
```
