<h2>Feature Flag는 왜 필요할까?</h2>
<p><span>서비스를 운영하다 보면 코드를 배포하는 일과 기능을 사용자에게 공개하는 일을 항상 같은 시점에 하고 싶지는 않다.</span></p>
<p><span>기능 개발은 끝났지만 아직 전체 사용자에게 열기에는 부담스러운 경우가 있다. 내부 계정이나 특정 조직에만 먼저 열어보고 싶은 기능도 있다. 반대로 이미 배포된 기능에서 문제가 생겼을 때, 코드를 다시 배포하지 않고 기능만 빠르게 끄고 싶은 경우도 있다.</span></p>
<p><figure class="imageblock widthContent"><span><img height="1060" src="https://blog.kakaocdn.net/dn/Kvzul/dJMcaayUtIN/u5My6qtUdrE3NBVbwq8pO0/img.png" width="2052" /></span></figure>
</p>
<p><span>이런 문제를 해결하기 위해 사용하는 방법 중 하나가 Feature Flag다. Feature Flag는 배포된 코드의 동작을 런타임에 바꿀 수 있게 해주는 장치다. </span><span>처음에는 Feature Flag를 단순히 &ldquo;새 기능을 숨겨두는 if문&rdquo; 정도로 생각했다. 그런데 여러 회사의 사례를 찾아보니, 실제로는 그보다 조금 더 넓게 사용되고 있었다.</span></p>
<p><span>Feature Flag의 핵심은 &ldquo;코드를 다시 배포하지 않고 동작을 바꿀 수 있다&rdquo;는 점이다. <span>코드는 이미 운영 환경에 배포되어 있다. 하지만 실제 요청에서 어떤 로직을 실행할지는 플래그 설정에 따라 달라진다. </span></span><span>이러한 구조 덕분에 기능 공개 시점을 배포 시점과 분리할 수 있고, 특정 사용자에게만 기능을 열 수도 있고, 문제가 생긴 기능을 빠르게 끌 수도 있다.</span></p>
<h2>Feature Flag 사용 사례</h2>
<h3 style="color: #000000; text-align: start;"><span>운영 중 시스템 동작을 바꾸는 스위치</span></h3>
<p><figure class="imageblock widthContent"><span><img height="1024" src="https://blog.kakaocdn.net/dn/nfndd/dJMcahLxv1x/C94NCUdVZkaDTu3s4GtVGk/img.png" width="1426" /></span></figure>
</p>
<h4>배포와 릴리즈의 분리</h4>
<p><span>Feature Flag가 가장 자주 언급되는 이유는 배포와 릴리즈를 분리할 수 있기 때문이다. </span></p>
<p><span>배포는 코드가 운영 환경에 올라가는 일이다. </span><span>릴리즈는 사용자가 그 기능을 실제로 사용할 수 있게 되는 일이다.</span></p>
<p><span>Feature Flag가 없으면 이 둘이 거의 동시에 일어난다. 코드를 배포하면 기능도 함께 열린다. 그래서 배포 자체가 사용자에게 영향을 주는 이벤트가 된다. </span><span>Feature Flag를 사용하면 코드는 먼저 배포하고, 기능 노출은 나중에 결정할 수 있다.</span></p>
<p>&nbsp;</p>
<p><span>당근의 Feature Toggle 글에서 인상 깊었던 문제 상황도 위와 같다.</span></p>
<p><span>내가 작업한 코드가 mainline에 들어갔다. 그런데 아직 테스트 중이라 사용자에게 노출되면 안 된다. 하지만 이 상태에서 다른 동료가 배포를 진행하면 mainline에 있는 내 코드도 함께 배포된다. 때문에, </span><span>테스트가 끝날 때까지 팀원이 배포를 기다리거나, 이미 merge한 코드를 revert해야 한다. 둘 다 자주 배포하는 팀에는 부담스러운 방식이다.</span></p>
<p><span>Feature Flag를 사용하면 다른 선택지가 생긴다. 코드는 mainline에 포함하고 배포하되, 사용자에게 노출되는 경로는 플래그로 막아둘 수 있다.</span></p>
<pre class="kotlin"><code>fun getHome(user: User): HomeResponse {
    if (featureFlag.isEnabled("home.new-layout", user)) {
    	return newHomeService.getHome(user) 
    } 
    return legacyHomeService.getHome(user) 
}</code></pre>
<p><span>플래그가 꺼져 있으면 기존 홈 화면만 동작한다. 신규 홈 화면 코드는 운영 환경에 올라가 있지만 사용자에게는 보이지 않는다.</span></p>
<p><span>이 방식은 기능 개발을 오래된 브랜치에 쌓아두지 않게 해준다. 변경사항을 작게 나눠 mainline에 계속 통합할 수 있고, 실제 공개 여부는 나중에 결정할 수 있다. </span></p>
<p><span>Feature Flag의 첫 번째 가치는 여기에 있다. </span><span>기능을 배포하지 않는 것이 아니라, 배포는 하되 아직 릴리즈하지 않는 상태를 만들 수 있다.</span></p>
<h4>장애 대응을 위한 스위치로 사용</h4>
<p><span>운영 중 특정 기능에서 문제가 생겼을 때, 코드를 수정하고 다시 배포하는 방식은 느릴 수 있다. 이미 우회 경로가 준비되어 있다면 플래그를 끄는 것만으로 문제 기능을 빠르게 비활성화할 수 있다. </span><span>예를 들어 홈 화면의 추천 영역이 있다고 해보자. 추천 서버가 불안정해져서 홈 화면 전체 응답 시간이 느려지고 있다면, 추천 영역만 잠시 끄는 선택을 할 수 있다.</span></p>
<pre class="kotlin" id="code_1781437421204"><code>fun getHome(user: User): HomeResponse {
    val banners = bannerService.getBanners(user)
    val recommendations =
    if (featureFlag.isEnabled("home.recommendation.enabled")) {
    	recommendationService.getRecommendations(user)
    } else {
    	emptyList()
    }
    
    return HomeResponse(
        banners = banners,
        recommendations = recommendations,
    )
}</code></pre>
<p>이런 플래그는 신규 기능 릴리즈용 플래그와 성격이 다르다. 릴리즈 플래그는 기능 공개가 끝나면 제거하는 것이 보통이다. 반면 킬 스위치는 장기적으로 유지될 수 있다. 장애 상황에서 특정 기능을 빠르게 끄기 위한 운영 장치이기 때문이다.</p>
<p>다만 킬 스위치를 만들 때는 반드시 fallback이 있어야 한다. Feature Flag는 기능을 꺼주는 스위치일 뿐이다. 기능을 껐을 때 시스템이 안전하게 동작하도록 만드는 것은 별도의 설계 문제다. 스위치만 있고 우회 경로가 없다면, 그것은 킬 스위치가 아니라 또 다른 장애 포인트가 될 수 있다.</p>
<p>&nbsp;</p>
<p>때문에 추천 영역을 끄면 화면이 어떻게 보여야 하는지, 외부 API 호출을 끄면 데이터는 어디에 남길지, 나중에 재처리할 수 있는지 같은 fallback 설계가 먼저 있어야 한다.</p>
<pre class="kotlin" id="code_1781440315787"><code>fun publishOrderCompleted(event: OrderCompletedEvent) { 
    if (!featureFlag.isEnabled("crm.order-completed-publish")) { 
        pendingCrmEventRepository.save(event) 
        return 
    } 
    crmClient.publish(event) 
}</code></pre>
<p><span>위 예시에서 핵심은 외부 전송을 그냥 버리지 않는 것이다. 플래그가 꺼져 있으면 pending 상태로 저장하고 나중에 재처리할 수 있어야 한다.</span></p>
<h4><span>실험 설정을 코드 배포 없이 바꾸기</span></h4>
<p><span>LINE NEWS 사례는 Feature Flag가 단순 ON/OFF를 넘어 Remote Configuration으로 확장되는 흐름을 잘 보여준다.</span></p>
<p><span>기존에는 A/B 테스트를 하려면 실험 이름, 그룹, 각 그룹의 동작을 프론트엔드 코드에 직접 매핑해야 했다. 실험 조건이 조금만 바뀌어도 코드 수정과 배포가 필요했다. 배포 주기가 2주라면, 작은 실험 하나를 수정하는 데도 리드타임이 길어질 수밖에 없다.</span></p>
<p><span>이 문제를 해결하려면 기능을 켜고 끄는 것만으로는 부족하다. 실험군마다 문구, 노출 개수, 정렬 방식, UI 표현 같은 값도 바꿀 수 있어야 한다. </span><span>예를 들어 홈 화면 추천 영역을 실험한다고 해보자.</span></p>
<pre class="kotlin" id="code_1781437580949"><code>{ 
    "control": { 
        "title": "추천 콘텐츠", 
        "initialCount": 30 
    }, 
    "treatment": { 
        "title": "회원님을 위한 추천", 
        "initialCount": 10 
    } 
}</code></pre>
<p><span>이런 값이 코드에 하드코딩되어 있으면 실험을 조정할 때마다 배포가 필요하다. 반대로 Remote Configuration으로 관리하면 배포 없이 실험 설정을 바꿀 수 있다. </span><span>이때 Feature Flag는 &ldquo;이 기능을 켤 것인가&rdquo;를 결정하고, Remote Configuration은 &ldquo;이 기능을 어떤 값으로 동작시킬 것인가&rdquo;를 결정한다. </span><span>실제 서비스에서는 둘이 함께 쓰이는 경우가 많다.</span></p>
<h4 style="color: #000000; text-align: start;"><span>특정 대상에게만 기능 제공하기</span></h4>
<p style="color: #333333; text-align: start;"><span>카카오페이의 Feature Flag 플랫폼 사례를 보면 단순 ON/OFF뿐 아니라 배포 비율 조정과 유저 타겟팅도 요구사항에 포함되어 있다.</span></p>
<p style="color: #333333; text-align: start;"><span>모든 기능이 모든 사용자에게 동시에 열릴 필요는 없다. 내부 직원에게만 먼저 열어볼 수도 있고, 특정 조직이나 특정 앱 버전 이상에서만 기능을 제공할 수도 있다. 유료 요금제나 관리자 권한을 가진 사용자에게만 보여야 하는 기능도 있다.<span>&nbsp;</span></span><span>이런 경우 Feature Flag는 단순한 boolean 값이 아니라, 현재 요청의 context를 기반으로 평가된다.</span></p>
<pre id="code_1781448087910" style="background-color: #f8f8f8; color: #383a42; text-align: start;"><code>val context = FeatureContext( 
    userId = user.id, 
    organizationId = user.organizationId, 
    appVersion = request.appVersion, 
    plan = user.plan, 
) 

