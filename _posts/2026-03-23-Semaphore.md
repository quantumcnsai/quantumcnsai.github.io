---
layout: post
title: "Ansible 자동화 운영의 완성: Semaphore 이해 및 구축 상세 가이드"
cover-img: /assets/img/post_background.jpg
thumbnail-img: /assets/post_images/otel_multicluster.png
share-img: /assets/post_images/otel_multicluster.png
tags: [ansible, automation, semaphore, devops, infrastructure, linux, mariadb, systemd, git, docker]
author: Cody
preview: "CLI 기반 Ansible 운영의 한계를 넘어, Web UI 기반 자동화 운영 플랫폼 Semaphore의 구조, 구성 요소, 구축 절차, 운영 포인트를 상세히 설명드립니다."
---

## 1. 들어가며: 왜 ‘운영 플랫폼’이 필요한가?

인프라 운영 환경에서는 서버 설정, 애플리케이션 배포, 점검, 구성 변경과 같은 반복적인 작업이 지속적으로 발생합니다.  
이러한 작업을 자동화하기 위해 **Ansible**과 같은 도구가 널리 사용되고 있습니다.

Ansible은 에이전트 설치 없이 SSH 기반으로 원격 서버를 제어할 수 있고, 사람이 직접 명령어를 반복 실행하는 대신 **Playbook**이라는 선언형 문서를 통해 작업을 표준화할 수 있다는 장점이 있습니다. 이 때문에 인프라 자동화, 운영 표준화, 배포 일관성 확보 측면에서 매우 강력한 도구로 평가받고 있습니다.

하지만 실제 운영 환경에서는 단순히 “자동화 도구를 사용한다”는 것만으로 충분하지 않은 경우가 많습니다. 특히 CLI 기반으로만 Ansible을 사용할 경우 다음과 같은 문제가 발생할 수 있습니다.

- **작업 이력 관리의 어려움:** 누가 언제 어떤 Playbook을 어떤 대상에 실행했는지 체계적으로 추적하기 어렵습니다.
- **협업 환경에서의 권한 통제 한계:** 여러 운영자가 동시에 사용하는 환경에서 사용자별 역할과 권한을 세밀하게 나누기 어렵습니다.
- **실행 결과 가시성 부족:** 실행 성공/실패 여부나 로그를 일관된 방식으로 관리하기 어렵습니다.
- **운영 표준화의 어려움:** 특정 담당자의 셸 히스토리나 로컬 환경에 의존하는 방식은 팀 단위 운영에 적합하지 않습니다.

이러한 문제는 단순한 불편함을 넘어, 운영 안정성·감사 가능성·협업 효율성에 직접적인 영향을 미칩니다.  
따라서 자동화는 단순 실행 수준을 넘어, **운영 관점에서 통제 가능한 플랫폼 형태로 확장**할 필요가 있습니다.

바로 이 지점에서 **Ansible Semaphore**를 검토할 수 있습니다.

---

## 2. Ansible과 Semaphore를 함께 이해하기

### 2.1 Ansible이란 무엇인가?

Ansible은 IT 자동화 도구입니다.  
서버 프로비저닝, 패키지 설치, 설정 파일 배포, 서비스 재시작, 애플리케이션 배포 등 다양한 작업을 자동화할 수 있습니다.

Ansible의 핵심 구성 요소는 일반적으로 다음과 같습니다.

- **Inventory:** 작업 대상 서버 목록
- **Module:** 실제 작업을 수행하는 기능 단위
- **Playbook:** 어떤 서버에 어떤 작업을 어떤 순서로 수행할지 정의한 YAML 문서
- **Role:** Playbook을 재사용 가능한 구조로 모듈화한 단위

Ansible 자체는 매우 강력하지만, 기본적으로는 CLI 중심 도구이기 때문에 팀 단위 운영 환경에서는 별도의 관리 계층이 필요해질 수 있습니다.

### 2.2 Ansible Semaphore란 무엇인가?

Ansible Semaphore는 Ansible Playbook을  
**웹 UI 기반으로 실행하고 관리할 수 있도록 하는 오픈소스 플랫폼**입니다.

기존 CLI 중심의 Ansible 위에 **관리 계층**을 추가하여 다음과 같은 기능을 제공합니다.

- Playbook 실행 제어
- 사용자 및 권한 관리
- 작업 이력 저장
- 실행 로그 확인
- 운영 표준화 지원

즉, Semaphore는 Ansible을 대체하는 도구가 아니라,  
Ansible을 보다 **운영 친화적인 형태로 관리**할 수 있게 해주는 플랫폼이라고 이해하시면 됩니다.

### 2.3 왜 Semaphore가 필요한가?

CLI 기반 Ansible 운영은 개인 단위 작업에는 효율적일 수 있습니다. 그러나 팀 단위 운영 환경에서는 다음과 같은 구조적 한계가 있습니다.

