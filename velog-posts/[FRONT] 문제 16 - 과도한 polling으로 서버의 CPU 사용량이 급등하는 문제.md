<h1 id="0-배경">0. 배경</h1>
<p>AWS에 배포한 웹 서비스는 크게 프론트엔드인 React.js, 백엔드 서버인 Spring Boot, 그 사이에 위치한 Nginx로 구성된다. 리버스 프록시 서버인 Nginx를 통해 React.js에서의 요청은 Nginx을 통해 Spring Boot로 전송된다. 아래는 간단한 서버 구성도이다.
<img alt="" src="https://velog.velcdn.com/images/dsj5508/post/8e94b32d-6024-479a-b7fe-67e88295ab2d/image.jpg" /></p>
<p>Spring과 Nginx는 각각 AWS의 EC2 인스턴스 위에서 실행되는데 두 EC2 인스턴스는 모두 AWS 무료버전<em>(free tier)_으로 사용할 수 있는 모델로, t2.micro</em>(1 Core)_ CPU와 1G의 RAM으로 초저사양이다. 그래서 하나의 인스턴스가 하나의 프로그램만 실행하도록 구성했다.</p>
<h1 id="1-문제">1. 문제</h1>
<p>React.js에서 socketIO 연결 이후에 채팅 전송 기능을 테스트하던 도중 AWS 모니터링 창에서 이상한 점이 보였다.</p>
<p>Spring을 실행하는 EC2 인스턴스에선 
<img alt="" src="https://velog.velcdn.com/images/dsj5508/post/e599cff8-0767-48e2-8bdd-96f3683e89e0/image.png" />
CPU 사용량이 순간적으로 21.78%까지,</p>
<p>Nginx를 실행하는 EC2 인스턴스는
<img alt="" src="https://velog.velcdn.com/images/dsj5508/post/f96d434f-f37a-4aac-9253-05aeacf15fa2/image.png" />
12.65%까지 올라갔다.</p>
<p>수치만으로 따졌을 땐 큰 문제가 아니라고 생각할 수 있겠지만 당시 테스트 환경은 클라이언트_(브라우저)_가 단 하나인 상황이였다. 아무리 초저사양 서버라도 1개의 클라이언트 때문에 CPU 사용량이 20%를 넘긴다는건 정상이라고 볼 수 없었다.</p>
<h1 id="2-해결-과정">2. 해결 과정</h1>
<h2 id="21-원인">2.1. 원인</h2>
<p>React.js에서는 채팅방에서 실시간으로 메시지를 주고받을 수 있는 기능이 있다. 이때 대화 내역을 조회하는 방식에서 문제가 있었는데, 아래가 그 문제의 코드들이다.</p>
<pre><code> // update messageListAdd commentMore actions

  useEffect(() =&gt; {
    getSocketResponse(room) //적절하지 못한 함수 이름
    .then((res) =&gt; {

      const previousMessageCount = messageList.length;
      const { scrollTop, scrollHeight, clientHeight } = messagesRef.current;
      const previouslyScrolled = scrollTop + clientHeight &gt;= scrollHeight * (1-1/previousMessageCount);
    }).catch((err) =&gt; {
        console.error(err);
    });
  });</code></pre><pre><code>export const getSocketResponse = async (room) =&gt; {
  try {
    const res = await api.get('api/v1/chat/' + room);
    return res.data;
  } catch (error) {
    console.log(error);
  }
}
//단순한 http api 요청 함수, 잘못된 함수 이름.</code></pre><p>채팅 내역을 조회하는 getSocketResponse 비동기 함수가 useEffect() 안에서 호출되는데
useEffect는 아래와 같이 마지막 소괄호가 닫히기 전에 의존성 배열을 넣을 수 있다.</p>
<pre><code>useEffect(()=&gt;{}, [someValue])</code></pre><p>그런데 의존성 배열이 명시되지 않았을 경우에는 컴포넌트가 리렌더링 될 때마다 해당 useEffect가 실행된다. 간혹 이런 경우 useEffect가 다른 useEffect 함수 혹은 _state_와 충돌해 무한 실행 루프에 빠지게 될 수 있는데, 지금 상황이 딱 그렇다.</p>
<p><img alt="" src="https://velog.velcdn.com/images/dsj5508/post/9427ebe5-b555-414f-b177-dd691651c72c/image.png" /></p>
<p>네트워크 패킷이 계속 들어오는데 저것들은 HTTP(S)요청에 대한 응답으로 React.js가 백엔드 서버에 계속 요청을 하고 있다는 의미다. 이미지의 윗부분에 시간 정보가 보이는데, 20000ms 즉 20초 조금 넘는 시간 동안 동일한 요청을 200번 넘게 보내고 있었던 것이다<em>(정상적인 경우에 페이지 초기 로딩 시 처음 보내는 요청은 40~60개다.)</em>.</p>
<p>초당 10회 주기의 이런 과도한 polling 때문에 백엔드 서버와 리버스 프록시 서버에 무리가 갔던 것이라고 확신했다.</p>
<h2 id="22-해결">2.2. 해결</h2>
<p>HTTP(S) API로 채팅 내역을 불러오는 함수는 초기 로딩 시 한번만 실행되면 되고 그 이후로는 Socket의 이벤트 리스닝 방식으로 들어오는 데이터를 그때그때 받아 화면에 추가해주기만 하면 된다.</p>
<pre><code>  useEffect(()=&gt;{
    getSocketResponse(room).then((res)=&gt;{ //잘못된 함수 이름
      setRawMessageList(res);
    })
  },[]); //새로고침 시 한번만 실행</code></pre><p>빈 의존성 배열을 추가해 초기 로딩 때만 대화 내역을 불러오도록 바꿨다.</p>
