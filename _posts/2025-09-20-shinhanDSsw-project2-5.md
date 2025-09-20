---
title: "🎱 챗봇 메인 서버에 대해 알아보자 ~ 1탄"
layout: post
categories: [project, spring_boot]
tags: [project, spring_boot]
toc: true
toc_sticky: true
toc_label: 목차
author_profile: true
permalink: /project/2/spring_boot/1
---

# 🖤 Intro
저번 아키텍처 시리즈에 이은 서버 뜯어보기 1탄이다.

개발을 진행하면서 여러 부분들을 고민하고 서버를 설계했는데, 다시 살펴보면서 에러가 있으면 고치기도 해보고....이참에 한 번 쭉 정리하려고 글을 쓰게 되었다.

# 🩶 Start
## Controller

```java
import com.ohgoodteam.ohgoodpay.recommend.dto.ChatStartRequestDTO;
import com.ohgoodteam.ohgoodpay.recommend.dto.datadto.llmdto.BasicChatResponseDTO;
import com.ohgoodteam.ohgoodpay.recommend.service.ChatService;
import com.ohgoodteam.ohgoodpay.recommend.util.ApiResponseWrapper;
import io.swagger.v3.oas.annotations.tags.Tag;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * AI 추천 채팅 컨트롤러
 */
@Tag(name = "Chat")
@RestController
@Slf4j
@RequiredArgsConstructor
@RequestMapping("/api/chat")
public class ChatController {
    private final ChatService chatService;

    @PostMapping()
    public ApiResponseWrapper<BasicChatResponseDTO> chat(
            @RequestBody ChatStartRequestDTO request,
            @AuthenticationPrincipal(expression = "customerId") Long customerId) {
        try {
            BasicChatResponseDTO response = chatService.chat(
//                    customerId,
                    request.getCustomerId(),
                    request.getSessionId(),
                    request.getInputMessage()
            );
            return ApiResponseWrapper.ok(response);
        } catch (IllegalArgumentException e) {
            return ApiResponseWrapper.error(400, e.getMessage());
        } catch (RuntimeException e) {
            // 로깅만 추가해서 추적 가능하도록 설정
            log.warn("Chat service runtime error: {}", e.getMessage());
            return ApiResponseWrapper.error(500, "일시적인 서비스 오류가 발생했습니다");
        } catch (Exception e) {
            log.error("Unexpected error in chat controller", e);
            return ApiResponseWrapper.error(500, "서버 내부 오류가 발생했습니다");
        }
    }
}
```

챗봇의 메인 컨트롤러이다. 챗봇의 경우는 단일 API로 처리하되, 플로우에 따른 요청만 내부에서 갈리기 때문에 Controller가 하나이다.

@AuthenticationPrincipal(expression = "customerId") Long customerId) 방식으로 JWT에서 customerId를 뽑아올 수 있도록 하였다.

ApiResponseWrapper를 통해 응답을 구성하도록 했는데, 이를 통해 좀 더 구조화된 형태로 json 응답이 가도록 구성해서 프론트에서 사용하기 편하도록 만들었다.

```java
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;

/**
 * API 응답을 표준화하기 위한 래퍼 클래스
 * @param <T>
 */
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

다음은 API 응답시 사용하는 Wrapper 클래스이다.

data에 응답에 필요한 내용이 들어가도록 구성했으며, 이를 통해 성공 상태 & 코드 & 서버 메세지와 따로 분리해두어 data를 프론트에서 사용하기 쉽도록 구성했다.

이런식으로 구성할 시, 응답은 다음과 같은 형태로 보내지게 된다.

```json
{
  "success": true,
  "code": 0,
  "message": "string",
  "data": {
    "sessionId": "string",
    "message": "string",
    "newHobby": "string",
    "products": [
      {
        "rank": 0,
        "name": "string",
        "price": 0,
        "image": "string",
        "url": "string",
        "category": "string"
      }
    ],
    "summary": "string",
    "flow": "string"
  }
}
```

## Service

### Main Chat Service

서비스의 경우는, 역할에 따라 분리하였기에 controller와 연결되는 메인 서비스가 있고, LLM 처리를 담당하는(즉, Fast api와 연결되는) 서비스, redis Cache를 관리하는 서비스, 플로우 관리 서비스, 입력 검증 서비스 이렇게 다섯 가지의 서비스가 존재한다.

우선 메인 서비스부터 살펴보자

```java
import com.ohgoodteam.ohgoodpay.recommend.dto.datadto.llmdto.BasicChatResponseDTO;

