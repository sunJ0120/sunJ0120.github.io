---
title: "[신한 DS SW아카데미 / 1차 프로젝트] [BE] User main 리팩터링 하기"
layout: single
categories: [project]
tags : [shinhanDS, project, refactor]
toc: true
toc_sticky: true
toc_label: 목차
author_profile : true
permalink: /project01/usermain/BE/refactor
---

> ⭐ 해당 프로젝트의 경우, 존재하는 레퍼런스에서 약간 변형하여 팀 프로젝트로 제작 중인 신한DS 부트캠프 1차 미니 프로젝트입니다.   
> {: .notice--info}

# 🤔 Intro

- 현재 우리 팀의 경우, 최종 배포가 완료되어 개발이 끝난 상태이다.
- 그러나 짧은 시간 안에(3주) 개발을 진행하다보니, 프로젝트가 끝나고 돌아봤을때 개인적으로 깔끔하지 못하고 마음에 안 드는 로직이 왕왕 있었다..ㅠ
- 그러므로 프로젝트가 마무리 된 이 시점에서, 내가 짠 코드들을 더 깔끔하게 고칠 수 있다면 어떻게 해야할지 회고를 하며 리팩토링을 해보고자 한다.
- 이전에 짠 코드와 비교하면서, 어느 부분을 왜 바꿨는지를 상세히 알아 볼 것이다!
- 전체 프로젝트에 대한 플로우 정리는 리팩토링이 끝난후 따로 정리할 예정이다.

# 😀 Start!
## 목표
- 기존의 코드에서 리팩토링을 진행해서 더 나은 코드를 만들어보자!

## 화면명세
- 일단은, 기능의 플로우를 어느정도 알아야 리팩토링 코드 이해가 가능하므로 간단한 user main 화면 명세 및 요구사항들을 알아보자.

<img src="/images/2025-07-05-ohGoodFood_refactor1/user_main.png" style="display: block; margin: 0 auto;" />
- main 화면의 명세는 다음과 같다.
    - 검색어를 통한 검색이 가능하며, 검색시 “위치 정보”가 적용되지 않게끔 한다. (검색어에 해당하는 모든 지역을 검색 가능)
    - "예약 가능만" 등의 버튼을 통해 조건에 맞는 가게 필터링이 가능하다.
        - 예약 가능만 : 매진, 마감 제외
        - 오늘픽업 : pickup_start가 오늘
        - 내일픽업 : pickup_start가 내일
        - 음식 종류 : 4개의 카테고리 각각에 속한 것을 찾을 수 있다.
    - 가게 마감시, 상세 정보로 들어갈 수 없다. (오픈전)
    - main 화면에 있는 리스트, 지도에는 같은 필터가 적용된다. (즉, 연동되어 있다.)

<img src="/images/2025-07-05-ohGoodFood_refactor1/user_map.png" style="display: block; margin: 0 auto;" />
- main 화면 지도 tab의 명세는 다음과 같다.
    - 마감인 가게는 클릭할 수 없다는 것은 main과 동일하다.
    - “이 위치에서 검색” 버튼을 누를 경우, center값을 현재 지도와 같게 조정해서 해당 위치 안의 가게를 띄운다.
    - IP를 기준으로 하므로 현재 위치를 잡아오는데 약간의 오차값이 존재하기 때문에, 범위는 반경 4km 정도로 넓게 잡는다.

## DTO & Mapper
### 기존 코드
```java
import java.sql.Time;
import java.sql.Timestamp;
import java.util.List;

import lombok.Data;
import lombok.NoArgsConstructor;

/**
 * [DTO] MainStore
 *
 * - Main에 나오는 가게 정보 카드를 사용하기 위한 DTO 클래스
 * - 카드 안에 들어가는 모든 정보들을 한 번에 저장해서 사용한다.
 */

@Data
@NoArgsConstructor
public class MainStore{
    //Store table에서 가져오는 정보들
    private String store_id;
    private String store_name;
    private String store_menu;
    private String store_status;
    private String category_bakery;
    private String category_fruit;
    private String category_salad;
    private String category_others;
    private Time closed_at;
    private String confirmed;
    private Double latitude;
    private Double longitude;

    //Product table에서 가져오는 정보들
    private int product_no;
    private Timestamp pickup_start;
    private Timestamp pickup_end;
    private int origin_price;
    private int sale_price;
    private Timestamp reservation_end;
    private int amount;

    //Image table에서 가져오는 정보들
    private String store_img;

    //[추가 정보] DB에는 없는 추가 정보
    private PickupStatus pickup_status; //오늘픽업인지 내일 픽업인지 마감인지를 저장
    private List<String> category_list; //카테고리 리스트 저장
    private List<String> mainmenu_list; //store_menu -> 메인메뉴 리스트 저장

    //[추가 정보] 검색용 상세정보
    private String product_explain; //검색용, 상품 상세

    //[추가 정보] 세션 체크를 위한 정보
    private String user_id;
}
```
- 기존에는 이렇게, 각 화면 하나에서 필요한 데이터들을 다 필드로 선언하는 식으로 구현했다.
- 지금와서 회고해보면, 너무 객체 지향적이 아닌 데이터 지향적인 방식으로 개발한게 아닌가...하는 생각이 든다.
    - 우선 이렇게 개발하면, 무엇보다 엔티티가 어떤 관계를 맺고 있는지 알 수 없다... 지나치게 데이터 의존적이 되는 것이다. (즉, 반드시 테이블을 열어봐야 필드 체크가 가능하고 무엇보다 유지 보수가 너무 어려워진다.)

> 🚨 이 방식의 문제점 정리
> 1. 도메인의 개념과 멀어져서 비즈니스 로직이 흩어진다.
> 2. 컬럼이 추가·삭제될 때마다 DTO 클래스가 너무 커지고, 구별이 힘들어진다.
> 3. 나중에 특정 기능(예: 상품 상세 조회)에만 쓰이는 값까지 죄다 들고 다녀야 해서 관리가 너무 어려워진다.

