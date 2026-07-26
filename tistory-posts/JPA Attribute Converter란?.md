<h2>들어가며</h2>
<p>JPA를 사용하다 보면 엔티티에서 사용하는 값과 데이터베이스에 저장해야 하는 값의 형태가 서로 다른 경우가 있다. 대표적으로 다음과 같은 상황이 있다.</p>
<ul>
<li>엔티티에서는 Boolean을 사용하지만 DB에는 Y, N으로 저장해야 하는 경우</li>
<li>개인정보를 DB에 저장하기 전에 암호화해야 하는 경우</li>
<li>객체를 JSON 문자열로 변환하여 저장해야 하는 경우</li>
<li>Enum을 별도의 코드 값으로 저장해야 하는 경우</li>
</ul>
<p>이러한 변환 로직을 엔티티나 서비스에 직접 작성하면 어떻게 될까? 데이터를 저장할 때마다 DB 형식으로 변환해야 하고, 조회한 뒤에는 다시 엔티티에서 사용하는 형식으로 변환해야 한다.</p>
<pre class="armasm"><code>val marketingAgreedValue =
    if (profile.marketingAgreed) "Y" else "N"
</code></pre>
<p>반대로 조회할 때는 다음과 같은 코드가 필요할 수 있다.</p>
<pre class="nix"><code>val marketingAgreed =
    databaseValue == "Y"
</code></pre>
<p>변환이 필요한 필드가 많아질수록 같은 코드가 여러 곳에 반복된다. 또한 저장하거나 조회할 때 변환 과정을 누락하지 않았는지도 계속 신경 써야 한다. JPA는 이러한 문제를 해결하기 위해 AttributeConverter를 제공한다. 이번 글에서는 AttributeConverter가 무엇인지 알아보고, 다음 두 가지 예제를 통해 실제 동작을 살펴보려고 한다.</p>
<h2>AttributeConverter란?</h2>
<p>AttributeConverter는 엔티티 속성과 데이터베이스 컬럼 사이의 값을 변환하기 위해 JPA에서 제공하는 인터페이스다. 인터페이스의 구조는 다음과 같다.</p>
<pre class="kotlin"><code>interface AttributeConverter&lt;X, Y&gt; {

    fun convertToDatabaseColumn(attribute: X): Y

    fun convertToEntityAttribute(dbData: Y): X
}
</code></pre>
<p>여기서 두 타입의 의미는 다음과 같다.</p>
<ul>
<li>X: 엔티티에서 사용하는 타입</li>
<li>Y: 데이터베이스 컬럼에 저장되는 타입</li>
</ul>
<h3>convertToDatabaseColumn</h3>
<p>엔티티 속성 값을 데이터베이스 컬럼 값으로 변환한다. 엔티티가 저장되거나 수정되어 SQL의 파라미터 값이 만들어질 때 사용된다.</p>
<pre class="excel"><code>엔티티 속성 값
    &darr;
convertToDatabaseColumn
    &darr;
DB 컬럼 값
</code></pre>
<h3>convertToEntityAttribute</h3>
<p>데이터베이스에서 조회한 컬럼 값을 엔티티 속성 값으로 변환한다.</p>
<pre class="excel"><code>DB 컬럼 값
    &darr;
convertToEntityAttribute
    &darr;