/**
 * 채팅 껍데기 서버 서비스 인터페이스
 *
 * interface 분리로 추후 구현체 변경 용이
 */
public interface ChatService {
    // 채팅 생성
    BasicChatResponseDTO chat(Long customerId, String sessionId, String message);
}
```

```java
import com.ohgoodteam.ohgoodpay.common.repository.CustomerRepository;
import com.ohgoodteam.ohgoodpay.recommend.dto.CustomerContextWrapper;
import com.ohgoodteam.ohgoodpay.recommend.dto.ValidationResult;
import com.ohgoodteam.ohgoodpay.recommend.dto.datadto.llmdto.*;
import com.ohgoodteam.ohgoodpay.recommend.service.fastapi.LlmService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

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
    private final CustomerRepository customerRepository; // DB 직접 접근용 (update)
    private final ChatCacheService chatCacheService; // 캐싱 서비스
    private final LlmService llmService; // FastAPI LLM 연동 서비스
    private final ValidCheckService validCheckService; // 입력 검증 서비스
    private final FlowService flowService; // 플로우 관리 서비스

    /**
     * 채팅 처리
     */
    @Override
    @Transactional(readOnly = true)
    public BasicChatResponseDTO chat(Long customerId, String sessionId, String message) {
      log.info("세션 아이디와 고객 아이디 체크 sessionId: {} for customerId: {}", sessionId, customerId);
  
      // 0. flow를 redis에서 가져오기
      String currentFlow = chatCacheService.getFlowBySession(sessionId);
  
      // start 플로우는 유효성 검증 없이 바로 LLM 처리
      if (currentFlow.equals("start")) {
        // 2. 플로우 전환 및 전처리
        String nextFlow = flowService.getNextFlow(currentFlow);
        chatCacheService.saveCntBySession(sessionId, 1); //count 초기화
        chatCacheService.saveFlowBySession(sessionId, nextFlow); //nextflow 초기화
        log.info("캐싱되어 있는 바뀐 플로우 확인하기 : {}", chatCacheService.getFlowBySession(sessionId));
  
        // 3. 캐싱된 데이터 수집
        CustomerContextWrapper context = collectCachedData(customerId, sessionId);
        // 4. LLM 요청
        BasicChatResponseDTO response = requestToLLM(sessionId, context, message, nextFlow);
        // 5. 응답 후처리
        processAfterResponse(response, customerId, sessionId);
        return response;
      }
  
      // 일반 플로우는 유효성 검증 후 처리
      ValidInputResponseDTO validated = validCheckService.validateInput(customerId, sessionId, message, currentFlow);
      ValidationResult validationResult = processValidationResult(validated, customerId, sessionId, currentFlow);
      if (validationResult.shouldReturn()) { //바로 리턴 해야 한다면, 플로우 변환 없이 처리
        return validationResult.getResponse();
      }
  
      // 2. 플로우 전환 및 전처리
      String nextFlow = flowService.getNextFlow(currentFlow);
      chatCacheService.saveCntBySession(sessionId, 1); //count 초기화
      chatCacheService.saveFlowBySession(sessionId, nextFlow); //nextflow 초기화
      log.info("캐싱되어 있는 바뀐 플로우 확인하기 : {}", chatCacheService.getFlowBySession(sessionId));
  
      // valid하지 않아서 해당 플로우를 스킵해야 할 경우, 정보를 저장하면 안 되기 때문이다.
      if (validated.isValid()) {
        saveFlowSpecificData(customerId, sessionId, nextFlow, validated);
      }
      // 3. 캐싱된 데이터 수집
      CustomerContextWrapper context = collectCachedData(customerId, sessionId);
      // 4. LLM 요청
      BasicChatResponseDTO response = requestToLLM(sessionId, context, message, nextFlow);
  
      // flow가 recommendation or re_recommendation 일 경우, products 캐싱 및 응답 처리
      if("recommendation".equals(response.getFlow()) || "re_recommendation".equals(response.getFlow())){
        response = handleRecommendationFlow(sessionId, response);
        // TODO : 캐싱 할 상품 모자랄 경우에 대한 해결책 및 로직 정의 필요.
      }
  
      // 5. 응답 후처리, 강제로 넘어간 경우는 저장 무시한다.
      if (validated.isValid()) {
        processAfterResponse(response, customerId, sessionId);
      }
      return response;
    }

    /**
     * 검증 결과 처리 및 재시도 로직
     */
    private ValidationResult processValidationResult(ValidInputResponseDTO validated, Long customerId, String sessionId, String currentFlow) {
        if (!validated.isValid()) {
            int cnt = chatCacheService.getCntBySession(sessionId);
            // 두 번 이하로 유효하지 않은 응답이 들어왔을경우, 다시 같은 플로우를 실행
            if (cnt < 2) {
                chatCacheService.saveCntBySession(sessionId, cnt + 1);
                BasicChatResponseDTO errorResponse = BasicChatResponseDTO.builder()
                        .sessionId(sessionId)
                        .message(validated.getMessage())
                        .newHobby("")
                        .products(null)
                        .build();
                return ValidationResult.returnResponse(errorResponse);
            } else if(cnt == 2) {
                // cnt == 2면 다음 플로우로 강제 진행하고 LLM 요청 계속 진행
                log.info("Validation 실패 {}번, 다음 플로우로 넘어가서 LLM 요청 진행", cnt);
                return ValidationResult.continueFlow();
            }
        }
        log.info("------------isValid True로 인해 플로우를 그대로 이어나간다.----------");
        return ValidationResult.continueFlow();
    }

    /**
     * 캐싱된 데이터 수집
     */
    private CustomerContextWrapper collectCachedData(Long customerId, String sessionId) {
        return CustomerContextWrapper.builder()
                .customerInfo(chatCacheService.getCustomerInfo(customerId))
                .hobby(chatCacheService.getHobby(customerId))
                .mood(chatCacheService.getMoodBySession(sessionId))
                .balance(chatCacheService.getBalance(customerId))
                .summary(chatCacheService.getSummaryBySession(sessionId))
                .build();
    }

    /**
     * 플로우별 특정 데이터 저장
     */
    private void saveFlowSpecificData(Long customerId, String sessionId, String nextFlow, ValidInputResponseDTO validated) {
        // 특정 플로우에서 mood 저장, 유효한 경우엔 캐싱하고 아닌 경우엔 저장 없이 플로우만 넘기도록 한다.
        if ("hobby_check".equals(nextFlow) && validated != null) {
            chatCacheService.saveMoodBySession(sessionId, validated.getInputMessage());
        }
    }

    /**
     * LLM 요청 처리
     */
    private BasicChatResponseDTO requestToLLM(String sessionId, CustomerContextWrapper context, String message, String nextFlow) {
        return llmService.generateChat(
                sessionId,
                context.getCustomerInfo(),
                context.getMood(),
                context.getHobby(),
                context.getBalance(),
                message,
                context.getSummary(),
                nextFlow
        );
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
        }
    }
}
```

인터페이스에는 Controller에서 사용하는 로직인 chat만 두고, 내부 서비스들은 인터페이스에 정의하지 않았다.

이는 **ISP (Interface Segregation Principle) : 클라이언트는 사용하지 않는 메서드에 의존하지 말아야 한다** 라는 원칙을 준수하기 위함이다. 그리고 내부 메서드들은 적절히 숨겨서 캡슐화를 해두는 것이 차후 유지보수 측면에서도 편리하기 때문이다.

여기는 로직이 좀 복잡하므로 method 하나하나 뜯어가면서 보도록 하자.

### 검증 결과 처리 및 재시도 로직

```java
/**
 * 검증 결과 처리 및 재시도 로직
 */
