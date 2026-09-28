<h2>다형성 JSON에서 문제가 생기는 지점</h2>
<p><span>Jackson의 </span><span>ObjectMapper</span><span>를 이용하면 JSON을 Java 객체로 쉽게 변환할 수 있다. 아래와 같은 코드가 있다고 생각해보자.</span></p>
<pre class="java" id="code_1790603560932"><code>// 역직렬화 코드
ObjectMapper objectMapper = new ObjectMapper();
Payment payment = objectMapper.readValue(json, Payment.class);

/*
// Payment.class 코드
public class Payment {

    private String orderId;
    private Card card;
}

// JSON 예시
{
  "orderId": "ORDER-1",
  "card": {
    "cardNumber": "1234-5678",
    "installment": 3
  }
}
*/</code></pre>
<p><span>이때 Jackson은 </span><span>Payment.class</span><span>를 기준으로 어떤 객체를 생성해야 하는지 판단하고, JSON의 각 프로퍼티를 Java 객체의 필드에 매핑한다. <span>Payment</span><span>를 역직렬화할 때 Jackson은 </span><span>orderId</span><span>는 </span><span>String</span><span>, </span><span>card</span><span>는 </span><span>Card</span><span> 타입이라는 것을 알 수 있다. </span></span><span>따라서 </span><span>Card</span><span> 객체를 생성하고 그 안에 </span><span>cardNumber</span><span>, </span><span>installment</span><span> 값을 매핑하면 된다. </span><span>그런데 </span><span>card</span><span>가 하나의 구체 타입이 아니라 인터페이스라면 이야기가 조금 달라진다. 예를 들어서 아래와 같은 상황이라고 가정해보자.</span></p>
<pre class="java" id="code_1790603739967"><code>public class Payment {
    private String orderId;
    private PaymentMethod paymentMethod;
}

// 인터페이스
public interface PaymentMethod {
}

// 구현체1
public class CardPayment implements PaymentMethod {

    private String cardNumber;
    private int installment;
}

// 구현체2
public class BankTransfer implements PaymentMethod {

