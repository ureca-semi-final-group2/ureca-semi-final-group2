# 🐙 말 많은 무너팀 (유레카 3기 대면 백엔드) 

<div align="center">
  <img src="https://github.com/user-attachments/assets/9a54a772-af78-4451-bc60-0528c480b4d6" width="470" alt="말많은무너팀 로고"
  />


  <h3>"말은 많지만 결과는 확실한, 유레카 대면 종합 프로젝트"</h3>

  <p>대용량 통신 요금 명세서 및 알림 발송 시스템</p>

  [![Build Status](https://img.shields.io/badge/build-passing-brightgreen)](#)
  [![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
</div>

<br/>

## 📌 목차
1. [프로젝트 소개](#-프로젝트-소개)
2. [주요 기능](#-주요-기능)
3. [아키텍처](#-아키텍처)
4. [기술 스택](#-기술-스택)
5. [시작 가이드](#-시작-가이드)
6. [팀원 소개](#-팀원-소개)

---

## 📝 프로젝트 소개
* **개발 기간**: 2026.01.06 ~ 2026.01.27
* **서비스 명**: [대용량 통신 요금 명세서 및 알림 발송 시스템]
* **핵심 목표**: 통신사의 핵심 업무인 대규모 정산과 알림 발송을 안정적으로 처리하기 위한 시스템입니다. 매월 수백만 건에 달하는 요금 청구 데이터를 배치 기반으로 정확하게 정산하고, 정산 결과를 기반으로 요금 청구서를 고객에게 이메일과 SMS를 통해 발송합니다.
  
* **배포 주소**: [🚀 서비스 바로가기 링크]()

## ✨ 주요 기능
* **✅ 1. 요금 정산 배치 시스템**: 정해진 날짜에 배치 기반 요금 정산 수행

* **✅ 2. 이벤트 기반 알림 발송 시스템**: 요금 정산 완료시 이벤트를 기반으로 Kafka를 활용해 이메일/SMS 발송
  - **비동기 발송**: Kafka Consumer를 통해 이메일 및 SMS를 비동기 발송(Mocking) 처리한다.
  - **금지 시간대 제어**: Redis에 설정된 금지 시간대에는 발송을 일시 정지하고 대기 큐로 돌린다.
  - **재시도 전략**: 이메일 발송 실패(1% 확률)시 SMS로 청구 내역을 재발송한다.

* **✅ 3. 모니터링 툴 시스템**: 대용량 배치 작업 현황 및 시스템 리소스 모니터링
  - **메트릭 수집**: Prometheus를 통해 시스템 자원 및 배치 수행 지표를 수집한다.
  - **로그 추적**: Loki와 Grafana를 연동하여 특정 유저의 발송 실패 원인을 로그로 추적한다.

* **✅ 4. 프론트 관리자 관제**
  - **실시간 대시보드**: 정산 진행률, Kafka Lag, 발송 성공/실패율을 시각화하여 보여준다.
  - **발송 정책 설정**: 금지 시간대 설정 및 즉시 발송 중단(Kill-Switch) 기능을 제공한다.
  - **콘텐츠 관리**: 가변 변수가 포함된 이메일/SMS 템플릿을 생성 및 수정한다.

## 🏗 아키텍처
<img width="8192" height="2113" alt="KakaoTalk_20260114_150641603" src="https://github.com/user-attachments/assets/08d38b80-9360-4791-8501-3c232768de8b" />


## 🛠 기술 스택

### Environment
<p>
  <img src="https://img.shields.io/badge/git-%23F05033.svg?style=for-the-badge&logo=git&logoColor=white">
  <img src="https://img.shields.io/badge/github-%23121011.svg?style=for-the-badge&logo=github&logoColor=white">
  <a href= "https://shadow-lychee-a03.notion.site/2e0ab48eeb4b8001bb42f7c35e987cd8?source=copy_link">
    <img src="https://img.shields.io/badge/Notion-%23000000.svg?style=for-the-badge&logo=notion&logoColor=white">
  </a>
  <a href="https://jack36140.atlassian.net/jira/software/projects/MOONU/summary" target="_blank">
  <img src="https://img.shields.io/badge/jira-0052CC?style=for-the-badge&logo=jira&logoColor=white">
</a>
</p>

### Development
<p>
  <img src="https://img.shields.io/badge/java-%23ED8B00.svg?style=for-the-badge&logo=openjdk&logoColor=white">
  <img src="https://img.shields.io/badge/spring%20boot-%236DB33F.svg?style=for-the-badge&logo=springboot&logoColor=white">
  <img src="https://img.shields.io/badge/react-%2320232a.svg?style=for-the-badge&logo=react&logoColor=%2361DAFB">
  <img src="https://img.shields.io/badge/postgresql-%234479A1.svg?style=for-the-badge&logo=postgresql&logoColor=white">
</p>
노션 링크에 저희 팀의 문서화가 자세히 되어있습니다.

---

## 🚀 시작 가이드

### 요구 사항 (Prerequisites)
* Java 17+
* Spring Boot 3.4.1
* PostgreSQL 18
* Kafka 3.5 (Confluent 7.5.0)

---

## 👥 팀원 소개

| 🐙 무너 1호 | 🐙 무너 2호 | 🐙 무너 3호 | 🐙 무너 4호 | 🐙 무너 5호 | 🐙 무너 6호 |
| :---: | :---: | :---: | :---: | :---: | :---: |
| <img src="https://github.com/hoonseok0710.png" width="100"/> | <img src="https://github.com/ashmin-yoon.png" width="100"/> | <img src="https://github.com/yubin012.png" width="100"/> | <img src="https://github.com/LeeGyeongYoon-BE.png" width="100"/> | <img src="https://github.com/dnwldla.png" width="100"/> | <img src="https://github.com/rettooo.png" width="100"/> |
| **최훈석 (팀장)** | **윤재민** | **박유빈** | **이경윤** | **임지우** | **최하영** |
| [@hoonseok0710](https://github.com/hoonseok0710) | [@ashmin-yoon](https://github.com/ashmin-yoon) | [@yubin012](https://github.com/yubin012) | [@LeeGyeongYoon-BE](https://github.com/LeeGyeongYoon-BE) | [@dnwldla](https://github.com/dnwldla) | [@rettooo](https://github.com/rettooo) |
| Backend / Infra | Backend / Infra | Backend / Kafka | Backend / DB | Backend / DB | Backend / Kafka |


---

### 🐙 역할
* **최훈석**: "모니터링 툴 작업, 이벤트 알림"
* **윤재민**: "배치 작업, 약정 및 생일 이벤트 할인"
* **박유빈**: "발송 배치 작업 후 Kafka 처리"
* **이경윤**: "부가 서비스 등급제 할인, 장기 고객 할인"
* **임지우**: "정산작업처리 "
* **최하영**: "대용량 데이터 카프카 처리"

