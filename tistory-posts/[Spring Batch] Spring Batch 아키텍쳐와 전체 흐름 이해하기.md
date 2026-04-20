<h2>Spring Batch는 무엇을 해결하려는 기술일까?</h2>
<p>애플리케이션을 운영하다보면 사용자의 요청에 즉시 응답하는 API 처리와는 조금 다른 종류의 작업이 필요해진다. 매일 새벽 주문 데이터를 정산하거나, 대량의 회원 데이터를 읽어서 상태를 갱신하거나, 외부 시스템에서 받은 파일을 검증한 뒤 데이터베이스에 적재하는 작업들을 예시로 들 수 있다. 이러한 작업들은 보통 대량의 데이터를 처리해야 하고, 일정한 주기로 반복 실행되며, 실패했을 때 어디까지 처리했는지 추적할 수 있어야 한다.&nbsp;</p>
<p>&nbsp;</p>
<p>Spring Batch는 이런 배치 작업을 안정적으로 만들기 위한 프레임워크다. 공식 문서에서도 Spring Batch를 대량 데이터 처리를 위한 경량이면서 포괄적인 배치 프레임워크로 설명하고 있다. 다만, Spring Batch 자체가 스케쥴러는 아니다. Quartz, Kubernetes CronJob 과 같은 스케쥴러가 특정 시점에 배치 실행을 트리거하고, Spring Batch는 실제 배치 처리와 상태 관리, 트랜잭션, 재시작, 재시도 등과 같은 기능을 담당한다고 보는 것이 좋다.</p>
<h2>Spring Batch의 아키텍쳐</h2>
<p><figure class="imageblock widthContent"><span><img height="1166" src="https://blog.kakaocdn.net/dn/b59lkB/dJMcadn9Dyv/KzZYtwX3TkKUrvX9v2YxNk/img.png" width="1640" /></span><figcaption>Spring Batch 아키텍쳐</figcaption>
</figure>
</p>
<p>Spring Batch의 실행 흐름을 아주 단순하게 표현하면 아래와 같다.</p>
<blockquote>Scheduler -&gt; JobLauncher/JobOperator -&gt; Job -&gt; Step1 -&gt; . . . -&gt; StepN</blockquote>
<p>위의 그림에서 검은색 선은 실제 처리 흐름을 의미한다. Scheduler가 배치를 실행하면 JobLauncer가 Job을 실행하고, Job은 자신에게 정의된 Step들을 순서대로 실행한다. 각 Step은 실제 데이터를 읽고, 가공하고, 쓰는 역할을 수행한다.</p>
<p>빨간색 선은 배치 실행 정보가 저장되는 흐름을 의미한다. Spring Batch는 단순히 코드를 실행하고 끝내는 것이 아니라, "어떤 Job이 실행되었는지", "성공했는지 실패했는지", "몇 건을 읽었고 몇 건을 썼는지", "어디까지 처리했는지"와 같은 메타데이터를 JobRepository를 통해 저장한다.&nbsp;</p>
<h3>Job</h3>
<p>Job은 하나의 배치 작업 전체를 의미한다. 예를 들어 "매일 새벽 주문 정산 배치"라는 작업이 있다고 해보자. 이 작업 전체가 하나의 Job이 된다. 하지만 실제 주문 정산은 여러 단계로 나눌 수 있다. (정산 대상 주문 조회, 주문 금액 계산, 정산 결과 저장, 정산 완료 알림 발송 등)</p>
<p>Spring Batch에서 Job은 이런 여러 단계를 하나로 묶는 최상위 단위이다. 즉, Job은 전체 배치 프로세스를 캡슐화하는 엔티티이며, 하나 이상의 Step을 담는 컨테이너이다. 또한 Job은 어떤 Step을 어떤 순서로 실행할지, 재시작 가능한 작업인지 같은 전역 설정을 가진다.</p>
<p><figure class="imageblock widthContent"><span><img height="736" src="https://blog.kakaocdn.net/dn/bLHfNa/dJMcafzyoTc/NscvklTL2IDGhAWfldfgx0/img.png" width="1244" /></span></figure>
</p>
<p>위 코드에서, simpleJob은 총 2개의 Step으로 구성된다. start()는 첫번째로 실행할 Step을 의미하고, next()는 다음에 실행할 Step을 의미한다.</p>
<h4>JobInstance와 JobExecution</h4>
<p>Job이 배치라는 프로그램 그 자체라면, JobInstance는 특정 조건으로 실행되는 배치 대상을 의미하며 JobExecution은 JobInstance를 실제로 실행한 시도를 의미한다.&nbsp;</p>
<p>&nbsp;</p>
<p>예를 들어, 매일 한번씩 실행되는 Job이 있다고 생각해보자. 이때, 날짜가 JobParameters로 들어간다면, 각 날짜별 실행은 서로 다른 JobInstance가 된다. 즉, 같은 Job이라도 식별 파라미터가 다르면 처리 대상이 다르기 때문에 다른 JobInstance가 되는 것이다. JobInstance는 Job + identifying JobParameters 조합으로 정의된다. 즉, JobInstance는 이 Job이 어떤 조건의 데이터를 처리하는가를 나타내는 단위이며 때문에 같은 Job이여도 파라미터가 다르면 다른 JobInstance이다.</p>
<p>&nbsp;</p>
<p>만약에 이렇게 매일 배치를 돌리다가, 2026-04-19일자에 배치가 실패했다고 가정해보자. 이후 같은 파라미터로 다시 실행하면 새로운 JobExecution이 생성되게 된다. 즉, 같은 식별 파라미터로 다시 실행하면 같은 JobInstance아래에 새로운 JobExecution이 생성되는 형태인 것이다. 즉, JobExecution은 Job을 실행하려는 단일 시도(single attempt)이다. 또한 같은 식별 파라미터로 실패한 JobInstance를 다시 실행하면, 기존 JobInstance는 그대로 유지되고 새로운 JobExecution이 만들어진다.</p>
<p>&nbsp;</p>
<p><b>JobInstance와 JobExecution을 나누는 이유?</b></p>
<p>이를 나누는 이유는 바로 "재시작" 때문이다. 총 100만건을 처리햐아하는 정산 배치에서 70만건까지만 처리하고 실패한 상태라고 가정해보자. 이때 다시 실행할 때에는 2가지 선택지가 존재한다.&nbsp;</p>
<p>1. 처음부터 다시 처리하기</p>
<p>2. 실패한 지점부터 이어서 처리하기</p>
<p>Spring Batch는 이 2번째 방식을 지원하기 위해 JobInstance와 JobExecution을 구분한다. <b>같은 JobInstance로 다시 실행한다</b>는 것은 '새로운 날짜의 정산이 아니라 이전에 실패했던 정산을 다시 실행하는걸로 인식을 하고 이전 실행 상태를 참고해야겠다'라는 의미가 된다.</p>
<p>반면, <b>새로운 JobInstance로 실행한다</b>는 것은 '완전히 새로운 처리 대상이기 때문에 이전 상태를 참고하지 않고 처음부터 시작해야겠다.' 라는 의미가 되는 것이다.</p>
<p>정리하면, 같은 JobInstance를 사용하면 이전 실행의 상태, 즉 ExecutionContext를 사용할 수 있다는 것을 의미하고, 새로운 JobInstance를 사용하면 처음부터 시작한다는 의미이다.</p>
<h4>JobRepository의 필요성</h4>
<p><figure class="imageblock widthContent"><span><img height="830" src="https://blog.kakaocdn.net/dn/cJtLCl/dJMcahKT5XX/KnYdNAfK7DJbpGoOleLz2k/img.png" width="1828" /></span></figure>
</p>
<p>Spring Batch의 중요한 특징 중 하나는 실행 상태를 저장한다는 점이다. Spring Batch는 JobRepository를 통해 어떤 Job/JobParameters 이(로) 실행되었는지, Job의 성공유무, 각 Step이 성공했는지 실패했는지, 몇 건을 읽고 처리하고 저장했는지, 어디까지 처리했는지, 재시작할 떄 필요한 상태는 무엇인지 등과 같은 정보를 저장한다. 그리고 이런 정보들은 Spring Batch 메타데이터 테이블에 저장된다.&nbsp;</p>
<p>이러한 메타데이터 테이블은 Java 도메인 객체와 밀접하게 매핑되며, JobInstance, JobExecution, JobParameters, StepExecution은 각각 BATCH_JOB_INSTANCE, BATCH_JOB_EXECUTION, BATCH_JOB_EXECUTION_PARAMS, BATCH_STEP_EXECUTION에 매핑된다. 또한 ExecutionContext는 Job과 Step 각각의 context 테이블에 저장된다.&nbsp;</p>
<p>첫번째 사진에서 JobRepository와 Database가 연결되어 있는 이유가 바로 이러한 이유때문이다. Spring Batch는 실행 중 계속해서 JobRepository를 통해 메타데이터를 저장하고 갱신한다.</p>
<h3>Step</h3>
<p>Step은 Job을 구성하는 실제 처리 단위이다. Job이 "주문 정산 배치"라면, Step은 그 안에 포함된 "주문 읽기", "정산 계산하기", "정산 결과 저장하기" 같은 개별 단계다. 즉, Step은 배치 작업의 독립적이고 순차적인 단계를 캡슐화하는 도메인 객체라고 이해하면 된다.&nbsp;</p>
<p>모든 Job은 하나 이상의 Step으로 구성되며, Step은 실제 배치 처리를 정의하고 제어하는데 필요한 정보를 가진다.</p>
<h4>Tasklet 방식</h4>
<p>Step의 대표적인 처리 방식은 크게 2가지가 있다. 그 중 Tasklet 방식은 단순한 작업을 하나의 execute() 메서드 안에서 처리한다.&nbsp;</p>
<pre class="java" id="code_1776612142776"><code>@Bean
public Step cleanupStep(
        JobRepository jobRepository,
        PlatformTransactionManager transactionManager
) {
    return new StepBuilder("cleanupStep", jobRepository)
            .tasklet((contribution, chunkContext) -&gt; {
                // 파일 삭제, 임시 테이블 정리, API 호출 등
                return RepeatStatus.FINISHED;
            }, transactionManager)
            .build();
}</code></pre>
<p>TaskletStep은 저장 프로시저 호출, 스크립트 실행, SQL update 같은 작업처럼 chunk 방식으로 표현하기 어색한 경우에 사용할 수 있다. Tasklet의 execute 메서드는 FINISHED를 반환하거나 예외가 발생할 때까지 TaskletStep에 의해 반복 호출될 수 있고, 각 호출은 트랜잭션으로 감싸진다.</p>
<h4>chunk 방식</h4>
<p>두번째는 chunk 방식이다. 대량의 데이터를 읽고, 가공하고, 쓰는 배치 작업에서 가장 일반적으로 사용된다. 대용량 데이터를 처리하는 배치 프로세스 특성상, 대상 데이터들을 하나의 트랜잭션으로 처리하기에는 어려움이 있기 때문에 대상 데이터를 chunk 단위로 쪼개어 트랜잭션 동작을 수행한다.</p>
<p><figure class="imageblock widthContent"><span><img height="404" src="https://blog.kakaocdn.net/dn/eyNRA0/dJMcag6hSSe/Dxlm3WylbkhPRkAUEx5BLk/img.png" width="729" /></span><figcaption>chunk-oriented processing</figcaption>
</figure>
</p>
<p>위의 그림을 보면, stepExecution 안에서, ItemReader, ItemProcessor, ItemWriter가 순서대로 호출된다.</p>
<p>데이터를 하나씩 읽고, 설정된 commit interval 만큼 chunk를 만든 뒤, 해당 chunk를 트랜잭션 경계 안에서 ItemWriter로 쓰고 커밋하는 방식이다.</p>
<p>&nbsp;</p>
<p><b>ItemReader</b><b></b></p>
<p>말 그대로 데이터를 읽는 역할을 한다. JdbcCursorItemReader, JpaPagingItemReader와 같은 구현체를 사용할 수 있다.&nbsp;</p>
<p>ItemReader의 핵심은 read() 메서드이다. 데이터를 하나씩 반환하다가 더 이상 읽을 데이터가 없으면 null을 반환한다. 정리하면 ItemReader는 Step의 입력을 한 item씩 가져오는 추상화이며, 더 이상 제공할 item이 없을 때 null을 반환하는 방식으로 동작한다.</p>
<p>&nbsp;</p>
<p><b>ItemProcessor</b></p>
<p>ItemProcessor는 읽은 데이터를 가공하는 역할을 한다. 예를 들어 회원 데이터를 읽은 뒤 휴면 회원 여부를 판단하거나, 주문 데이터를 읽은 뒤 정산 금액을 계산하거나, 유효하지 않은 데이터를 필터링할 수 있다. Processor는 선택사항이다. 단순히 읽은 데이터를 그대로 저장하는 흐름이라면, 생략할 수 있다. 또한 process()에서 null을 반환하면 해당 item은 writer로 전달되지 않는다. 즉, ItemProcessor는 item에 비즈니스 처리를 적용하거나 변환하는 진입점이며, Null 변환 시 해당 item이 write 대상에서 제외된다.</p>
<p>&nbsp;</p>
<p><b>ItemWriter</b></p>
<p>마지막으로 ItemWriter는 데이터를 쓰는 역할을 한다. DB에 insert/update를 하거나, 외부 API로 전송하거나, Kafka로 메세지를 발행할 수 있다. 중요한 점은 ItemWriter가 ItemReader처럼 item을 1건씩 받을 때마다 바로 호출되는 것이 아니라는 점이다. Spring Batch는 ItemReader와 ItemProcessor를 통해 처리한 데이터를 chunk size만큼 모아두었다가, 그 묶음을 ItemWriter에 한 번에 전달한다. 예를 들어 chunk size가 100이라면 reader와 processor는 100번 동작하지만, writer는 100건이 담긴 chunk를 받아 1번 호출된다.</p>
<p>다만 실제 저장 방식이 SQL 한 번으로 끝나는지, 내부적으로 여러 번 저장하는지는 사용하는 writer 구현체에 따라 달라진다.</p>
<pre class="java" id="code_1776609111440"><code>@Override
public void write(Chunk&lt;? extends Settlement&gt; chunk) {
    for (Settlement settlement : chunk) {
        settlementRepository.save(settlement);
    }
}</code></pre>
<p>만약 위와 같이 구현을 했다면, Spring Batch는 Writer를 한 번 호출했지만, Writer 내부에서는 save()를 여러번 호출하는 방식으로 동작하게 된다.</p>
<h3>ExecutionContext란?</h3>
<p>ExecutionContext는 배치 실행 중 필요한 상태 값을 저장하는 공간이다.</p>
<p>예를 들어 파일을 읽다가 10,000번째 라인에서 실패했다고 해보자. 재시작할 때 처음부터 다시 읽는 것이 아니라, 가능하다면 10,001번째 라인부터 다시 시작하고 싶을 수 있다. 이런 재시작 정보를 저장하는 데 ExecutionContext가 사용된다.</p>
<p>ExecutionContext는 크게 두 종류로 볼 수 있다. <b>Job ExecutionContext</b>는 Job 실행 범위에서 사용하는 context다. 여러 Step 사이에서 공유해야 하는 값이 있다면 Job context를 사용할 수 있다. <b>Step ExecutionContext</b>는 특정 Step 실행 범위에서 사용하는 context다. 해당 Step의 처리 위치, reader 상태, 중간 통계 같은 정보를 저장할 수 있다.</p>
<p>공식 문서에 따르면 Step 범위의 ExecutionContext는 Step의 commit point마다 저장되고, Job 범위의 ExecutionContext는 Step 실행 사이에 저장된다. 또한 재시작 기능을 위해 context에 저장되는 non-transient 값은 직렬화 가능해야 한다.&nbsp;</p>
<p>맨 위의 사진에서 각 StepExecution 아래에 Execution Context가 있고, JobExecution 내부에도 별도의 Execution Context가 있는 이유가 바로 이 범위 차이 때문이다.</p>
<h2>마무리</h2>
<p>Spring Batch의 전체 구조는 처음 보면 복잡해 보이지만, 큰 흐름은 단순하다. 흐름을 하나씩 따라가보자.,&nbsp;</p>
<p>먼저 외부에서 배치 실행 요청이 들어온다. 이것은 스케줄러일 수도 있고, 커맨드라인 실행일 수도 있고, 운영자가 어드민 화면에서 실행한 요청일 수도 있다. JobLauncher는 실행할 Job과 JobParameters를 받아 Job 실행을 시작한다.</p>
<p>그다음 JobRepository를 통해 JobInstance와 JobExecution 정보가 생성되거나 조회된다. 이때 같은 Job이라도 식별 가능한 JobParameters가 다르면 새로운 JobInstance가 만들어진다. 같은 JobInstance가 이전에 실패했고 다시 실행된다면 새로운 JobExecution이 생성된다.</p>
<p>그다음 실제 Job이 실행된다. Job은 자신에게 정의된 Flow에 따라 Step을 실행한다. 각 Step이 시작될 때 StepExecution이 생성되고, Step 처리 상태가 JobRepository에 저장된다. 처리 중 실패하면 현재 상태와 실패 정보가 저장된다. 재시작 가능한 Job이라면 다음 실행 때 저장된 메타데이터와 ExecutionContext를 기반으로 이어서 처리할 수 있다.</p>
<p>마지막 Step까지 정상 완료되면 JobExecution의 상태가 COMPLETED가 된다. 중간에 복구되지 않은 예외가 발생하면 FAILED가 된다.</p>