| 문제 | 설명 |
|------|------|
| 실행 이력 관리 부족 | 누가 언제 어떤 작업을 수행했는지 추적이 어렵습니다. |
| 협업 한계 | 팀 단위 운영 환경에서 사용자별 통제와 역할 분리가 어렵습니다. |
| 가시성 부족 | 실행 상태 및 결과 확인이 비효율적입니다. |
| 운영 표준화 부족 | 작업 방식이 담당자별로 달라질 수 있습니다. |

Semaphore는 이러한 문제를 해결하기 위한 관리 계층 역할을 수행합니다.

### 2.4 Semaphore의 주요 기능

- **Playbook 실행 및 Job 관리**
- **사용자 및 권한 관리(RBAC)**
- **실행 로그 및 이력 저장**
- **웹 UI 기반 운영**
- **프로젝트 단위 관리**
- **Git 기반 Playbook 연동**
- **알림 및 인증 확장 기능 지원**

---

## 3. 전체 아키텍처 이해

Semaphore 기반 자동화 운영 구조는 아래와 같이 이해할 수 있습니다.

```text
User → Web UI → Semaphore Server → Ansible → Target Infrastructure
                              ↓
                           MariaDB
```

이 구조에서 각 구성 요소의 역할은 다음과 같습니다.

- **User**  
  브라우저를 통해 Semaphore UI에 접근하는 사용자입니다.

- **Web UI / Semaphore Server**  
  사용자의 실행 요청을 받아 관리하고, 내부적으로 Ansible 작업을 실행합니다.

- **Ansible**  
  실제로 타깃 서버에 접속하여 작업을 수행하는 자동화 엔진입니다.

- **MariaDB**  
  사용자 정보, 프로젝트 정보, 작업 이력, 로그 등 운영 데이터를 저장합니다.

### 3.1 왜 데이터베이스가 필요한가?

Semaphore는 단순히 화면만 띄우는 UI가 아닙니다.  
사용자 정보, 프로젝트, 실행 이력, Job 상태 등 **상태 정보(State)**를 지속적으로 저장해야 합니다.

즉, Semaphore는 **Stateless 구조가 아니라 Stateful 플랫폼**이기 때문에 DB가 반드시 필요합니다.  
이번 구성에서는 MySQL 계열인 **MariaDB**를 사용하였습니다.

---

## 4. 구성 요소 상세 설명

### 4.1 Semaphore Server

Semaphore Server는 전체 시스템의 중심입니다.

주요 역할은 다음과 같습니다.

- Web UI 및 API 제공
- 사용자 요청 수신
- 프로젝트 및 Job 관리
- 내부적으로 Ansible 실행 orchestration 수행

Semaphore는 Go 기반으로 작성되어 비교적 가볍고, 단일 바이너리 형태로 배포되는 장점이 있습니다.

### 4.2 MariaDB

MariaDB는 Semaphore의 운영 데이터를 저장하는 저장소입니다.

예를 들어 다음과 같은 정보가 저장됩니다.

- 사용자 계정 정보
- 프로젝트 정보
- 실행 이력
- Job 로그
- 설정 정보 일부

DB를 별도 구성함으로써 운영 정보의 중앙 관리가 가능해지고, 서비스 재시작이나 시스템 재부팅 이후에도 상태를 유지할 수 있습니다.

### 4.3 systemd

systemd는 리눅스에서 표준적으로 사용되는 서비스 관리자입니다.

이번 구성에서는 Semaphore를 단순히 수동 실행하는 것이 아니라 **서비스 형태로 운영**하기 위해 systemd를 사용합니다. 
이를 통해 다음이 가능해집니다.

- 서버 부팅 시 자동 실행
- 비정상 종료 시 자동 재시작
- 서비스 상태 확인
- 운영 표준화

### 4.4 SSH Port Forwarding

초기 접근 단계에서는 외부에 3000 포트를 직접 노출하기보다,  
SSH 터널링을 통해 로컬에서 안전하게 UI에 접근할 수 있습니다.

예를 들어 로컬 PC에서 3000 포트를 열고 이를 원격 서버의 3000 포트로 전달하면, 브라우저에서는 `localhost:3000`으로 접속하면서도 실제로는 원격 서버의 Semaphore UI에 연결됩니다.

### 4.5 Git

Ansible Playbook은 코드이기 때문에 Git을 통해 버전 관리하는 것이 일반적입니다.

Semaphore를 통해 Git 저장소의 Playbook을 관리하면 다음과 같은 장점이 있습니다.

- 변경 이력 추적
- 협업 용이
- Rollback 가능
- 운영 표준화

### 4.6 Docker

첨부 로그에는 Docker 설치와 Compose 실행 내용도 포함되어 있습니다.

