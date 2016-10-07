# Vingle FrontEnd는 어떻게 만들어졌는가?
## AWS Lambda를 활용한 마이크로서비스 웹서버 구축

### Writer
*Front-End Developer*  
*Tylor Shin*

### 배경
기존의 Vingle Service는 Ruby on Rails를 기반으로 한 Monolithic System으로 구성되어 있었습니다. Web Server와 API Server 로직을 한 서비스에서 모두 담당하고 있는 형태였습니다. 이러한 아키텍쳐는 관리적 측면에서(특히 배포에 있어서) 클라이언트의 작은 스타일 하나가 바뀌거나, 어드민 기능전환조차도 Data 부분과 묶여서 한 번에 관리되어야 하는 단점 및 비효율을 가지고 있었습니다.

Vingle은 그 어떤 서비스보다 사용자 친화적인 서비스이며 클라이언트의 잦은 개선이 많이 일어나는 서비스이기 때문에, 위와 같은 구조에서는 프론트엔드만의 자유로운 배포 및 관리가 불가능했습니다.

따라서 저희는 이러한 행태를 개선하고자 새로운 아키텍쳐를 도입하기로 준비했습니다.

### 설계시 요구사항 - Requirements -
1. Scalability가 신경쓰지 않아도 확보되어야 한다. (더 이상 확장성에 신경쓰고 싶지 않다.)
2. Static HTML을 Serve 해주어야 한다. (Server-Side Rendering(isomorphic-rendering)이 가능하면 더욱 좋다.)
3. 같은 URL에 접속해도 User의 Desktop \/ Mobile 환경에 따라 다른 Destination에 Serve 해야 한다.
4. 기존 Rails에서 처리하던 비즈니스 로직들을 계속해서 잘 처리할 수 있어야 한다.
5. CDN(Akamai)을 최대한 활용할 수 있어야 한다.
6. 이 모든 것을 만족하면서 1초 안에 최초 렌더링이 완성되어야 한다.

