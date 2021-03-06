## Cloudflare Workers 무작정 따라하기

<br />

```
이 글은 서비스를 개발하는데 공식문서를 따라서 구현해보며 공부한 내용들을 정리할 계획입니다.
```

기존 창업팀에서 나와 새로운 창업팀으로 옮기게 되었다.  
이 과정에서 새로이 웹 서비스를 구축해야 했고 나름의 기준들로 스택들을 정하게 되었다,

중요한 점은 이 글의 제목처럼 기존에 사용하던 EC2 모놀리스 방식을 벗어나 서버리스를 선택했고,  
AWS와 Cloudflare 사이에서 고민하다 Cloudflare를 선택하게 되었다.
<br />  
<br />

### ※ Cloudflare를 선택한 이유

<br />

1. 저렴한 비용
   - AWS에서 지원하는 람다, cloudfront, dynamoDB, S3들에 비해
     Cloudflare에서 제공하는 Workers, Pages, KV, R2등이 기본 제공량도 많고 상대적으로 더 저렴하다.
   - 웹 페이지를 CDN으로 띄우는데 무료이다.
2. 더 나은 성능
   - AWS lambda, lambda@edge와 [비교한 글][3]을 봄.
3. GraphQL을 사용해도 CDN을 지원한다.

이외에도 인터넷을 쳐보면 둘을 비교한 글이 많다.  
모든 개발환경이 그렇듯 [자신의 환경][1]에 맞게 설정하면 된다.  
<br />
<br />
<br />  
<br />

## Get Started guide

<br />

서론을 마치고 Cloudflare의 기능들을 무작정 따라해보자  
https://developers.cloudflare.com/workers/

```
※ 오역이 포함되어 있을 수 있습니다. 제가 공부하며 이해한 내용들을 작성했습니다.
```

<br />

Cloudflare Workers는 서버리스 응용프로그램 플랫폼이다.  
전 세계 200개가 넘는 도시에 클라우드 네트워크를 구축했고,  
무료와 유료 버전을 제공한다.

> Cloudflare에서는 [playground][playground]라는 함수 테스팅 환경을 제공한다.  
> 이 환경에서 따라해보면 좀 더 직접적으로 공부가 될 것 같다.

이 가이드는 Cloudflare 계정 설정부터 첫 Workers 스크립트 배포까지 진행한다.
<br />
<br />
<br />  
<br />

### 1. Workers에 회원가입

<br />

배포를 시작하기 전에, [회원가입][sign up]부터 해야한다.  
Workers는 당신 소유의 도메인이나 workers.dev로 끝나는 무료 도메인을 얻을 수 있다.  
회원가입 과정은 \*.workers.dev 서브 도메인과 이메일 인증으로 진행된다.  
이 과정은 서비스 시험 과정이므로 무료 플랜을 선택하였다.
<br />
<br />
<br />
<br />

### 2. Workers CLI 설치

<br />

다음으로 Workers전용 CLI인 `wrangler`를 설치한다.  
생성, 구성, 빌드, 프리뷰와 배포등을 간편하게 만들어준다.  
`wrangler`를 설치하기 위해선 npm이 설치되어 있어야 하며  
가급적이면 nvm이나 Volta를 이용해 노드버전 변경이나 권한 제어를 편하게 하면 좋다.
<br />

npm

```sh
npm install -g @cloudflare/wrangler
```

yarn

```sh
yarn global add @cloudflare/wrangler
```

그 다음 설치가 잘 되었는지 `wrangler --version` 명령어로 확인해본다

```sh
wrangler --version
```

<img src="https://eumericano.s3.ap-northeast-2.amazonaws.com/dev/wrangler-version.png" alt="버전확인" style="width:50vw; min-width:400px;"/>

<br />
<br />
<br />
<br />

### 3. Workers CLI 설정

<br />

설치가 끝나면 `wrangler`를 통해 Workers의 리소스를 사용할 수 있도록 OAuth를 설정해 주어야 한다.

아래의 커맨드를 입력하면 자동으로 그 과정이 실행된다

```sh
wrangler login
```

<br />

※ 로그인 완료 후 화면

<img src="https://eumericano.s3.ap-northeast-2.amazonaws.com/dev/wrangler+login.png" alt="계정 로그인" style="width:50vw; min-width:500px;"/>

<br />
<br />   
<br />   
<br />

### 4. 새 프로젝트 만들기

<br />

