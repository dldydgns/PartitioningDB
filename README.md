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
여러 리뷰 페이지에서 수많은 상품 리뷰가 **빠르게 조회**되는 이유가 궁금해 성능 향상 기법을 조사하던 중,
리뷰 데이터가 풍부한 영화 데이터를 활용해 실습해보기로 결정했습니다.
**각 영화에 대한 여러 유저의 평점 평균**을 조회해보면서 실제 서비스 환경에서 성능 개선이 어떻게 이루어지는지를 직접 체감해보고자 했습니다.
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
DBeaver
mysqlslap
```
---
## :one: MySQL에 csv 파일 업로드하기
<!-- [추가] - DBeaver 활용하여 csv 자동 임포트 가능, CSV파일 import시 자동 테이블 생성 기능을 활용하려 하였지만 자동으로 인지한 속성이 부정확하여 에러 발생, 테이블 직접 생성 및 매핑으로 해결 -->
<strong>📍Trouble Shooting #1 </strong><br>

🤔문제  
DBeaver에서 데이터셋을 자동 improt 하며 오류 발생
<br>
<img width="300" height="200" alt="image (2)" src="https://github.com/user-attachments/assets/ed45b931-5e53-4f6e-9f3e-23bf032abd03" />
<br>
💡 원인<br>
Vachar(50)인 genres와 timestamp의 데이터가 테이블의 제약조건보다 길이가 길어 오류 발생<br>
<img width="500" height="400" alt="image" src="https://github.com/user-attachments/assets/561553d1-0fac-4b69-a42e-eede397d0d4e" />

✅해결<br>
CSV를 임포트 하면서 데이터 크기 제약조건을 50 -> 500으로 변경하여 데이터 불러오는 데 성공 <!-- 왜 500으로? -->

<br>

<strong>📍 Trouble Shooting #2 </strong><br>

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

Ubuntu에 성능 측정을 위한 mysqlslap을 설치했다.

```
sudo apt update
sudo apt install mysql-client
```

💡 mysqlslap 설치 이유<br>
**가상의 사용자가 동일한 쿼리를 반복 실행하는 상황에서의 평균 성능을 측정**하고자 하였다.
mysqlslap은 동시 접속과 반복 실행을 통한 부하 테스트가 가능하여,
파티셔닝 전후 성능을 객관적으로 비교하기 위한 벤치마크 도구로 적합하다고 판단했다.

<br>

<strong>📍Trouble Shooting #3 </strong><br>

🤔 문제<br>
mysqlslap 접속 에러

  ```bash
   mysqlslap: Error when connecting to server:
    Access denied for user 'root'@'localhost'</code>
  ```

💡 원인<br>
`root@localhost` 계정이 비밀번호 인증 방식이 아닌 `auth_socket` 방식으로 설정되어 있는 경우, `mysqlslap`과 같은 외부 툴에서 로그인 불가

✅ 해결  
MySQL에 root 계정으로 접속 후 인증 방식을 비밀번호 방식으로 변경<br>
```bash
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '원하는비밀번호';
FLUSH PRIVILEGES;
```

<br>

## 3️⃣ 실험 환경 통제
테스트를 진행하기 전
성능 테스트의 신뢰성을 높이기 위해 파티셔닝 외에 실행속도에 영향을 줄 요인을 제거하면서<br>
실험 간 **일관성을 유지**하고자 하였다.

AI를 활용하여 빠르게 설정<br>
Q. MySQL에서 파티셔닝 전후 성능 비교를 정확히 하려면 어떤 캐시나 환경 설정을 꺼야해?

A.<br>
| 설정 항목 | 설명 | 실험 시 권장 설정 | 비고|
|-----------|------|------------------|----------------|
| `query_cache_type` | 쿼리 결과를 캐시해서 재사용 | `OFF` | Mysql 0.8.x버젼부터 사용 안함 |
| `query_cache_size` | 쿼리 캐시 크기 | `0` | Mysql 0.8.x버젼부터 사용 안함 |
| `innodb_stats_on_metadata` | 테이블 정보를 조회할 때마다 통계 재계산 | `OFF` | Mysql 0.8.x버젼부터 사용 안함 |
| `FLUSH TABLES` | 테이블을 닫고 캐시된 데이터를 디스크에 저장 | `` | |

따라서 mysql 8.0.42 버젼을 사용중이므로 쿼리 실행전 'FLUSH TABLES'만 진행

<br>

## 4️⃣ 테이블 생성 및 테스트

### 📖 테이블 생성

> **각 영화에 대한 여러 유저의 평점 평균**을 빠르게 조회하기 위해서는 **리뷰들에 대한 조회**가 빨라야 하기 때문에 **ratings 테이블**을 파티셔닝 테이블로 선정하였다.

- **파티셔닝 전**<br>
  ```sql
  CREATE TABLE ratings (
   userId INT,
   movieId INT,
   rating FLOAT,
   timestamp BIGINT
  );
  ```

- **파티셔닝 후**<br>
> 어떤 컬럼을 기준으로 파티셔닝 하느냐에 따라서 성능이 어떻게 달라지는지 확인하기 위해, movieId, userId, rating 각각을 기준으로 파티셔닝을 적용하였다.<br>

  **1. movieId 기준 파티셔닝(Hash Partitioning)**  
  ```sql
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
    

  **2. userId 기준 파티셔닝 (Hash Partitioning)**  
  ```sql
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
  
  **3. rating 값 기준 파티셔닝 (Range Partitioning)**  
  ```sql
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

