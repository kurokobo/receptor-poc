# Demo 02: Invoking commands with parameters and payloads

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
[root@node01 /]# receptorctl work submit \
  --node executor01 \
  --rm \
  --follow \
  --no-payload \
  cat-files -- /etc/hostname /etc/os-release
/etc/hostname:1:node03.example.internal
/etc/os-release:1:NAME="CentOS Stream"
...
(WzjhPLJ0, released)
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
