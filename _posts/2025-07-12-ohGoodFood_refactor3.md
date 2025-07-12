---
title: "[신한 DS SW아카데미 / 1차 프로젝트] [BE] User order 리팩터링 하기"
layout: single
categories: [project]
tags : [shinhanDS, project, refactor]
toc: true
toc_sticky: true
toc_label: 목차
author_profile : true
permalink: /project01/userOrder/BE/refactor
---

> ⭐ 해당 프로젝트의 경우, 존재하는 레퍼런스에서 약간 변형하여 팀 프로젝트로 제작 중인 신한DS 부트캠프 1차 미니 프로젝트입니다.   
> {: .notice--info}

# 🤔 Intro

- main과 마찬가지로 내가 맡은 order도 리팩터링을 진행해보자!

# 😀 Start!
## 목표
- 기존의 코드에서 리팩토링을 진행해서 더 나은 코드를 만들어보자!
- 기존 코드 역시, 앞에서 봤던 bokmark, main과 비슷한 형태이므로 이번 포스팅에서는 기존 코드 없이 가겠다.

## 화면명세
- bookmark 화면의 화면 명세는 다음과 같다.
<table style="margin: 0 auto;">
  <tr>
    <td><img src="/images/2025-07-12-ohGoodFood_refactor3/intro1.jpg" style="display: block; margin: 0 auto;" /></td>
    <td><img src="/images/2025-07-12-ohGoodFood_refactor3/intro2.jpg" style="display: block; margin: 0 auto;" /></td>
    <td><img src="/images/2025-07-12-ohGoodFood_refactor3/intro3.jpg" style="display: block; margin: 0 auto;" /></td>
  </tr>
</table>

- 미확정 주문, 전체 주문, 확정 주문 세가지 필터링이 가능하다.
  - 확정 진행중인 주문은 reservation_end 한 시간 전에는 주문 취소가 불가능하다. 이런 경우, 막혀있다.
  - 픽업은 오늘 픽업 & 내일 픽업 두 가지 종류가 있다.
  - 픽업 완료된 주문은 리뷰를 쓸 수 있으며, 이미 리뷰가 있는 주문은 리뷰를 쓸 수 없다.
  - 취소의 경우, 구매자 취소 or 판매자 취소 두가지로 나뉜다.
  - 픽업 마감 시간까지 픽업을 하지 않은 경우, 구매자 취소 처리되어 바로 반영된다.

