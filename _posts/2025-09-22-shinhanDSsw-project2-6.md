---
title: "🎱 챗봇 플로우 관리 전략 변경하기"
layout: post
categories: [project, spring_boot]
tags: [project, spring_boot]
toc: true
toc_sticky: true
toc_label: 목차
author_profile: true
permalink: /project/2/spring_boot/2
---
# 🖤 Intro
서버 개발을 거의 끝내고 트러블 슈팅을 하면서, 로직 흐름에서 이상한 점을 느꼈다.

답변에 대한 검증은 currentFlow에서 하는데, 응답을 생성하는 것은 한 플로우를 이동해서 nextFlow에서 하는 것이다!

사실상 nextFlow에 대한 응답을 생성해야 하는게 맞으니까 이 방식이 틀린것은 아니지만, 이렇게 할 경우 "상품 추천"을 하는 플로우에서 문제가 생긴다.

상품 추천의 경우는 nextFlow에 대한 응답을 생성하는 것이 아니라 currentFlow에 대한 응답을 생성하는 것이고, 이 다음 응답을 생성할때 nextFlow에 대한 검증을 하는 것이기 때문이다. 즉 플로우의 흐름이 꼬인다.

> 이렇게 플로우가 제각각일 경우, 플로우 관리 로직이 너무 복잡해지고 차후 새로운 플로우가 추가 되었을때도 문제가 생긴다! 그래서 이 플로우를 모두가 동일한 흐름을 가져갈 수 있도록, 혹은 묶음별로 나눠서 그 묶음이 동일한 플로우를 가질 수 있도록 분기해야 한다.

그래서 오늘은 이 부분을 완전히 해결하면서, 로직을 리팩터링 해보고자 한다.

**중요하며 놓치지 말아야 하는 점은, flow를 Spring Boot 메인 서버에서 관리한다는 기존의 기조를 어기지 않는 것이다!**

# 🩶 Start

## 현재 코드 살펴보기

기존 ChatServiceImpl을 보면, 플로우별로 처리 방식이 달랐다:

```java
// 일반 플로우는 유효성 검증 후 처리
ValidInputResponseDTO validated = validCheckService.validateInput(customerId, sessionId, message, currentFlow);
// ...다음 플로우로 전환...
BasicChatResponseDTO response = requestToLLM(sessionId, context, message, nextFlow);

// 하지만 추천 플로우는 다른 방식
BasicChatResponseDTO response = requestToLLM(sessionId, context, message, currentFlow);
if("recommendation".equals(response.getFlow()) || "re-recommendation".equals(response.getFlow())){
    response = handleRecommendationFlow(sessionId, response);
}
```

이렇게 플로우 관리가 흐름마다 다른데, 각 관리에 대한 캡슐화도 제대로 안 되어 있기 때문에 문제가 생기는 것이다.

## 리팩터링 시작

코드를 고치기 위해 다음과 같이 플로우를 세 가지 방향으로 분기하였다:

```java
public enum FlowType {
    START,      // 시작 플로우 (유효성 검증 없이 바로 처리)
    QUESTION,   // 다음 질문을 생성하는 플로우
    RESPONSE    // 현재 입력에 대한 응답을 생성하는 플로우
}
```

우리 채팅 서비스의 경우, 입력에 대한 응답을 검증하는 플로우 vs 다음 질문을 생성하는 플로우 vs 단순 전환만을 진행하는 플로우 이렇게 세 가지 종류의 플로우가 있다.

그래서 이렇게 큰 세 가지의 묶음으로 나눠주는 것이다.

### FlowConfig로 메타데이터 관리

```java
@RequiredArgsConstructor
@Getter
public class FlowConfig {
    private final FlowType processingType;     // 처리 방식 (QUESTION/RESPONSE/START)*
    private final ChatFlowType nextFlow;      // 다음 플로우
    private final boolean isTerminal;         // 마지막 플로우인지
}
```

