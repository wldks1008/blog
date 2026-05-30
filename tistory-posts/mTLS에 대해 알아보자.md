<h2><span>mTLS란?</span></h2>
<p style="color: #333333; text-align: start;"><span>TLS 기반인 HTTPS를 떠올리면, 클라이언트가 서버를 신뢰할 수 있는지 검증하는 구조를 떠올린다. 브라우저가 특정 사이트에 접속할 때 서버는 인증서를 제시하고, 브라우저는 해당 인증서가 신뢰할 수 있는 인증기관, 즉 CA에 의해 발급되었는지 확인한다. 이 과정을 통해 사용자는 내가 접속한 서버가 진짜 브라우저가 맞는가를 검증할 수 있다.</span></p>
<p style="color: #333333; text-align: start;"><span>하지만 서버 입장에서는 반대 방향의 검증이 필요할 때가 있다. 즉, "이 요청을 보낸 클라이언트가 정말 허용된 시스템인가"를 네트워크 연결 단계에서 확인하고 싶은 경우이다. 이때 사용하는 방식이 mTLS이다.</span></p>
<p style="color: #333333; text-align: start;">&nbsp;</p>
<p><span>mTLS(상호 TLS)는Mutual TLS의 약자로, 말 그대로 상호 TLS 인증을 의미한다. 일반 TSL가 주로 클라이언트가 서버를 검증하는 구조라면, mTLS는&nbsp; 클라이언트와 서버가 서로의 인증서를 검증한다. 이 경우, 서버 역시 클라이언트 인증서를 검증함으로써 "이 요청이 허용된 시스템에서 온 요청인가"를 판단할 수 있다.</span></p>
<h3><span>기본 용어</span></h3>
<h4><span>인증서와 개인키</span></h4>
<p><span>인증서는 어떤 주체의 공개키와 그 주체의 정보를 담고 있는 문서다. 인증서에는 보통 주체 이름, 발급자, 공개키, 서명 알고리즘 등이 포함된다.&nbsp;</span><span>인증서는 공개되어도 된다. 중요한 것은 인증서에 대응되는 개인키이다. 개인키는 인증서의 공개키와 쌍을 이루는 비밀 키이다. 서버가 자신의 인증서를 제시할 때, 실제로 중요한 것은 자신이 이 인증서의 주인인 것을 증명하는 것이다. 그리고 이 증명은 개인키로 서명하는 과정을 통해 이루어진다. 때문에 개인키는 절대로 외부로 유출되어서는 안된다.</span></p>
<p><span>TLS handshake에서는 CertificateVerify 메시지는 인증서에 대응하는 개인키를 실제로 소유하고 있음을 증명하는 역할을 한다.&nbsp;</span></p>
<h4><span>CA</span></h4>
<p><span>CA는 Certifiacte Authority의 약자이며 인증서를 발급하고 서명하는 기관이다. 클라이언트나 서버는 상대방의 인증서를 직접 믿는 것이 아니라 해당 인증서를 발급한 CA를 신뢰한다.</span></p>
<p><span>예를 들어 서버 인증서가 Internal Root CA 또는 Internal Intermediate CA에 의해 발급되었다면, 클라이언트의 truststore에는 해당 CA 인증서가 들어 있어야 한다. 그래야 클라이언트가 서버 인증서의 신뢰 체인을 검증할 수 있다.</span><span></span></p>
<blockquote><b>mTLS에서는 조직이 직접 CA 역할을 하는 경우가 많다.<br /></b>mTLS는 보통 불특정 다수를 위한 구조가 아니다. 때문에 "아무나 접속 가능한 HTTPS"가 아니라 "사전에 허용된 시스템만 접속 가능한 HTTPS"가 필요하다. 그래서 조직이 직접 사설 CA를 만들고, 그 CA로 클라이언트 인증서와 서버 인증서를 발급한다.</blockquote>
<h4><span>keystore와 truststore</span></h4>
<p><span><b>Keystore</b>은 "내가 제시할 인증서와 개인키"를 보관하는 저장소이다. 서버 입장에서 keystore에는 서버 인증서와 서버 개인키가 들어간다. 클라이언트 입장에서 Keystore에는 클라이언트 인증서와 클라이언트 개인키가 들어간다. 즉, keystore는 나의 신분증과 신분증을 증명할 개인키를 보관하는 곳이다.</span><span></span></p>
<p>&nbsp;</p>
<p><span><b>Truststore</b>은 "내가 신뢰할 CA 인증서"를 보관하는 저장소이다. 클라이언트 입장에서 truststore에는 서버 인증서를 검증할 수 있는 CA 인증서가 들어간다. 서버 입장에서 truststore에는 클라이언트 인증서를&nbsp; 검증할 수 있는 CA 인증서가 들어간다. 즉, truststore는 상대방의 신분증을 검증하기 위해 내가 신뢰하는 발급기관 목록이다.&nbsp;</span></p>
<p>클라이언트가 서버 인증서를 검증하려면, 클라이언트 truststore에 서버 인증서를 발급한 CA 체인이 있어야 한다. 반대로 서버가 클라이언트 인증서를 검증하려면 서버 truststore에 클라이언트 인증서를 발급한 CA 체인이 있어야 한다.</p>
<h3><span>TLS와 mTLS의 차이</span></h3>
<h4><span>TLS의 동작</span></h4>
<p><span>일반적인 TLS는 보통 다음과 같은 방식으로 동작한다.</span></p>
<p><figure class="imageblock widthContent"><span><img height="472" src="https://blog.kakaocdn.net/dn/c1Uamr/dJMcabLdFnw/4G5XpZzICBkyKnPu7Gjy7K/img.png" width="1332" /></span><figcaption>출처: CLOUDFLARE</figcaption>
</figure>
</p>
<p><span>클라이언트가 서버에 접속하면 서버는 자신의 인증서를 클라이언트에게 전달한다. 클라이언트는 이 인증서가 신뢰할 수 있는 CA에 의해 발급되었는지 확인한다. 인증서의 도메인, 유효기간, 서명 체인 등이 정상이라면 클라이언트는 서버를 신뢰하고 암호화된 통신을 시작한다.&nbsp;</span></p>
<p>&nbsp;</p>
<p><span style="color: #666666;">(TLS의 동작 방식에 대한 자세한 글은 <a href="https://wing1008.tistory.com/50" rel="noopener" target="_blank">해당 글</a>에 포스팅되어 있으니, 궁금한 점이 있다면 참고하길 바란다.)</span></p>
<h4><span>mTLS의 동작</span></h4>
<p><span>mTLS는 서버도 클라이언트에게 인증서를 요구한다. </span></p>
<p><figure class="imageblock widthContent"><span><img height="541" src="https://blog.kakaocdn.net/dn/bsmQ10/dJMcagFJ6W3/OiBf69plQ4R92yIWkKQ381/img.png" width="1232" /></span><figcaption>출처: CLOUDFLARE</figcaption>
</figure>
</p>
<p>클라이언트는 자신의 인증서를 서버에게 전달할 때, 인증서만 보내는 것이 아니라 자신이 해당 인증서의 개인키를 실제로 가지고 있음을 증명해야 한다. 이를 위해 handshake transcript에 대해 개인키로 서명한 CertificateVerify 메세지를 함께 보낸다.&nbsp;</p>
<p>서버는 클라이언트 인증서가 신뢰할 수 있는 CA에서 발급되었는지, 유효기간이 지나지 않았는지, 사용 목적이 client authentication에 적합한지, 폐기된 인증서는 아닌지 등을 검증한다.</p>
<p>이렇게 서버와 클라이언트가 서로의 인증서를 정상적으로 검증하면 이후 통신은 협상된 세션 키를 통해 암호화된다. 이 시점 이후 HTTP 요청과 응답이 오간다.&nbsp;</p>
<h2>시스템적으로 mTLS를 구축하는 방법</h2>
<h3>mTSL 적용 구간 정하기</h3>
<p>시스템적으로 mTLS를 구축하는 방법은 여러가지가 있겠지만, 이번 글에서는 크게 2가지 방법에 대해서 다뤄보고자 한다.</p>
<h4>1. 외부 클라이언트와 API Gateway 사이에 mTSL를 적용하는 방식</h4>
<p>해당 방식에서는 gateway가 클라이언트 인증서를 검증하고, 내부 애플리케이션으로는 검증된 요청만 전달한다.&nbsp;</p>
<p>대표적으로 NGINX에서는 `ssl_verify_client on` 으로 클라이언트 인증서 검증을 활성화할 수 있다.&nbsp;</p>
<pre class="bash" id="code_1780148334847"><code>server {
    listen 443 ssl;
    server_name api.example.com;

    # 서버 인증서
    ssl_certificate     /etc/nginx/certs/server.crt;
    ssl_certificate_key /etc/nginx/certs/server.key;

    # 클라이언트 인증서를 검증할 CA
    ssl_client_certificate /etc/nginx/certs/client-ca.crt;
    ssl_verify_client on;
    ssl_verify_depth 2;

    location / {
        proxy_set_header X-Client-Verify $ssl_client_verify;
        proxy_set_header X-Client-DN $ssl_client_s_dn;
        proxy_set_header X-Client-Fingerprint $ssl_client_fingerprint;

        proxy_pass http://backend;
    }
}</code></pre>
<p>이 구조에서는 주의할 점이 있다. 애플리케이션이 X-Client-DN 같은 헤더를 신뢰한다면, 외부에서 들어온 동일 이름의 헤더를 반드시 제거해야 한다. 그렇지 않으면 공격자가 직접 X-Client-DN 헤더를 조작해 인증된 클라이언트인 것처럼 위장할 수 있다.</p>
<p>따라서 Gateway에서 mTLS를 종료하는 구조라면 외부 요청의 인증서 관련 헤더를 먼저 제거후&nbsp;<span>Gateway가 검증한 인증서 정보만 새 헤더로 주입하는 작업이 필요하다. 또한 뒷단의 서버에서는</span> gateway에서 온 트래픽만 신뢰하도록 해야한다.</p>
<h4>2. 애플리케이션 서버가 직접 mTLS를 처리하는 방식</h4>
<p>Spring Boot, Tomcat, Netty 같은 애플리케이션 런타임이 직접 클라이언트 인증서를 검증한다. 이 경우 애플리케이션 설정에서 keystore와 truststore를 관리해야 한다.</p>
<p>&nbsp;</p>
<p>Spring Boot 애플리케이션이 직접 mTLS를 처리한다면 `server.ssl.*` 설정을 사용할 수 있다. `server.ssl.client-auth`는 클라이언트 인증 모드를 의미하며, NEED는 클라이언트 인증이 필수이고, WANT는 클라이언트 인증서를 요청하지만 필수는 아닌 모드이다.&nbsp;</p>
<p>예시는 다음과 같다.</p>
<pre class="java" id="code_1780148116284"><code>server:
  port: 8443
  ssl:
    enabled: true

    # 서버가 자신을 증명하기 위해 사용할 인증서와 개인키
    key-store: classpath:server.p12
    key-store-password: changeit
    key-store-type: PKCS12
    key-alias: server

    # 서버가 클라이언트 인증서를 검증하기 위해 신뢰할 CA
    trust-store: classpath:server-truststore.p12
    trust-store-password: changeit
    trust-store-type: PKCS12

    # 클라이언트 인증서 필수
    client-auth: need</code></pre>
