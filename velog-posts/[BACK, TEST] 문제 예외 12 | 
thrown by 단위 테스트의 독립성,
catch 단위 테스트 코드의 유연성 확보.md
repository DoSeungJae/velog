<h1 id="0-배경-문제-정의-12">0. 배경 <em><a href="https://velog.io/@dsj5508/BACKTEST-%EB%AC%B8%EC%A0%9C-%EC%A0%95%EC%9D%98-12-%EB%8B%A8%EC%9C%84-%ED%85%8C%EC%8A%A4%ED%8A%B8%EC%97%90%EC%84%9C-%EB%AF%BC%EA%B0%90%ED%95%9C-%EB%8D%B0%EC%9D%B4%ED%84%B0%EA%B0%80-%EB%85%B8%EC%B6%9C%EB%90%98%EB%8A%94-%EB%AC%B8%EC%A0%9C#4-%EA%B7%B8%EB%9E%98%EC%84%9C-%EB%AC%B8%EC%A0%9C%EB%8A%94">문제 정의 12</a></em></h1>
<p>git에 저장될 테스트 코드를 작성하는 과정에서 민감한 정보를 포함하는 단위 테스트 코드가 있었다. 통합 테스트 방식을 사용하지 않는다는 전제 하에, 단위 코드에 포함되어서는 안되는 데이터들을 따로 관리하고, 이를 단위 테스트 클래스가 가져올 수 있는 기능을 구현하려했다.</p>
<h1 id="1-예외">1. 예외</h1>
<p>그러나 이번 문제정의서의 요구사항을 만족시킬 수 있는 방법은 없다.</p>
<h3 id="1-시도했던-방법--환경-변수-설정">1. 시도했던 방법 : 환경 변수 설정</h3>
<p>linux 환경 변수를 설정하고, java에서 설정된 값을 가지고오는 방법을 생각했다, 이렇게 하면 의도대로 단위 테스트 환경에서 클래스가 환경 변수를 가져와 문제를 해결할 수 있을 것이라 생각했다.</p>
<pre><code>#!/bin/bash

export MY_ENV_VAR=&quot;Hello, World.&quot;

export TEST=&quot;test var&quot;</code></pre><p>(Runtime 환경에서 제대로 작동하는 환경 변수 관련 코드 및 사진)</p>
<pre><code>        String envVarName=&quot;MY_ENV_VAR&quot;;
        String envVarValue=System.getenv(envVarName);

        String envVarName2=&quot;TEST&quot;;
        String envVarValue2=System.getenv(envVarName2);


        System.out.println(&quot;env1 :&quot;+envVarValue);
        System.out.println(&quot;env2 :&quot;+envVarValue2);</code></pre><p><img alt="" src="https://velog.velcdn.com/images/dsj5508/post/b2cafe49-34e0-44af-b68a-6ba25fcc901c/image.png" /></p>
<p>위와 같이 자바 Runtime에서는 환경 변수를 잘 가져오는 것을 볼 수 있다.</p>
<p>그러나 단위 테스트 환경에서는,</p>
<p><img alt="" src="https://velog.velcdn.com/images/dsj5508/post/7a6c70b4-6d29-4e28-ad2e-cf06f769b6d9/image.png" /></p>
<p>내 예상과 다르게 환경 변수 값을 가져올 수 없었다.</p>
<h3 id="2-실현이-불가능한-이유--단위-테스트의-독립성">2. 실현이 불가능한 이유 : 단위 테스트의 독립성</h3>
<p>단위 테스트는 함수 주로 개별적 모듈(클래스, 메서드)을 테스트하는 방법이다. 그래서 이 테스트에서는 <em>&quot;실제로&quot;</em> 다른 모듈을 실행하거나 그것에 접근하지 않는다. 단위 테스트의 목적 자체가 개별 메서드의 동작을 독립적으로 검증하는 것이므로, 다른 모듈의 의존성을 최소화해야하기 때문이다. 물론 코드 동작 상에 어쩔 수 없이 다른 모듈에 의존하는 경우가 있는데<em>(물론 거의 항상 그렇다.)</em>, 이런 경우에는 _모의 객체(Mock Object)_를 사용하여 외부 의존성을 가상으로 대체한다.</p>
<pre><code>    @Mock
    private MultipartFile file;</code></pre><pre><code>    @Test
    public void testGenerateFileName(){
        String originalFileName=&quot;test.txt&quot;;
        when(file.getOriginalFilename()).thenReturn(originalFileName);

        String fileName=fileService.generateFileName(file);

        assertTrue(fileName.contains(originalFileName));
    }</code></pre><p>위와 같이 <em>when().thenReturn()_과 같은 코드가 Mock으로 등록된 외부 클래스에 대한 동작을 (실제로 해당 모듈을 실행하지 않고) 대체하는 방법 중에 하나다. 외부 모듈이 _&quot;이렇게 호출되었을 때&quot;<strong>(when)</strong></em>, _&quot;이런 값이 나온다는 것&quot;<strong>(thenReturn)</strong>_을 보장한 상태에서, 테스트 대상 모듈이 얼마나 혼자서 잘 작동하는지를 시험하는 것이 이 단위 테스트의 의도인 것이다.</p>
