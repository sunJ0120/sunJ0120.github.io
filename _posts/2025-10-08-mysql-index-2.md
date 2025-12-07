---
title: "🎱 MySQL 인덱스 실습 - 전문 인덱스로 B-tree 인덱스에서 커버 못하는 부분 보완하기"
layout: post
categories: [MySQL, RDBMS]
tags: [MySQL, index]
toc: true
toc_sticky: true
toc_label: 목차
author_profile: true
permalink: /MySQL/index/2
---

# 🖤 Intro
저번 포스트에서는 B-Tree 인덱스를 직접 생성해서 어느정도의 성능 차이를 만들어 낼 수 있는지를 직접 살펴보았다.
이번 시간에는 전문 인덱스 (Full Text Index)를 직접 생성하고, 어느정도의 성능 차이가 나는지를 직접 살펴보도록 하자.

# 🩶 Start
## 전문 인덱스 실습을 위한 인덱스 생성

```sql
ALTER TABLE logs ADD FULLTEXT INDEX ft_idx_message (message) WITH PARSER ngram;
```

전문 검색을 위해서는 “어떤 컬럼”에 FULLTEXT INDEX를 생성할지를 인덱스 생성해야 검색이 가능하다.

참고로 알고리즘은 n-gram을 이용하도록 하자.

![전문인덱스.png](/images/2025-10-08-mysql-index-2/전문인덱스.png)

### DTO

```java
/**
 * full index test를 위한 응답을 담은 간단한 dto
 */
@Getter
@Builder
@AllArgsConstructor
@NoArgsConstructor
public class FullIndexPerfomanceTestResponse {
    // 검색 소요 시간
    private Long likeSearchTime;           // LIKE '%keyword%' 검색 시간
    private Long fullTextSearchTime;       // MATCH AGAINST 검색 시간

    // 검색 결과
    private Integer likeResultCount;       // LIKE 검색 결과 개수
    private Integer fullTextResultCount;   // 전문 검색 결과 개수

    // 성능 비교
    private Double performanceImprovement; // 성능 향상 비율 (LIKE 시간 / FULL TEXT 시간)

    // 테스트 정보
    private String testDescription;        // "인덱스 전" or "인덱스 후"
    private String keyword;                // 검색어
}
```

결국 전문검색을 하는 목적은,  LIKE로 전체 테이블을 FULL SCAN 하는 것과 어느정도의 차이가 나는지를 비교하기 위함이다.

이를 위해 다음과 같이 필드 값들을 생성해주었다.

### Repository

```java
// 전문 검색
// mysql 전용 함수라 querydsl로는 구현이 어려워서, 네이티브 쿼리로 작성
@Query(value = "SELECT * FROM logs " +
        "WHERE MATCH(message) AGAINST(:keyword IN BOOLEAN MODE)",
        nativeQuery = true)
List<Logs> searchByFullText(@Param("keyword") String keyword);
```

mysql 전용 함수는 기본 querydsl 문법으로는 생성이 불가능하여, 네이티브 쿼리로 만들어주었다.

### Service

```java
/**
 * 전문 인덱스의 정확한 성능을 검수하기 위한 method
 */
public FullIndexPerfomanceTestResponse fullIndexPerfomanceTest(String keyword){
    long startTime; //성능 측정을 위한 시작 시간 명시

    entityManager.clear(); // 각 테스트마다 독립적 측정을 위해 clear 해줘야 한다.
    startTime = System.currentTimeMillis();
    List<Logs> likeSearchLogs = logRepository.findByMessageContaining(keyword); // LIKE '%keyword%' 체크를 위해 Containing으로 한다.
    long likeSearchTime = System.currentTimeMillis() - startTime;

    entityManager.clear(); // 각 테스트마다 독립적 측정을 위해 clear 해줘야 한다.
    startTime = System.currentTimeMillis();
    List<Logs> fullTextLogs = logRepository.searchByFullText(keyword);
    long fullTextSearchTime = System.currentTimeMillis() - startTime;

    // (이전 시간 - 현재 시간) / 이전 시간 * 100
    double improvement = ((double)(likeSearchTime - fullTextSearchTime)
            / likeSearchTime) * 100;

    return FullIndexPerfomanceTestResponse.builder()
            .keyword(keyword)
            .likeSearchTime(likeSearchTime)
            .likeResultCount(likeSearchLogs.size())
            .fullTextSearchTime(fullTextSearchTime)
            .fullTextResultCount(fullTextLogs.size())
            .performanceImprovement(improvement)
            .testDescription("현재 전문 인덱스 상태")
            .build();
}
```

서비스 구성은 이전에 b-tree 성능 측정 했던것과 비슷하다.

마찬가지로, rest api 설명하는 단계가 아니므로 자세한 설명은 생략한다…

## 결과 분석

```
http://localhost:8085/api/logs/performance/full?keyword=894
```

```json
{
    "likeSearchTime": 187,
    "fullTextSearchTime": 40,
    "likeResultCount": 300,
    "fullTextResultCount": 300,
    "performanceImprovement": 78.6096256684492,
    "testDescription": "현재 전문 인덱스 상태",
    "keyword": "894"
}
```

894를 키워드로 검색한 결과를 postman으로 뽑아낸 것이다.

likeSearchTime, 즉 LIKE 쿼리를 활용하는 Contains에 비해 full scan이 무려 **78%(4.7배)나** 빠르다!!

187ms > 40ms 의 성능 차이로, 데이터가 더 많아지고, 키워드가 더 늘어난다면 더 큰 격차가 보일 것이다.

과연 왜 인덱스를 쓰는지 알겠달까…..

```
http://localhost:8085/api/logs/performance/full?keyword=로그 메세지
```

```json
{
    "likeSearchTime": 2414,
    "fullTextSearchTime": 11985,
    "likeResultCount": 100000,
    "fullTextResultCount": 100000,
    "performanceImprovement": -396.4788732394366,
    "testDescription": "현재 전문 인덱스 상태",
    "keyword": "로그 메세지"
}
```

자, 여기서 특이한 점이 있다.

지금 전문 인덱스를 사용하는 message 컬럼의 경우, 모든 10만개의 컬럼이 전부 저 “로그 메세지”라는 키워드를 사용하고 있는데, 이렇게 **너무 많은 결과를 가지고 있을 경우 인덱스를 사용하는 것이 300%** 나….더 느리다.

앞에서 인덱스는 생성 비용이 크기 때문에, 검색 결과가 전체의 25%가 넘어갈 경우 차라리 full table scan이 더 낫다고 말했던 것을 기억하는가?
그게 바로 이러한 이유 때문이다.

> 🌟 그래서 인덱스를 사용할때는, 인덱스가 과연 **결과를 효율적으로 도출할 수 있는가?**를 잘 생각해야 한다!!