<p>이 설정에서, key-store는 서버 자신의 인증서와 개인키를 담는다. trust-store는 클라이언트 인증서를 검증하기 위해 신뢰할 CA 인증서를 담는다. client-auth: need 를 설정하면 클라이언트 인증서를 제시하지 않는 요청은 TLS handshake 단계에서 실패한다.</p>
<p>애플리케이션이 외부 mTLS API를 호출하는 클라이언트라면 반대로 클라이언트의 keystore에는 클라이언트의 인증서와 개인키가 들어가고, truststore에는 호출 대상 서버 인증서를 검증할 CA가 들어간다.</p>
<h3>CA 전략 정하기</h3>
<p>mTLS에서 가장 중요한 것은 "무엇을 신뢰할 것인가"이다.&nbsp;사설 CA를 사용할 때는 보통 다음과 같은 구조를 둔다.</p>
<pre class="java" id="code_1780146593347"><code>Root CA
  └── Intermediate CA
        ├── server.example.internal 인증서
        ├── client-a 인증서
        └── client-b 인증서</code></pre>
<p>Root CA는 최대한 안전하게 보관하고, 실제 서버/클라이언트 인증서 발급은 Intermediate CA가 담당하는 구조가 일반적이다. 이렇게 하면 Intermediate CA 교체나 폐기 시 영향 범위를 줄일 수 있다.&nbsp;</p>
<h3>서버와 클라이언트의 keystore/truststore 구성하기</h3>
<p>mTLS에서는 서버와 클라이언트 양쪽 모두 keystore와 truststore가 필요하다.</p>
<pre class="java" id="code_1780146890933"><code>[Server]
- keystore   : 서버 인증서 + 서버 개인키
- truststore : 클라이언트 인증서를 발급한 CA 인증서

