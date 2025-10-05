<h1 id="0-배경">0. 배경</h1>
<p>지금까지 포스팅에서 언급해온 웹앱 프로젝트(쉐어로드;Sharoad)의 초기 단계를 배포했다.</p>
<p>아래는 쉐어로드의 회원가입 처리 로직의 일부분이다.</p>
<pre><code>    public UserResponseDTO makeNewUser(UserRequestDTO dto) {

        ...

        User existingUserMail = userRepository.findByEncryptedEmail(encryptedEmail);
        User existingUserNick = userRepository.findByNickName(safeNickname);
        if (existingUserMail != null) {
            throw new IllegalArgumentException(&quot;이미 사용중인 메일입니다.&quot;);
        }
        if(existingUserNick != null){
            throw new IllegalArgumentException(&quot;이미 사용중인 닉네임입니다.&quot;);
        }

        ...

    }</code></pre><p>입력받은 이메일과 닉네임이 DB에 저장돼 있는지를 먼저 확인하여 데이터 중복을 방지하도록 했다.</p>
<h1 id="1-문제">1. 문제</h1>
<p>그럼 위와 같이 중복 검사 로직이 있다면 데이터가 중복으로 저장되는 것을 더이상 걱정하지 않아도 될까? </p>
<p>그렇게 생각했었지만, 실사용자가 회원 가입을 할 때 문제가 생겼다.</p>
<p>동일한 이메일 / 닉네임으로 가입된 회원이 있었던 것이다.</p>
<p>중복 검사를 거쳐 회원가입이 되는 것이 분명하지만, 실제 DB에는 중복된 데이터가 저장되었다.</p>
<h1 id="2-해결-과정">2. 해결 과정</h1>
<h2 id="21-원인">2.1 원인</h2>
<p>쉐어로드에서는 이미 있는 이메일 혹은 닉네임으로 회원가입을 시도할 때 아래와 같은 에러 메시지가 분명히 나타난다.</p>
<p><img alt="" src="https://velog.velcdn.com/images/dsj5508/post/24eb91e5-5629-48ff-8cd3-be76985fb830/image.png" /></p>
<p>그럼에도 중복으로 저장되는 사례가 생겨 고민을 했었다. </p>
<p>문득</p>
<blockquote>
<p>회원가입 버튼을 동시에 여러번 누르지는 않았을까?</p>
</blockquote>
<p>하고 생각했었다. 그래서 개발 환경에서 시험해보기로 했다.</p>
<p>아래와 같이 회원 정보를 입력해서 
<img alt="" src="https://velog.velcdn.com/images/dsj5508/post/61e50a63-fda4-4dff-a1d5-ce6984c7f419/image.png" /></p>
<p>회원가입 버튼을 <strong>여러번</strong> 눌러보았다.</p>
<p><img alt="" src="https://velog.velcdn.com/images/dsj5508/post/209add5a-bfd2-4b20-8a35-f10d34510542/image.png" /></p>
<p>회원가입 성공 메시지가 여러번 뜬 것을 보고 바로 DB를 조회했고</p>
<p><img alt="" src="https://velog.velcdn.com/images/dsj5508/post/49fe7bb7-4a13-4bb6-989d-ce98c3fd0f1d/image.png" />
<em>(테스트 환경이다.)</em></p>
<p>예상대로 중복으로 저장됐다.</p>
<br />
회원가입 버튼을 동시에 여러번 눌러서 중복 저장이 된 것으로 추측되는데,

