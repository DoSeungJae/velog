<h1 id="0-문제-배경">0. 문제 배경</h1>
<h3 id="1-사용자-차단-기능">1. 사용자 차단 기능</h3>
<p>게시글<em>(article)_이나 댓글</em>(comment)_을 차단하여 작성자가 올린 모든 게시물을 차단하는 기능을 구현했다. 불쾌감을 일으키는 글을 내가 <strong>차단하면, 해당 게시글을 포함해 그 사람이 올린 다른 게시글 이나 댓글들이 모두 내 화면에서 보이지 않게</strong>된다.</p>
<p><img alt="" src="https://velog.velcdn.com/images/dsj5508/post/3fbe03c9-bdac-4095-af16-b75ae7ef631c/image.png" />
<img alt="" src="https://velog.velcdn.com/images/dsj5508/post/d4b43440-6d0c-4f84-b8fd-9e21de12b26f/image.png" /></p>
<p>차단 후 모습이다.
<img alt="" src="https://velog.velcdn.com/images/dsj5508/post/6a91986b-8c58-4bb6-ae19-7e59521d20b1/image.png" /></p>
<p><img alt="" src="https://velog.velcdn.com/images/dsj5508/post/04f3a9e0-d4d1-424b-ac5e-f762993af962/image.png" />
게시글 뿐만 아니라 댓글도 차단되었다. </p>
<p>의도된 대로 작동하는 것을 볼 수 있다.</p>
<h3 id="2-계정-탈퇴">2. 계정 탈퇴</h3>
<p>프로젝트 deliveryBox는 사용자가 <strong>계정을 탈퇴</strong>하는 기능도 제공한다.
<img alt="" src="https://velog.velcdn.com/images/dsj5508/post/fac60575-facb-4339-86ed-4b0645d5ebce/image.png" /></p>
<p><img alt="" src="https://velog.velcdn.com/images/dsj5508/post/1f052edb-b3d0-4758-8788-f03a866c3a7a/image.png" /></p>
<p>id가 66인 사용자가 삭제되었다.</p>
<h1 id="1-문제">1. 문제</h1>
<h3 id="1-탈퇴한-사용자의-게시글은-차단되지-않음">1. 탈퇴한 사용자의 게시글은 차단되지 않음.</h3>
<p>그러나 만약 내가 <strong>탈퇴한 사용자의 게시글을 차단</strong>한다면 어떻게 될까?</p>
<p><img alt="" src="https://velog.velcdn.com/images/dsj5508/post/475a58f3-4263-4aa3-b682-5578a1310888/image.png" /></p>
<p><strong>&quot;예&quot;</strong>를 눌러도 화면에서는 눈에띄는 에러 메시지 같은 것은 없다.</p>
<p><img alt="" src="https://velog.velcdn.com/images/dsj5508/post/163a0f8b-1655-4750-8103-9334dd71597e/image.png" /></p>
<p>그러나 <strong>차단한 글은 그대로 남아있다</strong>, 차단되지 않는다, 확실한 문제다.</p>
<h3 id="2-댓글을-작성한-사용자가-탈퇴했을-때-댓글이-표시되지-않음">2. 댓글을 작성한 사용자가 탈퇴했을 때 댓글이 표시되지 않음</h3>
<p>게시글의 경우에는 글을 작성한 사용자가 탈퇴하더라도 게시글은 여전히 조회되며 닉네임이 표시되는 곳에 <strong>&quot;알 수 없음&quot;</strong>이라고 되어있다. 하지만 이 포스팅을 작성하기 직전에 댓글의 경우 그렇지 않다는 것을 뒤늦게 알아차렸다. 탈퇴한 사용자의 댓글은 조회되지 않는 것이다.</p>
<p><img alt="" src="https://velog.velcdn.com/images/dsj5508/post/94727635-cfb9-427e-b71d-ce88266255d6/image.png" />
사실 이 글은 댓글이 있다.
<img alt="" src="https://velog.velcdn.com/images/dsj5508/post/6705be3a-040b-4758-966d-8c25fb4b4173/image.png" />
id가 115인데,
 <img alt="" src="https://velog.velcdn.com/images/dsj5508/post/0fc65911-bf64-412d-b472-8efc254e10aa/image.png" />
id가 3052인 데이터_(댓글)_은 user_id가 NULL이다. 댓글을 쓴 사람이 계정 탈퇴를 한 것이다. 그런 댓글이 하나라도 있으면 나머지 댓글까지 모두 조회되지 않는 상황이다.</p>
<p>원래는 아래와 같이 되어야 한다.
<img alt="" src="https://velog.velcdn.com/images/dsj5508/post/df7ac83d-dbc3-4716-8a81-382792fbccc4/image.png" /></p>
<p>deliveryBox를 개발해오면서 게시된 글이나 댓글은 작성자가 탈퇴하더라도 계속 유지되도록 결정했다. 그 글이 도움이 되고 가치 있는 글이라면, 사용자가 회원 탈퇴를 하더라도 다른 사용자에게 여전히 가치있고 도움이 되는 글일 것이기 때문이다. 그러므로 deliveryBox의 <strong>삭제된 사용자의 댓글은 여전히 조회돼야</strong> 한다. 그러나 현재 그렇지 않다는 점은 문제로 볼 수 있다. 정책상의 문제로도 볼 수 있는 것이다.</p>
<h1 id="2-요약-및-요구사항">2. 요약 및 요구사항</h1>
<p>삭제된 사용자의 게시글을 차단할 때, 삭제된 사용자의 댓글을 조회할 때 기능이 제대로 작동하지 않는 문제가 있다. </p>
<p>첫 번째 문제에 대해, 탈퇴한 사용자의 게시글 및 댓글을 차단하면 작성자의 모든 게시글, 댓글이 나타나지 않게 해야할 뿐만 아니라 기존의 <strong>사용자를 차단한 후에 차단된 사용자가 회원 탈퇴를 해도</strong> 동일하게 기능이 작동해야한다. 두 번째의 경우 작성자가 탈퇴한 댓글은 사용자 정보가 <strong>&quot;알 수 없음&quot;</strong>이라고 표시되며 내용, 작성된 시간 등의 <strong>나머지 내용은 유지</strong>되어야한다. </p>