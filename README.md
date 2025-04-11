### Class Note Q&A Study with Kubernetes

궁금증이 많은 우리 AWS CLOUDSCHOOL 4기 동기들을 위한 맞춤 서비스! 

일자별로 수업 시간에 질문을 올릴 수 있는 Q&A 사이트입니다.

**프로젝트 기간:** 24.4.22 ~ 24.5.7(16일)

**인원:** 4명




## **[ VM Architecture ]**

<img width="844" alt="Image" src="https://github.com/user-attachments/assets/468d8ed2-bd03-45e6-a2a7-3741bab471f2" />

### [ Ingress Architecture ]

<img width="636" alt="Image" src="https://github.com/user-attachments/assets/b72c40aa-d4ae-454b-9e0b-aa3ff01917dd" />



## **구현 기능**

- Nginx와 ingress를 활용한 기본 구조 구성 및 https 인증서 설정
- MongoDB의 고가용성 확보를 위한 Replica Set 구성 및 StatefulSet 활용
- EFK Stack(Elasticsearch, Fluentbit, Kibana)을 통한 로그 수집 및 시각화
- metrics-server로 리소스 모니터링 후 HPA(Horizontal Pod Autoscaler)를 통해 자동 확장 설정
- Network Policy, 리소스 할당(ResourceQuota), 그리고 LimitRange를 통한 보안과 자원 관리 강화

## **기술 스택:**

<img width="1074" alt="Image" src="https://github.com/user-attachments/assets/89da90a9-fc71-49aa-8381-c21aaa50a0b5" />

- **Backend**: Django
- **Frontend**: NextJS + NextUI
- **Database**: MongoDB (Replica Set 구성)

## **📌 구현 기능 상세**

### 1. MongoDB DataBase 이중화

<img width="376" alt="Image" src="https://github.com/user-attachments/assets/f82475ab-0efb-4cb5-85d5-4c709bf96490" />

[ MongoDB 이중화 WorkBook 보기 ](MongoDBCluster/README.md)











### 2. MongoDB Data구조

```json
{
  "date": "2024-05-05",
  "questions": [
    {
      "question_id": "q1",
      "title": "Django와 MongoDB 연동 방법은?",
      "author_nickname": "developer123",
      "image": "base64",
      "created_at": "2024-05-05T10:00:00Z",
      "upvotes": 5,
      "answers": [
        {
          "answer_id": "a1",
          "author_nickname": "expert456",
          "created_at": "2024-05-05T11:00:00Z",
          "content": "mongoengine 쓰면 돼요",
          "upvotes": 10
        },
        {
          "answer_id": "a2",
          "author_nickname": "newbie789",
          "created_at": "2024-05-05T12:00:00Z",
          "content": "djongo로 ORM처럼 연동할 수 있어",
          "upvotes": 3
        }
      ]
    }
  ]
}
```
