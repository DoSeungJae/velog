<h1 id="0-문제-배경">0. 문제 배경</h1>
<h3 id="1-단위-테스트-코드-작성">1. (단위) 테스트 코드 작성</h3>
<p>프로젝트를 지금까지 진행해오면서 테스트 코드는 작성하지 않았다. 테스트 코드는 내가 작성한 코드가 제대로 작동하는지 검증하기 위한 용도이기에 테스트 코드가 없어도 API를 전송 해주는 플랫폼인 postman을 사용해 코드 동작을 확인하면 됐기 때문에 딱히 필요를 느끼지 못했다.
<br /></p>
<p>하지만 최근 <em>실용주의 프로그래머_를 읽고 생각이 바뀌었다.
postman과 같은 플랫폼을 통해 코드를 검증하는 것은 물론 좋은 방법이나 특정 API를 호출했을 때 실행되는 메서드들의 코드</em>(혹은 그 메서드가 의존하는 코드)_가 변경되었을 때 그 기능이 여전히 이전처럼 잘 실행될 것이라고 보장할 수 없다는 것이 테스트 코드가 필요하다고 느낀 주요한 이유 중에 하나다. </p>
<p>테스트 코드가 없다면 이런 경우에 다시 API를 요청해야하는데, 이를 위해 필요한 여러 프로그램(DB, 웹 애플리케이션 등)을 전부 실행해야해서 많은 시간이 소요되지만, 테스트 코드를 작성했다면 코드가 바뀌더라도 이전에 작성해둔 테스트 코드를 변경된 코드에 적용하기만 하면 문제가 생기는지 그렇지 않은지 알 수 있으며 이는 위 과정보다 훨씬 빠르다.</p>
<p>통합 테스트, 단위 테스트 둘 다 필요하지만 일단 단위 테스트 코드부터 작성하는 것을 시작했다<em>(위 단락들에서 설명된 <strong>&quot;테스트&quot;</strong>는 <strong>&quot;단위 테스트&quot;</strong>를 의미한다.)</em>.</p>
<h3 id="2-민감한-데이터를-환경설정-파일에-저장">2. 민감한 데이터를 환경설정 파일에 저장</h3>
<p>프로젝트를 진행하면서 작성한 코드는 거의 다 _git_에 올려 저장한다. 협업, 백업의 이유도 분명히 있지만 커밋 로그를 훑어보면서 어떻게 진행되고 있는지 과정을 다시 살펴보기에도 좋고 자기만족이기도 하다.
<br /></p>
<p><img alt="" src="https://velog.velcdn.com/images/dsj5508/post/1463a9f5-a050-4abd-91c5-c4a5a28d306f/image.png" /></p>
<p>하지만 예외인 경우도 있다. 예를 들어 백엔드 서버가 DB에 연결할 때 사용되는 정보, 파일을 올리는 AWS 서버에 연결할 때 사용되는 액세스 키 혹은 비밀번호같이 다른 응용 프로그램에 접근할 때 사용되는 인증 정보들은 외부에 공개돼선 안된다. 이것들은 대신 다른 파일에 따로 저장되며 해당 파일은 <em>git_에 올리지 않도록 한다. 이를 환경설정 파일이라고 하는데, _Spring</em> 프레임워크에서는 _application.properties_와 _application.yml_이 그것이다. </p>
<p><img alt="" src="https://velog.velcdn.com/images/dsj5508/post/ecfbd166-3ffb-41bc-a927-14b3cad3e2c6/image.png" />
(application.yml, application.properties는 글자 색이 옅다, _git_에서 제외된 것이다.)</p>
<p>이렇게 저장된 정보들은 <em>@Value</em> 어노테이션을 통해 변수로서 불러들일 수 있으며 이런 방식을 통해 민감한 데이터를 숨기면서도 프로그램이 작동하게 만들 수 있다. 아래 예시 코드를 참고하자.</p>
<pre><code>package com.DormitoryBack.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import com.amazonaws.auth.AWSStaticCredentialsProvider;
import com.amazonaws.auth.BasicAWSCredentials;
import com.amazonaws.services.s3.AmazonS3;
import com.amazonaws.services.s3.AmazonS3ClientBuilder;

@Configuration
public class AWSConfig {

    @Value(&quot;${iamAccessKey}&quot;)
    private String iamAccessKey;

    @Value(&quot;${iamSecretKey}&quot;)
    private String iamSecretKey;

    @Value(&quot;${cloud.aws.region.static}&quot;)
    private String region;

