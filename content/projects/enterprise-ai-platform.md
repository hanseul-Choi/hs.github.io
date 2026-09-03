---
title: "기업용 AI 플랫폼 / OpenAPI 구축/운영"
---

_2025.03 ~ 현재_

## 기술

- Platform: Kubernetes, kubeadm, CRI-O, Cilium, Helm, Argo CD, Jenkins, Kong, Kyverno
- Cloud & Infrastructure: AWS, On-Premise, ALB, NLB, Harbor, Proxy, Firewall
- GPU & AI Serving: H100, A100, V100, MIG, NVLink, InfiniBand, vLLM
- Messaging & Database: Kafka, PostgreSQL, MariaDB, Elasticsearch
- Observability: Prometheus, Grafana, Elasticsearch
- Security: HashiCorp Vault, Kubernetes Secret, TLS
- Languages: Python, Shell Script

## 개요

사내 데이터 기반 생성형 AI와 업무 자동화 기능을 제공하는 기업용 AI 플랫폼을 운영하고 있다.
자체 LLM과 외부 상용 AI(GPT·Claude·Gemini)를 통합해 챗봇뿐 아니라 회의록 작성, 문서
체크리스트, PPT 생성 같은 업무 기능을 제공하며, 민감정보를 안전하게 다루기 위해 개발계·
운영계와 외부 연계 환경을 분리해 운영한다.

보안 요구가 높은 금융권 고객사에는 외부망과 차단된 온프레미스 Kubernetes 환경으로 플랫폼을
구축했고, 외부 고객이 AI 모델을 API로 사용할 수 있도록 별도 AWS OpenAPI 클러스터도 운영한다.

이 환경에서 GPU 기반 모델 서빙, Kubernetes, API Gateway, Kafka, 데이터베이스, CI/CD 및
모니터링 체계를 통합 운영하며, **신규 모델 배포와 서비스 장애 대응을 맡아 플랫폼을 안정적으로
운영하고 있다.**

## 역할

플랫폼 및 Kubernetes 운영

- kubeadm 기반 개발계·운영계·유틸리티 Kubernetes 클러스터 운영, Control Plane과 CPU·GPU Worker Node로 구성된 온프레미스 환경 관리
- CRI-O Container Runtime 및 Cilium CNI 운영, NLP·Vision·STT 등 서비스 영역별 Namespace 관리
- Helm Chart 작성 및 배포 표준화, Argo CD Project로 개발계·운영계 배포 분리
- Deployment/Service/Ingress/ConfigMap/Secret/PVC 관리 · Node Label과 `nodeSelector`로 CPU·GPU 워크로드 배치
- HPA, readiness/liveness probe 및 resource request·limit 운영
- Kong Ingress와 사내 Load Balancer를 연결한 API 진입 경로 관리

GPU 및 모델 서빙

- H100·A100·V100 등 수십 장 규모의 GPU 자원 운영, 자체 LLM 포함 수십 개 모델 서비스 관리
- GPU, HBM, CPU, Memory 자원 사용량 모니터링
- H100 MIG를 활용한 GPU 자원 분할 및 소규모 모델 배치 효율화
- NVLink 기반 동일 노드 내 다중 GPU 모델 서빙, InfiniBand 기반 노드 간 분산 학습·서빙 PoC
- 모델 OOM 발생 시 다중 GPU 할당 및 Node 배치 조정
- vLLM 기반 모델의 요청 대기·배치 처리 병목 분석, 대용량 모델의 NAS 로딩 병목 분석 및 GPU Node Local Cache 구성

CI/CD 및 모델 배포

- 신규 모델별 Dockerfile, Jenkins Pipeline, Helm Chart 구성 · GitLab 소스 기반 Jenkins 이미지 빌드
- Harbor Registry Push 및 Argo CD 배포 연동, 신규 모델의 Ingress·Kong Route·Health Check 구성
- 정기적으로 여러 모델을 개발계·운영계에 배포. 배포 요청부터 API 서빙까지 약 1시간 이내 구성
- 배포 후 장애 발생 시 이미지 버전 변경 및 Argo CD Rollback 수행

네트워크 및 보안

- Kong, 사내 Load Balancer, ALB, NLB, NodePort 통신 경로 운영, 내부망·외부망 방화벽 및 Proxy Whitelist 관리
- CRI-O·애플리케이션·Node별 Proxy·`NO_PROXY` 정책 관리, Kong TLS 인증서와 API Endpoint 운영
- 반복되는 Proxy 관련 사고 대응에 그치지 않고, Kyverno ClusterPolicy로 Deployment 컨테이너
  env에 Proxy 환경변수를 고정 주입해 애플리케이션별 수동 설정 누락을 구조적으로 방지
- Kubernetes Secret 및 환경변수 기반 민감정보 관리, 개발계에 Vault Agent Injector를 적용해
  API Key 등 민감 정보를 사이드카로 자동 주입하는 Secret 중앙관리 체계 도입
