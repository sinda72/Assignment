# 쿠폰 API 개발 과제

## INDEX
>1. **로컬 환경 서버 구동 가이드**
>2. 설계   
>   a. 요구사항   
>   b. 요구사항분석   
>   c. 데이터 모델링   
>   d. **ERD 캡처화면**   
>   e. API 별 흐름 정리 및 I/O 정의
>3. **API 명세**
>4. **문제 해결 전략**
-----
1. 로컬 환경 서버 구동 가이드   

    1. Spring boot 프로젝트 시작 (Spring Initializr)   
        1. IntelliJ IDEA 2022.1.3
        2. Spring Boot 2.7.5
        3. JAVA 11
        4. Gradle
        5. jar
        6. dependency 
            1. Spring Web
            2. MySQL
            3. JPA
            4. Lombok   
    2. application.properties
    
    
    ```markup
    # MySQL
    spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
    spring.datasource.url=jdbc:mysql://localhost:3306/Karrotpay?useSSL=false&useUnicode=true&serverTimezone=Asia/Seoul
    spring.datasource.username=root
    spring.datasource.password=tlsekdud98@A
    
    server.port=8080
    ```
    
    3. MySQL   
        a. DDL
        
        ```sql
        CREATE SCHEMA 'KARROTPAY';
        CREATE DATABASE 'KARROTPAY';
        ```
        
        ```sql
        //사용자 테이블
        CREATE TABLE USER(
           UID VARCHAR(13) NOT NULL,
           USER_NM VARCHAR(10) NULL,
           USER_PNO VARCHAR(15) NULL,
           USER_EMAIL VARCHAR(20) NULL,
           USER_ADDRESS VARCHAR(100) NULL,
           USER_DT TIMESTAMP DEFAULT CURRENT_TIMESTAMP not null,
           PRIMARY KEY (`UID`));
        ```
        
        ```sql
        //쿠폰 테이블
        CREATE TABLE COUPON(
           CP_CD VARCHAR(20) NOT NULL,
           CP_NM VARCHAR(30) NULL,
           CP_AMT INT NULL,
           CP_TOTAL_CNT INT NULL,
           CP_REMAIN_CNT INT NULL,
           PRIMARY KEY (`CP_CD`));
        ```
        
        ```sql
        //쿠폰발급내역 테이블
        CREATE TABLE COUPON_HISTORY(
           CP_ID VARCHAR(20) NOT NULL,
           UID VARCHAR(13) NOT NULL,
           CP_CD VARCHAR(20) NOT NULL,
           CP_NM VARCHAR(30) NULL,
           CP_BALKEUP_DT TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
           CP_USE_DT TIMESTAMP NULL,
           CP_USAGE INT DEFAULT 0,
           PRIMARY KEY (`CP_ID`),
           INDEX `fk_COUPON_HISTORY_USER_idx` (`UID` ASC) VISIBLE,
           INDEX `fk_COUPON_HISTORY_COUPON_idx` (`CP_CD` ASC) VISIBLE
        );
        ```
        
    
        b. 테이블 데이터 초기 세팅 및 PK 트리거 생성
    
        - COUPON 테이블 값 INSERT
    
        ```sql
        INSERT INTO COUPON(CP_CD, CP_NM, CP_AMT, CP_TOTAL_CNT, CP_REMAIN_CNT)
        VALUES ("C0001", "의류할인쿠폰", 50000, 1000, 999);
        //테스트 확인을 위해 coupon_history 테이블에 의류할인쿠폰 1개 발행되어 남은 수량 999개
        INSERT INTO COUPON(CP_CD, CP_NM, CP_AMT, CP_TOTAL_CNT, CP_REMAIN_CNT)
        VALUES ("E0001", "전자제품할인쿠폰", 100000, 300, 300);
        ```
    
        - USER 테이블 값 INSERT
    
        ```sql
        INSERT INTO USER(UID, USER_NM, USER_PNO, USER_EMAIL, USER_ADDRESS, USER_DT)
        VALUES ("sinda72", "신다영", "01053730000", "sinda72@naver.com", "경기도 ---", current_timestamp()),
           ("karrot0", "박당근", "01013310000", "karrot2@naver.com", "서울시 ---", current_timestamp()),
           ("karrot3", "강당근", "01012210000", "karrot3@naver.com", "서울시 ---", current_timestamp()),
           ("karrot4", "멍당근", "01012210220", "karrot4@naver.com", "서울시 ---", current_timestamp()),
           ("karrot", "김당근", "01012210220", "karrot@naver.com", "서울시 ---", current_timestamp());
        ```
    
        - COUPON_HISTORY 테이블 값 INSERT (테스트 확인을 위해)

        ```sql
        INSERT INTO COUPON_HISTORY(CP_ID, UID, CP_CD, CP_NM, CP_BALKEUP_DT, CP_USE_DT, CP_USAGE)
        VALUES ("2022112200001", "karrot0", "C0001", "의류할인쿠폰", 2022-11-22 17:17:48, null, 0);
        ```
    
        - COUPON_HISTORY 테이블 PK (CP_ID) 날짜+일련번호 조합 자동 생성 TRIGGER 구현

        ```sql

        //t_seq에서 일련번호 받아와 sysdate와 조합
        //t_seq을 매일 갱신해주는 작업이 필요할 거 같음
        create table t_seq(
            id int not null auto_increment primary key
        );
    
        DELIMITER $$
        create trigger tg_insert
        before insert on coupon_history
        for each row
        begin
            insert into t_seq values (null);
            set NEW.cp_id = CONCAT(date_format(sysdate(), "%Y%m%d"), LPAD(LAST_INSERT_ID(), 5, '0'));
        END$$
        DELIMITER ;
        ```
    
    4. POSTMAN 을 통해 API 명세에 명시된 URI로 응답 확인

        - 응답 확인 전 정상케이스 확인을 위해서는 데이터 초기화 작업 진행 필요합니다. (위의 데이터 초기 세팅)