<p>한단계 더 들어가 여러번 눌렀다고 왜 여러번 처리가 되며 왜 중복 저장이 됐을까? </p>
<h2 id="22-원인-2---백엔드-서버가-동일한-http-요청을-중복-수신한-상황에서-레이스-컨디션-발생">2.2. 원인 2 - 백엔드 서버가 동일한 HTTP 요청을 중복 수신한 상황에서 레이스 컨디션 발생</h2>
<h3 id="221-spring-boot의-http-요청-처리-과정">2.2.1. Spring Boot의 HTTP 요청 처리 과정</h3>
<p>우선 Spring Boot가 클라이언트로부터 받은 HTTP 요청을 처리하는 순서에 대해 알아보자.</p>
<p>Spring Boot는 기본적으로 내장 Tomcat를 사용해 HTTP 요청을 받는다.
Apache의 Tomcat은 잘 알려진 WAS(Web Application Service)로 클라이언트의 요청에 따라 웹의 동적인 데이터를 처리하는 프로그램이다<em>(물론 Tomcat은 Web Server의 기능도 포함한다.)</em>. </p>
<p>Tomcat은 그 내부에 HTTP Connector를 통해 클라이언트로부터 HTTP 요청을 수신하고
등록된 DispatcherServlet에게 해당 요청을 넘긴다.
Servlet은 Java에서 웹 요청과 응답을 처리하기 위한 클래스이며 DispatcherServlet은 Spring Boot가 기본 설정으로 가지는 Spring MVC 전용 Servlet이다.</p>
<p>요청을 전달받은 DispatcherServlet은 HTTP 요청을 분석해 적절한 Controller 메서드에 요청을 위임_(delegate)_한다.</p>
<p>Controller 메서드는 <em>(유효성 검사, 데이터 변환 이후)</em> Service 메서드를 호출한다.</p>
<p>호출된 Service 메서드는 <em>(필요하다면 DB 액세스를 포함한)</em> 비즈니스 로직을 실행해 결과를 Controller 메소드에 반환한다. </p>
<p>그 다음에 Controller 메서드는 결과를 DispatcherServlet에, DispatcherServlet은 HTTP 응답을 생성해 Tomcat에 다시 보내고, Tomcat이 클라이언트에게 최종 응답을 보내게된다.</p>
<p>순서를 정리하면 아래 이미지와 같다.
<img alt="" src="https://velog.velcdn.com/images/dsj5508/post/09f6abaa-992e-4683-84e7-cb01f2abca52/image.png" /></p>
<h3 id="222-tomcat의-thread-pool">2.2.2. Tomcat의 Thread Pool</h3>
<p>그런데 만약 Tomcat이 클라이언트로부터 여러개의 HTTP 요청을 <em>(거의)</em> 동시에 받는다면 어떻게 될까?
들어온 요청 중 하나를 먼저 처리하고 그 후 다음 요청을 <strong>순서대로</strong> 처리할까?</p>
<p>당연히 오늘날 컴퓨터 프로그램은 그런식으로 작동하지 않는다.
컴퓨터 프로그램은, 여러 연산을 여러개의 작업 단위를 두어 병렬적으로 처리하는 것이 순차적으로 처리하는 것 보다 훨씬 빠르다<em>(물론 연산 간 순서 관계가 없을 때 가능하다)</em>. </p>
<p>컴퓨터 과학에서는 이 작업<em>(연산)</em> 단위를 <strong>Thread</strong>라고 하는데
Tomcat에서도 HTTP 요청들이 들어오면 요청 하나를 하나의 Thread에 할당하여 처리한다.</p>
<p>그럼 이 Thread는 어떻게 관리가 될까?
Tomcat의 <strong>Thread Pool</strong>이 관리한다.
기본 설정으로 10개의 Thread를 미리 생성하여 요청을 대기한다. 만약 현재 Thread 개수보다 더 많은 요청이 들어오면 Thread를 새로 생성하고, 최대 Thread 개수보다 더 많은 요청이 들어오면 큐에 대기시킨다.
Thread를 미리 만들어 대기시켜 놓는 이유는 만약 그렇지 않는다면 요청이 들어올 때마다 매번 Thread를 생성하고 작업이 끝나면 제거해야하는데 이 과정이 불필요한 연산_(Overhead)_을 야기해 성능을 떨어뜨리기 때문이다.</p>
<h3 id="223-동일한-요청들을-동시에-처리하는-과정에서-레이스-컨디션-발생">2.2.3. 동일한 요청들을 동시에 처리하는 과정에서 레이스 컨디션 발생</h3>
<p>그럼 실제로 동일한 HTTP 요청이 동시에 들어왔는지 확인해보자.
아래는 Tomcat의 HTTP 수신 내역을 로그 파일로 저장한 것이다.
<img alt="" src="https://velog.velcdn.com/images/dsj5508/post/2d0b6105-5a90-4769-8c1d-bf46e968b810/image.png" /></p>
<p><strong>2.1. 원인</strong>에서 예측한 것 처럼 회원가입 요청이 <strong>동시에 여러번</strong> 수신됐다.
전술했듯이 회원가입 로직에는 아래와 같이 이메일 중복 검사 과정이 포함돼있지만</p>
<pre><code>        User existingUserMail = userRepository.findByEncryptedEmail(encryptedEmail);
        if (existingUserMail != null) {
            throw new IllegalArgumentException(&quot;이미 사용중인 메일입니다.&quot;);
        }
