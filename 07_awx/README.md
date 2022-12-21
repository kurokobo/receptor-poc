<!-- omit in toc -->
# Demo 07: Receptor in AWX and Automation Mesh

As of **AWX 21.10.1**.

<!-- omit in toc -->
## Table of Contents

- [Ansible Runner](#ansible-runner)
  - [Run playbook](#run-playbook)
  - [Remote job execution](#remote-job-execution)
    - [Transmit](#transmit)
    - [Worker](#worker)
    - [Process](#process)
  - [Use Execution Environment (EE)](#use-execution-environment-ee)
    - [Use public EE image](#use-public-ee-image)
    - [Use private EE image](#use-private-ee-image)
    - [Use EE with remote job execution](#use-ee-with-remote-job-execution)
- [AWX](#awx)
  - [Receptor configuration](#receptor-configuration)
    - [Receptor configuration for AWX](#receptor-configuration-for-awx)
    - [Receptor configuration for Execution Node](#receptor-configuration-for-execution-node)
  - [Deep dive](#deep-dive)
    - [Job execution with Container Group ("InCluster" authmethod)](#job-execution-with-container-group-incluster-authmethod)
    - [Job execution with Container Group ("Runtime" authmethod)](#job-execution-with-container-group-runtime-authmethod)
    - [Job execution with Instance Group](#job-execution-with-instance-group)
    - [Project update](#project-update)

## Ansible Runner

### Run playbook

```bash
cd 07_awx/runner
```

```bash
$ tree .
.
├── inventory
│   └── hosts
└── project
    └── demo.yml

2 directories, 2 files
```

```bash
$ ansible-runner run . -p demo.yml
...
TASK [ansible.builtin.debug] ***************************************************
ok: [localhost] => {
    "dump": {
        "hostname": "kuro-awx01",
        "user_dir": "/home/kuro",
        "user_id": "kuro",
        "virtualization_type": "VMware"
    }
}
...
```

### Remote job execution

```bash
cd 07_awx/runner_remote
```

#### Transmit

```bash
$ ansible-runner transmit . -p demo.yml | tee transmit.log
{"kwargs": {"ident": "81e67a815f6d4acc95446101ab2d0e3e", ...
{"zipfile": 1246}
UEsDBBQAAAAIAFEplFX/yIRIMQAAAEYAAAAKAAAALmdpdGlnbm9yZdPiU...{"eof": true}

$ cat transmit.log | head -n 1 | jq
{
  "kwargs": {
    ...
    "playbook": "demo.yml",
    ...
```

```bash
$ grep -A 1 zipfile transmit.log | tail -n 1 | sed 's/{"eof": true}//g' | base64 -d > transmit.zip
$ unzip -q transmit.zip -d transmit
$ ls -l transmit
total 4
drwxrwxr-x. 2 kuro kuro  19 Dec 14 04:52 inventory
drwxrwxr-x. 2 kuro kuro  22 Dec 14 04:52 project
-rw-rw-r--. 1 kuro kuro 819 Dec 14 05:16 transmit.log
```

#### Worker

```bash
$ cat transmit.log | ansible-runner worker | tee worker.log
...
{"uuid": "9ae64665-6fd2-95e0-564f-000000000006", "counter": 2, "stdout": "\r\nPLAY [localhost] ***************************************************************", ...
...
{"uuid": "c764303b-8118-42fc-9731-d8d60ad94ec0", "counter": 14, "stdout": "\r\nPLAY RECAP *********************************************************************...
...
{"zipfile": 11202}
UEsDBBQAAAAIALUwlVXNDA0KawYAAMARAAAHAAAAY29tbWFuZL...{"eof": true}
```

```bash
$ grep -A 1 zipfile worker.log | tail -n 1 | sed 's/{"eof": true}//g' | base64 -d > worker.zip
$ unzip -q worker.zip -d worker
$ ls -l worker
total 20
-rw-------. 1 kuro kuro 4544 Dec 14 20:49 command
drwxr-xr-x. 2 kuro kuro   23 Dec 14 20:49 fact_cache
drwx------. 2 kuro kuro    6 Dec 14 20:50 job_events
-rw-------. 1 kuro kuro    1 Dec 14 20:50 rc
-rw-------. 1 kuro kuro   10 Dec 14 20:50 status
-rw-rw-r--. 1 kuro kuro    0 Dec 14 20:49 stderr
-rw-------. 1 kuro kuro  966 Dec 14 20:50 stdout

$ cat worker/command | jq
{
  "command": [
    "ansible-playbook",
    "-i",
    "/tmp/tmppub6q_87/inventory",
    "demo.yml"
  ],
  "cwd": "/tmp/tmppub6q_87/project",
  ...

$ cat worker/stdout 
...
TASK [ansible.builtin.debug] ***************************************************
ok: [localhost] => {
    "dump": {
        "hostname": "kuro-awx01",
        "user_dir": "/home/kuro",
        "user_id": "kuro",
        "virtualization_type": "VMware"
    }
}
...
```

#### Process

```bash
$ cat worker.log | ansible-runner process ./process
...
TASK [ansible.builtin.debug] ***************************************************
ok: [localhost] => {
    "dump": {
        "hostname": "kuro-awx01",
        "user_dir": "/home/kuro",
        "user_id": "kuro",
        "virtualization_type": "VMware"
    }
}
...
```

```bash
$ ls -l process/artifacts
total 32
-rw-------. 1 kuro kuro 4544 Dec 14 20:49 command
drwxr-xr-x. 2 kuro kuro   23 Dec 14 21:18 fact_cache
drwx------. 2 kuro kuro 4096 Dec 14 21:18 job_events
-rw-------. 1 kuro kuro    1 Dec 14 20:50 rc
-rw-------. 1 kuro kuro   10 Dec 14 20:50 status
-rw-rw-r--. 1 kuro kuro    0 Dec 14 20:49 stderr
-rw-------. 1 kuro kuro  966 Dec 14 20:50 stdout
```

### Use Execution Environment (EE)

```bash
cd 07_awx/runner_ee
```

#### Use public EE image

```bash
$ ansible-runner run . -p demo.yml \
  --process-isolation \
  --container-image quay.io/ansible/awx-ee:latest \
  --container-option="--user=root"
...
TASK [ansible.builtin.debug] ***************************************************
ok: [localhost] => {
    "dump": {
        "hostname": "a02f803a5615",
        "user_dir": "/root",
        "user_id": "root",
        "virtualization_type": "container"
    }
}
...
```

```bash
$ podman ps
CONTAINER ID  IMAGE                          COMMAND          ...
a02f803a5615  quay.io/ansible/awx-ee:latest  ansible-playbook ...

$ podman inspect a02f803a5615
...
     "Config": {
          ...
          "Cmd": [
               "ansible-playbook",
               "-i",
               "/runner/inventory/hosts",
               "demo.yml"
          ],
          "Image": "quay.io/ansible/awx-ee:latest",
          ...
```

#### Use private EE image

```bash
$ cat env/settings
---
process_isolation: true
container_image: registry.example.com/ansible/ee:2.12-custom
container_auth_data:
  host: registry.example.com
  username: reguser
  password: Registry123!
  verify_ssl: false
```

```bash
$ ansible-runner run . -p demo.yml
...
TASK [ansible.builtin.debug] ***************************************************
ok: [localhost] => {
    "dump": {
        "hostname": "50e720f8ae7b",
        "user_dir": "/root",
        "user_id": "root",
        "virtualization_type": "container"
    }
}
...

$ podman events --filter event=pull --since 1h
...
2022-12-14 22:00:21.372323245 +0000 UTC image pull  registry.example.com/ansible/ee:2.12-custom
...

$ podman ps
CONTAINER ID  IMAGE                                        COMMAND          ...
50e720f8ae7b  registry.example.com/ansible/ee:2.12-custom  ansible-playbook ...
```

```bash
$ podman pull registry.example.com/ansible/ee:2.12-custom
Trying to pull registry.example.com/ansible/ee:2.12-custom...
Error: initializing source docker://registry.example.com/ansible/ee:2.12-custom: pinging container registry registry.example.com: Get "https://registry.example.com/v2/": x509: certificate signed by unknown authority
```

```bash
$ ps -ef | grep "podman run"
kuro       10964   10963  5 22:54 pts/1    00:00:08
/usr/bin/podman run
  ...
  --authfile=/tmp/ansible_runner_registry_668a4418-dafa-447b-9e76-e5adced33e6f_6x83947w/auth.json
  ...
  registry.example.com/ansible/ee:2.12-custom
  ansible-playbook -i /runner/inventory/hosts demo.yml

$ cat /tmp/ansible_runner_registry_668a4418-dafa-447b-9e76-e5adced33e6f_6x83947w/auth.json
{
    "auths": {
        "registry.example.com": {
            "auth": "cmVndXNlcjpSZWdpc3RyeTEyMyE="
        }
    }
}

$ cat /tmp/ansible_runner_registry_668a4418-dafa-447b-9e76-e5adced33e6f_6x83947w/auth.json \
  | jq -r '.auths."registry.example.com".auth' | base64 -d
reguser:Registry123!
```

```bash
$ ps -ef | grep "podman run"
kuro       10964   10963  5 22:54 pts/1    00:00:08 ...

$ cat /proc/10964/environ --show-nonprinting | sed 's/\^@/\n/g' | grep REGISTRIES
CONTAINERS_REGISTRIES_CONF=/tmp/ansible_runner_registry_668a4418-dafa-447b-9e76-e5adced33e6f_6x83947w/registries.conf
REGISTRIES_CONFIG_PATH=/tmp/ansible_runner_registry_668a4418-dafa-447b-9e76-e5adced33e6f_6x83947w/registries.conf

$ cat /tmp/ansible_runner_registry_668a4418-dafa-447b-9e76-e5adced33e6f_6x83947w/registries.conf
[[registry]]
location = "registry.example.com"
insecure = true
```

#### Use EE with remote job execution

```bash
$ ansible-runner transmit . -p demo.yml | ansible-runner worker | tee worker.log
...

$ grep -A 1 zipfile worker.log | tail -n 1 | sed 's/{"eof": true}//g' | base64 -d > worker.zip
$ unzip -q worker.zip -d worker

$ cat worker/ansible_version.txt 
ansible [core 2.12.5.post0]

$ cat worker/collections.json | jq
{
  "/usr/share/ansible/collections/ansible_collections": {
    "community.general": {
      "version": "6.0.0"
    ...

$ cat worker/command | jq
{
  "command": [
    "podman",
    "run",
    ...
    "--authfile=/tmp/ansible_runner_registry_30044782759d4ba9a5f8d787ce571cb0_2ju3l7gd/auth.json",
    ...
    "registry.example.com/ansible/ee:2.12-custom",
    "ansible-playbook",
    "-i",
    "/runner/inventory/hosts",
    "demo.yml"
  ],
  ...
  "env": {
    ...
    "CONTAINERS_REGISTRIES_CONF": "/tmp/ansible_runner_registry_30044782759d4ba9a5f8d787ce571cb0_2ju3l7gd/registries.conf",
    "REGISTRIES_CONFIG_PATH": "/tmp/ansible_runner_registry_30044782759d4ba9a5f8d787ce571cb0_2ju3l7gd/registries.conf"
  }
}
```

## AWX

### Receptor configuration

#### Receptor configuration for AWX

```yaml
- local-only: null

- log-level: debug

- node:
    firewallrules:
    - action: reject
      tonode: awx-bdfc84dd7-wp2b6
      toservice: control

- control-service:
    filename: /var/run/receptor/receptor.sock
    permissions: '0660'
    service: control

- work-command:
    allowruntimeparams: true
    command: ansible-runner
    params: worker
    worktype: local

- work-signing:
    privatekey: /etc/receptor/signing/work-private-key.pem
    tokenexpiration: 1m

- work-kubernetes:
    allowruntimeauth: true
    allowruntimeparams: true
    allowruntimepod: true
    authmethod: runtime
    worktype: kubernetes-runtime-auth

- work-kubernetes:
    allowruntimeauth: true
    allowruntimeparams: true
    allowruntimepod: true
    authmethod: incluster
    worktype: kubernetes-incluster-auth

- tls-client:
    cert: /etc/receptor/tls/receptor.crt
    key: /etc/receptor/tls/receptor.key
    name: tlsclient
    rootcas: /etc/receptor/tls/ca/receptor-ca.crt
```

```yaml
- tcp-peer:
    address: exec01.ansible.internal:27199
    tls: tlsclient
```

#### Receptor configuration for Execution Node

```yaml
---
- node:
    id: exec01.ansible.internal

- work-verification:
    publickey: /etc/receptor/work_public_key.pem

- log-level: info

- control-service:
    service: control
    filename: /var/run/receptor/receptor.sock
    permissions: 0660
    tls: tls_server

- tls-server:
    name: tls_server
    cert: /etc/receptor/tls/exec01.ansible.internal.crt
    key: /etc/receptor/tls/exec01.ansible.internal.key
    clientcas: /etc/receptor/tls/ca/mesh-CA.crt
    requireclientcert: true
    mintls13: False

- tls-client:
    name: tls_client
    cert: /etc/receptor/tls/exec01.ansible.internal.crt
    key: /etc/receptor/tls/exec01.ansible.internal.key
    rootcas: /etc/receptor/tls/ca/mesh-CA.crt
    insecureskipverify: false
    mintls13: False

- tcp-listener:
    port: 27199
    tls: tls_server

- work-command:
    worktype: ansible-runner
    command: ansible-runner
    params: worker
    allowruntimeparams: True
    verifysignature: True
```

### Deep dive

```bash
cd 07_awx/awx
```

#### Job execution with Container Group ("InCluster" authmethod)

```bash
$ kubectl -n awx exec deployment/awx -c awx-ee -- bash -c "tar zcf - /tmp/receptor/*/*" \
  | tar zxvf - --strip-components 4 -C cg_incluster
tmp/receptor/awx-bdfc84dd7-wp2b6/eq4pkWHs/status.lock
tmp/receptor/awx-bdfc84dd7-wp2b6/eq4pkWHs/status
tmp/receptor/awx-bdfc84dd7-wp2b6/eq4pkWHs/stdin
tmp/receptor/awx-bdfc84dd7-wp2b6/eq4pkWHs/stdout
```

```bash
$ cat cg_incluster/status | jq
{
  ...
  "WorkType": "kubernetes-incluster-auth",
  "ExtraData": {
    ...
    "KubePod": "---\napiVersion: v1\n...",
    ...
  }
}

$ cat cg_incluster/status | jq -r '.ExtraData.KubePod'
---
apiVersion: v1
kind: Pod
...
spec:
  ...
  containers:
  - args:
    - ansible-runner
    - worker
    - --private-data-dir=/runner
    image: registry.example.com/ansible/ee:2.12-custom
    imagePullPolicy: IfNotPresent
    name: worker
    ...
  imagePullSecrets:
  - name: automation-1568d-image-pull-secret-3
  serviceAccountName: default
```

```bash
$ kubectl -n awx get secret automation-1568d-image-pull-secret-3 -o yaml
apiVersion: v1
data:
  .dockerconfigjson: ewogICAgImF1dGhzIjogewogICAgICAgICJyZWdpc3RyeS5leGF...
kind: Secret
...
type: kubernetes.io/dockerconfigjson

$ kubectl -n awx get secret automation-1568d-image-pull-secret-3 -o jsonpath='{$.data.\.dockerconfigjson}' | base64 -d
{
    "auths": {
        "registry.example.com": {
            "auth": "cmVndXNlcjpSZWdpc3RyeTEyMyE="
        }
    }
}

$ kubectl -n awx get secret automation-1568d-image-pull-secret-3 -o jsonpath='{$.data.\.dockerconfigjson}' | base64 -d \
  | jq -r '.auths."registry.example.com".auth' | base64 -d
reguser:Registry123!
```

```bash
$ cat cg_incluster/stdin
{"kwargs": {"ident": 19, "playbook": "demo.yml", ...
{"zipfile": 1839}
UEsDBBQAAAAAABCnjlUAAAAAAAAAAAAAAAAKAAAAaW52ZW50b3J5...{"eof": true}

$ cat cg_incluster/stdin | head -n 1 | jq
{
  "kwargs": {
    ...
    "playbook": "demo.yml",
    "inventory": "inventory/hosts",
    "passwords": {
      ...
      "sudo password.*:\\s*?$": "Root123!",
      ...
      "SSH password:\\s*?$": "User123!",
      ...
    },
    "suppress_env_files": true,
    "envvars": {
      ...
      "ANSIBLE_SSH_CONTROL_PATH_DIR": "/runner/cp",
      "ANSIBLE_COLLECTIONS_PATHS": "/runner/requirements_collections:~/.ansible/collections:/usr/share/ansible/collections",
      "ANSIBLE_ROLES_PATH": "/runner/requirements_roles:~/.ansible/roles:/usr/share/ansible/roles:/etc/ansible/roles"
    },
    ...
```

```bash
$ grep -A 1 zipfile cg_incluster/stdin | tail -n 1 | sed 's/{"eof": true}//g' | base64 -d > cg_incluster/stdin.zip
$ unzip -q cg_incluster/stdin.zip -d cg_incluster/stdin_
$ ls -l cg_incluster/stdin_
total 0
drwx------. 2 kuro kuro  6 Dec 14 20:56 cp
drwxr-xr-x. 2 kuro kuro 54 Dec 14 20:56 env
drwxr-xr-x. 2 kuro kuro 19 Dec 14 20:56 inventory
drwxrwxr-x. 2 kuro kuro 22 Dec 14 20:13 project

$ python3 cg_incluster/stdin_/inventory/hosts | jq
{
  "all": {
    "hosts": [
      "localhost"
    ],
    ...
  },
  ...
  "_meta": {
    "hostvars": {
      ...
      "localhost": {
        "ansible_connection": "local",
        "ansible_python_interpreter": "{{ ansible_playbook_python }}",
        ...
      }
    }
  }
}
```

#### Job execution with Container Group ("Runtime" authmethod)

```bash
$ kubectl -n awx exec deployment/awx -c awx-ee -- bash -c "tar zcf - /tmp/receptor/*/*" \
  | tar zxvf - --strip-components 4 -C cg_runtime
tmp/receptor/awx-bdfc84dd7-wp2b6/aIefUYrp/status.lock
tmp/receptor/awx-bdfc84dd7-wp2b6/aIefUYrp/status
tmp/receptor/awx-bdfc84dd7-wp2b6/aIefUYrp/stdin
tmp/receptor/awx-bdfc84dd7-wp2b6/aIefUYrp/stdout
```

```bash
$ cat cg_runtime/status | jq
{
  ...
  "WorkType": "kubernetes-runtime-auth",
  "ExtraData": {
    ...
    "KubeConfig": "---\napiVersion: v1\n...",
    ...
    "KubePod": "---\napiVersion: v1\n...",
    ...
  }
}

$ cat cg_runtime/status | jq -r '.ExtraData.KubeConfig'
---
apiVersion: v1
clusters:
- cluster:
    insecure-skip-tls-verify: true
    server: https://192.168.0.219:6443
  ...
contexts:
- context:
    cluster: https://192.168.0.219:6443
    namespace: awx
    user: https://192.168.0.219:6443
  ...
current-context: https://192.168.0.219:6443
kind: Config
preferences: {}
users:
- name: https://192.168.0.219:6443
  user:
    token: eyJhbGciOiJSUzI1NiIsImtpZCI6IkI1LXotczRZN1RU...
```

```bash
$ cat cg_runtime/stdin | head -n 1 | jq
{
  "kwargs": {
    ...
    "playbook": "demo.yml",
    "inventory": "inventory/hosts",
    "passwords": {
      ...

$ grep -A 1 zipfile cg_runtime/stdin | tail -n 1 | sed 's/{"eof": true}//g' | base64 -d > cg_runtime/stdin.zip
$ unzip -q cg_runtime/stdin.zip -d cg_runtime/stdin_
$ ls -l cg_runtime/stdin_
total 0
drwx------. 2 kuro kuro  6 Dec 14 20:54 cp
drwxr-xr-x. 2 kuro kuro 54 Dec 14 20:54 env
drwxr-xr-x. 2 kuro kuro 19 Dec 14 20:54 inventory
drwxrwxr-x. 2 kuro kuro 22 Dec 14 20:13 project
```

#### Job execution with Instance Group

```bash
$ kubectl -n awx exec deployment/awx -c awx-ee -- bash -c "tar zcf - /tmp/receptor/*/*" \
  | tar zxvf - --strip-components 4 -C ig_podman
tmp/receptor/awx-bdfc84dd7-wp2b6/tPqM2J8t/status.lock
tmp/receptor/awx-bdfc84dd7-wp2b6/tPqM2J8t/status
tmp/receptor/awx-bdfc84dd7-wp2b6/tPqM2J8t/stdin
tmp/receptor/awx-bdfc84dd7-wp2b6/tPqM2J8t/stdout
```

```bash
$ cat ig_podman/status | jq
{
  ...
  "WorkType": "remote",
  "ExtraData": {
    "RemoteNode": "exec01.ansible.internal",
    "RemoteWorkType": "ansible-runner",
    "RemoteParams": {
      "params": "--private-data-dir=/tmp/awx_17_3jjx2zyd --delete"
    },
    ...
    "SignWork": true,
    "TLSClient": "tlsclient",
    ...
  }
}
```

```bash
$ cat ig_podman/stdin | head -n 1 | jq
{
  "kwargs": {
    ...
    "playbook": "demo.yml",
    "inventory": "inventory/hosts",
    "passwords": {
      ...
      "sudo password.*:\\s*?$": "Root123!",
      ...
      "SSH password:\\s*?$": "User123!",
      ...
    },
    ...
    "envvars": {
      ...
    },
    ...
    "container_image": "registry.example.com/ansible/ee:2.12-custom",
    "process_isolation": true,
    "process_isolation_executable": "podman",
    "container_options": [
      "--user=root",
      ...
      "--pull=missing"
    ],
    "container_auth_data": {
      "host": "registry.example.com",
      "username": "reguser",
      "password": "Registry123!",
      "verify_ssl": false
    },
    ...
  }
}
```

```bash
$ grep -A 1 zipfile ig_podman/stdin | tail -n 1 | sed 's/{"eof": true}//g' | base64 -d > ig_podman/stdin.zip
$ unzip -q ig_podman/stdin.zip -d ig_podman/stdin_
$ ls -l ig_podman/stdin_
total 0
drwx------. 2 kuro kuro  6 Dec 14 20:42 cp
drwxr-xr-x. 2 kuro kuro 54 Dec 14 20:42 env
drwxr-xr-x. 2 kuro kuro 19 Dec 14 20:42 inventory
drwxrwxr-x. 2 kuro kuro 22 Dec 14 20:13 project
```

#### Project update

```bash
$ kubectl -n awx exec deployment/awx -c awx-ee -- bash -c "tar zcf - /tmp/receptor/*/*" \
  | tar zxvf - --strip-components 4 -C sync_pj
tmp/receptor/awx-bdfc84dd7-wp2b6/YsoGXshO/status.lock
tmp/receptor/awx-bdfc84dd7-wp2b6/YsoGXshO/status
tmp/receptor/awx-bdfc84dd7-wp2b6/YsoGXshO/stdin
tmp/receptor/awx-bdfc84dd7-wp2b6/YsoGXshO/stdout

$ cat sync_pj/status | jq
{
  ...
  "WorkType": "local",
  "ExtraData": {
    ...
    "Params": "worker --private-data-dir=/tmp/awx_24_80ydyxhs"
  }
}

$ cat sync_pj/stdin | head -n 1 | jq
{
  "kwargs": {
    ...
    "playbook": "project_update.yml",
    "inventory": "inventory/hosts",
    "passwords": {
      ...
      "Password:\\s*?$": "Git123!",
      ...
    },
    ...
  }
}
```
