---
title: "🎱 [신한DS SW 프로젝트 2-1] API 짜는법 공부 - 스택별로 개발하는 법~"
layout: post
categories: [project, development]
tags: [project, development, RESTful, API]
toc: true
toc_sticky: true
toc_label: 목차
author_profile: true
permalink: /project/2/restful
---

# 🖤 Intro

사실 이번 프로젝트 하면서 가장 생각하기 어려웠던 부분이 바로, 개발 순서를 잡는 일이었다.

이렇게 중간중간 로직이 들어가는 서버 개발은 처음이라 (애초에 경험이 많이 없어서리….) 뭘 먼저 개발해야 할지 아니면 진짜 플로우 순서로 가야할지 너무너무 헷갈리는 것이다!!!!!

그래서 오늘은 이런 경우, 개발을 어떻게 해야 할지를 좀 정리해보고자 한다.

# 🩶 Start

## 현재 내가 개발해야 하는 챗봇의 구조는?

![image.png](/images/2025-09-08-shinhanDSsw-project2-1/3.png)

다음과 같이 서버가 두 대가 적절하게 데이터를 교환해야 하고, db도 메모리 db까지 총 두 대를 사용하기 때문에 상당히 구조가 복잡하다. 그래서 데이터를 교환하는게 중간중간 섞여있어, “껍데기”를 먼저 개발하고, 채워 나가는 방식을 사용하는 것이 좋다.

### 그래서 전략을 어떻게?

원래 생각했던건 flow 별로 순서대로 진행해야 하나 고민했지만, 이렇게 할 경우, 다음과 같은 단점이 있다.

우선, 앞 단계가 완성되지 않으면 뒷 단계 테스트가 불가능해져서 개발이 매우 느려진다. 특히 나의 경우는  sprint boot 위주로 공부를 해왔기 때문에, 생소한 fast api나 redis와 같은 기술을 사용하다가 막힐 가능성이 있다…. 이는 애자일 개발 방식에도 맞지 않다.

또한 각 서버 간 데이터 교환이나 DB 연동을 마지막에야 확인할 수 있기 때문에, 통합 테스트가 지연된다. 예상치 못한 에러가 번질 가능성도 있다.

또한, 나중에 구조를 바꿔야 할 때, 이미 완성된 부분들을 모두 수정해야 한다는 위험성이 존재한다.

그러므로 나의 경우는, 우선 껍데기에 해당하는 boot를 mock data로 개발하고, 그 다음에 fast api, 마지막에 사용자 경험 향상을 위한 redis 사용을 붙이도록 하자.

## API 개발 시작~

### BASE DTO

```java
/*
기본적으로 request에 항상 들어가는 정보를 담은 dto
 */
@Getter
@ToString
@SuperBuilder
@NoArgsConstructor
public abstract class BaseChatRequest {
    private Long customerId;
}
```

```java
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.ToString;
import lombok.experimental.SuperBuilder;

@Getter
@ToString
@SuperBuilder
@NoArgsConstructor
public abstract class BaseChatResponse {
    private String message;
    private String step;
}
```

나의 경우는 API 명세를 보면, 기본적으로 반복되는 부분이 있다.

이런 반복되는 부분은 이렇게 base dto로 묶어 주면 유지보수가 편하다.

base dto는 직접 사용하는 dto가 아닌, 공통 필드를 묶는 껍데기의 역할만 하기 때문에 실제 사용을 막기 위해 추상 클래스로 정의한다.

또한, 상속 구조를 사용하면서 & Builder 패턴을 같이 사용하기 때문에 @SuperBuilder를 사용해서 부모 클래스의 필드도 빌더에 포함되게끔 한다.

또한, request <> response간에 Jackson 역직렬화를 위해서 기본 생성자를 정의해야 한다. 그래서 @NoArgsConstructor를 달아준다.

### StartDto

```java
@Getter
@ToString(callSuper = true)
@SuperBuilder
@NoArgsConstructor
public class ChatStartRequest extends BaseChatRequest {
}
```

```java
@Getter
@ToString(callSuper = true)
@SuperBuilder
@NoArgsConstructor
public class ChatStartResponse extends BaseChatResponse {
}
```

채팅 시작에 사용되는 기본 api request, response 전용 dto이다.

마찬가지로, 상속을 이용하기 때문에 @SuperBuilder, @NoArgsConstructor를 달아준다.

### Service

```java
import com.ohgoodteam.ohgoodpay.recommend.dto.ChatCheckHobbyResponse;
import com.ohgoodteam.ohgoodpay.recommend.dto.ChatMoodResponse;
import com.ohgoodteam.ohgoodpay.recommend.dto.ChatStartResponse;
import com.ohgoodteam.ohgoodpay.recommend.dto.ChatUpdateHobbyResponse;

public interface ChatService {
    ChatStartResponse startChat(Long customerId);
}
```

