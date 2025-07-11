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

- <Strong> 데이터셋 선정 이유 </Strong><br>
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
## :one: MySQL에 csv 파일 업로드하기
<!-- [추가] - DBeaver 활용하여 csv 자동 임포트 가능, CSV파일 import시 자동 테이블 생성 기능을 활용하려 하였지만 자동으로 인지한 속성이 부정확하여 에러 발생, 테이블 직접 생성 및 매핑으로 해결 -->
<strong>📍Trouble Shooting #1 </strong><br>
<br>
<strong> 🤔 문제 </strong><br>
<br>
데이터셋 improt 하면서 오류 발생 <br>
<br>
<img width="300" height="200" alt="image (2)" src="https://github.com/user-attachments/assets/ed45b931-5e53-4f6e-9f3e-23bf032abd03" />
<br>
<br>
💡 원인<br>
Vachar(50)인 genres와 timestamp의 데이터가 테이블의 제약조건보다 길이가 길어 오류 발생
<img width="500" height="400" alt="image" src="https://github.com/user-attachments/assets/561553d1-0fac-4b69-a42e-eede397d0d4e" /><br>

✅해결<br>
CSV를 임포트 하면서 데이터 크기 제약조건을 50 -> 500으로 변경하여 데이터 불러오는 데 성공 <!-- 왜 500으로? -->

<br>

📍 Trouble Shooting #2

🤔문제  
파티셔닝 테이블로 데이터를 로딩할 때 **30분 이상 지연되는 현상 발생**

💡 원인  
초기에는 **가공되지 않은 전체 리뷰 데이터(약 2,300만 건)** 를 그대로 불러오고 있었다.  
파티셔닝 작업은 테이블 구조 변경뿐 아니라 **데이터 분할 작업이 동반되기 때문에**,  
데이터 양이 많을수록 **파티션 생성 및 데이터 삽입 시 오버헤드가 매우 크다.**

✅ 해결  
실험 효율성과 파티셔닝 성능 측정을 위해 **데이터량을 1,000만 건 이하**로 제한하고자 했었다.
따라서 movieId가 1000 이상인 영화의 리뷰는 제외하여
영화 종류를 줄이고, 데이터 양도 **약 2,300만 건 → 620만 건**으로 축소했다.
이로 인해 데이터 로딩 시간이 **30분 이상 → 약 5분**으로 크게 단축되었고,
보다 안정적인 환경에서 실험을 진행할 수 있었다.

<br>

## 2️⃣ mysqlslap 설치

Ubuntu에 성능 측정을 위한 sqlslap을 설치했다.

```
sudo apt update
sudo apt install mysql-client
```

💡 mysqlslab 설치 이유<br>
단순 조회 1회의 속도보다, **가상의 사용자가 동일한 쿼리를 반복 실행하는 상황에서의 평균 성능을 측정**하고자 하였다.
mysqlslap은 동시 접속과 반복 실행을 통한 부하 테스트가 가능하여,
파티셔닝 전후 또는 방식별 쿼리 성능을 객관적으로 비교하기 위한 벤치마크 도구로 적합하다고 판단했다.

<br>

<strong>📍Trouble Shooting #3 </strong><br>

🤔 문제<br>
mysqlslap 접속 에러

  ```bash
   mysqlslap: Error when connecting to server:
    Access denied for user 'root'@'localhost'</code>
  ```

💡 원인<br>
- `root@localhost` 계정이 비밀번호 인증 방식이 아닌 `auth_socket` 방식으로 설정되어 있는 경우, 
  `mysqlslap`과 같은 외부 툴에서 로그인 불가

✅ 해결  
MySQL에 root 계정으로 접속 후 인증 방식을 비밀번호 방식으로 변경<br>
```bash
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '원하는비밀번호';
FLUSH PRIVILEGES;
```

<br>

## 3️⃣ 파티셔닝 옵션별 테스트
테스트를 진행하기 전
성능 테스트의 신뢰성을 높이기 위해, **AI를 활용해 MySQL 환경 설정을 분석·적용**하였습니다.
특히 쿼리 캐시나 메타데이터 통계처럼 **결과에 영향을 줄 수 있는 요소들을 통제**해
실험 간 **일관성을 유지**하고자 하였습니다.

- <strong> AI활용 변인통제 </strong><br>
Q. MySQL에서 파티셔닝 전후 성능 비교를 정확히 하려면 어떤 캐시나 환경 설정을 꺼야해?

