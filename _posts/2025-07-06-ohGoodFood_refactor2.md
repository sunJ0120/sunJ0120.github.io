---
title: "[신한 DS SW아카데미 / 1차 프로젝트] [BE] User bookmark 리팩터링 하기"
layout: post
categories: [project]
tags : [shinhanDS, project, refactor]
toc: true
toc_sticky: true
toc_label: 목차
author_profile : true
permalink: /project01/userbookmark/BE/refactor
---

> ⭐ 해당 프로젝트의 경우, 존재하는 레퍼런스에서 약간 변형하여 팀 프로젝트로 제작 중인 신한DS 부트캠프 1차 미니 프로젝트입니다.   
> {: .notice--info}

# 🤔 Intro

- main과 마찬가지로 내가 맡은 bookmark 도 리팩터링을 진행해보자!

# 😀 Start!
## 목표
- 기존의 코드에서 리팩토링을 진행해서 더 나은 코드를 만들어보자!

## 화면명세
- bookmark 화면의 화면 명세는 다음과 같다.
<img src="/images/2025-07-06-ohGoodFood_refactor2/1.png" style="display: block; margin: 0 auto;" />

1. 별을 이용해서 북마크 추가 삭제가 가능하다.
2. 마감, 오픈 여부가 보인다.
3. 카테고리와 대표메뉴가 나와있다.
4. 마감이면 빈칸, 오픈이면 픽업 시간이 나와있다.
5. 판매가와 원가가 나와있다.
6. 가게 대표 이미지가 존재한다.

<img src="/images/2025-07-06-ohGoodFood_refactor2/2.png" style="display: block; margin: 0 auto;" />
1. 다음과 같이 별을 눌러서 북마크가 삭제될 경우, 흐려진다.
2. 별을 다시 누르면 북마크를 다시 추가할 수 있다.

## DTO & Mapper
### 기존 코드
```java
import lombok.Data;
import lombok.NoArgsConstructor;

/**
 * [DTO] Bookmark
 *
 * - user가 가진 bookmark DTO
 * - Service로직을 사용하기 위해 MainStore를 상속받도록 구성한다.
 */

@Data
@NoArgsConstructor
public class Bookmark extends MainStore{
    //Bookmark table에서 가져오는 정보들
    private Integer bookmark_no;
    private String user_id;
}
```
- 마찬가지로 데이터 의존적으로 설계되어 있을 뿐 아니라,
- 기존 main에서 사용하는 서비스 로직을 사용하기 위해 main의 객체를 상속받는 식으로 설계되어 불필요한 정보가 너무 많이 포함된다는 치명적인 단점이 존재한다.

