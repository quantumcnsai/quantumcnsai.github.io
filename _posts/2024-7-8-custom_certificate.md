---
layout: post
title: k8s 인증서 10년으로 만들기
subtitle: k8s 인증서 10년으로 만들기
cover-img: /assets/img/post_background.jpg
thumbnail-img: /assets/post_images/1.14.png
share-img: /assets/post_images/1.14.png
tags: [k8s]
author: Alex
preview: "k8s 인증서 유효기간을 10년으로 만들기 위해서는 openssl 명령어를 통해 수동으로 만들어 기존 인증서를 대체하는 방식으로 진행됩니다."
---

## k8s 인증서 10년으로 만들기
k8s 인증서 유효기간을 10년으로 만들기 위해서는 openssl 명령어를 통해 수동으로 만들어 기존 인증서를 대체하는 방식으로 진행됩니다.

### apiserver.crt 인증서 만들기

csr을 만들기 위한 conf 파일 작성
```
# apiserver-csr.conf
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[ dn ]
CN = kube-apiserver

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = k8s.smilegate.net
DNS.2 = kubernetes
DNS.3 = kubernetes.default
DNS.4 = kubernetes.default.svc
DNS.5 = kubernetes.default.svc.cluster
DNS.6 = kubernetes.default.svc.cluster.local    
IP.1 = 10.144.0.1
IP.2 = 10.130.172.31

[ v3_ext ]
authorityKeyIdentifier=keyid,issuer:always
basicConstraints=CA:FALSE
keyUsage=keyEncipherment,dataEncipherment
extendedKeyUsage=serverAuth,clientAuth
subjectAltName=@alt_names
```
csr 생성
```
openssl req -new -key /etc/kubernetes/pki/apiserver.key -out apiserver.csr -config apiserver-csr.conf
```
인증서 생성
```
openssl x509 -req -days 36500 -CAcreateserial \             
        -CA /etc/kubernetes/pki/ca.crt \             
        -CAkey /etc/kubernetes/pki/ca.key \
        -in /etc/kubernetes/qks/qks-custom-certificates/apiserver.csr \
        -out /etc/kubernetes/qks/qks-custom-certificates/server.crt \
        -extensions v3_ext -extfile /etc/kubernetes/qks/qks-custom-certificates/apiserver_csr.conf
        -sha256
```


### apiserver-kubelet-client.crt 인증서 만들기
csr을 만들기 위한 conf 파일 작성
```
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[ dn ]
O = system:masters
CN = kube-apiserver-kubelet-client

[ req_ext ]


[ v3_ext ]
authorityKeyIdentifier=keyid,issuer:always
basicConstraints=CA:FALSE
keyUsage=keyEncipherment,dataEncipherment
extendedKeyUsage=serverAuth,clientAuth
```
csr 생성
```
openssl req -new -key /etc/kubernetes/pki/apiserver-kubelet-client.key -out apiserver-kubelet-client.csr -config apiserver-kubelet-client-csr.conf
```
인증서 생성
```
openssl x509 -req -in apiserver-kubelet-client.csr -CA ../../ca.crt -CAkey ../../ca.key -CAcreateserial -out apiserver-kubelet-client.crt -days 3650 -extensions v3_ext -extfile apiserver-kubelet-client-csr.conf -sha256
```



### apiserver-etcd-client.crt

csr을 만들기 위한 conf 파일 작성
```
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[ dn ]
O = system:masters
CN = kube-apiserver-etcd-client

[ req_ext ]


[ v3_ext ]
authorityKeyIdentifier=keyid,issuer:always
basicConstraints=CA:FALSE
keyUsage=keyEncipherment,dataEncipherment
extendedKeyUsage=serverAuth,clientAuth
```
csr 생성
```
openssl req -new -key /etc/kubernetes/pki/apiserver-etcd-client.key -out apiserver-etcd-client.csr -config apiserver-etcd-client-csr.conf
```
인증서 생성
```
openssl x509 -req -in apiserver-etcd-client.csr -CA /etc/kubernetes/pki/etcd/ca.crt -CAkey /etc/kubernetes/pki/etcd/ca.key -CAcreateserial -out apiserver-etcd-client.crt -days 3650 -extensions v3_ext -extfile apiserver-etcd-client-csr.conf -sha256
```


### etcd-server.crt
csr을 만들기 위한 conf 파일 작성

```
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[ dn ]
CN = k8s-ubuntu22-m01

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = localhost
DNS.2 = k8s-ubuntu22-m01
IP.1 = 127.0.0.1
IP.2 = 10.10.10.235
IP.3 = 0:0:0:0:0:0:0:1

[ v3_ext ]
authorityKeyIdentifier=keyid,issuer:always
basicConstraints=CA:FALSE
keyUsage=keyEncipherment,dataEncipherment
extendedKeyUsage=serverAuth,clientAuth
subjectAltName=@alt_names
```
csr 생성
```
openssl req -new -key /etc/kubernetes/pki/etcd/server.key -out etcd-server.csr -config etcd-server-csr.conf
```
인증서 생성
```
openssl x509 -req -in etcd-server.csr -CA /etc/kubernetes/pki/etcd/ca.crt -CAkey /etc/kubernetes/pki/etcd/ca.key -CAcreateserial -out server.crt -days 3650 -extensions v3_ext -extfile etcd-server-csr.conf -sha256
```


