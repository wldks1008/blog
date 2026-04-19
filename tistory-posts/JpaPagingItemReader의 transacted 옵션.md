<h2>JpaPagingItemReader</h2>
<p>Spring Batch에서 대용량 데이터를 처리할 때 가장 먼저 부딪히는 문제는 &ldquo;데이터를 어떻게 안정적으로, 그리고 효율적으로 읽어올 것인가&rdquo;이다. 이때 등장하는 대표적인 Reader가 바로 <b>JpaPagingItemReader</b>다. 이름 그대로 JPA를 기반으로 데이터를 <b>페이징 방식으로 나눠서 읽어오는 Reader</b>다.<br />조금 더 쉽게 말하면, 데이터베이스에 있는 데이터를 한 번에 전부 가져오는 것이 아니라, <b>일정 개수씩 끊어서 가져오는 역할을 하는 컴포넌트</b>다. 예를 들어 100만 건의 데이터를 처리해야 한다고 했을 때, 한 번에 전부 메모리에 올리는 것이 아니라 1000건씩 나눠서 가져오는 방식이다. 이렇게 하면 메모리 사용량을 안정적으로 유지하면서도 대용량 처리가 가능해진다.</p>
<h3>필요성</h3>
<p>Spring Batch는 기본적으로 Reader &rarr; Processor &rarr; Writer 구조로 동작한다. 이 중 Reader는 &ldquo;데이터를 어디서 어떻게 가져올 것인가&rdquo;를 담당한다. 만약 Reader가 한 번에 모든 데이터를 가져오게 되면, <b>메모리 부족(OOM), GC 과부하, 처리 속도 저하</b>와 같은 문제가 발생할 수 있다.<br />JpaPagingItemReader는 이런 문제를 해결하기 위해 <b>페이징 기반 조회 전략</b>을 사용한다. 즉, 데이터를 쪼개서 가져오기 때문에 안정적인 처리가 가능하다.</p>
<h3>동작 방식</h3>
<p>JpaPagingItemReader의 핵심은 &ldquo;페이지 단위 조회&rdquo;다. 내부적으로는 다음과 같은 흐름으로 동작한다.</p>
<ol>
<li>첫 번째 페이지 조회 (offset 0, limit N)</li>
<li>데이터를 하나씩 Reader로 반환</li>
<li>해당 페이지 처리가 끝나면 다음 페이지 조회 (offset N, limit N)</li>
<li>이 과정을 반복</li>
</ol>
<p>여기서 중요한 점은, <b>JPA의 setFirstResult / setMaxResults를 이용해서 offset 기반 페이징을 수행한다는 것</b>이다.</p>
<p><figure class="imageblock widthContent"><span><img height="1140" src="https://blog.kakaocdn.net/dn/eB4Ppy/dJMcadaqcsy/K1EoXkPAiFu731kisG0iEk/img.png" width="2046" /></span><figcaption>출처: 카카오페이 기술 블로그</figcaption>
</figure>
</p>
<p>때문에 Limit Offset이 가지고 있는 태생적인 한계를 지니고 있다.</p>
<h3>Transacted 옵션</h3>
<p>JpaPagingItemReader를 보면 setTransacted(true/false) 옵션이 있는데, 해당&nbsp;옵션은 <b>Reader가 데이터를 읽을 때 트랜잭션을 걸고 읽을지를 설정</b>이다.&nbsp;</p>
<h4>1. transacted = true (Default)</h4>
<p>해당 값이 true일 경우 Reader가 JPA 조회를 할 때 자체 EntityTransaction을 열고 커밋하는 방식으로 동작한다.</p>
<p>즉, JpaPagingItemReader는 JPA 조회를 위해 별도의 EntityManager를 만들고, 기존 Spring managed transaction과 독립적인 새 트랜잭션 안에서 entity access를 수행한다.</p>
<p>&nbsp;</p>
<p>transacted=true인 경우 페이지마다 읽기용 트랜잭션이 새로 생성되고 종료되기 때문에, 해당 트랜잭션에서 조회한 엔티티를 계속 유지할 필요가 없다. 따라서 내부적으로 페이지 조회가 끝나면 영속성 컨텍스트를 clear하여 관리 중이던 엔티티들을 정리한다.&nbsp;</p>
<p><figure class="imageblock widthContent"><span><img height="338" src="https://blog.kakaocdn.net/dn/cDmkrE/dJMcabwW3bc/j3HlLZKbDRlZxw0dwQr3mK/img.png" width="626" /></span><figcaption>JpaPagingItemReader의 doReadPage() 구현</figcaption>
</figure>
</p>
<p>&nbsp;<br />내부 구현을 살펴보면 위와 같다. 보다 싶이, 새로운 페이지를 조회하기 전에 EntityManager를 flush하고 clear하여 영속성 컨텍스트를 정리한 뒤, 깨끗한 상태에서 현재 페이지 데이터를 조회하도록 구현이 되어 있다.</p>
<p>&nbsp;</p>
<p>하지만, 해당 옵션을 true로 설정할 경우, 주의해야할 점이 있다. ItemProcessor에서 Reader가 읽어온 엔티티를 변경하면, 그 변경이 <b>Writer가 아니라 Reader의 다음 페이지 조회 시점의 flush()로 DB에 반영될 수 있다.</b> 즉, &ldquo;Reader는 읽기만 한다&rdquo;고 생각했는데 transacted=true 때문에 Reader의 flush()가 예상치 못한 update를 발생시킬 수 있는 것이다.<br />&nbsp;</p>
<h4 style="color: #000000; text-align: start;">2. transacted = false</h4>
<p style="color: #000000; text-align: start;">반면, 해당 설정을 false로 설정하게 되면, Reader가 자체 트랜잭션을 만들지 않고 조회한다.</p>
<p>즉, Reader가 별도의 트랜잭션을 사용하지 않고, Step에서 사용 중인 영속성 컨텍스트를 함께 사용한다. 이 상태에서 clear()를 호출하면 Reader가 조회한 데이터뿐만 아니라, 현재 컨텍스트에서 관리 중이던 모든 엔티티가 함께 제거될 수 있다. 따라서 이러한 부작용을 방지하기 위해 전체를 초기화하는 대신, 조회한 엔티티만 선택적으로 분리하는 detach() 방식을 사용한다.</p>
<p><figure class="imageblock widthContent"><span><img height="500" src="https://blog.kakaocdn.net/dn/BBV2B/dJMcadhfeHv/nEwyDom8f748md8cM98pe0/img.png" width="746" /></span><figcaption>JpaPagingItemReader의 doReadPage() 구현</figcaption>
</figure>
</p>
<p>transacted가 false이면 Reader가 자체 EntityTransaction을 만들지 않는다. 따라서 Spring Batch 소스상 `!transacted` 분기에서는 쿼리 결과를 가져온 뒤 각 엔티티를 entityManager.detach(entity)로 분리해서 결과 목록에 넣는다. 즉 쿼리 결과로 가져온 엔티티를 JPA 관리 상태에서 떼어내서 넘겨주는 것이다.</p>
<p>&nbsp;</p>
<p>해당 옵션은 Reader는 정말 조회만 하게 만들고 싶을 때(Writer에서 저장 여부를 명시적으로 통제) 사용하면 유용하다고 한다. 다만, detach를 통해 영속성 컨텍스트의 관리에서 벗어났기 때문에 detached entity, lazy loading와 같은 예외 상황을 만나지 않도록 주의를 해야한다.</p>
<h2>영속성 엔티티와 Transacted 옵션</h2>
<p>최근에 배치 개발을 하다가, 정말 예상하지 못한 예외를 겪게 되었다. 해당 예외의 원인은 앞서 설명한 transacted 옵션과 연관이 있었는데 이에 대해서는 밑에서 좀 더 자세히 설명하도록 하겠다.</p>
<h3>DTO Projection 사용 시 발생하는 &ldquo;Non-entity object&rdquo; 예외</h3>
<p>transacted 옵션을 false로 사용하는 경우, JpaPagingItemReader는 조회된 결과를 영속성 컨텍스트에 계속 쌓아두지 않기 위해 각 엔티티를 detach() 하는 방식으로 관리한다. 이 방식은 메모리 사용을 줄이고, 불필요한 변경 감지를 방지하는 데 효과적이다.<br />하지만 이때 한 가지 주의해야 할 점이 있다. 바로 <b>DTO projection을 사용할 경우 예상치 못한 예외가 발생할 수 있다는 점</b>이다.&nbsp;<br /><br />예를 들어 다음과 같이 JPQL에서 DTO를 직접 생성하는 projection을 사용한다고 가정해보자.</p>
<pre class="kotlin"><code>SELECT new com.example.dto.MemberDto(m.id, m.name)
FROM Member m</code></pre>
<p>이 쿼리는 엔티티가 아닌 DTO 객체를 반환한다. 즉, 반환된 객체는 JPA가 관리하는 영속 엔티티가 아니라 단순한 자바 객체다.<br />문제는 transacted=false일 때 Reader 내부에서 수행되는 detach() 로직이다. detach()는 영속성 컨텍스트에 등록된 엔티티를 준영속 상태로 전환하는 메서드인데, DTO는 애초에 영속성 컨텍스트에 포함되지 않은 객체이기 때문에 이 대상이 될 수 없다.<br />따라서 DTO에 대해 detach()를 호출하게 되면, JPA는 해당 객체를 관리 대상 엔티티로 인식할 수 없고, 그 결과 Not an entity와 같은 예외가 발생하게 된다.<br />즉, JPA는 엔티티에 대해서만 영속성 컨텍스트를 통해 상태를 관리할 수 있으며, DTO는 이러한 관리 대상이 아니기 때문에 detach()의 적용 대상이 될 수 없다.</p>
<h3>어떻게 해결할 수 있을까?</h3>
<h4>DTO Projection을 포기하자.</h4>
<p>가장 단순한 방법은 DTO projection을 사용하지 않고 엔티티를 그대로 조회하는 것이다. 즉, Reader에서는 엔티티를 조회하고, 이후 Processor 단계에서 DTO로 변환하는 방식을 사용하는 것이다.<br />이 경우 조회된 객체는 영속성 컨텍스트의 관리 대상이 되므로, 이후 detach()가 호출되더라도 문제가 발생하지 않고 정상적으로 준영속 상태로 전환된다.<br />다만 이 방식은 한 가지 단점이 있다. 엔티티 전체를 조회하게 되므로, 실제로 필요한 컬럼만 선택적으로 조회하는 DTO projection에 비해 쿼리 효율이 떨어질 수 있다. 즉, 불필요한 컬럼까지 함께 조회하게 되어 커버링 인덱스를 활용하기 어렵거나, 전반적인 쿼리 성능 측면에서 불리해질 수 있다.</p>
<h4>다른 Reader 사용</h4>
<p>꼭 DTO projection을 사용해야 한다면 JpaPagingItemReader 대신 다른 Reader를 사용하는 것도 방법이다.대표적으로 JdbcPagingItemReader를 사용하는 방식이 있다.<br />JdbcPagingItemReader는 JPA의 영속성 컨텍스트를 사용하지 않고, JDBC를 기반으로 데이터를 조회한 뒤 RowMapper를 통해 원하는 형태로 매핑하는 구조를 가진다. 따라서 조회 결과가 엔티티가 아닌 DTO라 하더라도, 영속성 컨텍스트와의 충돌 없이 안전하게 사용할 수 있다.<br />특히 DTO projection이 필요한 경우, 필요한 컬럼만 선택적으로 조회할 수 있기 때문에 쿼리 효율 측면에서도 유리하다. 즉, 불필요한 컬럼을 포함한 엔티티 전체 조회가 아닌, 실제로 필요한 데이터만 조회할 수 있어 커버링 인덱스를 활용하기에도 적합하다.<br />다만 이 방식은 JPA의 장점을 사용할 수 없다는 단점이 있다. 예를 들어 JPQL을 활용한 추상화된 쿼리 작성이나, 엔티티 기반의 도메인 모델을 그대로 활용하기 어렵고, SQL을 직접 작성해야 하기 때문에 유지보수 측면에서 부담이 증가할 수 있다.<br />따라서 DTO projection이 반드시 필요하고, 쿼리 성능이 중요한 상황이라면 JdbcPagingItemReader와 같은 JDBC 기반 Reader를 사용하는 것이 더 적절한 선택이 될 수 있다.</p>
<h4>커스텀 Reader 구현</h4>
<p>커스텀 Reader를 직접 구현하여 detach() 로직을 제어하는 것이다. 앞서 살펴본 것처럼 transacted=false 환경에서 JpaPagingItemReader는 조회한 객체를 개별적으로 detach() 하는 방식으로 영속성 컨텍스트를 정리한다. 문제는 이 과정이 &ldquo;조회 결과가 엔티티일 것&rdquo;을 전제로 동작한다는 점이다. 따라서 DTO projection처럼 엔티티가 아닌 객체를 반환하는 경우에는 내부 detach() 로직과 충돌이 발생할 수 있다.<br />이러한 경우에는 Reader를 직접 구현하거나, 기존 JpaPagingItemReader를 확장하여 detach() 동작을 제어하는 방법을 고려할 수 있다. 예를 들어 조회 결과가 실제 엔티티인 경우에만 detach()를 수행하고, DTO인 경우에는 해당 로직을 건너뛰도록 처리하는 방식이다.<br />이 방식의 장점은 JPA 기반 조회를 유지하면서도, DTO projection 사용 시 발생하는 예외를 직접 우회할 수 있다는 점이다. 또한 프로젝트 상황에 맞게 Reader 동작을 세밀하게 제어할 수 있기 때문에, 단순히 엔티티 조회나 JDBC 기반 Reader로 전환하기 어려운 경우에는 하나의 대안이 될 수 있다.<br />다만 이 방법은 구현 복잡도가 높다는 단점이 있다. JpaPagingItemReader의 내부 동작 방식과 트랜잭션 처리 흐름, 영속성 컨텍스트 관리 방식까지 모두 이해한 상태에서 구현해야 하며, 잘못 구현할 경우 메모리 누수나 조회 일관성 문제로 이어질 수 있다. 또한 Spring Batch나 JPA 버전에 따라 내부 구현 차이에 영향을 받을 수 있어 유지보수 부담도 큰 편이다.<br />&nbsp;<br />&nbsp;</p>