Wrangler의 `generate` 명령어는 새로운 프로젝트를 만들어 준다.  
기본 설정으로 `default starter` 템플릿이 적용되어 생성된다.  
만약 커스텀 템플릿을 사용하고 싶다면, 생성한 템플릿의 URL을 추가로 입력하면 된다.

예시로, `first-worker`라는 이름의 기본 템플릿 Worker를 생성하려면 아래의 명령어를 입력하면 된다.

```sh
wrangler generate first-worker
```

Wrangler는 `first-worker`라는 디렉토리를 새로 생성하고, starter 템플릿을 연결해준다. 위 경우에는 기본 템플릿을 제공한다.  
Wrangler는 자동적으로 이름이 `first-worker`로 명명된 `wrangler.toml`파일을 생성한다.

※ wrangler.toml

<img src="https://eumericano.s3.ap-northeast-2.amazonaws.com/dev/wrangler.toml.png" alt="wrangler.toml" style="width:50vw; min-width:500px;"/>

[빠른 시작][starter templates] 페이지 에서 다른 스타터 템플릿들을 확인 할 수 있다.

예를들어, 타입 스크립트로 워커를 시작하고 싶다면 아래 명령어를 작성하면 된다.

```sh
wrangler generate first-worker https://github.com/cloudflare/worker-typescript-template
```

또한, 커스텀 템플릿을 만들고 싶다면 [wrangler init][wrangler init] 을 사용하면 된다.

<br />
<br />  
<br />  
<br />

### 5. 코드 작성

당신이 프로젝트를 생성했다면, 이제는 코드를 작성할 수 있다.

<br />   
<br />

#### 5-1. Hello World 이해하기

근본적으로 Worker는 2가지 구성요소로 이루어져 있다.
<br />

1. [FetchEvent][fetch event]들을 확인하기 위한 [Event Listner][event listner]들과
2. `.respondWith()` 매서드에 매개변수로 들어갈 Response객체를 반환하는 함수입니다.

<br />

**first-worker/index.js**

```javascript
addEventListener("fetch", (event) => {
  event.respondWith(handleRequest(event.request));
});

async function handleRequest(request) {
  return new Response("Hello worker!", {
    headers: { "content-type": "text/plain" },
  });
}
```

<br />
아래는 요청에 대한 응답 작업의 흐름을 보여주는 예시이다

1. `FetchEvent`에 대한 이벤트 리스너는 당신의 Worker로 들어온 모든 요청을 스크립트에게 수신하라고 지시한다. 이벤트 핸들러는 `event.request`가 포함된 `event`객체를 넘겨주고, `Request` 객체는 `FetchEvent`를 트리거한 HTTP 요청을 나타낸다.
2. `.respondWith()`를 통해, Worker에서 Response를 반환하는 것을 커스텀된 Response로 런타임 인터셉트해서 반환할 수 있다. (예제의 경우 텍스트 "Hello worker!"를 반환하도록 작성함)
   - 일반적으로 FetchEvent 핸들러는 `.respondWith()`나 `Promise<Response>`로 끝난다.
   - `FetchEvent` 객체는 다른 2가지 메서드인 예외처리 매서드와 Response를 리턴 후 작업을 처리해주는 매서드를 가지고 있다.

자세한 사항은 Fetch Event 디테일 페이지에서 확인하면 된다.
<br />  
<br />

#### 5-2. request 필터링 및 라우팅

<br />

모든 리퀘스트들에 대한 간단한 스크립트를 작성한 후, 해야할 다음 과정은 Worker 스크립트가 받는 request들을 기반으로 동적 response를 만드는 것 입니다.

<br />  
<br />  
<br />  
<br />

---

[references]

1. [https://blog.upstash.com/aws-lambda-vs-cloudflare-workers][1]
2. [https://isotropic.co/cloudflare-workers-vs-aws-lambda/][2]
3. [https://news.ycombinator.com/item?id=17445134₩][3]

[playground]: https://cloudflareworkers.com/
[sign up]: https://dash.cloudflare.com/sign-up/workers
[starter templates]: https://developers.cloudflare.com/workers/get-started/quickstarts/
[wrangler init]: https://developers.cloudflare.com/workers/cli-wrangler/commands/#init
[fetch event]: https://developers.cloudflare.com/workers/runtime-apis/fetch-event/
[event listner]: https://developers.cloudflare.com/workers/runtime-apis/add-event-listener/
[1]: https://blog.upstash.com/aws-lambda-vs-cloudflare-workers
[2]: https://isotropic.co/cloudflare-workers-vs-aws-lambda/
[3]: https://news.ycombinator.com/item?id=17445134