이는 Semaphore 자체 실행과 직접적으로 동일 단계는 아니지만, 이후 자동화 대상 환경이나 관련 운영 환경을 컨테이너 기반으로 확장할 가능성을 보여줍니다. 
즉, 인프라 자동화 범위를 VM/물리 서버뿐 아니라 컨테이너 환경까지 확장할 수 있다는 의미를 가집니다.

---

## 5. 구축 과정 

### 5.1 서비스 계정 생성

먼저 Semaphore 전용 시스템 계정을 생성합니다.

```bash
sudo adduser --system --group --home /home/semaphore semaphore
```

이 단계의 목적은 **권한 분리**입니다.  
서비스를 root 계정으로 직접 실행하지 않고, 최소 권한 원칙에 따라 별도 계정으로 분리하면 보안성과 운영 안정성이 향상됩니다.

---

### 5.2 MariaDB 설치

Semaphore의 상태 정보를 저장할 데이터베이스를 설치합니다.

```bash
sudo apt update
sudo apt install mariadb-server
sudo mysql_secure_installatio
```

이 단계에서는 MariaDB 설치 후 초기 보안 설정을 수행하게 됩니다.

---

### 5.3 DB 생성 및 확인

MariaDB에 접속한 뒤 Semaphore 전용 데이터베이스를 생성합니다.

```sql
CREATE DATABASE semaphore_db;
SHOW DATABASES;
```

`SHOW DATABASES;`를 통해 실제로 `semaphore_db`가 생성되었는지 확인할 수 있습니다.

---

### 5.4 DB 권한 설정

다음으로 Semaphore가 사용할 전용 DB 계정을 생성하고 권한을 부여합니다.

```sql
GRANT ALL PRIVILEGES ON semaphore_db.* TO semaphore_user@localhost IDENTIFIED BY '설정할 비밀번호';
FLUSH PRIVILEGES;
```

이 과정은 DB root 계정을 직접 사용하는 대신, 서비스 전용 계정을 분리하여 사용하는 운영 방식입니다. 
보안상 더 바람직한 방식입니다.

---

### 5.5 Semaphore 설치

Semaphore 패키지를 다운로드하고 설치합니다.

```bash
wget https://github.com/semaphoreui/semaphore/releases/download/v2.17.15/semaphore_2.17.15_linux_amd64.deb
sudo apt install ./semaphore_2.17.15_linux_amd64.deb
```

```text
https://github.com/ansible-semaphore/semaphore/releases/latest
```

실제 설치는 `v2.17.15` 패키지를 직접 다운로드하여 진행하였습니다.

---

### 5.6 초기 설정 (전체 옵션 포함)

설치 후 다음 명령으로 초기 설정을 진행합니다.

```bash
semaphore setup
```

로그 기준으로 설정 과정에서 다음 항목이 등장합니다.

- DB Engine: MySQL
- Host: `127.0.0.1:3306`
- User: `semaphore_user`
- Password: `twinkle`
- DB Name: `semaphore_db`
- Playbook path: `/tmp/semaphore`
- Public URL: 공란
- Email alerts: `no`
- Telegram / Slack / Rocket.Chat / Microsoft Teams: 기본값
- LDAP: `no`
- Config output directory: 기본값(`/home/ubuntu`)

이 초기 설정 과정은 단순 DB 연결만 의미하는 것이 아니라, Playbook 경로, 알림 연동, 인증 연동 가능성까지 포함한 운영 초기화 과정이라고 볼 수 있습니다.

---

### 5.7 설정 파일 및 권한 정리

초기 설정 후 생성된 `config.json` 파일의 권한과 위치를 정리합니다.

```bash
sudo chown semaphore:semaphore config.json
sudo mkdir /etc/semaphore
sudo chown semaphore:semaphore /etc/semaphore/
mv config.json /etc/semaphore/
sudo mv config.json /etc/semaphore/
ls -l /etc/semaphore/
```

여기서 설정 파일을 `/etc/semaphore` 아래로 이동하는 이유는,  
리눅스 시스템에서 서비스 설정 파일을 표준 경로 아래에 두기 위함입니다.

또한 `chown semaphore:semaphore`를 통해 Semaphore 서비스 계정이 해당 설정 파일을 읽을 수 있도록 권한을 정리합니다.

---

### 5.8 서비스 실행

설정이 완료되면 Semaphore를 직접 실행하여 정상 동작 여부를 확인할 수 있습니다.

```bash
semaphore server --config /etc/semaphore/config.json
```

첨부 로그 기준으로 다음과 같은 출력이 확인됩니다.

- `Loading config`
- `Validating config`
- `MySQL semaphore_user@127.0.0.1:3306 semaphore_db`
- `Tmp Path (projects home) /tmp/semaphore`
- `Port :3000`
- `Server is running`