엔티티 속성 값
</code></pre>
<hr contenteditable="false" />
<p>즉, AttributeConverter는 엔티티와 데이터베이스 사이에서 양방향 변환을 담당한다. 애플리케이션에서는 도메인에 적합한 타입을 그대로 사용하고, 데이터베이스에 어떤 형태로 저장할지는 Converter에 위임할 수 있다.</p>
<h2>실제 사용 예시</h2>
<h3>Boolean 값을 Y/N으로 저장하기</h3>
<p>엔티티에서는 참과 거짓을 표현할 때 Boolean을 사용하는 것이 자연스럽다. 하지만 데이터베이스나 외부 시스템과 연동하는 테이블에서는 참과 거짓을 Y, N으로 표현하는 경우가 있다. 예를 들어 엔티티에서는 다음과 같이 값을 사용하고 싶다고 가정해보자.</p>
<pre class="arcade"><code>var marketingAgreed: Boolean
</code></pre>
<p>반면 실제 데이터베이스에는 다음과 같이 저장해야 한다.</p>
<pre class="asciidoc"><code>marketing_agreed
----------------
Y
</code></pre>
<p>이때 AttributeConverter를 이용하면 엔티티의 Boolean과 DB의 Y, N을 자동으로 변환할 수 있다.</p>
<h4>BooleanToYnConverter 작성하기</h4>
<pre class="bash"><code>import jakarta.persistence.AttributeConverter
import jakarta.persistence.Converter

@Converter
class BooleanToYnConverter : AttributeConverter&lt;Boolean, String&gt; {

    override fun convertToDatabaseColumn(
        attribute: Boolean?,
    ): String? {
        return when (attribute) {
            true -&gt; "Y"
            false -&gt; "N"
            null -&gt; null
        }
    }

    override fun convertToEntityAttribute(
        dbData: String?,
    ): Boolean? {
        return when (dbData) {
            "Y" -&gt; true
            "N" -&gt; false
            null -&gt; null
            else -&gt; throw IllegalArgumentException(
                "Y/N만 Boolean으로 변환할 수 있다: $dbData",
            )
        }
    }
}</code></pre>
<p>convertToDatabaseColumn에서는 엔티티 값을 DB 값으로 변환한다. 반대로 convertToEntityAttribute에서는 DB 값을 엔티티 값으로 변환한다.</p>
<p>여기서 알 수 없는 값이 전달되었을 때 무조건 false를 반환하지 않고 예외를 발생시켰다. 만약 예외처리를 따로 하지 않고 구현하면 어떤 문제가 있을까?</p>
<p>DB에 A, 1, 빈 문자열과 같은 잘못된 값이 들어 있어도 모두 정상적인 false 값으로 처리된다.</p>
<p>잘못된 데이터를 조용히 정상 데이터로 변환하면 데이터 오류를 발견하기 어려워진다. 따라서 허용하지 않은 값에 대해서는 명시적으로 예외를 발생시키는 편이 안전하다.</p>
<h4>엔티티에 Converter 적용하기</h4>
<p>작성한 Converter는 엔티티 필드에 @Convert를 선언하여 적용할 수 있다.</p>
<pre class="less"><code>import jakarta.persistence.Column
import jakarta.persistence.Convert
import jakarta.persistence.Entity
import jakarta.persistence.GeneratedValue
import jakarta.persistence.GenerationType
import jakarta.persistence.Id
import jakarta.persistence.Table

@Entity
@Table(name = "converter_profiles")
class ConverterProfile(

    @Column(name = "name", nullable = false)
    var name: String,

    @Convert(converter = BooleanToYnConverter::class)
    @Column(
        name = "marketing_agreed",
        nullable = false,
        length = 1,
    )
    var marketingAgreed: Boolean,

) {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long = 0L
        protected set
}
</code></pre>
<p>marketingAgreed는 엔티티에서 Boolean 타입으로 선언되어 있다. 하지만 @Convert에 BooleanToYnConverter를 지정했기 때문에 데이터베이스에 저장될 때는 Converter가 실행된다.</p>
<pre class="routeros"><code>val profile = ConverterProfile(
    name = "member-a",
    marketingAgreed = true,
)

repository.save(profile)
</code></pre>
<p>애플리케이션에서는 일반적인 Boolean 값처럼 true를 설정했다. 하지만 실제 DB에는 다음과 같이 저장된다.</p>
<pre class="1c"><code>name      | marketing_agreed
----------|-----------------
member-a  | Y
</code></pre>
<p>조회할 때도 별도의 변환 코드는 필요하지 않다.</p>
<pre class="routeros"><code>val profile =
    repository.findById(id).orElseThrow()

