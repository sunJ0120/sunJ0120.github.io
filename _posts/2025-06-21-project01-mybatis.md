---
title: "[신한 DS SW아카데미 / 1차 프로젝트] 다중 필터 구현 - MyBatis로 동적 쿼리 구성하기"
layout: post
categories: [mybatis, project]
tags : [shinhanDS, project, mybatis, jsp, spring]
toc: true
toc_sticky: true
toc_label: 목차
author_profile : true
permalink: /project01/mybatis01
---

> ⭐ 해당 프로젝트의 경우, 존재하는 레퍼런스에서 약간 변형하여 팀 프로젝트로 제작 중인 신한DS 부트캠프 1차 미니 프로젝트입니다.   
> {: .notice--info}

# 🤔 Intro

![main](/images/2025-06-21-project01-mybatis/2.gif)

- 현재 진행중인 1차 프로젝트의 메인 페이지 필터 기능의 경우, 다중 선택이 가능하다.
- 다중 선택이 가능하므로, 어떤 필터가 추가되느냐에 따라 쿼리를 동적으로 구성해야 한다.
- 그러므로 MyBatis로 동적 쿼리를 조정하는 방법을 알아보자.


# 😀 Start!
## 필터 구성 전 mapper 쿼리..
```xml
<select id="selectAllStore" resultType="kr.co.ohgoodfood.dto.MainStore">
    SELECT
        s.store_id AS store_id, s.store_name AS store_name, s.store_menu AS store_menu, s.store_status AS store_status, s.category_bakery AS category_bakery, s.category_fruit AS category_fruit, s.category_salad AS category_salad, s.category_others AS category_others, s.closed_at AS closed_at,
        p.product_no AS product_no, p.pickup_start AS pickup_start, p.pickup_end AS pickup_end, p.origin_price AS origin_price, p.sale_price AS sale_price, p.reservation_end AS reservation_end, p.amount AS amount,
        MIN(i.store_img) AS store_img,
        b.bookmark_no AS bookmark_no,
    CASE WHEN b.bookmark_no IS NOT NULL THEN true ELSE false END AS bookmark
    FROM Store s
        JOIN Product p ON s.store_id = p.store_id
        JOIN Image i ON i.store_id = s.store_id
        LEFT JOIN Bookmark b ON b.store_id = s.store_id AND b.user_id = #{user_id}
    GROUP BY s.store_id, s.store_name;
</select>
```
- 현재 main 페이지의 model에 들어가는 dto인 MainStore DTO 객체를 구성하는 쿼리이다.
- Main과 Bookmark에서 동일한 객체를 사용하기에, bookmark 여부까지 left join으로 받아오도록 구성하였고, 이를 통해 차후 main 페이지에서 북마킹 기능을 추가해야 할시, 확장이 용이하도록 구성했다.
- 여기서 중요한 수정사항은, filter에 따라 동적으로 해당 쿼리를 구성하는 것이다.

## AJAX 요청 정보
- 현재 데이터 명세에 따른 AJAX Request json은 다음과 같다.
```json
{
  "store_status": "Y",
  "pickup_start": "2025-06-21",
  "category_bakery": "Y"
}
```
- store_status : 예약 가능 가게만 보여주도록 한다.
- pickup_start : 이를 통해 오늘 픽업인지 내일 픽업인지를 따진다.
- "category_bakery": "Y" : 카테고리 정보를 나타낸다.
    - 현재 카테고리는 네 개로, Store 테이블에 다음과 같은 형태로 저장되어 있다.
    1. "category_bakery" : String
    2. “category_fruit”: String
    3. “category_salad”: String
    4. “category_others”: String
- 카테고리 테이블을 따로 분리하지 않은 것은, 카테고리 수가 적기도 하고, 카테고리 추가 기능까지 확장할 계획이 아직 없기 때문이다. 이 부분은 강사님의 피드백을 받아서 구성한 부분이다.

## AJAX 구성 (JQuery)
```javascript
// 공통 AJAX 요청 함수
function sendFilterRequest() {
  $.ajax({
    url: '${pageContext.request.contextPath}/user/filter/store',
    type: 'POST',
    contentType: 'application/json',
    data: JSON.stringify(filterParams),
    //fragment 방식 사용, DOM을 다시 그리자니 너무 노가다라...
    success: function (responseHtml) {
      //로그 찍기
      console.log("[AJAX 응답] 서버에서 받은 HTML:", responseHtml);
      $('.productWrapper').html(responseHtml);
      applyBadgeStyles(); //오퍼시티 적용
    },
    error: function (xhr, status, error) {
      console.error("[AJAX 오류 발생]");
      console.error("status:", status);
      console.error("HTTP 상태 코드:", xhr.status);
      console.error("응답 텍스트:", xhr.responseText);
      console.error("error 객체:", error);
    }
  });
}
```
- 토글 이벤트를 통해 let *filterParams* = {};에 json 객체가 담기면, 이 ajax 요청 함수를 통해 Controller로 요청을 보내게 된다.
- 명세했던대로 data에 JSON 객체를 request한다.
    - 원래는 JSON 객체를 받아서 화면 데이터를 변경해서 상태 변경을 시도할까 하다가…(으레 다른 프레임워크들이 그렇듯이) JSP 방식에서는 너무 한계가 있는 방식이라 그냥 fragment를 갈아 끼우는 방식을 사용하기로 하였다.