<pre><code>  useEffect(()=&gt;{
    if(room==='') return ;
    const createdTime=getCurrentLocalDateTimeString(); 
    socketResponse.createdTime = createdTime;
    setRawMessageList((prev)=&gt;[...prev, socketResponse]);
  },[socketResponse]);</code></pre><p>여기에 올리지 않은 다른 코드에 의해 서버가 Socket 데이터를 보냈을 때 socketResponse에 그 값이 저장되게 된다. 그럼 의존성 배열에 socketResponse를 넣으면 Socket 데이터를 받을 때마다 위 useEffect가 실행된다.</p>
<pre><code>  useEffect(()=&gt;{
    if(rawMessageList.length===0) return ;
    const previousMessageCount = messageList.length;
    const { scrollTop, scrollHeight, clientHeight } = messagesRef.current;
    const previouslyScrolled = scrollTop + clientHeight &gt;= scrollHeight * (1-1/previousMessageCount);
    const newMessageList = handleMessages(rawMessageList);
    setMessageList(newMessageList);
    const existNewMessage = newMessageList.length &gt; previousMessageCount;
    if (previouslyScrolled &amp;&amp; existNewMessage) { setShouldScroll(true); };
  },[rawMessageList]);</code></pre><p>위에서부터 첫번째, 두번째 코드들을 통해 실시간으로 받은 새로운 메시지를 화면에 출력할 수 있도록 설정하는 코드도 작성했다. </p>
<p>바뀐 코드를 실행시켜 보면
<img alt="" src="https://velog.velcdn.com/images/dsj5508/post/a93112fe-fe53-492c-8244-72ff3b6e3bb2/image.png" /></p>
<p>이제 React.js가 요청을 한번만 보낸다.</p>
<h1 id="3-결과">3. 결과</h1>
<p>코드 수정 이후 1개의 클라이언트로 소켓 연결을 하고 채팅 메시지를 여러번 보냈고
<img alt="" src="https://velog.velcdn.com/images/dsj5508/post/734891e0-e07a-4da5-8bc9-fc0e4869aedf/image.png" />
그에 따른 Spring EC2, Nginx EC2 인스턴스의 CPU 사용량이다.</p>
<p><img alt="" src="https://velog.velcdn.com/images/dsj5508/post/57d267d5-894d-46a1-9001-5ddcbedf0237/image.png" />
<img alt="" src="https://velog.velcdn.com/images/dsj5508/post/7eac6a04-e672-4447-89ff-9a1b09903143/image.png" />
Spring은 5.%대 초반, Nginx는 4.%대 초반으로 더이상 사용량 급증이 발생하지 않고 안정적으로 성능이 유지되는 것을 볼 수 있다.</p>
<h1 id="4-마무리">4. 마무리</h1>
<p>브라우저의 개발자 도구에 네트워크 메뉴에서 비정상적으로 많은 패킷이 전송되는 것을 확인했고 React.js가 짧은 시간에 너무 많은 HTTPS 요청을 서버로 보내는 것이 원인이였다. 이에 소켓의 이벤트 리스닝 방식을 사용해 최대 21.78%였던 Spring 서버의 사용량을 최대 5.17%까지, 최대 12.65%였던 Nginx 서버의 경우 최대 4.02%까지 감소시켜 성능을 안정화시킬 수 있었다.</p>