<p>그러니까 다른 모듈을 직접적으로 테스트에 관여시키는게 불가능하다는 말이다. 그런 것은 통합 테스트에서나 가능하다.</p>
<h1 id="2-결론-및-예외-처리">2. 결론 및 예외 처리</h1>
<p>단위 테스트는 개별 요소가 독립적으로 기능을 잘 수행하는지 판단하기 위해 사용하는 테스트 기법이므로 문제 정의서의 요구사항인 단위 테스트 코드에서 실제 환경 변수 값에 접근하는 것은 불가능하며 접근 하려는 발상 자체가 이 테스트의 목적과 의도를 정확히 파악하지 못했다는 의미다.</p>
<p>그래서 다음과 같이 테스트 코드를 수정했다.</p>
<pre><code>    @Test
    public void testUploadFile() throws IOException {
        String originalFileName=&quot;test.txt&quot;;
        byte[] content=&quot;test content&quot;.getBytes();

        when(file.getOriginalFilename()).thenReturn(originalFileName);
        when(file.getInputStream()).thenReturn(new ByteArrayInputStream(content));
        when(file.getContentType()).thenReturn(&quot;text/plain&quot;);
        when(file.getSize()).thenReturn((long) content.length);

        when(s3Client.putObject(anyString(), anyString(), any(InputStream.class), any(ObjectMetadata.class))).thenReturn(null);
        String result=fileService.uploadFile(file);

        assertNotNull(result);
        assertTrue(result.startsWith(&quot;https://s3.amazonaws.com&quot;));
        verify(s3Client, times(1)).putObject(anyString(), anyString(), any(), any(ObjectMetadata.class)); 
    }</code></pre><p><em>when(s3Client~)</em> 구문 내부를 보자.
<em>any()</em>, <em>anyString()_과 같은 _Matcher_를 사용했다. Mockito 라이브러리에서 제공되는 _Matcher_는 테스트에서 모의 객체의 특정 메서드 호출을 가상으로 정의할 때 사용되는 메서드이다. _any()_는 아무 _Object(혹은 아무런 타입)</em>, <em>anyString()_은 아무 _String</em> 타입의 데이터를 넣었다고 가정했을 때를 의미하며 _Matcher_를 사용해 구체적인 값을 넣지 않아도 유연하게 단위 테스트 코드를 작성할 수 있다.</p>
<p>원래는 bucketName에 실제로 AWS S3에서 사용하는 버킷의 이름을 넣어 테스트 코드를 실행하려고 했으나, 코드에 실제 버킷의 이름을 포함하는 것은 보안상으로 적절하지 못하고 외부 환경의 실제 값을 가져오는 것도 단위 테스트에서는 불가능하기 때문에 아무 문자열을 bucketName으로 넣었을 때 잘 작동하도록 하기 위해 위 방법을 사용했다.</p>
<p>_문제 정의 12_는 애초부터 _단위 테스트_에 대한 이해가 부족한 상태에서 쓰여진 것이다. 애초에 해결할 수 없는 문제를 문제로 정의하였는데, 요구사항대로 문제를 해결하려고 하면 단위 테스트의 기본 원칙에 어긋나기 때문이다. 따라서, 올바른 단위 테스팅을 가정했을 때, 이는 해결할 수 없는 문제로 볼 수 있다. 해당 문제가 일어난 원인 자체를 완화하여 문제를 우회함으로써 효과적으로 문제를 관리할 수 있다.</p>