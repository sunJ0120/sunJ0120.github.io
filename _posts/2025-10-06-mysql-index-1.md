---
title: "🎱 MySQL 인덱스 실습 - 최대 83배 성능 개선 경험기"
layout: post
categories: [MySQL, RDBMS]
tags: [MySQL, index]
toc: true
toc_sticky: true
toc_label: 목차
author_profile: true
permalink: /MySQL/index/1
---

# 🖤 Intro
## 인덱스, 알고는 있었지만…

"인덱스 쓰면 빨라진다"는 건 누구나 안다. 면접에서도 단골 질문이고, 실무에서도 당연히 써야 한다고들 하지만... 솔직히 말하면 **얼마나** 빨라지는지는 직접 경험해본 적이 없었다.

십만 건? 백만 건? 그 정도 데이터를 실제로 다뤄본 적도 없고, 인덱스 없이 쿼리 돌려보고 "아 너무 느리네" 하면서 인덱스 걸어보고 "오 빨라졌다!" 이런 경험도 없었다.

그냥 막연하게 "인덱스는 좋은 거다", "B-Tree 구조로 되어 있다", "카디널리티가 높아야 한다" 이런 이론만 알고 있었을 뿐이다.

## 그래서 직접 해보기로 했다!!

마침 RealMySQL을 공부하면서 마침 딱 B-Tree 인덱스 파트 공부를 마친 참이라, 직접 실습해보기로 했다. 난 궁금한건 못참아!!!!!!

> 십만 건의 로그 데이터를 직접 생성하고
> 인덱스 없이 쿼리 성능을 측정해보고
> 단일 인덱스, 복합 인덱스를 각각 적용해서

진행했다..

결과를 미리 말하자면, 최대 **83배** 빨라지는 어마무시한..결과가 나왔다.

도파민이 퐁퐁 터지는 결과라 지금도 설레는 마음이 그득하다

그럼 어떤 과정으로 실습을 진행했는지를 더 구체적으로 살펴보자.

# 🩶 Start
## 실습을 시작하지~

실습의 단계는 다음과 같다.

1. 기본 설정 및 성능 분석을 위한 더미 데이터 추가
2. 기본 crud 구성 및 기능 test
3. 본격적인 성능 테스트 시작

## 기본 설정 및 성능 분석을 위한 더미 데이터 추가

### 스키마 생성 및 권한 부여

```sql
CREATE SCHEMA `indextest` DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci ;
```

지금 단계에서는 MySQL workbench를 사용하는 건 생략되어 있지만... 우선 스키마를 생성하고 다음의 조건을 넣어주었다.

- utf8mb4을 이용해서 이모지와 같은 특수문자도 완벽하게 저장해서 진짜 utf8을 사용할 수 있도록 한다.
- utf8mb4_unicode_ci를 이용해서 정확한 정렬이 가능하게 하고, 대소문자 구분을 안 하도록 한다.

```sql
-- 1. 사용자 생성
CREATE USER 'indextest_user'@'%' IDENTIFIED BY 'indextest1234';
-- 2. 스키마 권한 부여
GRANT ALL PRIVILEGES ON indextest.* TO 'indextest_user'@'%';
-- 3. 권한 적용
FLUSH PRIVILEGES;
```

마지막으로, 스키마를 사용할 사용자를 생성하고 권한을 부여해주면 실습에 사용할 스키마 구성은 끝이다.

### yml 파일 생성

```yaml
# 포트 설정
server:
  port: 8085

spring:
  datasource:
    url: jdbc:mysql://${DB_HOST:localhost}:${DB_PORT:3306}/${DB_NAME}?characterEncoding=UTF-8&serverTimezone=Asia/Seoul
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    driver-class-name: com.mysql.cj.jdbc.Driver
  jpa:
    hibernate:
      ddl-auto: update
    properties:
      hibernate:
        show_sql: true
        format_sql: true
        dialect: org.hibernate.dialect.MySQL8Dialect
        cache:
          use_second_level_cache: false  # 정확한 인덱싱 측정을 위한 캐시 끄기
          use_query_cache: false # default가 false로 되어 있음
```

yml 설정 파일이다.

cache단이야... 원래 기본값이 false이긴 하지만, 명확하게 명시하기 위해 false로 잡아준다.

username, password 같은 민감 정보는 전부 .env로 빼주었다.

### 더미 데이터 생성!