### 개선 코드
```java
import java.util.List;

import lombok.*;

/**
 * [DTO] MainStore
 *
 * - Main에 나오는 가게 정보 카드를 사용하기 위한 DTO 클래스
 * - 카드 안에 들어가는 모든 정보들을 한 번에 저장해서 사용한다.
 * - @Data로 불필요한 애노테이션을 늘리는 것 보다는, 필요한 애노테이션만 더해주었다.
 */

@Getter
@Setter
@ToString
@Builder
@AllArgsConstructor //Builder와 테스트용 생성자
@NoArgsConstructor  // MyBatis 매핑용 기본 생성자
public class MainStoreDTO {
    //Store table에서 가져오는 정보들
    private Store store;
    //Product table에서 가져오는 정보들
    private Product product;
    //Image table에서 가져오는 정보들
    private Image image;

    //[추가 정보] DB에는 없는 추가 정보
    private PickupStatus pickup_status; //오늘픽업인지 내일 픽업인지 마감인지를 저장
    private List<String> category_list; //카테고리 리스트 저장
    private List<String> mainmenu_list; //store_menu -> 메인메뉴 리스트 저장

    //[추가 정보] 세션 체크를 위한 정보
    private String user_id;
}
```
- JOIN하는 테이블을 객체로 매핑하여 관계가 쉽게 보이도록 구성하였다.
- **이를 통해 데이터 의존적이 아닌, 객체 지향적인 설계에 가까워지도록 하였다.**
- @Data의 경우, 쓰이지 않는 것까지 같이 설정되기에 이를 지우고 필요한 애노테이션만 남겨두었다.

> - 참고로 데이터 안정성을 위해 @Setter는 지양하는 것이 좋으나, MyBatis의 경우 Setter로 데이터를 설정하기 때문에 이를 사용한다.
> - 그러나 test code 등에서는 Builder 패턴을 사용하는 것이 유리하므로, @Builder와 @AllArgsConstructor 도 추가해서 동시에 사용하도록 한다.