if (featureFlag.isEnabled("dashboard.new-admin", context)) { 
    return newAdminDashboardService.getDashboard(user) 
} 
return legacyDashboardService.getDashboard(user)</code></pre>
<p style="color: #333333; text-align: start;"><span>여기서 중요한 것은 &ldquo;누구에게 켤 것인가&rdquo;를 코드에 박아두지 않는다는 점이다.<span>&nbsp;</span></span><span>코드에는 플래그 평가 지점만 남기고, 실제 대상 조건은 외부 설정으로 관리한다. 그래야 운영 중에도 대상 조직을 추가하거나, 특정 앱 버전만 제외하거나, 내부 사용자에게만 기능을 열 수 있다.</span></p>
<h4>안전하게 마이그레이션하기</h4>
<p><span>Feature Flag가 꼭 사용자 노출 기능에만 쓰이는 것은 아니다. </span><span>당근페이 아키텍처 전환 사례처럼, 기존 구조에서 새로운 구조로 점진적으로 옮겨갈 때도 사용할 수 있다.</span></p>
<p><span>기존 로직을 한 번에 모두 제거하고 새 로직으로 바꾸는 것은 위험하다. 특히 결제, 송금, 정산처럼 영향도가 큰 도메인에서는 더 그렇다. 이때 기존 로직과 신규 로직을 함께 두고, 플래그로 실행 경로를 제어할 수 있다.</span></p>
<pre class="kotlin" id="code_1781440182732"><code>fun transfer(command: TransferCommand): TransferResult { 
	if (featureFlag.isEnabled("transfer.use-new-usecase")) { 
    	return newTransferUseCase.execute(command) 
    } 
    return legacyTransferService.transfer(command) 
}</code></pre>
<p><span>처음에는 기존 경로를 사용한다. </span><span>새 구조가 준비되면 개발 환경이나 일부 조건에서만 신규 경로를 켠다. </span><span>문제가 없으면 적용 범위를 넓히고 </span><span>안정화되면 기존 경로를 제거한다. </span><span>이 방식은 Strangler Fig Pattern과도 잘 맞는다. 기존 시스템을 한 번에 갈아엎는 대신, 기능 단위로 조금씩 새 구조로 옮겨갈 수 있기 때문이다.</span></p>
<h3>Feature Flag 관리</h3>
<h4>플래그 서빙 경로는 가벼워야 한다</h4>
<p><span>Feature Flag는 사용자 요청 흐름 안에서 자주 평가된다. </span><span>그래서 플래그 값을 확인할 때마다 DB를 조회하거나, 복잡한 어드민 서버를 호출하면 안 된다. Feature Flag 자체가 병목이 될 수 있고, 플래그 시스템 장애가 서비스 장애로 전파될 수 있다.</span></p>
<p><span>요청 처리 경로에서는 가능한 한 가볍게 평가해야 한다. 로컬 캐시나 인메모리 설정을 사용하고, 설정 변경은 이벤트나 주기적 동기화로 반영하는 방식이 일반적이다. </span><span>Feature Flag는 런타임 제어 장치이기 때문에 실시간성이 중요하며 동시에 사용자 요청 흐름을 느리게 만들면 안 된다.</span></p>
<h4><span>플래그는 운영 데이터다</span></h4>
<p><span>Feature Flag를 코드 안의 if문으로만 보면 관리가 소홀해지기 쉽다. 하지만 운영 환경에서 플래그 값을 바꾸는 일은 배포만큼 영향도가 클 수 있다. </span><span>특정 기능이 사용자에게 노출될 수도 있고, 실험군이 바뀔 수도 있고, 내부 구현 경로가 변경될 수도 있다. 장애 대응용 플래그라면 특정 기능이 즉시 꺼질 수도 있다. </span><span>따라서 플래그에는 최소한 아래 정보가 필요하다.</span></p>
<pre class=""><code>이 플래그는 어떤 목적으로 만들어졌는가
누가 소유자인가
기본값은 무엇인가
어떤 환경에서 사용할 수 있는가
누가 변경할 수 있는가
언제 제거해야 하는가
변경 이력은 남고 있는가</code></pre>
<p><span>특히 변경 이력은 중요하다. </span><span>장애가 발생하면 보통 최근 배포부터 확인한다. 하지만 Feature Flag를 적극적으로 사용하는 시스템에서는 최근 플래그 변경도 함께 확인해야 한다. </span><span>배포는 없었는데 서비스 동작이 바뀌었다면, 원인은 플래그 변경일 수 있다.</span></p>
<h4>플래그는 반드시 정리해야 한다</h4>
<p><span>Feature Flag는 편리하지만 코드에 분기를 남긴다.</span></p>
<pre class="isbl"><code>if (featureFlag.isEnabled("some-feature")) {
    newPath()
} else {
    oldPath()
}</code></pre>
<p><span>플래그가 하나일 때는 괜찮다. 하지만 시간이 지나면서 이런 분기가 늘어나면 코드 경로가 복잡해진다. 테스트해야 할 조합도 늘어난다. 이미 사용하지 않는 old path가 계속 남아 있을 수도 있다. </span><span>당근의 글에서도 사용하지 않는 Feature Toggle은 바로 제거해야 한다고 말한다. 이 부분은 실제 운영에서 특히 중요하다. </span><span>릴리즈가 끝난 플래그는 제거해야 한다. </span><span>실험이 끝난 플래그도 정리해야 한다. </span><span>아키텍처 전환이 끝났다면 기존 경로와 전환 플래그를 삭제해야 한다. </span><span>장기 운영 플래그라면 왜 계속 유지해야 하는지 설명이 있어야 한다.</span></p>
<p><span>Feature Flag를 잘 쓰는 팀은 플래그를 많이 만드는 팀이 아니라, 필요 없어진 플래그를 잘 지우는 팀에 가깝다.</span></p>
<h2><span>마무리</span></h2>
<p><span>Feature Flag는 단순히 새 기능을 숨겨두는 if문이 아니다. </span><span>배포된 코드의 동작을 런타임에 바꾸기 위한 장치다. 이를 통해 배포와 기능 공개를 분리할 수 있고, 특정 사용자나 조직에만 기능을 열 수 있고, 실험 설정을 배포 없이 바꿀 수 있고, 아키텍처 전환을 점진적으로 진행할 수 있고, 문제가 생긴 기능을 빠르게 끌 수 있다.</span></p>
<p><span>다만 Feature Flag는 스위치일 뿐이다. </span><span>스위치를 켜고 끄는 대상이 안전하게 설계되어 있어야 한다. </span><span>플래그 평가 경로는 가벼워야 한다. </span><span>기본값과 fallback이 명확해야 한다. </span><span>변경 이력과 권한 관리가 필요하다. </span><span>필요 없어진 플래그는 반드시 제거해야 한다. </span><span>결국 Feature Flag를 잘 쓴다는 것은 if문을 잘 넣는 것이 아니다. </span><span>운영 중인 시스템의 동작을 코드 배포 없이 안전하게 바꿀 수 있도록, 기능 노출 방식과 설정 관리, 서빙 구조, 변경 이력, 제거 정책까지 함께 설계하는 일에 가깝다.</span></p>
<h3 style="color: #000000; text-align: start;">실습 코드</h3>
<p style="color: #333333; text-align: start;"><a href="https://github.com/wldks1008/feature-flag-pattern">https://github.com/wldks1008/feature-flag-pattern</a></p>
<figure contenteditable="false" id="og_1781440812617"><a href="https://github.com/wldks1008/feature-flag-pattern" rel="noopener" target="_blank">
<div class="og-image">&nbsp;</div>
<div class="og-text">
<p class="og-title">GitHub - wldks1008/feature-flag-pattern: feature flag 패턴 실습</p>
<p class="og-desc">feature flag 패턴 실습. Contribute to wldks1008/feature-flag-pattern development by creating an account on GitHub.</p>
<p class="og-host">github.com</p>
</div>
</a></figure>
<h3><span>참고자료</span></h3>
<ul>
<li><a href="https://docs.getunleash.io/get-started/what-is-a-feature-flag" rel="noopener&nbsp;noreferrer" target="_blank">https://docs.getunleash.io/get-started/what-is-a-feature-flag</a></li>
</ul>
<figure contenteditable="false" id="og_1781444689500"><a href="https://docs.getunleash.io/get-started/what-is-a-feature-flag" rel="noopener" target="_blank">
<div class="og-image">&nbsp;</div>
<div class="og-text">
<p class="og-title">What is a Feature Flag? Complete Guide to Feature Flags</p>
<p class="og-desc">Feature flags let you release, test, and manage features without redeploying. Here's what they are, why teams use them, and how they work.</p>
<p class="og-host">docs.getunleash.io</p>
</div>
</a></figure>
<p>&nbsp;</p>
<ul>
<li><a href="https://tech.kakaopay.com/post/feature-flag/" rel="noopener&nbsp;noreferrer" target="_blank">https://tech.kakaopay.com/post/feature-flag/</a></li>
</ul>
<figure contenteditable="false" id="og_1781440588314"><a href="https://tech.kakaopay.com/post/feature-flag/" rel="noopener" target="_blank">
<div class="og-image">&nbsp;</div>
<div class="og-text">
<p class="og-title">피처 플래그 개발기: 실시간 데이터 동기화를 향한 여정 | 카카오페이 기술 블로그</p>
<p class="og-desc">원활한 배포 및 장애 대응을 위한 피처 플래그를 개발하며 실시간 데이터 동기화를 위해 직면한 문제를 해결하기 위한 여정을 공유합니다.</p>
<p class="og-host">tech.kakaopay.com</p>
</div>
</a></figure>
<ul>
<li><a href="https://medium.com/daangn/%EB%A7%A4%EC%9D%BC-%EB%B0%B0%ED%8F%AC%ED%95%98%EB%8A%94-%ED%8C%80%EC%9D%B4-%EB%90%98%EB%8A%94-%EC%97%AC%EC%A0%95-2-feature-toggle-%ED%99%9C%EC%9A%A9%ED%95%98%EA%B8%B0-b52c4a1810cd" rel="noopener&nbsp;noreferrer" target="_blank">https://medium.com/daangn/%EB%A7%A4%EC%9D%BC-%EB%B0%B0%ED%8F%AC%ED%95%98%EB%8A%94-%ED%8C%80%EC%9D%B4-%EB%90%98%EB%8A%94-%EC%97%AC%EC%A0%95-2-feature-toggle-%ED%99%9C%EC%9A%A9%ED%95%98%EA%B8%B0-b52c4a1810cd</a></li>
</ul>
<figure contenteditable="false" id="og_1781440610126"><a href="https://medium.com/daangn/%EB%A7%A4%EC%9D%BC-%EB%B0%B0%ED%8F%AC%ED%95%98%EB%8A%94-%ED%8C%80%EC%9D%B4-%EB%90%98%EB%8A%94-%EC%97%AC%EC%A0%95-2-feature-toggle-%ED%99%9C%EC%9A%A9%ED%95%98%EA%B8%B0-b52c4a1810cd" rel="noopener" target="_blank">
<div class="og-image">&nbsp;</div>
<div class="og-text">
<p class="og-title">매일 배포하는 팀이 되는 여정(2) &mdash; Feature Toggle 활용하기</p>
<p class="og-desc">효율적이고 안정적인 배포를 위해 고민했던 것 중 하나인 Feature Toggle(Feature Flag)에 대한 이야기</p>
<p class="og-host">medium.com</p>
</div>
</a></figure>
<ul>
<li><a href="https://techblog.lycorp.co.jp/en/20231005a" rel="noopener&nbsp;noreferrer" target="_blank">https://techblog.lycorp.co.jp/en/20231005a</a></li>
</ul>
<figure contenteditable="false" id="og_1781440642988"><a href="https://techblog.lycorp.co.jp/en/20231005a" rel="noopener" target="_blank">
<div class="og-image">&nbsp;</div>
<div class="og-text">
<p class="og-title">How We Rearchitected A/B Testing at LINE NEWS</p>
<p class="og-desc">A/B tests play a crucial role in decision-making at LINE NEWS as they help determine what modificati...</p>
<p class="og-host">techblog.lycorp.co.jp</p>
</div>
</a></figure>
<ul>
<li><a href="https://techblog.lycorp.co.jp/en/deployment-strategies-by-feature-toggle" rel="noopener&nbsp;noreferrer" target="_blank">https://techblog.lycorp.co.jp/en/deployment-strategies-by-feature-toggle</a></li>
</ul>
<figure contenteditable="false" id="og_1781440663980"><a href="https://techblog.lycorp.co.jp/en/deployment-strategies-by-feature-toggle" rel="noopener" target="_blank">
<div class="og-image">&nbsp;</div>
<div class="og-text">
<p class="og-title">Easier, Flexible, and Lower Resource Cost Deployment Strategies by Feature Toggle</p>
<p class="og-desc">It's crucial to have various deployment strategies to ensure that new versions of your software is d...</p>
<p class="og-host">techblog.lycorp.co.jp</p>
</div>
</a></figure>