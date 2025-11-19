<h2 id="offset-페이징의-한계">offset 페이징의 한계</h2>
<p>보통 무한 스크롤을 구현하게 되면 </p>
<pre><code>SELECT * FROM article
WHERE board_id = :boardId
ORDER BY created_at DESC
LIMIT :size OFFSET :offset;</code></pre><p>이런 형식의 쿼리를 작성할 것이다. 
page와 size를 받아서 offset = (page - 1) * size, LIMIT size OFFSET offset 형태로 페이징을 진행한다. </p>
<p>그러나 데이터가 많아질수록 offset의 비용이 증가하기 때문에 페이지가 뒤로 갈수록 느려진다. </p>
<p>그리고 새로운 데이터가 중간에 끼어들 수 있다. 
만약 내가 3페이지를 보고 있었는데 새로운 글이 10개가 작성되면, 3페이지에 있던 글이 4페이지로 밀려난다.
스크롤을 내렸는데 방금 본 글이 또 위에 보일 수도 있는 것이다.</p>
<h2 id="redis-zset">Redis ZSET</h2>
<p>무한 스크롤 뿐만 아니라 이번 강의에서 자주 쓴 redis 자료구조이다.
ZSET은 member와 score를 sorted set으로 관리하는 자료구조이다.</p>
<p>대표적으로
ZADD key score member : 원소 추가
ZRANGE key start stop : 오름차순 범위 조회
ZREVRANGE key start stop : 내림차순 범위 조회
ZREM key member : 원소 제거
등의 명령어가 있다.</p>
<h2 id="cursor-기반-무한-스크롤-패턴">cursor 기반 무한 스크롤 패턴</h2>
<p>마지막으로 본 articleId를 기준으로 그 다음 걸 가져오는 것이 cursor 기반 패턴이다. </p>
<p>cursor가 없는 최초의 요청은 가장 최신 글부터 정해진 사이즈만큼 가져오고,
cursor가 있으면 커서보다 작은 것들 중에서 최신순으로 정해진 사이즈만큼 가져온다.</p>
<p>cursor 기반은 앞서 얘기한 offset의 단점을 커버 가능하다.
새 글이 추가되어도 내가 본 articleId 이후의 글들만 가져오기 때문에 글이 밀리지 않고, 
offset이 아니기 때문에 인덱스를 활용해 성능도 향상된다.</p>
<p>redis ZSET을 활용하면</p>
<pre><code>    // article-read::article::{articleId}
    private static final String KEY_FORMAT = &quot;article-read::article::%s&quot;;</code></pre><p>redis에서 articleId 목록을 가져와 id 리스트를 기반으로 상세 정보를 읽기 모델 (내 경우 ArticleQueryModelRepository)에서 가져오게 된다.</p>
<p>Snowflake 알고리즘으로 100만개의 게시글을 생성하고 MySQL offset과 Redis ZSET 성능을 비교했다. </p>
<p>총 게시글 수: 1000000건, 페이지 크기: 20건, 테스트 페이지 수: 100페이지로 테스트를 진행한 결과,</p>
<ul>
<li>MySQL OFFSET: 90710ms (평균 907ms/page)</li>
<li>Redis ZSET: 106ms (평균 1ms/page)</li>
</ul>
<p>로 ZSET의 성능이 훨씬 좋다는 것을 확인할 수 있다. </p>