</code></pre><p>왜 DB에서는 회원 정보가 중복으로 저장되었는지 알아야한다.</p>
<p>가상의 단위 시간에 따른 연산 처리 과정과 DB에 저장된 데이터에 관한 그림을 그려보았다. 
아래를 예시로 참고하자.</p>
<p>설명하기에 앞서 회원가입할 이메일은 <a href="mailto:email@email.com">email@email.com</a>이고, 회원가입 처리 로직은 3개의 하위 연산, &quot;이메일 중복검사&quot;, &quot;중복 여부 확인&quot;, &quot;DB에 이메일 저장&quot;으로 구성되며 각각의 연산은 1 단위시간_(t)_이 소요되는 것으로 정했다.</p>
<p><img alt="" src="https://velog.velcdn.com/images/dsj5508/post/e94bca41-ccbc-49c1-a7fb-2f02fa7087ed/image.png" /></p>
<p>위 예시에서 4개의 회원가입 요청이 동시에 들어온 상황이다.</p>
<p>첫번째 요청을 처리하기 위한 로직을 실행했다.
t1에 이메일 중복 검사를 하고 t2에 이메일을 사용할 수 있다는 것을 확인했다. 따라서 t3에 이메일을 저장했다.</p>
<p>두번째 로직 실행이다.
<strong>t2일 때 이메일 중복 검사</strong>를 진행하였고 <strong>t3일 때 이메일이 중복이 아님을 확인했</strong>으며 t4일 때 DB에 이메일을 저장했다.</p>
<p>3번째와 4번째 실행의 경우에는 이메일 중복 검사를 각각  t5, t6에 했고 이때는 이미 이메일이 저장된 상태이므로 중복 판정, 따라서 데이터를 저장하지 않았다.</p>
<p>문제는 두번째 실행에서 발생했다. 첫번째 실행에서 DB에 이메일을 저장한 시점은 t3인데, t3 이전에 실행된 이메일 중복 검사는 모두 <strong>중복 아님</strong>이라는 결과를 내놓는다<em>(사실 로직상 당연하다. 실제로 DB에 데이터가 없으니까)</em>. 그래서 두번째 실행에서는 DB에 이메일이 한번 더 저장되는 것이다.</p>
<p>일반화하자면 처음 회원가입 로직이 실행될 때, 이메일이 DB에 저장되기 전에 중복 검사를 했던 뒤따르는 실행들은 모두 당시 DB에 이메일이 없으니 중복이 아니라고 판단한 것이다. 그래서 그러한 실행들은 이메일을 중복으로 저장하게 된다. 반면, 이메일이 저장된 이후에 실행된 로직들은 중복 검사를 할 때 이미 이메일이 존재하는 것을 확인하므로 정상적으로 예외 처리가 된다.</p>
<h2 id="23-해결">2.3. 해결</h2>
<p>이메일 중복 검사는 비즈니스 로직_(DB 수준이 아니라)_에서만 진행됐기 때문에 동시에 실행된 회원가입 로직이 DB에 데이터를 중복으로 저장한 것이다. 중복 검사를 데이터베이스 단계에서도 실행하면 이 문제를 해결할 수 있을 것이다.
쉐어로드는 내부적으로 MySQL DB를 사용한다. MySQL의 user 테이블에 아래 제약 조건을 설정했다.</p>
<pre><code>ALTER TABLE user ADD CONSTRAINT unique_email UNIQUE (email);</code></pre><p>위 코드를 통해 더이상 데이터가 중복으로 저장되는 것은 막을 수 있겠지만, 사용자 측에서 중복 요청이라는 메시지를 받을 수 있게 해야한다.</p>
<p>그래서 아래와 같이 코드를 추가했다.</p>
<pre><code>#UserService의 회원가입 로직 일부
  try{
      userRepository.save(user);
  }catch(DataIntegrityViolationException e){
      if(e.getMessage().contains(&quot;unique_email&quot;)){
          throw new DuplicatedRequestException
          (new ErrorInfo(ErrorType.DuplicatedRequest, &quot;이미 처리된 요청입니다.&quot;));
      }
  }</code></pre><pre><code>#ExceptionManager(@RestControllerAdvice)
    @ExceptionHandler(DuplicatedRequestException.class)
    public ResponseEntity&lt;ErrorInfo&gt; duplicatedRequestException(DuplicatedRequestException e){
        return ResponseEntity.status(HttpStatus.CONFLICT).body(e.getErrorInfo());
    }</code></pre><pre><code>package com.-.globalException;

import com.-.ErrorInfo;

public class DuplicatedRequestException extends RuntimeException {

    private final ErrorInfo errorInfo;

    public DuplicatedRequestException(ErrorInfo errorInfo){
        super(errorInfo.getErrorMessage());
        this.errorInfo=errorInfo;
    }

    public DuplicatedRequestException(ErrorInfo errorInfo, Throwable cause){
        super(errorInfo.getErrorMessage(), cause); 
        this.errorInfo = errorInfo; 
    }