private ValidationResult processValidationResult(ValidInputResponseDTO validated, Long customerId, String sessionId, String currentFlow) {
    if (!validated.isValid()) {
        int cnt = chatCacheService.getCntBySession(sessionId);
        // 두 번 이하로 유효하지 않은 응답이 들어왔을경우, 다시 같은 플로우를 실행
        if (cnt < 2) {
            chatCacheService.saveCntBySession(sessionId, cnt + 1);
            BasicChatResponseDTO errorResponse = BasicChatResponseDTO.builder()
                    .sessionId(sessionId)
                    .message(validated.getMessage())
                    .newHobby("")
                    .products(null)
                    .build();
            return ValidationResult.returnResponse(errorResponse);
        } else if(cnt == 2) {
            // cnt == 2면 다음 플로우로 강제 진행하고 LLM 요청 계속 진행
            log.info("Validation 실패 {}번, 다음 플로우로 넘어가서 LLM 요청 진행", cnt);
            return ValidationResult.continueFlow();
        }
    }
    log.info("------------isValid True로 인해 플로우를 그대로 이어나간다.----------");
    return ValidationResult.continueFlow();
}
```

입력 유효성 검사를 추가했기 때문에 해당 메서드를 추가하였다.

이는 사용자의 현재 입력이 현재 플로우에 알맞은 입력인지를 혹인하기 위한 메서드이다.

ValidInputResponseDTO라는 validation 결과물을 가지고 판단하는데, 이를 잠깐 살펴보자.

```java
/*
 * FAST API - 현재 사용자의 input이 유효한지 검증 결과를 내보내는 DTO
 */
