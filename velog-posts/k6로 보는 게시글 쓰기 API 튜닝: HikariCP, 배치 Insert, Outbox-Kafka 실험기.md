<h2 id="hikaricp-커넥션-풀-튜닝">HikariCP 커넥션 풀 튜닝</h2>
<blockquote>
<p>목표</p>
</blockquote>
<ol>
<li>쓰기 요청이 늘어났을 때 대기가 튈까?</li>
<li>HikariCP maximumPoolSize, minimumIdle 조절이 얼마나 영향을 줄까?</li>
</ol>
<p>우선 HikariCP란 connection pooling을 제공하는 JDBC DataSource의 구현체이다.
Hikarism는 미리 커넥션을 풀에 담아두고, 요청이 들어올 때 스레드가 풀을 요청하면 Hikari가 connection을 제공하는 형태이다.</p>
<table>
<thead>
<tr>
<th>실험</th>
<th>maxPoolSize</th>
<th>RPS (req/s)</th>
<th>p95 (ms)</th>
<th>max (ms)</th>
<th>비고</th>
</tr>
</thead>
<tbody><tr>
<td>A</td>
<td>10</td>
<td>193.82</td>
<td>100</td>
<td>308</td>
<td>baseline</td>
</tr>
<tr>
<td>B</td>
<td>30</td>
<td>169.81</td>
<td>108</td>
<td>288</td>
<td></td>
</tr>
<tr>
<td>C</td>
<td>50</td>
<td>165.17</td>
<td>117</td>
<td>259</td>
<td></td>
</tr>
</tbody></table>
<p><img alt="" src="https://velog.velcdn.com/images/retrina0678/post/cdb4c7e6-d41a-45b0-a40c-ffd95d3f2b15/image.png" />
maxPoolSize = 10일 때 해당 시점에 p값이 확 튀는 걸 확인할 수 있다. </p>
<h2 id="jpahibernate-batch-insert">JPA/Hibernate Batch insert</h2>
<blockquote>
<p>목표
단건 n번 insert와 배치 1번 insert의 차이 확인</p>
</blockquote>
<p>Hibernate 배치는 모두 50으로 고정했다.
코드 단에서는 간단하게 /batch api를 추가했다. </p>
<pre><code>@Transactional
    public List&lt;ArticleResponse&gt; createBatch(List&lt;ArticleCreateRequest&gt; requests) {
        return requests.stream()
                .map(this::create)
                .toList();
    }</code></pre><p>이정도?
<br /></p>
<table>
<thead>
<tr>
<th>모드</th>
<th>1요청당 insert 수</th>
<th>req/s</th>
<th>rows/s</th>
<th>p95 (ms)</th>
<th>비고</th>
</tr>
</thead>
<tbody><tr>
<td>단건</td>
<td>1</td>
<td>299</td>
<td>299</td>
<td>63</td>
<td>baseline</td>
</tr>
<tr>
<td>배치 (10개)</td>
<td>10</td>
<td>30.39</td>
<td>303.86</td>
<td>744.41</td>
<td></td>
</tr>
<tr>
<td>배치 (20개)</td>
<td>20</td>
<td>10.31</td>
<td>200.62</td>
<td>2450</td>
<td></td>
</tr>
</tbody></table>
<br />
확실히 배치가 늘어날수록 지연이 늘어나는 것을 확인할 수 있다.
지연은 늘어나도 초당 row는 단건보다 배치 모드에서 더 높다.

<p>오히려 배치가 20개일 때는 단건보다 성능이 떨어진다.
특히 10개일 때와 비교해서 거의 모든 지표면에서 더 성능이 안 좋은 것을 확인할 수 있다.</p>
<p>따라서 단건 모드와 10개의 배치 모드만 비교하자면,
각각 모두 장단점이 있다.</p>
<p>단건 모드는 지연 시간이 적으므로 빠르게 요청을 처리할 수 있지만, 그 수가 많아지면 db에 오헤드가 많이 가해질 수 있다.
배치 모드는 레이어 간 오버헤드를 줄일 수 있지만 클라이언트는 보이는 속도가 느리기 때문에 답답함을 느낄 수 있다.</p>
<h2 id="outboxkafka-릴레이-튜닝">Outbox/Kafka 릴레이 튜닝</h2>
<table>
<thead>
<tr>
<th>실험</th>
<th>relay delay (sec)</th>
<th align="right">batch size</th>
<th>쓰기 p95 (ms)</th>
<th>outbox backlog(추정)</th>
<th>비고</th>
</tr>
</thead>
<tbody><tr>
<td>A</td>
<td>10</td>
<td align="right">100</td>
<td>132</td>
<td>0</td>
<td>baseline</td>
</tr>
<tr>
<td>B</td>
<td>1</td>
<td align="right">1000</td>
<td>116.02</td>
<td>0</td>
<td>저지연 relay</td>
</tr>
<tr>
<td>C</td>
<td>10</td>
<td align="right">100</td>
<td>126.39</td>
<td>0</td>
<td>Kafka 느리게</td>
</tr>
</tbody></table>
<br />

<pre><code>List&lt;Outbox&gt; outboxes = outboxRepository
    .findAllByShardKeyAndCreatedAtLessThanEqualOrderByCreatedAtAsc(
        shard,
        LocalDateTime.now().minusSeconds(10),
        Pageable.ofSize(100)
    );</code></pre><p>실험 A, B에서는 MessageRelay 코드에서 relay delay와 batch size를 조절했다. 
실험 C에서는 kafka에서 ack 설정을 all로 바꾸고 compression을 주어 일부러 성능을 떨어뜨렸다. </p>
<p>outbox backlog는 실험 후에 outbox에 남아 있는 레코드 수로 비교했다.
실험 중에 backlog을 찍어보면</p>
<pre><code>mysql&gt; select count(*) from outbox;
+----------+
| count(*) |
+----------+
|        7 |
+----------+</code></pre><p>outbox에 데이터가 조금씩 쌓인 걸 볼 수 있다.</p>
<p>내 예상과는 다르게 모든 시나리오에서 유의미한 차이점을 찾지 못했다. </p>
<p>relay delay가 증가하면 backlog가 더 오래 쌓여 지연이 증가하거나,
batch size가 증가하면 Kafka 전송 부하가 눈에 보일 정도로 커질 줄 알았다.</p>
<p>테스트의 부하가 그렇게 크지 않았던 것인지 kafka에 지연을 걸어도, 위처럼 제약을 걸어도 성능이 비슷하게 나왔다.
실제 서비스처럼 다른 db나 테이블과의 연산이 없었기 때문에 큰 차이를 보이지 않았다는 것이 나의 생각이다.</p>
<p>추후에 더 공부를 해서 정확한 결론을 얻게 된다면 추가하겠다. </p>
<h3 id="참고-자료">참고 자료</h3>
<hr />
<p><a href="https://techblog.woowahan.com/2664/">https://techblog.woowahan.com/2664/</a></p>