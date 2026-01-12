# 🐙 말 많은 무너팀 (유레카 3기 대면 벡엔드) 

<div align="center">
  <img width="470" alt="말많은무너팀_로고" src="https://github.com/user-attachments/assets/6aac8890-fd93-42c5-b06b-5bb247160969" />

  <h3>"말은 많지만 결과는 확실한, 유레카 대면 종합 프로젝트"</h3>

  <p>프로젝트에 대한 한 줄 설명을 여기에 적어주세요.</p>

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
* **✅ 2. 이벤트 기반 알림 발송 시스템**: 요금 정산 완료 이벤트를 기반으로 Kafka를 활용해 이메일/SMS 발송
* **✅ 3. 모니터링 툴 시스템**: 대용량 배치 작업 현황 및 시스템 리소스 모니터링

## 🏗 아키텍처
> 아래는 예시 다이어그램입니다. 실제 구조에 맞게 수정하거나 이미지를 넣으세요.

```mermaid
graph LR
    A[Frontend] -- API --> B[Backend Server]
    B -- Query --> C[(MySQL DB)]
    B -- Cache --> D[(Redis)]
```

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
  <img src="https://img.shields.io/badge/mysql-%234479A1.svg?style=for-the-badge&logo=mysql&logoColor=white">
</p>
노션 링크에 저희 팀의 문서화가 자세히 되어있습니다.

---

## 🚀 시작 가이드

### 요구 사항 (Prerequisites)
* Java 17+, Node.js 18+
* MySQL 8.0

### 설치 및 실행
```bash
# 프로젝트 클론
$ git clone [https://github.com/your-repo/project.git](https://github.com/your-repo/project.git)

# 백엔드 실행
$ cd backend
$ ./gradlew bootRun

# 프론트엔드 실행
$ cd frontend
$ npm install && npm start
```
---

## 👥 팀원 소개

| 🐙 무너 1호 | 🐙 무너 2호 | 🐙 무너 3호 | 🐙 무너 4호 | 🐙 무너 5호 | 🐙 무너 6호 | 🐙 무너 7호 |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| <img src="https://github.com/identicons/user.png" width="100"/> | <img src="https://github.com/identicons/user.png" width="100"/> | <img src="https://github.com/identicons/user.png" width="100"/> | <img src="https://github.com/identicons/user.png" width="100"/> | <img src="https://github.com/identicons/user.png" width="100"/> | <img src="https://github.com/identicons/user.png" width="100"/> | <img src="https://github.com/identicons/user.png" width="100"/> |
| **최훈석 (팀장)** | **윤재민** | **박유빈** | **이경윤** | **임지우** | **유효주** | **최하영** |
| [@github_id](https://github.com/) | [@github_id](https://github.com/) | [@github_id](https://github.com/) | [@github_id](https://github.com/) | [@github_id](https://github.com/) | [@github_id](https://github.com/) | [@github_id](https://github.com/) |
| Backend / DB | Frontend / UI | Backend / Infra | Frontend / Design | Backend / API | Frontend / UX | Data / QA |


---

### 🐙 무너팀의 한마디!
* **김이름**: "말은 많지만 코드는 간결하게! 끝까지 완주합시다."
* **박이름**: "즐겁게 소통하며 최고의 시너지를 내보아요!"
* **최이름**: "기술적인 도전이 기대되는 프로젝트입니다."
* **이이름**: "사용자 경험을 최우선으로 생각하겠습니다."
* **이이름**: "사용자 경험을 최우선으로 생각하겠습니다."
* **이이름**: "사용자 경험을 최우선으로 생각하겠습니다."
* **이이름**: "사용자 경험을 최우선으로 생각하겠습니다."
