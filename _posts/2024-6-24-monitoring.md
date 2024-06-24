---
layout: post
title: 쿠버네티스에서 Application을 모니터링하기
subtitle: 쿠버네티스 입문자들을 위한 자료
cover-img: /assets/img/post_background.jpg
thumbnail-img: /assets/post_images/grafana_1.jpeg
share-img: /assets/post_images/grafana_1.jpeg
tags: [k8s, prometheus, grafana]
author: Alex
preview: "preview test"
---

# 쿠버네티스에서 Application을 모니터링하기
## 목차
- [쿠버네티스에서 Application을 모니터링하기](#쿠버네티스에서-application을-모니터링하기)
  - [목차](#목차)
    - [참고 자료](#참고-자료)
  - [Prometheus](#prometheus)
    - [주요개념](#주요개념)
    - [Metric의 종류](#metric의-종류)
    - [Exporter의 종류](#exporter의-종류)
    - [Jobs and Instances](#jobs-and-instances)
    - [PromQL](#promql)
  - [Prometheus Operator](#prometheus-operator)
    - [정의](#정의)
  - [Grafana](#grafana)
  - [k8s 클러스터 요약 Dashboard 설명](#k8s-클러스터-요약-dashboard-설명)
    - [클러스터 요약](#클러스터-요약)
    - [노드 요약 정보 테이블](#노드-요약-정보-테이블)
    - [노드별 상세 정보](#노드별-상세-정보)
      - [Disk Space Used Basic](#disk-space-used-basic)
      - [Network Sockstat](#network-sockstat)
      - [Open FIle Descriptor/Context switches](#open-file-descriptorcontext-switches)
  - [Custom Metric을 이용한 Application 모니토링(예제)](#custom-metric을-이용한-application-모니토링예제)

### 참고 자료
- [https://prometheus-operator.dev](https://prometheus-operator.dev/)
- [https://prometheus.io](https://prometheus.io/)

<div style="page-break-after: always;"></div>

## Prometheus
### 주요개념

**Metrics**: Prometheus에서 데이터를 수집하는 기본 단위입니다. 각 메트릭은 특정한 이름과 레이블 세트를 가지며, 시간에 따른 값의 변화를 기록합니다.

**Labels**: 메트릭을 구분하고 쿼리할 때 사용하는 키-값 쌍입니다. 예를 들어, 서버의 메트릭을 수집할 때 '서버의 위치', '서버의 이름' 등을 레이블로 사용할 수 있습니다.

**Time Series**: 동일한 메트릭 이름과 레이블 세트를 가진 데이터 포인트의 시퀀스입니다. Prometheus는 이 타임 시리즈 데이터를 사용하여 시간 경과에 따른 메트릭의 변화를 추적합니다.

**Query**: Prometheus는 자체 쿼리 언어인 PromQL을 사용하여 저장된 데이터를 검색하고 계산할 수 있습니다. 이를 통해 사용자는 복잡한 쿼리를 작성하여 메트릭 데이터에서 유용한 정보를 추출할 수 있습니다.

**Alerts**: Prometheus는 정의된 조건에 따라 알림을 발생시킬 수 있습니다. 사용자는 규칙을 설정하여 특정 메트릭이 주어진 임계값을 초과하거나 특정 상태에 도달했을 때 알림을 받을 수 있습니다.

**Exporters**: Prometheus는 다양한 시스템과 서비스로부터 메트릭을 수집하기 위해 Exporter라는 에이전트를 사용합니다. 각 Exporter는 특정 서비스나 시스템에 대한 메트릭을 수집하여 Prometheus가 이해할 수 있는 형식으로 변환합니다.

**Scraping**: Prometheus 서버는 설정된 간격으로 Exporter나 수집 대상 시스템에 HTTP 요청을 보내어 메트릭을 "스크레이핑"합니다. 이 과정을 통해 Prometheus는 정기적으로 메트릭 데이터를 수집하고 저장합니다.

**Storage**: Prometheus는 수집된 데이터를 로컬 스토리지에 시계열 데이터베이스 형식으로 저장합니다. 이 데이터는 후에 쿼리, 시각화 및 알림 생성을 위해 사용됩니다.
  
![](/images/mon_2.jpeg)

### Metric의 종류
Prometheus에서 사용하는 메트릭 종류는 주로 네 가지로 분류됩니다. 각각의 메트릭 유형은 서로 다른 유형의 정보를 모니터링하고 측정하는 데 사용됩니다:

**Counter:**

- 카운터는 단순히 증가만 하는 메트릭으로, 주로 요청 수, 완료된 작업 수, 에러 발생 수 등을 측정하는 데 사용됩니다.
- 카운터 값은 절대 감소하지 않으며 (재시작 등의 경우를 제외하고), 주로 누적 값을 관찰하는 데 사용됩니다.
  
**Gauge:**

- 게이지는 특정 시점에서의 값을 측정하는 메트릭으로, 값이 증가하거나 감소할 수 있습니다.
- 메모리 사용량, 현재 활성 세션 수, 온도 등 현재 상태를 나타내는 값을 모니터링하는 데 주로 사용됩니다.

**Histogram:**

- 히스토그램은 샘플을 관찰하고 이를 구성된 버킷에 분류하여, 샘플들의 분포를 관찰할 수 있게 해줍니다.
- 주로 응답 시간, 요청 크기 등을 측정하는 데 사용되며, 각 버킷은 특정 범위의 값을 나타냅니다.
- 히스토그램은 측정 값의 총합과 버킷 별 누적 카운트를 제공합니다.

**Summary:**

- 서머리도 샘플의 분포를 측정하는 데 사용되지만, 히스토그램과는 다르게 특정 분위수(quantiles)의 관찰 값을 제공합니다.
- 응답 시간의 90번째 백분위수 같은 지표를 계산할 때 유용합니다.
- 서머리는 관측된 값의 총합과 카운트, 그리고 설정된 분위수에 따른 값을 제공합니다.

이러한 메트릭 유형들은 각기 다른 모니터링 목적에 따라 선택하여 사용될 수 있으며, Prometheus의 유연성과 강력한 데이터 수집 및 쿼리 능력의 기반이 됩니다.
![](/images/mon_1.jpeg)

### Exporter의 종류
- Node Exporter
- Client Library
- DCGM

### Jobs and Instances
**Jobs:**

- Job은 Prometheus가 수집하는 메트릭들의 한 그룹을 의미합니다. 일반적으로 job은 동일한 유형의 여러 인스턴스에서 메트릭을 수집할 때 사용되는 레이블입니다.
- 예를 들어, 여러 서버에서 실행 중인 동일한 애플리케이션의 메트릭을 수집하는 경우, 이러한 서버 그룹 전체를 나타내기 위해 "job" 레이블을 사용할 수 있습니다. 모든 서버가 "api-server"라는 job에 속할 수 있으며, Prometheus는 이 job으로부터 메트릭을 수집합니다.
- Job은 Prometheus 설정에서 정의됩니다. 이를 통해 Prometheus는 어떤 엔드포인트에서 메트릭을 수집해야 하는지, 그리고 해당 메트릭을 어떤 job 이름으로 그룹화할지를 알게 됩니다.

**Instances:**

- Instance는 job 내에서 개별적으로 메트릭을 수집하는 대상을 의미합니다. 보통은 하나의 애플리케이션 인스턴스나 하나의 서버를 가리킵니다.
- 예를 들어, "api-server" job이 여러 서버에서 실행 중이라면, 각 서버는 "api-server" job의 별도 인스턴스로 간주됩니다. - Prometheus는 각 인스턴스에서 메트릭을 독립적으로 수집하고, 이 메트릭들을 인스턴스 레이블로 구분합니다.
Instance는 일반적으로 호스트명과 포트 번호를 포함하는데, 이는 Prometheus가 메트릭을 수집할 정확한 위치를 나타냅니다

![](/images/mon_3.png)


### PromQL
[https://prometheus.io/docs/prometheus/latest/querying/basics/](https://prometheus.io/docs/prometheus/latest/querying/basics/)

<div style="page-break-after: always;"></div>

## Prometheus Operator
### 정의
Prometheus Operator는 Kubernetes 클러스터 내에서 Prometheus를 보다 쉽게 배포하고 관리할 수 있도록 설계된 도구입니다. Kubernetes 리소스와 밀접하게 통합되어 있으며, Kubernetes API를 확장하여 Prometheus 설정을 자동화하고 간소화합니다. 다음은 Prometheus Operator의 주요 개념들입니다:

**Custom Resource Definitions (CRDs):**

- Prometheus Operator는 Custom Resource Definitions (CRDs)를 사용하여 Prometheus 관련 설정을 Kubernetes 네이티브 리소스처럼 관리할 수 있게 합니다. 이를 통해 사용자는 Kubernetes 리소스를 정의하고 조작하는 방식으로 Prometheus 인스턴스, Alertmanager, 서비스 모니터링 등을 구성할 수 있습니다.
  
**Prometheus Custom Resource:**

- Prometheus 리소스는 클러스터 내에서 실행할 Prometheus 인스턴스를 정의합니다. 사용자는 이 리소스를 통해 Prometheus 서버의 버전, 설정, 스토리지 요구사항 등을 지정할 수 있습니다.

**ServiceMonitor:**

- ServiceMonitor 리소스는 Prometheus가 모니터링할 서비스를 발견하고 설정하는 방법을 정의합니다. 이는 서비스의 레이블을 기반으로 하며, Prometheus가 어떤 서비스를 스크래핑할지, 어떤 엔드포인트와 포트를 사용할지 등을 지정합니다.

**Alertmanager Custom Resource:**

- Alertmanager 리소스를 통해 클러스터 내에서 실행되는 Alertmanager 인스턴스를 구성할 수 있습니다. 이를 통해 Prometheus로부터의 알림을 관리하고, 알림의 라우팅, 그룹화, 중복 제거, 발송 방법 등을 설정할 수 있습니다.

**PrometheusRule:**

- PrometheusRule 리소스는 알림 규칙과 레코딩 규칙을 정의합니다. 이 리소스를 통해 사용자는 Prometheus 쿼리 언어(PromQL)를 사용하여 메트릭에 기반한 알림 규칙과 새로운 타임 시리즈 데이터를 생성하는 규칙을 설정할 수 있습니다.

**Operator Pattern:**

- Prometheus Operator는 Kubernetes의 Operator 패턴을 따릅니다. 이는 복잡한 애플리케이션을 Kubernetes 상에서 운영하는 논리를 코드로 구현한 것으로, 사용자가 애플리케이션을 보다 쉽게 배포하고 관리할 수 있게 해줍니다.

![](/images/prom_operator_1.svg)



<div style="page-break-after: always;"></div>

## Grafana

Grafana는 강력한 시각화 및 분석 플랫폼으로, 다양한 데이터 소스에서 데이터를 수집하여 인터랙티브한 대시보드 형태로 시각화합니다. 사용자는 Grafana를 사용하여 데이터를 쉽게 조회, 모니터링 및 분석할 수 있습니다. 다음은 Grafana의 주요 개념들입니다:

**Dashboards:**

- 대시보드는 여러 위젯 또는 패널을 통해 시각화된 데이터의 모음입니다. 사용자는 대시보드를 통해 다양한 데이터 소스로부터의 정보를 한눈에 볼 수 있습니다.
- 각 대시보드는 특정 목적이나 관점을 반영하여 구성될 수 있으며, 사용자는 필요에 따라 여러 대시보드를 생성하고 관리할 수 있습니다.

**Panels:**

- 패널은 대시보드 내에서 개별적인 차트, 그래프, 테이블 등의 시각화를 보여주는 컴포넌트입니다. 각 패널은 특정 쿼리에 기반하여 데이터를 시각화하며, 다양한 시각화 유형(예: 그래프, 히트맵, 가이지 등)을 지원합니다.

**Data Sources:**

- Grafana는 다양한 데이터 소스와 연동될 수 있습니다. Prometheus, InfluxDB, Elasticsearch, MySQL, PostgreSQL 등 다양한 데이터베이스 및 모니터링 툴과의 연동이 지원됩니다.
- 사용자는 Grafana에 데이터 소스를 추가하고, 해당 데이터 소스에서 데이터를 조회하여 대시보드를 생성할 수 있습니다.

**Query:**

- 패널 내에서 데이터를 시각화하기 위해, Grafana는 데이터 소스에 쿼리를 실행합니다. 이 쿼리는 사용자가 정의하며, 대시보드에 표시할 데이터를 결정합니다.

**Alerts:**

- Grafana는 데이터 포인트가 사전 정의된 임계값을 초과할 때 알림을 발송하는 기능을 제공합니다. 이를 통해 사용자는 시스템의 문제점을 신속하게 인지하고 대응할 수 있습니다.

**Plugins:**

- Grafana는 다양한 플러그인을 지원하여 기능을 확장할 수 있습니다. 데이터 소스 플러그인, 패널 플러그인, 앱 플러그인 등을 설치하여 Grafana의 기능을 확장하고 사용자 정의화할 수 있습니다.

**사용자 및 권한 관리:**

- Grafana는 사용자 계정 관리, 사용자 그룹 및 역할 기반의 액세스 컨트롤을 지원합니다. 이를 통해 대시보드, 패널 및 데이터 소스에 대한 세밀한 접근 제어가 가능합니다.
- Grafana는 이러한 개념들을 통해 복잡한 데이터를 쉽게 시각화하고, 효율적인 데이터 분석 및 모니터링을 가능하게 합니다.

![](/images/grafana_1.jpeg)

<div style="page-break-after: always;"></div>

## k8s 클러스터 요약 Dashboard 설명

### 클러스터 요약

Panel 이름 |	설명 
----------|--------
정상 노드 수 | 상태가 Ready인 노드 수
비정상 노드 수 | 상태가 Ready가 아닌 노드 수
생성가능한 파드 대비 배포수 | 생성가능한 파드 대비 현재배포된 파드 수
클러스터 파드 Capacity |  생성가능한 파드수
CPU 토탈 코어 대비 요청량 | requested/total
클러스터 CPU Capacity | 클러스터 총 CPU 코어
메모리 토탈 코어 대비 요청량 | requested/total
클러스터 메모리 Capacity | 클러스터 총 메모리 용량
GPU 토탈 코어 대비 요청량 | requested/total
클러스터 GPU Capacity | 클러스터 총 GPU 갯수



### 노드 요약 정보 테이블

Column 이름 |	설명
--------------|--------
ip | 노드 아이피 주소
hostname | 노드 호스트 이름
uptime | 마지막 부팅 이후 경과 시간
memory | 노드 총 메모리
CPU Cores | 최근 5분간 CPU 시스템에서 쓴 사용량 
5m load | 최근 5분간 CPU 전체 사용량
Memory used | 메모리 사용량
Partition used | 디스크 사용량
Disk read | 초당 평균 디스크에서 읽은 데이터 용량
Disk write | 초당 평균 디스크에 쓴 데이터 용량
CurrEstab | 연결되어 있는 TCP connection 수
TCP-tw | 종료된 TCP connection 수
Download* | 초당 평균 다운로드 트래픽
Upload* | 초당 평균 업로드 트래픽
  
### 노드별 상세 정보
Panel 이름 |	설명 
----------|--------
Uptime  | 마지막 부팅 이후 경과 시간
CPU Cores | 총 CPU 코어 개수
Total RAM | 총 메모리 용량
CPU Busy | CPU 사용량
Usage RAM | 메모리 사용량
Used Max Mount | 가장 큰 파티션 기준 디스크 사용량
Used SWAP | 스왑 메모리 사용량

#### Disk Space Used Basic
Column 이름 |	설명
-----------|--------
Device | 디바이스 이름
Filesystem | 파일시스템 형식
Mounted on | 파일 시스템상 마운트된 위치
Size | 디스크 크기
Avail | 남은 용량
Used | 사용량


Panel 이름 |	설명 
----------|--------
Internet traffic per hour All | nic별 네트워크 트래픽/시간
CPU% Basic | 모드(user,system,io, total)별 CPU 사용량
Memory Basic | 메모리 사용량(free, used, total)
Network bandwidth usage per second All | nic별 네트워크 bandwidth
System Load | 1m, 5m, 15m 단위 시스템 로드(cpu 사용량)
Disk R/W Data | 디스크에 읽고/쓴 데이터 bytes
Disk Space Used% Basic | 디스크 사용량
Disk IOps Completed | 디스크 io 횟수/초
Time Spent Doing I/Os | 디스크의 io에 소요한 시간
Disk R/W Time | io별 소요 시간

#### Network Sockstat
항목 이름 |	설명 
----------|--------
TCP-tw | 종료된 TCP connection 수
CurrEstab | 연결되어 있는 TCP connection 수
TCP_inuse | 사용중인 TCP socket 수
UDP_inuse | 사용중인 UDP socket 수
TCP_alloc | alloc 상태인 TCP socket 수
TCP_passive_opens | 해당 노드에서 listen 중인 포트에 연결하여 열린 TCP socket
TCP_active_opens | 해당 노드로 부터 시작되어 열린 TCP socket
TCP_inSegs | 수신한 TCP segment 수
TCP_retransSegs | 재전송한 TCP segment 수
TCP_outSegs | 송신한 TCP segment 수

#### Open FIle Descriptor/Context switches
항목 이름 |	설명 
----------|--------
file descriptor | 할당된 file descriptor 수
context switches | 이뤄진 context switch 수

## Custom Metric을 이용한 Application 모니토링(예제)

Server.go
```
package main

import (
	"fmt"
	"net/http"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

var pingCounter = prometheus.NewCounter(
	prometheus.CounterOpts{
		Name: "ping_request_count",
		Help: "No of request handled by Ping handler",
	},
)

func ping(w http.ResponseWriter, req *http.Request) {
	pingCounter.Inc()
	fmt.Fprintf(w, "pong")
}

func main() {
	prometheus.MustRegister(pingCounter)

	http.HandleFunc("/ping", ping)
	http.Handle("/metrics", promhttp.Handler())
	http.ListenAndServe(":8090", nil)
}
```

Dockerfile
```
FROM golang:1.22.1-alpine AS build
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN GOARCH=amd64 go build -o server .

FROM alpine:latest
WORKDIR /app
COPY --from=build /app/server .
EXPOSE 8090
CMD ["./server"]

```



Deployment.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prom-example
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prom-example
      release: qks-monitoring
  template:
    metadata:
      labels:
        app: prom-example
        release: qks-monitoring
    spec:
      containers:
      - name: mymetric
        image: registry.smg.quantumcns.io/qms/prom_example:v0.2
        ports:
        - name: metrics
          containerPort: 8090
```

Service.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: prom-example
  labels:
    app: prom-example
    release: qks-monitoring
spec:
  selector:
    app: prom-example  
    release: qks-monitoring
  ports:
    - name: metrics
      protocol: TCP
      port: 8090   
      targetPort: metrics
  type: ClusterIP
```

ServiceMonitor.yaml
```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  annotations:
  labels:
    app: prom-example
    release: qks-monitoring
  name: prom-example
spec:
  endpoints:
  - path: /metrics
    port: metrics
    interval: 15s
  namespaceSelector:
    matchNames:
    - qks-monitoring
  selector:
    matchLabels:
      app: prom-example
      release: qks-monitoring
```

alertmanagerconfig.yaml
```
apiVersion: monitoring.coreos.com/v1alpha1
kind: AlertmanagerConfig
metadata:
  name: config-example
  labels:
    alertmanagerConfig: example
spec:
  route:
    groupBy: ['job']
    groupWait: 30s
    groupInterval: 5m
    repeatInterval: 12h
    receiver: 'webhook'
  receivers:
  - name: 'webhook'
    webhookConfigs:
    - url: https://webhook.site/6016d666-7cc1-4f81-bb8f-442ef2a9a400
      send_resolved: false
```
prometheusrule.yaml
```
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  creationTimestamp: null
  labels:
    release: qks-monitoring
    prometheus: example
    role: alert-rules
  name: prometheus-example-rules
spec:
  groups:
  - name: Count greater than 5
    rules:
    - alert: CountGreaterThan5
      expr: ping_request_count > 5
      for: 10s
```