### 🧪 1차 테스트

#### 🏃 테스트 진행 
**각 영화에 대한 여러 유저의 평점 평균**을 구하려면 movieId로 그룹화해서 조회하게 된다.<br>
따라서 파티셔닝도 movieId를 기준으로 하면, 쿼리가 필요한 데이터에 더 빠르게 접근할 수 있어 성능 향상에 도움이 될 것으로 예상했다.

사전 기대 결과
> 파티션키가 select문에서 where조건에 사용될 때만 성능 개선이 나타날 것이라고 예상했다.
> 반대로, 파티션키와 조회조건이 다를 경우 불필요한 파티션 접근과 추가 연산 발생하여 오히려 성능이 저하될 수 있다고 생각했다.
<br>
 1. 파티셔닝❌
  <img width="1587" height="170" alt="Image" src="https://github.com/user-attachments/assets/96f81bf8-c218-403e-b17b-7893738cbd42" />
  2. movieId(Hash)
  <img width="1590" height="297" alt="Image" src="https://github.com/user-attachments/assets/ccff7b75-c1c4-4894-838c-0cb5369312f8" />
  3. userId(Hash)
  <img width="1587" height="263" alt="Image" src="https://github.com/user-attachments/assets/3e3a4838-a4d8-42ca-8918-bdaa999725eb" />
  4. rating(Range)
  <img width="1587" height="217" alt="Image" src="https://github.com/user-attachments/assets/aed91d77-e045-4797-b256-cb0739312286" />

- 테스트 결과 요약
  | 테스트 케이스            | 평균 실행 시간(초) | 최소(초) | 최대(초) | 비고                   |
  |-------------------------|-------------------|----------|----------|--------------------------|
  | 파티셔닝 전             | 28.466            | 27.241   | 29.194   | 기준값                    |
  | movieId 해시 파티셔닝   | 0.599             | 0.571    | 0.651    | **가장 빠름**             |
  | userId 해시 파티셔닝    | 12.138            | 11.407   | 12.324   |                           |
  | rating RANGE 파티셔닝   | 11.531            | 11.466   | 11.685   |                           |

#### 📌 1차 결론

movieId를 파티셔닝 키로 지정한 테이블이 조회성능이 가장 빨랐지만, 다른 속성들을 기준으로 파티셔닝 한 테이블들 또한 성능향상이 일어났다. 이때까지는 파티셔닝이 디스크를 분할하기 때문에 IO속도가 빨라 질 것이라는 추측을 하였다.

### 🧪 2차 테스트

#### 🏃 테스트 진행 

사용자 입장에서는 특정 영화의 정보를 조회를 할 때 영화의 이름을 기준으로 조회한다.
따라서 movie테이블과 ratings테이블을 조인하여 사용하게 되는데, 이때도 파티셔닝이 조회성능 향상에 도움을 줄 것이라 예상했다.