### 개선 코드
```java
import lombok.*;

import java.util.List;

/**
 * [DTO] BookmarkDTO
 *
 * - Bookmark에 나오는 가게 정보 카드를 사용하기 위한 DTO 클래스
 * - 카드 안에 들어가는 모든 정보들을 한 번에 저장해서 사용한다.
 * - @Data로 불필요한 애노테이션을 늘리는 것 보다는, 필요한 애노테이션만 더해주었다.
 */

@Getter
@Setter
@ToString
@Builder
@AllArgsConstructor //Builder와 테스트용 생성자
@NoArgsConstructor  // MyBatis 매핑용 기본 생성자
public class BookmarkDTO {
    //Store table에서 가져오는 정보들
    private Store store;
    //Product table에서 가져오는 정보들
    private Product product;
    //Image table에서 가져오는 정보들
    private Image image;
    //Bookmark에서 가져오는 정보들
    private Bookmark bookmark;

    //[추가 정보] DB에는 없는 추가 정보
    private PickupStatus pickup_status; //오늘픽업인지 내일 픽업인지 마감인지를 저장
    private List<String> category_list; //카테고리 리스트 저장
    private List<String> mainmenu_list; //store_menu -> 메인메뉴 리스트 저장
}
```
- 이를 통해 해당 DTO는 어떤 객체를 참조하는지 등을 좀 더 명확하게 알 수 있다.
- 마찬가지로, 이렇게 객체를 참조하는 방식으로 DTO를 구성할 경우, resultMap을 이용해야 한다.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- [refactor] bookmark 위한 전용 mapper -->
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http:/mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="kr.co.ohgoodfood.dao.UserBookmarkMapper">
    <resultMap id="BookmarkDTO" type="kr.co.ohgoodfood.dto.BookmarkDTO">
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

    <!-- Bookmark 연관관계 매핑 -->
        <association property="bookmark" javaType="kr.co.ohgoodfood.dto.Bookmark">
            <id    column="bookmark_no"   property="bookmark_no"/>
            <result column="user_id"   property="user_id"/>
        </association>
    </resultMap>
    <!-- selectAllBookmark mapper, bookmark에서 이용한다. -->
    <select id="selectAllBookmark" parameterType="map" resultMap="BookmarkDTO">
        SELECT
            s.store_id AS store_id, s.store_name AS store_name, s.store_menu AS store_menu, s.store_status AS store_status, s.category_bakery AS category_bakery,s.category_fruit AS category_fruit, s.category_salad AS category_salad, s.category_others  AS category_others, s.closed_at AS closed_at,
            pf.product_no AS product_no, pf.pickup_start AS pickup_start,pf.pickup_end AS pickup_end, pf.origin_price AS origin_price, pf.sale_price AS sale_price, pf.product_explain AS product_explain, pf.reservation_end AS reservation_end, pf.amount AS amount,
            img.first_img AS store_img
        FROM Bookmark b
        JOIN Store s ON s.store_id = b.store_id
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
            AND b.user_id = #{user_id}
            <!-- ORDER BY 순서 : 오픈, 매진, 마감순 정렬 -->
        </where>
        ORDER BY
            CASE
                WHEN s.store_status = 'Y' AND amount >  0 THEN 1
                WHEN s.store_status = 'Y' AND amount =  0 THEN 2
            ELSE 3
        END;
    </select>

    <!-- deleteBookmark mapper -->
    <delete id="deleteBookmark" parameterType="map">
        DELETE FROM Bookmark
            WHERE store_id = #{store_id}
            AND user_id = #{user_id};
    </delete>

    <!-- insertBookmark mapper -->
    <delete id="insertBookmark" parameterType="map">
        INSERT INTO Bookmark (user_id, store_id)
        VALUES (#{user_id}, #{store_id});
    </delete>
</mapper>
```
- main에서 리팩토링 했던 것과 같은 방식으로, resultMap을 이용해서 객체 지향적인 설계에 가깝도록 구성한다.
- 쿼리는 기존과 동일하게 사용한다.
- 마찬가지로, bookmark 전용 mapper를 분리한다.

```java
import kr.co.ohgoodfood.dto.BookmarkDTO;
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Param;

import java.util.List;

/**
 * UserBookmarkMapper
 *
 * 사용자 bookmark 페이지에서 사용하는 mapper interface
 */
@Mapper
public interface UserBookmarkMapper {
    /**
     * 사용자 북마크 화면에서 표시할 가게 목록을 조회
     *
     * @param user_id          조회 대상 user_id
     * @return                 특정 user가 가진 BookmarkDTO 리스트
     */
    List<BookmarkDTO> selectAllBookmark(String user_id);

    /**
     * 사용자의 특정 북마크를 삭제 처리
     *
     * @param user_id          조회 대상 user_id
     * @param store_id         user_id + store_id 조합으로 삭제
     * @return                 영향받은 행(row) 수
     */
    int deleteBookmark(@Param("user_id") String user_id,
                       @Param("store_id") String store_id);

    /**
     * 사용자 특정 북마크를 추가 처리
     *
     * @param user_id          조회 대상 user_id
     * @param store_id         북마크에 추가할 store 정보
     * @return                 영향받은 행(row) 수
     */
    int insertBookmark(@Param("user_id") String user_id,
                       @Param("store_id") String store_id);
}

```
- mapper interface는 다음과 같이 구성한다.
    - 각 매퍼에 id="selectOneStoreByStoreId" 아이디값으로 정의한 것이 여기서는 method의 이름이다.
- 주석에는 각 DTO안에 무엇이 들어가고, 무엇을 기준으로 어떤 return 값이 나오는지를 좀 더 명확히 볼 수 있도록 구성했다.

### [추가] FilterDTO
```java
import jakarta.validation.constraints.NotNull;
import lombok.*;

/**
 * [DTO] BookmarkFilter
 *
 * - user가 가진 bookmark를 삭제 & 추가하기 위해 필요한 정보 모음
 * - 유효성 검증을 위해 DTO로 따로 분리하였다.
 * - @Data로 불필요한 애노테이션을 늘리는 것 보다는, 필요한 애노테이션만 더해주었다.
 */

@Getter
@Setter
@ToString
@Builder
@AllArgsConstructor //Builder와 테스트용 생성자
@NoArgsConstructor  // MyBatis 매핑용 기본 생성자
public class BookmarkFilter {
    private int bookmark_no;

    private String user_id;
    @NotNull(message = "store_id는 필수값입니다")
    private String store_id; //insert 기능을 위해 추가.
}
```
- main에서처럼 마찬가지로 유효성 검사 값들을 달아준다.
- insert, delete시 id 값들은 무조건!!!! 있어야 하기 때문에 NotNull Anotation 처리한다.
    - 그러나 지금 내 구조에서, user_id는 controller에서 session단에서 빼서 주입하기 때문에 여기서는 binding 애노테이션을 제거해야 한다

## Service
- 다음으로는 서비스 로직을 고쳐보자.
- main에서 그랬듯이, 서비스 로직도 맞춰서 일부 변경하였다.

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
     * 북마크를 삭제하기 위한 기능이다.
     *
     * @param bookmarkFilter     : Bookmark 삭제시 필요한 정보값이 담긴 DTO
     * @return                   : 실행 결과 행 수에 따라 Boolean
     */
    @Override
    public boolean deleteUserBookMark(BookmarkFilter bookmarkFilter) {
        String user_id = bookmarkFilter.getUser_id();
        String store_id = bookmarkFilter.getStore_id();

        int cnt = userMapper.deleteBookmark(user_id, store_id);

        if (cnt == 1) {
            return true;
        }
        return false; //delete 실패!
    }

    /**
     * 북마크를 추가하기 위한 기능이다.
     *
     * @param bookmarkFilter     : Bookmark 삭제시 필요한 정보값이 담긴 DTO
     * @return                   : 결과 행 수에 따라 Boolean
     */
    @Override
    public boolean insertUserBookMark(BookmarkFilter bookmarkFilter) {
        String user_id = bookmarkFilter.getUser_id();
        String store_id = bookmarkFilter.getStore_id();

        int cnt = userMapper.insertBookmark(user_id, store_id);

        if (cnt == 1) {
            return true;
        }
        return false; //insert 실패!
    }
    //......
}
```
- main에서 존재했던 문제점과 비슷하게, 예외처리가 안 되어 있고 하나의 userService안에 모든 코드가 다 들어가 있다는 단점이 있다.
- 전체적인 리팩터링 방식은 main에서 진행했던 것과 동일한 방식으로 진행하였다.

### 개선 코드

```java
import kr.co.ohgoodfood.dto.BookmarkDTO;
import kr.co.ohgoodfood.dto.BookmarkFilter;

import java.util.List;

/**
 * UserBookmarkService interface
 * - UserBookmarkService 기능 틀 interface
 * - 유지 보수 및 확장 편의성을 위해 interface로 구성한다.
 */

public interface UserBookmarkService {
    //[Controller 로직] 북마크 Controller 연결 로직
    List<BookmarkDTO> getBookmarkList(String user_id);

    //[Controller 로직] 북마크 삭제 Controller 연결 로직
    boolean deleteUserBookMark(BookmarkFilter bookmarkFilter);

    //[Controller 로직] 북마크 추가 Controller 연결 로직
    boolean insertUserBookMark(BookmarkFilter bookmarkFilter);
}
```

```java
import kr.co.ohgoodfood.dao.UserBookmarkMapper;
import kr.co.ohgoodfood.dto.BookmarkDTO;
import kr.co.ohgoodfood.dto.BookmarkFilter;
import kr.co.ohgoodfood.dto.PickupStatus;
import kr.co.ohgoodfood.exception.InvalidPickupDataException;
import kr.co.ohgoodfood.util.StringSplitUtils;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.util.List;

/**
 * UserBookmarkServiceImpl.java - UserBookmarkService interface 구현체
 *
 * @see UserBookmarkService - 세부 기능은 해당 클래스인 UserBookmarkServiceImpl에 구현한다.
 * - 의존성 주입은 생성자 주입으로 구성한다.
 * - 스프링은 기본 빈 주입이 싱글톤이기 때문에, 따로 싱글톤 처리 없이 @Service로 해결한다.
 */

@Slf4j
@RequiredArgsConstructor
@Service
public class UserBookmarkServiceImpl implements UserBookmarkService {
    private final UserBookmarkMapper userBookmarkMapper;
    private final UserCommonService userCommonService;

    /**
     * 사용자가 가진 북마크 리스트를 가져오는 method
     *
     * @param user_id           : 현재 세션에 접속한 사용자 id
     * @return                  : bookmarkList (Bookmark DTO의 리스트 객체)
     */
    @Override
    public List<BookmarkDTO> getBookmarkList(String user_id){
        List<BookmarkDTO> bookmarkList = userBookmarkMapper.selectAllBookmark(user_id);

        // 여기에 카테고리 이름과 pickup 상태를 저장
        for(BookmarkDTO bookmark : bookmarkList){
            PickupStatus pickup_status;
            try{
                //product가 없음, 마감 상태 getProduct().으로 접근할때 nullpointerException을 막기 위함이다.
                if(bookmark.getProduct() == null){
                    pickup_status = PickupStatus.CLOSED;
                }else{
                    pickup_status = userCommonService.getPickupDateStatus(
                            bookmark.getStore().getStore_status(),
                            bookmark.getProduct().getPickup_start(),
                            bookmark.getProduct().getAmount()
                    );
                }
            } catch (InvalidPickupDataException e){
                log.info("픽업 상태 계산 실패(storeId={}): {}",
                        bookmark.getStore().getStore_id(), e.getMessage());
                //이상 데이터 값의 경우, continue로 숨김처리 및 pickup_status 계산 안함
                continue;
            }
            bookmark.setPickup_status(pickup_status);
            bookmark.setCategory_list(userCommonService.getCategoryList(bookmark.getStore()));
            bookmark.setMainmenu_list(StringSplitUtils.splitMenu(bookmark.getStore().getStore_menu(), "\\s*\\|\\s*"));
        }

        return bookmarkList;
    }

    /**
     * 북마크를 삭제하기 위한 기능이다.
     *
     * @param bookmarkFilter     : Bookmark 삭제시 필요한 정보값이 담긴 DTO
     * @return                   : 실행 결과 행 수에 따라 Boolean
     */
    @Override
    public boolean deleteUserBookMark(BookmarkFilter bookmarkFilter) {
        String user_id = bookmarkFilter.getUser_id();
        String store_id = bookmarkFilter.getStore_id();

        int cnt = userBookmarkMapper.deleteBookmark(user_id, store_id);

        if (cnt == 1) {
            return true;
        }
        return false; //delete 실패!
    }

    /**
     * 북마크를 추가하기 위한 기능이다.
     *
     * @param bookmarkFilter     : Bookmark 삭제시 필요한 정보값이 담긴 DTO
     * @return                   : 결과 행 수에 따라 Boolean
     */
    @Override
    public boolean insertUserBookMark(BookmarkFilter bookmarkFilter) {
        String user_id = bookmarkFilter.getUser_id();
        String store_id = bookmarkFilter.getStore_id();

        int cnt = userBookmarkMapper.insertBookmark(user_id, store_id);

        if (cnt == 1) {
            return true;
        }
        return false; //insert 실패!
    }
}
```
- main에서 그랬던 것처럼, 마찬가지로 interface와 그 구현체로 구성한다.
- 또한, 기존에 리팩토링에서 판별 로직들을 전부 userMainService에 넣었었는데, bookmark에서도 같은 로직을 사용하기 때문에 이를 UserCommonService로 분리하였다.
- commonService로 공통 로직을 분리하였으므로, userCommonService.getPickupDateStatus 이렇게 class에 method로 접근해서 사용하면 된다.

```java
import kr.co.ohgoodfood.dto.PickupStatus;
import kr.co.ohgoodfood.dto.Store;

import java.sql.Timestamp;
import java.util.List;

/**
 * UserCommonService interface
 * - UserCommonService 기능 틀 interface
 * - 유지 보수 편의성 및, DIP 원칙을 준수하기 위해 interface로 구성한다.
 * - bookmark와 main에서 동시에 사용하는 판별 로직을 common으로 분리
 */

public interface UserCommonService {
    //[판별 로직] 오늘 픽업, 내일 픽업, 마감 판별 연결 로직
    public PickupStatus getPickupDateStatus(String store_status, Timestamp pickup_start, Integer amount);

    //[판별 로직] 카테고리 List<String> 저장 로직
    public List<String> getCategoryList(Store store);
}
```
```java
import kr.co.ohgoodfood.dto.PickupStatus;
import kr.co.ohgoodfood.dto.Store;
import kr.co.ohgoodfood.dto.StoreCategory;
import kr.co.ohgoodfood.exception.InvalidPickupDataException;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.sql.Timestamp;
import java.time.LocalDate;
import java.util.ArrayList;
import java.util.List;

/**
 * UserCommonServiceImpl.java - UserCommonService interface 구현체
 *
 * @see UserCommonService - 세부 기능은 해당 클래스인 UserCommonServiceImpl에 구현한다.
 * - 공통적으로 사용하는 판별 로직(오늘 픽업 & 내일픽업 여부, categoryList 구현 등..)을 commonservice로 분리한다.
 * - 의존성 주입은 생성자 주입으로 구성한다.
 * - 스프링은 기본 빈 주입이 싱글톤이기 때문에, 따로 싱글톤 처리 없이 @Service로 해결한다.
 */

@Slf4j
@Service
public class UserCommonServiceImpl implements UserCommonService {
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
- 이를 통해 bookmark에서도 이를 깔끔하게 공용으로 사용할 수 있다.
- 그외 나머지 예외처리 로직은 main에서 처리했던 것과 동일하게 처리한다.

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

    //.......

    /**
     * 해당 user가 가진 북마크 리스트를 조회한다.
     *
     * @param userMainFilter 요청 파라미터와 바인딩되어 뷰로 전달되는 DTO
     * @param model          뷰에 전달할 데이터(Model)
     * @param session        현재 HTTP 세션(로그인된 사용자 정보)
     * @return               users/userBookmark.jsp로 포워딩
     */
    @GetMapping("/bookmark")
    public String userBookmark(@ModelAttribute UserMainFilter userMainFilter,
                               Model model,
                               HttpSession session){

        //세션에서 받아오는 로직
        Account loginUser = (Account) session.getAttribute("user");
        String user_id = loginUser.getUser_id();

        List<Bookmark> bookmarkList = usersService.getBookmarkList(user_id);
        model.addAttribute("bookmarkList", bookmarkList);

        return "users/userBookmark"; // /WEB-INF/views/users/userBookmark.jsp로 forwarding
    }

    /**
     * 해당 user가 가진 북마크 리스트 중, 특정 북마크를 삭제한다.
     *
     * @param bookmarkFilter bookMark delete에 필요한 필드 정보가 담긴 DTO
     * @param model          뷰에 전달할 데이터(Model)
     * @param session        현재 HTTP 세션(로그인된 사용자 정보)
     * @return               json 응답, 성공시 {"code" : 200} / 실패시 {"code" : 500}
     */
    @PostMapping("/bookmark/delete")
    @ResponseBody //json으로 code응답을 주기 위함이다.
    public Map<String,Integer> userBookmarkDelete(@RequestBody BookmarkFilter bookmarkFilter,
                                                  Model model,
                                                  HttpSession session){
        //세션에서 받아오는 로직
        Account loginUser = (Account) session.getAttribute("user");
        String user_id = loginUser.getUser_id();

        //bookmark를 위해 user_id 세팅
        bookmarkFilter.setUser_id(user_id);

        //delete bookmark 실행
        boolean result = usersService.deleteUserBookMark(bookmarkFilter);
        return Collections.singletonMap("code", result ? 200 : 500);
    }

    /**
     * 해당 user가 가진 북마클 리스트에서 삭제 된 것을 살리기 위함이다.
     *
     * @param bookmarkFilter bookMark delete에 필요한 필드 정보가 담긴 DTO
     * @param model          뷰에 전달할 데이터(Model)
     * @param session        현재 HTTP 세션(로그인된 사용자 정보)
     * @return               json 응답, 성공시 {"code" : 200} / 실패시 {"code" : 500}
     */
    @PostMapping("/bookmark/insert")
    @ResponseBody //json으로 code응답을 주기 위함이다.
    public Map<String,Integer> userBookmarkInsert(@RequestBody BookmarkFilter bookmarkFilter,
                                                  Model model,
                                                  HttpSession session){
        //세션에서 받아오는 로직
        Account loginUser = (Account) session.getAttribute("user");
        String user_id = loginUser.getUser_id();

        //bookmark를 위해 user_id 세팅
        bookmarkFilter.setUser_id(user_id);

        //delete bookmark 실행
        boolean result = usersService.insertUserBookMark(bookmarkFilter);
        return Collections.singletonMap("code", result ? 200 : 500);
    }
    //.....
}
```
- Controller의 문제점 역시 main에서 봤던 것과 동일하다.
- 한 클래스가 너무 많은 책임을 지지 않도록 Bookmark 전용 컨트롤러를 분리하고, BindingResult를 추가하는 방향으로 리팩터링을 진행했다.

### 개선 코드

```java
import jakarta.validation.Valid;
import kr.co.ohgoodfood.dto.*;
import kr.co.ohgoodfood.service.users.UserBookmarkService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.validation.BindingResult;
import org.springframework.web.bind.annotation.*;

import javax.servlet.http.HttpSession;
import java.util.Collections;
import java.util.List;
import java.util.Map;

/**
 * UserBookmarkController
 *
 * 사용자 bookmark 페이지 전용 기능을 처리하는 컨트롤러입니다.
 * - GET  /user/bookmark              : 해당 user_id가 가진 bookmark 목록 조회
 * - POST /user/bookmark/delete       : 해당하는 bookmark 삭제
 * - POST /user/bookmark/insert       : 해당하는 bookmark 추가
 */

@Controller
@RequestMapping("/user")
@Slf4j
@RequiredArgsConstructor
public class UserBookmarkController {
    private final UserBookmarkService userBookmarkService;

    /**
     * 해당 user가 가진 북마크 리스트를 조회한다.
     *
     * @param model         뷰에 전달할 데이터(Model)
     * @param session       현재 HTTP 세션(로그인된 사용자 정보)
     * @return              users/userBookmark.jsp로 포워딩
     */
    @GetMapping("/bookmark")
    public String userBookmark(Model model,
                               HttpSession session) {
        //세션에서 받아오는 로직
        Account loginUser = (Account) session.getAttribute("user");
        String user_id = loginUser.getUser_id();

        List<BookmarkDTO> bookmarkList = userBookmarkService.getBookmarkList(user_id);
        model.addAttribute("bookmarkList", bookmarkList);

        return "users/userBookmark"; // /WEB-INF/views/users/userBookmark.jsp로 forwarding
    }

    /**
     * 해당 user가 가진 북마크 리스트 중, 특정 북마크를 삭제한다.
     *
     * @param bookmarkFilter bookMark delete에 필요한 필드 정보가 담긴 DTO (store_id, user_id)
     * @param session        현재 HTTP 세션(로그인된 사용자 정보)
     * @return               json 응답, 성공시 {"code" : 200} / 실패시 {"code" : 500} / 유효성 검사 실패시 {"code" : 400}
     */
    @PostMapping("/bookmark/delete")
    @ResponseBody //json으로 code응답을 주기 위함이다.
    public Map<String, Integer> userBookmarkDelete(@Valid @RequestBody BookmarkFilter bookmarkFilter,
                                                   BindingResult br,
                                                   HttpSession session) {
        //세션에서 받아오는 로직
        Account loginUser = (Account) session.getAttribute("user");
        String user_id = loginUser.getUser_id();

        //bookmark를 위해 user_id 세팅
        bookmarkFilter.setUser_id(user_id);

        //유효성 검사 필드가 하나 이므로, 단일 구성
        if (br.hasFieldErrors("store_id")) {
            return Collections.singletonMap("code", 400);
        }
        
        //delete bookmark 실행
        boolean result = userBookmarkService.deleteUserBookMark(bookmarkFilter);
        return Collections.singletonMap("code", result ? 200 : 500);
    }

    /**
     * 해당 user가 가진 북마클 리스트에서 삭제 된 것을 다시 추가하기 위함이다.
     *
     * @param bookmarkFilter bookMark delete에 필요한 필드 정보가 담긴 DTO (store_id, user_id)
     * @param session        현재 HTTP 세션(로그인된 사용자 정보)
     * @return               json 응답, 성공시 {"code" : 200} / 실패시 {"code" : 500} / 유효성 검사 실패시 {"code" : 400}
     */
    @PostMapping("/bookmark/insert")
    @ResponseBody //json으로 code응답을 주기 위함이다.
    public Map<String, Integer> userBookmarkInsert (@Valid @RequestBody BookmarkFilter bookmarkFilter,
                                                    BindingResult br,
                                                    HttpSession session){
        //세션에서 받아오는 로직
        Account loginUser = (Account) session.getAttribute("user");
        String user_id = loginUser.getUser_id();

        //bookmark를 위해 user_id 세팅
        bookmarkFilter.setUser_id(user_id);

        //유효성 검사 필드가 하나 이므로, 단일 구성
        if (br.hasFieldErrors("store_id")) {
            return Collections.singletonMap("code", 400);
        }

        //delete bookmark 실행
        boolean result = userBookmarkService.insertUserBookMark(bookmarkFilter);
        return Collections.singletonMap("code", result ? 200 : 500);
    }
}
```
- /bookmark
    - main에서 그랬던것처럼, 여기서도 불필요한 modelAttrbute를 삭제한다.
- /bookmark/delete
    - filterDTO에 유효성 검사 애노테이션이 추가 되었으므로 유효성 검사가 필요하다.
    - store_id가 없을 경우, 삭제 로직을 진행할 수 없으므로 ajax에 400 code를 전달한다.
        - 화면 모달로 처리할 예정이다.
- /bookmark/insert
    - /bookmark/delete와 로직 흐름은 같다. 여기도 store_id 없으면 insert가 진행될 수 없으므로 마찬가지로 400에러 처리한다.