----
2. 설계
    1. 요구사항
        1. 동시에 많은 요청이 몰릴 수 있다고 가정해요.
        2. 서버 애플리케이션은 단일 서버가 아닌, 멀티 서버 환경에서 동작한다고 가정해요.
    2. 요구사항분석
        1. 다음의 5가지 기능을 가진다. 쿠폰 발급, 쿠폰 사용, 사용자가 보유한 쿠폰 목록 조회, 쿠폰 ID로 쿠폰 정보 조회, 전체 쿠폰 발급 현황 조회 
        2. 의류 할인 쿠폰 5만원권과 전자제품 할인 쿠폰 10만원권으로 쿠폰은 두 가지 종류이다.
        3. 사용자는 두 가지의 쿠폰을 모두 발급 받을 수 있다.
        4. 사용자는 같은 쿠폰을 중복으로 가질 수 없다. 
        5. 각 쿠폰은 한정된 수량을 가진다. 각 각 1000장, 300장
        6. **예외처리와 검증 꼼꼼하게**
        7. 공통기능 그룹화 여부 고민 (데이터 검증)
    3. 데이터 모델링
        1. PK의 데이터 타입은 문자형으로 지정한다. (숫자형의 제한된 크기 한계)
        2. COUPON_HISTORY TABLE의 식별자는 트리거 생성해서 기본키의 유일성을 보장한다.
        3. COUPON_HISTORY TABLE은 코드 ID(대표 PK)만 알아도 원하는 레코드 접근 가능한 대표키를 가진다.
        4. COUPON_HISTORY TABLE은 대표 PK가 존재하더라도 각 FK 조합이 유니크한 특징을 가짐. 또한, 다수의 UID와 CP_CODE를 사용한 조회 작업을 위한 해당 컬럼의 인덱스를 생성한다.
        5. 동시성 제어를 위한 솔루션을 제공한다. - 트랜잭션과 비관적락 방식 (읽고 쓰기 작업이 모두 불가한 배타락(SELECT ~FOR UPDATE))
        6. 파티셔닝 적용 여부를 고려한다. - X (과제에서 가정한 쿠폰은 총 1300장. 다른 쿠폰이 존재한다고 가정해도 쿠폰이라는 서비스 특성 상 하루에 제공하는 쿠폰 개수는 제한적이라 판단)
    4. ERD
        
        ![KARROT_ERD_PNG](https://user-images.githubusercontent.com/43509229/203265043-7e93213c-800d-4229-8b5d-a6a6239d3492.png)
        ![Untitled (1)](https://user-images.githubusercontent.com/43509229/203265123-271417e4-0d43-4fe3-9b01-1bd751fadf6c.png)

        
    
    5. API 별 흐름 정리 및 I/O 정의 (서비스 단위 그룹화)
    
        **[ IssueCouponService.java ]**

        1. **쿠폰 발급 API**


            | IN | OUT |
            | --- | --- |
            | 사용자 ID | 쿠폰 ID |
            | 쿠폰 코드 | 쿠폰 코드 |
            |  | 쿠폰 발급 일시 |

            : USER T (SELECT), COUPON T (SELECT), COUPON_HISTORY T (SELECT, UPDATE)

            1. 존재하는 사용자인지 확인
            2.  쿠폰 코드 확인 + 해당 쿠폰의 잔여 수량 확인 (쿠폰 발급 수량은 제한되어있다.)
            3.  사용자 ID와 쿠폰 코드 조합이 이미 존재하는 지 확인 (사용자는 중복 쿠폰을 가질 수 없고, 이미 사용한 쿠폰이더라도 중복 발급은 불가하다.)
            4.  사용자 ID와 쿠폰 코드를 받아 쿠폰 발급 테이블에 거래 INSERT 

                : 쿠폰 ID, 사용자 UID, 쿠폰명, 쿠폰코드, 발급 일시, 사용 일시, 사용 여부(N)

            5.  쿠폰 테이블 잔여 수량 UPDATE
            - 동시성 제어를 위한 트랜잭션 고려 [위 작업의 원자성 보장]
                - @Lock(LockModeType.PESSIMISTIC_WRITE) 선언해 select for update 설정
            - 쿠폰 내역 테이블 KEY 값 (CP_ID) 트리거 생성 (날짜 + 일련번호)

        **[ UseCouponService.java ]**

        2. **쿠폰 사용 API**


            | IN | OUT |
            | --- | --- |
            | 사용자 ID | 쿠폰 ID |
            | 쿠폰 ID | 쿠폰 코드 |
            |  | 쿠폰 사용 일시 |

            : USER T (SELECT), COUPON T (SELECT)

            : COUPON_HISTORY T ( SELECT, UPDATE )

            1. 존재하는 사용자인지 확인
            2. 사용자 ID와 쿠폰 ID 존재 여부 확인
            3. 쿠폰 ID에 해당하는 쿠폰 레코드 사용하지 않은 쿠폰인지 확인
            4. 사용자 ID와 쿠폰 ID에 해당하는 쿠폰 레코드 - 사용일시, 사용여부 갱신

        **[ SelectCouponService.java ]**

        - 조건 컬럼에 따른 COUPON HISTORY 조회 (찾고자 하는 특정 데이터) 그룹
        3. **사용자가 보유한 쿠폰 목록 조회 API**


            | IN | OUT |
            | --- | --- |
            | 사용자 ID | 쿠폰 ID |
            |  | 쿠폰 코드 |
            |  | 쿠폰 발급 일시 |
            |  | 쿠폰 사용 여부 |
            |  | 쿠폰 사용 일시(NULLABLE) |

            : USER T (SELECT)

            : COUPON_HISTORY T ( SELECT )

            1. 존재하는 사용자인지 확인
            2. SELECT WHERE 사용자 ID, 사용 가능 쿠폰

        4. **쿠폰 ID로 쿠폰 정보 조회 API**


            | IN | OUT |
            | --- | --- |
            | 쿠폰 ID | 사용자 ID |
            |  | 쿠폰 ID |
            |  | 쿠폰 코드 |
            |  | 쿠폰 발급 일시 |
            |  | 쿠폰 사용 여부 |
            |  | 쿠폰 사용 일시(NULLABLE) |

            : COUPON_HISTORY T ( SELECT )

            1. 존재하는 쿠폰 ID인지 확인
            2. SELECT

        **[ SelectAllService.java ]**

        - 단순 테이블 현황 확인을 위한 그룹
        5. **전체 쿠폰 발급 현황 조회 API**


            | IN | OUT |
            | --- | --- |
            |  | 쿠폰 코드 |
            |  | 쿠폰명 |
            |  | 쿠폰 금액 |
            |  | 전체 쿠폰 수량 |
            |  | 남아있는 쿠폰 수량 |

            COUPON T (SELECT)

            1. SELECT 해서 LIST, Pagination 활용해 표현 
    
3. API 명세
    
    
    | INDEX | METHOD | URI | Description |
    | --- | --- | --- | --- |
    | 1 | POST | http://localhost:8080/issue/coupon?uid=karrot&cpCd=C0001 | 쿠폰 발급 API |
    | 2 | POST | http://localhost:8080/use/coupon?uid=karrot0&cpId=2022112200001 | 쿠폰 사용 API |
    | 3 | GET | http://localhost:8080/select/userCoupon?uid=karrot | 사용자 보유 쿠폰 조회 API |
    | 4 | GET | http://localhost:8080/select/cpIdCoupon?cpId=2022112200001 | 쿠폰ID로 쿠폰 정보 조회 API |
    | 4 | GET | http://localhost:8080/selectAll/coupon | 전체 쿠폰 발급 현황 조회API |

4. 문제 해결 전략

- 필요한 엔티티와 각 기능 별 로직 흐름 파악 및 주어진 조건에 해당하는 로직 구성 ex) 쿠폰 제한 수량 등
- API 특징 별 서비스 그룹화 (5개의 API에서 4개 서비스 도출)
- 메인 테이블의 유니크한 키 생성을 위한 생성 방안 고민
    - 쿠폰 내역 테이블 KEY 값 (CP_ID) 시퀀스 생성 (날짜 + 일련번호)
    - 아쉬운 점, 일련번호를 생성하는 테이블이 매일 초기화되면 좋겠다.
- 동시성 처리를 위한 트랜잭션 방법 고려
    - 데이터의 일관성 보장 최우선 고려
    - 격리 수준 MYSQL의 DEFAULT는 REAPEATABLE READ(현재 트랜잭션보다 작은 트랜잭션에서 변경한 데이터만 보여줌)
    - SEALIZAL은 동시성 처리 성능이 떨어져 X - PHANTOM READ 문제는 LOCK으로 해결
    - 비관적 락 - 한정된 쿠폰 수량을 순차적으로 제공하기 위해 필요할 것으로 예상한다. 성능은 낙관적 락보다 안 좋지만, 한정된 쿠폰 수량이라는 점에서 대기 후 순차적으로 진행하는 것이 낙관적 락 사용으로 인해 발생 가능한 자원 낭비 관점에서 더욱 좋을 것으로 판단된다.
        - `@Lock(value = LockModeType.PESSIMISTIC_WRITE)`
    - 트랜잭션 내부에서 새로운 트랜잭션을 생성하는 경우는 해당하지 않아 전파 속성은 고려하지 않음.
- 발생 가능한 오류에 대한 예외처리
- 당근페이 기술스택 사용 (SPRING, JAVA, MYSQL, JPA)
- 테스트 코드 작성해 확인 후 코드 작성 및 개발한 서비스 테스트 코드 작성
- POSTMAN 으로 API 동작 확인
