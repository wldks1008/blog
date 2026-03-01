<h2>들어가며</h2>
<p><a href="https://wing1008.tistory.com/43" rel="noopener" target="_blank"><span>이전의</span></a> <a href="https://wing1008.tistory.com/44" rel="noopener" target="_blank"><span>글들</span></a>에서 트랜잭션 격리 수준에 따라 데이터 일관성이 어떻게 관리되는지 살펴보았다. 특히 Repeatable Read 격리 수준에서 Undo Log를 활용하여 어떻게 데이터 일관성을 보장하는지에 대해 다루었는데, 설명이 다소 간략해 아쉬운 부분이 있었다. 이번 글에서는 그 내용을 보완하여, InnoDB가 Consistent Read와 Read View를 통해 일관성을 유지하는 과정을 좀 더 자세히 살펴보고자 한다.<br />&nbsp;</p>
<hr />
<p>실습 환경은 아래 그림과 같다.&nbsp;</p>
<p><figure class="imageblock widthContent"><span><img height="492" src="https://blog.kakaocdn.net/dn/HSw3S/btsQAGqEJPz/e1r56TrWGRO7hewfc3Q0E0/img.png" width="1332" /></span></figure>
</p>
<p>데이터베이스에는 users 테이블과 wallet 테이블, 총 두 개의 테이블이 존재하며 각각의 테이블에 3개의 데이터가 저장되어 있다.</p>
<h2>단일 트랜잭션 상황</h2>
<p>먼저 기본적인 단일 트랜잭션 상황을 가정해보자.</p>
<p><figure class="imagegridblock">
  <div class="image-container"><span style="width: 45.8358%;"><img alt="" height="636" src="https://blog.kakaocdn.net/dn/lgLbL/btsQCmdn2M3/RKO0phWTUNVsel7sNCQjR0/img.png" width="928" /></span><span style="width: 53.0014%;"><img alt="" height="454" src="https://blog.kakaocdn.net/dn/DwWQI/btsQBEr8s8f/skVxxxSLGgPhsXrLPxEY7k/img.png" width="766" /></span></div>
</figure>
</p>
<p>하나의 트랜잭션에서 wallet 테이블의 Money 칼럼 값이 100보다 큰 레코드에 대해 각각 10씩 차감한 뒤 commit을 수행한다. 이후 또 다른 트랜잭션에서 동일한 조건(Money &gt;= 100)으로 데이터를 조회하면, 조건을 만족하는 레코드는 Id=3인 데이터만 반환된다.</p>
<h2>다중 트랜잭션 상황</h2>
<p>이제 2개의 트랜잭션이 동시에 실행되는 상황을 살펴보자.</p>
<h3>1.</h3>
<p><figure class="imagegridblock">
  <div class="image-container"><span style="width: 61.1994%;"><img alt="" height="642" src="https://blog.kakaocdn.net/dn/IgdTw/btsQCCNO9Fx/SKq7959071k2sQcJbnzQGk/img.png" width="926" /></span><span style="width: 37.6378%;"><img alt="" height="850" src="https://blog.kakaocdn.net/dn/Fo62H/btsQBiJAkT9/5Euw3C5VC6N8IWfzshW4Kk/img.png" width="754" /></span></div>
</figure>
</p>
<p>트랜잭션1에서 wallet 테이블의 money 칼럼 값이 100보다 큰 레코드에 대해 각각 10씩 차감했지만 아직 commit을 하지 않은 상태에서 다른 트랜잭션(트랜잭션2)에서 money&gt;=100 조건으로 조회했을 때는 언두로그를 통해 변경 이전의 값, 즉 차감되기 전 상태의 레코드가 조회된다.<br />여기까지는 이전 글에서도 다루었던, 기본적인 동작이다.&nbsp;</p>
<h3>2.</h3>
<p><figure class="imagegridblock">
  <div class="image-container"><span style="width: 61.2757%;"><img alt="" height="692" src="https://blog.kakaocdn.net/dn/1Dxoq/btsQANv9M1T/RSeQlKYGR2xKkOTEKiFDi0/img.png" width="1016" /></span><span style="width: 37.5615%;"><img alt="" height="820" src="https://blog.kakaocdn.net/dn/ASZKw/btsQBlzyjUM/yTC9cGKfckLYOWXPbNCgyK/img.png" width="738" /></span></div>