    @Bean
    public AmazonS3 amazonS3(){
        BasicAWSCredentials awsCredentials=new BasicAWSCredentials(iamAccessKey, iamSecretKey);

        return AmazonS3ClientBuilder.standard()
            .withRegion(region)
            .withCredentials(new AWSStaticCredentialsProvider(awsCredentials))
            .build();
    }

}</code></pre><h1 id="1-문제">1. 문제</h1>
<h3 id="1-단위-테스트-환경에서는">1. (단위) 테스트 환경에서는?</h3>
<p>그럼 다음과 같은 테스트 환경에서도 역시 <em>@Value</em> 어노테이션을 사용할 수 있을까?</p>
<pre><code>@Slf4j
@ExtendWith(MockitoExtension.class)
public class FileServiceTest {

    @InjectMocks
    private FileService fileService;

    @Mock
    private AmazonS3 s3Client;

    @Mock
    private MultipartFile file;

    @Value(&quot;cloud.aws.s3.bucket&quot;)
    private String bucketName;

    @BeforeEach
    public void setUp(){
        MockitoAnnotations.openMocks(this);
        fileService.setBucketName(bucketName);

    }

    @Test
    public void testTest(){
        assertEquals(&quot;delivery-box&quot;, bucketName);
    }
 }
</code></pre><p>결과는 다음과 같다.</p>
<p><img alt="" src="https://velog.velcdn.com/images/dsj5508/post/f77eb50b-213a-4769-8ca6-c869714854e1/image.png" /></p>
<p><img alt="" src="https://velog.velcdn.com/images/dsj5508/post/488ad375-b9c8-488e-a31e-51cd6544302f/image.png" /></p>
<p>설정 파일에 있는 속성을 테스트 환경에서는 <em>@Value</em> 어노테이션이 데이터로 가져올 수 없다.</p>
<h3 id="2-단위-테스트에서는-spring-컨텍스트가-없다">2. 단위 테스트에서는 Spring 컨텍스트가 없다.</h3>
<p>위 테스트 코드는 <em>단위 테스트</em> 코드다. <em>단위 테스트_란 개발 단계에서 개별 클래스나 메서드의 동작을 검증하는 테스트인데, _@Value</em> 어노테이션으로 환경설정 파일에서 데이터를 가져올 수 없는 이유는 <em>단위 테스트_가 실행될 때에는 _Spring 컨텍스트_를 불러오지 않기 때문이다. _Spring 컨텍스트_는 _Spring</em> 프레임워크가 실행될 때 설정되는 애플리케이션 구동에 필요한 기본 정보들이며 <em>application.properties_와 같은 설정 파일도 이에 포함된다. 그래서 _@Value</em> 어노테이션을 사용해 봤자 환경설정 파일에 있는 값을 불러들일 수 없는 것이다.</p>
<h3 id="3-springboottest를-사용하면">3. @SpringBootTest를 사용하면?</h3>
<p>물론 테스트 클래스에 @SpringBootTest 어노테이션을 지정하여 실행하면 환경설정의 값을 불러들일 수는 있으나, 이 방식은 <em>통합 테스트_이다. _Spring</em> 프로그램 자체를 실행한다는 의미다. 그리하여 <em>Spring 컨텍스트_를 로드할 수는 있으나 이런 맥락에서는 코드 동작의 독립성을 보장할 수 없어 다른 코드의 영향을 받으므로 문제가 생겼을 때 정확한 진단이 어려우며 간단한 코드라도 테스트하는데에 단위 테스트에 비해 오랜 시간이 걸려 결국 전체 개발 프로세스가 지체되는 결과를 초래할 수 있다</em>(통합 테스트 자체가 의미 없다는 말이 아니다, <strong>단위 테스트</strong> 코드를 <strong>통합 테스트</strong>처럼 실행하면 안된다는 것이다)_.</p>
<h3 id="4-그래서-문제는">4. 그래서 문제는</h3>
<p>그래서 문제는 원래 작성하고자 하는 <em>단위 테스트_에서는 _@Value</em> 어노테이션을 통해 환경설정 파일에 접근할 수 없다. 임시로 테스트할 때는 해당 클래스에 민감한 데이터를 선언해도 상관이 없지만 이런 수준 낮은 코드는 _git_에 올릴 수가 없기 때문에 어떻게든 이 수줍음 많은 데이터들을 안으로 숨겨 넣어야한다.</p>
<h1 id="2-결론-및-요구사항">2. 결론 및 요구사항</h1>
<p>개발 과정 중 이미 작성한 코드를 수정할 때에 테스트 코드의 중요성을 느껴 테스트 코드를 작성하다가 특히 단위 테스트에서 환경설정 코드를 불러올 수 없는 문제를 맞닥뜨렸다. 
단위 테스트 코드 역시 _git_에 저장해야한다는 점을 고려해봤을 때 테스트 환경에서 밖으로 노출되어서는 안되는 데이터들을 따로 관리하고 이를 단위 테스트 클래스가 가져올 수 있어야 하도록 구현해야하며, 과정에서 _@SpringBootTest_와 같은 통합 테스트 방식을 사용해서는 안된다.</p>