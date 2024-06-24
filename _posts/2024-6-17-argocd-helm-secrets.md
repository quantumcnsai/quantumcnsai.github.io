---
layout: post
title: ArgoCD와 Helm Secret Plugin을 함께 사용하기
subtitle: Application 배포를 안전하고 쉽게 하기
cover-img: /assets/img/post_background.jpg
thumbnail-img: /assets/img/argocd.png
share-img: /assets/img/argocd.png
tags: [argocd, helm]
author: Alex
---

## 환경 설정
Helm은 사용자 Application을 k8s 환경에 편하게 설치하고 업데이트할 수 있게 도와주는 유용한 툴이지만 secret 데이터를
별도로 관리해주는 기능이 기본적으로 들어 있지 않기 때문에 사용자의 password, token 등 데이터가 노출될 있는 위험이 있습니다.

이를 보완하기 위헤서 Helm Secret Plugin을 추가로 설치하여 사용하는 것이 좋습니다. 이 플러그인은 helm chart의 
values 파일을 암호화해서 저장 및 관리합니다. 즉 암호화한 상태로 소스 저장소에 저장해도 데이터 노출 우려를 줄일 수 있습니다. 

그러나 Helm Secret Plugin은 파일을 직접 암호화를 하는 것이 아니고 별도 백엔드 프로그램을 쓰도록 설계가 되어 있기 때문에
추가로 설치할 필요가 있습니다.

즉 Helm Secret Plugin을 사용하기 위해서는 백엔드 프로그램 sops를 필수로 설치해줘야 합니다.

### sops 설치

```
# sops binary 다운로드
curl -LO https://github.com/getsops/sops/releases/download/v3.8.1/sops-v3.8.1.linux.amd64

# binary를 PATH로 이동 
mv sops-v3.8.1.linux.amd64 /usr/local/bin/sops

# binary에 실행 권한 부여
chmod +x /usr/local/bin/sops
```

### Public:Private 키
그 다음으로 필요한 것이 sops에서 파일 암호화 시 사용할 키 파일입니다. 키 파일은 파일 암호화, 암호화된 파일을 수정 또는 원복할 때 사용됩니다. 키를 가지고 있는 사람이 secret 파일을 보거나 수정할 수 있기 때문에 키를 잘 관리하는것이 중요합니다.

키는 gpg 와 age 두가지 지원되기는 하지만 helm secret plugin 공식 문서를 보면 age 사용을 권장하고 있는 것을 확인할 수 있습니다. 그러므로 이 가이드에서도 age 를 이용하는 방식으로 설명드리겠습니다.

sops와 argocd에서 모두 지원하는 키 방식이 두 가지입니다.
- gpg : 클라언트를 설치해서 키 관리히기 때문에 난이도가 있음
- age : secret plugin 공식 문서에서 권장하는 방식입니다.

### age 설치

github repo에서 최신 버전을 정보를 획득히기
```
AGE_LATEST_VERSION=$(curl -s "https://api.github.com/repos/FiloSottile/age/releases/latest" | grep -Po '"tag_name": "v\K[0-9.]+')
```
curl를 이용하여 최신 버전의 .tar.gz를 다운로드
```
curl -Lo age.tar.gz "https://github.com/FiloSottile/age/releases/latest/download/age-v${AGE_LATEST_VERSION}-linux-amd64.tar.gz"
```
다운로드한 .tar.gz 압축 풀기
```
tar xf age.tar.gz
```
압축풀고 age와 age-keygen을 /usr/local/bin로 이동
```
sudo mv age/age /usr/local/bin
sudo mv age/age-keygen /usr/local/bin
```
필요없는 파일 정리
```
rm -rf age.tar.gz
rm -rf age
```
age와 age-keygen을 동작 확인
```
age -version
age-keygen -version
```


### age를 이용한 키 생성: 
```
age-keygen -o key.txt
```
생성한 키에 대한 위치와 퍼블릭키를 아래와 같이 환경변수로 등록해주면 sops에서 해당키를 사용할 수 있게 됩니다. 
```
export SOPS_AGE_KEY_FILE="/root/key.txt"
export SOPS_AGE_RECIPIENTS="age 퍼블릭키"
```


### Helm secrets 플러그인 설치

``` helm plugin install https://github.com/jkroepke/helm-secrets --version v4.6.0 ```

여기까지는 필요한 프로그램을 개발 또는 관리 서버에 설치하고 세팅하는 과정이였으며 실제로 argocd내에 helm secrets plugin 설정하는 부분이 지금부터 시작합니다/


## ArgoCD에 Helm Secrets 플러그인 추가하기 


ArgoCD에 secrets plugin 추가하는 방법이 2가지가 있습니다
- ArgoCD 컨테이너 이미지를 직접 빌드하는 방식
- initContainer 사용해서 필요한 세팅을 하는 방법  