println(profile.marketingAgreed)
</code></pre>
<p>JPA가 DB에서 조회한 Y를 Converter에 전달하고, Converter가 이를 다시 true로 변환해주기 때문이다.</p>
<h3>문자열을 자동으로 암호화하여 저장하기</h3>
<p>AttributeConverter는 단순한 타입 변환뿐만 아니라 문자열을 암호화하는 용도로도 사용할 수 있다. 예를 들어 엔티티에서는 개인정보나 민감 정보를 평문으로 다루되, 데이터베이스에는 암호문으로 저장하고 싶다고 가정해보자.</p>
<p>이러한 변환도 Converter를 통해 구현할 수 있다. 다음은 문자열을 AES-GCM 방식으로 암호화하는 학습용 예제다.</p>
<h4>AesGcmStringConverter 작성하기</h4>
<pre class="kotlin"><code>import jakarta.persistence.AttributeConverter
import jakarta.persistence.Converter
import java.security.SecureRandom
import java.util.Base64
import javax.crypto.Cipher
import javax.crypto.spec.GCMParameterSpec
import javax.crypto.spec.SecretKeySpec

@Converter
class AesGcmStringConverter :
    AttributeConverter&lt;String, String&gt; {

    override fun convertToDatabaseColumn(
        attribute: String?,
    ): String? {
        if (attribute == null) {
            return null
        }

        val iv = ByteArray(IV_LENGTH).also {
            secureRandom.nextBytes(it)
        }

        val cipher = Cipher
            .getInstance("AES/GCM/NoPadding")
            .apply {
                init(
                    Cipher.ENCRYPT_MODE,
                    SecretKeySpec(learningKey, "AES"),
                    GCMParameterSpec(TAG_LENGTH, iv),
                )
            }

        val encrypted = cipher.doFinal(
            attribute.toByteArray(Charsets.UTF_8),
        )

        return Base64.getEncoder()
            .encodeToString(iv + encrypted)
    }

    override fun convertToEntityAttribute(
        dbData: String?,
    ): String? {
        if (dbData == null) {
            return null
        }

        val decoded = Base64.getDecoder()
            .decode(dbData)

        require(decoded.size &gt; IV_LENGTH) {
            "암호화된 데이터의 길이가 올바르지 않다."
        }

        val iv = decoded.copyOfRange(
            0,
            IV_LENGTH,
        )

        val encrypted = decoded.copyOfRange(
            IV_LENGTH,
            decoded.size,
        )

        val cipher = Cipher
            .getInstance("AES/GCM/NoPadding")
            .apply {
                init(
                    Cipher.DECRYPT_MODE,
                    SecretKeySpec(learningKey, "AES"),
                    GCMParameterSpec(TAG_LENGTH, iv),
                )
            }

        val decrypted = cipher.doFinal(encrypted)

        return String(
            decrypted,
            Charsets.UTF_8,
        )
    }

    companion object {

        private const val IV_LENGTH = 12
        private const val TAG_LENGTH = 128

        private val secureRandom = SecureRandom()

        // 학습용 키다.
        // 운영 환경에서는 절대로 하드코딩하면 안 된다.
        private val learningKey =
            "jpa-study-key-16"
                .toByteArray(Charsets.UTF_8)
    }
}
</code></pre>
<p>저장 시에는 다음 과정이 수행된다.</p>
<pre class="properties"><code>평문 문자열
    &darr;
랜덤 IV 생성
    &darr;
AES-GCM 암호화
    &darr;
IV와 암호문 결합
    &darr;
Base64 문자열로 변환
    &darr;
DB 저장
</code></pre>
<p>조회 시에는 반대 과정이 수행된다.</p>
<pre class="properties"><code>DB 문자열
    &darr;
Base64 디코딩
    &darr;
IV와 암호문 분리
    &darr;
AES-GCM 복호화
    &darr;