    public ErrorInfo getErrorInfo(){
        return errorInfo;
    }
}
</code></pre><pre><code>//프론트 엔드에서 회원가입을 요청을 보내는 코드 중 일부
        try{
            const response=await axios.post(path,body,{});
            return response.data;
        }catch(error){
            const errorType=error.response.data.errorType;
            const errorMessage=error.response.data.errorMessage;
            if(errorType===&quot;DuplicatedRequest&quot;){
                toast.error(errorMessage);
            }else{
                toast.error(&quot;회원가입 처리 중 알 수 없는 오류가 발생했습니다.&quot;); 
            }  
        }</code></pre><p>이제 다시 회원가입 버튼을 중복으로 눌러보겠다.
<img alt="" src="https://velog.velcdn.com/images/dsj5508/post/9b13be5c-37c3-4ce9-bb44-bcff2f70e48b/image.png" /></p>
<p><img alt="" src="https://velog.velcdn.com/images/dsj5508/post/863ac7a1-0c6b-4946-83ac-33e3c66055d1/image.png" /></p>
<p>버튼을 여러번 눌렀을 때 사용자는 이제 요청이 중복되었다는 것을 알게 되었다<em>(다만 성공 메시지가 여러번 뜨는 것은 해결하지 못했다. 원인도 파악하지 못했다.)</em>.</p>
<p><img alt="" src="https://velog.velcdn.com/images/dsj5508/post/7e65dd77-fcc0-4ebd-99a0-123401015091/image.png" />
DB에도 계정이 하나만 저장되었으므로 문제가 해결되었다.</p>
<h1 id="3-리뷰">3. 리뷰</h1>
<p>사용자가 버튼을 동시에 클릭해 동시에 여러개의 HTTP 요청이 서버로 전송되었다. Spring Boot에 내장된 Tomcat이 Thread Pool을 통해 동일한 HTTP 요청들을 동시에 받아 처리했기 때문에 DB에서 레이스 컨디션이 발생하여 회원정보가 중복으로 저장된 문제였다.</p>
<p>MySQL의 사용자 테이블에 UNIQUE CONSTRAINT를 설정해 DB에 데이터가 중복으로 저장되는 치명적인 문제는 해결했고, 사용자에게 명확한 오류 메시지를 전달했기 때문에 개선점은 분명히 있다고 생각한다.
그럼에도 불구하고 중복 클릭 시 일부 응답 메시지가 올바르지 않는 등의 문제는 여전히 남아있다.<br /><br />
소프트웨어에서 발생하는 문제는 프로그래밍 스킬만으로 해결할 수 없다.
개발에 사용한 언어는 물론이고 라이브러리, 프레임워크, 미들웨어의 작동 방식과 그것들 간의 상호작용 과정을 알고 있어야 원인을 명확하게 파악할 수 있다.</p>
<p>문제의 원인을 명확하게 알아야 하는 이유는, 내 생각으론, </p>
<blockquote>
<p>원인에 대한 깊은 이해는 더 나은 해결 방법을 떠올릴 수 있게 해주기 때문이다.
문제가 일어난 이유를 근본적으로 이해하면 할 수록, 그 문제를 해결하는 방법도 더 정교해진다.</p>
</blockquote>
<p>예를들어 단순히 <strong>&quot;회원가입 버튼을 중복 클릭해서 백엔드 서버에서 동시에 여러번 처리가 됐다.&quot;</strong>라고만 이해하면, 프론트엔드에서 버튼을 한 번 누르면 클릭을 비활성화하는 기능_(이 방식은 단지 UI 수준의 해결책이다, 클라이언트는 100% 신뢰할 수 없다.)_이나 지금처럼 UNIQUE CONSTRAINT를 적용시키는 방법 이상으로는 생각하지 못할 것이다.</p>
<p>하지만 프론트엔드에서 생성된 HTTP 요청을 서버가 처리하는 과정을 알게 되면 한가지 더 나아보이는 방법을 떠올릴 수 있다.
Spring Boot는 프론트엔드의 요청을 내장 Tomcat의 Connector -&gt; (Thread Pool -&gt;) DispatcherServlet -&gt; Controller 메서드... 등의 순서로 처리한다. 
그래서 중복 요청이 DispatcherServlet으로 넘어왔을 때 혹은 넘어오기 전에 HTTP 요청이 중복으로 수신됐다는 것을 감지하고 미리 조치를 취하는 기능을 떠올리게 되었다.
이 방법은 불필요한 연산을 방지하고 현재 방식보다 UX를 좀 더 개선할 수 있을 것이다.</p>
<p>이후에 이 기능을 구현해볼 예정이고 결과가 좋다면 오픈소스 라이브러리로 공개해보겠다.</p>