@Getter
@ToString
@Builder
@AllArgsConstructor
@NoArgsConstructor
public class ValidInputResponseDTO {
    private Long customerId;
    private String sessionId;
    private String inputMessage; // 사용자가 보낸 메시지, 한 번더 보낼때 재사용 하기 위함이다.

    @JsonProperty("isValid") // fast api랑 json 직렬화를 맞추기 위함이다.
    private boolean isValid; // 입력이 유효한지 여부
    private String message; // LLM이 생성한 응답 메시지, 유효하지 않을 경우에만 사용할 예정.
    private String flow; // 보냈던 대화 흐름 (다음 흐름으로 가도록 하기 위함이다. 이게 있어야 받아서 redis에 플로우 저장하고 다시 llm 요청 가능)
}

```

다음과 같이 현재 플로우와 LLM이 생성한 메세지, 그리고 LLM이 판단하기에 “현재 플로우에 알맞은지”를 판단한다.

다시 돌아가서 살펴보자면, Valid한 입력일 경우에는 바로 조건 체크 없이 넘어가지만, Valid한 입력이 아니라고 나올 경우에는 이 횟수가 “2번 이상인지”를 살펴본다. 즉, 첫번째로 입력한 값이 이상값이 아닐 경우엔 기회를 주지만, 두 번째 부터는 해당 플로우를 그냥 스킵하기 위해 이 조건을 넣은 것이다.

이를 통해 사용자 경험을 좋게 하되, 이상 값을 계속 입력하는 것을 방지하고자 하였다.

두 번 이하일 경우에는 error 메세지를 내보내고 ValidationResult.returnResponse(errorResponse)를 내보내는 것으로 되어 있는데 이 부분도 한 번 살펴보도록 하자.

```java
/**
 * Validation 결과를 담는 DTO
 */
@AllArgsConstructor
@Getter
public class ValidationResult {
    private final boolean shouldReturn;
    private final BasicChatResponseDTO response;

    public static ValidationResult continueFlow() {
        return new ValidationResult(false, null);
    }

    public static ValidationResult returnResponse(BasicChatResponseDTO response) {
        return new ValidationResult(true, response);
    }

    public boolean shouldReturn() {
        return shouldReturn;
    }
}
```

이는 플로우를 계속할지, 아니면 에러 응답을 내보내고 해당 service를 종료해서 다시 유효성 검증을 하게 할지를 판단하기 위핸 DTO이다.

returnResponse일 경우에는 바로 응답을 내보내는데, 이는 우리가 재응답을 받아야 하는 경우, LLM에서 받은 메세지를 그대로 내보내기 때문에 (즉, 메세지 생성을 요청하지 않음) 다음과 같이 continueFlow() & returnResponse(BasicChatResponseDTO response)를 나눠서 구성한 것이다.

```java
/**
 * 캐싱된 데이터 수집
 */