평문 문자열 반환
</code></pre>
<h4>암호화 Converter 적용하기</h4>
<p>암호화할 엔티티 필드에도 @Convert를 선언한다.</p>
<pre class="less"><code>@Entity
@Table(name = "converter_profiles")
class ConverterProfile(

    @Column(name = "name", nullable = false)
    var name: String,

    @Convert(converter = AesGcmStringConverter::class)
    @Column(
        name = "secret_memo",
        nullable = false,
        length = 500,
    )
    var secretMemo: String,

    @Convert(converter = BooleanToYnConverter::class)
    @Column(
        name = "marketing_agreed",
        nullable = false,
        length = 1,
    )
    var marketingAgreed: Boolean,

) {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long = 0L
        protected set
}
</code></pre>
<p>애플리케이션에서는 암호화 여부를 신경 쓰지 않고 평문을 그대로 설정한다.</p>
<pre class="routeros"><code>val profile = ConverterProfile(
    name = "member-a",
    secretMemo = "외부에 노출되면 안 되는 값",
    marketingAgreed = true,
)

repository.save(profile)
</code></pre>
<p>엔티티 내부에서는 원문이 사용되지만 실제 DB에는 암호화된 문자열이 저장된다. 이후 JPA로 엔티티를 다시 조회하면 Converter가 자동으로 복호화를 수행한다. 즉, 서비스 코드에서는 평문을 사용하지만 DB에는 암호문이 저장되는 구조가 만들어진 것이다.</p>
<h4>실제 DB 값을 확인해보자</h4>
<p>그런데 JPA로 엔티티를 조회하면 Converter가 자동으로 복호화한다. 그렇다면 데이터베이스에 정말 암호문이 저장되었는지는 어떻게 확인할 수 있을까? JPA Repository가 아닌 JdbcTemplate을 사용하면 Converter를 거치지 않고 실제 DB 컬럼 값을 직접 조회할 수 있다.</p>
<pre class="kotlin"><code>@SpringBootTest
@Transactional
class ConverterProfileTest(

    @Autowired
    private val repository: ConverterProfileRepository,

    @Autowired
    private val jdbcTemplate: JdbcTemplate,

    @PersistenceContext
    private val entityManager: EntityManager,

) {

    @Test
    fun `문자열은 암호화되어 저장되고 조회할 때 복호화된다`() {
        val plainText = "노출되면 안 되는 값"

        val saved = repository.saveAndFlush(
            ConverterProfile(
                name = "member-a",
                secretMemo = plainText,
                marketingAgreed = true,
            ),
        )

        val databaseValue = jdbcTemplate.queryForObject(
            """
            SELECT secret_memo
            FROM converter_profiles
            WHERE id = ?
            """.trimIndent(),
            String::class.java,
            saved.id,
        )

        assertNotEquals(
            plainText,
            databaseValue,
        )

        entityManager.clear()

        val reloaded = repository
            .findById(saved.id)
            .orElseThrow()

        assertEquals(
            plainText,
            reloaded.secretMemo,
        )
    }
}
</code></pre>
<p>&nbsp;</p>
<h3>번외) autoApply 옵션</h3>
<p>지금까지는 엔티티 필드마다 @Convert를 명시적으로 선언했다.</p>
<pre class="angelscript"><code>@Convert(
    converter = BooleanToYnConverter::class,
)
var marketingAgreed: Boolean
</code></pre>
<p>@Converter의 autoApply 옵션을 사용하면 특정 타입에 Converter를 자동 적용할 수도 있다.</p>
<pre class="angelscript"><code>@Converter(autoApply = true)
class BooleanToYnConverter :
    AttributeConverter&lt;Boolean, String&gt; {
    // ...
}
</code></pre>
<p>이렇게 설정하면 엔티티의 Boolean 속성에 해당 Converter가 자동으로 적용된다. 따라서 다음과 같이 @Convert를 생략할 수 있다.</p>
<pre class="kotlin"><code>@Column(
    name = "marketing_agreed",
    nullable = false,
    length = 1,
)
var marketingAgreed: Boolean
</code></pre>
<p>편리해 보이지만 적용 범위를 주의해야 한다. 프로젝트의 모든 Boolean 컬럼이 동일한 방식으로 저장된다는 보장은 없기 때문이다.</p>
<pre class="properties"><code>marketing_agreed &rarr; Y/N
deleted          &rarr; 0/1
enabled          &rarr; BOOLEAN
</code></pre>
<p>이러한 프로젝트에서 BooleanToYnConverter를 전역으로 자동 적용하면 원하지 않는 컬럼까지 Y, N으로 변환될 수 있다. 따라서 특정 필드에만 변환을 적용하려면 다음과 같이 명시적으로 선언하는 편이 안전하다.</p>
<pre class="angelscript"><code>@Convert(
    converter = BooleanToYnConverter::class,
)
var marketingAgreed: Boolean
</code></pre>
<p>autoApply = true는 해당 타입에 동일한 저장 규칙이 일관되게 적용되는 경우에 사용하는 것이 적절하다.</p>
<h3>AttributeConverter는 언제 사용하면 좋을까?</h3>
<p>AttributeConverter는 엔티티의 단일 속성과 DB 컬럼 값 사이에 일관된 양방향 변환 규칙이 있을 때 유용하다. 대표적인 예시는 다음과 같다.</p>
<ul>
<li>Boolean &harr; Y/N</li>
<li>Enum &harr; 코드</li>
<li>값 객체 &harr; 문자열</li>
<li>객체 &harr; JSON</li>
<li>평문 &harr; 암호문</li>
<li>날짜&nbsp;타입&nbsp;&harr;&nbsp;특정&nbsp;문자열&nbsp;형식</li>
</ul>
<p>하지만 모든 변환을 Converter로 해결하는 것이 좋은 것은 아니다. 다음과 같이 복잡한 DB 매핑이 필요하다면 별도의 엔티티나 @Embeddable이 더 적합할 수 있다.</p>
<ul>
<li>하나의 객체가 여러 컬럼으로 분리되는 경우</li>
<li>변환 과정에서 외부 API 호출이 필요한 경우</li>
<li>검색 조건마다 다른 변환 전략이 필요한 경우</li>
<li>다른 엔티티와의 연관관계가 필요한 경우</li>
<li>변환 결과가 DB 종류나 실행 환경에 강하게 의존하는 경우</li>
</ul>
<p>즉, Converter는 하나의 속성을 하나의 DB 컬럼 표현으로 변환하는 단순하고 결정적인 로직에 가장 잘 어울린다.</p>
<h2>정리</h2>
<p>관련된 내용을 정리하면 아래와 같다.</p>
<ol>
<li>convertToDatabaseColumn은 엔티티 값을 DB 값으로 변환한다.</li>
<li>convertToEntityAttribute는 DB 값을 엔티티 값으로 변환한다.</li>
<li>@Convert를 사용하면 특정 엔티티 필드에 Converter를 적용할 수 있다.</li>
<li>autoApply = true를 사용하면 특정 타입에 자동 적용할 수 있지만 적용 범위를 주의해야 한다.</li>
<li>JdbcTemplate을 사용하면 Converter를 거치지 않은 실제 DB 값을 확인할 수 있다.</li>
</ol>
<p>AttributeConverter는 단순한 편의 기능처럼 보이지만, 엔티티 모델과 데이터베이스 표현을 분리하는 데 유용한 기능이다. 서비스 코드에서는 도메인에 적합한 타입을 사용하고, 데이터베이스 저장 형식은 Converter가 담당하도록 만들 수 있다.</p>
<p>즉, <span style="color: #333333; text-align: start;">AttributeConverter를 사용하면 엔티티와 DB 사이에서 반복되는 변환 로직을 한 곳으로 모을 수 있으며 </span>중복되는 변환 코드를 줄이고, 엔티티 모델을 보다 명확하게 유지할 수 있다.</p>