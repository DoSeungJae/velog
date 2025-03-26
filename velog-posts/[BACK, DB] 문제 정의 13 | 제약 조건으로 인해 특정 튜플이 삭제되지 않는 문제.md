<h1 id="0-문제-배경">0. 문제 배경</h1>
<p>개발하고 있는 웹앱에서 사용자 계정 탈퇴 기능을 구현했다. 하지만 일부 계정의 경우 기능이 제대로 작동하지 않는 경우가 있었다. </p>
<h1 id="1-문제">1. 문제</h1>
<p>그래서 postman에서 사용자의 계정 탈퇴 http 요청을 보냈다(대상 id는 58).
<img alt="" src="https://velog.velcdn.com/images/dsj5508/post/613cf9a0-d153-43ac-bddb-dc55e40cf82c/image.png" /></p>
<p>서버는 삭제에 성공했다고 <strong>응답</strong>하긴 하지만,</p>
<p><img alt="" src="https://velog.velcdn.com/images/dsj5508/post/73a395d2-e136-41c2-9736-1bd94ab50835/image.png" /></p>
<p>id가 58인 사용자를 조회해 보면 아직 지워지지 않고 남아있는 것을 볼 수 있다.
직접 DB(MySQL)에서 삭제 SQL(DELETE)을 입력한 것 역시
<img alt="" src="https://velog.velcdn.com/images/dsj5508/post/5f7807c3-b32e-4896-8558-f8acbe7f369e/image.png" /></p>
<pre><code>ERROR 1451 (23000): Cannot delete or update a parent row: a foreign key constraint fails (`deliveryBox`.`comment`, CONSTRAINT `FK8kcum44fvpupyw6f5baccx25c` FOREIGN KEY (`user_id`) REFERENCES `user` (`id`))</code></pre><p>라는 에러 메시지가 나타나면서 실패했다.</p>
<h1 id="2-요구사항">2. 요구사항</h1>
<p>모든 사용자 삭제 요청이 (정의된 조건을 만족한다면) 정상적으로 처리돼야 한다.
뿐만 아니라 왜 요청에 실패했음에도 서버가 왜 <strong>성공</strong>했다는 <strong>응답</strong>을 보냈는지도 확인해야 한다.</p>