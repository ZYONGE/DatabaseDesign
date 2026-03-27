# 시흥시 공공 자전거 대여 서비스 DB 설계

## 1. 프로젝트 개요
1. 경기도 시흥시의 열악한 교통환경을 개선하기 위해, 시민 대상 공공 자전거 대여 서비스를 기획한다.
2. 서비스 운영에 필요한 개체(엔터티)를 12개 이상 정의하고, MySQL 기반 관계형 데이터베이스를 설계한다.
3. 시흥시 전역(동 단위)으로 서비스 확장을 가정하여 지역별 수요, 대여소, 거치소, 운영 인력을 통합 관리한다.

## 2. 설계 목표
- 지역(동)별 인구 밀집도와 이용 수요를 반영한 자전거 배치 전략 지원
- 사용자 가입부터 대여, 반납, 결제, 마일리지, 리뷰까지 전 과정 데이터 추적
- 유지보수와 고장 신고를 포함한 운영 관제 데이터 축적
- 확장 가능한 스키마 구조(정규화 기반) 확보

## 3. 핵심 기능 범위
- 회원 관리: 가입, 상태 관리, 이용 제한 처리
- 자전거/거치소 관리: 재고, 상태, 배치 현황
- 대여/반납 처리: 시작 대여소, 반납 대여소, 이용 시간/거리 계산
- 결제/마일리지: 이용요금 결제 및 포인트 적립/사용
- 운영 관리: 정비 이력, 신고 이력, 관리자 처리 기록
- 사용자 경험 관리: 이용 리뷰 및 평점

## 4. 엔터티 목록 (14개)
1. `Region` (지역)
2. `Station` (대여소)
3. `Dock` (거치소)
4. `Bicycle` (자전거)
5. `User` (사용자)
6. `MembershipPlan` (정기권/요금제)
7. `PassPurchase` (정기권 구매 이력)
8. `Rental` (대여 이력)
9. `Payment` (결제 이력)
10. `MileageTransaction` (마일리지 변동 이력)
11. `Review` (리뷰)
12. `IncidentReport` (고장/민원 신고)
13. `Maintenance` (정비 이력)
14. `AdminStaff` (운영 관리자)

## 5. 엔터티별 주요 속성

### 5.1 Region
- `region_id` (PK)
- `region_name` (동 이름)

### 5.2 Station
- `station_id` (PK)
- `region_id` (FK -> Region)
- `station_name`

### 5.3 Dock
- `dock_id` (PK)
- `station_id` (FK -> Station)
- `dock_status` (EMPTY, OCCUPIED, BROKEN)

### 5.4 Bicycle
- `bicycle_id` (PK)
- `serial_no` (UNIQUE)
- `bike_status` (AVAILABLE, IN_USE, MAINTENANCE, LOST)
- `current_station_id` (FK -> Station, NULL 가능)

### 5.5 User
- `user_id` (PK)
- `login_id` (UNIQUE)
- `password_hash`
- `user_status` (ACTIVE, SUSPENDED, WITHDRAWN)

### 5.6 MembershipPlan
- `plan_id` (PK)
- `plan_name`
- `duration_days`
- `price`

### 5.7 PassPurchase
- `purchase_id` (PK)
- `user_id` (FK -> User)
- `plan_id` (FK -> MembershipPlan)
- `start_date`
- `end_date`

### 5.8 Rental
- `rental_id` (PK)
- `user_id` (FK -> User)
- `bicycle_id` (FK -> Bicycle)
- `start_station_id` (FK -> Station)
- `end_station_id` (FK -> Station, NULL 가능)
- `start_time`
- `end_time` (NULL 가능)
- `rental_status` (RENTED, RETURNED, OVERDUE)

### 5.9 Payment
- `payment_id` (PK)
- `user_id` (FK -> User)
- `amount`
- `payment_time`
- `payment_status` (SUCCESS, FAIL, CANCEL)

### 5.10 MileageTransaction
- `mileage_tx_id` (PK)
- `user_id` (FK -> User)
- `tx_type` (SAVE, USE, EXPIRE, ADJUST)
- `points`
- `tx_time`

### 5.11 Review
- `review_id` (PK)
- `rental_id` (FK -> Rental, UNIQUE)
- `user_id` (FK -> User)
- `rating` (1~5)

### 5.12 IncidentReport
- `incident_id` (PK)
- `reporter_user_id` (FK -> User)
- `incident_type` (BROKEN, SAFETY, LOST, ETC)
- `reported_at`
- `incident_status` (OPEN, IN_PROGRESS, RESOLVED)

### 5.13 Maintenance
- `maintenance_id` (PK)
- `bicycle_id` (FK -> Bicycle)
- `staff_id` (FK -> AdminStaff)
- `started_at`
- `ended_at`

### 5.14 AdminStaff
- `staff_id` (PK)
- `staff_name`
- `role` (OPERATOR, ENGINEER, ADMIN)

