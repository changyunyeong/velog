<p>게시글 읽기 중 무한 스크롤로 구현한 부분에 대해서 부하 테스트를 진행해봤다.</p>
<h2 id="k6">K6</h2>
<p>Grafana k6는 오픈소스 부하 테스팅 툴이다. 
JavaScript로 쉽게 스크립트를 작성할 수 있다.</p>
<pre><code>K6_WEB_DASHBOARD=true K6_WEB_DASHBOARD_EXPORT=html-report.html k6 run {script-name}.js</code></pre><p>로 하면 웹 대시보드의 결과가 html 파일로 저장된다.
export 옵션을 제외하면 대시보드가 웹까지만 나온다.</p>
<p>대시보드는 시각적으로 보기 편한 것 같고, 조금 더 자세한 수치는 로그로 남는 것 같다.</p>
<p>아직은 k6를 처음 써 본 것이라 가장 간단한 부하 테스트 정도만 진행해봤다.
어느정도 익숙해지면 <a href="https://velog.io/@hyungki26/K6%EB%9E%80-%EC%95%84%ED%82%A4%ED%85%8D%EC%B3%90-%EB%B0%8F-%EC%A3%BC%EC%9A%94%EA%B8%B0%EB%8A%A5">해당 블로그</a>를 참고해서 더 많은 실험을 진행해 봐도 될 것 같다.</p>
<p>** http_reqs **
테스트 전체 동안 k6가 보낸 http 요청의 총 개수이다. 
** http_reqs/s **
초당 처리된 요청 수, 즉 RPS (Requests Per Seconds)이다. </p>
<pre><code>http_reqs......................: 220627 3677.288028/s</code></pre><p>내 실험에서는 해당 수치가 나왔으므로 테스트 기간 동안 총 220,627번 요청이 갔고, 초당 3677건의 요청이 서버로 들어갔다고 볼 수 있다.</p>
<p>** VU (동시 사용자 수) **
VU 수 만큼의 사용자가 동시에 요청을 보낸다고 생각하면 된다.</p>
<p><img alt="" src="https://velog.velcdn.com/images/retrina0678/post/64ac1cc5-345a-4d47-8537-24a3181363ac/image.png" /></p>
<p>vu와 http_reqs를 한 번에 확인할 수 있다.</p>
<p>** http_req_duration **
요청을 보낸 시점부터 응답 바디를 모두 받을 때까지 걸린 시간이다.</p>
<ul>
<li>avg
말 그대로 요청 시간의 평균이다. 다만 모든 평균값이 그렇듯이 소수의 outlier 때문에 값이 왜곡될 수 있다. </li>
<li>med
중앙값이다.</li>
<li>p90
전체 요청 중 90%가 해당 시간 이하에서 끝난다는 얘기다. 
결과로 보면 전체 요청 중 2ms 이내에서 90%의 요청이 끝나는 것이다.</li>
<li>p95
전체의 95%가 이 시간 안에 처리된다.
SLO(서비스 수준 목표)를 정할 때 가장 많이 쓰인다. </li>
<li>p99
전체의 99%가 이 시간 안에 처리된다.
꼬리 지연(tail latency, 거의 최악에 가까운 사용자 경험)을 볼 때 해당값을 참고하면 된다. </li>
</ul>
<p><img alt="" src="https://velog.velcdn.com/images/retrina0678/post/5dff1a43-4dd9-4075-b1f0-ef01bb80d7a0/image.png" /></p>
<h2 id="실험-환경-및-결과-요약">실험 환경 및 결과 요약</h2>
<p>우선은 로컬에서 테스트를 돌렸다. 
맥북 에어 m1칩 사용 중이며, Docker로 Kafka, MySQL, Redis를 돌리고 있다.</p>
<p>실험 시나리오는 읽기 전용 테스트로 간단하게 해보았다.</p>
<table>
<thead>
<tr>
<th>실험</th>
<th>설정 요약</th>
<th>http_reqs</th>
<th>http_reqs/s (RPS)</th>
<th>avg(ms)</th>
<th>p90(ms)</th>
<th>p95(ms)</th>
<th>에러율(http_req_failed)</th>
</tr>
</thead>
<tbody><tr>
<td>무한 스크롤 읽기</td>
<td>최대 20 VU, 단계적으로 부하</td>
<td>220627</td>
<td>3677.29</td>
<td>2.45</td>
<td>3.67</td>
<td>4.53</td>
<td>0%</td>
</tr>
<tr>
<td>게시글 쓰기</td>
<td>최대 20 VU, 단계적으로 부하</td>
<td>17924</td>
<td>298.73</td>
<td>31.05</td>
<td>55.38</td>
<td>63.3</td>
<td>0%</td>
</tr>
</tbody></table>
<p>사실 표의 결과랑 웹 대시보드 결과랑 완전히 동일하지는 않다. 
웹 대시보드 보다가 iTerm 들어가보니까 창이 날라가서...^^ 직전에 vscode 터미널로 동일한 실험을 진행했던 기록으로 대체했다.</p>
<p>읽기 같은 경우엔  테스트라 에러율이 0으로 나온 것 같다. 
단위가 ms가 좀 커보일 순 있는데 3.6k RPS라 성능이 그렇게 나쁘진 않다.</p>
<p>쓰기는 당연히 읽기보단 부하가 조금 더 나왔다. 
DB INSERT + Kafka outbox 등 부하가 큰 작업이 포함된 쓰기 API가
단순 SELECT 기반의 읽기 API보다 성능이 떨어지는 건 당연하다. 
다만, 쓰기도 기능이 많이 들어가지 않았기 때문에 로컬에서도 값이 나쁘지 않게 나온 것 같다.</p>