- 금융권 고객사 완전 폐쇄망 AI 플랫폼 구축·운영 지원, AWS OpenAPI 클러스터와 온프레미스 모델 서버 간 방화벽 통신 관리

모니터링 및 장애 대응

- Prometheus·Grafana 기반 Pod·Node·GPU 자원 모니터링, Kong 로그로 Endpoint별 5xx·504 오류 집계
- Kafka Consumer Lag·리밸런싱 및 메시지 처리 오류 분석, PostgreSQL·MariaDB Connection과 실행 쿼리 점검
- CPU Throttling, Network/Disk I/O, Load Average 분석, kubelet·CRI-O·Kernel·Core Dump 로그를 연계한 NodeNotReady 분석
- 상시 운영 문의 및 장애 대응, 반복 장애에 대한 점검 명령어·운영 체크리스트 작성

## 구조도

_실제 구조도 이미지는 추후 추가 예정입니다. 우선 텍스트로 정리합니다._

```
                           [사내 사용자]
                                  │
                       [사내 AI 플랫폼 Web] — 사내 Load Balancer
                                  │
                          [Kong Gateway] TLS / Auth / Routing
        ┌─────────────────────────┼────────────────────────┐
        ▼                         ▼                        ▼
 [개발계 Cluster]          [운영계 Cluster]        [외부 AI 서비스]
  kubeadm/CRI-O/Cilium     kubeadm/CRI-O/Cilium    GPT/Claude/Gemini
        │                         │
   NLP · STT/회의록 · Vision · 업무 자동화 API (PPT/체크리스트)
        │                         │
        └────────────┬────────────┘
                      ▼
             [AI Model Serving]
             H100/A100/V100 · vLLM · MIG · NVLink · InfiniBand
             NAS / Node Local Cache

[Utility Cluster]              [Data & Messaging]
Jenkins → Harbor               Kafka
Argo CD → 개발계·운영계 배포     PostgreSQL / MariaDB
Prometheus / Grafana           Elasticsearch
Elasticsearch

[외부 고객] → [AWS OpenAPI Cluster] (ALB/NLB) → 온프레미스 모델 서버 연동

[금융권 고객사] → 완전 폐쇄망 Kubernetes Cluster (자체 AI 모델, GPU Worker Node, 외부망 차단)
```

## 문제 및 해결

### 1. Kafka Consumer 리밸런싱으로 인한 회의록 처리 실패

상황 — 회의록 서비스는 STT와 NLP를 Kafka 메시지로 순차 처리하는 비동기 구조였다. 서로
다른 Topic을 쓰면서도 동일 Consumer Group을 사용해, 모델 추론이 길어지면 Polling이 지연되고
리밸런싱 후 오래된 요청이 만료된 토큰으로 실행됐다. 처리 성공률이 약 70%까지 떨어졌다.

원인 — 역할이 다른 Consumer가 동일 Group을 공유, 장시간 추론으로 `max.poll.interval.ms`
초과, 리밸런싱에 따른 메시지 재할당과 토큰 만료, Timeout·Retry 로직 부족이 겹쳤다.

해결 — Consumer Group을 서비스별로 분리하고 `max_poll_interval_ms`를 추론 시간에 맞춰
조정했다. Timeout·Retry 정책을 추가하고 만료 토큰 작업을 차단했으며, 실패 시 재처리하도록
로직을 보완했다.

결과 — 처리 성공률을 약 70%에서 99% 수준으로 개선했고, 이후 동일 오류가 재발하지 않았다.

### 2. NAS 기반 대용량 모델 로딩 병목

상황 — 대용량 모델을 원격 NAS에 저장해 Pod에서 PVC로 로딩했는데, 로딩에 2시간 이상 걸려
신규 배포를 근무 외 시간에만 진행해야 했다.

원인 — 단순 대역폭이 아니라 I/O 패턴이 병목이었다. 모델 로딩은 다수의 가중치·메타데이터
파일에 반복 접근하는데, NAS의 Random I/O·Metadata 처리에서 지연이 누적됐다.

해결 — 배포 시점에 모델을 매번 복사하는 대신, 배포 전 대상 GPU Node 로컬 디스크로 모델을
미리 복제하는 사전 다운로드 Job을 구성했다. Pod는 NAS 대신 Node Local Path를 읽도록 하고,
사용하지 않는 이전 모델을 정기적으로 정리했다.

결과 — 로딩 소요 시간 자체보다 로딩 시점을 배포 전으로 옮겨, 근무 외 시간에만 가능했던
대용량 모델 배포를 업무 시간에도 진행할 수 있게 됐다.

### 3. H100 MIG 기반 GPU 자원 효율화

상황 — 일부 모델은 H100 연산 성능은 필요했지만 GPU Memory 전체를 쓰지 않아, 모델마다
한 장을 독점 배치하면 유휴 HBM이 남았다.

