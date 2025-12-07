---
title: "🎱 생애 첫 투 서버 도전기, 서버 두 개를 클린~하게 운영해보아용 - [1탄] 아키텍처 구조를 왜 이렇게 잡았어?"
layout: post
categories: [project, development]
tags: [project, development, architecture]
toc: true
toc_sticky: true
toc_label: 목차
author_profile: true
permalink: /project/2/architecture/1
---

# 🖤 Intro

이번에 처음으로 두 개의 서버를 연동해서 백엔드 개발을 진행하게 되었다.

이유는 챗봇을 구현할때 LLM을 사용하기 위해 파이썬 서버를 붙여야 했기 때무넹

사실 SPRING AI를 사용하면 자바로도 AI 개발이 가능하다고는 하지만.....

솔직히 아직 AI 파트에서는 파이썬을 따라가기엔 멀었다는게 내 생각이고, 차후 모델 학습 등을 고려한 확장 방향을 고려하면, 어렵더라도 이 방향으로 가는게 맞지 않을까….싶어서 이런 선택을 하게 되었다.

그리고 이 참에 이 구조에 대한 틀을 잘 잡아두면, 내가 너무너무 해보고 싶었던 MSA 설계할 때 정말 큰 도움이 될 것 같아서!!! 도전해보기로 하였다:)

# 🩶 Start

## FAST API 단일 ? VS BOOT & FAST API?

처음 서버 구조를 잡을때, 두 가지 의견 사이에서 나와 데이터팀 동생이 고민을 굉장히 많이 했다.

내 입장에서는 사실 후자로 진행하고 싶었던 가장 큰 이유가, 내가 SPRING 백엔드 개발자로 취업을 원하기도 하고, 파이썬으로 서버 개발을 한 적도 없고 애초에 선호하지 않기 때문이었는데…

그와중에 문득 궁금해졌다.

> 😲 그럼, 과연 FAST API 단일 서버로 구현하는 것에 비해 BOOT & FAST API 투 서버로 챗봇을 구현하는건 어떤 장점이 있을까? 취업이라는 이유를 제외하고, 왜 우리가 이 선택을 하는 것이 좋은걸까?

### 1. 관심사의 분리

나의 경우는 애초에 투 서버를 진행하기로 하면서, 다음과 같은 규칙을 세웠었다.

- SpringBoot만 DB에 접근하고, Front와 통신한다.
- Fast api는 llm과 연동, 외부 api와 연동하는 역할을 맡고, db 접근과 front 통신 등의 업무를 일절 하지 않는다.

이렇게 해서 양쪽이 전부 DB에 접근해서 트랜잭션이 꼬이는 경우를 방지하고(이를 통해 데이터 정합성을 보장한다.), Front 단에서도 어떤 서버에 request를 날려야 하는지를 명확히 할 수 있도록 구성했다.

그리고 사실상 역할분리를 도메인 기준으로 명확하게 분리한 것은,,,차후 MSA라는 목표를 위해서!!!이기도 하다.

### 2. 기술 스택 최적화

알다시피. Spring boot는 웹 서버 개발에 최적화 되어 있다. 특히 라이브러리가 풍부하고, 뭣보다 내가 JPA랑 Querydsl, 그리고 자바의 객체 지향적인 구조 이 삼합을 정말 사랑!하기 때문에…서버 개발은 자신이 있었다.

그리고 fast api는 python 생태계 특성상 ai에 정말 특화되어 있다. llm 연동 뿐 아니라 차후 정밀성을 위해 model 학습을 붙일때도 scikit-learn, Tensorflow 등을 사용하기에도 매우 좋다. (즉, 미래적 관점에서 확장성이 진짜 좋다.)

spring ai라는게 최근에 생긴건 알고 있지만, 해당 영역에 생태계가 풍부한 언어를 활용하는게 맞는 방향이라고 생각해서 이러한 결정을 내렸다.

## 그렇다면 아키텍처 구조는 어떻게 구성해야해?

자 여기서 문제는 이 부분이다.

나의 경우는, 최대한 SRP 원칙을 지키기 위해 서버당 역할 분리를 명확히 해둘 예정이었다.

그렇기에 아키텍처를 명확하게 짜두지 않으면, 개발할때 매우 많은 부분이 꼬일 것 같았다.

그래서 몇 번(사실 한 다섯 번 정도..) 갈아엎은 끝에, 다음과 같이 챗봇 아키텍처를 구성하였다.

![image.png](/images/2025-09-13-shinhanDSsw-project2-1/1.png)

현재는 로컬 개발이라 REDIS를 DOCKER에 띄웠는데, EC2에 배포한 후에는 원래 ElasticCache로 변경할 예정이었다.

근데 사실….우리 프로젝트를 강사님 서버에 띄워야 할 수도 있어서, 도커 컨테이너링 한 지금의 구조를 유지해야 할 수도 있겠다. 이건 CI/CD 할 때 가봐야 알듯 싶다.

구조의 포인트를 간단히 설명하자면 다음과 같다.

1. 최대한 3-tier로 명확히 분리되도록 구성하였다.