## DTO & Mapper
```java
import lombok.*;

/**
 * [DTO] UserOrderDTO.java
 *
 * - Order List 정보에 필요한 DTO
 * - 카드 안에 들어가는 모든 정보들을 한 번에 저장해서 사용한다.
 * - @Data로 불필요한 애노테이션을 늘리는 것 보다는, 필요한 애노테이션만 더해주었다.
 */

@Getter
@Setter
@ToString
@Builder
@AllArgsConstructor //Builder와 테스트용 생성자
@NoArgsConstructor  // MyBatis 매핑용 기본 생성자
public class UserOrderDTO {
    //Store table
    private Store store;

    //Paid table
    private Paid paid;

    //Product table
    private Product product;

    //Order table
    private Orders order;

    //Image table
    private Image image;

    //[추가 정보] DB에는 없는 추가 정보
    private Boolean block_cancel; //확정 시간 한 시간 전에 취소하지 못하도록 변수 생성
    private Boolean has_review; //리뷰가 존재하는 주문인지 판단하기 위함이다.
    private int point; //주문에 해당하는 point 지급을 위한 컬럼 (order 금액의 1%)
}
```
- 마찬가지로, 객체를 직접 주입하는 식으로 바꿔주었기 때문에 mapper 역시 resultMap을 활용하는 방향으로 바꿔줘야 한다.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- [refactor] order을 위한 전용 mapper -->
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http:/mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="kr.co.ohgoodfood.dao.UserOrderMapper">
    <!-- id는 resultMap의 식별자, type은 결과를 채워넣을 DTO 클래스 -->
    <resultMap id="UserOrderDTO" type="kr.co.ohgoodfood.dto.UserOrderDTO">
        <!-- store 연관관계 매핑 -->
        <association property="store" javaType="kr.co.ohgoodfood.dto.Store">
            <id     column="store_id"          property="store_id"/>
            <result column="store_name"        property="store_name"/>
        </association>

        <!-- Paid 연관관계 매핑 -->
        <association property="paid" javaType="kr.co.ohgoodfood.dto.Paid">
            <id column="paid_no"   property="paid_no"/>
            <result column="paid_price"   property="paid_price"/>
            <result column="paid_status"   property="paid_status"/>
            <result column="paid_point"   property="paid_point"/>
        </association>

        <!-- product 연관관계 매핑 -->
        <association property="product" javaType="kr.co.ohgoodfood.dto.Product">
            <id     column="product_no"          property="product_no"/>
            <result column="pickup_start"        property="pickup_start"/>
            <result column="pickup_end"      property="pickup_end"/>
            <result column="sale_price"   property="sale_price"/>
            <result column="reservation_end"   property="reservation_end"/>
        </association>

        <!-- Orders 연관관계 매핑 -->
        <association property="order" javaType="kr.co.ohgoodfood.dto.Orders">
            <id column="order_no"   property="order_no"/>
            <result column="ordered_at"   property="ordered_at"/>
            <result column="quantity"   property="quantity"/>
            <result column="order_status"  property="order_status"/>
            <result column="canceld_from"  property="canceld_from"/>
            <result column="order_code"   property="order_code"/>
            <result column="user_id"   property="user_id"/>
        </association>

        <!-- Image 연관관계 매핑 -->
        <association property="image" javaType="kr.co.ohgoodfood.dto.Image">
            <result column="store_img"   property="store_img"/>
        </association>
    </resultMap>

    <!-- selectOrderList mapper -->
    <select id="selectOrderList" resultMap="UserOrderDTO">
        SELECT
            s.store_name AS store_name, s.store_id AS store_id,
            pa.paid_no AS paid_no, pa.paid_price AS paid_price, pa.paid_status AS paid_status, pa.paid_point AS paid_point,
            p.product_no AS product_no, p.pickup_start AS pickup_start, p.pickup_end AS pickup_end, p.sale_price AS sale_price, p.reservation_end AS reservation_end,
            o.order_no AS order_no, o.ordered_at AS ordered_at, o.quantity AS quantity, o.order_status AS order_status, o.canceld_from AS canceld_from,
            o.order_code AS order_code, o.user_id AS user_id,
            MIN(i.store_img) AS store_img,
        CASE WHEN COUNT(r.review_no) > 0 THEN 1 ELSE 0 END AS has_review
        FROM Store s
            JOIN Product p ON s.store_id = p.store_id
            JOIN Image   i ON i.store_id = s.store_id
            JOIN Orders  o  ON o.product_no = p.product_no
            JOIN Paid    pa ON pa.order_no = o.order_no
            LEFT JOIN Review r ON r.order_no = o.order_no
        <where>
            <!-- user_id는 무조건 있어야 하므로, null 아닐 때 조건은 제거. -->
            AND o.user_id = #{filter.user_id}
            <!-- ready는 제외 -->
            AND o.order_status != 'ready'
            <if test="filter.order_status != null">
                <choose>
                    <!-- order_status가 List일 때 IN 절 -->
                    <when test="filter.order_status instanceof java.util.List">
                        AND o.order_status IN
                        <foreach item="status"
                                 collection="filter.order_status"
                                 open="("
                                 separator=","
                                 close=")">
                            #{status}
                        </foreach>
                    </when>
                    <!-- 단일값일때 처리 -->
                    <otherwise>
                        AND o.order_status = #{filter.order_status}
                    </otherwise>
                </choose>
            </if>
            <!-- ORDER BY 순서 : 확정진행중, 확정완료, 픽업완료, 취소순 -->
        </where>
        GROUP BY
            pa.paid_no, pa.paid_price, pa.paid_status,
            s.store_name, s.store_id,
            p.product_no, p.pickup_start, p.pickup_end, p.sale_price,
            o.order_no, o.ordered_at, o.quantity,
            o.order_status, o.canceld_from, o.order_code, o.user_id
        ORDER BY
        CASE o.order_status
            WHEN 'reservation' THEN 1
            WHEN 'confirmed'   THEN 2
            WHEN 'pickup'      THEN 3
            WHEN 'cancel'      THEN 4
        ELSE 5
        END,
        o.ordered_at DESC;
    </select>

    <!-- updateOrderStatus mapper-->
    <update id="updateOrderStatus" parameterType="kr.co.ohgoodfood.dto.UserOrderRequest">
        UPDATE Orders
        <set>
            <if test="order_request.order_status != null">
                order_status = #{order_request.order_status},
            </if>
            <if test="order_request.order_status == T(kr.co.ohgoodfood.dto.OrderStatus).CANCEL">
                canceld_from = #{order_request.canceld_from},
            </if>
            <if test="order_request.order_status == T(kr.co.ohgoodfood.dto.OrderStatus).CONFIRMED">
                order_code = #{order_request.order_code},
            </if>
        </set>
        WHERE order_no = #{order_request.order_no}
        AND user_id = #{order_request.user_id};
    </update>

    <!-- restoreProductAmount mapper-->
    <update id="restoreProductAmount" parameterType="kr.co.ohgoodfood.dto.UserOrderRequest">
        UPDATE Product
        SET amount = amount + #{order_request.quantity}
        WHERE product_no = #{order_request.product_no}
    </update>

