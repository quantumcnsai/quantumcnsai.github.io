---
layout: post
title: Opentelemetry로 멀티클러스터 Observablity 시스템 구성하기
subtitle: Opentelemetry 사용
cover-img: /assets/img/post_background.jpg
thumbnail-img: /assets/post_images/otel_multicluster.png
share-img: /assets/post_images/otel_multicluster.png
tags: [k8s, moninoring, opentelemetry, metric, log, trace]
author: Alex
preview: "Opentelemetry는 애플리케이션 및 시스템에서 제공되는 메트릭, 로그, 주척 정보를 수집하고 모니터링하기 위한 표준을 정의합니다. 기존에 다른 Observablity 툴들과 연동되는 인터페이스를 제공함로써 사용자가 원하는 시각화 툴이나 저장소를 그대로 사용할 수 있습니다."
---

# Opentelemetry로 멀티클러스터 Observablity 시스템 구성하기
## 목차
- [Opentelemetry로 멀티클러스터 Observablity 시스템 구성하기](#쿠버네티스에서 Opentelemetry를 이용하여 Observablity 시스템 구성하기)
  - [목차](#목차)
    - [참고 자료](#참고-자료)
  - [Opentelemetry 소개](#Opentelemetry-소개)
    - [Opentelemetry이란 무엇인가?](#opentelemetry이란-무엇인가?)
    - [Opentelemetry의 기본 구성](#opentelemetry의-기본-구성)
      - [OpenTelemetry의 주요 개념](#openTelemetry의-주요-요소)
      - [OpenTelemetry의 구성](#openTelemetry의-구성)
    - [Opentelemetry 설치 및 설정](#opentelemetry-설치-및-설정)
      - [Opentelemetry 설치하기](#opentelemetry-설치하기)
      - [Opentelemetry 설정하기](#opentelemetry-설정하기)
    - [Opentelemetry로 모니터링 시스템 구성하기](#Opentelemetry로-모니터링-시스템-구성하기)
    - [Opentelemetry로 MultiCluster 모니터링 시스템 구성하기](#opentelemetry로-multicluster-모니터링-시스템-구성하기)

### 참고 자료
- [https://opentelemetry.io/docs/](https://opentelemetry.io/docs/)

<div style="page-break-after: always;"></div>

## Opentelemetry 소개
### Opentelemetry이란 무엇인가?

OpenTelemetry는 분산된 시스템에서 성능 데이터를 수집, 처리, 내보내는 것을 표준화한 오픈소스 Observability 프레임워크입니다. 주로 애플리케이션의 메트릭, 로그, 트레이스를 수집하고 이를 분석 도구나 관측 플랫폼으로 전송하는 데 사용됩니다.


### Opentelemetry의 기본 구성
#### OpenTelemetry의 주요 개념 ####
OpenTelemetry는 분산시시템에 대한 상태 및 성능 대한 정보를 가지고 있는 데이터를 다룹니다.  

- **Metric**: CPU 사용량, 메모리 사용량, 요청 처리 시간 등과 같은 정량적인 성능 데이터를 수집합니다. 이를 통해 시스템 성능 상태를 모니터링하고 알림을 설정할 수 있습니다.

- **Log**: 애플리케이션의 이벤트, 오류 메시지 등을 기록하는 방식입니다. 로그는 발생한 문제나 이슈를 디버깅하는 데 매우 유용합니다.
  
- **Trace**: 분산 시스템에서 각 서비스나 컴포넌트 간의 요청 흐름을 추적하는 기능입니다. 이를 통해 애플리케이션의 성능을 분석하고, 문제가 발생한 지점을 식별할 수 있습니다.

#### OpenTelemetry의 구성 ####
- **SDK**: 애플리케이션에서 OpenTelemetry를 사용하여 데이터를 생성하고 수집하는 라이브러리입니다.
- **Collector**: 여러 서비스로부터 메트릭, 로그, 트레이스를 수집하여 이를 다양한 백엔드 시스템(예: Prometheus, Jaeger, Loki 등)에 전송할 수 있는 구성 요소입니다.
- **Exporter**: 데이터를 원하는 백엔드로 내보내는 모듈입니다. 예를 들어, 로그를 Loki로 보내거나 트레이스를 Jaeger로 보낼 수 있습니다.

![](/assets/post_images/otel_diagram1.png)


## Opentelemetry 설치 및 설정
### Opentelemetry 설치하기 ####

OpenTelemetry를 쿠버네티스(Kubernetes) 환경에 설치하는 방법은 여러 가지가 있습니다. 그중에서 쿠버네티스에서는 일반적으로는 OpenTelemetry Collector를 Operator로 설치하여 데이터를 수집하고 처리하는 방식이 적합합니다. 

OpenTelemetry Operator는 쿠버네티스에서 OpenTelemetry Collector를 자동으로 관리하는 Kubernetes Operator입니다. 이를 통해 클러스터에서 Collector를 더 쉽게 배포하고 관리할 수 있습니다.

설치 단계:
OpenTelemetry Operator 설치: 먼저 OpenTelemetry Operator를 설치합니다.

```
kubectl apply -f https://github.com/open-telemetry/opentelemetry-operator/releases/download/v0.65.0/opentelemetry-operator.yaml
```

Collector 리소스 생성: OpenTelemetryCollector CRD(Custom Resource Definition)를 사용하여 Collector 인스턴스를 정의할 수 있습니다. 예를 들어:
```
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: simplest
spec:
  config: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
    processors:

    exporters:
      # NOTE: Prior to v0.86.0 use `logging` instead of `debug`.
      debug:

    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: []
          exporters: [debug]
```

Collector 배포: 위에서 작성한 CRD를 적용하여 OpenTelemetry Collector를 배포합니다.

```
kubectl apply -f otel-collector-crd.yaml
```
### Opentelemetry 설정
OpenTelemetry가 데이터를 어떻게 수줍하고 처리할지는 OpentelemetryCollector 리소스를 생성할 떄 config 부분애 설정합니다.
config 부분은 다음과 같이 4가지 항목으로 구성됩니다. 
이중에서 receivers와 exporters는 plugin 형태이며 연동하고자 하는 tool와 맞는 plugin을 선택해서 사용합니다. 
- **receivers**: telemetry 데이터를 어디서 어떻게 수집할지 설정하는 부분입니다.
- **processors**: 수집한 데이터를 가공하거나 변경할 수 있는 기능입니다.
- **exporters**: 최종 데이터를 어디로 어떤 프로토콜로 보낼자 설정합니다.
- **service**: 위 3가지 plugin들을 이용해서 실제로 데이터를 수집하고 처리할 파이프라인을 구성합니다.

![alt text](/assets/post_images/otel_arch.png)

service 하위에 있는 pipelines 항목에는 여러개의 pipeline을 정의할 수 있으며 위 예제와 같이 logs, metrics, traces 로 구분해서 정의합니다.
즉 하나의 OpenTelemetry Collector에서 3가지 telemetry 데이터 모두에 대한 처리를 할 수 있다는 것입니다.

```
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318
processors:
  batch:

exporters:
  otlp:
    endpoint: otelcol:4317

extensions:
  health_check:
  pprof:
  zpages:

service:
  extensions: [health_check, pprof, zpages]
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp]
```
## Opentelemetry로 모니터링 시스템 구성하기
OpenTelemtry로 수집한 telemetry 데이터를 저정하는 저장소와 시각화 툴을 연동해야 온전한 모니탕링 시스템이 됩니다.
여기서 OpenTelemetry + GrafanaStack을 이용한 모니터링 시스템을 보여주고 있습니다.

![](/assets/post_images/otel_onecluster.png)

```
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: otel-collector
spec:
  mode: deployment
  config: |
        receivers:
          filelog:
            exclude:
              - /var/log/pods/*/otel-collector/*.log
            include:
              - /var/log/pods/*/*/*.log
            include_file_name: false
            include_file_path: true
            start_at: beginning
          prometheus:
            config:
              scrape_configs:
                - enable_compression: true
                  enable_http2: true
                  follow_redirects: true
                  honor_timestamps: true
                  job_name: otel-qks-mariadb
                  kubernetes_sd_configs:
                    - enable_http2: true
                      follow_redirects: true
                      kubeconfig_file: ""
                      namespaces:
                        names:
                          - accu
                        own_namespace: false
                      role: endpoints
                  metrics_path: /metrics
                  relabel_configs:
                    - action: replace
                      replacement: $$1
                      separator: ;
                      source_labels:
                        - job
                      target_label: __tmp_prometheus_job_name
                    - action: keep
                      regex: (accu-mariadb);true
                      replacement: $$1
                      separator: ;
                      source_labels:
                        - __meta_kubernetes_service_label_app_kubernetes_io_instance
                        - __meta_kubernetes_service_labelpresent_app_kubernetes_io_instance
                  scheme: http
                  scrape_interval: 15s
                  scrape_protocols:
                    - OpenMetricsText1.0.0
                    - OpenMetricsText0.0.1
                    - PrometheusText0.0.4
                  scrape_timeout: 10s
                  track_timestamps_staleness: false
    exporters:
      debug: {}
      loki:
        endpoint: http://loki-gateway.accu-monitoring.svc.cluster.local/loki/api/v1/push
      otlp:
        endpoint: http://otel.accuinsight.co:80
        tls:
          insecure: true
      otlphttp:
        endpoint: http://loki.accu-monitoring.svc.cluster.local:3100/otlp
    processors:
      attributes/clustername:
        actions:
          - action: insert
            key: cluster_name
            value: sk
      transform:
        metric_statements:
          - context: datapoint
            statements:
              - set(attributes["namespace"], resource.attributes["k8s.namespace.name"])
              - set(attributes["container"], resource.attributes["k8s.container.name"])
              - set(attributes["pod"], resource.attributes["k8s.pod.name"])
    service:
      pipelines:
        logs:
          exporters:
            - loki
          receivers:
            - filelog
        metrics:
          exporters:
            - debug
            - otlp
          processors:
            - transform
            - attributes/clustername
          receivers:
            - prometheus
```

## Opentelemetry로 MultiCluster 모니터링 시스템 구성하기
OpenTelemetry는 자체적으로 지원하는 OTLP 프로토클로 외부 시스템과 연동뿐만 아니라 Collector끼리 서로 연동할 수 있습니다.
이 기능을 이용하면 아래 그림과 같이 Workload 클러스터에 있는 Collector들이 수집한 데이터를 중앙에 있는 Management 클러스터에 설치된 Collector로 전송하고 메인 Collector는 각 telemetry 데이터를 해당 백엔드 저장소에 저장하고 이를 Grafana 시각화하는 방식입니다.

![](/assets/post_images/otel_multicluster.png)

아래 메인 Collector 설정 예시입니다:

```
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: qks-main-collector
spec:
  mode: deployment
  serviceAccount: qks-opentelemetry-collector
  managementState: managed
  config: |
    receivers:
      otlp:
        protocols:
          http: {}
          grpc: {}
    processors: {}
    exporters:
      debug: {}
      prometheus:
        endpoint: 0.0.0.0:9090

    service:
      pipelines:
        metrics:
          receivers: [otlp]
          processors: []
          exporters: [debug, prometheus]
```

각 workload 클러스터에 있는 Collector는 아래와 같이 설정합니다. 여기서 핵심은 export 부분에 메인 collector로 otlp로 프로토클로 데이터 전송한다는 설정입니다.

그리고 각 workload 클러스터에서 수집된 metric 정보를 구분하기 위해 모든 metric에 cluster_name 항목을 추가해주는 processer가 있습니다.

```
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: qks-collector
spec:
  mode: deployment
  serviceAccount: qks-opentelemetry-collector
  config: |
    receivers:
      prometheus:
        config:
          scrape_configs:
            - job_name: qks-mariadb-otel
              honor_timestamps: true
              track_timestamps_staleness: false
              scrape_interval: 15s
              scrape_timeout: 10s
              scrape_protocols:
              - OpenMetricsText1.0.0
              - OpenMetricsText0.0.1
              - PrometheusText0.0.4
              metrics_path: /metrics
              scheme: http
              enable_compression: true
              follow_redirects: true
              enable_http2: true
              relabel_configs:
              - source_labels: [job]
                separator: ;
                target_label: __tmp_prometheus_job_name
                replacement: $$1
                action: replace
              - source_labels: [__meta_kubernetes_service_label_app_kubernetes_io_instance, __meta_kubernetes_service_labelpresent_app_kubernetes_io_instance]
                separator: ;
                regex: (qks-mariadb);true
                replacement: $$1
                action: keep
               ...
               생략 
               ...
              kubernetes_sd_configs:
              - role: endpoints
                kubeconfig_file: ""
                follow_redirects: true
                enable_http2: true
                namespaces:
                  own_namespace: false
                  names:
                  - qks
    processors:
      transform:
        metric_statements:
          - context: datapoint
            statements:
            - set(attributes["namespace"], resource.attributes["k8s.namespace.name"])
            - set(attributes["container"], resource.attributes["k8s.container.name"])
            - set(attributes["pod"], resource.attributes["k8s.pod.name"])
      attributes/clustername:
        actions:
        - key: cluster_name
          value: "qms"
          action: insert
    exporters:
      # NOTE: Prior to v0.86.0 use `logging` instead of `debug`.
      debug: {}
      otlp:
        endpoint: http://otel.quantumcns.ai:80 # The address of the second collector
        tls:
          insecure: true

    service:
      pipelines:
        metrics:
          receivers: [prometheus]
          processors: [transform, attributes/clustername]
          exporters: [debug, otlp]
``` 