private CustomerContextWrapper collectCachedData(Long customerId, String sessionId) {
    return CustomerContextWrapper.builder()
            .customerInfo(chatCacheService.getCustomerInfo(customerId))
            .hobby(chatCacheService.getHobby(customerId))
            .mood(chatCacheService.getMoodBySession(sessionId))
            .balance(chatCacheService.getBalance(customerId))
            .summary(chatCacheService.getSummaryBySession(sessionId))
            .build();
}
```

에러 체킹이 끝났을 경우, 즉 해당 플로우를 계속 이어나가도 괜찮다는 허락을 받았을 경우 다음과 같이 응답 생성에 필요한 모든 캐싱 데이터를 수집한다.

모든 캐싱 데이터는 CustomerContextWrapper라는 이름으로 묶어서 작성및 관리가 편리하도록 하였다.

```java
/**
 * 플로우별 특정 데이터 저장
 */
private void saveFlowSpecificData(Long customerId, String sessionId, String nextFlow, ValidInputResponseDTO validated) {
    // 특정 플로우에서 mood 저장, 유효한 경우엔 캐싱하고 아닌 경우엔 저장 없이 플로우만 넘기도록 한다.
    if ("hobby_check".equals(nextFlow) && validated != null) {
        chatCacheService.saveMoodBySession(sessionId, validated.getInputMessage());
    }
}
```

특정 플로우 안에서는 값을 캐싱해야 하는 경우가 있기에, 특정 플로우 안에서 값을 저장하는 메서드를 생성하였다.

hobby_check가 다음 플로우일 경우에는, 이전 플로우가 mood_check이므로 saveMoodBySession를 통해 session 관리 값으로 입력받은 mood를 저장해준다.

참고로 mood 같은 경우는 세션 안에만 있는 값이라 따로 db에 저장을 안해서 llm 요청 후 처리가 아닌 요청 전 처리 값으로 빼두었다. 요청 후 처리 값으로 하면 response에 또 쓸데없이 mood를 추가해야 하기에..

```java
/**
 * LLM 요청 처리
 */
private BasicChatResponseDTO requestToLLM(String sessionId, CustomerContextWrapper context, String message, String nextFlow) {
    return llmService.generateChat(
            sessionId,
            context.getCustomerInfo(),
            context.getMood(),
            context.getHobby(),
            context.getBalance(),
            message,
            context.getSummary(),
            nextFlow
    );
}
```

이 부분이 LLM에 요청하는 부분이다.

llmService안에 generateChat을 통해 LLM에 메세지를 요청한다.

llmService단은 차후 fast api 서버 구조와 함께 보기 위해 우선은 넘어가도록 하겠다.

```java
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
    }
}
```

이 부분이 응답 후 데이터들을 처리하기 위한 로직이다.

취미가 변동된 경우에는 newHobby단에 값이 들어오기 때문에 조건을 다음과 같이 판단하도록 한다.

취미 변동 후에는 레디스와 db에 모두 새 취미를 업로드 해준다.

요약본 역시 LLM이 생성해주는 요약본을 저장해야 하므로, summary에 값이 있을 경우 레디스에 요약본을 넣어준다.

사실 지금 구조에서는 DB에 요약본을 저장할 필요는 없을 것 같으나, 현재 DB에 요약본을 저장하는 테이블이 있으므로, 향후 이 부분에 요약본을 DB에 업데이트 하는 로직을 추가할 예정이다. 우선순위는 아니다.

```java
/**
 * 추천 플로우 처리 (상품 캐싱 및 응답 재생성)
 */