사전 기대 결과
> 파티셔닝은 조인 성능에도 도움을 줄 것이다.
> 파티셔닝키와 조인키, 파티셔닝키와 조회조건 모두가 일치 한 경우에만 조회 성능 향상이 일어 날 것이다.

  ```sql
  -- 일반 테이블
  SELECT SQL_NO_CACHE m.title AS 제목, ROUND(AVG(r.rating), 2) AS 평점
  FROM movies m
  JOIN ratings r ON r.movieId = m.movieId
  WHERE m.title Like('Toy Story')
  GROUP BY m.title;
  
  -- 파티셔닝 테이블 (movieId 기준)
  SELECT m.title AS 제목, ROUND(AVG(r.rating), 2) AS 평점
  FROM movies m
  JOIN ratings_partitioned_by_movieId r ON r.movieId = m.movieId
  WHERE m.title Like('Toy Story')
  GROUP BY m.title;
  
  -- 파티셔닝 테이블 (rating 기준)
  SELECT m.title AS 제목, ROUND(AVG(r.rating), 2) AS 평점
  FROM movies m
  JOIN ratings_partitioned_by_rating r ON r.movieId = m.movieId
  WHERE m.title Like('Toy Story')
  GROUP BY m.title;

  -- 파티셔닝 테이블 (userId 기준)
  SELECT m.title AS 제목, ROUND(AVG(r.rating), 2) AS 평점
  FROM movies m
  JOIN ratings_partitioned_by_userId r ON r.movieId = m.movieId
  WHERE m.title Like('Toy Story')
  GROUP BY m.title;
  ```

- 테스트 결과 요약
  | 테스트 케이스         | 평균 실행 시간(초) | 최소(초)  | 최대(초)  | 비고  |
  | --------------- | ----------- | ------ | ------ | --- |
  | 파티셔닝 없음         | 34.079      | 33.829 | 34.403 | 기준값 |
  | movieId 해시 파티셔닝 | 15.972      | 15.841 | 16.045 |     |
  | rating 해시 파티셔닝  | 15.925      | 15.810 | 16.048 |     |
  | userId Range 파티셔닝  | 15.935      | 15.832 | 16.049 |     |

#### 📌 2차 결론

예상과 다르게 성능 향상을 기대했던 movieId를 파티션 키로 지정한 경우보다 <br>
오히려 다른 컬럼 기준으로 파티셔닝한 테이블들에서 비슷하거나 더 나은 성능을 보였다.
따라서 movieId 파티셔닝 테이블에서 성능 향상이 발생하지 않은 이유를 찾기위해 AI를 활용하여 답을 찾아보았다

### ⚠️ 예상외의 결과

#### ❔ 문제상황 인지

AI를 활용하여 빠르게 문제를 인지하였다.

<strong>Q1. mysql에서 파티셔닝을 한 테이블과 일반 테이블과의 조인에서 고려해야 할 사항이 있을까?</strong>

A.<br>
✅ 1. 파티션 프루닝(Partition Pruning)이 불가능할 수 있음
- 조인에서는 파티션 키가 WHERE절 또는 ON절에 명시적으로 사용되지 않으면 MySQL이 어떤 파티션을 사용할지 판단 못해 모든 파티션을 스캔할 수 있습니다.
- 예: 파티션 키가 userId인데, 조인 조건 또는 WHERE절에 userId가 없으면 프루닝 안 됨.

✅ 2. 인덱스 사용이 제한될 수 있음
- 파티셔닝된 테이블은 파티션 키에 포함되지 않은 컬럼에 대한 인덱스가 효과적이지 않을 수 있습니다.
- 또한, 일부 조인 전략에서는 인덱스를 무시하고 풀 테이블 스캔이 발생할 수 있습니다.
  
✅ 3. 파티션 키와 조인 키가 다르면 성능저하 가능성 있음
- 일반 테이블과 조인 시, 조인 키 ≠ 파티션 키인 경우, 불필요한 파티션 접근이 늘어나고 해시 리디스트리뷰션이 일어날 수 있어 성능이 떨어질 수 있습니다.

**Q2. 파티션 푸르닝이 안되는 이유에는 뭐가있을까?**

A. LIKE, BETWEEN, IN 등의 조건 사용 (일부 경우)
- LIKE는 패턴 매칭이라 정적인 값이 아님
- IN, BETWEEN은 복잡한 조건이거나 값이 많을 경우 판단 불가