2. 위에서 말했던 대로, 서버당 역할을 명확히 분리하였다.

특히 몇 번 리팩터링 끝에 redis에 대한 서버의 역할도 분리를 한 상태인데, boot의 경우 redis에서 조회 & 없을시 read-through 패턴으로 db에서 꺼내와서 캐시에 저장하는 것 까지 담당하고 (즉, 사실상 조회만 담당하는 셈이다)
fast api의 경우는 redis에 저장하는 역할을 담당한다. (기분 데이터, 채팅 데이터 등…)

이렇게 redis에 대한 역할까지 분리한 이유는 다음과 같다.

### 왜 REDIS에 대한 역할도 분리했을까?
#### 1. DB 접근은 boot만 가능하다.

나의 경우는 read-through 패턴을 사용했기에, redis에 캐싱 값이 없을 경우, db를 조회하는 로직을 실행해서 db에서 값을 가져와서 redis에 캐싱하고, 값을 가져온다.

이 부분 적용 로직은 다음과 같다

`ChatCacheService` 일부

```java
// 고객 기본 정보 조회 (Read-through)
public CustomerCacheDTO getCustomerInfo(Long customerId) {
    return cacheStore.getOrLoad(
            CacheSpec.CUSTOMER,
            customerId,
            CustomerCacheDTO.class,
            () -> customerRepository.findCustomerCacheInfoById(customerId)
    );
}
```

`CacheStore` 일부

```java
// Read-Through: 캐시 미스 시 로더 실행 → 캐시에 저장 후 반환하도록 설정한다.
public <T> T getOrLoad(CacheSpec spec, Object userId, Class<T> type, Supplier<T> loader) {
    // 1) 캐시 먼저 조회
    T cached = this.get(spec, userId, type);
    if (cached != null) {
        return cached;
    }

    // 2) 미스면 "로더" 실행 (여기서 DB 접근이 수행됨)
    // 여기서 Supplier<T>를 사용해서 조회 로직을 숨길 수 있다. 이를 통해 조회 로직이 서비스 단으로 새어나가지 않도록 한다.
    T loaded = loader.get();

    // 3) 값이 있으면 캐시에 넣고 반환 (키/TTL은 spec에서 가져옴)
    if (loaded != null) {
        this.put(spec, userId, loaded, spec.getTtl());
    }
    return loaded;
}
```

즉, 조회를 하려면 기본적으로 db에 접근하는 로직이 들어간다.

그러므로 모든 조회는 boot가 하고, 대신에 이 조회한 값들을 속 서버인 fast api에 전달할때 request로 담도록 한다.

무엇보다. spring에는 HikariCP 등 검증된 라이브러리가 존재하기 때문에 DB 커넥션 풀 관리를 효율적으로 할 수 있다.

#### 2. llm 값들은 fast api가 받는다.

기본적으로 llm응답값은 fast api가 먼저 response로 받고, 그걸 redis에 캐싱하고, 캐싱 후에 spring boot에 보내야 한다.

즉, fast api에서 캐싱하는게 더 빠르다.

또한, 나의 경우는 메세지를 가져오는 방식이 token수에 따라 커팅하는 방식을 사용하는데, (실제 많은 챗봇들이 이러한 방식으로 구현되어 있다고 한다.)

token 전략의 경우, python에서 tiktoken을 이용하면 편한 관리가 가능하기 때문에(심지어 이건 우리가 사용하는 OpenAI에서 공식적으로 제공하는 라이브러리이다…굳) fast api에 위임하고자 했다. 즉, message관리를 전부 python에 위임한다는 것인데

그렇게 할꺼라면 db 저장이 필요없는 값들은 전부 fast api에서 담당하는게 맞다고 생각했기에, “저장”단을 fast api에 위임했다.

`class RedisService` 일부

```python
def save_user_mood(self, session_id: str, mood: str) -> bool:
    """사용자 mood 저장"""
    try:
        key = f"user_mood:{session_id}"
        mood_data = {
            "mood": mood,
            "timestamp": datetime.now().isoformat()
        }
        
        self.redis_client.setex(
            key, 
            timedelta(hours=24), 
            json.dumps(mood_data, ensure_ascii=False)
        )
        logger.info(f"Mood 저장 성공: session_id={session_id}, mood={mood}")
        return True
        
    except Exception as e:
        logger.error(f"Mood 저장 실패: session_id={session_id}, error={e}")
        return False
```
## REDIS를 도입한 이유?

이것도 질문이 들어왔어서 정리하고자 한다.

REDIS는 사실, 우리가 좀 더 채팅 플로우를 우리가 원하는 방향으로 조작하기 쉽게 하기 위해 도입했다.

사실상 우리가 추천에 필요한 모든 정보 (사용자 정보, 한도, 기분..)가 전부 DB에 다 들어가 있는 것도 아니고, 이 컨텍스트가 있어야 더 정밀한 개인화 추천이 가능하기 때문이다.

또한, 서버가 달라서 세션 스토리지에 값들을 저장해서 양쪽에서 사용하게 할 수 없으며, 한도나 개인 정보 같은 민감 정보를 세션 스토리지에 넣기에는 보안적 부담감이 너무 컸다.

