
---
## 목차

1. [시작에 앞서](#51-시작에-앞서)
2. [검색을 위한 스펙](#52-검색을-위한-스펙)
3. [스프링 데이터 JPA를 이용한 조회 기능](#53-스프링-데이터-JPA를-이용한-조회-기능)
4. [리포지터리/DAO에서 스펙 사용하기](#54-리포지터리/DAO에서-스펙-사용하기)
5. [스펙 조합](#55-스펙-조합)
6. [정렬 지정하기](#56-정렬-지정하기)
7. [페이징 처리하기](#57-페이징-처리하기)
8. [스펙 조합을 위한 스펙 빌더 클래스](#58-스펙-조합을-위한-스펙-빌더-클래스)
9. [동적 인스턴스 생성](#59-동적-인스턴스-생성)
10. [하이버네이트 @Subselect 사용](#510-하이버네이트-@Subselect-사용)
---
## 5.1 시작에 앞서

**CQRS**란 명령 모델과 조회 모델을 분리하는 패턴이다.
**명령 모델**은 **상태를 변경하는 기능**을 구현할 때 사용하고, **조회 모델**은 **데이터를 조회하는 기능**을 구현할 때 사용한다.

도메인 모델은 상태를 변경하기 때문에 주로 명령 모델을 사용한다.

---
## 5.2 검색을 위한 스펙

다양한 검색 조건을 조합해야 할 때 find 매서드를 매번 정의하기 보다 **스펙**을 사용할 수 있다. **스펙**은 애그리거트가 **특정 조건을 충족하는지 검사**할 때 사용되는 인터페이스이다.
```java
public interface Specification<T> {
	public boolean isStatisfiedBy(T agg);
}
```

리포지터리나 DAO는 검색 대상을 걸러내는 용도로 스펙을 사용한다.
```java
public class MemoryOrderRepository implements OrderRepository {
	//리포지토리가 모든 애그리거트를 보관하고 있을 경우
	public List<Order> findAll(Specification<Order> spec) {
		List<Order> allOrders = findAll();
		return allOrders.stream().filter(order -> spec.isSatisfiedBy(order))
								.toList();
	}
}

//스펙을 생성해서 리포지토리에 전달
Specification<Order> ordererSpec = new OrderSpec("madvirus");
List<Order>orders = orderRepository.findAll(ordererSpec);

//다음과 같이 생성됨
//DB에서 유연한 where절을 생성함
public class OrderSpec implements Specification<Order> {  
  
	private String orderer;  
	private String status;  
	  
	public OrderSpec(String orderer, String status) {  
		this.orderer = orderer;  
		this.status = status;  
	}  
	  
	@Override  
	public Predicate toPredicate(Root<Order> root, CriteriaQuery<?> query, CriteriaBuilder cb) {  
		List<Predicate> predicates = new ArrayList<>();  
		  
		if (orderer != null) {  
			predicates.add(cb.equal(root.get("orderer"), orderer));  
		}  
		  
		if (status != null) {  
			predicates.add(cb.equal(root.get("status"), status));  
		}  
		  
		return cb.and(predicates.toArray(new Predicate[0]));  
	}  
}

```

실제는 위와 같이 만들 수 없다. (모든 애그리거트 보관이 불가하고 성능에 issue 발생)

---
## 5.3 스프링 데이터 JPA를 이용한 조회 기능

```java
//스프링 데이터 JPA가 제공하는 Specification 인터페이스
//package, import 생략

public interface Specification<T> extends Serializable {
	//not, where, and, or 매서드 생량
	
	//T는 JPA 엔티티 타입을 의미
	@Nullable
	Predicate toPredicate(Root<T> root, CriteriaQuery<?> query, CriteriaBuilder cb);
}
```

- 엔티티 타입이 OrderSummary다.
- ordererId 프로퍼티 값이 지정한 값과 동일하다.
```java

//OrderSummary에 대한 검색 조건을 표현하는 클래스
public class OrdererIdSpec implements Specification<OrderSummary> {
	private String ordererId;
	
	public OrdererIdSpec(String ordererId) {
		this.ordererId = ordererId;
	}
	
	//CriteriaQuery<?> query는 정렬, 최적화, 그룹 바이등을 위해 넘겨받음
	@Override
	public Predicate toPredicate(Root<OrderSummary> root, 
								CriteriaQuery<?> query,
								CriteriaBuilder cb
	) {
		//ordererId 값이 생성자로 전달받은 ordererId와 동일한지 비교
		//즉 where orderer_id = ? 이걸 만든다는 의미
		//OrderSummary_ 클래스는 JPA 정적 메타 모델을 정의한 코드
		return cb.equal(root.get(OrderSummary_.ordererId), ordererId);
		
		//문자열을 쓰고 싶다면
		//return cb.equal(root.<String>get("ordererId"), ordererId);
	}
}

//메타 모델의 생김새
//문자열 대신 필드에 안전하게 접근하고자 생성 (문자열일 경우 오류 잡기가 어려움)
@StaticMetamodel(OrderSummary.class)  
public class OrderSummary_ {  
	public static volatile SingularAttribute<OrderSummary, Long> ordererId;  
}
```

여러개를 모아둘 수도 있다.
```java
public class OrderSummarySpecs{
	public static Specification<OrderSummary> ordererId(String ordererId) {
	
		//람다식을 이용한 객체 생성
		return (Root<OrderSummary> root, CriteriaQuery<?> query, CriteriaBuilder cb)
			->
			cb.equal(root.<String>get("ordererId"), ordererId);
	}
	
	public static Specification<OrderSummary> orderDateBetween(LocalDateTime from, LocalDateTime to) {
		return (Root<OrderSummary> root, CriteriaQuery<?> query, CriteriaBuilder cb)
			->
			cb.between(root.get(OrderSummary_.orderDate), from, to);
	}
}

//생성 시
Specification<OrderSummary> betweenSpec = OrderSummarySpecs.orderDateBetween(from, to);
```

---
## 5.4 리포지터리/DAO에서 스펙 사용하기

스펙을 충족하는 엔티티를 검색하기 위해서 findAll() 메서드를 사용한다.
```java
public interface OrderSummaryDao extends Repository<OrderSummary, String> {
	//spec 개체 생성 후 finAll() 매서드로 검색
	//Specification<OrderSummary> spec = new OrdererIdSpec("user1");
	//List<OrderSummary> results = orderSummaryDao.findAll(spec);
	List<OrderSummary> findAll(Specification<OrderSummary> spec);
}
```

---
## 5.5 스펙 조합

스펙을 조합하는 두 매서드는 **and**와 **or**가 있다.
```java
public interface Specification<T> extends Serializable {
	//not, where매서드 생략
	default Specification<T> and (@Nullable Specification<T> other) {...}
	default Specification<T> or (@Nullable Specification<T> other) {...}
	
	//T는 JPA 엔티티 타입을 의미
	@Nullable
	Predicate toPredicate(Root<T> root, CriteriaQuery<?> query, CriteriaBuilder cb);
}

Specification<OrderSummary> spec1 = OrderSummarySpecs.ordererId("user1");
Specification<OrderSummary> spec2 = OrderSummarySpecs.orderDateBetween(
	LocalDateTime.of(2022,1,1,0,0,0),
	LocalDateTime.of(2022,1,2,0,0,0);
);
Specification<OrderSummary> spec3 = spec1.and(spec2);
//혹은 OrderSummarySpecs.ordererId("user1").and(OrderSummarySpecs.orderDateBetween(from,to))
```

---
## 5.6 정렬 지정하기

2가지 방식을 사용해 정렬이 가능하다
- 매서드 이름에 OrderBy를 사용해서 정렬 기준 지정
- Sort를 인자로 전달

```java
public interface OrderSummaryDao extends Repository<OrderSummary, String> {
	List<OrderSummary> findByOrdererIdOrderByNumberDesc(String ordererId);
}
```

findByOrdererIdOrderByNumberDesc는 다음 조회 쿼리를 생성한다
- ordererId 프로퍼티 값을 기준으로 검색 조건 지정
- number 프로퍼티 값 역순으로 정렬

2개 이상도 가능하지만 이름이 길어진다는 단점이 있다. 이럴 경우는 Sort 타입을 사용한다.
```
findByOrdererIdOrderByNumberAscOrderDateDesc
```
```java
public interface OrderSummaryDao extends Repository<OrderSummary, String> {
	List<OrderSummary> findByOrdererId(String ordererId, Sort sort);
	List<OrderSummary> findAll(Specification<OrderSummary> spec, Sort sort);
}

Sort sort = Sort.by("number").ascending();
List<OrderSummary> results = orderSummaryDao.findByOrdererId("user1", sort);

//2개 이상의 경우
Sort sort1 = Sort.by("number").ascending();
Sort sort2 = Sort.by("orderDate").descending();
Sort sort = sort1.and(sort2);
```

---
## 5.7 페이징 처리하기

Pageable 타입을 이용한다.
```java
public interface MemberDataDao extends Repository<MemberData, String> {
	//PageRequest 객체를 통해 Pageable 생성
	List<MemberData> findByNameLike(String name, Pageable pageable);
}

//페이지 번호는 0부터 시작
PageRequest pageReq = PageRequest.of(1,10); //페이지 번호, 페이지에 들어갈 수
List<MemberData> user = memberDataDao.findByNameLike("사용자%", pageReq);
```

PageRequest와 Sort를 사용하여 정렬 순서를 지정할 수 있다.
```java
Sort sort = Sort.by("name").descending();
PageRequest pageReq = PageRequest.of(1,2,sort);
List<MemberData> user = memberDataDao.findByNameLike("사용자%", pageReq);
```

Page 타입을 이용하여 조건에 해당하는 전체 갯수도 구할 수 있다.
Pageable을 사용하는 메서드의 리턴 타입이 Page일 경우 목록 조회와 COUNT 쿼리도 실행해서 개수를 구한다.
```java
Pageable pageReq = PageRequest.of(2,3);
Page<MemberData> page = memberDataDao.findByBlocked(false, pageReq);
List<MemberData> content = page.getContent(); //조회 결과 목록
Long totalElements = page.getTotalElements(); //전체 개수
int totalPages = page.getTotalPages(); //전체 페이지 번호
int number = page.getNumber(); //현재 페이지 번호
int numberOfElements = page.getNumberOfElements(); //조회 결과 개수
int size = page.getSize(); //페이지 크기
```

findAll도 Pageable 사용이 가능하나 리턴 타입이 List면 COUNT 쿼리를 실행하지 않는다. (Page 여야만 리턴)
스펙을 사용하는 findAll의 경우 Pagealbe 타입을 사용하면 리턴 타입에 관계 없이 COUNT를 실행한다.

처음부터 N개의 데이터가 필요하다면 findFirst'N'~라는 함수를 사용할 수 있다. (first/top도 존재)

---
## 5.8 스펙 조합을 위한 스펙 빌더 클래스

스펙을 조합해야 하는 경우 스펙 빌더를 사용할 수 있다.
```java
Specification<MemberData> spec = Specification.where(null);
if(searchRequest.isOnlyNotBlocked()) {
	spec = spec.and(MemberDataSpecs.nonBlocked());
}
if(StringUtils.hasText(searchRequest.getName())) {
	spec = spec.and(MemberDataSpecs.nameLike(searchRequest.getName()));
}
List<MemberData> results = memberDataDao.findAll(spec, PageRequest.of(0,5));

//스펙 빌더를 사용
//가독성 증가
// and(), ifHasText(), ifTrue()가 있고 그 외에는 필요한 매서드를 추가해서 사용하면 된다
Specification<MemberData> spec = SpecBuilder.builder(MemberData.class)
	.ifTrue(searchRequest.isOnlyNotBlocked(),
		() -> MemberDataSpecs.nonBlocked())
	.ifHasText(searchRequest.getName(),
		() -> MemberDataSpecs.nameLike(searchRequest.getName()))
	.toSpec();
List<MemberData> results = memberDataDao.findAll(spec, PageRequest.of(0,5));
```

---
## 5.9 동적 인스턴스 생성

JPQL의 select문 등으로 통해 인스턴스를 생성할 수 있다.
```java
@Query("""
	select new com.myshop.order.query.dto.OrderView(
		o.number, o.state, m.name, m.id, p.name
	)
	....
""")

public class OrderView {
	....
	private final String number;
	private final OrderState state;
	private final String memberName;
	private final String memberId;
	private final String productName;
	
	public OrderView(
		OrderNo number,....
	) {
		this.number = number.getNumber();
	}
}
```

---
## 5.10 하이버네이트 @Subselect 사용

JPA 확장 기능으로 @Subselect를 제공한다.
(쿼리 결과를 @Entity로 매핑할 수 있는 기능)
```java
@Entity
@Immutable
@Subselect (
	"""
		select o.order_number as number,
		o.version, o.orderer_id, o.orderer_name,
		o.total_amounts, o.receiver_name, o.state, o.order_date,
		p.product_id, p.name as product_name
		from purchase_order o inner join order_line ol
		on o.order_number = ol.order_number
		cross join product p
		where
		ol.line_idx = 0
		and ol.product_id = p.product_id
	"""
)
@Synchronize({"purchase_order", "order_line", "product"})
public class OrderSummary {
	@Id
	private String number;
	private long version;
	@Column(name = "orderer_id")
	private String ordererId;
	@Column(name = "orderer_name")
	private String ordererName;
	...
	
	protected OrderSummary() {
	}
}
```

하이버네이트 전용 애너테이션 **@Immutable, @Subselect, @Synchronize**가 있다.
해당 태그를 사용 시 테이블이 아닌 쿼리 결과를 @Entity로 매핑할 수 있다.

@Subselect는 조회 쿼리를 값으로 갖는다. 뷰와 마찬가지로 @Subselect로 조회한 @Entity는 수정이 불가능하다. (이 때 수정하게 된다면 update 쿼리가 실행되고 매핑한 테이블이 없어 에러발생) 수정 관련 에러 방지를 위해 @Immutable을 사용한다.

```java
//purchase_order 테이블 조회
Order order = orderRepository.findById(orderNumber);
order.changeShippingInfo(newInfo);//상태 변경

//트랜잭션을 커밋하는 시점에 변경사항을 DB로 저장함. (현재 커밋 없음)

//변경 내역이 DB에 반영되지 않았는데 purchase_order 테이블에서 조회
List<OrderSummary> summaries = orderSummaryRepository.findByOrdererId(userId);
```
위와 같은 문제를 해결하기 위해 @Synchronize를 사용한다. 지정한 테이블 관련 변경이 존재하면 flush를 통해 변경 내역이 반영되게 한다.

@Subselect는 이름 그대로 query 절을 서브 쿼리 형태로 가지고 온다. from 절 내부 () 에 들어감