### peer.crt
csr을 만들기 위한 conf 파일 작성
```
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[ dn ]
CN = k8s-ubuntu22-m01

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = localhost
DNS.2 = k8s-ubuntu22-m01
IP.1 = 127.0.0.1
IP.2 = 10.10.10.235
IP.3 = 0:0:0:0:0:0:0:1

[ v3_ext ]
authorityKeyIdentifier=keyid,issuer:always
basicConstraints=CA:FALSE
keyUsage=keyEncipherment,dataEncipherment
extendedKeyUsage=serverAuth,clientAuth
subjectAltName=@alt_names
```
csr 생성
```
openssl req -new -key /etc/kubernetes/pki/etcd/peer.key -out etcd-peer.csr -config etcd-peer-csr.conf
```
인증서 생성
```
openssl x509 -req -in etcd-peer.csr -CA /etc/kubernetes/pki/etcd/ca.crt -CAkey /etc/kubernetes/pki/etcd/ca.key -CAcreateserial -out peer.crt -days 3650 -extensions v3_ext -extfile etcd-peer-csr.conf -sha256
```


### etcd-healthcheck-client.crt
csr을 만들기 위한 conf 파일 작성
```
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[ dn ]
CN = kube-etcd-healthcheck-client
O = system:masters 

[ req_ext ]

[ v3_ext ]
authorityKeyIdentifier=keyid,issuer:always
basicConstraints=CA:FALSE
keyUsage=keyEncipherment,dataEncipherment
extendedKeyUsage=serverAuth,clientAuth
```
csr 생성
```
openssl req -new -key /etc/kubernetes/pki/etcd/healthcheck-client.key -out etcd-healthcheck-client.csr -config etcd-healthcheck-client-csr.conf
```
인증서 생성
```
openssl x509 -req -in etcd-healthcheck-client.csr -CA /etc/kubernetes/pki/etcd/ca.crt -CAkey /etc/kubernetes/pki/etcd/ca.key -CAcreateserial -out healthcheck-client.crt -days 3650 -extensions v3_ext -extfile etcd-healthcheck-client-csr.conf -sha256
```


### front-proxy-client.crt

csr을 만들기 위한 conf 파일 작성
```
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[ dn ]
CN = front-proxy-client 

[ req_ext ]

[ v3_ext ]
authorityKeyIdentifier=keyid,issuer:always
basicConstraints=CA:FALSE
keyUsage=keyEncipherment,dataEncipherment
extendedKeyUsage=serverAuth,clientAuth
```
csr 생성
```
openssl req -new -key /etc/kubernetes/pki/front-proxy-client.key -out front-proxy-client.csr -config front-proxy-client-csr.conf
```
인증서 생성
```
openssl x509 -req -in front-proxy-client.csr -CA /etc/kubernetes/pki/front-proxy-ca.crt -CAkey /etc/kubernetes/pki/front-proxy-ca.key -CAcreateserial -out front-proxy-client.crt -days 3650 -extensions v3_ext -extfile front-proxy-client-csr.conf -sha256
```


### admin.conf 인증서 생성
csr을 만들기 위한 conf 파일 작성
```
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[ dn ]
CN = kubernetes-admin
O = system:masters 

[ req_ext ]

[ v3_ext ]
authorityKeyIdentifier=keyid,issuer:always
basicConstraints=CA:FALSE
keyUsage=keyEncipherment,dataEncipherment
extendedKeyUsage=serverAuth,clientAuth
```
k8s admin에 대한 비공개키는 admin.conf 파일 안에 있기 때문에 먼저 해당 키를 복사해서 kubernetes-admin.key 파일로 만들어줘야합니다. csr 파일 서명하기 위해 비공개 키가 단독 파일 형태로 있어야 하기 때문입니다.
csr 생성
```
openssl req -new -key /etc/kubernetes/kubernetes-admin.key -out kubernetes-admin.csr -config kubernetes-admin-csr.conf
```
인증서 생성
```
openssl x509 -req -in kubernetes-admin.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out kubernetes-admin.crt -days 3650 -extensions v3_ext -extfile kubernetes-admin-csr.conf -sha256
```

```
cat kubernetes-admin.crt | base64 -w 0
```

### Create controller-manager.conf
csr을 만들기 위한 conf 파일 작성
```
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[ dn ]
CN = system:kube-controller-manager

[ req_ext ]

[ v3_ext ]
authorityKeyIdentifier=keyid,issuer:always
basicConstraints=CA:FALSE
keyUsage=keyEncipherment,dataEncipherment
extendedKeyUsage=serverAuth,clientAuth
```

admin.conf에서 했던 방식과 동일하게 controller-manager.conf에 있는 비공개키를 단독파일로(cm.key) 저장합니다.
csr 생성
```
openssl req -new -key cm.key -out controller-manager-conf.csr -config controller-manager-conf.conf
```
인증서 생성
```
openssl x509 -req -in controller-manager-conf.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out controller-manager.crt -days 3650 -extensions v3_ext -extfile controller-manager-conf.conf -sha256
```

### Create kube-scheduler.conf
csr을 만들기 위한 conf 파일 작성
```
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[ dn ]
CN = system:kube-scheduler

[ req_ext ]

[ v3_ext ]
authorityKeyIdentifier=keyid,issuer:always
basicConstraints=CA:FALSE
keyUsage=keyEncipherment,dataEncipherment
extendedKeyUsage=serverAuth,clientAuth
```

admin.conf에서 했던 방식과 동일하게 kube-scheduler.conf에 있는 비공개키를 단독파일로(scheduler.key) 저장합니다.

csr 생성
```
openssl req -new -key scheduler.key -out scheduler-conf.csr -config scheduler-conf.conf
```
인증서 생성
```
openssl x509 -req -in scheduler-conf.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out scheduler-conf.crt -days 3650 -extensions v3_ext -extfile scheduler-conf.conf -sha256
```