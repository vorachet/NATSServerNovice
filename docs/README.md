
# Installation Guide for TLS-Enabled NATS Server

# Installing CFSSL

Creating certificates are requried and CFSSL can help simplify the creation of certificates more easily. See https://github.com/cloudflare/cfssl

Ubuntu
```
sudo apt-get install -y golang-cfssl
```

Mac
```
brew install cfssl 
```

# Creating certificates

## Client certificate

ca.json
```
{
    "CN": "YourCN",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
               "C": "YourInfo",
               "L": "YourInfo",
               "O": "YourInfo",
               "ST": "YourInfo"
        }
    ]
}
```

Command to generate client certificate
```
cfssl gencert -initca ca.json | cfssljson -bare ca
```

Expected output files
```
├── ca-key.pem
├── ca.csr
├── ca.pem
```

## Server certificate

In order to generate server certificate, client certificate files are required.

config.json
```
{
    "signing": {
        "default": {
            "expiry": "43800h"
        },
        "profiles": {   
            "server": {
                "expiry": "43800h",
                "usages": [
                    "signing",
                    "digital signing",
                    "key encipherment",
                    "server auth"
                ]
            }
        }
    }
}
```

server.json
```
{
    "CN": "YourCN",
    "hosts": [
        "127.0.0.1",
        "myhost.mydomain.com"
    ]
}
```

Command to generate certificate
```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=config.json -profile=server server.json | cfssljson -bare server
``` 


Expected output files
```
├── server-key.pem
├── server.csr
└── server.pem
```

# Running NATS server with TLS

## Creating a minimal TLS-enabled configuration file

tls.conf
```
listen: 0.0.0.0:4222
tls: {
  cert_file: "./certs/server.pem"
  key_file: "./certs/server-key.pem"
}
```

## Running NATS server

Note that modifyig host file for `myhost.mydomain.com` is needed when you want to test locally.
```
nats-server -c tls.conf
```

# Running NATS.C 

Use programs given by https://github.com/nats-io/nats.c/tree/main/examples


## PUBSUB

```
./subscriber -s myhost.mydomain.com:4222 \
    -subj test -txt hello \
    -print \
    -tls \
    -tlscacert ./certs/ca.pem \
    -tlscert ./certs/ca.pem \
    -tlskey ./certs/ca-key.pem
```

```
./publisher -s myhost.mydomain.com:4222 \
    -subj test -txt hello \
    -txt hello! \
    -count 10 \
    -tls \
    -tlscacert ./certs/ca.pem \
    -tlscert ./certs/ca.pem \
    -tlskey ./certs/ca-key.pem 
```

## REQREP

```
./replier -s myhost.mydomain.com:4222 \
    -subj test -txt hello \
    -print \
    -tls \
    -tlscacert ./certs/ca.pem \
    -tlscert ./certs/ca.pem \
    -tlskey ./certs/ca-key.pem
```

```
./requester -s myhost.mydomain.com:4222 \
    -subj test -txt hello \
    -print \
    -txt hello! \
    -count 10 \
    -tls \
    -tlscacert ./certs/ca.pem \
    -tlscert ./certs/ca.pem \
    -tlskey ./certs/ca-key.pem 
```

# Docker composer 

```
version: "3"
services:
  nats:
    image: nats:latest
    hostname: nats-server
    command: "-c /tls.conf -D"
    volumes:
      - "./tls.conf:/tls.conf"
      - "./certs/server.pem:/certs/server.pem"
      - "./certs/server-key.pem:/certs/server-key.pem"
    ports:
      - "4222:4222"
      - "8222:8222"

```