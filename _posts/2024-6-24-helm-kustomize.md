---
layout: post
title: 쿠버네티스에서 Helm과 Kustomize으로 Application 배포하기
subtitle: 쿠버네티스 입문자들을 위한 자료
cover-img: /assets/img/post_background.jpg
thumbnail-img: /assets/post_images/grafana_1.jpeg
share-img: /assets/post_images/grafana_1.jpeg
tags: [k8s, helm, kustomize]
author: Alex
---

# Helm과 Kustomize 사용법
## 목차
- [Helm과 Kustomize 사용법](#helm과-kustomize-사용법)
  - [목차](#목차)
    - [참고 자료](#참고-자료)
  - [Helm 소개](#helm-소개)
    - [Helm이란 무엇인가?](#helm이란-무엇인가)
    - [Helm의 기본 구성 요소](#helm의-기본-구성-요소)
      - [Chart: Helm 패키지의 구조](#chart-helm-패키지의-구조)
      - [Repository: 차트 저장과 공유](#repository-차트-저장과-공유)
      - [Releases: 배포 관리](#releases-배포-관리)
    - [Helm 설치 및 설정](#helm-설치-및-설정)
      - [Helm 설치하기](#helm-설치하기)
    - [차트 사용](#차트-사용)
      - [차트 검색 및 추가](#차트-검색-및-추가)
      - [차트 설치, 업그레이드 및 롤백](#차트-설치-업그레이드-및-롤백)
      - [차트 값 커스터마이징](#차트-값-커스터마이징)
    - [차트 만들기](#차트-만들기)
      - [차트 생성 및 Packaging](#차트-생성-및-packaging)
      - [Chart 구성:](#chart-구성)
      - [의존성 관리](#의존성-관리)
    - [고급 Helm 기능](#고급-helm-기능)
      - [Helm 훅 (Hooks): 배포 흐름 제어](#helm-훅-hooks-배포-흐름-제어)
  - [Kustomize](#kustomize)
    - [Kustomize의 개념](#kustomize의-개념)
    - [Kustomize의 핵심 요소](#kustomize의-핵심-요소)
      - [Base와 Overlay의 개념](#base와-overlay의-개념)
      - [Kustomize의 사용법](#kustomize의-사용법)
      - [Kustomization 파일](#kustomization-파일)
    - [고급 기능](#고급-기능)
      - [Transformers](#transformers)
      - [Generators](#generators)
        - [ConfigMapGenerator](#configmapgenerator)
        - [SecretsGenerator](#secretsgenerator)
        - [HelmChartInflationGenerator](#helmchartinflationgenerator)
    - [Kustomize와 Helm의 차이점](#kustomize와-helm의-차이점)

### 참고 자료
- [https://helm.sh/docs/](https://helm.sh/docs/)
- [https://kubectl.docs.kubernetes.io/references/kustomize/builtins/](https://kubectl.docs.kubernetes.io/references/kustomize/builtins/)

<div style="page-break-after: always;"></div>

## Helm 소개
### Helm이란 무엇인가?
Helm은 Kubernetes에서 사용되는 패키지 매니저로, 애플리케이션의 정의, 설치, 업그레이드를 간소화하는 도구입니다.


### Helm의 기본 구성 요소
![](/images/helm1.png)
#### Chart: Helm 패키지의 구조
Helm의 패키지로, 쿠버네티스 클러스터 내에서 애플리케이션을 실행하는데 필요한 모든 리소스와 설정을 담고 있는 파일의 집합입니다. 차트는 YAML 파일로 구성된 템플릿과 메타데이터로 구성됩니다.
#### Repository: 차트 저장과 공유
차트를 저장하고 공유하는 공간입니다. 사용자는 자신의 차트를 생성하여 레포지토리에 업로드할 수 있고, 필요에 따라 다른 사용자의 차트를 검색하고 사용할 수 있습니다.
#### Releases: 배포 관리
차트가 클러스터에 설치되었을 때의 인스턴스입니다. 하나의 차트로 여러 번의 설치가 가능하며, 각각은 독립적인 릴리스로 관리됩니다.

### Helm 설치 및 설정
#### Helm 설치하기
[Helm 설치 가이드 링크](https://helm.sh/docs/intro/install/)

### 차트 사용
#### 차트 검색 및 추가
Repo 추가
   ```
    helm repo add [repo_name] [repo_url]
    helm repo update
   ```

Chart 검색

    helm search repo [keyword]

#### 차트 설치, 업그레이드 및 롤백
설치
   ```
   helm install [release_name] [chart_name]
   ```
구성 값 설정
   ```
   helm install [release_name] [chart_name] --values custom_values.yaml
   helm install [release_name] [chart_name] --set key=value
   ```
업그레이드 및 롤백
   ```
   helm upgrade [release_name] [chart_name]
   helm rollback [release_name] [revision]
   ```

Chart 삭제
   ```
   helm uninstall [release_name]
   ```

#### 차트 값 커스터마이징
```
helm install [release_name] [chart_name] --values custom_values.yaml
helm install [release_name] [chart_name] --set key=value
```

### 차트 만들기
#### 차트 생성 및 Packaging
``` 
helm create [chart_name] 
```
#### Chart 구성:
1. **Chart.yaml**
기본 정보 파일: 차트의 메타데이터를 포함하며, 차트의 이름, 버전, 설명 등이 포함됩니다. 이 파일은 차트의 아이덴티티를 정의하고 다른 차트와 구별하는 데 사용됩니다.
2. **values.yaml**
기본 설정 파일: 차트의 설치 시 사용되는 기본 구성 값들을 정의합니다. 사용자는 이 파일에 정의된 값을 오버라이드하여 사용자 정의 설정을 적용할 수 있습니다.
3. **templates/**
쿠버네티스 리소스 템플릿: 이 디렉토리에는 차트의 리소스를 생성하는 데 사용되는 템플릿 파일들이 들어 있습니다. 각 파일은 쿠버네티스 API 객체를 정의하며, Helm의 템플릿 언어를 사용하여 동적으로 값들을 주입할 수 있습니다.
4. **charts/**
종속 차트: 다른 차트에 의존하는 차트들을 포함하는 디렉토리입니다. 주로 복잡한 애플리케이션을 모듈화하여 관리할 때 사용됩니다. 예를 들어, 웹 애플리케이션 차트가 데이터베이스 차트에 의존할 수 있습니다.
5. **crds/**
Custom Resource Definitions (CRDs): 차트가 사용하는 커스텀 리소스 정의를 포함합니다. 이 디렉토리의 CRD 파일들은 차트 설치 시 클러스터에 먼저 설치되어야 하는 리소스를 정의합니다.
6. **.helmignore**
차트 패키징에서 제외할 파일 목록: .gitignore 파일과 유사하게 작동하며, 차트를 패키징할 때 무시해야 할 파일이나 디렉토리를 지정합니다.

#### 의존성 관리

Chart.yaml 파일의 dependencies 필드에 아래와 같이 리스트업합니다.

```
dependencies:
  - name: apache
    version: 1.2.3
    repository: https://example.com/charts
  - name: mysql
    version: 3.2.1
    repository: https://another.example.com/charts
```

 의존성 관리 명령어:
```
helm dependency list

helm dependency update

helm dependency build
```

### 고급 Helm 기능
#### Helm 훅 (Hooks): 배포 흐름 제어
Helm hook은 Helm 차트의 생명주기 동안 특정 포인트에서 사용자 정의 작업을 실행할 수 있는 기능을 제공합니다. 

Annotation Value |	Description
---------|---------
pre-install |	Executes after templates are rendered, but before any resources are created in Kubernetes
post-install |	Executes after all resources are loaded into Kubernetes
pre-delete |	Executes on a deletion request before any resources are deleted from Kubernetes
post-delete |	Executes on a deletion request after all of the release's resources have been deleted
pre-upgrade	| Executes on an upgrade request after templates are rendered, but before any resources are updated
post-upgrade |	Executes on an upgrade request after all resources have been upgraded
pre-rollback |	Executes on a rollback request after templates are rendered, but before any resources are rolled back
post-rollback |	Executes on a rollback request after all resources have been modified
test |	Executes when the Helm test subcommand is invoked ( view test docs)

예제: 
```
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ .Release.Name }}"
  labels:
    app.kubernetes.io/managed-by: {{ .Release.Service | quote }}
    app.kubernetes.io/instance: {{ .Release.Name | quote }}
    app.kubernetes.io/version: {{ .Chart.AppVersion }}
    helm.sh/chart: "{{ .Chart.Name }}-{{ .Chart.Version }}"
  annotations:
    # This is what defines this resource as a hook. Without this line, the
    # job is considered part of the release.
    "helm.sh/hook": post-install
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    metadata:
      name: "{{ .Release.Name }}"
      labels:
        app.kubernetes.io/managed-by: {{ .Release.Service | quote }}
        app.kubernetes.io/instance: {{ .Release.Name | quote }}
        helm.sh/chart: "{{ .Chart.Name }}-{{ .Chart.Version }}"
    spec:
      restartPolicy: Never
      containers:
      - name: post-install-job
        image: "alpine:3.3"
        command: ["/bin/sleep","{{ default "10" .Values.sleepyTime }}"]
```
  
<div style="page-break-after: always;"></div>

## Kustomize
### Kustomize의 개념
- Kustomize는 Kubernetes 리소스의 Manifest 파일을 관리하는 도구입니다. 
- 원본 Manifest를 도대로 사용자가 정의한 변경내용을 적용하여 새로운/커스커마이즈된 manifest를 생성해주는 도구입니다.   
- 코드 중복을 최소화하면서 여러 환경(개발, 스테이징, 프로덕션 등)에 대해 동일한 애플리케이션을 다르게 설정할 수 있게 해줍니다.

![](/images/kustomize2.jpeg)
  
### Kustomize의 핵심 요소

**kustomization.yaml** 파일
- *kustomization.yaml* 파일로, Kustomize의 작업을 지시합니다. 이 파일은 사용할 리소스, 적용할 패치, 그리고 다른 설정(예: 네임스페이스 변경, 라벨 추가)을 정의합니다.

**변형에 대한 정의**:
- **Transformer**
  - 여러 리소스에 대한 일괄적으로 적용 작업 수행 (namespace, label, annotation, prefix 등)
- **Patch**
  - 리소스의 특정 부분을 수정하기 위한 패치 파일을 지정합니다. 패치는 strategic, JSON, 그리고 merge patch의 형식을 사용할 수 있습니다.
- **Generator**
  - ConfigMaps과 Secrets를 동적으로 생성할 수 있도록 합니다.

#### Base와 Overlay의 개념
여러 환경(개발, 스테이징, 프로덕션 등)에 대해 동일한 애플리케이션을 다르게 설정할 수 있게 해줍니다.
![](/images/kustomize1.png)

#### Kustomize의 사용법

결과 출력:
```
kubectl kustomize <kustomization_directory>
```

결과 적용:
```
kubectl apply -k <kustomization_directory>
```

#### Kustomization 파일

- **Resources**:
이 섹션은 Kustomize가 처리할 기본 리소스 파일이나 다른 kustomization 디렉토리의 경로를 나열합니다. 리소스는 일반적으로 Kubernetes의 YAML 정의 파일들입니다.
- **Patches**:
리소스의 특정 부분을 수정하기 위한 패치 파일을 지정합니다. 패치는 strategic, JSON, 그리고 merge patch의 형식을 사용할 수 있습니다. 이를 통해 기존 리소스에 변형을 가하여 특정 환경에 맞게 조정할 수 있습니다.
- **Generators**:
ConfigMaps과 Secrets를 동적으로 생성할 수 있도록 합니다. 리터럴 값, 파일 내용 또는 기타 소스로부터 데이터를 가져와서 Kubernetes에서 사용할 수 있는 ConfigMaps 또는 Secrets 리소스로 변환합니다.
- **Namespace**:
적용될 모든 리소스의 네임스페이스를 지정합니다. 이 설정은 모든 리소스에 일괄적으로 적용되어 네임스페이스를 오버라이딩할 수 있습니다.
- **NamePrefix and NameSuffix**:
모든 리소스의 이름 앞이나 뒤에 특정 접두사 또는 접미사를 추가하여, 리소스 관리를 보다 구체적이고 조직적으로 할 수 있도록 합니다.
- **CommonLabels and CommonAnnotations**:
모든 리소스에 공통으로 적용될 라벨이나 어노테이션을 추가합니다. 이는 리소스를 그룹화하고 관리하는 데 유용하며, 특히 대규모 시스템에서 리소스를 식별하고 필터링하는 데 도움이 됩니다.
- **Images**:
리소스 내에서 사용되는 이미지의 이름, 태그, 또는 다이제스트를 변경할 수 있습니다. 이를 통해 다른 환경에서 다른 버전의 이미지를 사용하거나, 이미지 리포지토리를 쉽게 변경할 수 있습니다.
- **Replicas**:
특정 리소스의 복제본 수를 조정할 수 있습니다. 이는 예를 들어 프로덕션 환경에서는 더 많은 복제본을 사용하고, 개발 환경에서는 적은 복제본을 사용하도록 설정할 때 유용합니다.

```
# This is the root kustomization file, which ties everything together

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# Resources to include
resources:
  - deployment.yaml
  - service.yaml

# Prefix added to all resource names
namePrefix: dev-

# Namespace in which resources will be deployed
namespace: my-app-namespace

# Common labels applied to all resources
commonLabels:
  app: my-app
  env: development

# Patches for environment-specific adjustments
patchesStrategicMerge:
  - patch_deployment.yaml
  - patch_service.yaml

# ConfigMap generator with literals
configMapGenerator:
  - name: my-app-config
    literals:
      - key1=value1
      - key2=value2

# Secret generator using literal values (secrets should be encrypted in real use)
secretGenerator:
  - name: my-app-secret
    literals:
      - username=admin
      - password=s3cr3t

```

### 고급 기능
#### Transformers
트랜스포머는 한 번에 여러 필드나 설정에 걸쳐 변경을 적용할 수 있습니다. 이는 Kustomization에 있는 모든 리소스에 일관된 업데이트나 구성을 적용하는 데 유용합니다.

- **NamePrefix 및 NameSuffix 트랜스포머**: 리소스 이름에 접두사나 접미사를 추가하여, 같은 네임스페이스 안에서 동일 애플리케이션의 여러 인스턴스를 구분하는 데 도움을 줍니다.
- **LabelTransformer**: Kustomization의 모든 리소스에 라벨을 추가하거나 업데이트합니다. 리소스를 조직하거나 정책을 적용하거나 접근을 제어하는 데 중요합니다.
- **AnnotationsTransformer**: LabelTransformer와 유사하지만 어노테이션을 위한 것으로, 쿠버네티스 자체가 직접 사용하지 않는 메타데이터를 리소스에 첨부하는 데 사용됩니다.
- **ImageTransformer**: 모든 리소스의 이미지 이름, 태그 또는 다이제스트를 변경하는데, 새 이미지 버전의 롤아웃 동안 특히 유용합니다.

kustomization.yaml
```
resources:
  - deployment.yaml
  - service.yaml

transformers:
  - label_transformer.yaml

```

label_transformer.yaml
```
apiVersion: builtin
kind: LabelTransformer
metadata:
  name: add-env-label
labels:
  environment: dev
fieldSpecs:
  - path: metadata/labels
    create: true
```

#### Generators
##### ConfigMapGenerator

kustomization.yaml
```
generatorOptions:
  labels:
    fruit: apple

configMapGenerator:
- name: my-java-server-props
  behavior: merge
  files:
  - application.properties
  - more.properties
- name: my-java-server-env-vars
  literals: 
  - JAVA_HOME=/opt/java/jdk
  - JAVA_TOOL_OPTIONS=-agentlib:hprof
  options:
    disableNameSuffixHash: true
    labels:
      pet: dog
- name: dashboards
  files:
  - mydashboard.json
  options:
    annotations:
      dashboard: "1"
    labels:
      app.kubernetes.io/name: "app1"

```

##### SecretsGenerator
리스트에 정의된 아이템별로 각각 Secret가 생성됩니다.
```
secretGenerator:
- name: app-tls
  files:
  - secret/tls.cert
  - secret/tls.key
  type: "kubernetes.io/tls"
- name: app-tls-namespaced
  # you can define a namespace to generate
  # a secret in, defaults to: "default"
  namespace: apps
  files:
  - tls.crt=catsecret/tls.cert
  - tls.key=secret/tls.key
  type: "kubernetes.io/tls"
- name: env_file_secret
  envs:
  - env.txt
  type: Opaque
- name: secret-with-annotation
  files:
  - app-config.yaml
  type: Opaque
  options:
    annotations:
      app_config: "true"
    labels:
      app.kubernetes.io/name: "app2"
```
##### HelmChartInflationGenerator
- helmCharts
- helmGlobals

```
helmCharts:
- name: minecraft
  repo: https://kubernetes-charts.storage.googleapis.com
  version: v1.2.0
  releaseName: test
  namespace: testNamespace
  valuesFile: values.yaml
  additionalValuesFiles:
  - values-file-1.yml
  - values-file-2.yml
```

```
helmGlobals:
  chartHome: my-charts-dir
```

### Kustomize와 Helm의 차이점
- Templating의 유무
- 복잡성
- 업그레이드 및 롤백 