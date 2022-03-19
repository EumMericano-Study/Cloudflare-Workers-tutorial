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

```
1. 저렴한 비용
   - AWS에서 지원하는 람다, cloudfront, dynamoDB, S3들에 비해
     Cloudflare에서 제공하는 Workers, Pages, KV, R2등이 기본 제공량도 많고 상대적으로 더 저렴하다.
   - 웹 페이지를 CDN으로 띄우는데 무료이다.
2. 더 나은 성능
   - 직접 테스트를 해보지 않았지만 AWS lambda, lambda@edge보다 [벤치마크][3]에서 앞섰다는 글을 보았다.
3. GraphQL을 사용해도 CDN을 지원한다.
```

이외에도 인터넷을 쳐보면 둘을 비교한 글이 많다.  
모든 개발환경이 그렇듯 [자신의 환경][1]에 맞게 설정하면 된다.  
<br />

서론을 마치고 Cloudflare의 기능들을 무작정 따라해보자  
https://developers.cloudflare.com/workers/
<br />
<br />
<br />  
<br />

## Get Started guide

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

---

[references]

1. [https://blog.upstash.com/aws-lambda-vs-cloudflare-workers][1]
2. [https://isotropic.co/cloudflare-workers-vs-aws-lambda/][2]
3. [https://news.ycombinator.com/item?id=17445134₩][3]

[playground]: https://cloudflareworkers.com/
[sign up]: https://dash.cloudflare.com/sign-up/workers
[starter templates]: https://developers.cloudflare.com/workers/get-started/quickstarts/
[wrangler init]: https://developers.cloudflare.com/workers/cli-wrangler/commands/#init
[1]: https://blog.upstash.com/aws-lambda-vs-cloudflare-workers
[2]: https://isotropic.co/cloudflare-workers-vs-aws-lambda/
[3]: https://news.ycombinator.com/item?id=17445134