```java
/**
 * 로그 더미 데이터 저장을 위한 클래스
 *
 * ApplicationRunner를 이용해서 스프링 부트가 완전히 시작된 직후 자동으로 시작.
 * 배치 방식으로 묶어서 한 번에 저장
 **/
@Component
@RequiredArgsConstructor
@Slf4j
public class DataInitializer implements ApplicationRunner {
    private final LogRepository logRepository;

    @Override
    public void run(ApplicationArguments args) throws Exception {
        // 애플리케이션을 시작하면서 자동으로 실행된다.
        log.info("더미데이터 생성 시작");

        // 이미 해당하는 데이터가 있을 경우, 실행된 것이므로 스킵한다.
        if(logRepository.count() > 0){
            log.info("데이터가 이미 존재하므로, 스킵합니다.");
            return;
        }

        // 10만개의 더미 데이터 로그를 생성
        List<Logs> logs = new ArrayList<>();
        Random random = new Random();

        // Level enum의 모든 값을 배열로 가져옴
        Level[] levels = Level.values();  // [ERROR, WARN, INFO, DEBUG]

        for(int i = 1; i <= 100000; i++){
            Logs logEntity = Logs.builder()
                    .dateTime(LocalDateTime.now().minusDays(random.nextInt(30))) //최근 30일을 랜덤하게 넣는다.
                    .level(levels[random.nextInt(levels.length)]) //길이 내에서 랜덤 정수를 생성
                    .userId((long) random.nextInt(10000)) //10000 중에서 랜덤한 숫자를 생성
                    .message("로그 메세지 : " + i)
                    .build();

            logs.add(logEntity); //배열에 더하기//배치 방식으로, 1000 단위로 저장
            if(i % 1000 == 0){
                logRepository.saveAll(logs);
                logs.clear();
                log.info("{}개 째 저장 완료", i);
            }
        }
        log.info("더미 데이터 생성 완료.");
    }
}
```

우선 더미 데이터를 생성하기 위한 메서드를 만들어야 한다.

이 토이 프로젝트의 목적은 로그 데이터를 생성해서 select로 비교하기 위함이다.

ApplicationRunner는 스프링 프로젝트 시작 시점에 실행된다는 특징이 있다. 이를 이용하면 애플리케이션 시작 시점에 로그 데이터들을 생성할 수 있다.

데이터가 이미 존재하는 상태에서 데이터를 넣어주는 건 낭비이므로, 데이터가 존재하면 해당 메서드를 스킵하도록 한다.

효율성을 위해, 배치 방식으로 1000개 단위로 묶어서 saveAll로 한 번에 저장되도록 한다.

![로그데이터.png](/images/2025-10-06-mysql-index-1/로그데이터.png)

더미 데이터가 잘 들어간 것을 볼 수 있다.

### Controller

```java
@Controller
@Slf4j
@RequiredArgsConstructor
@RequestMapping("/api/logs")
public class LogController {
    private final LogService logService;
    /**
     * GET /api/logs : 전체 로그 조회
     */
    @GetMapping
    public ResponseEntity<List<LogResponse>> findAll(){
        List<LogResponse> logs = logService.findAll();

        return ResponseEntity.ok(logs);
    }

    /**
     * GET /api/logs/search : 조건별 검색
     *
     * 로그 조회의 핵심으로, 이를 통해 해당 요소를 인덱스로 설정했을때, 성능이 얼마나 빨라지는지를 알 수 있다.
     */
    @GetMapping("/search")
    public ResponseEntity<List<LogResponse>> search(
            @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime dateTime, //dateTime은 controller에서 변환해야 깔끔하다.
            @RequestParam(required = false) Level level, //ENUM 타입 자동 반환
            @RequestParam(required = false) String message,
            @RequestParam(required = false) Long userId
    ){
        List<LogResponse> logs = logService.searchByConditions(dateTime, level, message, userId);

        return ResponseEntity.ok(logs);
    }

    /**
     * GET /api/logs/performance : 인덱스 성능 테스트
     *
     * 인덱스 성능 테스트를 위한 api
     * 이를 통해 인덱스를 생성했을때, 어떻게 생성했는지에 따른 시간을 측정하기 위함이다.
     */
    @GetMapping("/performance")
    public ResponseEntity<PerformanceTestResponse> performance(){
        PerformanceTestResponse log = logService.performanceTest();

        return ResponseEntity.ok(log);
    }
}
```

