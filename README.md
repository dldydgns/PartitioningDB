# PartitioningDB
MySQL 파티션을 활용한 대용량 데이터처리 성능 개선 학습 프로젝트입니다.

# 🧪 PartitionBench

**국민건강보험공단 건강검진 통계 데이터를 활용하여, MySQL의 파티셔닝 기능이 성능에 어떤 영향을 미치는지를 분석하는 프로젝트입니다.**  
일반 테이블과 파티셔닝 테이블에서 동일한 조건의 조회 쿼리를 실행하고, 실행 시간을 비교하여 성능 개선 효과를 확인합니다.

---

## 📌 프로젝트 목표

- 건강검진 통계 데이터를 기반으로 한 대용량 테이블 구축
- **MySQL 파티셔닝 전략**(RANGE, LIST 등)을 적용한 테이블 구성
- 동일 데이터에 대해 **비파티셔닝 vs 파티셔닝 테이블** 성능 비교
- JDBC(Java)로 성능 측정 및 결과 출력

---

## ⚙️ 기술 스택

| 구분 | 사용 도구 |
|------|-----------|
| DBMS | MySQL 8.x |
| DB 툴 | DBeaver |
| 백엔드 | Java 21 |
| JDBC 드라이버 | mysql-connector-java |
| 빌드 도구 | Gradle |
| OS | Windows 11 or macOS |

---

## 🗂️ 데이터 출처

- 국민건강보험공단 건강검진 통계 (공공데이터포털)
- [https://www.data.go.kr/](https://www.data.go.kr/)

---

## 📈 테이블 구성

| 테이블명 | 설명 |
|----------|------|
| `health_stat_normal` | 일반 테이블 (파티셔닝 없음) |
| `health_stat_partitioned` | 파티셔닝 적용 테이블 (예: 연도별 RANGE 파티셔닝) |

---

## 📋 진행 계획

| 단계 | 내용 | 상태 |
|------|------|------|
| 1단계 | CSV → MySQL 데이터 이관 | ✅ 완료 |
| 2단계 | 일반/파티셔닝 테이블 설계 및 생성 | 🔄 진행 중 |
| 3단계 | Java(JDBC) 프로젝트 생성 | 🔲 예정 |
| 4단계 | 조건별 조회 쿼리 및 시간 측정 | 🔲 예정 |
| 5단계 | 성능 비교 결과 정리 및 결론 도출 | 🔲 예정 |

---

## ▶️ 실행 방법

### 1. 환경 설정

```bash
# 프로젝트 클론
git clone https://github.com/yourusername/PartitionBench.git
cd PartitionBench