### 배경
최근 엄청난 속도로 부상하고 있는 서버리스 아키텍쳐와 이를 지원하는 프레임워크들([Serverless](https://serverless.com/), [APEX](https://github.com/apex/apex))을 보며 위 요구사항들을 모두 만족시킬 수 있는 구조를 설계하고자 했다.

다만, 굳이 아직까지는 프레임워크까지 사용할 일이 없어서 자체적인 배포 및 관리 환경을 구현하였다.

### 설계
[Image]
1. 현 Rails (후 Route53)이 API Gateway를 Hit한다.
2. API Gateway가 미리 설정해 둔 stage variables(각 플랫폼의 Active Version, HTTP request headers, meta data)와 User가 Hit한 path를 가지고 Lambda를 invoke한다.
3. Lambda는 전달받은 데이터들로 User의 플랫폼을 파악한 후 해당하는 플랫폼의 Active version에 해당하는 index.html 파일을 AWS-SDK를 통해 S3로부터 받아온다. (이 때 만약 Lambda Container가 계속 유지되고 있었다면, S3로부터 다운받지 않고 Memory cached된 index.html의 string을 Server한다.)
4. 받아온 HTML을 text/html로 serve한다.
5. (클라이언트 사이드) 유저는 index.html을 최초 렌더링하고 CDN으로부터 Front-End bundled JS를 받아온다.
6. JS를 기반으로 한 Client-Side rendering을 수행한다.

### 각 단계별 설명
**1. Rails(Route53이나 Nginx로 교체될 예정) -> API Gateway Hit**  
Rails가 User가 접속한 Path, HTTP header, 기타 metadata들과 함께 API Gateway에 HTML 요청 request를 보낸다.

**2. API Gateway -> Lambda**  
API Gateway에는 stage variables라는 기능이 존재한다.
이는 하나의 hash안에 key, value의 형태로 data를 지정할 수 있는 기능인데, 이를 Lambda로 전달하는 것이 가능하다.
Lambda에서는 접속 Header의 User-Agent를 Parsing하여 어떤 플랫폼을 Serve할 것인가를 결정한 뒤,
해당 플랫폼의 S3 내의 index.html 위치를 stage variables로부터 받아서 Serve한다.
따라서 각 플랫폼 별 Repository에서는 배포 절차가 API Gateway의 stage variables를 배포할 S3의 index.html 위치로 바꿔주는 것이 된다.

또한 매번 Request가 있을 때마다 S3에서 index.html을 받아와서 보내줄 필요는 없다. Lambda container에는 일정한 생명주기가 있는데(자세히 기술되어 있지는 않지만), container가 삻아있는 동안에는 해당 컨테이너에서 모든 요청을 처리한다.(하는 듯 하다.)

따라서  
```
export const getFile = (CURRENT_DINGLE_VERSION) => {
  if (!!cachedFile) {
    return Promise.resolve(cachedFile));
  } else {
    return new Promise((resolve, reject) => {
      request(
        `https://s3.example.net/desktop_web/${CURRENT_DINGLE_VERSION}/index.html`,
        (error, res, body) => {
          console.error(error);
          if (error) reject(error);
          else {
            cachedFile = body;
            resolve(body);
          }
        }
      )
    });
  }
};
```
와 같이 작성하면 memory cache를 사용해서 요청을 최소화 할 수 있다.
(또한 Lambda의 과금 정책 중 하나인 '함수 실행 시간' 역시 최소화 할 수 있다.)

**3. Lambda -> S3**  
Lambda Function 안에서 AWS-SDK를 사용해도 좋고, CDN을 사용해도 좋다.

**4. Client -> CDN(S3) bundleJS**  
Rails를 통해서 User가 접속한 주소 자체는 유효하므로(Reverse Proxy 형태로 구현되어 있어서) 아직까지는 따로
Client-Side Router와 original request를 연결해주는 로직은 필요 없다.

index.html에서

```
<script src="https://akamai.example.net/desktop_web/2016-10-07T03-01-45.558Z/bundle.js"></script>
```

와 같이 로드한다.


### 관리
** Deploy **
Jenkins를 CI로 사용한다.  
master branch에 merge되면 자동으로 Jenkins가 Jenkinsfile을 통해 빌드를 시작한다.

빌드는 생각보다 간단하며 다음과 같은 단계들로 구성되어 있다.
  - checkout master
  - load Docker image (node6.5.0 + zip package installed)
  - npm install
  - run npm build script
    - transpile ES6 scripts
    - zip lambda function scripts with required node_modules
    - add git tag to current deploying branch as current date&time(ex: 2016-10-07T03-01-45.558Z)
    - push to S3
    - Notify result to Slack

위의 과정들이 진행되면 압축된 Lambda function 파일들이 각 배포 버젼에 맞게 S3에 올라가게 된다.

마지막 실제 프로덕션 배포 및 적용은 또 다른 Jenkinsfile.deploy.prod 로 관리하며, 하는 일은 다음과 같다.
  - checkout master
  - load Docker image(node 6.5.0)
  - npm install
  - update Lambda function to target version (reassign S3 bucket & key)

이 마지막 배포는 AWS Console에서 Lambda 항목에서도 할 수 있다.
하지만 Jenkins를 통해서만 하는 것이 사후 관리에 더 용이하고 추적도 쉬우므로(Jenkins에 기록되지 않은 배포 및 롤백 기록이 있다는 것 자체가 좋다고 생각하지 않기 때문에) 되도록 Jenkins를 통해서만 하는 것을 원칙으로 한다.

** Rollback **
롤백 역시 간단하며 Jenkins를 활용한 방법과 실제 AWS Console에서 진행하는 방법이 존재한다. 이 경우에도 역시, AWS Console보다는  Jenkins를 최우선으로 한다.  
사실 Jenkins에서 이전 성공했던 빌드를 찾아 버튼 하나만 눌러주면 바로 해당하는 버젼이 target version이 되어 배포되게 된다.
즉, 위 전체 배포과정에서 마지막 실제 프로덕션 배포 과정을 다시하는 것이다. (어차피 S3에 압축된 lambda 파일들은 남아있으므로))

** Log && Monotoring **
AWS Console에서 Cloudwatch를 통해 확인한다.
추후에 AWS-SDK를 이용해서 dashboard 같은 것을 만들면 훨씬 체계화 된 모니터링이 가능할 듯 하다.

### 결론
처음 이 구조를 생각했을 때 가장 걱정했던 것 중 하나는 initial response의 속도 문제였다. 그런데 생각보다 속도가 잘 나와서 크게 걱정하지 않아도 되어서 다행이었다.(초기 index.html로딩까지 200~800ms) 현재 Rails를 통과해서 API Gateway를 찍게 설계되어 있는데, 이 부분을 스킵하고 API Gateway로 넘어간다면 더 빠르게 렌더링 될 수 있다.

Test는 Lambda function이 nodeJS 기반이므로 mocha + mockery + sinon + chai 조합으로 작성했다.
build script 작성에 gulp는 아예 사용하지 않았으며,
각 stage를 하나의 Promise를 return 하는 function으로 해서 계속해서 실행할 수 있는 형태로 작성하였다.

example
```
buildLambdaScript()
  .then(() => zipLambdaFiles())
  .then(() => addTag())
  .then(() => pushToS3())
  .then(() => recordTag(NEW_TAG))
  .then(() => {
    console.log('recording git tag is done');
  });
```

### 후기
