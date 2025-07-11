## 🎯 프로젝트 목표

파티셔닝 적용 전후의 **조회 성능 차이를 비교 분석**하고,  
**파티셔닝이 실제로 성능 개선에 얼마나 효과적인지 실험적으로 확인**하는 것을 최종 목표로 합니다.

## 🦕Team
<table>
  <tr>
    <td align="center">
      <a href="https://github.com/Minkyoungg0">
        <img src="https://avatars.githubusercontent.com/Minkyoungg0" width="100px;" alt=""/>
        <br />
        <sub><b>문민경</b></sub>
      </a>
    </td>
    <td align="center">
      <a href="https://github.com/Gill010147">
        <img src="https://avatars.githubusercontent.com/Gill010147" width="100px;" alt=""/>
        <br />
        <sub><b>황병길</b></sub>
      </a>
    </td>
    <td align="center">
      <a href="https://github.com/dldydgns">
        <img src="https://avatars.githubusercontent.com/dldydgns" width="100px;" alt=""/>
        <br />
        <sub><b>이용훈</b></sub>
      </a>
    </td>
  </tr>
</table>

---
## 🦕활용 데이터
MovieReview Dataset (https://grouplens.org/datasets/movielens/)

- <Strong> 데이터셋 선정 이유 </Strong><br> --> 뭐하지?
여러 리뷰 페이지에서 수많은 상품에 대한 리뷰가 **빠르게 조회**되는 이유가 궁금해 성능 향상 기법을 찾던 중, DB 파티셔닝에 대해 알게 되었습니다.
이를 직접 실습해보고 이해하기 위해 리뷰 데이터가 풍부한 영화 데이터를 선정하게 되었습니다.
**각 영화에 대한 여러 유저의 평점을 평균화**하여, **장르별로 평점이 가장 높은 영화**를 손쉽게 확인할 수 있도록 구현함이 목표입니다.
<br>

<strong> 테이블 구조 설계 </strong> <br>
> <strong>movies</strong>
> | 컬럼명    | 타입         | 설명                   |
> |-----------|--------------|------------------------|
> | movieId   | INT (PK)     | 영화 고유번호          |
> | title     | TEXT         | 영화 제목              |
> | genres    | VARCHAR(500) | 장르(파이프 구분)      |

> <strong>ratings</strong>
> | 컬럼명    | 타입         | 설명                   |
> |-----------|--------------|------------------------|
> | userId    | INT          | 사용자 ID              |
> | movieId   | INT          | 영화 ID                |
> | rating    | FLOAT        | 평점(0.5~5.0)          |
> | timestamp | BIGINT       | 평점 등록 시각(UNIXTIME)|

---

## 🦕개발 환경
```
mysql  Ver 8.0.42-0ubuntu0.24.04.1 for Linux on x86_64 ((Ubuntu))
```
---
## 1️⃣ MySQL에 csv 파일 업로드하기
[추가] - DBeaver 활용하여 csv 자동 임포트 가능, CSV파일 import시 자동 테이블 생성 기능을 활용하려 하였지만 자동으로 인지한 속성이 부정확하여 에러 발생, 테이블 직접 생성 및 매핑으로 해결

<strong>📍Trouble Shooting #1 </strong><br>
<br>
<strong>🤔 문제 </strong><br>
<br>
데이터셋 improt 하면서 오류 발생 <br>
<br>
<img width="300" height="200" alt="image (2)" src="https://github.com/user-attachments/assets/ed45b931-5e53-4f6e-9f3e-23bf032abd03" />
<br>
<br>
💡 원인<br>
특정 컬럼에서 데이터 크기 타입이 작아서 문제 발생

✅ 해결<br>
CSV를 임포트하기 전, 테이블을 직접 생성하면서 원하는 컬럼만 포함하고, 각 컬럼의 크기 타입을 명확히 지정하기로 결정함
필요한 컬럼과 컬럼 타입을 지정할수있음
links 테이블은 외부 링크와 연결되는 테이블이므로 불필요해서 삭제

[추가] - 가공되지 않은 데이터가 너무 많아 불필요한 대기시간이 길었다, 영화 id값을 기준으로 1000이상은 모두 제거하였다. (약 2300만->620만개의 리뷰 데이터) 

<br>
## 2️⃣ mysqlslap 설치
Ubuntu에 성능 측정을 위한 sqlslap을 설치했다.
```
sudo apt update
sudo apt install mysql-client
```

## 3️⃣ 파티셔닝 경우의 수 실험

<strong>📍Trouble Shooting #2 </strong><br>
<br>
<strong>🤔 문제 </strong><br>
<br>
데이터가 너무 커서 
모든 테이블을 join 후 파티셔닝을 진행하려 했지만 데이터의 크기가 커서 시간이 너무 오래걸림<br>
<br>
<br>
💡 원인<br>


✅ 해결<br>
<br>
<br>

#### 파티셔닝 전  : 'movieId' 값을 기준으로 'rating'값의 평균을 조회할 때
  <img width="1544" height="164" alt="image (3)" src="https://github.com/user-attachments/assets/4447bbee-440e-45da-b9aa-77583bcde8b5" />


- **1. Hash Partitioning (movieId 기준)**  
  해시 함수를 이용해 `movieId` 값을 균등하게 분산시켜,  
  특정 영화 조회 시 빠른 데이터 접근이 가능하도록 설계함.
  
  <details>
  <summary><strong>파티셔닝 후</strong></summary>
    
  <img width="1581" height="194" alt="image (2)" src="https://github.com/user-attachments/assets/a7d16200-9f06-4de3-b10e-b0eb9fe3e003" />
  

  </details>

- **2. userId 기준 파티셔닝 (Range 또는 Hash)**  
  영화 조회 시 `userId`와의 연관성이 적어,  
  조회 성능 향상에는 큰 도움이 되지 않음.
  <details>
  <summary><strong>파티셔닝 후</strong></summary>
  <img width="1585" height="195" alt="image (1)" src="https://github.com/user-attachments/assets/6ff23e2f-ea3a-4b0b-bf76-a4dedd0b7b4c" />
  </details>

- **3. rating 값 기준 파티셔닝**  
  평점 값에 따라 데이터를 분할하였으나,  
  조회 시 인덱스 활용에 제한이 있어 오히려 성능 저하가 발생함.
  <details>
  <summary><strong>파티셔닝 후</strong></summary>
  <img width="1589" height="198" alt="image" src="https://github.com/user-attachments/assets/f454d41a-5d70-4347-bcbe-8388014dff94" />
  </details>

---
## ❓ 실험 환경 통제를 위한 질문 & 답변 (Troubleshooting 기록)

### Q1. 성능 비교 실험을 하기 전, 가장 먼저 해야 할 일은?

> **A. MySQL 서버의 설정(서버 변수)을 확인하고 통제하는 것이 첫 단계입니다.**  
> 실험 결과가 정확하고 반복 가능하려면, 쿼리 캐시나 자동 최적화 같은 기능이 개입하지 않도록 설정을 통제해야 합니다.


### Q2. 어떤 설정을 조정해야 하나요?

| 설정 항목 | 설명 | 실험 시 권장 설정 |
|-----------|------|------------------|
| `query_cache_type` | 쿼리 결과를 캐시해서 재사용 | `OFF` |
| `query_cache_size` | 쿼리 캐시 크기 | `0` |
| `innodb_stats_on_metadata` | 테이블 정보를 조회할 때마다 통계 재계산 | `OFF` |
| `performance_schema` | 성능 수집 기능, 오버헤드 발생 가능 | `OFF (선택)` |

### Q3. 실제 설정을 어떻게 확인하고 조정하나요?

#### ✅ 설정 확인
```sql
SHOW VARIABLES LIKE 'query_cache%';
SHOW VARIABLES LIKE 'innodb_stats_on_metadata';
SHOW VARIABLES LIKE 'performance_schema';
```
---

### 🔸 트러블슈팅

### ❌ mysqlslap 접속 에러

- **❗에러 메시지**
  ```bash
   mysqlslap: Error when connecting to server:
    Access denied for user 'root'@'localhost'</code>
  ```
- **원인**
- `root@localhost` 계정이 비밀번호 인증 방식이 아닌 `auth_socket` 방식으로 설정되어 있는 경우, 
  `mysqlslap`과 같은 외부 툴에서 로그인 불가

- **확인 방법**
```bash
sudo mysql
SELECT user, host, plugin FROM mysql.user WHERE user = 'root';
```

- **해결 방법** <br>
MySQL에 root 계정으로 접속 후 인증 방식을 비밀번호 방식으로 변경<br>
```bash
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '원하는비밀번호';
FLUSH PRIVILEGES;
```
---

### 📌 결론 및 회고

> 파티셔닝은 단순히 데이터를 나누는 것이 아니라,  
> **조회 패턴을 고려한 물리적 구조 최적화 전략**입니다.  
> 정규화와는 목적과 시점이 다르며,  
> **대용량 데이터 처리에 필수적인 튜닝 기법**으로써 성능 향상에 직접적인 영향을 줍니다.
