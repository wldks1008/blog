<h2>들어가며</h2>
<p>최근에 코드를 실행하다가 신기한 상황을 발견했다. 서비스 로직에서 A 테이블의 데이터를 update하고 있었는데, 트랜잭션이 끝난 뒤 확인해보니 A의 히스토리 이력을 저장하는 B 테이블에도 데이터가 자동으로 쌓이고 있었다. 때문에 나는 당연히 아래와 같은 코드를 찾기 시작했다.</p>
<pre class="css"><code>historyRepository.save(...)</code></pre>
<p>그런데 아무리 찾아봐도 B 테이블에 위와 같이 명시적으로 직접 저장하는 로직은 없었다. 도대체 어떻게 이 데이터를 저장하는 걸까?</p>
<p>결론부터 말하면, 애플리케이션 서비스 코드가 히스토리를 직접 저장한 것이 아니었다. 해당 엔티티에 @Audited가 붙어 있었고, Hibernate Envers가 Hibernate의 flush 과정에 끼어들어 변경 이력을 자동으로 저장하고 있었다.</p>
<h2>Hibernate Envers</h2>
<blockquote>@Audited가 붙은 엔티티의 insert, update, delete 이력을 별도의 audit table에 자동으로 저장해주는 Hibernate 모듈</blockquote>
<h3>Envers는 무엇을 해결하려는 기술일까?</h3>
<p>애플리케이션을 운영하다 보면 현재 데이터만으로는 부족한 경우가 있다.</p>
<p>예를 들어 주문 테이블이 있다고 해보자. 해당 테이블에는 현재 주문 상태만 남는다. 그런데 서비스를 운영하며 오는 CS 문의라던가, 오류 대응이라던가 여러 가지 상황에 의해 현재 상태뿐 아니라 다음과 같은 질문에 답해야 할 때가 많다.&nbsp;</p>
<ul>
<li>이 주문은 언제 취소되었을까?</li>
<li>취소되기 전 상태는 무엇이었을까?</li>
<li>배송지는 언제 변경되었을까?</li>
<li>한 트랜잭션 안에서 주문과 결제가 함께 변경되었을까?</li>
</ul>
<p>이런 요구사항을 처리하기 위해서 보통 히스토리 테이블을 따로 만들어 데이터의 변경 이력을 관리하는 편이다.</p>
<pre class="java" id="code_1778166184461"><code>@Transactional
public void cancelOrder(Long orderId) {
    PurchaseOrder order = orderRepository.findById(orderId)
            .orElseThrow();

    order.cancel();

    orderHistoryRepository.save(
            PurchaseOrderHistory.from(order)
    );
}</code></pre>
<p>본론으로 돌아가서, 주문 히스토리를 관리하는 히스토리 테이블을 만들었다. 그리고 주문이 변경될 때마다 아래와 같이 서비스 코드에서 직접 히스토리 데이터를 저장하는 방식으로 구현을 완료했다.</p>
<p>이 방식은 명시적이라는 장점이 있지만, 반복 코드가 많아진다. 주문 수정, 주문 취소, 배송지 변경, 관리자 수정, 배치 수정 등 주문을 변경하는 코드가 여러 곳에 흩어져 있다면, 모든 변경 지점마다 히스토리 저장 코드를 넣어야 한다.</p>
<p>문제는 여기서 끝나지 않는다. 어떤 곳에서는 히스토리를 저장하고, 어떤 곳에서는 깜빡할 수 있다. 히스토리 저장 로직이 비즈니스 로직과 섞여서 코드가 복잡해질 수도 있다. 여러 엔티티가 한 트랜잭션에서 함께 변경될 때, 하나의 변경 묶음으로 추적하기도 쉽지 않다.</p>
<p>Hibernate Envers는 이런 문제를 해결하기 위한 Hibernate 모듈이다.</p>
<h3>Hibernate Envers와 Spring Data Envers는 다르다</h3>
<p>이름이 비슷해서 헷갈릴 수 있는데, 역할을 나눠서 이해하는 것이 좋다. 먼저 <b>Hibernate Envers</b>가 실제 변경 이력을 저장한다. 즉, 엔티티 변경을 감지하고, REVINFO와 audit table에 insert SQL을 추가하는 주체는 Hibernate Envers다.</p>
<p>반면 <b>Spring Data Envers</b>는 Spring Data JPA 환경에서 Envers 이력을 Repository 스타일로 조회할 수 있게 도와주는 기능에 가깝다. 예를 들어 Hibernate Envers만 사용해도 AuditReader로 이력을 조회할 수 있다.</p>
<pre class="java" id="code_1778166311860"><code>AuditReader auditReader = AuditReaderFactory.get(entityManager);

List&lt;Number&gt; revisions = auditReader.getRevisions(PurchaseOrder.class, orderId);