</figure>
</p>
<p>트랜잭션1에서 wallet 테이블의 money 칼럼 값이 100보다 큰 레코드에 대해 각각 10씩 차감했지만 아직 commit을 하지 않은 상태에서 트랜잭션2는 user 테이블에 대해 select for update 구문을 통해 조회하는 쿼리를 실행하였다. 이후 트랜잭션1은 commit 되었고, 트랜잭션2에서 wallet 테이블에서 money&gt;=100 조건으로 조회하는 쿼리를 실행하였다. 이전 예시의 결과와 다르게 트랜잭션1에 의해 변경된 값들이 반영되어 조회가 되었다.&nbsp;<br />처음에 나는 user 테이블에서 데이터를 읽어올 때, <span style="color: #333333;">for update 조건을 붙이면 잠금을 사용한 읽기를 하여 테이블의 값을 직접 읽어오고, 이후 트랜잭션1이 커밋되고 트랜잭션2에서 money&gt;=100 조건으로 조회를 하는 시점에 언두 로그를 통해 데이터를 조회하기 때문에 여전히 트랜젹션1이 변경하기 전의 상황의 값들을 조회할 것이라고 생각했다. 이 추측이 틀린 이유는 무엇일까?</span><br /><span style="color: #333333;">Undo Log와 Read View 이 2가지 개념을 혼동하였기 때문이다. 이에 대해 자세히 알아보자.</span></p>
<h4><span style="color: #333333;">트랜잭션의 고유 번호는 어떻게 부여될까?</span></h4>
<blockquote>모든 InnoDB의 트랜잭션은 고유한 트랜잭션 번호(순차적으로 증가하는 값)을 가지며, 언두 영역에 백업된 모든 레코드에는 변경을 발생시킨 트랜잭션의 번호가 포함되어 있다. ... REAPEATABLE READ 격리 수준에서는 MVCC를 보장하기 위해 실행 중인 트랜잭션 가운데 가장 오래된 트랜잭션 번호보다 트랜잭션 번호가 앞선 언두 영역의 데이터는 삭제할 수 없다.</blockquote>
<p><span style="color: #333333;">위 내용은 RealMySQL 1권의 5장에 기술되어 있는 내용이다. </span><span style="color: #333333;">여기서, "모든 트랜잭션은 '고유한', '순차적으로 증가하는' 트랜잭션 번호를 가진다." 라는 문구에 의해, 트랜잭션이 begin 할 때, 즉시 테이블의 기본키 값처럼 트랜잭션 고유 번호가 증가하는 것이라 생각했었다.</span><br />&nbsp;<br /><span style="color: #333333;">하지만, 이는 잘못된 오해이다. 트랜잭션이 시작될 때 무조건 번호를 받는 게 아니라, </span><span style="color: #333333;"><b>쓰기 작업(INSERT, UPDATE, DELETE 등)을 실제로 수행할 때</b></span><span style="color: #333333;">&nbsp;</span><span style="color: #333333;"><b>고유</b></span><span style="color: #333333;">&nbsp;</span><span style="color: #333333;"><b>ID가 할당</b></span><span style="color: #333333;">된다. 때문에 단순히 Select만 실행하는 Read-Only 트랜잭션에서는 트랜잭션 ID를 소모하지 않는다. 이는 </span><a href="https://dev.mysql.com/doc/refman/8.4/en/information-schema-innodb-trx-table.html" rel="noopener" target="_blank"><span><span style="color: #333333;">공식</span></span></a><span style="color: #333333;"> </span><a href="https://dev.mysql.com/doc/refman/8.4/en/innodb-performance-ro-txn.html" rel="noopener" target="_blank"><span><span style="color: #333333;">문서</span></span></a><span style="color: #333333;">에도 아래와 같이 기술되어 있다.</span></p>
<blockquote>InnoDB assigns a transaction ID only to read-write transactions. Read-only transactions that do not modify data do not receive a transaction ID.</blockquote>
<h4>Read View 와 Undo Log</h4>
<p>1. Read View</p>
<p>InnoDB에서 Read View와 Undo Log는 모두 MVCC(Multi-Version Concurrency Control) 기반으로 동작하지만, 그 목적과 역할이 다르다. Read View는 <b>InnoDB에서 일관된 읽기(Consistent Read)를 제공하기 위해 사용하는</b> <b>스냅샷 구조체</b>다. 쉽게 말해, 트랜잭션이 SELECT 쿼리를 실행할 때 해당 시점의 데이터 상태를 보장하기 위해 생성되는 것이다. Read View에는 현재 트랜잭션이 볼 수 있는 가장 최신의 커밋된 트랜잭션 ID가 포함되어 있어, 현재 트랜잭션이 볼 수 없는 트랜잭션들을 추적하여, 해당 트랜잭션들이 변경한 데이터를 읽지 않도록 할 수 있다. 또한, 현재 트랜잭션 ID 정보를 포함하여 자신이 수행한 작업이 다른 트랜잭션에 영향을 미치지 않도록 해준다.<br />&nbsp;</p>
<p>2. Undo Log</p>
<p>Undo Log는 <b>데이터 변경 작업을 취소할 수 있도록 이전 상태를 기록하는 로그</b>다. 주로 트랜잭션 롤백 시 사용되며, 데이터의 이전 버전을 복원하는 데 활용된다. 즉, Undo Log를 통해, InnoDB는 트랜잭션이 중단되었을 때 데이터의 무결성을 유지하며, 이전 상태로 복원할 수 있는 것이다.</p>
<hr />
<p>즉, Read View는 "언제 시점을 데이터를 볼 것인가"를 결정하는 즉, 트랜잭션이 볼 수 있는 데이터 상태를 정의하는 구조, Undo Log는 "그 시점의 실제 데이터"를 제공하는 저장소인 것이다.</p>
<h4>정리</h4>
<p>다시 돌아가서, 처음 상황을 살펴보자. 트랜잭션 간의 흐름을 그려보면 아래와 같다.</p>
<p><figure class="imageblock widthContent"><span><img height="1002" src="https://blog.kakaocdn.net/dn/rfVYf/btsQEGvPlet/ElD38aKbBDIoDuGszeIbH1/img.png" width="1390" /></span></figure>
</p>
<p>REPEATABLE READ 격리 수준에서는 트랜잭션 내 <a href="https://dev.mysql.com/doc/refman/8.4/en/innodb-consistent-read.html" rel="noopener" target="_blank"><span><b>첫 번째 읽기에서 생성된 스냅샷을 계속 사용</b></span></a>한다.<br />트랜잭션1이 Update 쿼리를 실행하는 순간, 해당 트랜잭션의 번호가 증가하게 되며 이때의 트랜잭션 고유 번호를 10이라고 가정해보자. 이어서 트랜잭션2에서 Read View가 생성되는 시점을 살펴보면, select ... for update 쿼리를 실행할 때, Read View가 생성되게 되며 트랜잭션 ID가 증가하여 11 이라는 트랜잭션 고유 번호가 부여되게 된다. 이후 트랜잭션1이 Commit을 하게 된다. 이후, money &gt;= 100 조건으로 조회를 할 때에는, 기존의 Read View를 재사용하게 되는데 트랜잭션1이 Commit되었으므로 <b>트랜잭션1의 변경사항이 가시적이므로 언두 로그를 사용하지 않고 현재 데이터의 테이블을 읽어오는 것</b>이다.<br />&nbsp;<br />즉, MVCC는 <b>Read View를 통해 "현재 버전이 보여도 되는가?"를 먼저 판단하고, 안 되는 경우에만 과거 버전(Undo Log)을 찾아가는 방식</b>이다. 이에 관한 내용도 <a href="https://dev.mysql.com/doc/refman/8.4/en/innodb-multi-versioning.html" rel="noopener" target="_blank"><span>공식 문서</span></a>에서 찾을 수 있었다.</p>
<blockquote>When a secondary index record is delete-marked or the secondary index page is updated by a newer transaction, InnoDB looks up the database record in the clustered index. In the clustered index, the record's DB_TRX_ID is checked, and the correct version of the record is retrieved from the undo log if the record was modified after the reading transaction was initiated.<br /><br /># 한 줄 요약<br />&gt; Secondary index가 삭제되거나 갱신되어도, InnoDB는 clustered index에서 DB_TRX_ID를 확인하고, 필요하면 undo log를 통해 트랜잭션 시작 시점에 맞는 데이터를 읽는다.</blockquote>
<h3>3.</h3>
<p>이어서 다음 상황을 살펴보자.</p>
<p><figure class="imagegridblock">
  <div class="image-container"><span style="width: 66.5643%;"><img alt="" height="668" src="https://blog.kakaocdn.net/dn/0sed3/btsQEOf9S5s/IcBSOvbNnCWo9NdiyzZSz1/img.png" width="930" /></span><span style="width: 32.2729%;"><img alt="" height="1120" src="https://blog.kakaocdn.net/dn/kFMC6/btsQFbWEDcw/LgjuEd90sf7RfCOKIQsF90/img.png" width="756" /></span></div>
