---
title: "피움 서비스의 Querydsl 도입 이유"
description: "우아한테크코스 5기 피움 프로젝트를 진행하면서, 필터링 기능을 구현하기 위해 Querydsl 라이브러리를 도입한 과정을 정리한다."
date: 2023-08-14
update: 2023-08-14
tags:
  - Querydsl
---

![](.index_images/querydsl.png)

> 이 글은 우테코 피움팀 크루 '[하마드](https://github.com/rawfishthelgh)'가 작성했습니다.

## 개요
---
우아한테크코스 5기 피움 프로젝트를 진행하면서, 필터링 기능을 구현하기 위해 Querydsl 라이브러리를 도입한 과정을 정리한다.
## 문제상황
---
### 필터링 기능 도입
내가 담당한 파트는 아래와 같다.
>"사용자가 보유한 반려 식물의 관리 이력을 최신 순으로 조회하기" 

`반려 식물의 관리 이력` 은 `History` 라는 테이블 및 엔티티 객체에 저장되고 관리된다.

`History` 객체 및 테이블 형태는 아래와 같이 생겨먹었다.

**History 객체**
```java
@Entity
@Getter
@Table(name = "history")
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class History extends BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotNull
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "pet_plant_id", nullable = false)
    private PetPlant petPlant;

    @NotNull
    @Column(name = "event_date", nullable = false)
    private LocalDate date;

    @NotNull
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "history_category_id", nullable = false)
    private HistoryCategory historyCategory;

    @Valid
    @NotNull
    @Embedded
    private HistoryContent historyContent;
	...
```
**history 테이블**
![](https://velog.velcdn.com/images/rg970604/post/c71da5e7-4a11-4ced-84d9-4d81764abd29/image.png)

기존에는 기록의 카테고리에 상관없이 전체를 불러와 조회를 하는 형태였지만, 피움 서비스는 4차 스프린트 과정에서 

>원하는 카테고리(historyCategory) 조합에 따라서 해당 카테고리들에 해당하는 관리 기록(History)들만 조회해 온다

라는 기능을 추가하게 되었다.

`/history?petPlantId=1&page=0&size=20&filter=”lastwaterDate,flowerpot…”`
위 url의 쿼리 파라미터 중 filter에 해당하는 값들을 기준으로 필터링을 수행하는 것이다.

### 어떤 필터링 조건이냐에 따라 쿼리 형태가 어마무시하게 변화한다.

`History` 의 카테고리 목록은

`location, flowerpot, waterCycle, light, wind, lastWaterDate`

총 여섯 가지를 보유하고 있다.

이 카테고리를 활용한 필터링이 전혀 들어오지 않을 수도(0개), 모두 들어올수도(6개), 몇 개만 들어올 수도 있다(1~5개).

또한 개수가 같아도, 어떤 카테고리가 조합되느냐에 따라 쿼리 형태가 변화한다.

예를 들면 
```sql
// 필터링 조건이 location, flowerpot인 경우
where history.pet_plant_id = 반려식물ID 
        and history_category.history_type in (location,flowerpot)
        
// 필터링 조건이 flowerpot, waterCycle인 경우
where history.pet_plant_id = 반려식물ID
        and history_category.history_type in (flowerpot,waterCycle)
        
// 어쩌구저쩌구 수많은 경우
...
```
따라서, 계산해 봤을 때 
>6C0 + 6C1 + 6C2 + 6C3 + 6C4 + 6C5  + 6C6 
= 1 + 6 + 15 + 20 + 15 + 6 + 1 
= 64

총 64개의 경우의 수가 등장하게 된다 ;;

따라서 현재 이 문제를 쌩으로 해결하려면, 조합에 따라 총64개의 분기문을 작성해야 한다...

그런데 64개의 분기문을 작성하면 만들고 끝이 아니라, 필터링 조건이 하나만 늘어나도 엄청난 양의 분기문의 추가와 수정이 일어난다. 이럴 바엔 그냥 개발자를 그만 두는게 나을 수도 있다.

따라서! 피움은 동적 쿼리를 쉽게 해결할 수 있는 도구를 선택 및 도입하기로 했다.

## 대안분석 및 선택과정
---

java / spring에서 동적 쿼리를 해결하는 도구는 여럿 존재한다. 우리가 어떤 대안을 떠올렸는지와, 이를 도입하지 않은/도입한 이유를 간단하게 정리하면 아래와 같다.

### **Mybatis**

JdbcTemplate의 동적 쿼리 한계를 효율적으로 해결하기 위해 등장한 SQL Mapper 프레임워크이다. 하지만 우리는 SQL Mapper가 아닌 JPA라는 ORM을 사용하기에 navite한 sql 쿼리를 직접 작성하지 않으므로 도입하지 않았다.

### **JPA - Criteria Query**

JPA에서는 JPQL이라는 쿼리 기술을 사용하여 엔티티 객체를 대상으로 질의할 수 있는 기능을 제공한다. 그러나 JPQL도 sql mapper 처럼 String 형태로 작성하므로 컴파일 단계에서 오류를 잡아내기 힘든데, Criteria Query 클래스를 사용하면 자바 코드로 JPQL을 작성할 수 있어 타입 안전성을 제공한다. 또한 조건문을 활용하여 동적 쿼리를 처리할 수 있는 기능을 제공한다. 그럼 이 방법론을 써야겠다는 생각이 슬슬 들게 된다.

그러나...Criteria Query에 대한 레퍼런스를 찾아보니, 대부분의 대답은 이러했다.
> 쓰지 마세요 가독성 안좋습니다...

그래서 이게 어떤 형태를 갖는지 대충 찾아봤다. gpt 선생님에게 
`where history.pet_plant_id = 반려식물ID 
        and history_category.history_type in (타입 조건들...)`
코드 작성을 요청드리니 다음과 같은 코드 형태가 태어났다. 아래 코드는 그냥 Criteria Query가 이런 형태를 갖는구나 정도로 생각하자.

```java
public class DynamicCriteriaQueryExample {

    public static void main(String[] args) {
        EntityManager entityManager = ...; // EntityManager 생성 코드

        CriteriaBuilder criteriaBuilder = entityManager.getCriteriaBuilder();

        // Criteria Query 생성
        CriteriaQuery<History> criteriaQuery = criteriaBuilder.createQuery(History.class);
        Root<History> historyRoot = criteriaQuery.from(History.class);

        // 동적으로 생성되는 조건을 위한 리스트 생성
        List<Predicate> predicates = new ArrayList<>();

        // 동적 조건 추가
        int plantId = 반려식물ID; // 동적으로 설정되는 값
        predicates.add(criteriaBuilder.equal(historyRoot.get("petPlantId"), plantId));

        List<String> types =히스토리 타입들; // 동적으로 설정되는 값
        predicates.add(historyRoot.get("historyType").in(types));

        // 모든 동적 조건을 AND로 결합
        criteriaQuery.where(predicates.toArray(new Predicate[0]));

        // 결과 쿼리 실행
        List<History> resultList = entityManager.createQuery(criteriaQuery).getResultList();

        // 결과 처리 등의 로직
    }
}

```
딱 봐도 뭔가 코드가 더럽고, 어떤 sql이 탄생하는지 유추가 정말 힘들다. 위 코드를 보고 주석의 상세한 설명 없이 동적 where in 절이 탄생하는게 유추되는가? 코드 형태가 sql이 아닌 객체 중심적이다보니 발생한 문제이다.

### **Querydsl**
자바 코드로 jpql을 생성할 수 있는 Criteria Query의 이점을 유지하면서, sql과 비슷한 형태로 코드를 작성할 수 있는 장점까지 챙겨간 오픈소스 라이브러리다.

따라서 피움은 
>1. 가독성 : sql과 비슷한 형태의 자바 코드 작성
>2. 타입 안전성 : 자바 코드로 작성해 컴파일 단계에서 오류 파악 가능
>3. 생산성 : join, 서브쿼리, 집계함수 등 해당 서비스에서 필요한 복잡한 쿼리 형태를 간단하게 작성 가능한 api 제공
>4. 유지보수성 : 현재 사용하고 있는 라이브러리, 프레임워크를 변경하지 않고 개발 가능

동적 쿼리를 구현하는 과정에서, 다른 대안에 비해 위 네 가지의 가치를 창출할 수 있다는 결론을 내리고 Querydsl을 도입했다.

다음 포스팅에서는 Querydsl 라이브러리를 우리 서비스에 어떤 형태로 적용했는지 구현 과정을 설명하도록 하겠다.
