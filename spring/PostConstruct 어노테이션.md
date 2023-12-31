# @PostConstruct

Jwt 토큰과 관련된 속성들이 정의된 ```JwtProperties``` 클래스가 있다. 해당 클래스에는 토큰 생성에 필요한 시크릿키가 포함되어 있는데, 원칙적으로 이 시크릿키는 사용자에게 유출되지 않고 오직 서버에서만 알고있어야 한다.  
```
@RequiredArgsConstructor
public class JwtProperties {

  public static final String SECRET = "Reminiscene";
  public static final int ACCESS_EXPIRATION_TIME = 1000*60*10;
  public static final int REFRESH_EXPIRATION_TIME = 1000*60*30;
  public static final String TOKEN_PREFIX = "Bearer ";
  public static final String HEADER_STRING = "access_token";
}
```
근데 git에 올릴 때 해당 파일이 그대로 올라가면서 보안상 중요한 역할을 하는 시크릿키가 노출되는 문제가 생겼다. 해당 파일을 ignore 시키는 간단한 해결책이 있었지만 다른 방법을 시도해보고 싶었다.  

## @Values
가장 처음 시도한 방법은 **```@Values```** 어노테이션을 사용하는 것이었다.
```application.properties```에 있는 시크릿키 속성을 클래스 필드에 의존성주입 해주는 방법이다.
```
@Value("${spring.jwt.secret}")
  private String secret;
```
하지만 또다른 문제가 생겼다. ```JwtProperties``` 클래스의 모든 속성은 **static**으로 선언되어 다른 클래스에서 필요하면 클래스를 통해 호출해왔는데, ```@Values``` 어노테이션은 오직 인스턴스 변수에 대해서만 사용할 수 있었던 것...  
이거 하나 고치겠다고 ```JwtProperties```를 사용하는 다른 클래스들에 의존성 주입을 해주고 호출방법을 클래스->객체로 바꾸자니 여간 귀찮은 일이 아니었다.


다음 방법을 시도해보자... 열심히 머리굴려서 다음 방법을 생각해냈다.
1. ```@Value```를 통해 ```application.properties```에 있는 속성을 인스턴수변수 ```secret```에 주입한다.
2. ```secret```에 들어있는 값을 클래스변수 ```SECRET```에 삽입한다.

그렇다면 ```JwtProperties``` 클래스의 생성자에서 ```secret```을 ```SECRET```에 넣어주면 되지 않을까?  
```
public JwtProperties() {
  log.info(String.format("%s, %s", secret, SECRET));  // 출력값: null, null
}
```
싶었지만 그건 빈의 생명주기를 몰라서 할 수 있는 소리였다.  
생성자가 호출되는 시점에 ```secret```을 출력해보니 ```null```이었다.
```@Value```는 빈의 생성/의존성주입이 이루어진 후 초기화되는 시점에 동작한다. 하지만 생성자는 빈이 생성되고 의존성주입이 발생하는 시점에 호출된다. ```secret```에 값이 들어가기도 전에 무언가를 넣으려고 뻘짓했던 것이다.

## @PostConstruct
이에 구글링하다가 우연히 **```@PostConstruct```** 어노테이션에 대해 발견했다.
```@PostConstruct```는 다음과 같은 특징을 가진다.
- 빈 생성, 의존성주입 후 초기화 시점에 동작
- 빈 생명주기에서 단 한번만 수행될 것을 보장해 빈이 여러번 초기화되는 것을 방지
- 최신 스프링에서 가장 권장하는 방법
- ```jakarta.annotation.PostConstruct``` 패키지에 포함되어 스프링에 종속되지 않음
- ```@ComponentScan```과 어울림 (이유 모름)  
  
이 어노테이션을 사용하면 의존성주입으로 **```secret```에 값이 들어간 후!!!** 동작할 수 있을 것이다.
그 결과 작성한 코드는 다음과 같다.  
```
public class JwtProperties {

  @Value("${spring.jwt.secret}") private String secret;
  public static String SECRET;

  @PostConstruct
  public void init() {
    SECRET = this.secret;
    log.info(String.format("%s, %s", secret, SECRET));  // 올바른값 출력됨
  }
}
```
의도한대로 잘 동작하는 것을 확인할 수 있었다.