FlowConfig를 통해 현재 플로우, 다음 플로우를 설정하는 역할을 한다.

### FlowProcessor 인터페이스

```java
public interface FlowProcessor {
    BasicChatResponseDTO process(FlowContext context);
    boolean canHandle(FlowType flowType);
}
```

좋은 설계를 위해서는 플로우 묶음별로 처리 방식을 캡슐화해두는 것이 좋다. 그러므로 캡슐화의 근본이 되는 FlowProcessor를 만들어서 검증을 시도한다.

여기서 중요한 것이, 기존의 코드에서 흩어져서 parameter로 들어가던 값들을 하나의 DTO로 합치는 것이다.

### FlowContext로 파라미터 통합

```java
@Builder
@Getter
public class FlowContext {
    private Long customerId;
    private String sessionId;
    private String message;
    private ChatFlowType currentFlow;
}
```

기존에는 유효성 검증을 할 때 `customerId, sessionId, message, currentFlow` 이것들을 각각 넘겨줬는데, 이를 하나의 DTO로 묶어서 어떤 요소들을 이용해야 유효성 검증이 가능한지를 명확히 했다.

### DTO 분리 전략

우리의 경우는 이미 FastAPI에서 유효성 검증 요청을 할 때 사용하는 DTO가 있다:

```java
@Builder
public class ValidInputRequestDTO {
    private Long customerId;
    private String sessionId;
    private String inputMessage;
    private String flow; // String 타입
}
```

그렇다면 과연 동일한 내용의 DTO를 두 개씩이나 생성할 필요가 있을까?

그러나 다음과 같은 이유로 두 가지 DTO 모두 필요하다:

- **External vs Internal**: 외부 API 통신과 내부 처리의 분리
- **String vs Enum**: 명확한 구분을 통한 타입 안전성 확보
- **책임 분리**: 각각 다른 레이어의 책임을 가짐
- **유지보수**: 외부 API 변경 시 내부 로직 영향 최소화

### UnifiedFlowProcessor로 단일화

처음에는 QuestionFlowProcessor, ResponseFlowProcessor, StartFlowProcessor로 나누려 했지만, 막상 구현해보니 차이라고 할 것이 `nextFlow.getValue()` vs `context.getCurrentFlow().getValue()` 하나뿐이었다. 불필요한 로직이 너무 많아서, 이 부분을 하나로 합치고 조건분기 하는게 낫다는 생각을 했다.

```java
@Service
@RequiredArgsConstructor
public class UnifiedFlowProcessor implements FlowProcessor {
    private final ValidCheckService validCheckService;
    private final ChatCacheService chatCacheService;
    private final FlowConfiguration flowConfiguration;
    private final LlmService llmService;

    @Override
    public BasicChatResponseDTO process(FlowContext context) {
        // 1. 현재 플로우에 대한 입력 검증
        ValidInputResponseDTO validated = validCheckService.validateInput(
                context.getCustomerId(),
                context.getSessionId(),
                context.getMessage(),
                context.getCurrentFlow().getValue());

        // 2. 검증 실패 처리
        if (!validated.isValid()) {
            return handleValidationFailure(context, validated);
        }

        // 3. 검증 성공시 입력값 캐싱
        saveFlowSpecificData(context.getSessionId(), context.getCurrentFlow(), validated);

        // 4. 다음 플로우로 전환
        FlowConfig config = flowConfiguration.getFlowConfig(context.getCurrentFlow());
        ChatFlowType nextFlow = config.getNextFlow();
        if (nextFlow != null) {
            chatCacheService.saveFlowBySession(context.getSessionId(), nextFlow.getValue());
            chatCacheService.saveCntBySession(context.getSessionId(), 1);
        }

        // 5. QUESTION vs RESPONSE에 따라서 LLM에 주입할 플로우 분기
        String targetFlow = getTargetFlow(context);
        CustomerContextWrapper contextWrapper = collectCacheData(context);

        return llmService.generateChat(
                context.getSessionId(),
                contextWrapper.getCustomerInfo(),
                contextWrapper.getMood(),
                contextWrapper.getHobby(),
                contextWrapper.getBalance(),
                context.getMessage(),
                contextWrapper.getSummary(),
                targetFlow
        );
    }

    @Override
    public boolean canHandle(FlowType flowType) {
        return FlowType.QUESTION.equals(flowType) || FlowType.RESPONSE.equals(flowType);
    }

    /**
     * FlowType.QUESTION vs FlowType.RESPONSE에 따라 targetFlow 결정
     **/
    private String getTargetFlow(FlowContext context) {
        FlowConfig config = flowConfiguration.getFlowConfig(context.getCurrentFlow());
        return config.getProcessingType() == FlowType.QUESTION
                ? config.getNextFlow().getValue()     // 다음 플로우 질문
                : context.getCurrentFlow().getValue(); // 현재 플로우 응답
    }
}
```

