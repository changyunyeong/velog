<h2 id="resilience4j">Resilience4j</h2>
<p>Resilience4j 란,
함수형 프로그래밍(functional programming)으로 설계된 경량의 내결함성(fault tolerance) 라이브러리이다.</p>
<p>** Circuit Breaker **
<em>추후에 제대로 circuit breaker를 이용해서 TC를 작성해봐야겠다.</em></p>
<p>상태
<img alt="" src="https://velog.velcdn.com/images/retrina0678/post/9a73e455-6d3c-4066-9e03-4af1c4f4d34b/image.png" /></p>
<ul>
<li>일반 상태<ul>
<li>closed: 정상적으로 요청 처리 가능</li>
<li>open: 장애 상황으로 간주, 이후 모든 요청 거부</li>
<li>half_open: 장애 여부 확인하기 위해 일시적으로 호출 허용, 결과에 따라 상태 전환</li>
</ul>
</li>
<li>특수 상태<ul>
<li>disabled: circuit breaker 비활성화</li>
<li>forced_open: 특정 조건에서 강제로 open, 즉 특정 조건에서는 요청 거부 </li>
</ul>
</li>
</ul>
<p>circuit Breaker는 슬라이딩 윈도우 기반으로 호출 결과를 기록 및 집계하고 그 결과에 따라 상태를 전환한다. 
기본은 closed 상태로 시작, 서비스의 호출 결과는 성공 또는 실패로 기록한다.
이때 실패의 비율이 사전에 정의한 임계치를 초과하면 장애 상태로 판단해 open 상태로 전환한다.
이후로는 설정에 따라 상태를 전환할 수 있다.</p>
<p>이번 실험에선 circuit breaker 옵션을 yml에 설정했다.</p>
<pre><code class="language-yml">resilience4j:
  circuitbreaker:
    instances:
      viewClient:
        slidingWindowType: COUNT_BASED
        slidingWindowSize: 20
        minimumNumberOfCalls: 10
        failureRateThreshold: 50
        slowCallDurationThreshold: 2s
        slowCallRateThreshold: 50
        waitDurationInOpenState: 10s
        permittedNumberOfCallsInHalfOpenState: 5</code></pre>
<p>옵션에 대해 하나씩 알아보겠다.</p>
<ul>
<li>slidingWindowType
COUNT_BASED, TIME_BASED 중 택</li>
<li>slidingWindowSize<ul>
<li>COUNT_BASED: 개수 단위, 위 경우엔 20개 요청 수</li>
<li>TIME_BASED: 초 단위 </li>
</ul>
</li>
<li>minimumNumberOfCalls
임계치 초과 여부를 판단하기 위한 최소 호출 수 
만약 해당 값이 없다면 한 개의 요청이 실패했을 때 실패율이 100프로가 되어 상태가 변할 수 있다</li>
<li>failureRateThreshold
호출 결과 실패율에 대한 임계치 </li>
<li>slowCallDurationThreshold
느린 요청의 판단 기준 </li>
<li>slowCallRateThreshold
느린 요청 비율에 대한 임계치 설정
해당 비율이 임계치 이상이면 open으로 상태 변환</li>
<li>waitDurationInOpenState
open -&gt; half_open으로 상태 전환하기 위한 대기 시간 설정</li>
<li>permittedNumberOfCallsInHalfOpenState
half_open일 때 허용 가능한 호출 수 </li>
</ul>
<h2 id="실험-배경--목표">실험 배경 &amp; 목표</h2>
<p>장애 상황: view-service에 일부러 Thread.sleep(3000)을 넣어서 조회수 조회가 3초씩 걸린다.
보호 장치가 없다면 article-read 응답이 3초씩 지연되기 때문에 서비스 전체에 장애가 생길 수 있다.
하여 Rate Limiting으로 안전밸브를 두어 얼마까지 버틸 수 있나 실험해 볼 것이다.</p>
<p>Resilience4j로 View 호출을 감싸고, RateLimitInterceptor로 해당 api 진입 자체를 제한해 방어막을 두는 형태이다.</p>
<pre><code class="language-java">    @OptimizedCacheable(type = &quot;articleViewCount&quot;, ttlSeconds = 1)
    @CircuitBreaker(name = &quot;viewClient&quot;, fallbackMethod = &quot;countFallback&quot;)
    @Retry(name = &quot;viewClient&quot;)
    public long count(Long articleId) {
        log.info(&quot;[ViewClient.count] articleId={}&quot;, articleId);

        Long result = restClient.get()
                .uri(&quot;/v1/articles-views/articles/{articleId}/count&quot;, articleId)
                .retrieve()
                .body(Long.class);

        return result != null ? result : 0L;
    }

    // Circuit open / 예외 발생 시 degrade 전략
    private long countFallback(Long articleId, Throwable throwable) {
        log.warn(&quot;[ViewClient.countFallback] degrade to 0, articleId={}, cause={}&quot;,
                articleId, throwable.toString());
        return 0L; // 캐시도 없으면 0으로 처리
    }</code></pre>
<p>예외를 try-catch로 막지 않아 circuit breaker가 인식하도록 하였다. </p>
<h2 id="실험-결과">실험 결과</h2>
<p>이번에도 실험은 k6로 진행하였다. </p>
<table>
<thead>
<tr>
<th>LIMIT</th>
<th>전체 RPS</th>
<th>성공 RPS</th>
<th>전체 p95 (ms)</th>
<th>2xx p95 (ms)</th>
</tr>
</thead>
<tbody><tr>
<td>50</td>
<td>≈ 4317</td>
<td>≈ 50</td>
<td>≈ 4.15</td>
<td>≈ 29.39</td>
</tr>
<tr>
<td>100</td>
<td>≈ 4514</td>
<td>≈ 100</td>
<td>≈ 4.15</td>
<td>≈ 12.93</td>
</tr>
<tr>
<td>200</td>
<td>≈ 2919</td>
<td>≈ 200</td>
<td>≈ 7.61</td>
<td>≈ 22.22</td>
</tr>
</tbody></table>
<p>자세한 값은 계산하진 않았지만 대략적으로 봤을 때 성공 rps가 limit과 거의 일치한다. 
p95도 ms 수준으로 유지되고, 실제 비즈니스 로직까지 들어간 성공 응답으로 봤을 땐 더 낮은 숫자를 유지한다. </p>
<p>즉, Rate limiting이 n rps만 받아들이고 나머지는 429로 처리하면서 서비스 부하를 예방했다. </p>
<p>Resilience4j에서는 느린 요청 임계치를 넘어서면 바로 상태가 바뀌기 때문에 빠른 응답과 서비스 기능을 유지할 수 있다.</p>
<p>번외로, limiting을 해제했을 때도 테스트해보았는데,
<img alt="" src="https://velog.velcdn.com/images/retrina0678/post/9ff2ce75-dcaf-4bf7-8a2d-27ed912c0f31/image.png" /></p>
<p>예상과는 다르게 엄청난 차이를 보이진 않았다.
러프하게 보자면, p95가 20ms 정도 나와서 약 5배 정도 차이가 난다.
다만 게시글 읽기라는 가벼운 서비스였기 때문에 서비스가 터지지 않고 잘 유지되었을 수도 있다.
숫자에서 차이를 보인 건 분명하므로 서비스가 더 커지거나 요청이 무거워진다면 분명 차이를 보일 것이다. </p>
<hr />
<h3 id="참고자료">참고자료</h3>
<p><a href="https://resilience4j.readme.io/docs">https://resilience4j.readme.io/docs</a>
<a href="https://meetup.nhncloud.com/posts/385">https://meetup.nhncloud.com/posts/385</a></p>