private BasicChatResponseDTO handleRecommendationFlow(String sessionId, BasicChatResponseDTO response) {
    if ("recommendation".equals(response.getFlow())) {
        // 새 상품 캐싱
        chatCacheService.saveProductsBySession(sessionId, response.getProducts());
        // TODO : 캐싱 할 상품 모자랄 경우에 대한 해결책 및 로직 정의 필요.
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
```

추천이나 재 추천 상태일 경우, 레디스에서 상품을 가져와서 응답해야 하기에 다음과 같은 method를 마련하였다.

chatCacheService.getProductsBySession(sessionId) 안에는 제약조건이 있어서 상품을 3개씩 가져오도록 되어 있는데, 해당 부분에 대한 건 차후 포스팅에서 다루도록 하겠다.

```java
/**
 * 채팅 처리
 */
@Override
@Transactional(readOnly = true)
public BasicChatResponseDTO chat(Long customerId, String sessionId, String message) {
    log.info("세션 아이디와 고객 아이디 체크 sessionId: {} for customerId: {}", sessionId, customerId);

    // 0. flow를 redis에서 가져오기
    String currentFlow = chatCacheService.getFlowBySession(sessionId);

    // start 플로우는 유효성 검증 없이 바로 LLM 처리
    if (currentFlow.equals("start")) {
        // 2. 플로우 전환 및 전처리
        String nextFlow = flowService.getNextFlow(currentFlow);
        chatCacheService.saveCntBySession(sessionId, 1); //count 초기화
        chatCacheService.saveFlowBySession(sessionId, nextFlow); //nextflow 초기화
        log.info("캐싱되어 있는 바뀐 플로우 확인하기 : {}", chatCacheService.getFlowBySession(sessionId));

        // 3. 캐싱된 데이터 수집
        CustomerContextWrapper context = collectCachedData(customerId, sessionId);
        // 4. LLM 요청
        BasicChatResponseDTO response = requestToLLM(sessionId, context, message, nextFlow);
        // 5. 응답 후처리
        processAfterResponse(response, customerId, sessionId);
        return response;
    }

    // 일반 플로우는 유효성 검증 후 처리
    ValidInputResponseDTO validated = validCheckService.validateInput(customerId, sessionId, message, currentFlow);
    ValidationResult validationResult = processValidationResult(validated, customerId, sessionId, currentFlow);
    if (validationResult.shouldReturn()) { //바로 리턴 해야 한다면, 플로우 변환 없이 처리
        return validationResult.getResponse();
    }

    // 2. 플로우 전환 및 전처리
    String nextFlow = flowService.getNextFlow(currentFlow);
    chatCacheService.saveCntBySession(sessionId, 1); //count 초기화
    chatCacheService.saveFlowBySession(sessionId, nextFlow); //nextflow 초기화
    log.info("캐싱되어 있는 바뀐 플로우 확인하기 : {}", chatCacheService.getFlowBySession(sessionId));

    // valid하지 않아서 해당 플로우를 스킵해야 할 경우, 정보를 저장하면 안 되기 때문이다.
    if (validated.isValid()) {
        saveFlowSpecificData(customerId, sessionId, nextFlow, validated);
    }
    // 3. 캐싱된 데이터 수집
    CustomerContextWrapper context = collectCachedData(customerId, sessionId);
    // 4. LLM 요청
    BasicChatResponseDTO response = requestToLLM(sessionId, context, message, nextFlow);

    // flow가 recommendation or re_recommendation 일 경우, products 캐싱 및 응답 처리
    if("recommendation".equals(response.getFlow()) || "re_recommendation".equals(response.getFlow())){
        response = handleRecommendationFlow(sessionId, response);
        // TODO : 캐싱 할 상품 모자랄 경우에 대한 해결책 및 로직 정의 필요.
    }

    // 5. 응답 후처리, 강제로 넘어간 경우는 저장 무시한다.
    if (validated.isValid()) {
        processAfterResponse(response, customerId, sessionId);
    }
    return response;
}
```

자, 이제 모든 method에 대해서 살펴 보았으므로, 이를 종합한 chat 메서드를 보도록 하자.

중요한 것은, 플로우를 가져와서 이를 통해 입력 값과 함께 보내 “응답 유효성 검증”을 LLM에 먼저 요청한 다음에, 유효하거나 두 번 이상 틀렸을 경우 LLM에 메세지 생성을 요청하는 것이다.

또한, flow가 start일 경우에는 사용자 입력값이 없으므로 유효성 검사를 진행할 수 없기에 따로 조건으로 빼주었다.

일반 플로우의 경우. validationResult.shouldReturn() 이어서 바로 리턴해야 하는 경우 (유효성 검증 로직 한 번 틀렸을 경우 여기에 걸린다.) 바로 return해서 사용자가 다시 입력할 수 있도록 한다.

유효성 검증을 통과했을 경우, chat을 보내기 위한 준비를 하는데 이때, 유효성 검증이 두 번 이상 틀려서 스킵해야 하는 상태일 경우 틀린 응답을 캐싱하면 안되기에 다음의 조건을 준 것이다.

```java
if (validated.isValid()) {
    saveFlowSpecificData(customerId, sessionId, nextFlow, validated);
}
```

그 다음에는 캐싱된 요소들을 수집해서 LLM에 요청을 보낸다.

그 후에 요청 후 처리해야 할 것들을 처리하는데, 이 과정에서 상태가 "recommendation"이나 "re_recommendation"인 경우에는 캐싱되어 있는 상품들을 더해서 응답할 수 있도록 한다.

또한, 마지막 응답 후 처리의 경우 마찬가지로 강제로 넘어가서 값이 없는 경우는 무시하도록 처리한다.

## 회고…

나름 많이 신경써서 만든 서비스이긴 하지만, 나름대로의 아쉬움은 존재한다.

우선 메인 chat() 메서드에 조건 분기가 과도하게 집중되어 알아보기가 어렵다.

사실 이마저도 1차 리팩터링때 메서드 별로 분리한건데도 이렇게 메인 메서드에 if 조건분기라던지, 플로우별 메서드 라던지…이 모든게 너무 세부적으로 많이 쪼개져있다.

또한 start 플로우와 일반 플로우의 처리 방식에 차이가 있음에도 한 메서드에 몰려있어 가시성이 너무 안좋다.

그래서 향후 리팩터링 시에는 플로우 타입 별로 메서드를 정의해서 가시성이 좋게 만들 예정이다.

또한 메인 메서드는 라우터의 역할만 수행하고, 플로우별 핵심 로직들은 플로우별 메서드 안에서 돌아갈 수 있도록 하여 응집도를 높여보려고 한다.

### 오버 엔지니어링에 대한 고찰

이정도 수준까지 챗봇을 심화 구현 하면서 나름대로의 고민이 굉장히 많았다.

일반적인 챗봇의 경우는 이 정도까지 심화 구현 하는 경우가 많이 없기 때문이다...

과연 챗봇에 이정도 품을 들이는게 올바른 선택인가? 오버 엔지니어링이 아닐까? 솔직히 수 많은 고민을 했다.

이 고민은 내가 올린 팀 PR에도 그대로 녹아있다….

[**[sunjung] feat: 채팅 플로우 관리를 boot 메인 서버에 위임**](https://github.com/OhGoodTeam/OhgoodpayBE/pull/22)

![image.png](/images/2025-09-20-shinhanDSsw-project2-5/1.png)

그러나 우리 수업 과정상 메인 서버를 spring boot로 진행해야 한다면, 서버간의 역할을 명확히 하는 것이 맞기에 이런 방식이 절대 오버 엔지니어링이 아닌, 아키텍처상 필수 요구사항에 해당한다고 생각한다.

fast api 하나로만 진행했다면 프롬프터 엔지니어링으로 대부분의 문제를 해결했겠지만, boot서버를 메인으로 두고자 한다면 이렇게 메인 서버에서 플로우 관리를 책임지게 하고, redis를 이용해서 서버 상태를 stateless하게 유지하는게 향후 확장성에 굉장히 유리할 것이라고 생각한다.

또한, 금융권에서는 보안을 위해 실제로 코어 뱅킹 서비스와 ai 서비스를 나눠서 운영하는 경우가 많기에 합리적인 방식이었다고 생각한다. **결국 우리도 금융 도메인의 서비스이므로**

나머지 로직에 대한 분석은 다음 포스팅에서~
