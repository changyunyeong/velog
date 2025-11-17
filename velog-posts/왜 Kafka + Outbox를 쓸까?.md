<p>인프런 쿠케 님의 <a href="https://inf.run/nK6J8">대규모 시스템 설계 강의</a>를 듣고 대규모 시스템에 대해 더 공부하고 정리하고자 시리즈 시작!
<br />
이전 프로젝트에서는 monolithic 서비스로만 개발을 해왔어서 이중 쓰기 문제 (한 번의 요청으로 여러 곳을 동시에 업데이트 해야하는 문제)를 고민해본 적은 없었다.</p>
<p>해당 강의에서는 microservice 아키텍쳐를 사용하기 때문에 하나에 요청이 들어왔을 때 여러 개의 서비스 + 여러 개의 db + kafka 같은 메시지 브로커에 동시적으로 업데이트를 해야한다.</p>
<h3 id="이중-쓰기-문제">이중 쓰기 문제</h3>
<p>현재 제공하고 있는 서비스는 게시글 작성 및 조회, 게시글 조회수, 댓글, 좋아요, 인기글이 있다. 
만약 게시글에 좋아요를 누른다고 가정했을 때 좋아요 수 1 증가 + 게시글 조회수 1 증가 + 인기글 점수 업데이트가 동시에 되어야 하는 것이다. </p>
<p>여기서 </p>
<ol>
<li>DB 저장은 성공, Kafka 전송에 실패</li>
<li>Kafka는 전송 성공, 캐시 증가 중 예외</li>
<li>재시도 로직 실행 중 이벤트 중복 발생 가능
등등의 많은 에러 상황이 발생할 수 있다.</li>
</ol>
<p>이렇게 오류가 발생했을 때 <strong>“어디까지 성공했고 어디서 실패했는지”</strong> 추적하기가 어렵다는 것이 이중 쓰기 문제이다.</p>
<h3 id="outbox-패턴">Outbox 패턴</h3>
<p>Outbox 패턴은 간단하게 메시지를 메시지 브로커에 직접 보내는 것이 아니라, 먼저 아웃박스 테이블 (db)에 저장한 후 별도의 프로세스가 이를 읽어서 kafka 같은 외부 시스템으로 이벤트를 보내는 것이다.</p>
<p>비즈니스 로직 처리 (좋아요 테이블에 insert) -&gt; outbox 테이블에 이벤트 저장 -&gt; outbox 테이블을 폴링하면서 kafka로 레코드 전송 -&gt; 완료되면 해당 이벤트 삭제하거나 상태 업데이트</p>
<pre><code class="language-Outbox.java">public class Outbox {
    private Long id;            // Outbox 레코드 ID
    private String aggregateType; // ARTICLE, LIKE 같은 도메인 타입
    private Long aggregateId;     // 대상 엔티티 ID
    private String eventType;     // LIKE_CREATED, LIKE_DELETED 등
    private String payload;       // 이벤트를 JSON으로 직렬화한 문자열
    private Long shardKey;        // 샤딩/락 분산용 키
    private LocalDateTime createdAt;
}
</code></pre>
<p>프로젝트마다 다르지만 보통은 이런 테이블 형태를 가진다.
outbox에 insert가 비즈니스 로직 저장과 같은 트랜잭션 내에서 커밋된다.</p>
<h3 id="kafka">Kafka</h3>
<p>카프카에 대해선 나중에 더 자세하게 공부하도록 하고, 여기서는 기본적인 것만 정리한다.</p>
<ul>
<li>topic<ul>
<li>그냥 다른 곳에서도 흔히 쓰이는 그런 토픽 맞다 (채널 이름)</li>
</ul>
</li>
<li>partition<ul>
<li>하나의 토픽을 여러 개의 파티션으로 나누어 병렬 처리가 가능</li>
<li>같은 key를 사용하면 같은 파티션으로 분류</li>
</ul>
</li>
<li>consumer group<ul>
<li>같은 이벤트를 그룹 내에서 한 번만 처리 가능하도록 하는 단위</li>
</ul>
</li>
<li>offset<ul>
<li>consumer가 토픽/파티션에서 어디까지 읽었는지 가리키는 포인터</li>
</ul>
</li>
</ul>
<h3 id="outbox--kafka">Outbox + Kafka</h3>
<p>비즈니스 로직 처리 (좋아요 테이블에 insert) -&gt; outbox 테이블에 이벤트 저장 -&gt; outbox 테이블을 폴링하면서 kafka로 레코드 전송 -&gt; 완료되면 해당 이벤트 삭제하거나 상태 업데이트</p>
<p>** 비즈니스 로직 처리 -&gt; outbox 테이블에 이벤트 저장 **
like insert + outbox entity insert </p>
<pre><code>@Transactional
public void like(Long articleId, Long userId) {
    likeRepository.save(new Like(articleId, userId));

    EventPayload payload = new LikeCreatedPayload(articleId, userId);
    Event event = Event.of(eventId, EventType.LIKE_CREATED, payload);

    outboxRepository.save(Outbox.from(event)); // Outbox 테이블에 쌓기
}</code></pre><p>이 두 개는 같은 트랜잭션 내에서 커밋!</p>
<p>** outbox 테이블을 폴링하면서 kafka로 레코드 전송 **
아직 전송하지 않은 레코드를 kafka로 전송한다.</p>
<pre><code>@Async(&quot;messageRelayPublishEventExecutor&quot;)
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void publishEvent(OutboxEvent outboxEvent) {
    publishEvent(outboxEvent.getOutbox());
}</code></pre><p>** 완료되면 해당 이벤트 삭제하거나 상태 업데이트 **</p>
<pre><code>public void listen(String message, Acknowledgment ack) {

        log.info(&quot;[HotArticleEventConsumer.listen] received message: {}&quot;, message);
        Event&lt;EventPayload&gt; event = Event.fromJson(message);

        if (event != null) {
            hotArticleService.handleEvent(event);
        }
        ack.acknowledge();
    }</code></pre><p>결제/재고 등 강한 일관성이 필요한 서비스가 아니기 때문에 데이터 일관성이 조금 늦춰지더라도 outbox + kafka가 가능하다! </p>