- .productWrapper 영역만 갈아 끼우면 되므로, 이 영역을 지정한다.

- fragment는 따로 파일을 빼야 해서 다음과 같이 파일을 분리해서 복사해주었다.

![main](/images/2025-06-21-project01-mybatis/1.png)
- 파일 경로는 반드시 MvcConfig.java 설정에 있는 뷰 리졸버 경로에 맞춰준다.

## UserMapper.java
```java
@Mapper
public interface UserMapper {
    //[user] user main 화면 & 북마크에서 보이는 가게 리스트
    List<MainStore> selectAllStore(@Param("user_id") String user_id,
                                   @Param("filter") Map<String, String> filterParams);
 //.....
 }
```
- 필터 적용으로 filterParams을 받아오게 되었으므로, 이를 @Param 애노테이션으로 묶어서 map으로 전송한다.

## 필터 구성 후 UserMapper.xml
```xml
<select id="selectAllStore" parameterType="map" resultType="kr.co.ohgoodfood.dto.MainStore">
      SELECT
          s.store_id AS store_id, s.store_name AS store_name, s.store_menu AS store_menu, s.store_status AS store_status, s.category_bakery AS category_bakery, s.category_fruit AS category_fruit, s.category_salad AS category_salad, s.category_others AS category_others, s.closed_at AS closed_at,
          p.product_no AS product_no, p.pickup_start AS pickup_start, p.pickup_end AS pickup_end, p.origin_price AS origin_price, p.sale_price AS sale_price, p.reservation_end AS reservation_end, p.amount AS amount,
          MIN(i.store_img) AS store_img,
          b.bookmark_no AS bookmark_no,
      CASE WHEN b.bookmark_no IS NOT NULL THEN true ELSE false END AS bookmark
      FROM Store s
          JOIN Product p ON s.store_id = p.store_id
          JOIN Image i ON i.store_id = s.store_id
          LEFT JOIN Bookmark b ON b.store_id = s.store_id AND b.user_id = #{user_id}

      <where>
          <if test="filter.store_status != null">
              AND s.store_status = #{filter.store_status}
          </if>
          <if test="filter.pickup_start != null">
              AND DATE(p.pickup_start) = #{filter.pickup_start}
          </if>
          <if test="filter.category_bakery != null">
              AND s.category_bakery = #{filter.category_bakery}
          </if>
          <if test="filter.category_fruit != null">
              AND s.category_fruit = #{filter.category_fruit}
          </if>
          <if test="filter.category_salad != null">
              AND s.category_salad = #{filter.category_salad}
          </if>
          <if test="filter.category_others != null">
              AND s.category_others = #{filter.category_others}
          </if>
      </where>

      GROUP BY s.store_id, s.store_name;
  </select>s.store_id, s.store_name;
</select>
```
- 다음처럼 IF 조건을 더해준다.
- Y인 경우만 들어오므로, null 체크만 해주도록 하자.
- MyBatis에서는 이처럼 xml 방식으로 if 문을 사용해서 동적으로 쿼리 조건을 조정할 수 있다.

## UserController
- AJAX에서 /user/filter/store로 Controller 요청을 보내기로 하였으므로, 해당하는 컨트롤러를 구현해야 한다.
```java
@PostMapping("/filter/store")
public String filterStoreList(@RequestBody Map<String, String> filterParams, Model model){
    log.info("[log/UsersController.filterStoreList] 받은 필터 파라미터 결과 log : {}", filterParams);

    //임시 하드코딩 값, 실제로는 세션에서 받아온다.
    String user_id = "u04";

    List<MainStore> mainStoreList = usersService.getMainStoreList(user_id, filterParams);
    model.addAttribute("mainStoreList", mainStoreList);

    // JSP fragment만 리턴
    return "users/fragment/userMainStoreList";
}
```
- usersService에 구현되어 있는 getMainStoreList를 그대로 이용한다.
    - 어차피 필터 조건이 붙어있는 mapper가 그대로 연결되어 있기 때문에, 그저 service를 이용해주면 된다. (MVC 구조)
- MVC 구조를 따라서 model에 객체를 담아주고 JSP fragment 링크로 return 해준다.
    - 앞에서도 얘기했지만, fragment 경로는 뷰 리졸버에 있는 그대로 맞춰준다.
- 이렇게 하면 받아온 쿼리를 fragment에 연결해서 해당 fragment의 DOM 데이터를 변경하게 된다.
- AJAX에서는 지정한 부분에 변경된 fragment DOM을 붙여준다. 이를 통해 화면 상태를 변경할 수 있고, 다중 필터 구현이 가능하다:)