`getTargetFlow`를 추가해서 큰 플로우 타입에 따라 현재 플로우를 내보낼지, 다음 플로우를 내보낼지를 결정한다. 이렇게 처리하는 메서드를 둠으로써 큰 묶음별로 플로우를 선택할 수 있도록 한다.

## Factory Pattern 적용

### Factory Pattern이 필요한 이유

현재 크게는 두 묶음으로 플로우가 나눠져 있지만, 세부적으로 다양한 플로우가 존재하기에 이를 관리할 것이 필요하다. 또한 챗봇이 "자유롭다"는 특성상, 향후 새로운 타입의 플로우가 추가될 가능성이 있다.

이럴 경우 하나의 메서드를 두고 관리하고, 메인 서비스 로직에서는 해당 플로우를 모르게끔 관리하는 것이 좋다.

### FlowProcessorFactory 구현

```java
@Component
@RequiredArgsConstructor
public class FlowProcessorFactory {
    private final List<FlowProcessor> processors;

    public FlowProcessor getProcessor(FlowType flowType) {
        return processors.stream()
                .filter(processor -> processor.canHandle(flowType))
                .findFirst()
                .orElseThrow(() -> new IllegalArgumentException("processor 타입을 찾을 수 없습니다: " + flowType));
    }
}
```

스트림을 통해서 해당 플로우를 관리할 수 있는 processor를 찾는다. 이때 사용하는 것이 바로 우리가 구현했던 `canHandle`이다.

### 최종 ChatServiceImpl

