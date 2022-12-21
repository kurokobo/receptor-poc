<!-- omit in toc -->
# Demo 06: Encryption, firewall, work signing

<!-- omit in toc -->
## Table of Contents

- [Preparation](#preparation)
- [Case 01: Allowed peers](#case-01-allowed-peers)
- [Case 02-1: Encryption / Below-the-mesh encryption](#case-02-1-encryption--below-the-mesh-encryption)
- [Case 02-2: Encryption / Above-the-mesh encryption](#case-02-2-encryption--above-the-mesh-encryption)
- [Case 03: Firewall](#case-03-firewall)
- [Case 04: Work signing](#case-04-work-signing)

## Preparation

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

## Case 01: Allowed peers

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

## Case 02-1: Encryption / Below-the-mesh encryption

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

## Case 02-2: Encryption / Above-the-mesh encryption

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

## Case 03: Firewall

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
[root@c01 /]# receptorctl ping executor01
Reply from executor01 in 455.761µs
Reply from executor01 in 609.2µs
Reply from executor01 in 414.827µs
Reply from executor01 in 751.812µs
```

```bash
[root@c02 /]# receptorctl ping executor01
Reply from executor01 in 406.311µs
Reply from executor01 in 640.188µs
Reply from executor01 in 590.659µs
Reply from executor01 in 1.32654ms
```

```bash
[root@c03 /]# receptorctl ping executor01
ERROR: blocked by firewall
ERROR: blocked by firewall
ERROR: blocked by firewall
ERROR: blocked by firewall
```

```bash
$ docker compose logs c03
...
c03  | WARNING 2022/12/09 06:32:11 Received unreachable message from relayer01
c03  | WARNING 2022/12/09 06:32:12 Received unreachable message from relayer01
c03  | WARNING 2022/12/09 06:32:13 Received unreachable message from relayer01
c03  | WARNING 2022/12/09 06:32:14 Received unreachable message from relayer01
...
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
[root@c01 /]# receptorctl ping executor01
Reply from executor01 in 455.761µs
Reply from executor01 in 609.2µs
Reply from executor01 in 414.827µs
Reply from executor01 in 751.812µs

[root@c01 /]# receptorctl ping executor02
Reply from executor02 in 485.397µs
Reply from executor02 in 655.979µs
Reply from executor02 in 617.134µs
Reply from executor02 in 1.550298ms
```

```bash
[root@c02 /]# receptorctl ping executor01
Reply from executor01 in 406.311µs
Reply from executor01 in 640.188µs
Reply from executor01 in 590.659µs
Reply from executor01 in 1.32654ms

[root@c02 /]# receptorctl ping executor02
ERROR: blocked by firewall
ERROR: blocked by firewall
ERROR: blocked by firewall
ERROR: blocked by firewall
```

```bash
$ docker compose logs c02
...
c02  | WARNING 2022/12/09 06:38:56 Received unreachable message from executor02
c02  | WARNING 2022/12/09 06:38:57 Received unreachable message from executor02
c02  | WARNING 2022/12/09 06:38:58 Received unreachable message from executor02
c02  | WARNING 2022/12/09 06:38:59 Received unreachable message from executor02
...
```

```bash
[root@c02 /]# receptorctl work release --all
Released:
(xFD0u0uh, released)
```

## Case 04: Work signing

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