컨테이너 이미지를 빌드하는 것이 새로운 버전으로 업그래이드 시 매번 빌드해야 하는 단점이 있기 때문에 업그래드를 비교적 쉽게 할 수 있는 
initContainer 사용하는 방식으로 설명드릴 예정입니다.

이를 위해서 ArgoCD Helm Chart를 설치 values를 통해 필요한 설정을 해줍니다.

먼저 argocd가 외부에 있는 values 파일 위치로 허용하는 스키마를 추가해줍니다. 
```
server:
  config:
    helm.valuesFileSchemes: >-
      secrets+gpg-import, secrets+gpg-import-kubernetes,
      secrets+age-import, secrets+age-import-kubernetes,
      secrets,secrets+literal,
      https, http
```

플러그인 추가를 위헤 필효한 환경 변수 세팅
```
repoServer:
  env:
    - name: HELM_PLUGINS
      value: /custom-tools/helm-plugins/
    - name: HELM_SECRETS_SOPS_PATH
      value: /custom-tools/sops
    - name: HELM_SECRETS_BACKEND
      value: sops
    - name: HELM_SECRETS_VALUES_ALLOW_SYMLINKS
      value: "false"
    - name: HELM_SECRETS_VALUES_ALLOW_ABSOLUTE_PATH
      value: "true"
    - name: HELM_SECRETS_VALUES_ALLOW_PATH_TRAVERSAL
      value: "false"
    - name: HELM_SECRETS_WRAPPER_ENABLED
      value: "true"
    - name: HELM_SECRETS_DECRYPT_SECRETS_IN_TMP_DIR
      value: "true"
    - name: HELM_SECRETS_HELM_PATH
      value: /usr/local/bin/helm
```
initContainer에서 sops, helm secrets 바이너리를 다운로드하도록 script 추가
```
  initContainers:
    - name: download-tools
      image: alpine:latest
      imagePullPolicy: IfNotPresent
      command: [sh, -ec]
      env:
        - name: HELM_SECRETS_VERSION
          value: "4.6.0"
        - name: SOPS_VERSION
          value: "3.8.1"
      args:
        - |
          mkdir -p /custom-tools/helm-plugins
          wget -qO- https://github.com/jkroepke/helm-secrets/releases/download/v${HELM_SECRETS_VERSION}/helm-secrets.tar.gz | tar -C /custom-tools/helm-plugins -xzf-;
          wget -qO /custom-tools/sops https://github.com/getsops/sops/releases/download/v${SOPS_VERSION}/sops-v${SOPS_VERSION}.linux.amd64
          cp /custom-tools/helm-plugins/helm-secrets/scripts/wrapper/helm.sh /custom-tools/helm
          chmod +x /custom-tools/*
      volumeMounts:
        - mountPath: /custom-tools
          name: custom-tools

```

ArgoCD의 Repo 서버 컨테이너에 앞서 설장 필요한 파일들을 불륨을 통해 마운트시켜줍니다. 

```
  volumeMounts:
    - mountPath: /custom-tools
      name: custom-tools
    - mountPath: /usr/local/sbin/helm
      subPath: helm
      name: custom-tools
    - mountPath: /helm-secrets-private-keys/
      name: helm-secrets-private-keys

  volumes:
    - name: custom-tools
      emptyDir: {}
    - name: helm-secrets-private-keys
      secret:
        secretName: helm-secrets-private-keys
```

ArgoCD Helm Chart 설치 전 전에 생성한 age 키를 argocd에서 사용할 수 있게 k8s secret 형태로 생성해줘야 합니다. 

``` kubectl create secret generic helm-secrets-private-keys --from-file=key.txt=key.txt ```

이제 작성한 values 파일을 이용해 ArgoCD helm chart를 설치해주면 secrets 플러그인을 사용할 수 있는 argocd 인스턴스가 설치됩니다.

## 사용 방법

ArgoCD에서 App 생성 시 암호화된 values 파일을 사용하려고 할 떄 기존과 동일할 방식으로 values 파일을 지정해주면 되는데 
단 암호화된 values 파일 경로를 줄 떄 앞에 스크마를 맞게 줘야 합니다.

values 파일 설정 예시:

```
secrets+age-import:///helm-secrets-private-keys/key.txt?values-enc.yaml
secrets+age-import:///helm-secrets-private-keys/key.txt?http://gitlab.com/values-enc.yaml
```

첫번째로 스키마를 ```secrets+age-import://``` 으로 설정. 이것은 age 키와 age 키로 암호화된 파일을 불러온다는 뜻입니다.

그다음으로 ```/helm-secrets-private-keys/key.txt``` 는 repoServer 컨테이너 안에 마운트 되어 있는 age 키 파일 위치입니다.

마지막으로 **?** 뒤에 ```?http://gitlab.com/values-enc.yaml``` 는 암호화된 values 파일의 위치입니다.


참초 문서:
- [https://technotim.live/posts/install-age/](age 설치 가아드)
- [https://github.com/getsops/sops/releases](sops 설치 가이드)