[Client]
- keystore   : 클라이언트 인증서 + 클라이언트 개인키
- truststore : 서버 인증서를 발급한 CA 인증서</code></pre>
<p>해당 설정에서 누락이나 문제가 생긴다면, 인증서 에러를 맞닥뜨리게 된다.&nbsp;</p>
<h4>PKIX path building failed 에러</h4>
<pre class="java" id="code_1780147046520"><code>javax.net.ssl.SSLHandshakeException:
  PKIX path building failed:
  sun.security.provider.certpath.SunCertPathBuilderException:
  unable to find valid certification path to requested target</code></pre>
<p>위 에러는, Java 환경에서 발생할 수 있는 대표적인 인증서 에러이다. 해당 에러는 Java가 상대방 인증서를 검증하기 위해 신뢰 가능한 인증 경로를 만들려고 했지만 실패했다는 뜻이다.</p>
<p>Java 입장에서는 상대방이 제시한 인증서에서 시작해 중간 CA를 거쳐, 최종적으로 자신이 신뢰하는 trust anchor까지 도달해야 한다. 하지만 'PKIX path building failed' 에러의 경우, 상대방 인증서를 받긴 했지만 해당 인증서를 내가 신뢰하는 CA까지 연결할 수 없어 TSL handshake에 실패했다는 것을 의미한다.</p>
<p>&nbsp;</p>
<p>mTLS에서는 해당 에러가 발생할 원인의 경우의 수가 더 많다.&nbsp;</p>
<p><b>1. 클라이언트가 서버 인증서를 신뢰하지 못하는 경우</b></p>
<p>클라이언트 애플리케이션의 truststore에 서버 인증서를 발급한 CA가 없으면 실패한다. 이 경우, 에러 로그는 보통 API를 호출하는 Java 어플리케이션 쪽에 남게 된다.</p>
<p>&nbsp;</p>
<p><b>2. 서버가 클라이언트 인증서를 신뢰하지 못하는 경우</b></p>
<p>서버의 truststore에 클라이언트 인증서를 발급한 CA가 없으면 실패한다. 이 경우 클라이언트 입장에서는 단순히 handshake failure, connection reset, bad certificate, certificate required 같은 에러만 볼 수도 있다. 실제 원인은 서버나 gateway 로그에 남는 경우가 많다.</p>
<p><b>3. 인증서 체인이 불완전한 경우</b></p>
<p>상대방이 leaf 인증서만 보내고 intermediate 인증서를 보내지 않으면, 검증자가 trust anchor까지 경로를 만들지 못할 수도 있다.</p>
<pre class="java" id="code_1780147600669"><code>받은 인증서:
server.example.com 인증서만 있음