```java
@Service
@Slf4j
@Transactional
@RequiredArgsConstructor
public class ChatServiceImpl implements ChatService {
    private final CustomerRepository customerRepository;

    /**
     * 고객 ID 유효성 검증
     *
     * 채팅에서 이를 반복적으로 사용하기 때문에 따로 별도의 메서드로 분리하였다.
     */
    private void validateCustomerId(Long customerId) {
        if (customerId == null || customerId <= 0) {
            throw new IllegalArgumentException("유효하지 않은 고객 ID입니다");
        }
    }

    /**
     * 고객명 조회 및 유효성 검증
     *
     * 채팅에서 이를 반복적으로 사용하기 때문에 따로 별도의 메서드로 분리하였다.
     */
    private String getValidCustomerName(Long customerId) {
        String name = customerRepository.findByCustomerId(customerId)
                .map(CustomerEntity::getName)
                .orElse(null);
        
        if (name == null || name.trim().isEmpty()) {
            throw new IllegalArgumentException("존재하지 않는 고객입니다");
        }
        
        return name;
    }

    /**
     * 채팅 시작 처리
     * 고객 유효성 검증 → 고객명 조회 → 인사 메시지 생성
     */
    @Override
    @Transactional(readOnly = true)
    public ChatStartResponse startChat(Long customerId) {
        validateCustomerId(customerId);
        String name = getValidCustomerName(customerId);

        // TODO: Redis 캐싱 구현 (CustomerCacheDto 사용)
        
        // TODO: FastAPI 연동 구현
        String greetingMessage = String.format("안녕 나는 너만의 오레이봇봇 ~ 나를 레이라고 불러줘 %s~ 오늘 기분은 어때?", name);

        return ChatStartResponse.builder()
                .message(greetingMessage)
                .step("mood_input")
                .build();
    }
}
```

Service의 경우, 인터페이스 - 구현체 패턴을 구현해서 확장성 및 관리가 쉽도록 구성했다.

특히 여기서 유효성 검사 로직들은 따로 인터페이스에 적용하지 않고, 우리가 api 단에서 사용하는 로직만 인터페이스 - 구현체 패턴을 사용해서 구분지었다.

특히, 고객명 조회나 검증은 여러 Service에서 반복적으로 사용 되므로, 따로 유효성 검증 method로 빼주었다.

현재는 fast api를 처리하기 전이라, 이 부분과 redis 부분은 따로 자리를 빼두고 하드 코딩으로 메세지를 구성해서 넣어주었다.

ChatStartResponse.builder를 통해 값을 넣어서 response DTO의 data : 란에 값이 들어가도록 했다. 이는 Controller 단에서 내가 구현한 ApiWrapper에서 이를 어떻게 사용하는지를 보며 확인할 예정이다.

Service는 기본적으로 @Transactional을 맨 위에 달고, update, delete를 제외한 모든 것에 @Transactional(readOnly = true)를 달아주는게 가장 좋으므로, 해당 패턴으로 구현하였다.

### Controller

```java
/**
 * 추천 채팅 API
 * v1: Mock 데이터로 구현, 향후 Redis + FastAPI 연동 예정
 */
@Tag(name = "추천 채팅", description = "AI 추천 챗봇 관련 API")
@RestController
@RequiredArgsConstructor
@RequestMapping("/api/chat")
public class ChatController {
    private final ChatService chatService;

    //채팅 시작 api
    @Operation(summary = "채팅 시작", description = "고객 ID로 채팅 시작 후 개인화된 인사 메시지 반환")
    @ApiResponses(value = {
            @ApiResponse(responseCode = "200", description = "채팅 시작 성공"),
            @ApiResponse(responseCode = "400", description = "잘못된 고객 ID"),
            @ApiResponse(responseCode = "500", description = "서버 내부 오류")
    })
    @PostMapping("/start")
    public ApiResponseWrapper<ChatStartResponse> startChat(
            @Parameter(description = "채팅 시작 요청 (고객 ID 포함)")
            @RequestBody ChatStartRequest request
    ) {
        try {
            ChatStartResponse resp = chatService.startChat(request.getCustomerId());
            return ApiResponseWrapper.ok(resp);
        } catch (IllegalArgumentException e) {
            return ApiResponseWrapper.error(400, e.getMessage());
        } catch (Exception e) {
            return ApiResponseWrapper.error(500, "서버 내부 오류가 발생했습니다");
        }
    }
}
```

swagger로 test하기 위해 관련 어노테이션을 달아주었다.

Controller에서는 오직 view 단에 결과를 뿌리기 위한 역할만 부여해서, 최대한 구현 로직을 Service 단에서 처리하도록 역할을 분리했다.

return의 경우, 커스텀 ApiResponseWrapper로 감싸서 response json 형식이 일정하도록 구성했는데, 이를 살펴보자.

### ApiResponseWrapper

```java
@Getter
@AllArgsConstructor
@NoArgsConstructor
@Builder
public class ApiResponseWrapper<T> {
    private boolean success;
    private int code;
    private String message;
    private T data;

    public static <T> ApiResponseWrapper<T> ok(T data) {
        return ApiResponseWrapper.<T>builder()
                .success(true)
                .code(200)
                .message("OK")
                .data(data)
                .build();
    }

    public static <T> ApiResponseWrapper<T> error(int code, String message) {
        return ApiResponseWrapper.<T>builder()
                .success(false)
                .code(code)
                .message(message)
                .build();
    }
}
```

다음과 같이 공통으로 사용할 Api응답 Wrapper를 구성하였다.

이를 통해, Service에서 반환한 DTO가 data 필드에 들어가도록 구성했고, 이 또한 Builder 패턴으로 구현했다.

### 스웨거로 확인~

![image.png](/images/2025-09-08-shinhanDSsw-project2-1/1.png)

![image.png](/images/2025-09-08-shinhanDSsw-project2-1/2.png)