즉, 설정 파일이 정상적으로 로드되었고, DB 연결과 포트 바인딩이 정상 동작함을 확인할 수 있습니다.

---

### 5.9 SSH 터널링

보안을 위해 외부 포트를 직접 열지 않고 SSH 터널링으로 접근합니다.

```bash
ssh -L 3000:localhost:3000 ubuntu@SERVER_IP -i <pem key>
```

이후 브라우저에서 다음과 같이 접속할 수 있습니다.

```text
http://localhost:3000
```

이 방식은 초기 테스트나 제한된 접근 환경에서 매우 유용합니다.

---

### 5.10 systemd 설정

Semaphore를 지속적으로 운영하려면 systemd 서비스로 등록하는 것이 좋습니다.

```ini
[Unit]
Description=Ansible Semaphore
Documentation=https://docs.ansible-semaphore.com/
Wants=network-online.target
After=network-online.target
ConditionPathExists=/usr/bin/semaphore
ConditionPathExists=/etc/semaphore/config.json

[Service]
ExecStart=/usr/bin/semaphore server --config /etc/semaphore/config.json
ExecReload=/bin/kill -HUP $MAINPID
Restart=always
RestartSec=10s
User=semaphore
Group=semaphore

[Install]
WantedBy=multi-user.target
```

이후 다음 명령으로 적용합니다.

```bash
systemctl daemon-reload
systemctl enable --now semaphore.service
systemctl status semaphore.service
```

로그 기준으로 서비스는 `active (running)` 상태로 정상 동작하였습니다.

---

### 5.11 Git 설정

첨부 로그에는 Git 초기화 및 Semaphore 계정에 대한 `safe.directory` 설정이 포함되어 있습니다.

```bash
git init
git add .
git commit -m "init"

sudo -u semaphore git config --global --add safe.directory /home/ubuntu/k8s
sudo -u semaphore git config --global --add safe.directory /home/ubuntu/k8s/.git
```

이는 Semaphore 서비스 계정이 해당 Git 저장소를 신뢰하도록 설정하는 작업입니다.  
멀티 사용자 환경에서는 Git이 저장소 소유권을 엄격하게 보기 때문에, 서비스 계정과 저장소 소유 계정이 다를 경우 이런 설정이 필요할 수 있습니다.

---

### 5.12 Docker 구성

첨부 로그 마지막에는 Docker 설치 및 Compose 실행 기록이 포함되어 있습니다.

```bash
curl -fsSL https://get.docker.com | sh
sudo docker compose -f /home/ubuntu/docker-compose.yaml up -d
docker compose up -d
```

이 단계는 Semaphore 본체 설치의 핵심 절차라기보다,  
자동화 대상 또는 부가 운영 환경을 컨테이너 기반으로 다루기 위한 확장 단계로 이해하시면 됩니다.

---

## 6. 운영 설계 포인트

### 6.1 권한 분리

- 서비스 계정 분리
- DB 계정 분리

운영 환경에서는 최소 권한 원칙이 중요합니다.  
Semaphore 서비스 계정과 DB 계정을 분리하면 보안 사고 발생 시 영향 범위를 줄일 수 있습니다.

### 6.2 보안

- SSH 터널 기반 접근
- 포트 노출 최소화
- 민감정보 마스킹 필요

실제 외부 공개 문서에서는 비밀번호, IP, 키 파일명 등 민감 정보를 그대로 노출하지 않는 것이 중요합니다.

### 6.3 안정성

- systemd 자동 재시작
- 서비스 상태 확인 가능
- 서버 재부팅 이후 자동 실행

즉, 운영 환경에서는 단순 실행보다 서비스화가 중요합니다.

### 6.4 확장성

- Kubernetes 연동 가능
- CI/CD 연계 가능
- Git 기반 자동화 체계 확장 가능
- 알림 및 인증 연동 확장 가능

Semaphore는 단순한 1회성 UI가 아니라, 이후 운영 자동화 체계를 넓혀갈 수 있는 기반 도구가 될 수 있습니다.

---

## 7. 마무리

Ansible은 강력한 자동화 도구이지만,  
운영 환경에서는 **실행 그 자체보다 관리 체계**가 더 중요해질 수 있습니다.

Semaphore는 이러한 지점을 보완하여,  
Ansible을 단순 CLI 도구에서 **운영 플랫폼**으로 확장할 수 있도록 돕습니다.

특히 사용자 관리, 실행 이력, 서비스화, Git 연계, 확장성 측면에서  
팀 단위 인프라 운영 환경에 적합한 구조를 제공할 수 있습니다.

현재 기준으로는 실제 도입 완료가 아니라 검토 및 구축 검증 단계로 볼 수 있지만,  
운영 자동화 체계를 한 단계 확장하는 관점에서 충분히 검토할 가치가 있는 구성입니다.