```java
/**
 * 채팅 껍데기 서버 서비스 구현체
 *
 * 고객과의 실시간 채팅, LLM 응답 생성, 추천 서비스 연동을 담당
 * Redis 캐시를 통한 성능 최적화 및 외부 FastAPI 서버와의 통신 처리
 */
@Service
@Slf4j
@Transactional
@RequiredArgsConstructor
public class ChatServiceImpl implements ChatService {
    private final ChatCacheService chatCacheService;
    private final FlowProcessorFactory processorFactory;
    private final FlowConfiguration flowConfiguration;
    private final CustomerRepository customerRepository;

    /**
     * 채팅 처리
     */
    @Override
    @Transactional(readOnly = true)
    public BasicChatResponseDTO chat(Long customerId, String sessionId, String message) {
        log.info("세션 아이디와 고객 아이디 체크 sessionId: {} for customerId: {}", sessionId, customerId);

        // 1. 현재 플로우 조회
        String currentFlowStr = chatCacheService.getFlowBySession(sessionId);
        ChatFlowType currentFlow = ChatFlowType.fromValue(currentFlowStr);

        // 2. 플로우 설정 조회
        FlowConfig config = flowConfiguration.getFlowConfig(currentFlow);
        if (config == null) {
            throw new IllegalStateException("플로우 설정을 찾을 수 없습니다: " + currentFlow);
        }

        // 3. 적절한 프로세서 선택 - 팩토리 패턴
        FlowProcessor processor = processorFactory.getProcessor(config.getProcessingType());

        // 4. 플로우를 주입할 컨텍스트 생성
        FlowContext context = FlowContext.builder()
                .customerId(customerId)
                .sessionId(sessionId)
                .message(message)
                .currentFlow(currentFlow)
                .build();

        BasicChatResponseDTO response = processor.process(context);

        // 프로세스 후에 할 일을 하기
        processAfterResponse(response, customerId, sessionId);

        // 추천일 경우. 상품 가져오는 로직을 추가
        if("recommendation".equals(response.getFlow()) || "re-recommendation".equals(response.getFlow())){
            response = handleRecommendationFlow(response, sessionId);
        }

        return response;
    }

    /**
     * 응답 후처리 (DB 업데이트, 캐싱)
     */
    private void processAfterResponse(BasicChatResponseDTO response, Long customerId, String sessionId) {
        // 취미의 경우는 db에 저장하는 값으로, 차후 llm이 한 번 처리한 다음 저장할 수 있도록 리팩터링 하기 위해 요청 후 처리하도록 하였다.
        if (!response.getNewHobby().equals("")) {
            customerRepository.updateHobbyByCustomerId(customerId, response.getNewHobby());
            chatCacheService.saveHobby(customerId, response.getNewHobby()); //취미 저장
            log.info("취미 DB 업데이트 완료 - customerId: {}, hobby: {}", customerId, response.getNewHobby());
        }
        // 캐싱
        // 요약본 저장
        if (response.getSummary() != null && !response.getSummary().isEmpty()) {
            chatCacheService.saveSummaryBySession(sessionId, response.getSummary());
            // TODO : 요약본 DB에 저장하는 로직 필요 (우선순위 낮음)
        }
    }

    /**
     * 추천 플로우 처리 (상품 캐싱 및 응답 재생성)
     */
    private BasicChatResponseDTO handleRecommendationFlow(BasicChatResponseDTO response, String sessionId) {
        if ("recommendation".equals(response.getFlow())) {
            // 새 상품 캐싱
            chatCacheService.saveProductsBySession(sessionId, response.getProducts());
        }

        return BasicChatResponseDTO.builder()
                .sessionId(response.getSessionId())
                .message(response.getMessage())
                .newHobby(response.getNewHobby())
                .products(chatCacheService.getProductsBySession(sessionId)) // 3개 제한
                .summary(response.getSummary())
                .flow(response.getFlow())
                .build();
    }
}
```

현재 프로세스에서 FlowConfig를 가져와서 처리 방식을 가져와서, 팩토리 메서드가 적절하게 처리할 수 있도록 한다. FlowContext의 경우, 플로우에 따른 검증 처리, LLM 처리를 위해 필요하다. 이를 currentFlow 기준으로 생성해서 processor의 process를 진행한 다음, 후처리 후에 응답을 생성한다.

## 리팩터링 결과 정리

이러한 리팩터링을 통해~

- 기존 200줄의 복잡한 분기 로직이 50줄의 깔끔한 위임 패턴으로 변경
- 플로우별 일관된 처리 방식 확립
- 새로운 플로우 추가 시 기존 코드 수정 없이 프로세서만 추가하면 됨
- Spring Boot 서버에서 플로우를 중앙 관리한다는 기본 방침 유지

복잡한 플로우 관리를 체계적이고 확장 가능한 구조로 개선할 수 있었다.

물론 어느정도 AI의 도움을 받기는 했지만! 참 나는 코드 리팩터링 하면서 이렇게 코드가 이쁘게 변할때가 제일 기분이 좋다.

좀 더 경험을 많이 쌓아서 온전히 내 힘만으로 예쁜 코드를 짤 수 있도록 해야징!