</figure>
</p>
<p>1.<br />트랜잭션1에서 update쿼리를 통해 <span style="color: #333333;">wallet 테이블의 money 칼럼 값이 100보다 큰 레코드에 대해 각각 10씩 차감하였다. 이때, 트랜잭션 고유 번호가 증가하여 할당되며, 이때의 할당 받은 번호를 10이라고 가정하자. 또한, 데이터가 변경되는 작업이므로 변경 이전 값은 Undo Log에 기록된다.</span><br /><span style="color: #333333;">2. </span><br /><span style="color: #333333;">트랜잭션1이 커밋되기 전에 트랜잭션2에서 select 쿼리를 통해 wallet 테이블의 값을 읽어왔다. 이때의 경우에는 비잠금 select문이기 때문에 트랜잭션 번호가 할당되지 않은 채로 Read View가 생성된다. 즉, 해당 select문이 해당 트랜잭션에서 처음으로 실행되는 select문이기 때문에 그 시점의 "데이터베이스 상태"를 기준으로 스냅샷이 결정된다. 아직, 트랜잭션1(번호:10)이 commit되기 이전이기 때문에 visible하지 않아 언두 로그에서 데이터를 읽어오게 된다.</span><br /><span style="color: #333333;">3.</span><br /><span style="color: #333333;">트랜잭션1이 commit되었다.</span><br /><span style="color: #333333;">4. </span><br /><span style="color: #333333;">이후, 트랜잭션1에서 commit을 수행하고 이어서 트랜잭션2에서 select ... for update 구문을 통해서 money&gt;=100 조건으로 조회하는 쿼리를 실행하였다. 이때는 for update 조건을 통한 읽기 작업이기 때문에 트랜잭션 번호 11이 부여된다. 하지만, </span><span style="color: #333333;"><b>for update의 경우</b></span><span style="color: #333333;">에는 </span><span style="color: #333333;"><b>현재 최신의 데이터를 기준으로 동작</b></span><span style="color: #333333;">한다. 따라서 for update를 활용한 select 구문은 read view를 무시하고 현재의 커밋된 데이터를 조회한다.</span><br />5.<br />이어서 두번째 비잠금 select문을 수행하였다. 2번 단계 실행되던 select문과 동일한 트랜잭션에서 실행되는 쿼리이기 때문에 일관된 읽기(consistent read)를 보장해야 한다. 이를 위해, MySQL InnoDB에서는 동일한 트랜잭션에서는 첫 번째 일관된 읽기가 생성한 스냅샷(데이터 상태)을 그대로 읽는다. 때문에, 이 두 번째 SELECT도<b> 첫 번째 SELECT 시점의 데이터를 그대로 읽어 트랜잭션1에서 commit한 변경 내역을 볼 수 없다.</b><br />즉, InnoDB의 REPEATABLE READ 격리 수준에서 트랜잭션 내 모든 SELECT는 첫 번째 SELECT 시점의 "데이터 스냅샷"을 기반으로 읽는다. 이를 통해 <b>트랜잭션 내에서 같은 쿼리를 반복해도 항상 같은 결과</b>를 얻을 수 있게 되는 것이다.<br />&nbsp;<br />&nbsp;<br />&nbsp;<br />&nbsp;</p>