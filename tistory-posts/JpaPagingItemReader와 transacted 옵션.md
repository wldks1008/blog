<h2>JpaPagingItemReader</h2>
<p>Spring Batch에서 대용량 데이터를 처리할 때 가장 먼저 부딪히는 문제는 &ldquo;데이터를 어떻게 안정적으로, 그리고 효율적으로 읽어올 것인가&rdquo;이다. 이때 등장하는 대표적인 Reader가 바로 <b>JpaPagingItemReader</b>다. 이름 그대로 JPA를 기반으로 데이터를 <b>페이징 방식으로 나눠서 읽어오는 Reader</b>다.</p>
<p>조금 더 쉽게 말하면, 데이터베이스에 있는 데이터를 한 번에 전부 가져오는 것이 아니라, <b>일정 개수씩 끊어서 가져오는 역할을 하는 컴포넌트</b>다. 예를 들어 100만 건의 데이터를 처리해야 한다고 했을 때, 한 번에 전부 메모리에 올리는 것이 아니라 1000건씩 나눠서 가져오는 방식이다. 이렇게 하면 메모리 사용량을 안정적으로 유지하면서도 대용량 처리가 가능해진다.</p>
<h3>필요성</h3>
<p>Spring Batch는 기본적으로 Reader &rarr; Processor &rarr; Writer 구조로 동작한다. 이 중 Reader는 &ldquo;데이터를 어디서 어떻게 가져올 것인가&rdquo;를 담당한다. 만약 Reader가 한 번에 모든 데이터를 가져오게 되면, <b>메모리 부족(OOM), GC 과부하, 처리 속도 저하</b>와 같은 문제가 발생할 수 있다.</p>
<p>JpaPagingItemReader는 이런 문제를 해결하기 위해 <b>페이징 기반 조회 전략</b>을 사용한다. 즉, 데이터를 쪼개서 가져오기 때문에 안정적인 처리가 가능하다.</p>
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
<h3 style="color: #000000; text-align: start;">Transacted 옵션</h3>
<p>JpaPagingItemReader를 보면 setTransacted(true/false) 옵션이 있는데, 해당&nbsp;옵션은 <b>Reader가 데이터를 읽을 때 트랜잭션을 어떻게 사용할지 결정하는 설정</b>이다.</p>
<h4>1. transacted = true (Default)</h4>
<p>해당 값이 true일 경우 Reader가 <b>자체 트랜잭션을 열어서 데이터를 조회</b>한다. 즉, 페이지마다 트랜잭션이 새로 열리며 조회 후 영속성 컨텍스트를 clear 한다.</p>
<p>문제는 이 경우 <b>페이지마다 읽는 시점이 달라질 수 있어서 데이터 일관성이 깨질 수 있다는 점</b>이다. 즉, 중간에 데이터가 변경되면 다음 페이지에서 다른 결과를 볼 수도 있다.<br /><br /></p>
<p>예를 들어보자. 최초의 테이블의 데이터 상태가 사진과 같다고 가정해보자.&nbsp;</p>
<p><figure class="imageblock widthContent"><span><img height="528" src="https://blog.kakaocdn.net/dn/JPCV5/dJMcagru9Tp/AnnpoK2k2EmsDbZ8kb91fK/img.png" width="1594" /></span></figure>
</p>
<p>status가 READY인 데이터를 조회한다고 가정했을 때, `SELECT ... LIMIT 2 OFFSET 0` 해당 쿼리를 실행하게 되면 결과 값은 id가 1,2 가 나오게 될 것이다.</p>
<p>하지만, 그 직후 id가 2인 데이터가 DONE 상태로 변경되는 쿼리가 실행되었다고 가정해보자. 이후에 `SELECT ... LIMIT 2 OFFSET 2` 쿼리를 실행하게 된다면 transacted가 true인 상황에서는 매번 새로운 트랜잭션을 실행하기 때문에 id가 4,5인 데이터가 조회되게 된다.</p>
<p>즉, id가 3인 데이터가 누락되는 문제가 발생하게 되는 것이다.</p>
<h4>2. transacted = false</h4>
<p>해당 설정을 false로 설정하게 되면, Step에서 관리하는 트랜잭션을 그대로 사용하게 된다. 즉, 하나의 트랜잭션 안에서 계속 읽으며 필요할 때 detach를 통해 엔티티를 분리한다. 쉽게 말해서 조회 기준 시점이 하나로 고정된다.</p>
<p>해당 방식은 <b>읽기 일관성을 유지하는 데 유리하다</b>. 특히 커서 기반처럼 &ldquo;쭉 읽어야 하는 상황&rdquo;에서는 더 안전하다.<br /><br />아까와 같은 상황에서, 최초로 `SELECT ... LIMIT 2 OFFSET 0` 해당 쿼리를 실행하게 되면 이 시점에 DB의 &ldquo;조회 기준 스냅샷&rdquo;이 잡힌다. 중간에 데이터가 Update 되어 변경이 발생하더라도 현재 트랜잭션(T)에서는 여전히 옛날 상태(스냅샷)를 보고 있다. 때문에 `SELECT ... LIMIT 2 OFFSET 2` 쿼리를 실행하게 되더라도 이 트랜잭션은 처음 시작할 때 본 스냅샷 기준으로 계속 조회하기 때문에 id가 3,4인 데이터가 조회되게 된다.</p>
<p><span style="color: #333333; text-align: start;">하지만, 해당 옵션의 경우에는<span>&nbsp;</span></span>전체 데이터를 다 읽을 때까지 트랜잭션 유지하고 있기 때문에 DB 리소스를 오래 잡고 있는다는 단점이 존재한다.</p>
<h4>영속성 엔티티</h4>
<p>JpaPagingItemReader가 엔티티를 조회하면, 조회된 객체는 보통 <b>영속성 컨텍스트</b>에 들어간다. 즉, JPA가 그 객체를 계속 추적하고 있다는 뜻이다. 영속 상태 엔티티는 다음 특징이 있다.</p>
<ul>
<li>JPA가 변경 여부를 추적한다</li>
<li>1차 캐시에 보관된다. -&gt; 때문에 같은 엔티티를 다시 조회하면 DB를 직접 찌르지 않고 캐시에서 줄 수도 있다.</li>
</ul>
<p><b>1. transacted=true</b></p>
<p>transacted=true인 경우 페이지마다 읽기용 트랜잭션이 새로 생성되고 종료되기 때문에, 해당 트랜잭션에서 조회한 엔티티를 계속 유지할 필요가 없다. 따라서 내부적으로 페이지 조회가 끝나면 영속성 컨텍스트를 clear하여 관리 중이던 엔티티들을 정리한다.&nbsp;</p>
<p><figure class="imageblock widthContent"><span><img height="338" src="https://blog.kakaocdn.net/dn/cDmkrE/dJMcabwW3bc/j3HlLZKbDRlZxw0dwQr3mK/img.png" width="626" /></span><figcaption>JpaPagingItemReader의 doReadPage() 구현</figcaption>
</figure>
</p>
<p>&nbsp;</p>
<p>내부 구현을 살펴보면 위와 같다. 보다 싶이, 새로운 페이지를 조회하기 전에 EntityManager를 flush하고 clear하여 영속성 컨텍스트를 정리한 뒤, 깨끗한 상태에서 현재 페이지 데이터를 조회하도록 구현이 되어 있다.</p>
<p>&nbsp;</p>
<p><b>2. transacted=false</b></p>
<p>반면 transacted 옵션이 false인 경우에는 Reader가 별도의 트랜잭션을 사용하지 않고, Step에서 사용 중인 영속성 컨텍스트를 함께 사용한다. 이 상태에서 clear()를 호출하면 Reader가 조회한 데이터뿐만 아니라, 현재 컨텍스트에서 관리 중이던 모든 엔티티가 함께 제거될 수 있다. 따라서 이러한 부작용을 방지하기 위해 전체를 초기화하는 대신, 조회한 엔티티만 선택적으로 분리하는 detach() 방식을 사용한다.</p>
<p><figure class="imageblock widthContent"><span><img height="500" src="https://blog.kakaocdn.net/dn/BBV2B/dJMcadhfeHv/nEwyDom8f748md8cM98pe0/img.png" width="746" /></span><figcaption>JpaPagingItemReader의 doReadPage() 구현</figcaption>
</figure>
</p>
<h2>예상치 못한 예외를 만나다.</h2>
<p>최근에 배치 개발을 하다가, 정말 예상하지 못한 예외를 겪게 되었다. 해당 예외의 원인은 앞서 설명한 transacted 옵션과 연관이 있었는데 이에 대해서는 밑에서 좀 더 자세히 설명하도록 하겠다.</p>
<h3>DTO Projection 사용 시 발생하는 &ldquo;Non-entity object&rdquo; 예외</h3>
<p>transacted 옵션을 false로 사용하는 경우, JpaPagingItemReader는 조회된 결과를 영속성 컨텍스트에 계속 쌓아두지 않기 위해 각 엔티티를 detach() 하는 방식으로 관리한다. 이 방식은 메모리 사용을 줄이고, 불필요한 변경 감지를 방지하는 데 효과적이다.</p>
<p>하지만 이때 한 가지 주의해야 할 점이 있다. 바로 <b>DTO projection을 사용할 경우 예상치 못한 예외가 발생할 수 있다는 점</b>이다.&nbsp;</p>
<p><br />예를 들어 다음과 같이 JPQL에서 DTO를 직접 생성하는 projection을 사용한다고 가정해보자.</p>
<pre class="kotlin" id="code_1775403984553"><code>SELECT new com.example.dto.MemberDto(m.id, m.name)
FROM Member m</code></pre>
<p>이 쿼리는 엔티티가 아닌 DTO 객체를 반환한다. 즉, 반환된 객체는 JPA가 관리하는 영속 엔티티가 아니라 단순한 자바 객체다.</p>
<p>문제는 transacted=false일 때 Reader 내부에서 수행되는 detach() 로직이다. detach()는 영속성 컨텍스트에 등록된 엔티티를 준영속 상태로 전환하는 메서드인데, DTO는 애초에 영속성 컨텍스트에 포함되지 않은 객체이기 때문에 이 대상이 될 수 없다.</p>
<p>따라서 DTO에 대해 detach()를 호출하게 되면, JPA는 해당 객체를 관리 대상 엔티티로 인식할 수 없고, 그 결과 Not an entity와 같은 예외가 발생하게 된다.</p>
<p>즉, JPA는 엔티티에 대해서만 영속성 컨텍스트를 통해 상태를 관리할 수 있으며, DTO는 이러한 관리 대상이 아니기 때문에 detach()의 적용 대상이 될 수 없다.</p>
<h3>어떻게 해결할 수 있을까?</h3>
<h4>DTO Projection을 포기하자.</h4>
<p>가장 단순한 방법은 DTO projection을 사용하지 않고 엔티티를 그대로 조회하는 것이다. 즉, Reader에서는 엔티티를 조회하고, 이후 Processor 단계에서 DTO로 변환하는 방식을 사용하는 것이다.</p>
<p>이 경우 조회된 객체는 영속성 컨텍스트의 관리 대상이 되므로, 이후 detach()가 호출되더라도 문제가 발생하지 않고 정상적으로 준영속 상태로 전환된다.</p>
<p>다만 이 방식은 한 가지 단점이 있다. 엔티티 전체를 조회하게 되므로, 실제로 필요한 컬럼만 선택적으로 조회하는 DTO projection에 비해 쿼리 효율이 떨어질 수 있다. 즉, 불필요한 컬럼까지 함께 조회하게 되어 커버링 인덱스를 활용하기 어렵거나, 전반적인 쿼리 성능 측면에서 불리해질 수 있다.</p>
<h4>다른 Reader 사용</h4>
<p>꼭 DTO projection을 사용해야 한다면 JpaPagingItemReader 대신 다른 Reader를 사용하는 것도 방법이다.대표적으로 JdbcPagingItemReader를 사용하는 방식이 있다.</p>
<p>JdbcPagingItemReader는 JPA의 영속성 컨텍스트를 사용하지 않고, JDBC를 기반으로 데이터를 조회한 뒤 RowMapper를 통해 원하는 형태로 매핑하는 구조를 가진다. 따라서 조회 결과가 엔티티가 아닌 DTO라 하더라도, 영속성 컨텍스트와의 충돌 없이 안전하게 사용할 수 있다.</p>
<p>특히 DTO projection이 필요한 경우, 필요한 컬럼만 선택적으로 조회할 수 있기 때문에 쿼리 효율 측면에서도 유리하다. 즉, 불필요한 컬럼을 포함한 엔티티 전체 조회가 아닌, 실제로 필요한 데이터만 조회할 수 있어 커버링 인덱스를 활용하기에도 적합하다.</p>
<p>다만 이 방식은 JPA의 장점을 사용할 수 없다는 단점이 있다. 예를 들어 JPQL을 활용한 추상화된 쿼리 작성이나, 엔티티 기반의 도메인 모델을 그대로 활용하기 어렵고, SQL을 직접 작성해야 하기 때문에 유지보수 측면에서 부담이 증가할 수 있다.</p>
<p>따라서 DTO projection이 반드시 필요하고, 쿼리 성능이 중요한 상황이라면 JdbcPagingItemReader와 같은 JDBC 기반 Reader를 사용하는 것이 더 적절한 선택이 될 수 있다.</p>
<h4>커스텀 Reader 구현</h4>
<p>커스텀 Reader를 직접 구현하여 detach() 로직을 제어하는 것이다. 앞서 살펴본 것처럼 transacted=false 환경에서 JpaPagingItemReader는 조회한 객체를 개별적으로 detach() 하는 방식으로 영속성 컨텍스트를 정리한다. 문제는 이 과정이 &ldquo;조회 결과가 엔티티일 것&rdquo;을 전제로 동작한다는 점이다. 따라서 DTO projection처럼 엔티티가 아닌 객체를 반환하는 경우에는 내부 detach() 로직과 충돌이 발생할 수 있다.</p>
<p>이러한 경우에는 Reader를 직접 구현하거나, 기존 JpaPagingItemReader를 확장하여 detach() 동작을 제어하는 방법을 고려할 수 있다. 예를 들어 조회 결과가 실제 엔티티인 경우에만 detach()를 수행하고, DTO인 경우에는 해당 로직을 건너뛰도록 처리하는 방식이다.</p>
<p>이 방식의 장점은 JPA 기반 조회를 유지하면서도, DTO projection 사용 시 발생하는 예외를 직접 우회할 수 있다는 점이다. 또한 프로젝트 상황에 맞게 Reader 동작을 세밀하게 제어할 수 있기 때문에, 단순히 엔티티 조회나 JDBC 기반 Reader로 전환하기 어려운 경우에는 하나의 대안이 될 수 있다.</p>
<p>다만 이 방법은 구현 복잡도가 높다는 단점이 있다. JpaPagingItemReader의 내부 동작 방식과 트랜잭션 처리 흐름, 영속성 컨텍스트 관리 방식까지 모두 이해한 상태에서 구현해야 하며, 잘못 구현할 경우 메모리 누수나 조회 일관성 문제로 이어질 수 있다. 또한 Spring Batch나 JPA 버전에 따라 내부 구현 차이에 영향을 받을 수 있어 유지보수 부담도 큰 편이다.</p>
<p>&nbsp;</p>
<p>&nbsp;</p>