그리고 무엇보다 우리의 경우는 벡터DB를 사용하지 않기 때문에 DB에 채팅 요약본만 저장해두는 상태인데, 요약본 만으로는 우리가 원하는 플로우로 정밀하게 챗봇을 조정할 수 없다고 생각했다.

그래서 오고간 message를 저장할 캐싱 db가 필요했고, 모든 니즈가 잘 맞아 떨어져서 redis를 채택하게 되었다.

얘기하던 중 다음과 같은 의견도 나왔어서 정리하고자 한다.

### 그럼 파이썬 서버에 그냥 list 선언해서 message들 전부 안에 넣고, 그걸로 플로우 조작하는 방식으로 간단하게 풀어내면 안되나?

사실 내가 좋아하지 않는 방식이라 고려하지 않은 것도 맞지만, Redis에 대비해서 이유를 정확하게 정의하고 가는게 좋을 것 같아서 정리하고자 한다.

우선 이렇게 하지 않은 가장 큰 이유는, 이렇게 할 경우 차후 **“채팅 데이터를 저장해야 하는 방향”으로 리팩터링**을 하거나 서비스 확장을 기획할 경우, 고칠 부분이 너무 많고 확장을 할 수 없게 되어 버리기 때문이다.

이는 객체 지향 소프트웨어 원칙중 하나인 확장에는 열려있고 수정에는 닫혀있어야 한다는 OCP 원칙에도 위배된다. 물론 파이썬은 객체지향 언어가 아니다만…..난 자바 개발자니깐

또한 결정적으로 다음의 두 가지 문제가 존재한다.

#### 1. 메모리 누수의 위험이 있고 서버 안정성이 떨어진다.

챗봇의 특성상 데이터가 계속 쌓인다.

또한 챗봇의 특성상, 컨텍스트 데이터의 복잡성과 용량 때문에 메모리 부하가 일어나기 쉽고, 가비지 컬렉션에 부하가 발생할 수 있다.

뭐 우리야 한 달 프로젝트여서 이거까지 고려해야 하나 싶을 수 있지만, 프로젝트로 끝날게 아니고 우리는 우리가 “현업 서비스”에서 어떻게 할지를 고려하는게 맞다고 나는 생각하기 때문에 (취업할꺼니까) 이 부분을 고려하는게 맞다고 생각한다.
### 2. List는 Thread-Safe 하지 않기 때문에 동시성 문제가 존재하고, 이로 인해 여러 스레드가 동시에 접근하면 충돌이 발생할 수 있다.

챗봇에서는 대화 순서가 보장되는 것이 굉장히 중요하기 때문에, 이 순서가 바뀌면 맥락 자체가 꼬여서 우리가 원하는 방향으로 흐름을 제어하기가 힘들 수 있다.

또한, 사용자간 데이터가 혼재될 수 있다는 명확한 단점이 존재한다. 예를 들어보자.

```python
# 동시성 문제로 발생할 수 있는 시나리오
# 사용자 A: "내 알레르기 정보: 견과류"
# 사용자 B: "맛집 추천해줘"

# Thread-unsafe 상황에서:
# 사용자 B가 사용자 A의 알레르기 정보로 추천받을 수 있음! 🚨
# → 심각한 보안/프라이버시 문제
```

```python
# 실제로 일어날 수 있는 시나리오
import threading
import time

chat_data = []  # 전역 리스트

def process_user_a():
    chat_data.append({
        "user": "A", 
        "type": "allergy_info",
        "content": "견과류 알레르기 있음"
    })
    time.sleep(0.1)  # 네트워크 지연 시뮬레이션

def process_user_b():
    time.sleep(0.05)  # 약간의 지연
    # 사용자 B가 추천 요청
    chat_data.append({
        "user": "B",
        "type": "recommendation_request", 
        "content": "맛집 추천해줘"
    })
    
    # LLM에게 컨텍스트 전달 시
    recent_context = chat_data[-2:]  # 최근 2개 메시지
    # → 사용자 A의 알레르기 정보가 포함됨! 🚨

# 동시 실행
threading.Thread(target=process_user_a).start()
threading.Thread(target=process_user_b).start()
```

결과적으로....

```python
User B: "맛집 추천해줘"
LLM Context: ["견과류 알레르기 있음", "맛집 추천해줘"]
LLM Response: "견과류를 피한 안전한 레스토랑을 추천드릴게요..."

User B: "??? 나 알레르기 없는데?" 🤔
```

알레르기 뿐 아니라, 개인적보가 섞일 경우엔 개인정보 노출로 보호법 위반의 위험성도 존재한다고 한다.

예시는 다음과 같다.

```python
# 사용자 A: "우리 회사 근처 (강남역)"  
# 사용자 B: "점심 추천해줘"

# → 사용자 B가 A의 위치 정보로 추천받음
# → 개인정보보호법 위반 가능성
```

Redis의 경우는 기본적으로 Thread-safe한 환경이라 이를 방지할 수 있다.