| 설정 항목 | 설명 | 실험 시 권장 설정 | 비고|
|-----------|------|------------------|----------------|
| `query_cache_type` | 쿼리 결과를 캐시해서 재사용 | `OFF` | Mysql 0.8.x버젼부터 사용 안함 |
| `query_cache_size` | 쿼리 캐시 크기 | `0` | Mysql 0.8.x버젼부터 사용 안함 |
| `innodb_stats_on_metadata` | 테이블 정보를 조회할 때마다 통계 재계산 | `OFF` | Mysql 0.8.x버젼부터 사용 안함 |
| `FLUSH TABLES` | 테이블을 닫고 캐시된 데이터를 디스크에 저장 | `` | |

-> 따라서 'FLUSH TABLES' 설정만 적용했다.

#### 📌 'movieId' 값을 기준으로 'rating'값의 평균을 조회할 때

예상결과: movieId 기준으로 Partitioning을 했을 때만 성능향상, rating과 userId로 Partitioning을 했을 때는 오히려 성능 감소


- **파티셔닝 전**
  <img width="1544" height="164" alt="image (3)" src="https://github.com/user-attachments/assets/4447bbee-440e-45da-b9aa-77583bcde8b5" />

- **파티셔닝 후**
  
  **1. Hash Partitioning (movieId 기준)**  
    ```
    CREATE TABLE ratings_partitioned_by_movieId (
    userId INT,
    movieId INT,
    rating FLOAT,
    timestamp BIGINT,
    PRIMARY KEY (userId, movieId)
    )
    PARTITION BY HASH(movieId)
    PARTITIONS 24;
  ```
      
    <img width="1581" height="194" alt="image (2)" src="https://github.com/user-attachments/assets/a7d16200-9f06-4de3-b10e-b0eb9fe3e003" />
    

  **2. userId 기준 파티셔닝 (Range 또는 Hash)**  
    ```
    CREATE TABLE ratings_partitioned_by_userId (
    userId INT,
    movieId INT,
    rating FLOAT,
    timestamp BIGINT,
    PRIMARY KEY (userId, movieId)
    )
    PARTITION BY HASH(userId)
    PARTITIONS 5;
    
    ```
    <img width="1585" height="195" alt="image (1)" src="https://github.com/user-attachments/assets/6ff23e2f-ea3a-4b0b-bf76-a4dedd0b7b4c" />
  
  **3. rating 값 기준 파티셔닝**  
    ```
    CREATE TABLE ratings_partitioned_by_rating (
    userId INT,
    movieId INT,
    rating INT,  -- rating을 정수형으로 변환 (예: 0~5 범위로 제한)
    timestamp BIGINT,
    PRIMARY KEY (userId, movieId, rating)
    )
    PARTITION BY RANGE (rating) (
    PARTITION p0_1 VALUES LESS THAN (1),
    PARTITION p1_2 VALUES LESS THAN (2),
    PARTITION p2_3 VALUES LESS THAN (3),
    PARTITION p3_4 VALUES LESS THAN (4),
    PARTITION p4_5 VALUES LESS THAN (6)  -- 5 포함을 위해 6으로 지정
    );
    
    ```
    <img width="1589" height="198" alt="image" src="https://github.com/user-attachments/assets/f454d41a-5d70-4347-bcbe-8388014dff94" />

- 테스트 결과 요약

| 테스트 케이스            | 평균 실행 시간(초) | 최소(초) | 최대(초) | 비고                      |
|-------------------------|-------------------|----------|----------|---------------------------|
| 파티셔닝 전             | 28.466            | 27.241   | 29.194   | 기준값                    |
| movieId 해시 파티셔닝   | 0.599             | 0.571    | 0.651    | **가장 빠름**             |
| userId 해시 파티셔닝    | 12.138            | 11.407   | 12.324   | 프루닝 불가, 개선 제한적  |
| rating RANGE 파티셔닝   | 11.531            | 11.466   | 11.685   | 데이터 불균형, 개선 제한적|

-> movieId 파티셔닝이 가장 빠르긴 하지만, 예상했던 userId와 rating 파티셔닝의 성능 저하가 나타나지 않았다.
원인이 뭘까?

---

### 📌 결론 및 회고

> 파티셔닝은 단순히 데이터를 나누는 것이 아니라,  
> **조회 패턴을 고려한 물리적 구조 최적화 전략**입니다.  
> 정규화와는 목적과 시점이 다르며,  
> **대용량 데이터 처리에 필수적인 튜닝 기법**으로써 성능 향상에 직접적인 영향을 줍니다.