</mapper>
```
- 이전에 리팩토링 했던 것과 같은 방식으로, resultMap을 이용해서 객체 지향적인 설계에 가깝도록 구성한다.
- 쿼리는 기존과 동일하게 사용한다.
- 마찬가지로, Order에서 사용하는 전용 mapper를 분리한다.

```java
import kr.co.ohgoodfood.dto.UserOrderDTO;
import kr.co.ohgoodfood.dto.UserOrderFilter;
import kr.co.ohgoodfood.dto.UserOrderRequest;
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Param;

import java.util.List;

/**
 * UserOrderMapper
 *
 * 사용자 order 페이지에서 사용하는 mapper interface
 */
@Mapper
public interface UserOrderMapper {
    /**
     * 사용자의 모든 주문내역을 출력
     *
     * @param userOrderFilter  필터 DTO
     * @return                 UserOrder 리스트
     */
    List<UserOrderDTO> selectOrderList(@Param("filter") UserOrderFilter userOrderFilter);

    /**
     * 사용자가 주문 상태를 변경해야 할때 사용한다.
     *
     * @param userOrderRequest 필터 DTO
     */
    int updateOrderStatus(@Param("order_request") UserOrderRequest userOrderRequest);

    /**
     * 사용자 주문 취소시, Product의 amount를 복원하기 위함이다.
     *
     * @param userOrderRequest 필터 DTO
     */
    int restoreProductAmount(@Param("order_request") UserOrderRequest userOrderRequest);

}
```
- mapper interface는 다음과 같이 구성한다.
- 마찬 가지로 주석에는 각 DTO안에 무엇이 들어가고, 무엇을 기준으로 어떤 return 값이 나오는지를 좀 더 명확히 볼 수 있도록 구성했다.

### [추가] FilterDTO
- Order에는 이 UserOrderDTO외에도 상태를 업데이트 하기 위한 DTO가 필요하다. 해당 DTO는 화면에 뿌리는 용도가 아니므로, 필드를 사용했던 그 방식을 그대로 따라가기로 한다.

```java
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.PositiveOrZero;
import lombok.*;

/**
 * [DTO] UserOrderRequest.java
 *
 * - OrderList에서 필요한 CRUD 기능을 수행하기 위한 DTO
 */

@Getter
@Setter
@ToString
@Builder
@AllArgsConstructor //Builder와 테스트용 생성자
@NoArgsConstructor  // MyBatis 매핑용 기본 생성자
public class UserOrderRequest {
    private String user_id;