REST API를 분석하는 단계가 아니므로 controller-service-repository에 대한 깊은 설명은 하지 않겠다.
controller의 경우는 전체 로그 검색 & 조건별 검색(쿼리) & 성능 테스트 세 가지로 나눠서 구성하였다.
참고로, 성능 테스트의 경우 시작 시점과 쿼리 종료 시점의 ms를 비교해서 인덱스 사용 전후의 시간 차이를 비교하기 위해 생성한 것이다.

```java
/**
 * index test를 위한 응답을 담은 간단한 dto
 */
@Getter
@Builder
@AllArgsConstructor
@NoArgsConstructor
public class PerformanceTestResponse {
    // 전체 로그 개수
    private Long totalRecords;

    // 단일 컬럼 인덱스 테스트
    private Long levelSearchTime;
    private Long userIdSearchTime;
    private Long messageSearchTime;
    // 추가) containing에서 인덱스 실험
    private Long messageContainingSearchTime;

    // 복합 인덱스 테스트
    private Long levelAndUserIdSearchTime;
    private Long levelAndMessageSearchTime;

    // 조회된 결과 개수 (옵션)
    private Integer levelSearchCount;
    private Integer userIdSearchCount;
    private Integer levelAndUserIdSearchCount;

    // 인덱스 존재 여부 (나중에 비교용)
    private String testDescription; // "인덱스 전" or "인덱스 후"
}
```

참고로 그래서 이 성능 측정 dto는 이렇게 구성되어 있다.

### Service

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class LogService {
    private final LogRepository logRepository;
    private final EntityManager entityManager;  // LogService - 직접 사용 용 (캐시 클리어, 네이티브 쿼리 등)

    /**
     * 전체 로그를 조회하기 위한 method
     * @return
     */
    public List<LogResponse> findAll(){
        List<Logs> logEntityList = logRepository.findAll();
        List<LogResponse> logResponseList = getLogResponses(logEntityList);

        return logResponseList;
    }

    /**
     * 조건별로 로그를 조회하기 위한 method
     */
    public List<LogResponse> searchByConditions(
            LocalDateTime dateTime,
            Level level,
            String message,
            Long userId){
        // 파라미터들이 모두 선택 쿼리이기 때문에, 동적 쿼리가 필요.

        List<Logs> logEntityList = logRepository.searchByConditions(dateTime, level, message, userId);
        List<LogResponse> logResponseList = getLogResponses(logEntityList);

        return logResponseList;
    }

    /**
     * 인덱스의 정확한 성능을 검수하기 위한 method
     */
    public PerformanceTestResponse performanceTest(){
        long startTime; //성능 측정을 위한 시작 시간 명시

        // 1. level 검색
        entityManager.clear(); // 각 테스트마다 독립적 측정을 위해 clear 해줘야 한다.
        startTime = System.currentTimeMillis();
        List<Logs> errorLogs = logRepository.findByLevel(Level.ERROR);
        long levelSearchTime = System.currentTimeMillis() - startTime;

        // 2. userId 검색
        entityManager.clear(); // 각 테스트마다 독립적 측정을 위해 clear 해줘야 한다.
        startTime = System.currentTimeMillis();
        List<Logs> userLogs = logRepository.findByUserId(1L);
        long userIdSearchTime = System.currentTimeMillis() - startTime;

        // 3. message 검색
        entityManager.clear(); // 각 테스트마다 독립적 측정을 위해 clear 해줘야 한다.
        startTime = System.currentTimeMillis();
        List<Logs> messageLogs = logRepository.findByMessageStartingWith("가나디"); // LIKE '가나디%' 체크를 위해 StartingWith으로 한다.
        long messageSearchTime = System.currentTimeMillis() - startTime;

        // 3-2. message 검색 2
        // 이번에는, 인덱스가 통하지 않는 contain으로 인덱스의 효과를 측정해보자.
        entityManager.clear(); // 각 테스트마다 독립적 측정을 위해 clear 해줘야 한다.
        startTime = System.currentTimeMillis();
        List<Logs> messageContainingLogs = logRepository.findByMessageContaining("가나디"); // LIKE '%가나디%' 체크를 위해 Containing으로 한다.
        long messageContainingSearchTime = System.currentTimeMillis() - startTime;

        // 복합 검색 (level + userId)
        entityManager.clear(); // 각 테스트마다 독립적 측정을 위해 clear 해줘야 한다.
        startTime = System.currentTimeMillis();
        List<Logs> levelAndUserIdLogs = logRepository.findByLevelAndUserId(Level.ERROR, 1L);
        long levelAndUserIdSearchTime = System.currentTimeMillis() - startTime;

        // 복합 검색 (level + message)
        entityManager.clear(); // 각 테스트마다 독립적 측정을 위해 clear 해줘야 한다.
        startTime = System.currentTimeMillis();
        List<Logs> levelAndMessageLogs = logRepository.findByLevelAndMessageStartingWith(Level.ERROR, "가나디");
        long levelAndMessageSearchTime = System.currentTimeMillis() - startTime;

        return PerformanceTestResponse.builder()
                .totalRecords(logRepository.count())
                .levelSearchTime(levelSearchTime)
                .userIdSearchTime(userIdSearchTime)
                .messageSearchTime(messageSearchTime)
                .messageContainingSearchTime(messageContainingSearchTime) //인덱스가 안먹히는지 봐야 한다.
                .levelAndUserIdSearchTime(levelAndUserIdSearchTime)
                .levelAndMessageSearchTime(levelAndMessageSearchTime)
                .levelSearchCount(errorLogs.size())
                .userIdSearchCount(userLogs.size())
                .levelAndUserIdSearchCount(levelAndUserIdLogs.size())
                .testDescription("현재 인덱스 상태")
                .build();
    }

    // entity to dto를 위한 method
    private static List<LogResponse> getLogResponses(List<Logs> logEntityList) {
        List<LogResponse> logResponseList = logEntityList.stream()
                .map(logs -> LogResponse.from(logs))
                .collect(Collectors.toList());
        return logResponseList;
    }
}
```

서비스에서 repository에서 쿼리 조회를 한 결과들을 entity > dto 해주는 기본적인 역할을 수행한다. (마찬가지로 REST API에 대한 게 목적이 아니므로 이 부분 설명 생략한다..)

여기서 중요한 것은, 성능 측정을 위한 메서드인 `performanceTest()` 부분이다.

여기서 JPA의 영속성 컨텍스트(1차 캐시) 기능을 무효화해서 더 정확한 결과 측정을 위해 `entityManager.clear();`를 각 쿼리 실행 전에 필수로 진행하였다.

### 😯 아니 EntityManager를 두 번 선언해도 되는 거야?

여기서 의문이 들었던 부분은 이것이다.

나의 경우는 이미 QuerydslConfig에 `private final EntityManager entityManager;`가 생성되어 있는데, 여기서 한 번 더 사용한다면 뭔가 문제가 생길 것 같았기 때문!

```java
// QuerydslConfig에서
private final EntityManager entityManager;

