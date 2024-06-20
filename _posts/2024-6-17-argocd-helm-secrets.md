---
layout: post
title: ArgoCD와 Helm Secret Plugin을 함께 사용하기2
subtitle: Application 배포를 안전하고 쉽게 하기2
cover-img: /assets/img/path.jpg
thumbnail-img: /assets/img/thumb.png
share-img: /assets/img/path.jpg
tags: [books, test]
author: Alex
---

사전에 설치할 패키지: age, sops

- https://technotim.live/posts/install-age/
- https://github.com/getsops/sops/releases
  
Helm secrets 플러그인 설치

``` helm plugin install https://github.com/jkroepke/helm-secrets --version v4.6.0 ```

helm secret plugin은 helm chart의 values 파일을 암호화하는 방식이기 때문에 암호화 할 때 사용될 개인 키를 생성하고 관리할 필요가 있습니다. 이 키를 argocd에서 암호화된 values 파일로 설치할 때 사용하기도 합니다.
 
키는 gpg 와 age 두가지 지원되기는 하지만 helm secret plugin 공식 문서를 보면 age 사용을 권장하고 있는 것을 확인할 수 있습니다. 그러므로 이 가이드에서도 age 를 이용하는 방식으로 설명드리겠습니다.

age를 이용한 키 생성: 

age-keygen -o key.txt
- export SOPS_AGE_KEY_FILE="/root/key.txt"
- export SOPS_AGE_RECIPIENTS=age125tye5k53k5e3acdc4e6hhh7fxlnlsyfw4cp9k5s9qxsj7xceyxq62s03v

age 키를 argocd에서 사용하기 위해 kubernetes secret 형태로 만듭니다.
kubectl create secret generic helm-secrets-private-keys --from-file=key.txt=key.txt
argocd 에서 secrets plugin 추가하는 방법이 2가지가 있습니다
argocd custom image 관리
initContainer 사용해서 세팅 
TODO: 두가지 방법에 대한 장단점을 상세하게 설명을 추가 해야함


helm-secrets-private-keys 시크릿 생성 <- age로 만든 키값으로 생성  
argocd repo server의 initContainer에 필요한 script 추가
helm-secrets-private-keys 시크릿을 argocd repo server 파드에 마운트해줘야 argocd가 암호화된 values 파일을 사용해서 차트 설치할 수 있다.


values 파일 설정:
```
secrets+age-import:///helm-secrets-private-keys/key.txt?values-enc.yaml
secrets+age-import:///helm-secrets-private-keys/key.txt?http://gitlab.com/values-enc.yaml
```

