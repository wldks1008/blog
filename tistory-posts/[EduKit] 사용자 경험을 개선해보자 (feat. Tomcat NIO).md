<h2>문제점</h2>
<p>1차 MVP를 마친 이후, Edukit는 사용자의 피드백을 바탕으로 지속적으로 개선을 이어가고 있다. 성능 및 부하 테스트 과정에서, 서비스의 핵심 기능인 생기부 응답 AI 생성 로직에서 부하 테스트 도중 톰캣 쓰레드 풀이 고갈되는 문제가 발견되었다.</p>
<p><figure class="imageblock widthContent"><span><img height="814" src="https://blog.kakaocdn.net/dn/coBTaF/btsQoB2QPtT/qBR2lrAFfMrz8XiNxBwDjk/img.png" width="1124" /></span><figcaption>기존의 흐름</figcaption>
</figure>
</p>
<p>현재 구조를 살펴보면, 사용자가 프롬프트를 입력하면 이를 기반으로 OpenAI 서버에 AI 응답 생성을 요청하는 전 과정이&nbsp;<b>동기식 처리</b>로 이루어져 있다. 이로 인해 각 요청이 톰캣의 스레드를 점유하게 되고, 스레드 풀이 모두 소진되면서 다른 요청들이 대기 상태에 놓였다. 결과적으로 전체 API 응답 속도가 지연되는 병목 현상이 발생한 것이다.<br />톰캣 스레드 고갈도 문제지만, 사용자 체감 속도가 느리다는 점이 더 큰 문제였다. 현재 서비스는 사용자의 한 번의 생성 요청에 대해 3가지 버전의 응답을 생성하며, 이 3가지 모두가 완료되어야 클라이언트에 응답을 보낼 수 있다. 즉, 응답이 도착하기까지 평균 40초 동안 사용자는 빈 화면을 보게 되는 것이다. 이러한 구조 때문에 사용자 체감 속도가 크게 떨어지게 되었다.&nbsp;<br />이번 글에서는 이러한 구조를 어떻게 개선하였는지에 관해 작성해보고자 한다.</p>
<h2>고민 과정</h2>
<h3>1. 동기를 비동기로</h3>
<p>가장 큰 문제였던 동기식 처리 흐름을 먼저 비동기 방식으로 전환했다.&nbsp;</p>
<p><figure class="imageblock widthContent"><span><img height="648" src="https://blog.kakaocdn.net/dn/cLKH6i/btsQHiXiR6Y/WawuRiIb9XKOIlpivs1rS0/img.png" width="1524" /></span><figcaption>수정된 흐름</figcaption>
</figure>
</p>
<p>기존에는 사용자가 입력한 값을 보내는 Post 요청 1회로 OpenAI API 호출까지 함께 처리하였는데, 이 과정에서 커넥션과 스레드가 점유되는 것이 근본적인 원인으로 판단하였기 때문이다. 개선된 흐름은 아래와 같다.<br />최초의 POST 요청에서는 요청에 대응하는 작업 ID(TaskId)를 생성하고, 이를 클라이언트에 즉시 반환한다. 동시에 비동기로 OpenAI API를 호출하여, 사용자가 입력한 값과 내부에서 정의한 프롬프트를 결합해 생활기록부 생성을 요청한다. 이후 클라이언트는 POST 요청으로 받은 작업 ID를 활용해, 별도의 GET 요청을 통해 AI가 생성한 응답을 조회하도록 API를 두 단계로 분리하였다.</p>
<h3>2. AI가 생성한 응답 값을 어떻게 받아올까?</h3>
<h4>Polling 방식</h4>
<p>가장 단순한 방식은 클라이언트가 발급받은 TaskId를 이용하여 서버에 주기적으로 요청을 보내고, AI 응답 생성이 완료되었는지 확인하는 것이다.</p>
<p><figure class="imageblock widthContent"><span><img height="1052" src="https://blog.kakaocdn.net/dn/bhSXvf/btsQGssDUrO/jFKbqKgCnA8CjQeqkI3sv1/img.png" width="1766" /></span></figure>
</p>
<p>위 그림과 같이 AI가 응답 생성을 완료하면 결과는 DB나 캐시에 저장되고, 서버는 클라이언트 요청 시 해당 상태를 조회한다. 아직 완료가 되지 않았다면 "미완성" 응답을 반환하고 클라이언트는 설정한 주기가 지나면 다시 요청을 보낸다. 이처럼 Polling 방식은 주기적인 요청을 반복하다가 생성이 완료되면 최종 결과를 받아오는 방식이다.<br />요청 주기는 고정값으로 설정할 수도 있지만, 트래픽 급증 상황을 고려해 지수적 증가 방식으로 조정할 수도 있다. 이 값은 클라이언트 설정에 따라 달라진다. 이러한 폴링(Polling) 방식은 OpenAI 응답이 완성될 때까지 커넥션과 스레드를 점유하지 않고 짧고 간헐적인 요청만 발생시킨다. 때문에 기존 방식에서 문제였던 톰캣 스레드 풀 고갈을 어느 정도 예방할 수는 방식이다.<br />&nbsp;<br />하지만, Polling 방식은 사용자 체감 속도를 개선해주지는 못했다. 여전히 클라이언트는 AI가 응답을 생성하는데까지 걸리는 시간인 평균 40초가 지난 뒤에야 결과를 받아볼 수 있었기 때문이다. 더욱이 클라이언트가 서버로부터 아직 완성되지 못했다는 응답을 받은 직후에 AI가 생성을 완료하더라도, 클라이언트가 다음 요청을 보낼 때까지 기다려야 했다. 즉, 최악의 경우 기존 동기식 요청보다 더 늦게 결과가 전달될 가능성도 존재했다.<br />이를 완화하기 위해, 빈 화면 대신 현재 수행 중인 작업 단계를 보여주자는 아이디어가 있었다. 이를 위해 AI의 진행 상황을 DB에 단계별로 기록하고 클라이언트가 이를 조회하도록 하는 방식이다. 그러나 이 경우 단계마다 DB write 연산이 발생해 부하와 잠금(lock) 비용이 늘어날 수 있다는 문제가 있었다. 물론 DB 대신 Redis 같은 캐시를 활용하면 부하 문제는 완화할 수 있다. 하지만 캐시를 사용하더라도 Polling 방식의 본질적인 한계는 여전히 남는다. 클라이언트는 주기적으로 요청을 보내야 하고, 요청 직후 응답이 생성되더라도 다음 주기까지 기다려야 한다. 또한 사용자가 늘어날수록 서버와 캐시에 반복 요청이 누적되어 트래픽 부하가 커지는 문제 역시 해결되지 않았다. 이러한 이유로, 우리는 Polling 대신 다른 접근 방식을 찾아보게 되었다.</p>
<h4>SSE 방식</h4>
<p>Polling의 한계를 보완하기 위해 찾아본 방식은 바로 SSE 방식이었다. SSE는 Server Sent Event의 약자로, 서버에서 이벤트가 발생하면 클라이언트로 직접 push 해주는 방식이다. 때문에 클라이언트가 주기적으로 요청을 보낼 필요가 없다. 구조상 websocket과 비슷하지만 서버에서 클라이언트로만 향하는 단방향 통신만 지원한다는 점에서 차이가 있다. 현재 우리 서비스는 서버에서 AI 응답이 생성된 시점에만&nbsp; 클라이언트로 결과를 전달하는 구조이기 때문에 굳이 양방향 통신이 가능한 WebSocket을 사용할 필요는 없다고 판단하였다.</p>
<p><figure class="imageblock widthContent"><span><img height="1042" src="https://blog.kakaocdn.net/dn/pyxX7/btsQG1BuPpP/zN2TrCU2f8JDwbXFDK88vK/img.png" width="1678" /></span></figure>
</p>
<p>Polling 방식과 비교했을 때, SSE(Server-Sent Events)의 가장 큰 장점은 실시간성이다. 클라이언트가 주기적으로 서버에 요청을 보내는 Polling과 달리, SSE는 서버에서 이벤트가 발생하면 즉시 클라이언트로 데이터를 푸시할 수 있다. 따라서 응답 지연 없이 실시간으로 업데이트를 전달할 수 있으며, 단계별 진행 상황도 별도의 DB나 캐시를 거치지 않고 바로 클라이언트에 전송할 수 있다.<br />SSE는 단방향 통신이므로 구현이 단순하다. 또한 HTTP 기반이기 때문에 방화벽이나 프록시 환경에서도 별도의 설정 없이 사용할 수 있다. 연결이 유지되는 동안 서버는 이벤트 스트림을 계속 클라이언트로 전송할 수 있으며, 클라이언트는 EventSource API를 통해 자동 재연결 기능을 활용할 수 있다.<br />그렇다면 SSE를 사용하면 톰캣 스레드 풀 고갈 문제도 해결될까? 일반적으로 SSE 연결은 클라이언트와 서버 간 커넥션을 계속 유지해야 하므로, 전통적인 Spring MVC의 Thread per Request 모델에서는 각 요청이 서버 스레드를 점유하게 되어 스레드 풀 고갈 문제가 발생할 수 있다. 하지만 <b>Spring Boot 3.2부터 지원하는 SseEmitter는 내부적으로 비동기적으로 작동하므로, SSE 연결 동안 톰캣의 스레드를 점유하지 않고 이벤트를 전송할 수 있다.&nbsp;</b></p>
<h2>SseEmitter</h2>
<h3>Spring MVC에서의 SseEmitter</h3>
<p><a href="https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-ann-async.html" target="_self"><span><span style="background-color: #e6f5ff;"><span style="color: #0070d1;">공식문서</span></span></span></a>에 따르면 Spring MVC의 SseEmitter는 비동기 스트리밍 방식으로 동작하며, 이를 이용하면 요청 처리 스레드를 점유하지 않고 지속 연결을 유지할 수 있다. 현재 톰캣은 기본적으로 NIO Connector를 사용하기 때문에, 하나의 스레드가 여러 연결의 입출력을 관리할 수 있으며 이 덕분에 수천 개의 동시 SSE 연결을 비교적 적은 스레드로 처리할 수 있다. 여기서 NIO Connector란 무엇일까?</p>
<h4>NIO 기반의 톰캣 동작 방식</h4>
<p><figure class="imageblock widthContent"><span><img height="336" src="https://blog.kakaocdn.net/dn/b6zdan/btsQZndtXYH/UYYykDXbuDMEVA2XitTQl0/img.png" width="1798" /></span></figure>
</p>
<p>스프링 애플리케이션을 실행하면, 위와 같이 빨간색 박스와 같은 로그가 출력된다. 이는 톰캣이 기본적으로 NIO(Non-blocking I/O) Connector를 사용한다는 의미이다.</p>
<p><figure class="imageblock widthContent"><span><img height="247" src="https://blog.kakaocdn.net/dn/rMM2L/btsQ0BJcUXn/1hMz8Kx5A2tVJV9Xr8c6Gk/img.png" width="926" /></span></figure>
</p>
<p>톰캣의 내부 동작을 간단히 정리하면 아래와 같다.</p>
<p><br />1. Acceptor라는 컴포넌트가 클라이언트의 연결 요청을 받는다.<br />2. 연결이 맺어지면 톰캣은 이를 NIO Channel이라는 객체로 감싸고, 이벤트 큐에 등록한다.<br />3. Poller가 이벤트 큐에서 읽기/쓰기 요청을 감지하고, 실제 처리는 워커 스레드가 담당한다. 이때, 해당 워커 스레드가 바로 우리가 작성한 비즈니스 로직(컨트롤러, 서비스 등)을 실행하며, 로그에 출력되는 http-nio-8080-exec-* 스레드가 바로 이것이다. 그냥 흔히 우리가 말하는 톰캣 스레드 라고 부르는 것이 이 워커 스레드라고 생각하면 된다.</p>
<p><br />즉, 톰캣은 NIO 기반 구조 덕분에 소켓 연결 관리(Acceptor, poller)와 실제 비즈니스 처리(워커 스레드)를 분리해서 효율적으로 확장성을 확보한다.</p>
<h4>SseEmitter의 동작 원리</h4>
<p>앞서 살펴본 것처럼 톰캣은 Acceptor -&gt; Poller -&gt; 워커 스레드(http-nio-80-exec-*) 구조로 요청을 처리한다. 일반적인 HTTP 요청의 경우, 클라이언트 요청이 들어오면 워커 스레드가 컨트롤러를 실행하고 응답을 작성한 뒤 종료된다. 따라서 워커 스레드 수는 곧 동시 처리 가능한 요청 수와 직결된다.<br />하지만 SSE(Server-Sent Events)는 연결을 오래 유지하면서 서버가 여러 번 데이터를 푸시하는 특성을 가진다. 만약 이때 워커 스레드가 계속 점유된다면, 수천 개의 클라이언트가 동시에 연결했을 때 스레드 풀이 쉽게&nbsp; 고갈되어 서버가 더 이상 요청을 처리할 수 없게 된다.<br />&nbsp;<br /><b>SseEmitter</b>는 바로 이런 스레드 풀 고갈 문제를 해결하기 위해 <b>비동기로 동작</b>한다.</p>
<p><figure class="imageblock widthContent"><span><img height="402" src="https://blog.kakaocdn.net/dn/8Ech8/dJMcaaRbkWb/46ugcdaWB7XVosDe94F3v1/img.png" width="1078" /></span></figure>
</p>
<p>현재 우리 서비스의 전체적인 구조는 위의 그림과 같다. 만약 클라이언트가 GET 요청으로 SSE 엔드포인트를 호출하면, 톰캣 스레드가 요청을 받아 SSEEmitter 객체를 생성하고 반환하게 된다. 이때, SSE 연결이 유지되는 동안 스레드가 블로킹되는 것이 아니라, NIO Selector가 소켓 채널을 모니터하게 된다. 즉, 클라이언트와의 TCP 연결은 메모리상의 채널 객체로만 존재하며, 스레드는 점유하지 않는 것이다.</p>
<p>AI pipeLine 측에서 Redis에 메시지를 발생하고 스프링 서버에서 RedisStreamConsumer의 워커 스레드가 Redis에서 메시지를 읽어와 SSEEmitter.send()를 호출하면, 그때서야 톰캣의 NIO 워커 스레드 풀에서 스레드 하나가 할당되어 실제 데이터 전송을 수행한다. 이후 전송이 완료되면 이 스레드는 다시 풀로 반환되는 구조로 동작한다. 정리하면, <b>데이터 전송 순간에만 짧게 스레드를 사용하고, 대기 시간에는 스레드를 점유하지 않는다</b>.</p>
<p>&nbsp;<br />결과적으로 SseEmitter를 사용하면 다수의 클라이언트와 연결을 유지하면서도 워커 스레드를 효율적으로 활용할 수 있다. 하지만 그럼에도 여전히 단점은 존재한다. Tomacate의 NIO는 어디까지나 네트워크 I/O 단계에서만 논블록킹 방식으로 동작한다. 즉, 실제 요청을 처리하는 비즈니스 로직을 처리하는 부분은 여전히 서블릿 기반의 MVC이기 때문에 동시에 응답할 수 있는 횟수는 스레드 풀의 수에 의존하게 된다. 때문에 <span>실제 이벤트를 push하는 시점에는 워커스레드가 점유되더라도, 동시 요청 수가 너무 많을 경우에는 결국 push 하는 시점에 ThreadPool 고갈 문제에 마주칠 수 있다.</span></p>
<h3>Spring WebFlux 기반 SSE</h3>
<p>Spring WebFlux는 Reactor 기반 논블로킹 런타임(Netty, Undertow 등) 위에서 동작한다. 클라이언트 요청이 들어오면 논블로킹 이벤트 루프가 요청을 처리하며 SSE 스트림은 Flux&lt;ServerSentEvent&gt; 형태로 반환된다. 이벤트가 발생하면 이벤트 루프가 직접 데이터를 네트워크로 전송하므로 워커 스레드가 아닌 논블로킹 I/O 스레드만으로 다수의 클라이언트를 동시에 처리할 수 있다. WebFlux 환경에서는 요청 처리부터 데이터 전송까지 완전 논블로킹 구조이므로 서버 자원을 훨씬 효율적으로 활용할 수 있다. 즉, 별도의 Redis 리스너 스레드가 필요하지 않기 때문에 Redis 메시지 수신, 변환 처리, SSE 전송이 모두 동일한 이벤트 루프에서 처리되므로 스레드 간 컨텍스트 스위칭이 최소화되는 것이다.</p>
<hr />
<p>WebFlux 기반의 SSE는 서버 자원 활용 측면에서 훨씬 효율적이지만, 이를 적용하려면 기존 Spring MVC로 작성된 코드를 모두 WebFlux 기반으로 수정해야 하는 대규모 작업이 필요하다. 현재 서버 자원이 충분하므로 무리하게 WebFlux로 이전하기보다는, MVC에서 제공하는 비동기 SSE를 활용하여 1차적인 리팩토링을 완료하였다. 만약 사용자의 트래픽이 급격히 증가하여 톰캣 스레드 풀이 자주 고갈되는 상황이 발생한다면, 그때 WebFlux로의 이전을 검토해보기로 하였다.<br />&nbsp;</p>