// LogService에서
private final EntityManager entityManager;
```

결론부터 말하자면 상관없다.
왜냐면, Spring의 EntityManager는 프록시 객체이기 때문이다!

> 이 둘은 실제 찐 EntityManager가 아니고, EntityManager를 가리키는 프록시 객체이기 때문에 Thread-safe하게 관리가 되고 있고, 각 트랜잭션마다 실제 EntityManager를 바인딩해주기 때문에 각 요청마다 독립적인 EntityManager 사용이 가능하고, 이로 인해 개발자는 신경 쓸 필요 없이 주입만 받으면 독립적으로 사용이 가능한 것이다!
>

### LIKE의 차이점

그리고 여기서 또 중요한 것은 이것이다.

바로 LIKE 쿼리의 특이점 때문인데, 이 두 가지를 비교해보자.

```java
// 3. message 검색
entityManager.clear(); // 각 테스트마다 독립적 측정을 위해 clear 해줘야 한다.
startTime = System.currentTimeMillis();
List<Logs> messageLogs = logRepository.findByMessageStartingWith("가나디"); // LIKE '가나디%' 체크를 위해 StartingWith으로 한다.
long messageSearchTime = System.currentTimeMillis() - startTime;

// 3-2. message 검색 2
// 이번에는, 인덱스가 통하지 않는 contain으로 인덱스의 효과를 측정해보자.
entityManager.clear(); // 각 테스트마다 독립적 측정을 위해 clear 해줘야 한다.
startTime = System.currentTimeMillis();
List<Logs> messageContainingLogs = logRepository.findByMessageContaining("가나디"); // LIKE '%가나디%' 체크를 위해 Containing으로 한다.
long messageContainingSearchTime = System.currentTimeMillis() - startTime;

