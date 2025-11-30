<blockquote>
<p>조회수 카운트를 얼마나 정확하게 잡아야 하고, 그 대신 인프라 비용을 얼마나 감당할 수 있을까?</p>
</blockquote>
<p>그래서 같은 유저가 새로고침(f5) 를 1000번 연타하는 상황과,
서로 다른 유저 100명이 들어오는 상황을 나눠서 redis, db 수를 비교해서 일관성과 비용을 분석해볼 것이다.</p>
<pre><code>public Long increase(Long articleId, Long userId) {

    // 1) 동일 유저 중복 조회 필터링
    if (!articleViewDistributedLockRepository.lock(articleId, userId, TTL)) {
        return articleViewCountRepository.read(articleId);
    }

    // 2) Redis 카운터 증가
    Long count = articleViewCountRepository.increase(articleId);

    // 3) 배치 백업 + 이벤트 발행
    if (count % BACK_UP_BATCH_SIZE == 0) {
        articleViewCountBackUpProcessor.backUp(articleId, count);
    }
    return count;
}</code></pre><h2 id="per-user-락">per-user 락</h2>
<pre><code>public boolean lock(Long articleId, Long userId, Duration ttl) {
    String key = &quot;view::article::%s::user::%s::lock&quot;.formatted(articleId, userId);
    return redisTemplate.opsForValue().setIfAbsent(key, &quot;&quot;, ttl);
}</code></pre><p>첫 조회시엔 setIfAbsent가 True를 반환해 view count가 증가되고, ttl 내에 다시 조회를 하면 락 재획득에 실패하기 때문에 조회수가 증가하지 않는다. </p>
<p>db 백업은 redis view count가 BACK_UP_BATCH_SIZE의 배수일 때만 수행한다. </p>
<p><strong>같은 유저가 새로고침(f5) 를 1000번 연타</strong>
동일한 유저가 같은 글을 1000번 조회, 즉 스팸성 트래픽이 어떻게 반영되는지 확인할 것이다.</p>
<table>
<thead>
<tr>
<th>BACK_UP_BATCH_SIZE</th>
<th>groundTruth(요청 수)</th>
<th>Redis viewCount</th>
<th>DB viewCount</th>
<th>elapsed(ms)</th>
<th>비고</th>
</tr>
</thead>
<tbody><tr>
<td>1</td>
<td>1000</td>
<td>1</td>
<td>1</td>
<td>638</td>
<td>유니크 뷰 1, DB까지 즉시 반영</td>
</tr>
<tr>
<td>10</td>
<td>1000</td>
<td>1</td>
<td>0</td>
<td>560</td>
<td>DB 미반영</td>
</tr>
<tr>
<td>100</td>
<td>1000</td>
<td>1</td>
<td>0</td>
<td>~560</td>
<td>(이전 실험) DB 미반영</td>
</tr>
<tr>
<td>1000</td>
<td>1000</td>
<td>1</td>
<td>0</td>
<td>885</td>
<td>DB 미반영</td>
</tr>
</tbody></table>
<p>per-user 락 덕분에, F5 스팸 1000번은 유니크 뷰 1회로 처리한다.
BACK_UP_BATCH_SIZE가 1이 아닐 때 db에 기록되지 않은 건 배치 조건을 미충족하기 때문이다.</p>
<p><strong>서로 다른 유저 100명이 들어오는 상황</strong>
100개에 스레드를 놓고 각 스레드당 10회 요청해 부하 테스트를 진행할 것이다.</p>
<table>
<thead>
<tr>
<th>BACK_UP_BATCH_SIZE</th>
<th>groundTruth(요청 수)</th>
<th>Redis viewCount</th>
<th>DB viewCount</th>
<th>elapsed(ms)</th>
<th>비고</th>
</tr>
</thead>
<tbody><tr>
<td>1</td>
<td>919</td>
<td>100</td>
<td>100</td>
<td>1449</td>
<td>DB/이벤트 부하 때문에 1000건을 다 못 쏨 (타임아웃)</td>
</tr>
<tr>
<td>10</td>
<td>919</td>
<td>100</td>
<td>100</td>
<td>611</td>
<td>마찬가지로 일부 요청 미완료</td>
</tr>
<tr>
<td>100</td>
<td>1000</td>
<td>100</td>
<td>100</td>
<td>~580</td>
<td>1000건 모두 처리, Redis/DB 일치</td>
</tr>
<tr>
<td>1000</td>
<td>1000</td>
<td>100</td>
<td>0</td>
<td>551</td>
<td>DB 백업 조건 미도달 → DB=0, Redis에만 100 저장</td>
</tr>
</tbody></table>
<p>각기 다른 100명의 유저를 가정했기에 유니크 뷰도 100으로 잘 확인됐다.
배치 사이즈가 100 이하일 땐 redis와 db의 일관성이 잘 유지된다.
그러나 배치가 커질 땐 db에 카운트가 기록되지 않아 일관성이 깨졌다. 만약 이때 redis에 장애가 생긴다면 100개의 뷰는 아예 사라질 수도 있다.</p>
<p>요청 수가 1000개가 안 될 때도 있는데, 배치가 작을수록 처리 시간이 늘어나서 타임아웃 내에 요청을 다 처리하지 못했기 때문이다. </p>
<h2 id="결론">결론</h2>
<p><strong>유니크 뷰 관점</strong>
동일 유저의 스팸은 유니크 1회로 처리하고, 많은 유저가 들어오는 글은 유저 수만큼 집계됐다.
이는 ttl 기반 per-user lock이 잘 동작한다는 것을 확인할 수 있다.</p>
<p><strong>db 관점</strong></p>
<ul>
<li>BACK_UP_BATCH_SIZE = 1, 10
redis와 db간의 일관성이 잘 유지된다.
그러나 부하가 커지면서 트래픽도 db에 반영될 수 있다.</li>
<li>BACK_UP_BATCH_SIZE = 100
적당한 부하에도 날리는 요청 없이 잘 처리했다.
이번 실험 중에선 가장 적절하다고 볼 수 있지만 아주 강한 일관성을 제공하진 못한다.</li>
<li>BACK_UP_BATCH_SIZE = 1000
db에 write를 하지 않기 때문에 부하가 제일 적다.
그러나 db가 너무 느리게 따라와서 뷰 카운트 0으로 보이는 시간이 길어진다.</li>
</ul>
<p>그래서,
배치를 하나만 두는 것은 너무 과투자에 가깝고, 배치를 1000개 두는 건 데이터의 유실 리스크가 크다.
그러기에 해당 프로젝트에서는 배치를 100정도 두는 게 최적이라고 볼 수 있다.</p>