---

#### 👏 1차로 찾은 원인

1. **`LIKE('Toy Story')`** 구문을 사용하여 **정확한 상수 비교가 아니었기 때문에** MySQL 옵티마이저가 **파티션 프루닝을 적용하지 못했고**, 성능 저하가 발생했음 이를 `=` 조건으로 바꾸면 프루닝이 적용되어 성능 향상이 가능할 것으로 예상

2. 그러나 EXPLAIN 키워드를 붙여 예상되는 파티션 이용 항목을 확인한 결과, 여전히 파티션 프루닝이 적용되지 않았다.
```sql
-- 파티션 확인 코드
EXPLAIN
SELECT m.title AS 제목, ROUND(AVG(r.rating), 2) AS 평점
FROM movies m
JOIN ratings_partitioned_by_movieId r ON r.movieId = m.movieId
WHERE m.title = 'Toy Story (1995)'
GROUP BY m.title;
```
<img width="976" height="350" alt="Image" src="https://github.com/user-attachments/assets/47c71149-7c7c-4afe-8c5f-4a7d2baf25ad" />

#### 👏 2차로 찾은 원인

1. ratings_partitioned_by_movieId 테이블은 movieId 기준으로 파티셔닝 되어있다.<br>
조인 조건도 r.movieId = m.movieId 처럼 movieId를 사용하고 있지만, WHERE 조건은 movieId가 직접 사용되지 않고,<br>
대신 m.title = 'Toy Story (1995)'처럼 **영화 제목(title)** 만 사용되고 있다.
```sql
-- 문제 있었던 쿼리
WHERE m.title LIKE('Toy Story');   -- 프루닝 불가 ❌
  
-- 성능 개선을 위한 수정
WHERE m.movieId = 1;         -- 프루닝 가능 ✅
```
2. 따라서 movieId를 직접 지정한 경우에 파티션 푸르닝이 발생한다.
```sql
--- 파티션 코드 확인
EXPLAIN
SELECT m.title AS 제목, ROUND(AVG(r.rating), 2) AS 평점
FROM movies m
JOIN ratings_partitioned_by_movieId r ON r.movieId = m.movieId
WHERE m.movieId = 1;
GROUP BY m.title;
```
<img width="917" height="395" alt="Image" src="https://github.com/user-attachments/assets/13674daf-ee0a-4697-acc4-edadd068a1c0" />

#### 👏 3차로 찾은 원인

