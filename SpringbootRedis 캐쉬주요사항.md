# Springboot Redis 캐쉬 주요사항

1.grdle 필요라이브러리 추가
1) spring-data-redis : 레디스 데이터 서버 설정에 필요한 라이브러리
2) spring-boot-starter-cache: 스프링부트용 레디스 캐쉬 사용에 필요한 라이브러리(스프링부트 버전업에 따라 Deprecated 됨)
3) jedis: redis clients

//redis 예시
compile 'org.springframework.data:spring-data-redis'
compile 'redis.clients:jedis'
compile group: 'org.springframework.boot', name: 'spring-boot-starter-cache', version: '2.0.1.RELEASE'

2. Application.java 파일에 @EnableCaching 적용.
- 관련 Annotation은 포스트 프로세서가 구분 된 메소드를 찾기 위해 모든 빈을 검사하고 모든 호출을 가로 채기 위해 프록시를 생성

EX)
@SpringBootApplication
@EnableCaching
public class Application extends SpringBootServletInitializer {
	public static void main(String[] args) {
		SpringApplication.run(Application.class, args);
	}
}

3. 주의사항 class에 데이터를 담는 자바빈의 경우 모든 객체에 Serializable implements 되어있어야 함.
  - 되어있지 않은 경우 오류 발생.
  - 하기 예시는 lombok 과 같이 사용하는 경우의 예시임
@Data
public class AuthToken implements Serializable{
    private static final long serialVersionUID = 4404236100290581280L;
    private String apikey;
    private String Cookie;
    public AuthToken(String apikey,String Cookie) {
        this.apikey = apikey;
        this.Cookie = Cookie;
    }
}

/**Redis 캐쉬 옵션 검토**/
SpEL(스프링 표현언어) 와 사용가능.

@Cacheable - 캐쉬 자동 관리 어노테이션

Spring은 캐시 된 데이터를 수동으로 관리하기위한 주석 
-메소드 실행 후 캐시가 수행되면  해당 결과의 객체값을 가지고 있다가 다음 호출시 결과가 캐쉬에서 로드된다.
-캐쉬는 메모리에 저장되는 구조이기 때문에 모든 데이터를 캐쉬해서는 안되며 가장 로드가 잦은 서비스 등에 
 제한적으로 사용하는 것이 바람직하다.
-데이터 변경이 많고 실시간 처리및 확인 요구되는 요건상에 적용은 바람직하지 않음. 

EX 1) SpEL 의 표현예제 
@Cacheable(value = "post-single", key = "#id", unless = "#result.shares < 500")
public Post getPostByID(@PathVariable String id) throws PostNotFoundException {

EX 2) 클래스내 변수 isbn 의 캐쉬 저장.
@Cacheable(value="books", key="#isbn")
public Book findBook(ISBN isbn, boolean checkWarehouse, boolean includeUsed)

Ex3) 클래스내 isbn 자바 빈 파일의 rawNumber 캐쉬 저장.
@Cacheable(value="books", key="#isbn.rawNumber")
public Book findBook(ISBN isbn, boolean checkWarehouse, boolean includeUsed)

Ex4) someType 메소드 호출 및 isbn 파라미터 입력 및 출력 값  캐쉬 저장.
@Cacheable(value="books", key="T(someType).hash(#isbn)")
public Book findBook(ISBN isbn, boolean checkWarehouse, boolean includeUsed)

Ex5) name 자바 빈의 length  값 조건 부여에 따른  캐쉬 저장
@Cacheable(value="book", condition="#name.length < 32")
public Book findBook(String name)

Ex6)Cache SpEL에서 사용 가능한 메타데이터

이름	       위치	                        설명	                            예시
methodName	root object	                   호출되는 메서드의 이름	                #root.methodName
method	        root object	                   호출되는 메서드	                  #root.method.name
target	        root object	                   호출되는 대상 객체	                 #root.target
targetClass	root object	                   호출되는 대상 클래스	                #root.targetClass
args	        root object	                   대상을 호출하는데 사용한 인자(배열)	   #root.args[0]                                    
caches  	root object	                   현재 실행된 메서드  캐시의 컬렉션      #root.caches[0].nam                             
argument        evaluation                        메서드 인자의 이름.                   iban나 a0 (p0를  사용하거나
name	        context              이유로든 이름을                         별칭으로 p<#arg> 형식을 사용할 수 있다.).
                                                   사용할 수 없다면
                                                   (예: 디버깅 정보가 없는 경우)
                                                   a<#arg>에서 인자 이름을 사용할 수도 
                                                  있고 #arg은 (0부터 시작하는) 인자의 
                                                  인덱스를 의미

@CachePut - 데이터를 수동 관리 어노테이션

메서드 실행에 영향을 주지 않고 캐시를 갱신할 수 있다. 즉 메서드를 항상 실행하고 그결과를 옵션에 따라 캐시에 보관한다.
메서드 흐름 최적화보다는 꼭 캐시 생성을 해야 하는 경우에 사용한다.

EX1) 사용자를 업데이트 한 경우 캐시에 결과를 저장하여 나중에 조회 할 수 있다.
EX2)아래 메소드는 항상 실행되고 결과는 캐시에 저장된다.
@CachePut(value = 'user', key = "#id")
public User updateUser(Long id, UserDescriptor descriptor)

@CacheEvict - 캐쉬 삭제 어노테이션

@CacheEvict(value = 'user', key = "#id")
public void deleteUser(Long id)
캐시에서 데이터를 제거 (evict)하려는 경우, 즉 캐시를 제거하려는 경우가 있습니다. 
EX)사용자 정보를  삭제하는 경우에 사용 

EX1) allEntries 옵션 - 전체/부분 객체 삭제 시 사용 (true(전체)/false(부분)  
false 설정의 경우 books 객체 전체가 아닌  키의 삭제를 진행. 
@CacheEvict(value = "books", allEntries=true)
public void loadBooks(InputStream batch)

EX2)beforeInvocation 옵션- (true) 메서드 실행 이후 이전에 제거하는 경우(true)
/(false)메서드 실행 이후 메서드 가 성공적으로 완료 되면 캐시에서 제거

@Caching -@CacheEvict나 @CachePut처럼 같은 계열의 어노테이션을 여러 개 지정할때 사용하는 어노테이션

EX1) 하기 클래스 내 primary 캐시객체의 삭제,  secondary 캐시 객체 내 pO키 의 삭제
@Caching(evict = { @CacheEvict("primary"), @CacheEvict(value = "secondary", key = "#p0") })
public Book importBooks(String deposit, Date date)