해결 — H100 여러 장에 MIG를 적용해 모델별 HBM 요구량에 따라 GPU를 여러 Instance로
분할하고, Node와 MIG Profile에 Label을 적용해 `nodeSelector`로 배치했다.

결과 — GPU를 10여 개의 독립 자원 단위로 구성해 소규모 모델의 동시 배포 수를 늘렸고,
추가 증설 없이 기존 장비 활용도를 개선했다.

### 4. CRI-O Proxy 누락으로 인한 ImagePullBackOff

상황 — 개발계 Argo CD Pod가 외부 Registry에서 이미지를 가져오지 못하고
`ImagePullBackOff` 상태가 됐다. Node에서 직접 `curl`은 정상이었다.

원인 — 사용자 Shell에는 Proxy가 적용됐지만, 이미지 Pull을 수행하는 CRI-O systemd
서비스에는 Proxy 환경변수가 빠져 있었다.

해결 — CRI-O용 systemd Drop-in으로 Proxy 환경변수를 적용하고, 내부망 대역을
`NO_PROXY`에 추가한 뒤 재시작·재검증했다.

결과 — 이미지 Pull이 정상화됐고, 이후 Kyverno ClusterPolicy로 Proxy 환경변수를 자동
주입해 동일 문제를 구조적으로 방지했다.

### 5. GPT·SSE API의 504 오류 개선

상황 — 운영계에서 Kong을 통해 호출되는 GPT·Gemini·Claude SSE API에 504 오류가
반복됐다. 대부분 장시간 연결을 유지하는 SSE Endpoint에 집중돼 있었다.

분석 — Gateway 설정만이 아니라 요청이 통과하는 전체 계층을 나눠 확인했다 — Kong 로그,
Backend Thread·Connection 수, 요청마다 새로 생성되던 `httpx.AsyncClient`의 Connection Pool
비효율, SSE Buffer·Timeout 설정, 외부 AI API 응답시간까지 확인했다.

해결 — Connection Pool을 확장하고 `httpx.AsyncClient`를 Singleton으로 재사용했다. 연결
지연 시 요청을 종료하고 재시도하도록 했으며, Kong·Nginx의 SSE Buffer·Timeout을 조정하고
계층별 오류 로그를 분리했다.

결과 — 일간 504 오류를 100건 이상에서 1건 미만 수준으로 대폭 줄였다. 잔여 오류는 장기
요청과 외부 AI 응답 지연 중심으로 계속 모니터링하고 있다.

## 성과

- 상당 규모(수만 명)가 사용하는 기업용 AI 플랫폼의 개발계·운영계 인프라 운영
- 개발계·운영계·유틸리티 및 금융권 고객사 폐쇄망 Kubernetes 환경 관리
- H100·A100·V100 등 수십 장 규모의 GPU 자원 운영, 자체 LLM 포함 수십 개 모델 서비스 관리
- 정기적인 신규 모델 배포 — 요청부터 CI/CD·Ingress·API 서빙까지 약 1시간 이내
- MIG 적용으로 GPU를 10여 개의 독립 자원 단위로 구성
- 회의록 Kafka 처리 성공률 약 70% → 99%로 개선
- GPT·SSE 504 오류 일 100건 이상 → 1건 미만으로 감소
- 대용량 모델의 NAS 로딩 병목을 분석해 사전 다운로드 구조로 개선, 배포 가능 시간대 확대
- CRI-O Proxy 누락으로 인한 이미지 Pull 실패를 구조적으로 방지
- Kong·Kafka·DB·GPU·Kubernetes·Linux 계층을 연계한 장애 분석 체계 구축
- 금융권 완전 폐쇄망 환경에 기업용 AI 플랫폼 구축·운영 지원

## 회고

AI 플랫폼의 장애는 단일 애플리케이션이나 GPU 계층만 봐서는 원인을 찾기 어렵다. 하나의 요청은
Kong, Backend, Kafka, Database, Model Server, GPU, 외부 AI API까지 여러 계층을 거치기
때문에, 각 계층을 구분해 확인하는 습관이 문제 해결 속도를 좌우했다.

또한 Kafka Consumer Group, CRI-O Proxy, NAS 모델 로딩처럼 애플리케이션 외부의 설정이
서비스 성공률과 배포시간에 직접 영향을 준다는 점을 확인했다. 반복되는 장애는 개별 대응에
그치지 않고 공통 점검 항목과 운영 체크리스트로 정리했다.

향후에는 다음을 개선하고 싶다.

- GPT·SSE API의 Endpoint별 SLO 정의와 분산 추적 도입
- Kafka Consumer Lag·처리 성공률 대시보드 구축
- GPU·모델 요청량 추이를 시계열 예측에 활용한 사전 Capacity Planning과 이상 탐지(AIOps) 도입
- Vault를 운영계까지 확대하고 Secret Rotation 자동화
- 장애 로그·메트릭을 자동 분석해 초기 보고서를 생성하고 반복 복구 작업을 자동화