필요한 체인:
server.example.com &rarr; Intermediate CA &rarr; Root CA

결과:
Intermediate CA를 찾지 못해 path building 실패</code></pre>
<p>이 경우, 해결책은 서버가 full chain을 보내도록 설정하거나, 검증자 truststore에 필요한 intermediate CA를 추가하는 것이다.</p>
<p>&nbsp;</p>
<p><b>4. 다른 truststore를 보고 있는 경우</b></p>
<p>인증서를 분명 import 했는데도 계속 에러가 난다면, 애플리케이션이 실제로 그 truststore를 보고 있지 않을 가능성이 있다. 이 경우, "실제 실행 중인 프로세스가 어떤 truststore를 사용하는가"를 확인해야 한다.</p>
<hr contenteditable="false" />
<p>이 외에도 여러 원인이 존재할 수 있다. 특히 mTLS는 양방향 검증이기 때문에 하나의 인증서 에러에 대해 정말 다양한 이유가 존재할 수 있다. 때문에 이를 해결할 때는 무작정 인증서를 Import하기 보다는 어느 방향의 검증이 실패했는지 먼저 찾고, "내 truststore 문제인지, 상대방 truststore 문제인지" 구분하는 것이 중요하다.</p>
<h2>번외)</h2>
<h3>Root CA와 Intermediate CA</h3>
<h4>Root CA</h4>
<p>Root CA는 인증서 신뢰 체계의 최상위 인증 기관이다. Root CA는 더 위에 있는 다른 CA에게 인증받지 않는다. 그래서 보통 자기 자신이 자기 자신을 서명한다. 이를 self-signed certificate, 즉 자체 서명 인증서라고 한다.&nbsp;</p>
<h4>Intermediate CA</h4>
<p>Intermediate CA는 Root CA 아래에 있는 중간 인증 기관이다. Root CA가 직접 서버 인증서나 클라이언트 인증서를 발급할 수도 있지만, 실무에서는 보통 그렇게 하지 않는다. 가장 큰 이유는 보안과 운영 안정성 때문이다. Root CA는 신뢰 체계의 최상위라서 매우 중요하다, Root CA의 개인키가 유출되면 그 Root CA가 서명한 모든 인증서 체계가 흔들릴 수 있다. 그래서 Root CA는 보통 최대한 안전하게 보관한다. 대신 Root CA가 Intermediate CA 인증서를 발급하고, Intermediate CA가 실제 서버/클라이언트 인증서를 발급한다. 만약, 클라이언트 인증서 발급 체계에 문제가 생기면 해당 CA만 교체하거나 폐기하면 되므로, Root CA 전체를 바꾸는 것보다 영향 범위가 훨씬 작아지게 된다.</p>
<p>구조는 아래와 같다.</p>
<pre class="kotlin" id="code_1780149173042"><code>&lt;위계 구조&gt;
Root CA
  └── Intermediate CA
        └── Server Certificate
        
&lt;예시&gt;
MyCompany Root CA
  ├── MyCompany Server Intermediate CA
  │     ├── api-a.internal 서버 인증서
  │     └── api-b.internal 서버 인증서
  │
  └── MyCompany Client Intermediate CA
        ├── batch-client 인증서
        └── partner-client 인증서</code></pre>
<p>각 단계의 서명 관계는 아래와 같다.</p>
<ul>
<li>Server Certificate는 Intermediate CA가 서명한다.</li>
<li>Intermediate CA Certificate는 Root CA가 서명한다.</li>
<li>Root&nbsp;CA&nbsp;Certificate는&nbsp;자기&nbsp;자신이&nbsp;서명한다.</li>
</ul>
<p>즉, 서버 인증서를 검증할 때는 다음 경로를 따라 올라간다.</p>
<pre class="kotlin" id="code_1780149221113"><code>api.my-service.com 인증서
  &rarr; MyCompany Intermediate CA
    &rarr; MyCompany Root CA</code></pre>
<p>그리고 클라이언트나 서버의 truststore에 MyCompany Root CA가 존재한다면 신뢰할 수 있는 요청으로 판단하는 것이다.</p>