잠시 테이블 생성 쿼리를 가져오겠다.
```sql
CREATE TABLE ratings (
 userId INT,
 movieId INT,
 rating FLOAT,
 timestamp BIGINT
);

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

1. 위의 테이블을 비교하면 파티션 뿐만 아니라 기본키를 지정하였다. 기본키를 지정시 다음과 같은 작업을 한다
  - MySQL / MariaDB: 기본키 지정 시 클러스터형 인덱스(Primary Key Index)를 자동 생성
  - Oracle / PostgreSQL: UNIQUE 인덱스를 자동 생성
  - SQL Server: 클러스터형 또는 넌클러스터형 인덱스를 자동 생성 (설정에 따라 다름)

2. 따라서 MySQL을 사용했기 때문에 **기본키 지정 시 자동으로 클러스터형 인덱스가 생성**되었고,<br>
   이 차이로인해, 1차 테스트에서 잘못된 파티셔닝 키를 선택을 했을 때도 **인덱스 유무가 성능에 더 큰 영향**을 주어 성능 향상이 발생했던 것이다.

### 🧪 3차 테스트

#### 🏃 테스트 진행 

올바른 파티셔닝 키 선택에 대한 뚜렷한 성능 차이를 다시 한번 검증하기 위해 ratings 테이블에 기본키를 추가하였고,<br>
이 파티셔닝 테이블을 활용한 조인의 성능 향상도 다시 확인해보기 위해 조인 조건인 movieId 컬럼에 단일 인덱스도 함께 지정하여 성능을 측정해 보았다.

사전 기대 결과
> 기본 테이블 조회성능 향상(원인: 인덱스)
> 처음에 예상했던 결과처럼 파티션키와 조회조건이 동일한 경우에 단일 검색 및 조인 조회 성능 향상 발생

- 질의문
  + 개선 전
    <img width="1583" height="170" alt="Image" src="https://github.com/user-attachments/assets/416af0a2-2742-4c90-8b34-09f98f2b81b6" />
  + 개선 후
    <img width="1587" height="170" alt="Image" src="https://github.com/user-attachments/assets/a0d0bd90-0046-4a4c-8dc1-9dd863596c98" />

- 질의문
  + 개선 전
  <img width="1272" height="428" alt="Image" src="https://github.com/user-attachments/assets/67a3274f-da6a-4e34-920c-b0befc9c997d" />
    
  + 개선 후
    <img width="1591" height="708" alt="Image" src="https://github.com/user-attachments/assets/64dceafe-0c07-46a0-a63b-0255c09cefb2" />

- 테스트 결과 요약
  | 테스트 케이스         | 평균 실행 시간(초) | 최소(초)  | 최대(초)  | 비고  |
  | --------------- | ----------- | ------ | ------ | --- |
  | 파티셔닝 없음         | 72.793      | 69.514 | 78.860 | 개선이후 |
  | movieId 해시 파티셔닝 | 29.582      | 27.524 | 38.980 | 개선이후 |
  | rating 해시 파티셔닝  | 28.837      | 28.741 | 28.953 | 개선이후 |
  
  | 테스트 케이스         | 평균 실행 시간(초) | 최소(초)  | 최대(초)  | 비고  |
  | --------------- | ----------- | ------ | ------ | --- |
  | movieId 해시 파티셔닝 | 28.879      | 28.603 | 29.226 |  movieId 인덱스 추가 전   |
  | movieId 해시 파티셔닝 | 10.487      | 10.269 | 10.850 |  movieId 인덱스 추가 후   |


#### 📌 3차 결론

1차 실험에서 예상했던 결과가 발생했다. 잘못된 파티셔닝은 오히려 시간이 오래걸렸다. 조인 성능 또한 인덱스 추가 및 쿼리문 수정을 통해 실행속도가 비약적으로 상승하였다.


## 📌 결론 및 회고

> 이번 프로젝트를 통해 **파티셔닝이 항상 성능 향상을 보장하는 만능 해결책이 아니라는 점**을 명확히 알 수 있었다. 초기 테스트에서는 파티셔닝 자체의 효과보다 기본 키 생성에 따른 인덱스 효과가 성능에 더 큰 영향을 미쳤다는 점을 발견했다. 이는 **실험 환경을 정확히 통제하는 것의 중요성**을 깨닫게 된 계기가 되었다.

> 가장 핵심적인 성능 향상 요인은 '**파티션 프루닝**'이었으며, 이를 위해서는 **조회 조건에 파티션 키를 명시적으로 사용하는 것이 필수적**이었습니다. `EXPLAIN`을 통해 쿼리 실행 계획을 분석하며, 조인 쿼리에서 옵티마이저가 파티션을 제대로 활용하도록 유도하는 것이 얼마나 중요한지 체감할 수 있었다.

> 결론적으로, 파티셔닝은 쿼리와 데이터 접근 패턴에 대한 깊은 이해를 바탕으로 **신중하게 설계될 때 비로소 강력한 성능 개선 도구가 될 수 있다는 것**을 배웠다. 이론으로만 알던 지식을 직접 실험하고 검증하며 데이터베이스 성능 최적화에 대한 실질적인 경험을 쌓을 수 있었던 의미 있는 프로젝트였다.

## 🚀 발전 가능성

> 프로젝트를 통해 얻은 가장 큰 자산은 **다양한 상황에 맞는 데이터베이스 성능 향상 기법을 적용할 수 있는 능력**이다.
> 파티셔닝, 인덱싱 등의 기법이 특정 쿼리 패턴과 데이터 분포에 따라 어떻게 성능에 영향을 미치는지 직접 실험하고 검증하며 실질적인 감각을 익힐 수 있었다.
> 이 경험을 토대로 앞으로 어떤 데이터베이스 시스템을 다루게 되더라도, **해당 시스템의 특성과 요구사항을 분석하여 고성능 데이터 조회가 가능하도록 발전**시킬 수 있을 것이다.