    private String bankCode;
    private String accountNumber;
}</code></pre>
<p><span>Producer에서는 상황에 따라 </span><span>CardPayment</span><span>나 </span><span>BankTransfer</span><span>를 </span><span>paymentMethod</span><span>에 넣어 JSON으로 직렬화할 수 있다. 직렬화 자체는 문제가 없다. 직렬화 시점에는 이미 생성된 객체가 존재하기 때문에 Jackson이 해당 객체의 실제 런타임 타입을 확인할 수 있기 때문이다. </span><span>예를 들어 변수의 선언 타입이 </span><span>PaymentMethod</span><span>라고 하더라도 실제로 들어 있는 객체가 </span><span>CardPayment</span><span>라면 Jackson은 이를 </span><span>CardPayment</span><span>로 인식하고 해당 객체의 프로퍼티를 읽어 JSON으로 직렬화할 수 있다.</span></p>
<p>&nbsp;</p>
<p><span>문제는 반대 방향인 역직렬화다. 역직렬화 시점에는 아직 어떤 구현체의 객체도 생성되어 있지 않다. Jackson은 JSON과 </span><span>paymentMethod</span><span>의 선언 타입인 </span><span>PaymentMethod</span><span>를 가지고 새 객체를 생성해야 한다. 하지만 </span><span>PaymentMethod</span><span>는 인터페이스이기 때문에 직접 인스턴스를 생성할 수 없고, JSON에 별도의 타입 정보가 없다면 </span><span>CardPayment</span><span>와 </span><span>BankTransfer</span><span> 중 어떤 구현체를 생성해야 하는지도 판단할 수 없다.</span></p>
<p><span>역직렬화하는 시점에 Jackson이 알고 있는 것은 </span><span>paymentMethod</span><span>의 타입이 </span><span>PaymentMethod</span><span>라는 사실뿐이다. </span><span>하지만 </span><span>PaymentMethod</span><span>는 인터페이스이기 때문에 직접 객체를 생성할 수 없다. </span><span>그리고 JSON만 봐서는 어떤 구현체를 생성해야 하는지도 알 수 없다. </span><span>물론, 사람이 보면 </span><span>어떤 구현체를 선택해야하는지</span><span> 쉽게 판단할 수 있다.&nbsp;</span><span>하지만 Jackson이 기본적으로 애플리케이션에 존재하는 모든 </span><span>PaymentMethod</span><span> 구현체를 찾아서 각 필드 구성을 비교한 뒤 가장 적절한 클래스를 선택해주는 것은 아니다. </span><span>결국 Jackson 입장에서는 <span style="color: #333333;">"<span style="text-align: center;">PaymentMethod</span><span style="text-align: center;">&nbsp;대신 실제로 어떤 클래스를 생성해야 하는가?" 라는</span> </span>질문에 대한 답이 없는 것이다. </span><span>때문에 별도의 설정 없이 역직렬화를 시도하면 </span><span>Cannot construct instance</span><span>와 같은 런타임 예외가 발생한다.</span></p>
<h2>다형성 역직렬화를 위한 타입 정보</h2>
<p><span>이를 해결하려면 JSON 안에 어떤 타입인지 구분할 수 있는 정보가 있어야 한다. </span><span>예를 들어 아래처럼 JSON 필드에&nbsp;</span><span>type</span><span>이라는 프로퍼티를 추가할 수 있다.</span></p>
<pre class="java" id="code_1790604294151"><code>{
  "orderId": "ORDER-1",
  "paymentMethod": {
    "type": "CARD",
    "cardNumber": "1234-5678",
    "installment": 3
  }
}</code></pre>
<p><span>type</span><span>이 </span><span>CARD</span><span>라면 </span><span>CardPayment</span><span>, </span><span>BANK_TRANSFER</span><span>라면 </span><span>BankTransfer</span><span>를 생성하도록 약속하는 것이다. </span><span>Jackson에서는 이런 다형성 처리를 위해 </span><span>@JsonTypeInfo</span><span>와 </span><span>@JsonSubTypes</span><span>를 제공하고 있다.</span></p>
<pre class="java" id="code_1790604325502"><code>@JsonTypeInfo(
    use = JsonTypeInfo.Id.NAME,
    include = JsonTypeInfo.As.PROPERTY,
    property = "type"
)
@JsonSubTypes({
    @JsonSubTypes.Type(
        value = CardPayment.class,
        name = "CARD"
    ),
    @JsonSubTypes.Type(
        value = BankTransfer.class,
        name = "BANK_TRANSFER"
    )
})
public interface PaymentMethod {
}</code></pre>
<p><span>두 어노테이션은 역할이 조금 다르다. </span><span>@JsonTypeInfo</span><span>는 </span><b><span>JSON에서 타입을 어떤 방식으로 표현하고 찾을 것인지</span></b><span>를 설정한다. </span><span>반면 </span><span>@JsonSubTypes</span><span>는 </span><b><span>찾아낸 타입 값과 실제 Java 클래스를 연결</span></b><span>한다. </span><span>위 설정에서는 JSON의 </span><span>type</span><span> 프로퍼티를 확인하고 그 값이 </span><span>CARD</span><span>라면 </span><span>CardPayment</span><span>를 생성한다. </span><span>즉, `"type": "CARD"` 라는 <span>JSON의 정보와 CardPayment.class<span>를 연결해주는 것이 </span><span>@JsonSubTypes</span><span>다.</span></span></span></p>
<p><span>즉, 정리하면 </span><span>@JsonSubTypes</span><span>만 선언한다고 다형성 역직렬화가 활성화되는 것은 아니라는 것이다. </span><span>@JsonSubTypes</span><span>는 어떤 subtype이 존재하는지를 알려주는 역할이고, 실제로 타입 정보를 어떻게 읽을지는 </span><span>@JsonTypeInfo</span><span>를 통해 설정해야 한다.</span></p>
<h3><span>JsonTypeInfo</span><span></span></h3>
<h4><span>use</span></h4>
<p><span>@JsonTypeInfo</span><span>에는 다형성 처리 방식을 결정하는 몇 가지 옵션이 있다. </span><span>가장 먼저 </span><span>use</span><span>다. </span><span>use</span><span>는 JSON에서 어떤 값을 타입 식별자로 사용할 것인지를 의미한다.&nbsp;</span></p>
<p><span>1. Id.NAME</span></p>
<p><span>일반적으로 가장 사용하기 편한 방식이다. <span>JSON에는 실제 Java 클래스 이름 대신 별도로 정의한 이름이 들어간다. <span>Java 클래스 이름이 변경되더라도 </span><span>"CARD"</span><span>라는 JSON 규격은 그대로 유지할 수 있다. </span></span></span><span>Producer와 Consumer가 서로 독립적으로 배포되는 구조라면 Java 클래스 이름을 그대로 노출하는 것보다 이런 식으로 논리적인 타입 이름을 사용하는 편이 훨씬 낫다.</span></p>
<p>&nbsp;</p>
<p><span>2. </span><span>Id.CLASS</span></p>
<p><span>Id.CLASS</span><span>는 Java 클래스의 Fully Qualified Class Name 자체를 타입 정보로 사용한다. <span>JSON은 대략 다음과 같은 형태가 된다.</span></span></p>
<pre class="java" id="code_1790604576882"><code>{
  "@class": "com.example.payment.CardPayment",
  "cardNumber": "1234-5678",
  "installment": 3
}</code></pre>
<p><span>별도의 이름을 등록하지 않아도 Jackson이 클래스를 바로 찾을 수 있다는 장점은 있다. </span><span>다만 서비스 간 메시지에 사용하기에는 결합도가 너무 높아진다. </span><span>패키지를 이동하거나 클래스 이름을 변경하는 것만으로도 메시지 규격이 변경되기 때문이다. </span><span>Java 내부 구현이 그대로 외부 JSON 계약에 노출된다는 점도 좋지 않다. </span><span>특히 신뢰할 수 없는 외부 데이터를 역직렬화하는 상황에서는 class name 기반의 다형성 역직렬화가 보안 측면에서도 주의가 필요하다. </span><span>때문에 외부 API나 서비스 간 이벤트라면 개인적으로 </span><span>Id.CLASS</span><span>를 선택할 이유는 많지 않다고 생각한다. Id.MINIMAL_CLASS<span style="color: #333333; text-align: start;"><span>&nbsp;</span>역시 클래스명을 조금 더 짧게 표현할 뿐 Java 클래스 구조에 의존한다는 특징은 비슷하다.</span></span></p>
<p>&nbsp;</p>
<p><span>3. </span><span>Id.DEDUCTION</span></p>
<p><span>타입을 명시적으로 넣지 않고 JSON에 존재하는 필드를 기반으로 subtype을 추론하는 방식도 있다.</span></p>
<pre class="java" id="code_1790604654444"><code>@JsonTypeInfo(use = JsonTypeInfo.Id.DEDUCTION)
@JsonSubTypes({
    @JsonSubTypes.Type(CardPayment.class),
    @JsonSubTypes.Type(BankTransfer.class)
})
public interface PaymentMethod {
}</code></pre>
<p><span>예를 들어 </span><span>CardPayment</span><span>만 </span><span>cardNumber</span><span>라는 필드를 가지고 있다면, <span>Jackson이 필드 구성을 보고 </span><span>CardPayment</span><span>라고 판단할 수 있다.</span></span></p>
<p><span>처음 보면 꽤 편리해 보인다. </span><span>하지만 타입을 결정하는 기준이 명시적인 값이 아니라 JSON 구조에 의존하게 된다. </span><span>처음에는 subtype마다 필드가 확실하게 구분되더라도 모델이 변경되면서 서로 비슷한 구조를 가지게 될 수 있다. </span><span>타입을 구분하는 것이 중요한 메시지라면 차라리 </span><span>"type": "CARD"</span><span>처럼 명시적으로 표현하는 편이 계약을 이해하기도 쉽고 변경에도 대응하기 좋다고 생각한다.</span></p>
<h4><span>include</span></h4>
<p><span>include</span><span>는 타입 정보를 JSON의 어디에 표현할지 결정한다. </span><span>가장 일반적인 방식은 </span><span>PROPERTY</span><span>다.</span></p>
<pre class="java" id="code_1790605040283"><code>{
  "orderId": "ORDER-1",
  "paymentMethod": {
    "type": "CARD",
    "cardNumber": "1234-5678",
    "installment": 3
  }
}</code></pre>
<p><span>즉, 위의 예시처럼&nbsp;<span>타입 정보가 일반 프로퍼티처럼 객체 안에 들어가는 형태다. </span></span><span>이외에도 </span><span>WRAPPER_OBJECT</span><span>, </span><span>WRAPPER_ARRAY</span><span>, </span><span>EXTERNAL_PROPERTY</span><span> 등의 옵션이 존재한다.&nbsp;</span></p>
<p>&nbsp;</p>
<p><span>이미 객체 안에 타입을 나타내는 필드가 존재한다면 </span><span>EXISTING_PROPERTY</span><span>를 사용할 수도 있다.</span></p>
<pre class="java" id="code_1790605100769"><code>@JsonTypeInfo(
    use = JsonTypeInfo.Id.NAME,
    include = JsonTypeInfo.As.EXISTING_PROPERTY,
    property = "type",
    visible = true
)</code></pre>
<p><span>EXISTING_PROPERTY</span><span>는 Jackson이 타입 정보를 위해 별도의 프로퍼티를 만드는 것이 아니라 객체에 이미 존재하는 프로퍼티를 타입 식별자로 사용한다. </span><span>여기서 같이 볼 수 있는 옵션이 </span><span>visible</span><span>이다. </span><span>기본값은 </span><span>false로, </span><span>Jackson은 </span><span>type</span><span> 값을 subtype을 결정하기 위해 사용하고 나면 실제 객체의 일반적인 프로퍼티에는 전달하지 않는다. <span>하지만 객체에서도 </span><span>type</span><span> 값을 그대로 사용하고 싶다면 true로 변경하면 된다. <span>그러면 타입 판별에 사용된 값이 실제 객체의 </span><span>type</span><span> 프로퍼티에도 바인딩된다.</span></span></span><span><span></span></span></p>
<h3><span>JsonSubTypes</span></h3>
<p><span>@JsonSubTypes</span><span>의 역할은 비교적 단순하다. <span>value</span><span>에는 실제 subtype 클래스를 지정하고, <span>name</span><span>에는 JSON에서 사용할 타입 식별자를 지정한다.</span></span></span><span><span><span></span></span></span></p>
<h4><span><span><span>name</span></span></span></h4>
<pre class="java" id="code_1790605193323"><code>@JsonSubTypes({
    @JsonSubTypes.Type(
        value = CardPayment.class,
        name = "CARD"
    ),
    @JsonSubTypes.Type(
        value = BankTransfer.class,
        name = "BANK_TRANSFER"
    )
})</code></pre>
<p><span>이렇게 등록하면 </span><span>"CARD"</span><span>라는 값과 </span><span>CardPayment.class</span><span>가 연결된다. </span><span>Subtype 쪽에 직접 이름을 정의하는 방법도 있다.</span><span></span></p>
<p><span>경우에 따라서는 하나의 subtype에 여러 이름을 허용할 수도 있다. <span>예를 들어 기존에는 </span><span>"CREDIT_CARD"</span><span>라는 값을 사용했지만 새로운 규격에서는 </span><span>"CARD"</span><span>를 사용한다고 해보자.</span></span></p>
<pre class="java" id="code_1790605587398"><code>@JsonSubTypes.Type(
    value = CardPayment.class,
    names = {"CARD", "CREDIT_CARD"}
)</code></pre>
<p><span>두 값을 동일한 </span><span>CardPayment</span><span>로 역직렬화할 수 있기 때문에 메시지 스키마를 변경하는 과정에서 하위 호환성을 유지하는 용도로 활용할 수 있다.</span></p>
<h4><span>defaultImpl</span></h4>
<p><span>@JsonTypeInfo</span><span>에는 등록되지 않은 타입이나 타입 정보가 없는 경우 사용할 기본 구현체를 지정하는 </span><span>defaultImpl</span><span>도 있다. <span>Consumer가 아직 지원하지 않는 타입을 받을 수 있는 환경이라면 유용해 보일 수 있다. </span></span><span>하지만 이 역시 사용 목적을 명확히 하는 것이 좋다. <span>예를 들어 Producer에 새로운 결제 방식이 추가됐다고 해보자. <span>Consumer는 아직 </span><span>CRYPTO</span><span>를 지원하지 않는다. </span></span></span><span>이때 역직렬화 자체를 실패시키면 적어도 Consumer가 처리할 수 없는 메시지가 들어왔다는 사실이 명확하다.</span></p>
<p><span>반대로 모든 알 수 없는 타입을 </span><span>UnknownPayment</span><span>로 받아버리면 역직렬화에는 성공하지만 이후 로직에서 해당 메시지가 정상적으로 처리되고 있는 것처럼 보일 수도 있다. </span><span>결국 역직렬화에 성공하는 것과 메시지를 올바르게 처리하는 것은 다른 문제다. </span><span>따라서 </span><span>defaultImpl</span><span>을 사용한다면 이후에 알 수 없는 타입을 어떻게 처리할 것인지도 함께 설계해야 한다.</span></p>
<h2><span><span>다형성 JSON을 사용하는 것이 좋은 설계일까?</span></span></h2>
<p><span>@JsonTypeInfo</span><span>와 </span><span>@JsonSubTypes</span><span>를 사용하면 꽤 편리하다.</span></p>
<p><span>Java에서는 인터페이스를 기준으로 모델을 정의할 수 있고, Consumer에서는 별도로 JSON을 분석하지 않아도 Jackson이 적절한 구현체까지 만들어준다. </span><span>Subtype의 개수가 많지 않고, 해당 타입들이 실제 도메인에서도 자연스럽게 하나의 추상 타입으로 묶이는 구조라면 충분히 사용할 만한 방법이라고 생각한다.</span></p>
<p><span>다만 서비스 간 메시지에 사용한다면 조금 더 신중하게 볼 필요가 있다고 생각한다. </span><span>가장 먼저 생각할 부분은 메시지와 Java 타입 사이의 결합이다. </span><span>특히 </span><span>Id.CLASS</span><span>처럼 Java 클래스 자체를 타입 정보로 사용하면 내부 구현이 그대로 메시지 계약이 되어버린다. 물론, </span><span>이 문제는 </span><span>Id.NAME</span><span>을 사용하면 상당히 줄일 수 있지만, 그렇다고 다형성 메시지 자체의 변경 비용이 사라지는 것은 아니다. </span><span></span><span></span></p>
<p><span>특히, 개인적으로 또 하나 신경 쓰이는 부분은 중요한 메시지 규칙이 Jackson 어노테이션 안에 숨어들 수 있다는 점이다. <span>코드는 분명 간결하다. </span></span><span>하지만 어떤 타입의 메시지를 지원하고 있고, 어떤 값을 어떤 클래스에 연결하는지가 Jackson 설정의 일부가 된다.</span></p>
<p><span>Subtype이 몇 개 없을 때는 문제가 크지 않지만 메시지 종류가 계속 증가하면 하나의 부모 타입이 너무 많은 메시지 규격을 알고 있게 될 수도 있다. </span><span>특히 API나 Kafka 메시지 DTO와 도메인 모델을 같은 클래스로 사용한다면 직렬화 규칙이 도메인 모델에 직접 섞이는 문제도 생긴다. </span><span>이런 경우에는 타입을 메시지 바깥에서 명시적으로 관리하는 방법이 나을 수 있다고 생각한다.</span></p>
<p><span>Consumer는 </span><span>eventType</span><span>에 따라 적절한 handler를 선택하고 해당 handler가 </span><span>payload</span><span>를 자신의 DTO로 역직렬화하도록 만들 수 있다. 해당 구현 방식은</span><span>&nbsp;@JsonSubTypes</span><span>를 사용하는 것보다 작성해야 하는 코드는 조금 많아진다. </span><span>대신 어떤 이벤트를 지원하는지, 지원하지 않는 이벤트가 들어왔을 때 어떻게 처리하는지, 이벤트의 버전을 어떻게 관리하는지가 코드에 조금 더 명시적으로 드러난다.</span></p>
<p><span>물론 어느 한쪽이 항상 더 좋은 방법이라고 보기는 어렵다. </span><span>같은 애플리케이션 안에서 제한된 subtype을 처리한다면 Jackson의 다형성 기능이 훨씬 간결할 수 있다. </span><span>반대로 여러 서비스가 오랫동안 공유하는 메시지 계약이라면 역직렬화의 편의성보다 타입 추가와 변경에 따른 호환성까지 같이 생각할 필요가 있다.</span></p>
<h2>정리</h2>
<p><span>Jackson은 JSON을 역직렬화할 때 대상 Java 타입을 기준으로 생성할 객체를 결정한다. </span><span>따라서 필드의 타입이 구체 클래스라면 큰 문제가 없지만 인터페이스나 추상 클래스라면 JSON만으로 실제 구현체를 결정할 수 없다. </span></p>
<p><span>이를 해결하기 위해 </span><span>@JsonTypeInfo</span><span>로 타입 정보를 읽는 방법을 정의하고, </span><span>@JsonSubTypes</span><span>로 타입 식별자와 실제 Java 클래스를 연결할 수 있다. </span><span>사용 방법 자체는 어렵지 않다.</span></p>
<p><span>다만 다형성 JSON이 서비스 간 메시지에 사용되기 시작하면 단순히 Jackson의 역직렬화 문제로만 볼 수는 없다. </span><span>새로운 subtype을 추가하는 것이 곧 새로운 메시지 타입을 추가하는 것이 될 수 있고, Producer와 Consumer 사이의 호환성 문제로 이어진다.</span></p>
<p><span>그래서 개인적으로는 Jackson이 다형성을 지원한다는 이유만으로 메시지를 다형성 구조로 만드는 것은 조금 조심스러운 편이다. </span><span>다형성이 실제 데이터 모델에서도 자연스럽고 subtype의 범위가 명확하다면 좋은 선택이 될 수 있다. </span><span>하지만 타입이 계속 늘어날 가능성이 있거나 여러 서비스가 해당 JSON을 장기간 계약으로 사용한다면, </span><span>Jackson이 이를 역직렬화할 수 있는가보다 이 타입 구조를 메시지 계약으로 가져가는 것이 적절한가를 먼저 고민하는 편이 좋다고 생각한다.</span></p>