- 이렇게 바꿀경우, mapper에서 resultMap을 사용해 컬럼과 필드를 매핑해야 하기 때문에 mapper역시 수정이 필요하다.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- [refactor] main을 위한 전용 mapper -->
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http:/mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="kr.co.ohgoodfood.dao.UserMainMapper">
    <!-- id는 resultMap의 식별자, type은 결과를 채워넣을 DTO 클래스 -->
    <resultMap id="MainStoreDTO" type="kr.co.ohgoodfood.dto.MainStoreDTO">
        <!-- store 연관관계 매핑 -->
        <association property="store" javaType="kr.co.ohgoodfood.dto.Store">
            <id     column="store_id"          property="store_id"/>
            <result column="store_name"        property="store_name"/>
            <result column="store_status"      property="store_status"/>
            <result column="store_menu"      property="store_menu"/>
            <result column="category_bakery"   property="category_bakery"/>
            <result column="category_fruit"   property="category_fruit"/>
            <result column="category_salad"   property="category_salad"/>
            <result column="category_others"   property="category_others"/>
            <result column="closed_at"   property="closed_at"/>
            <result column="confirmed"   property="confirmed"/>
            <result column="latitude"   property="latitude"/>
            <result column="longitude"   property="longitude"/>
        </association>

        <!-- product 연관관계 매핑 -->
        <association property="product" javaType="kr.co.ohgoodfood.dto.Product">
            <id     column="product_no"          property="product_no"/>
            <result column="pickup_start"        property="pickup_start"/>
            <result column="pickup_end"      property="pickup_end"/>
            <result column="origin_price"   property="origin_price"/>
            <result column="sale_price"   property="sale_price"/>
            <result column="reservation_end"   property="reservation_end"/>
            <result column="amount"   property="amount"/>
            <result column="product_explain"   property="product_explain"/>
        </association>

        <!-- Image 연관관계 매핑 -->
        <association property="image" javaType="kr.co.ohgoodfood.dto.Image">
            <result column="store_img"   property="store_img"/>
        </association>
    </resultMap>

    <!-- selectAllStore mapper, main에서 이용한다. -->
    <select id="selectAllStore" parameterType="map" resultMap="MainStoreDTO">
        SELECT
            s.store_id AS store_id, s.store_name AS store_name, s.store_menu AS store_menu, s.store_status AS store_status, s.category_bakery AS category_bakery,s.category_fruit AS category_fruit, s.category_salad AS category_salad, s.category_others  AS category_others, s.closed_at AS closed_at, s.latitude AS latitude, s.longitude AS longitude,
            pf.product_no AS product_no, pf.pickup_start AS pickup_start,pf.pickup_end AS pickup_end, pf.origin_price AS origin_price, pf.sale_price AS sale_price, pf.product_explain AS product_explain, pf.reservation_end AS reservation_end, pf.amount AS amount,
            img.first_img AS store_img
        FROM Store s
        LEFT JOIN Product pf
            ON pf.store_id = s.store_id
            AND DATE(pf.reservation_end) = CURDATE()
        LEFT JOIN (
            SELECT
                store_id, MIN(store_img) AS first_img
            FROM Image
            GROUP BY store_id
        ) img ON img.store_id = s.store_id
        <where>
            s.confirmed = 'Y'
            <if test="filter.store_status != null">
                AND s.store_status = #{filter.store_status} AND pf.amount >= #{filter.amount}
            </if>
            <if test="filter.pickup_start != null">
                AND DATE(pf.pickup_start) = #{filter.pickup_start}
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
            <!-- 위도/경도 필터링: 반경 4km, 검색어가 들어갔을때는 위도 경도 필터링 안걸리게 한다. -->
            <if test="filter.latitude != null and filter.longitude != null and (filter.search == null or filter.search == '')">
                AND s.latitude  BETWEEN #{filter.latitude}  - 0.027 AND #{filter.latitude}  + 0.027
                AND s.longitude BETWEEN #{filter.longitude} - 0.034 AND #{filter.longitude} + 0.034
            </if>
            <!-- 가게 이름, 가게 대표 메뉴, 가게 상세 검색 필터링 추가 -->
            <if test="filter.search != null and filter.search != ''">
                AND (
                    s.store_name LIKE CONCAT('%', #{filter.search}, '%')
                    OR s.store_menu LIKE CONCAT('%', #{filter.search}, '%')
                    OR pf.product_explain LIKE CONCAT('%', #{filter.search}, '%')
                )
            </if>
            <!-- ORDER BY 순서 : 오픈, 매진, 마감순 정렬 -->
        </where>
        ORDER BY
            CASE WHEN s.store_status = 'Y' AND amount >  0 THEN 1
                WHEN s.store_status = 'Y' AND amount =  0 THEN 2
                ELSE 3
        END;
    </select>

    <!-- selectOneStoreByStoreId mapper, main map에서 이용한다. -->
    <select id="selectOneStoreByStoreId" parameterType="map" resultMap="MainStoreDTO">
        SELECT
            s.store_id AS store_id, s.store_name AS store_name, s.store_menu AS store_menu, s.store_status AS store_status, s.category_bakery AS category_bakery,s.category_fruit AS category_fruit, s.category_salad AS category_salad, s.category_others  AS category_others, s.closed_at AS closed_at,
            pf.product_no AS product_no, pf.pickup_start AS pickup_start,pf.pickup_end AS pickup_end, pf.origin_price AS origin_price, pf.sale_price AS sale_price, pf.product_explain AS product_explain, pf.reservation_end AS reservation_end, pf.amount AS amount,
            img.first_img AS store_img
        FROM Store s
        LEFT JOIN Product pf
            ON pf.store_id = s.store_id
            AND DATE(pf.reservation_end) = CURDATE()
        LEFT JOIN (
            SELECT
                store_id,
                MIN(store_img) AS first_img
            FROM Image
            GROUP BY store_id
        ) img ON img.store_id = s.store_id
        <where>
            s.confirmed = 'Y'
            <if test="filter.store_status != null">
                AND s.store_status = #{filter.store_status} AND pf.amount >= #{filter.amount}
            </if>
            <if test="filter.pickup_start != null">
                AND DATE(pf.pickup_start) = #{filter.pickup_start}
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
            <!-- 가게 id로 필터링해서 하나만 가져와야 하는 경우 -->
            AND s.store_id = #{filter.store_id};
        </where>
    </select>
</mapper>
```
- 다음과 같이 resultMap을 사용해서 의존성이 있는 table안에서 매핑하고, 해당 정보를 사용하면 DTO를 위에서 썼던 방식대로 사용할 수 있다.

> **간단한 코드 설명**
> - <resultMap id="MainStoreDTO" type="kr.co.ohgoodfood.dto.MainStoreDTO">
>   - resultMap의 이름 id는 MainStore, type은 MainStore DTO 타입에 매핑한다.
> - <association property="store" javaType="Store">
>   - javaType="Store" : class 이름을 의미한다.
>   - property="store" : 우리가 MainStore 안에 선언한 프로퍼티 이름이다.
> - <id column="store_id" property="store_id"/>
>   - db의 컬럼과 현재 변수명을 일치시켜 주는 것이다. 
>   - 우리팀의 경우, 편의상 snake case를 그대로 이용하기로 했어서 똑같이 매핑해주면 된다.

```java
import kr.co.ohgoodfood.dto.MainStoreDTO;
import kr.co.ohgoodfood.dto.UserMainFilter;
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Param;

import java.util.List;

/**
 * UserMainMapper
 *
 * 사용자 main 페이지에서 사용하는 mapper interface
 */

@Mapper
public interface UserMainMapper {
    /**
     * 사용자 메인 화면 영역에 표시할 가게 목록을 조회
     *
     * @param userMainFilter   필터링 정보가 들어간 DTO (카테고리, 픽업 날짜, 스토어 오픈 여부, 위도, 경도, [검색] (가게 이름, 가게 대표 메뉴, 가게 상세))
     * @return                 필터 적용된 MainStore 리스트
     */
    List<MainStoreDTO> selectAllStore(@Param("filter") UserMainFilter userMainFilter);

    /**
     * 사용자 지도 영역에 핀을 클릭했을때 가게 정보 조회
     * 고른 가게의 id로 select 하는 것이므로, 위도 경도 정보나 검색 필터링 값들은 필요없다.
     *
     * @param userMainFilter   필터링 정보가 들어간 DTO (카테고리, 가게 id, 픽업 날짜, 스토어 오픈 여부, [검색] (가게 이름, 가게 대표 메뉴, 가게 상세))
     * @return                 필터 적용된 MainStore 요소
     */
    MainStoreDTO selectOneStoreByStoreId(@Param("filter") UserMainFilter userMainFilter);
}
```

- MyBatis에서는 mapper interface에 id와 같은 이름을 가진 메서드를 똑같이 명시해줘야 하므로, 다음과 같이 만들어 준다.
- 주석은 각 DTO안에 무엇이 들어가고, 무엇을 기준으로 어떤 return 값이 나오는지를 좀 더 명확히 볼 수 있도록 구성했다.

### [추가] FilterDTO
```java
/**
 * [DTO / ENUM] PickupStatus
 *
 * - Y,N 으로 들어오는 타입들을 매핑하기 위한 ENUM CLASS
 */

public enum YesNo {
    Y, N
}
```

```java
import jakarta.validation.constraints.*;
import lombok.*;

import java.time.LocalDate;

/**
 * [DTO] UserMainStoreFilter.java
 *
 * - Main에서 가게 정보를 사용할때, filtering을 위한 dto
 * - ajax 요청에서 들어가는 모든 정보를 여기에 넣어서 구성한다.
 * - @Data로 불필요한 애노테이션을 늘리는 것 보다는, 필요한 애노테이션만 더해주었다.
 */

@Getter
@Setter
@ToString
@Builder
@AllArgsConstructor //Builder와 테스트용 생성자
@NoArgsConstructor  // MyBatis 매핑용 기본 생성자
public class UserMainFilter {
    //지도에서 사용하기 위한 store_id
    private String store_id;

    //카테고리 modal 토글
    @NotNull(message = "category_bakery는 필수값입니다")
    private YesNo category_bakery;

    @NotNull(message = "category_fruit는 필수값입니다")
    private YesNo category_fruit;

    @NotNull(message = "category_salad는 필수값입니다")
    private YesNo category_salad;

    @NotNull(message = "category_others는 필수값입니다")
    private YesNo category_others;

    //예약 가능만 토글
    @NotNull(message = "store_status는 필수값입니다")
    private YesNo store_status;

    @PositiveOrZero(message = "amount는 0 이상이어야 합니다")
    private int amount;

    //오늘예약 & 내일 예약 토글
    @FutureOrPresent(message = "pickupStart는 오늘 또는 이후 날짜여야 합니다")
    private LocalDate pickup_start;

    //[추가] 검색어
    private String search;

    //위도, 경도
    @DecimalMin(value = "-90.0", message = "latitude 범위는 -90 ~ 90 입니다")
    @DecimalMax(value = "90.0",  message = "latitude 범위는 -90 ~ 90 입니다")
    private Double latitude;

    @DecimalMin(value = "-180.0", message = "longitude 범위는 -180 ~ 180 입니다")
    @DecimalMax(value = "180.0",  message = "longitude 범위는 -180 ~ 180 입니다")
    private Double longitude;
}
```
- main의 MyBatis 동적 쿼리에서 쓰이는 filter dto이다.
- filter dto는 화면에 뿌리는 용도가 아닌, filter 버튼으로 걸러내기 위한 용도이므로 필드를 하나하나 명시하고, 이상값 방지를 위해 검증 애노테이션을 붙여주었다.
    - 원래는 검증 애노테이션을 엔티티 dto에도 붙여주는게 맞으나, 같이 사용하는 엔티티는 최대한 건들지 않으려고 하고 있어서 filterDTO에만 넣어 주었다.
- 이를 통해 이상 로직을 통해서 혹은 모종의...여러 이유로 DB에 이상값이 들어왔을 경우에도 대응이 가능하다.

```java
<!-- 예외처리 검증 애노테이션 -->
<dependency>
	<groupId>org.hibernate.validator</groupId>
	<artifactId>hibernate-validator</artifactId>
	<version>8.0.0.Final</version>
</dependency>
<dependency>
	<groupId>org.glassfish</groupId>
	<artifactId>jakarta.el</artifactId>
	<version>4.0.2</version>
</dependency>
```
- Spring framework에서 예외 처리를 하기 위해서는 다음의 의존성을 추가하면 된다.


## Service
- 서비스 로직의 경우, 특히 에러 처리와 null 데이터 값 처리를 제대로 안했어서 main 화면으로 이동이 불가능했던 치명적인 버그가 발생했었다.
- 그렇기에, 좀 더 심혈을 기울여서 리팩토링을 진행했다.

### 기존 코드
```java
/**
 * UsersServiceImpl.java - UsersService interface 구현체
 *
 * @see UsersService - 세부 기능은 해당 클래스인 UsersServiceImpl에 구현한다.
 * 의존성 주입은 생성자 주입으로 구성한다.
 */

@Slf4j
@RequiredArgsConstructor
@Service
public class UserServiceImpl implements UsersService{
    private final UserMapper userMapper;
    private final AwsS3Config awsS3Config;

    /**
     * 메인 화면에 뿌릴 DTO리스트를 가져오는 method
     *
     * @param userMainFilter : 필터링을 위한 객체가 담겨있다.
     * @return               : mainStoreList (MainStore DTO의 리스트 객체)
     */
    @Override
    public List<MainStore> getMainStoreList(UserMainFilter userMainFilter) {
        List<MainStore> mainStoreList = userMapper.selectAllStore(userMainFilter);

        // 카테고리 이름과 pickup 상태를 저장
        for(MainStore mainStore : mainStoreList){
            PickupStatus pickupStatus = getPickupDateStatus(mainStore);
            mainStore.setPickup_status(pickupStatus == null ? PickupStatus.CLOSED : pickupStatus);
            mainStore.setCategory_list(getCategoryList(mainStore));
            mainStore.setMainmenu_list(StringSplitUtils.splitMenu(mainStore.getStore_menu(), "\\s*\\|\\s*"));
        }
        return mainStoreList;
    }

    /**
     * 지도에 표시할 가게 정보를 가져오는 method
     *
     * @param userMainFilter : 필터링을 위한 객체가 담겨있다. main에서 사용하는걸 그대로 사용한다
     * @return               : mainStore
     */
    //selectOneStoreByStoreId
    @Override
    public MainStore getMainStoreOne(UserMainFilter userMainFilter){

        MainStore mainStore = userMapper.selectOneStoreByStoreId(userMainFilter);

        mainStore.setPickup_status(getPickupDateStatus(mainStore));
        mainStore.setCategory_list(getCategoryList(mainStore));
        mainStore.setMainmenu_list(StringSplitUtils.splitMenu(mainStore.getStore_menu(), "\\s*\\|\\s*"));

        return mainStore;
    }

    /**
     * 사용자가 가진 북마크 리스트를 가져오는 method
     *
     * @param user_id           : 현재 세션에 접속한 사용자 id
     * @return                  : bookmarkList (Bookmark DTO의 리스트 객체)
     */
    @Override
    public List<Bookmark> getBookmarkList(String user_id){
        List<Bookmark> bookmarkList = userMapper.selectAllBookmark(user_id);

        // 여기에 카테고리 이름과 pickup 상태를 저장
        for(Bookmark bookmark : bookmarkList){
            bookmark.setPickup_status(getPickupDateStatus(bookmark));
            bookmark.setCategory_list(getCategoryList(bookmark));
            bookmark.setMainmenu_list(StringSplitUtils.splitMenu(bookmark.getStore_menu(), "/"));
        }

        return bookmarkList;
    }

    /**
     * LocalDate.now()로 오늘픽업, 내일픽업, 매진, 마감 상태를 판별합니다.
     *
     * @param mainStore          : 판별이 필요한 데이터가 담긴 객체
     * @return                   : PickupStatus ENUM 객체
     */
    @Override
    public PickupStatus getPickupDateStatus(MainStore mainStore) {
        LocalDate today = LocalDate.now();

        // [마감] - store_status = N
        if("N".equals(mainStore.getStore_status())){
            return PickupStatus.CLOSED;
        }else{
            if(mainStore.getPickup_start() == null){
                return PickupStatus.CLOSED;
            }
            LocalDate pickupDate = mainStore.getPickup_start().toLocalDateTime().toLocalDate();
            // [매진] - amount = 0
            if(mainStore.getAmount() == 0){
                return PickupStatus.SOLD_OUT;
            }else{
                // [오늘픽업] 현재 날짜와 같음
                if (pickupDate.isEqual(today)) {
                    return PickupStatus.TODAY;
                }
                // [내일픽업] 현재 날짜 + 1과 같음
                // else if (pickupDate.isEqual(today.plusDays(1))) {
                //     return PickupStatus.TOMORROW;
                // }
                else {
                    return PickupStatus.TOMORROW;
                }
            }
        }
        // throw new IllegalStateException();
    }

    /**
     * LocalDate.now()로 오늘픽업, 내일픽업만을 판별합니다.
     * Orders 페이지에서는 마감,매진 값은 필요 없기 때문에, 이것만을 판별하는 로직을 따로 만듭니다.
     *
     * @param userOrder        : 판별이 필요한 데이터가 담긴 객체
     * @return                 : PickupStatus ENUM 객체
     */
    @Override
    public PickupStatus getOrderPickupDateStatus(UserOrder userOrder) {
        LocalDate today = LocalDate.now();
        LocalDate pickupDate = userOrder.getPickup_start().toLocalDateTime().toLocalDate();

        // [오늘픽업] 현재 날짜와 같음
        if (pickupDate.isEqual(today)) {
            return PickupStatus.TODAY;
        }
//        // [내일픽업] 현재 날짜 + 1과 같음
//        if (pickupDate.isEqual(today.plusDays(1))) {
//            return PickupStatus.TOMORROW;
//        }
        //불안정 하긴 하지만, 이틀 뒤가 pick_up start인 경우가 없기 때문에 나머진 다 내일 픽업
        return PickupStatus.TOMORROW;
    }

    /**
     * |(구분자) 구분은 확장성을 위해 프론트 단에 위임
     * 서버에서는 리스트에 담아서 보내도록 한다.
     *
     * @param mainStore          : 판별이 필요한 데이터가 담긴 객체
     * @return                   : 카테고리 이름이 담긴 List
     */
    @Override
    public List<String> getCategoryList(MainStore mainStore) {
        List<String> category_list = new ArrayList<>();

        if(mainStore.getCategory_bakery().equals("Y")){
            category_list.add("빵 & 디저트");
        }

        if(mainStore.getCategory_fruit().equals("Y")){
            category_list.add("과일");
        }

        if(mainStore.getCategory_salad().equals("Y")){
            category_list.add("샐러드");
        }

        if(mainStore.getCategory_others().equals("Y")){
            category_list.add("그 외");
        }

        return category_list;
    }
    //.......
```
- 보면, exception 처리가 제대로 안 되어 있고, null을 잡는 부분도 임시 처방을 해둔터라 로직이 깔끔하지 않다.
- 또한 기존에는 UserServiceImpl이라는 하나의 클래스에 유저 화면에서 사용하는 모든 서비스를 넣었었는데, 이는 정말 유지보수에 좋지 않은 방식이어서 이번에는 main에서 쓰는 기능을 따로 분리했다.
- 리팩토링시 가장 집중했던 부분은 exception을 처리하는 부분이었다. 이제 개선된 코드를 살펴보자.

### 개선 코드
```java
import kr.co.ohgoodfood.dao.UserMainMapper;
import kr.co.ohgoodfood.dto.*;
import kr.co.ohgoodfood.exception.InvalidPickupDataException;
import kr.co.ohgoodfood.util.StringSplitUtils;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.sql.Timestamp;
import java.time.LocalDate;
import java.util.ArrayList;
import java.util.List;

/**
 * UserMainServiceImpl.java - UsersMainService interface 구현체
 *
 * @see UserMainService - 세부 기능은 해당 클래스인 UserMainServiceImpl에 구현한다.
 * - 의존성 주입은 생성자 주입으로 구성한다.
 * - 스프링은 기본 빈 주입이 싱글톤이기 때문에, 따로 싱글톤 처리 없이 @Service로 해결한다.
 */

@Slf4j
@RequiredArgsConstructor
@Service
public class UserMainServiceImpl implements UserMainService {
    private final UserMainMapper userMainMapper;

    /**
     * 메인 화면에 뿌릴 DTO리스트를 가져오는 method
     *
     * @param userMainFilter    : 필터링을 위한 객체가 담겨있다. ()
     * @return                  : MainStoreDTOList (MainStore DTO의 리스트 객체)
     */
    @Override
    public List<MainStoreDTO> getMainStoreList(UserMainFilter userMainFilter) {
        List<MainStoreDTO> mainStoreList = userMainMapper.selectAllStore(userMainFilter);

        // 카테고리 이름과 pickup 상태를 저장
        // 이상 데이터 값이 있어도 , 이는 로그에 남기고 정상 데이터들은 잘 보여주기 위해 continue 처리한다.
        for(MainStoreDTO mainStore : mainStoreList){
            PickupStatus pickup_status;
            try{
                //product가 없음, 마감 상태 getProduct().으로 접근할때 nullpointerException을 막기 위함이다.
                if(mainStore.getProduct() == null){
                    pickup_status = PickupStatus.CLOSED;
                }else{
                    pickup_status = getPickupDateStatus(
                            mainStore.getStore().getStore_status(),
                            mainStore.getProduct().getPickup_start(),
                            mainStore.getProduct().getAmount()
                    );
                }
            } catch (InvalidPickupDataException e){
                log.info("픽업 상태 계산 실패(storeId={}): {}",
                        mainStore.getStore().getStore_id(), e.getMessage());
                //이상 데이터 값의 경우, continue로 숨김처리 및 pickup_status 계산 안함
                continue;
            }
            mainStore.setPickup_status(pickup_status);
            mainStore.setCategory_list(getCategoryList(mainStore.getStore()));
            mainStore.setMainmenu_list(StringSplitUtils.splitMenu(mainStore.getStore().getStore_menu(), "\\s*\\|\\s*"));
        }

        return mainStoreList;
    }

    /**
     * 지도에 표시할 가게 정보를 가져오는 method
     *
     * @param userMainFilter    : 필터링을 위한 객체가 담겨있다. main에서 사용하는걸 그대로 사용한다
     * @return                  : MainStoreDTO
     */
    //selectOneStoreByStoreId
    @Override
    public MainStoreDTO getMainStoreOne(UserMainFilter userMainFilter){
        MainStoreDTO mainStore = userMainMapper.selectOneStoreByStoreId(userMainFilter);
        PickupStatus pickup_status;

        if(mainStore.getProduct() == null){
            pickup_status = PickupStatus.CLOSED;
        }else{
            //단일건의 경우, 따로 예외처리 하거나 null을 return 하지 않고 ControllerAdvice에서 처리하도록 한다.
            pickup_status = getPickupDateStatus(
                    mainStore.getStore().getStore_status(),
                    mainStore.getProduct().getPickup_start(),
                    mainStore.getProduct().getAmount()
            );
        }

        mainStore.setPickup_status(pickup_status);
        mainStore.setCategory_list(getCategoryList(mainStore.getStore()));
        mainStore.setMainmenu_list(StringSplitUtils.splitMenu(mainStore.getStore().getStore_menu(), "\\s*\\|\\s*"));

        return mainStore;
    }

    /**
     * LocalDate.now()로 오늘픽업, 내일픽업, 매진, 마감 상태를 판별합니다.
     *
     * @param store_status      : store가 현재 오픈 상태인지 판단
     * @param pickup_start      : store가 오늘 픽업인지, 내일 픽업인지를 판단
     * @param amount            : store가 매진인지 판단, nullable 이므로 컬렉션 객체로 만든다.
     *
     * @return                  : PickupStatus ENUM 객체
     */
    @Override
    public PickupStatus getPickupDateStatus(String store_status, Timestamp pickup_start, Integer amount) {
        //parameter가 null인경우, nullpointerException이 발생하므로, Integer로 감싼다.
        LocalDate today = LocalDate.now();

        // [마감] - store_status = N
        if(store_status.equals("N")){
            return PickupStatus.CLOSED;
        }else{
            LocalDate pickupDate;
            if (pickup_start == null) {
                throw new InvalidPickupDataException(store_status, pickup_start, amount);
            }
            //한번 null 위험 처리를 하고나면, 그 다음부터는 time_stamp 형식 예외 등의 로직만 잡으므로, 더 안전하게 체크가 가능하다.
            try{
                pickupDate = pickup_start.toLocalDateTime().toLocalDate();
            } catch (Exception e){
                //에러 로그를 보존한채 넘긴다.
                throw new InvalidPickupDataException(store_status, pickup_start, amount, e);
            }

            // [매진] - amount = 0
            if(amount == null || amount == 0){
                return PickupStatus.SOLD_OUT;
            }else{
                // [오늘픽업] 현재 날짜와 같음
                if (pickupDate.isEqual(today)) {
                    return PickupStatus.TODAY;
                } else if (pickupDate.isEqual(today.plusDays(1))) { //[내일픽업] 현재 날짜 + 1과 같음
                     return PickupStatus.TOMORROW;
                }
            }
        }
        throw new InvalidPickupDataException(store_status, pickup_start, amount); //custom exception 던지기
    }

    /**
     * |(구분자) 구분은 확장성을 위해 프론트 단에 위임
     * 서버에서는 리스트에 담아서 보내도록 한다.
     *
     * @param store             : store dto 내부에 있는 cateogory_ 값에 따라 list를 구현하기 위함이다.
     * @return                  : 카테고리 이름이 담긴 List
     */
    @Override
    public List<String> getCategoryList(Store store) {
        List<String> category_list = new ArrayList<>();

        if(store.getCategory_bakery().equals("Y")){
            category_list.add(StoreCategory.BAKERY.getDisplayName());
        }

        if(store.getCategory_fruit().equals("Y")){
            category_list.add(StoreCategory.FRUIT.getDisplayName());
        }

        if(store.getCategory_salad().equals("Y")){
            category_list.add(StoreCategory.SALAD.getDisplayName());
        }

        if(store.getCategory_others().equals("Y")){
            category_list.add(StoreCategory.ETC.getDisplayName());
        }

        return category_list;
    }
}
```
- 스프링은 기본이 싱글톤 빈 이므로, 따로 싱글톤으로 관리할 필요가 없기에 기존처럼 @Service 애노테이션만 붙여두었다.
- getPickupDateStatus 로직 같은 경우는, 기본적으로 N일때는 close로 가서 null값이 넘어오지 않는 것이 맞다. 정상 플로우에서는 가게가 오픈 상태인데 product가 없는 일은 존재하지 않기 때문이다.

> 📌 **그러나 내가 간과한 것이 있다. db에 항상 정상 플로우의 데이터만 들어오는 것이 아니라는 것이다.**
> - 서버가 한 번 중단 되었다가 다시 시작되어서 가게 상태 변경 기능이 스킵되거나, 모종의 이유로 db에 이상값이 들어갈 경우, 에러 처리가 되어 있지 않으면 nullable 변수에서 반드시 nullPointException이 발생한다.
> - 또한 우리 db의 경우, 특히 가게가 "마감"상태일때 product table 값 전체가 null인 경우의 수가 있어서 (오늘 아직 올린 상품이 없으므로, 오늘 기준 조인하기에 12시가 지난 기준, 오픈하지 않은 가게는 product가 없다.) 특히나 이 null 데이터를 잘 신경썼어야만 했다.

> ⭐ **해결책**
> - InvalidPickupDataException라는 커스텀 Exception을 구성
> - getPickupDateStatus() 내부에서 exception 처리를 할 뿐 아니라 메서드를 부르기 전 밖에서도 product 객체 null 체크를 해서, 메서드 파라미터에서 null 에러가 나지 않도록 구성
> - 또한, 정상 플로우가 아닌 데이터가 들어와서 에러가 발생할 경우, error log로 어떤 데이터가 이상값인지를 남겨서 추적이 쉽게 하였으며, continue를 통해 해당 이상 데이터를 제외하고 list에 담을 수 있도록 하였다.

- 예외 처리 외에 getCategoryList의 StoreCategory역시, 기존에는 단순 String이었으나 안전성과 확장 편의성을 위해 category를 ENUM 타입으로 따로 구성하여 지정하였다.

```java
import lombok.Getter;
import lombok.RequiredArgsConstructor;

/**
 * [DTO / ENUM] StoreCategory
 *
 * - 확장성 및 유지보수성을 위해 ENUM type으로 구성
 * - .displayName을 이용하면 지정한 string을 화면에서 사용할 수 있습니다.
 */

@Getter //display name을 쉽게 가져오기 위함
@RequiredArgsConstructor //주어진 한글 인자로 인스턴스 생성
public enum StoreCategory {
    BAKERY("빵 & 디저트"),
    FRUIT("과일"),
    SALAD("샐러드"),
    ETC("그 외");

    private final String displayName;

    @Override
    public String toString() {
        return displayName;
    }
}
```

## Exception
```java
import org.springframework.http.HttpStatus;

/**
 * ApplicationException.java
 *
 * 애플리케이션 전반에서 쓰이는 예외의 공통 베이스 클래스.
 * errorCode나 httpStatus 같은 공통 필드를 여기 정의하고, 개별 예외는 메세지만 넘겨주도록 한다.
 */
public abstract class ApplicationException extends RuntimeException {
    private final String errorCode;
    private final HttpStatus httpStatus;

    protected ApplicationException(String errorCode, String message, HttpStatus httpStatus) {
        super(message);
        this.errorCode = errorCode;
        this.httpStatus = httpStatus;
    }

    //예외 체이닝 생성자 추가
    protected ApplicationException(String errorCode, String message, HttpStatus httpStatus, Throwable cause) {
        super(message, cause);
        this.errorCode = errorCode;
        this.httpStatus = httpStatus;
    }

    public String getErrorCode() {
        return errorCode;
    }

    public HttpStatus getHttpStatus() {
        return httpStatus;
    }
}
```
- Exception은 공용으로 사용할 틀에 해당하는 ApplicationException과 하위 구현 Exception으로 구성하여 틀을 고정하도록 구성했다.
- ApplicationException은 생성자로 httpStatus, code, message를 가지며 Throwable cause를 받는 생성자가 있어 예외 메세지를 보존하는 것이 가능하도록 구성했다.

```java
import org.springframework.http.HttpStatus;

import java.sql.Timestamp;

public class InvalidPickupDataException extends ApplicationException {

    //ERROR_CODE 상수
    private static final String ERROR_CODE = "PICKUP_DATA_INVALID";

    public InvalidPickupDataException(String storeStatus, Timestamp pickupStart, int amount) {
        super(
                ERROR_CODE,
                formatMessage(storeStatus, pickupStart, amount),
                HttpStatus.INTERNAL_SERVER_ERROR
        );
    }

    //에러 메세지 보존
    public InvalidPickupDataException(String storeStatus, Timestamp pickupStart, int amount, Throwable cause) {
        super(
                ERROR_CODE,
                formatMessage(storeStatus, pickupStart, amount),
                HttpStatus.INTERNAL_SERVER_ERROR,
                cause
        );
    }

    private static String formatMessage(String storeStatus, Timestamp pickupStart, int amount) {
        return String.format(
                "픽업 데이터 상태가 올바르지 않습니다. DB를 확인해주세요: store_status=%s, pickup_start=%s, amount=%d",
                storeStatus, pickupStart, amount
        );
    }
}
```
- main에서 getPickupDateStatus시 사용하는 customException이다. 기본적으로 pickup_start 등이 null일 경우 발생하는 excepion이기 때문에 다음과 같이 구성하였다.

> - ERROR_CODE : "PICKUP_DATA_INVALID"로 정의
> - String.format : "픽업 데이터 상태가 올바르지 않습니다. DB를 확인해주세요: store_status=%s, pickup_start=%s, amount=%d"로 정의했다.

- 상위 구현 객체인 ApplicationException을 보면 알 수 있듯이, 에러 e를 보존할 수 있는 생성자와 e가 없는 생성자 두 가지로 구성하였다.

## Controller
### 기존 코드
```java

/**
 * UsersController
 *
 * 사용자 페이지 전용 기능을 처리하는 컨트롤러입니다.
 * - POST /user/signup	              : 사용자 회원가입 페이지
 * - GET  /user/main                  : 사용자 메인 화면 조회
 * - POST /user/filter/store          : AJAX 기반 가게 목록 필터링
 * - GET  /user/bookmark              : 해당 user_id가 가진 bookmark 목록 조회
 * - POST /user/bookmark/delete       : 해당하는 bookmark 삭제
 * - POST /user/bookmark/insert       : 해당하는 bookmark 추가
 * - GET  /user/main/orderList        : 유저가 가진 orderList 목록 조회
 * - POST /user/filter/order          : AJAX 기반 오더 목록 필터링
 * - POST /user/order/cancel          : 유저가 선택한 오더 주문 취소
 * - GET  /user/map/pin               : AJAX 기반 핀으로 선택한 스토어 fragment 조회
 * - GET  /user/mypage                : 유저 mypage 이동
 * - GET  /user/reviewList            : 하단 메뉴바 Review탭 이동시 전체 리뷰 목록 조회
 *
 */
@Controller
@RequestMapping("/user")
@Slf4j
@RequiredArgsConstructor
public class UsersController {
    private final UsersService usersService;
    private final CommonService commonService;
    private final PayService payService;

    // 지도 사용을 위한 앱키
    @Value("${kakao.map.appKey}")
    private String kakaoMapAppKey;

    /**
     * 사용자 메인 화면을 조회하고, 가게 목록을 뷰에 바인딩한다.
     *
     * @param userMainFilter 요청 파라미터와 바인딩되어 뷰로 전달되는 DTO
     * @param model          뷰에 전달할 데이터(Model)
     * @return               포워딩할 JSP 뷰 이름 ("users/userMain")
     */
    @GetMapping("/main")
    public String userMain(@ModelAttribute UserMainFilter userMainFilter,
                           Model model){

        List<MainStore> mainStoreList = usersService.getMainStoreList(userMainFilter);
        model.addAttribute("kakaoMapAppKey", kakaoMapAppKey);
        model.addAttribute("mainStoreList", mainStoreList);

        return "users/userMain";
    }

    /**
     * AJAX 필터링 결과에 따른 가게 목록을 조회하고 뷰 프래그먼트만 반환한다.
     *
     * @param userMainFilter JSON 바디로 전달된 필터 정보 (필터 DTO에 자동 매핑)
     * @param model          뷰에 전달할 데이터(Model)
     * @return               가게 카드 목록만 포함한 JSP 프래그먼트 ("users/fragment/userMainStoreList")
     */
    @PostMapping("/filter/store")
    public String filterStoreList(@RequestBody UserMainFilter userMainFilter,
                                  Model model){

        List<MainStore> mainStoreList = usersService.getMainStoreList(userMainFilter);
        model.addAttribute("mainStoreList", mainStoreList);
        // JSP fragment만 리턴
        return "users/fragment/userMainStoreList";
    }

    /**
     * map에서 pin 선택한 가게의 정보를 AJAX로 조회하고 뷰 프래그먼트만 반환한다.
     *
     * @param userMainFilter JSON 바디로 전달된 필터 정보 (필터 DTO에 자동 매핑)
     * @return               가게 정보를 포함한 JSP 프래그먼트 ("users/fragment/userMapPinStore")
     */
    @GetMapping("/map/pin")
    public String getMapPinStore(@ModelAttribute UserMainFilter userMainFilter,
                                 Model model){

        MainStore mainStore = usersService.getMainStoreOne(userMainFilter);
        model.addAttribute("mainStore", mainStore);

        //fragment return
        return "users/fragment/userMapPinStore";
    }
    //........
}
```
- Service 로직이 그랬듯이, 마찬가지로 user 페이지에서 사용하는 모든 기능이 한 컨트롤러에 있어서 너무 많은 책임을 한 클래스가 지고 있다는 문제점이 있다.
- 또한, 주석이 명확하지 않고 (어떤 정보가 어디에 들어가는지..), 필요없는 파라미터가 주입되고 있는 메서드도 있기에 이런 부분들을 위주로 리팩토링 작업을 진행했다.

### 개선 코드

```java
import jakarta.validation.Valid;
import kr.co.ohgoodfood.dto.*;
import kr.co.ohgoodfood.service.users.UserMainService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.validation.BindingResult;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.annotation.*;

import java.time.LocalDate;
import java.util.List;

/**
 * UsersController
 *
 * 사용자 메인 페이지 전용 기능을 처리하는 컨트롤러입니다.
 * - GET  /user/main           : 사용자 메인 화면 조회
 * - POST /user/filter/store   : AJAX 기반 가게 목록 필터링
 * - GET  /user/map/pin        : AJAX 기반 핀으로 선택한 스토어 fragment 조회
 */

@Controller
@RequestMapping("/user")
@Slf4j
@RequiredArgsConstructor
public class UserMainController {
    private final UserMainService userMainService;

    // 지도 사용을 위한 앱키
    @Value("${kakao.map.appKey}")
    private String kakaoMapAppKey;

    /**
     * 사용자 메인 화면을 조회하고, 가게 목록을 뷰에 바인딩한다.
     *
     * @param model          뷰에 전달할 데이터(Model)
     * @return               포워딩할 JSP 뷰 이름 ("users/userMain")
     */
    @GetMapping("/main")
    public String userMain(Model model){

        UserMainFilter userMainFilter = new UserMainFilter(); //초기에는 filter 값이 없으므로 빈값 생성
        List<MainStoreDTO> mainStoreList = userMainService.getMainStoreList(userMainFilter);
        model.addAttribute("kakaoMapAppKey", kakaoMapAppKey);
        model.addAttribute("mainStoreList", mainStoreList);

        return "users/userMain";
    }

    /**
     * AJAX 필터링 결과에 따른 가게 목록을 조회하고 뷰 프래그먼트만 반환한다.
     *
     * @param userMainFilter JSON 바디로 전달된 필터 정보 (카테고리, 예약 가능 여부, 픽업 날짜, 검색어, 위도&경도 필터링)
     * @param br             유효성 검증을 위한 바인딩 값들을 저장
     * @param model          뷰에 전달할 데이터(Model)
     * @return               가게 카드 목록만 포함한 JSP 프래그먼트 ("users/fragment/userMainStoreList")
     */
    @PostMapping("/filter/store")
    public String filterStoreList(@Valid @RequestBody UserMainFilter userMainFilter,
                                  BindingResult br,
                                  Model model){

        if(br.hasErrors()){
            for (FieldError fe : br.getFieldErrors()) {
                switch (fe.getField()) {
                    case "amount":
                        userMainFilter.setAmount(0); // 이상값일 경우 0으로 처리
                        break;
                    case "pickup_start":
                        userMainFilter.setPickup_start(LocalDate.now());
                        break;
                    case "category_bakery":
                        userMainFilter.setCategory_bakery(YesNo.N);  // 필터 적용을 막기 위해 N으로 처리
                    case "category_fruit":
                        userMainFilter.setCategory_bakery(YesNo.N);  // 필터 적용을 막기 위해 N으로 처리
                    case "category_salad":
                        userMainFilter.setCategory_salad(YesNo.N);  // 필터 적용을 막기 위해 N으로 처리
                    case "category_others":
                        userMainFilter.setCategory_others(YesNo.N);  // 필터 적용을 막기 위해 N으로 처리
                    case "store_status":
                        userMainFilter.setStore_status(YesNo.N); //이상값일 경우 마감처리
                    case "latitude" :
                        userMainFilter.setLatitude(37.5593799298988); //설정해둔 기본 위도로 지정
                    case "longitude" :
                        userMainFilter.setLongitude(126.922667641634); //설정해둔 기본 경도로 지정
                        break;
                }
            }
        }

        List<MainStoreDTO> mainStoreList = userMainService.getMainStoreList(userMainFilter);
        model.addAttribute("mainStoreList", mainStoreList);

        // JSP fragment만 리턴
        return "users/fragment/userMainStoreList";
    }

    /**
     * map에서 pin 선택한 가게의 정보를 AJAX로 조회하고 뷰 프래그먼트만 반환한다.
     * 이미 선택한 정보 안에서 화면에 띄울 정보만 있으면 되므로, 위&경도 필터링은 제외한다.
     *
     * @param userMainFilter JSON 바디로 전달된 필터 정보 (카테고리, 예약 가능 여부, 픽업 날짜, 검색어 필터링)
     * @param br             유효성 검증을 위한 바인딩 값들을 저장
     * @return               가게 정보를 포함한 JSP 프래그먼트 ("users/fragment/userMapPinStore")
     */
    @GetMapping("/map/pin")
    public String getMapPinStore(@Valid @ModelAttribute UserMainFilter userMainFilter,
                                 BindingResult br,
                                 Model model){

        if(br.hasErrors()){
            for (FieldError fe : br.getFieldErrors()) {
                switch (fe.getField()) {
                    case "amount":
                        userMainFilter.setAmount(0); // 이상값일 경우 0으로 처리
                        break;
                    case "pickup_start":
                        userMainFilter.setPickup_start(LocalDate.now());
                        break;
                    case "category_bakery":
                        userMainFilter.setCategory_bakery(YesNo.N);  // 필터 적용을 막기 위해 N으로 처리
                    case "category_fruit":
                        userMainFilter.setCategory_bakery(YesNo.N);  // 필터 적용을 막기 위해 N으로 처리
                    case "category_salad":
                        userMainFilter.setCategory_salad(YesNo.N);  // 필터 적용을 막기 위해 N으로 처리
                    case "category_others":
                        userMainFilter.setCategory_others(YesNo.N);  // 필터 적용을 막기 위해 N으로 처리
                    case "store_status":
                        userMainFilter.setStore_status(YesNo.N); //이상값일 경우 마감처리
                        break;
                }
            }
        }

        MainStoreDTO mainStore = userMainService.getMainStoreOne(userMainFilter);
        model.addAttribute("mainStore", mainStore);

        // JSP fragment만 리턴
        return "users/fragment/userMapPinStore";
    }
}
```
- /main
    - 기존과 달리, main에서는 main 뷰를 띄우고 바로 위치 정보를 filter로 얻고 fragment를 반환하는 로직으로 이뤄지기 때문에, 쓸데없는 값인 @ModelAttribute 값을 제거하였다.
    - service 단에서 같은 로직을 써야 하기 때문에 빈 UserMainFilter 값을 넘겨준다.
        - 🤔 솔직히, 이것 역시 쓸데없는 부분이라 마음에 살짝 안들긴 하는데… 더 좋은 방안이 생기면 수정사항으로 달아두겠다.
- /filter/store
    - 이 부분에는 filter DTO에 예외처리 로직이 추가된 만큼, BindingResult 값을 달아두었다
    - 이를 통해 필터링에 유효하지 않은 값이 들어올시, 기본값으로 매핑하도록 구성했다.
    - 각 기본 값의 이유들은 주석으로 명확하게 설명해두었다.
        - 🤔 “그외” 카테고리의 경우, 이상값이 들어왓을때 기본으로 이 값만 Y 처리해도 괜찮지 않을까…싶긴 한데 고민중이다.
- /map/pin
    - 이 부분에는 filter DTO에 예외처리 로직이 추가된 만큼, BindingResult 값을 달아두었다
    - 이미 filter에서 필터링된 값으로 띄워지므로, 화면 구성에 필요한 정보들만 filter 값에 넣어주면 된다.
    - 그러므로 위, 경도 값은 제외한다.