PurchaseOrder oldOrder = auditReader.find(
        PurchaseOrder.class,
        orderId,
        revisions.get(0)
);</code></pre>
<p>하지만 Spring Data JPA를 쓰고 있다면 다음처럼 Repository에서 이력을 조회하고 싶을 수 있다. <span style="color: #333333; text-align: start;">이때 사용하는 것이 Spring Data Envers다.<span>&nbsp;</span></span></p>
<pre class="java" id="code_1778166324525"><code>public interface PurchaseOrderRepository
        extends JpaRepository&lt;PurchaseOrder, Long&gt;,
                RevisionRepository&lt;PurchaseOrder, Long, Integer&gt; {
}</code></pre>
<p>Spring Data Envers를 사용하려면 spring-data-envers 의존성을 추가하고, Repository가 RevisionRepository를 확장하도록 만들며, 대상 엔티티에는 @Audited가 붙어 있어야 한다.&nbsp;</p>
<h4>REV와 REVTYPE</h4>
<p>먼저 <b>REV</b>는 revision 번호다. Envers에서 revision은 단순히 audit row 하나의 id라기보다는, <b>하나의 트랜잭션에서 발생한 감사 대상 변경 묶음</b>에 가깝다. 예를 들어 하나의 트랜잭션에서 주문과 결제 정보가 함께 변경되었다고 해보자. 만약 PurchaseOrder와 Payment가 모두 @Audited 대상이라면, 두 변경은 같은 트랜잭션 안에서 발생했으므로 같은 revision으로 묶일 수 있다. 따라서 revision을 보면 단순히 &ldquo;이 row가 몇 번째로 바뀌었는가?&rdquo;뿐 아니라, &ldquo;어떤 트랜잭션에서 어떤 엔티티들이 함께 바뀌었는가?&rdquo;라는 관점으로도 볼 수 있다.</p>
<p>&nbsp;</p>
<p><b>REVTYPE</b>은 변경 종류를 나타낸다. Hibernate 문서 기준으로 REVTYPE 값은 다음과 같다.</p>
<div>
<table border="1" style="border-collapse: collapse; width: 100%;">
<tbody>
<tr>
<td>0</td>
<td>INSERT</td>
</tr>
<tr>
<td>1</td>
<td>UPDATE</td>
</tr>
<tr>
<td>2</td>
<td>DELETE</td>
</tr>
</tbody>
</table>
</div>
<h4>Audit 테이블 설정</h4>
<p>기본 설정 값으로는 REVINFO라는 테이블이지만, Envers의 테이블 이름은 바꿀 수 있다.&nbsp;</p>
<p>먼저 엔티티별 audit table 이름은 @AuditTable로 지정할 수 있다.</p>
<pre class="java" id="code_1778402153246"><code>@Entity
@Audited
@Table(name = "orders")
@AuditTable(value = "order_history")
public class Order {

    @Id
    @GeneratedValue
    private Long id;

    private String status;

    private String receiverName;

    private LocalDateTime createdAt;