```

`findByMessageStartingWith("가나디");`의 경우는 `가나디%`로 치환되고,
`findByMessageContaining("가나디");`의 경우는 `%가나디%`로 치환된다.

앞에서 인덱스에 대한 공부를 할 때 언급했지만, 인덱스의 경우는 왼쪽을 기준으로 하기 때문에 StartingWith는 인덱스의 효과를 볼 수 있지만, Containing은 인덱스의 효과를 볼 수 없을 것이다.

즉, 이 실험에서 우리가 중요하게 살펴봐야 할 부분에 대한 "가설"은 다음과 같다.

> message 컬럼에 인덱스가 적용된 상태에서:
>
> `StartingWith` (LIKE '가나디%'): 인덱스 활용 → 빠른 조회
> `Containing` (LIKE '%가나디%'): 인덱스 미활용 → 느린 조회 (Full Scan)
>
> **따라서 두 메서드 간 성능 차이를 통해 인덱스의 효과를 정량적으로 측정할 수 있을 것이다**
>

## performance test 결과 (no index)

```json
{
    "totalRecords": 100000,
    "levelSearchTime": 691,
    "userIdSearchTime": 145,
    "messageSearchTime": 167,
    "messageContainingSearchTime": 218,
    "levelAndUserIdSearchTime": 132,
    "levelAndMessageSearchTime": 189,
    "levelSearchCount": 24975,
    "userIdSearchCount": 8,
    "levelAndUserIdSearchCount": 2,
    "testDescription": "현재 인덱스 상태"
}
```

인덱스를 생성하기 전에 postman을 통해 성능 response를 받은 결과이다.
보면 알겠지만, 십만 개라는 많지 않은 양의 데이터임에도 단일한 조건을 검색했을 때조차 상당히 오래 걸리는 것을 볼 수 있다.

## 단일 인덱스 생성해서 쿼리 속도 비교하기~

다음으로 우선, 단일 인덱스만 생성해서 성능을 비교해보도록 한다.

```sql
-- level 인덱스
CREATE INDEX idx_logs_level ON logs(level);
-- userId 인덱스
CREATE INDEX idx_logs_user_id ON logs(user_id);
-- message 인덱스
-- text나 blob의 경우는 인덱스를 생성할때 길이를 생성해야 한다.
CREATE INDEX idx_logs_message ON logs(message(255));
```

자, 단일 인덱스 생성에 성공했다. 이제 postman을 다시 실행해보자.

```json
{
    "totalRecords": 100000,
    "levelSearchTime": 374,
    "userIdSearchTime": 5,
    "messageSearchTime": 4,
    "messageContainingSearchTime": 218,
    "levelAndUserIdSearchTime": 3,
    "levelAndMessageSearchTime": 3,
    "levelSearchCount": 24975,
    "userIdSearchCount": 8,
    "levelAndUserIdSearchCount": 2,
    "testDescription": "현재 인덱스 상태"
}
```

이 어마무시한 차이가 눈에 보이는가?

이를 직접 비교해서 표로 나타내보면 다음과 같다

| 측정 항목 | 인덱스 적용 전 (ms) | 인덱스 적용 후 (ms) | 개선율 | 성능 향상 |
| --- | --- | --- | --- | --- |
| **levelSearchTime** | 691 | 374 | -45.9% | 1.85배 빨라짐 |
| **userIdSearchTime** | 145 | 5 | -96.6% | 29배 빨라짐 |
| **messageSearchTime** | 167 | 4 | -97.6% | 41.75배 빨라짐 |
| **levelAndUserIdSearchTime** | 132 | 3 | -97.7% | 44배 빨라짐 |
| **levelAndMessageSearchTime** | 189 | 3 | -98.4% | 63배 빨라짐 |

단일 인덱스 만으로 이정도 성능을 내는 것이 가능하다.

그렇다면 복합 인덱스를 생성하면 어떻게 될까?

## 복합 인덱스 생성해서 쿼리 속도 비교하기~

```sql
-- level, userId 복합 인덱스
CREATE INDEX idx_logs_level_user_id ON logs(level, user_id);
-- level, message 복합 인덱스
CREATE INDEX idx_logs_level_message ON logs(level, message(255));
```

```json
{
    "totalRecords": 100000,
    "levelSearchTime": 397,
    "userIdSearchTime": 3,
    "messageSearchTime": 2,
    "levelAndUserIdSearchTime": 5,
    "levelAndMessageSearchTime": 4,
    "levelSearchCount": 24975,
    "userIdSearchCount": 8,
    "levelAndUserIdSearchCount": 2,
    "testDescription": "현재 인덱스 상태"
}
```

| 측정 항목 | 인덱스 적용 전 (ms) | 인덱스 적용 후 (ms) | 개선율 | 성능 향상 |
| --- | --- | --- | --- | --- |
| **levelSearch** | 691 | 397 | -42.5% | 1.7배 빨라짐 |
| **userIdSearch** | 145 | 3 | -97.9% | 🔥 **48배** 빨라짐 |
| **messageSearch** | 167 | 2 | -98.8% | 🔥 **83배** 빨라짐 |
| **levelAndUserIdSearch** | 132 | 5 | -96.2% | 🔥 **26배** 빨라짐 |
| **levelAndMessageSearch** | 189 | 4 | -97.9% | 🔥 **47배** 빨라짐 |

사실 통계적으로 의미 있는 차이가 나지는 않는다. 지금은 로그 데이터가 십만개밖에 없어서…그렇다고 한다. 이미 단일 인덱스 하나 만으로 어느정도 커버링이 되는 상태라.

데이터가 수천 수만에 이를 경우, 복합 인덱스가 힘을 발휘한다고 한다!!

## 가설 검증 : messageSearchTime vs messageContainingSearchTime 비교

```json
{
    "totalRecords": 100000,
    "levelSearchTime": 397,
    "userIdSearchTime": 3,
    "messageSearchTime": 2,
    "messageContainingSearchTime": 218,
    "levelAndUserIdSearchTime": 5,
    "levelAndMessageSearchTime": 4,
    "levelSearchCount": 24975,
    "userIdSearchCount": 8,
    "levelAndUserIdSearchCount": 2,
    "testDescription": "현재 인덱스 상태"
}
```

참고로 말하자면, 이건 인덱스 생성 후의 결과임에도 `"messageContainingSearchTime": 218`의 경우, 전혀 인덱스의 효과를 못 보고 있는 것을 볼 수 있다.

즉 우리가 세웠던 가설대로 확실히 startingWith는 인덱스 효과가 있지만, containing은 인덱스 효과가 없는 것을 볼 수 있다.

### 추가) 카디널리티

![카디널리티.png](/images/2025-10-06-mysql-index-1/카디널리티.png)

참고로 인덱스에는 카디널리티라는 것이 존재한다.

카디널리티가 클수록 선택도가 높다는 것을 의미한다.

선택도가 무엇을 의미하는지는 이미 개념에서 다뤄서 알고 있다시피, 선택도가 높을수록 인덱스 하나가 유일성이 높다는 것을 의미한다.

> 그러므로 실제로 카디널리티가 높을수록, 인덱스의 효과가 크다.
>

### 추가) 실행 계획 비교

```sql
EXPLAIN SELECT * FROM logs WHERE level = 'ERROR' AND user_id = 1;
```

실행 계획을 비교해보면 실제로 복합 인덱스가 잘 쓰여지고 있는지를 알 수 있다.

실행 계획을 실행해서 살펴보면, **key가 idx_logs_level_user_id로 되어 있기 때문에**, 복합 인덱스가 잘 쓰이고 있는 것을 볼 수 있다. 그냥 진짜로 데이터 수가 작아서 유의미한 결과가 안나온 것일 뿐..

## 현재 버전을 깃허브에 tag로 기록하기!

```PowerShell
sspur@sunj-PC MINGW64 /c/developer/GitHub/study_repo/RealMySQL (main)
$ git tag v1.0-basic-index -m "기본 인덱스 최적화 : 최대 83배 성능 개선"

sspur@sunj-PC MINGW64 /c/developer/GitHub/study_repo/RealMySQL (main)
$ git show v1.0-basic-index
tag v1.0-basic-index
Tagger: sunJ0120 <sspure1214@gmail.com>
Date:   Mon Oct 6 19:58:16 2025 +0900

//......

sspur@sunj-PC MINGW64 /c/developer/GitHub/study_repo/RealMySQL (main)
$ git push origin v1.0-basic-index

```

참고로 이건 보너스인데, 인덱스의 경우는 계속 버전 업을 할 예정이라 태그를 달아두었다.

지금은 베이직 인덱스라 v1이고, 차후 커버링 인덱스 등의 심화는 v2, v3..에서 다룰 예정이다.

이렇게 태그를 달아서, 특정 시점의 마일 스톤을 저장해주면 된다.

![태그달기.png](/images/2025-10-06-mysql-index-1/태그달기.png)