    //해당 order의 주문 정보를 update 하는 것이므로, order_no는 not null
    @NotNull(message = "order_no는 필수값입니다")
    private int order_no;

    //해당 poduct의 주문 정보를 update 하는 것이므로, product_no는 not null
    @NotNull(message = "product_no는 필수값입니다")
    private int product_no;

    // 되살리는 주문 수량은 0 이상이어야 한다.
    @PositiveOrZero(message = "quantity는 0 이상이어야 합니다")
    private int quantity;

    //user or store
    private String canceld_from;

    // 주문 상태 (confirmed, pickup, cancel, ready)
    private String order_status;

    private int order_code;
}
```
- 마찬 가지로 상태 update에 쓰이는 dto역시 애노테이션을 붙여준다.
- @NotNull을 통해, 상태 업데이트시 반드시 필요한 필드에 이를 붙여준다.
- @PositiveOrZero(message = "quantity는 0 이상이어야 합니다")
  - 업데이트 하는 수량은 반드시 0 이상이어야 하므로 이 애노테이션을 붙여준다.
- canceld_from, order_status 도 enum 타입으로 맞춰주는게 편하긴 하나, db에서 받아서 변환하는 과정이 까다롭고 & 현재 vo를 공용으로 사용하고 있어서 이 프로젝트 내에서는 enum으로 분리하지 않고 그대로 string으로 두기로 한다.

```java
import lombok.Data;
import lombok.NoArgsConstructor;

import java.util.List;

/**
 * [DTO] UserOrderFilter.java
 *
 * - Order List 정보를 띄울때, filtering을 위한 dto
 * - ajax 요청에서 들어가는 모든 정보를 여기에 넣어서 구성한다.
 */

@Data
@NoArgsConstructor
public class UserOrderFilter {
  //Order 정보를 가져오기 위한 user_id
  private String user_id;

  //Order 상태 정보로 필터링 하기 위함
  private List<String> order_status;
}
```
- UserOrderFilter는 토글 필터링을 통한 결과를 가져오기 위함인데, 따로 바꿀 내용은 없어서 그대로 두었다.

## Controller
```java

import jakarta.validation.Valid;
import kr.co.ohgoodfood.dto.*;
import kr.co.ohgoodfood.exception.OrderCancelException;
import kr.co.ohgoodfood.service.users.UserOrderService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.validation.BindingResult;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;

import javax.servlet.http.HttpSession;
import java.util.List;

/**
 * UserOrderController.java
 *
 * - GET  /user/orderList      : 유저가 가진 orderList 목록 조회
 * - POST /user/filter/order   : AJAX 기반 오더 목록 필터링
 * - POST /user/order/cancel   : 유저가 선택한 오더 주문 취소
*/

@Controller
@RequestMapping("/user")
@Slf4j
@RequiredArgsConstructor
public class UserOrderController {
    private final UserOrderService userOrderService;

    /**
     * 세션에 있는 유저가 가진 주문 목록을 조회한다.
     *
     * @param model                뷰에 전달할 데이터(Model)
     * @param session              현재 HTTP 세션(로그인된 사용자 정보)
     * @return                     users/userOrders.jsp로 포워딩
     */
    @GetMapping("/orderList")
    public String userOrderList(Model model,
                                HttpSession session){

        Account loginUser = (Account) session.getAttribute("user");
        String user_id = loginUser.getUser_id();

        UserOrderFilter userOrderFilter = new UserOrderFilter();

        userOrderFilter.setUser_id(user_id); //필터에 id값 추가
        List<UserOrderDTO> userOrderList = userOrderService.getUserOrderList(userOrderFilter);

        log.info("UserOrderServiceImpl.getUserOrderList() - userOrderList : {}", userOrderList.toString());

        model.addAttribute("userOrderList", userOrderList);

        return "users/userOrders";
    }