    private LocalDateTime updatedAt;
}</code></pre>
<p>이렇게 하면 기본 이름인 orders_AUD 대신 order_history 테이블에 이력이 저장된다. 전체 audit table suffix도 아래와 같은 설정을 통해 바꿀 수도 있다.</p>
<pre class="java" id="code_1778402179817"><code>spring.jpa.properties.org.hibernate.envers.audit_table_suffix=_history</code></pre>
<h3>save()가 없는데 왜 update가 발생할까?</h3>
<p>이제 다시 처음 코드로 돌아가보자.</p>
<pre class="java" id="code_1778399817951"><code>@Transactional
public void cancelOrder(Long orderId) {
    PurchaseOrder order = orderRepository.findById(orderId)
            .orElseThrow();

    order.cancel();

    // orderRepository.save(order) 없음
    // orderHistoryRepository.save(...) 없음
}</code></pre>
<p>여기에는 save()가 없다. 그런데도 원본 주문 테이블은 update되고, audit table에는 이력이 insert된다.</p>
<p>Hibernate는 변경 내용을 persistence context 안에 모아두었다가 flush 시점에 DB와 동기화한다. 즉, <span>트랜잭션 commit 직전 flush 발생하면, </span><span>Hibernate가 dirty checking 수행하게 되면서&nbsp;</span><span>변경된 필드를 감지하고 update SQL 생성되는 흐름인 것이다.</span></p>
<h4><span>audit table insert는 누가 할까?</span></h4>
<p>원본 테이블 update는 dirty checking으로 설명된다. 그러면 audit table insert는 누가 할까?</p>
<p>바로 Hibernate Envers다. Envers는 Hibernate의 이벤트 흐름에 붙어 있다. 엔티티 insert, update, delete가 발생하면 Envers listener가 해당 변경을 보고 audit work를 만든다.</p>
<p>트랜잭션 commit 시점에 flush가 발생하면 Hibernate는 PurchaseOrder가 변경되었음을 감지한다. 그 결과 원본 테이블에 update SQL이 만들어진다. 그리고 이 엔티티가 @Audited 대상이므로 Envers가 revision 정보를 만들고 audit table insert를 추가한다.</p>
<p>&nbsp;</p>
<p>정리하면, 전체적인 흐름은 아래와 같다.</p>
<ol>
<li>원본 엔티티 변경</li>
<li>Hibernate dirty checking</li>
<li>원본 테이블 update SQL 생성</li>
<li>Envers listener가 변경 감지</li>
<li>REVINFO insert 생성</li>
<li>audit table insert 생성</li>
<li>같은&nbsp;트랜잭션&nbsp;안에서&nbsp;함께&nbsp;반영</li>
</ol>
<h3>Envers의 단점과 주의할 점</h3>
<p>Envers는 편리하지만, 모든 상황에 만능은 아니다.</p>
<h4>1. audit table의 created_at을 잘못 해석하기 쉽다.</h4>
<p>Envers를 사용할 때 헷갈리는 지점이 있다. 바로 audit table의 created_at이다.</p>
<p>예를 들어 Order 엔티티에 createdAt, updatedAt이 있다고 하자. 그러면 audit table에도 created_at, updated_at 컬럼이 생길 수 있다. 이때 order_history.created_at을 보고 &ldquo;이 히스토리 row가 생성된 시간&rdquo;이라고 생각하면 안 된다. <b>audit table의 created_at은 history row 생성 시간이 아니다</b>. 원본 엔티티의 createdAt 값이다. 즉, 해당 revision 시점에 엔티티가 가지고 있던 createdAt 값이 복사된 것이다. 따라서 이력 발생 시각을 보려면 revision table의 timestamp를 봐야 한다.</p>
<p>따라서 변경 이력 화면에서 &ldquo;변경 발생 시간&rdquo;을 보여주고 싶다면 audit table의 created_at으로 정렬하면 안 된다. revision table을 join해서 revision timestamp를 봐야 한다.</p>
<h3>2. bulk update/delete는 audit이 누락될 수 있다.</h3>
<p>Envers는 Hibernate entity lifecycle과 event listener 흐름에 기대어 동작한다. 따라서 다음처럼 엔티티를 조회하고 값을 바꾸는 방식은 audit이 남는다.</p>
<pre class="java" id="code_1778402535947"><code>@Transactional
public void cancelOrder(Long orderId) {
    Order order = orderRepository.findById(orderId)
            .orElseThrow();

    order.cancel();
}</code></pre>
<p>하지만 JPQL bulk update는 다르다.</p>
<pre class="java" id="code_1778402548190"><code>@Modifying
@Query("""
    update Order o
       set o.status = :status
     where o.status = :targetStatus
""")
int bulkUpdateStatus(
        @Param("status") String status,
        @Param("targetStatus") String targetStatus
);</code></pre>
<p>이런 쿼리는 엔티티를 하나씩 영속 상태로 올려서 dirty checking을 수행하는 방식이 아니다. DB에 직접 update query를 실행하는 방식에 가깝다. 그 결과 원본 테이블은 바뀌었는데 audit table에는 이력이 남지 않을 수 있다.</p>
<p><a href="https://docs.hibernate.org/orm/6.0/userguide/html_single/#entity-inheritance" rel="noopener" target="_blank">Hibernate 문서</a>에서도 bulk operation은 entity lifecycle event dispatch 방식 때문에 Envers에서 지원되지 않으며, audit history가 불완전해질 수 있으므로 피해야 한다고 적혀있다.</p>
<p><figure class="imageblock widthContent"><span><img height="282" src="https://blog.kakaocdn.net/dn/bddHy2/dJMcajozggD/t659yf7k18PkJ4IHaZTJfK/img.png" width="1968" /></span></figure>
</p>
<p>따라서 Envers를 사용하는 프로젝트에서는 다음 코드를 조심해야 한다.</p>
<ul>
<li>@Modifying JPQL update/delete</li>
<li>CriteriaUpdate/Delete</li>
<li>native SQL update/delete</li>
<li>대량&nbsp;변경&nbsp;배치&nbsp;쿼리</li>
</ul>
<h3>3. audit table 데이터가 계속 늘어난다.</h3>
<p>Envers의 기본 전략은 audit row를 insert만 한다. 즉, Envers는 변경될 때마다 audit row를 추가한다. 변경이 많은 엔티티라면 audit table이 빠르게 커진다. audit table이 커지면 조회 성능, 인덱스 크기, 보관 정책을 함께 고민해야 한다. 단순히 @Audited만 붙이고 끝내면 나중에 audit table이 예상보다 커질 수 있다.</p>
<p>&nbsp;</p>