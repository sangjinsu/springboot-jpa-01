# springboot-jpa-01
실전! 스프링 부트와 JPA 활용1 - 웹 애플리케이션 개발 



### 엔티티 연관관계 매핑

- 외래키가 있는 쪽에서 매핑 

```java
@Entity
@Getter
@Setter
public class Member {

    @Id
    @GeneratedValue
    @Column(name = "member_id")
    private Long id;

    private String name;

    @Embedded
    private Address address;

    @OneToMany(mappedBy = "member")
    private List<Order> orders = new ArrayList<>();
}
```

- Orders 에서 매핑

```java
@Entity
@Table(name = "orders")
@Getter
@Setter
public class Order {
    @Id
    @GeneratedValue
    @Column(name = "order_id")
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "member_id")
    private Member member;

    private LocalDateTime orderDate;

    @Enumerated(EnumType.STRING)
    private OrderStatus status;
    
    // 연관관계 매핑 
    public void setMember(Member member) {
        this.member = member;
        member.getOrders().add(this);
    }
}
```



### OneToMany ManyToOne

```java
@Entity
public class OrderItem {
    @Id
    @Column(name = "order_item_id")
    private Long id;

    @ManyToOne
    @JoinColumn(name = "item_id")
    private Item item;

    @ManyToOne
    @JoinColumn(name = "order_id")
    private Order order;

    private Integer orderPrice;
    private Integer count;

}
```

```java
@Entity
@Table(name = "orders")
@Getter
@Setter
public class Order {
    @Id
    @GeneratedValue
    @Column(name = "order_id")
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "member_id")
    private Member member;

    @OneToMany(mappedBy = "order")
    private List<OrderItem> orderItems = new ArrayList<>();

    @OneToOne
    private Delivery delivery;

    private LocalDateTime orderDate;

    @Enumerated(EnumType.STRING)
    private OrderStatus status; // 주문 상태 [ORDER, CANCEL]

}
```



### 엔티티 설계시 주의점

#### 엔티티에는 가급적 Setter를 사용하지 말자 

#### 모든 연관관계는 지연로딩으로 설정

- 즉시로딩 EAGER는 예측이 어렵고 어떤 SQL이 실행될지 추적하기 어렵다. 특히 JPQL 실행시 N+1 문제가 자주 발생한다 
- 실무에서 모든 연관관계는 지연로딩으로 설정해야 한다 
- 연관된 엔티티를 함께 DB를 조회해야 하면 fetch join 또는 엔티티 그래프 기능을 사용한다 
- OneToOne, ManyToOne 관계는 기본이 즉시로딩이므로 직접 지연로딩으로 설정해야 한다 

### 컬렉션은 필드에서 초기화 하자 

- null 문제에서 안전하다 
- 하이버네이트는 엔티티를 영속화할 때, 컬렉션을 감싸서 하이버네이트가 제공하는 내장 컬렉션으로 변경한다. 만약 `getOrders()`처럼 임의의 메서드에서 컬렉션을 잘못 생성하면 하이버네이트 내부 메커니즘에 문제가 발생할 수 있다 



### 연관관계 메서드

- 기본편에서 다시 알아보자 

```java
Order
    
    // 연관관계 메서드 
    public void setMember(Member member) {
        this.member = member;
        member.getOrders().add(this);
    }

    public void addOrderItem(OrderItem orderItem) {
        orderItems.add(orderItem);
        orderItem.setOrder(this);
    }

    public void setDelivery(Delivery delivery) {
        this.delivery = delivery;
        delivery.setOrder(this);
    }
```



### 애플리케이션 아키텍처

#### 계층형 구조 

- controller, web : 웹 계층
- service: 비즈니스 로직 트랜잭션 처리
- repository: JPA 를 직접 사용하는 계층, 엔티티 매니저 사용
- domain: 엔티티가 모여 있는 계층, 모든 계층에서 사용 



### 기술 설명 중요

```java
@Service
@Transactional(readOnly = true)
@RequiredArgsConstructor
public class MemberService {

    private final MemberRepository memberRepository;

//    @Autowired
//    public MemberService(MemberRepository memberRepository) {
//        this.memberRepository = memberRepository;
//    }

    /**
     * 회원 가입
     *
     * @param member
     * @return
     */
    @Transactional
    public Long join(Member member) {
        validateDuplicateMember(member);
        memberRepository.save(member);
        return member.getId();
    }


    private void validateDuplicateMember(Member member) {
        List<Member> findMembers = memberRepository.findByName(member.getName());
        if (!findMembers.isEmpty()) {
            throw new IllegalStateException("이미 존재하는 회원입니다");
        }
    }


    public List<Member> findMembers() {
        return memberRepository.findAll();
    }

    public Member findOne(Long memberId) {
        return memberRepository.findOne(memberId);
    }
}
```

- @Service
- @Transactional: 트랜잭션, 영속성 컨텍스트
  - readOnly=true: 데이터의 변경이 없는 읽기 전용 메서드에 사용, 영속성 컨텍스트를 플러시 하지 않으므로 약간의 성능 향상으로 읽기 전용에는 다 적용한다 

- @Autowired
  - 생성자 인젝션에 많이 사용
  - 생성자가 하나면 생략 가능

#### 생성자 주입

- 생성자 주입 방식 권장
- 변경 불가능한 안전한 객체 생성 가능
- 생성자가 하나면 @Autowired 생략 가능
- final 키워드 추가하면 컴파일 시점에 memberRepository 설정하지 않는 오류를 체크할 수 있다 



#### lombok

- @RequiredArgsConstructor

```java
Repository
@RequiredArgsConstructor
public class MemberRepository {
//    @PersistenceContext
//    private EntityManager em;
    
    private EntityManager em;
```