    /**
     * AJAX 필터링 결과에 따른 주문 목록을 조회하고 뷰 프래그먼트만 반환한다.
     *
     * @param userOrderFilter      JSON 바디로 전달된 필터 정보 (order_status 정보)
     * @param model                뷰에 전달할 데이터(Model)
     * @param session              현재 HTTP 세션(로그인된 사용자 정보)
     * @return                     필터링 된 가게 주문 목록만 포함한 JSP 프래그먼트 ("users/fragment/userOrderList")
     */
    @PostMapping("/filter/order")
    public String filterOrderList(@RequestBody UserOrderFilter userOrderFilter,
                                  Model model,
                                  HttpSession session){
        //세션에서 받아오는 로직
        Account loginUser = (Account) session.getAttribute("user");
        String user_id = loginUser.getUser_id();

        userOrderFilter.setUser_id(user_id); //필터에 id값 추가
        List<UserOrderDTO> userOrderList = userOrderService.getUserOrderList(userOrderFilter);
        model.addAttribute("userOrderList",userOrderList);

        return "users/fragment/userOrderList";
    }

    /**
     * 세션에 있는 유저가 가진 주문 목록중 선택한 것을 취소한다.
     *
     * @param userOrderRequest     요청 파라미터와 바인딩되어 뷰로 전달되는 DTO (order_no, product_no에 해당하는 주문의 order_status 번경 & quantity 복원)
     * @param session              현재 HTTP 세션(로그인된 사용자 정보)
     * @param redirectAttributes   리다이렉트 시에도 메세지를 담을 수 있도록 하는 객체
     * @return                     PRG : /user/orderList 로 리다이렉트
     */
    @PostMapping("/order/cancel")
    public String cancelOrder(@Valid @ModelAttribute UserOrderRequest userOrderRequest,
                              BindingResult br,
                              HttpSession session,
                              RedirectAttributes redirectAttributes){

        // 유효성 검증 결과가 있다면, error 메시지를 모델에 담아서 다시 뷰로 리다이렉트
        if (br.hasErrors()) {
            // redirectAttributes에 에러 메시지를 담는다.
            redirectAttributes.addFlashAttribute("errorMsg", "[ERROR!] 주문 취소에 실패했습니다.");
            // 다시 list로 리다이렉트, 취소 주문 요청 실패
            return "redirect:/user/orderList";
        }

        Account loginUser = (Account) session.getAttribute("user");
        String user_id = loginUser.getUser_id();

        userOrderRequest.setUser_id(user_id);

        //주문 취소의 경우는, 두 테이블을 UPDATE 하므로 @Transactional 처리, 그러므로 예외처리해준다.
        try {
            userOrderService.updateUserOrderCancel(userOrderRequest);
            redirectAttributes.addFlashAttribute("msg", "주문이 정상적으로 취소되었습니다.");
        } catch (OrderCancelException e) {
            log.error(e.getMessage()); // 로그에 에러 메시지 출력
            // 트랜잭션은 exception 던졌을 때 롤백됨
            redirectAttributes.addFlashAttribute("errorMsg", "[ERROR!] 주문 취소에 실패했습니다.");
        }
        return "redirect:/user/orderList";
    }
}
```
- /userOrderList
  - main에서 그랬던 것처럼, 여기서도 불필요한 modelAttrbute를 삭제한다.
- /filter/order
  - 로직 자체에 변동 사항은 없다.
  - order filter dto에는 유효성 검사 애노테이션이 존재하지 않기 때문에 따로 bindingResult를 해줄 필요가 없다.
- /order/cancel
  - order cancel의 경우, br 값에 에러가 있으면 실행되지 말아야 하기 때문에 br.hasErrors()시, redirectAttributes에 error message를 담아서 redirect 하도록 한다.

# 결과

<table style="margin: 0 auto;">
  <tr>
    <td><img src="/images/2025-07-12-ohGoodFood_refactor3/1.jpg" style="display: block; width : 200pt;" /></td>
    <td><img src="/images/2025-07-12-ohGoodFood_refactor3/2.jpg" style="display: block; width : 200pt;" /></td>
    <td><img src="/images/2025-07-12-ohGoodFood_refactor3/3.png" style="display: block; width : 200pt;" /></td>
  </tr>
</table>