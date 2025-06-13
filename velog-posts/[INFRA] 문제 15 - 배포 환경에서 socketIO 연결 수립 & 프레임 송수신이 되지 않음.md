<h1 id="0-배경">0. 배경</h1>
<p>배포 환경에서는 Spring Boot 백엔드 서버를 AWS의 EC2 인스턴스에 배포했다. 백엔드 서버가 HTTPS로 프론트엔드 서버와 통신할 수 있도록 도와주는 리버스 프록시 서버인 Nginx도 마찬가지로 또다른 EC2 인스턴스에 배포했고, React.js로 구현된 프론트엔드 서버는 AWS의 Amplify로 배포했다.
아래는 배포 환경에서 서버 구성도를 간단히 표시한 것이다<em>(DB 등 Spring Boot의 의존성은 생략했다.)</em>.
<img alt="" src="https://velog.velcdn.com/images/dsj5508/post/f91262ba-09ab-4b6f-bde8-a77dd65a08e3/image.jpg" /></p>
<p>React.js는 Nginx에 HTTPS 요청을 보내면, Nginx가 그 요청을 Spring Boot 서버에게 HTTP로 요청한 후 받은 응답을 그대로 React.js에다 전해준다.</p>
<p>배포 환경에서의 React.js와 Spring Boot의 코드는 개발 환경과 동일하지만<em>(배포 환경을 위한 환경 변수, 그와 관련된 일부 코드 제외)</em>, 프론트엔드 서버가 HTTP로 직접 백엔드 서버에 요청하는 개발 환경의 서버 구성과는 다르다. 
<img alt="" src="https://velog.velcdn.com/images/dsj5508/post/a15810e1-e71e-4039-b7a3-024cc97be97f/image.jpg" /></p>
<h1 id="1-socketio-연결-수립이-되지-않는-문제">1. socketIO 연결 수립이 되지 않는 문제</h1>
<p>프론트엔드 서버가 백엔드 서버에 socketIO 연결을 할 수 없는 문제가 있었다. 웹 브라우저의 console에서는 handshake가 실패했다고 한다.</p>
<p>백, 프론트 모두에 credentials를 허용하는 것으로 문제를 해결할 수 있다.</p>
<pre><code>package com.DormitoryBack.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.CorsRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class CORSConfig implements WebMvcConfigurer{

    @Value(&quot;${allowedOrigin1}&quot;)
    private String allowedOrigin1;

    @Value(&quot;${allowedOrigin2}&quot;)
    private String allowedOrigin2;

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        if(allowedOrigin2==null || allowedOrigin2.isEmpty()){
            registry.addMapping(&quot;/**&quot;)
                .allowedOrigins(allowedOrigin1)
                .allowCredentials(true)
                .allowedMethods(&quot;GET&quot;,&quot;POST&quot;, &quot;PUT&quot;, &quot;PATCH&quot;, &quot;DELETE&quot;, &quot;OPTIONS&quot;, &quot;HEAD&quot;);
        }else{
            registry.addMapping(&quot;/**&quot;)
                .allowedOrigins(allowedOrigin1, allowedOrigin2)
                .allowCredentials(true)
                .allowedMethods(&quot;GET&quot;,&quot;POST&quot;, &quot;PUT&quot;, &quot;PATCH&quot;, &quot;DELETE&quot;, &quot;OPTIONS&quot;, &quot;HEAD&quot;);
        }
    }
}</code></pre><pre><code>s=io(socketBaseUrl,{
                query: `username=${username}&amp;room=${room}`,
                withCredentials: true
            })</code></pre><h2 id="11-credentials">1.1. Credentials</h2>
<p>Credentials는 쿠키, Authorization 인증 헤더, TLS client certificates(증명서)를 포함하는 인증 정보를 말한다.
<em>(출처: <a href="https://inpa.tistory.com/entry/AXIOS-%F0%9F%93%9A-CORS-%EC%BF%A0%ED%82%A4-%EC%A0%84%EC%86%A1withCredentials-%EC%98%B5%EC%85%98">https://inpa.tistory.com/entry/AXIOS-📚-CORS-쿠키-전송withCredentials-옵션</a> [Inpa Dev 👨‍💻:티스토리])</em></p>
<p>클라이언트가 서버에 인증을 요청할 때 포함하는 정보라는 것인데, 위 설정을 추가하면 문제가 해결되는 것으로 보아 HTTPS(WSS)를 통한 웹소켓 연결에서는 HTTP(WS)와 다르게 handshake 과정에서 암시적으로 Credentials를 보내는 것으로 보인다<em>(개발 환경, 배포 환경 모두 웹소켓 연결을 할 때 Credentials를 &quot;코드&quot;에 선언하지 않았다. 그런데 배포 환경에서만 위 설정이 필요한 것으로 보아 HTTPS를 사용하는 경우 내부적으로 Credentials를 전송하는 것으로 추측할 수 있는 것이다.)</em>. </p>
<h1 id="2-연결-수립-이후-프레임데이터-송수신이-되지-않는-문제">2. 연결 수립 이후 프레임(데이터) 송수신이 되지 않는 문제</h1>
<p>그렇게 소켓 연결이 잘 되었어도 프레임_(웹소켓 전송 데이터 단위)_을 주고받을 수 없었다.
아래는 react.js에서 소켓 연결을 위해 작성된 코드다.</p>
<pre><code>useEffect(() =&gt; {
        if(username==null || room==0){
            return ;
        }
        const socketBaseUrl = `${process.env.REACT_APP_SOCKET_URL}`;
        let s;
        const env= process.env.REACT_APP_ENV;
        if(env&amp;&amp;env === &quot;dev&quot;){
            s = io(socketBaseUrl, {
                query: `username=${username}&amp;room=${room}`
            });
        }
        else{
            s=io(socketBaseUrl,{
                query: `username=${username}&amp;room=${room}`,
                withCredentials: true
            })
        }

        setSocket(s);
        s.on(&quot;connect&quot;, () =&gt; {
            setConnected(true);
        });
        s.on(&quot;connect_error&quot;, (error) =&gt; {
            console.error(&quot;SOCKET CONNECTION ERROR&quot;, error);
        });
        s.on(&quot;readMessage&quot;, (res) =&gt; {
            console.log(&quot;SOCKET DATA RECEIVED&quot;, res);
            setSocketResponse({
                room: res.room,
                message: res.message,
                username: res.username,
                messageType: res.messageType,
                createdAt: res.createdAt
            });
        });

        return () =&gt; {
            s.disconnect();
        };
    }, [room, username]);</code></pre><p>코드에서는 env에 따라서 연결할 때 코드가 약간 다른데, 
env 변수가 &quot;dev&quot;일 경우 _withCredential:true_가 없고 그렇지 않을 때는 있다.
해당 변수는 코드가 개발 환경에서 실행되는지, 배포 환경에서 실행되는지 구분하기 위한 것으로, react.js가 환경 변수를 읽음으로써 환경을 파악할 수 있다.</p>
<p>env가 선언된 라인 조금 위에 웹소켓 서버의 url을 선언하는 부분이 있는데 이 값도 환경 변수에서 가져온다.
개발 환경에서는 프로젝트 내에 환경 변수를 저장하는 파일에서, 배포 환경에서는 아래의 스크린샷에서 볼 수 있듯이 Amplify의 환경 변수 설정 탭에서 경로를 설정했다.
<img alt="" src="https://velog.velcdn.com/images/dsj5508/post/a2b9a25b-7953-4552-a835-95519b7505c1/image.jpg" /></p>
<p>그런데, 상술했듯이 배포 환경에서는 프론트엔드의 요청은 백엔드 서버로 바로 가지 않고 리버스 프록시 서버를 거쳐 간다고 했다. 그러니 Amplify의 환경 변수에서 표시된 REACT-APP-SOCKET-URL의 값_(wss://~~dns.org)_은 백엔드 서버에 웹소켓 서버의 주소가 아니라 리버스 프록시 서버 내에서 웹소켓 요청을 전달을 요청하는 주소이다.</p>
<p>아래 리버스 프록시 서버인 Nginx 설정 코드를 한번 보자.
<img alt="" src="https://velog.velcdn.com/images/dsj5508/post/af5fd754-77eb-4c02-bfad-8b5b5479eec6/image.jpg" /></p>
<p>리버스 프록시의 도메인인 ---dns.org에서 /socket.io/에 대한 요청 즉 ---dns.org/socket.io/에 대한 요청을 원래 웹소켓 서버 http:/---:8085/socket.io/로 보낸다.</p>
<p>위 Amplify 환경 변수 스크린샷 처럼 wss://---dns.org로 설정해놓으면 결과적으로는 잘 된다.
하지만 이전에는 경로를 wss://---dns.org/socket.io/로 해놓았는데 이 주소로는 프레임 송수신이 되지 않았다. 웹 브라우저의 network 창에서는 네트워크 패킷에 대한 자세한 정보가 제공되는데 거기서 socket의 프레임에 표시된 에러 메시지 _&quot;Invalid namespace&quot;_를 통해 올바른 주소로 설정했다.</p>
<p>직관적으론 더 적절해 보이는 경로 wss://---dns.org/socket.io/가 틀린 이유는 react.js에 socketIO Client가 작동하는 방식에 있다.</p>
<pre><code>s=io(socketBaseUrl,{
    query: `username=${username}&amp;room=${room}`,
    withCredentials: true
})</code></pre><p>socketIO Client를 통해 웹소켓 서버에 연결하는 코드다.
원래는 파라미터로 들어가는 json 안에 _path: --_와 같이 경로를 표시해야하는데 제공이 되지 않았을 경우에 자동으로 /socket.io/로 설정된다. 그렇게 해서 위 코드에서는 --dns.org/socket.io/로 연결이 되는 것인데 만약 그 환경 변수를 --dns.org/socket.io/로 해놓으면 실제 연결 주소는<br />--dns.org/socket.io/socket.io/가 되는 것이어서 프레임 송수신이 되지 않는 것은 당연하다.</p>
<h1 id="2-마무리">2. 마무리</h1>
<p>Nginx, HTTPS가 추가된 배포 환경에서는 웹소켓 연결 방식이 개발 환경과 달랐고 이전에는 잘 모르고 사용했던 socketIO Client 작동 방식을 알게 되어 경로를 맞게 설정해 결과적으로 잘 작동하게 만들었다. 이때까지 배포 환경에서 맞딱뜨린 문제 중에서 시간이 가장 오래걸리고 성가셨던 만큼 해결하고 나서의 성취감도 가장 크다. 개발을 하면서 생긴 일련의 문제들을 해결해오면서 계속해서 드는 생각은, 어떤 문제든 어떻게든 해결하는 방법은 있다는 것이다. 비록 그 해결 방법이 내가 구체적으로 구상한 것과 다를 지라도 방법은 무조건 어떻게든 있다.</p>