## 6. 관계 설계 (ERD 기준)
- Region (1) - (N) Station
- Station (1) - (N) Dock
- Station (1) - (N) Bicycle (현재 위치 기준, 논리적 관계)
- User (1) - (N) Rental
- Bicycle (1) - (N) Rental
- Station (1) - (N) Rental (start_station_id)
- Station (1) - (N) Rental (end_station_id)
- MembershipPlan (1) - (N) PassPurchase
- User (1) - (N) PassPurchase
- User (1) - (N) Payment
- Rental (1) - (0..1) Review
- User (1) - (N) MileageTransaction
- User (1) - (N) IncidentReport
- Bicycle (1) - (N) Maintenance
- AdminStaff (1) - (N) Maintenance

## 7. 비즈니스 규칙
1. 자전거는 `bike_status = AVAILABLE`이고 거치 가능한 대여소가 있을 때만 대여 가능하다.
2. 진행 중 대여(`rental_status = RENTED`)는 사용자당 동시에 1건만 허용한다.
3. 반납 시 `end_time`, `end_station_id`를 기록하고 상태를 `RETURNED`로 변경한다.
4. 이용요금은 서비스 요금 정책에 따라 계산한다.
5. 리뷰는 반납 완료된 대여 건에 대해 1회만 작성 가능하다.
6. 신고 접수 후 정비 이력이 생성되면 `IncidentReport.incident_status`는 `IN_PROGRESS` 이상이어야 한다.

## 8. 정규화 요약
- 제1정규형(1NF): 반복 그룹 제거, 모든 속성 원자값 유지
- 제2정규형(2NF): 복합키 부분 종속 제거 (대부분 단일 PK 사용)
- 제3정규형(3NF): 이행 종속 제거
  - 지역 정보는 Region으로 분리
  - 요금제 정보는 MembershipPlan으로 분리
  - 마일리지 변동은 별도 이력 테이블로 분리

## 9. 인덱스 및 제약조건 설계
- UNIQUE 인덱스
  - `User.login_id`
  - `Bicycle.serial_no`
  - `Review.rental_id`
- 검색 인덱스
  - `Rental(user_id, start_time)`
  - `Rental(bicycle_id, start_time)`
  - `Payment(user_id, payment_time)`
  - `IncidentReport(incident_status, reported_at)`
- CHECK 제약
  - `rating BETWEEN 1 AND 5`
  - `amount >= 0`
  - `points <> 0`

## 10. 핵심 SQL 스키마 예시 (MySQL)
```sql
CREATE TABLE Region (
    region_id INT AUTO_INCREMENT PRIMARY KEY,
    region_name VARCHAR(50) NOT NULL UNIQUE
);

CREATE TABLE Station (
    station_id INT AUTO_INCREMENT PRIMARY KEY,
    region_id INT NOT NULL,
    station_name VARCHAR(100) NOT NULL,
    CONSTRAINT fk_station_region
        FOREIGN KEY (region_id) REFERENCES Region(region_id)
);

CREATE TABLE User (
    user_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    login_id VARCHAR(50) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    user_status VARCHAR(20) NOT NULL
);

CREATE TABLE Bicycle (
    bicycle_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    serial_no VARCHAR(100) NOT NULL UNIQUE,
    bike_status VARCHAR(20) NOT NULL,
    current_station_id INT,
    CONSTRAINT fk_bicycle_station
        FOREIGN KEY (current_station_id) REFERENCES Station(station_id)
);

CREATE TABLE Rental (
    rental_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    bicycle_id BIGINT NOT NULL,
    start_station_id INT NOT NULL,
    end_station_id INT,
    start_time DATETIME NOT NULL,
    end_time DATETIME,
    rental_status VARCHAR(20) NOT NULL,
    CONSTRAINT fk_rental_user
        FOREIGN KEY (user_id) REFERENCES User(user_id),
    CONSTRAINT fk_rental_bicycle
        FOREIGN KEY (bicycle_id) REFERENCES Bicycle(bicycle_id),
    CONSTRAINT fk_rental_start_station
        FOREIGN KEY (start_station_id) REFERENCES Station(station_id),
    CONSTRAINT fk_rental_end_station
        FOREIGN KEY (end_station_id) REFERENCES Station(station_id)
);
```

## 11. 보고서 작성 시 추가할 분석 항목
- 시간대별 대여량(출퇴근 시간 피크) 분석
- 동별 자전거 회전율과 재배치 필요 지수 산출
- 자전거 고장률과 정비 주기 상관관계 분석
- 사용자 유형(연령/직업)별 이용 패턴 분석

## 12. 기대 효과
- 시민 이동 편의 향상 및 단거리 이동 교통 분산
- 데이터 기반 자전거 배치/정비 의사결정 가능
- 운영 효율화 및 사용자 만족도 개선

## 13. 정보 공개_사용자
- 본인 계정 정보, 대여/반납 이력, 결제 이력, 마일리지 내역만 조회 가능
- 타 사용자 개인정보 및 운영 내부 데이터는 비공개
- 리뷰 작성 시 닉네임 기반 최소 정보만 공개

## 14. 정보 공개_관리자
- 전체 대여소/자전거 상태, 신고/정비 이력, 통계 데이터 조회 가능
- 사용자 개인정보는 운영 목적 범위에서 최소 항목만 열람
- 권한(운영자/정비자/관리자)별 조회 및 수정 기능을 분리