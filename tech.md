# SSE로 프로젝트 고도화 하기 - 타이머 기능 동기화

안녕하세요, 우아한테크코스 6기 레모네 입니다.
위 주제를 선택하게 된 이유는, 제가 진행하고 있는 프로젝트인 “코딩해듀오" 에서 SSE를 사용해서 기능을 개선한 경험을 공유하고 싶었기 때문입니다.

저희 프로젝트는 “페어 프로그래밍에 필요한 기능(예: 타이머, 메모장)을 하나의 서비스에서 사용할 수 있다면 얼마나 편리할까?” 라는 생각에서 출발했습니다. 혹시라도 페어 프로그래밍이라는 용어를 처음 들어본 분들이 계실 수 있으니 페어 프로그래밍란 무엇인지 짧게 소개해 드리겠습니다. 
페어 프로그래밍은 “네비게이터”와 “드라이버” 두 역할로 나뉩니다.
문제 해결 방법을 구두로 설명하는 네비게이터와, 네비게이터의 설명에 따라 코드를 작성하는 드라이버는 특정 시간(예: 10분) 동안 자신의 역할을 수행한 뒤 반대 역할로 교체합니다. 
내가 10분동안 드라이버 역할을 수행했다면, 10분 뒤에는 네비게이터가 되는 것이죠.

서론이 길었습니다. 이제 본격적으로 프로젝트에 SSE를 도입하게 된 배경을 알아볼까요?

## 동기화가 필요해
저희 서비스에서 “방 만들기" 버튼을 누르면 페어 프로그래밍을 할 수 있는 공간을 생성할 수 있습니다.
![image](https://github.com/user-attachments/assets/85cf2dc2-6084-4190-b462-c59dcb97da07)

이 공간을 “페어룸" 이라고 부르겠습니다.
만약 해당 페어룸에 한 명의 사용자만 접속하고 있다면 문제가 발생하지 않습니다.
그러나 페어룸에 여러 사용자가 접속하게 된다면 큰 문제가 발생합니다.
바로 타이머가 동기화 되지 않는다는 점이죠.
![image](https://github.com/user-attachments/assets/ef9ba964-7c8a-497e-a535-f2f5fd74e20f)

> 🍋 레모네: 페어룸 만들었어! 이제 타이머를 실행하고 페어 프로그래밍을 시작하자.   
> 🍎 사과: 타이머를 내 노트북으로도 확인하고 싶으니까, 레모네가 만든 페어룸에 접속해야겠다. 어, 그런데 타이머 화면이 레모네와 다르네?

이 문제를 해결하기 위해 어떤 기술을 사용해야 할까요?
 
## 웹소켓 vs SSE
동기화를 하려면 실시간으로 데이터를 전송해 줄 방법이 필요한데요,
가장 많이 쓰이는 기술로 웹소켓과 SSE 가 있습니다.
두 기술을 비교하며 저희 서비스에 어떤 방식이 더 적절할지 찾아봅시다.

### SSE (Server-Sent-Event)
SSE는 http 프로토콜 기반 실시간 통신 방법입니다.
SSE의 특징은 단방향 통신이라는 점인데요, 서버에서 클라이언트로 서버가 일방적으로 데이터를 전송합니다. 
http 프로토콜 특성상 요청-응답 주기가 반복되어야 할텐데, 어떻게 요청이 없는데도 서버가 지속적으로 클라이언트에게 데이터를 전송할 수 있을까요?
정답은 서버가 보내는 데이터가 하나의 응답이 아닌, chunk 단위의 스트림이기 때문입니다.


클라이언트가 SSE 통신을 시작할 것이라는 연결 요청을 보내면, SSE 커넥션이 생성됩니다.
그 때부터 해당 커넥션으로 서버가 데이터를 전송할 수 있습니다.

위에서 데이터가 chunk 단위로 전송된다는 표현을 사용했는데요, 간단하게 말하자면 “데이터를 작은 단위로 쪼개서 계속 보낼거야. 아직 데이터를 다 보내지 않았으니 응답은 끝나지 않았어!” 라는 의미입니다.
결국 하나의 요청에 하나의 응답이라는 특성은 유지한 채, 데이터를 지속적으로 클라이언트에게 전송할 수 있는 것이죠.
커넥션이 끊어질 때까지 클라이언트는 서버가 보내는 데이터를 실시간으로 받을 수 있습니다.

이러한 특징 때문에 SSE는 알림 시스템을 구축할 때 주로 사용됩니다.
  
### 웹소켓
웹소켓은 웹소켓 프로토콜 기반 실시간 통신입니다. http 프로토콜이 아닌, 웹소켓 프로토콜을 사용합니다.
SSE와 구분되는 가장 큰 특징은 양방향 통신을 한다는 점입니다.
양방향 통신이 가능하다는 것은, 클라이언트와 서버가 동시에 데이터를 주고받을 수 있다는 뜻입니다.
http 프로토콜에서는 클라이언트의 요청이 올 때까지 기다린 뒤 응답을 보내거나, 서버에서 클라이언트 방향으로만 지속적으로 데이터를 보낼 수 있었습니다.

그러나 웹소켓은 서버가 클라이언트이 요청 없이 응답할 수도 있고, 서버가 응답하지 않아도 클라이언트가 여러번 요청할 수도 있습니다.
사실상 클라이언트와 서버, 요청과 응답이 구별되지 않는 동등한 상태가 되는 것이죠.

그래서 웹소켓은 채팅 애플리케이션에 주로 쓰입니다.

## 그래서 어떤 기술을 사용할까?
지금까지 SSE와 웹소켓에 대해 알아보았습니다.
이제 어떤 방식을 선택할지 결정할 시간이 되었네요.

타이머를 동기화 할 때 어떤 정보가 필요한지 생각해 봅시다.
타이머 시작 버튼을 눌렀을 때, 타이머 시간이 1초씩 줄어든다. 
타이머 중단 버튼을 눌렀을 때, 타이머 시간이 흐르지 않는다.

타이머 시작 요청을 보냈을 때, 1초씩 줄어드는 시간 정보만 필요하다는 것을 알 수 있습니다.
타이머 시간을 서버가 관리한다는 가정 하에, 서버가 클라이언트에게 타이머가 종료되기까지 남은 시간을 1초마다 전송해 주면 되겠네요.

그럼 여기서 떠오르는 기술이 하나 있을 텐데요, 바로 SSE 입니다.
웹소켓이 제공하는 양방향 통신이라는 이점이 타이머 동기화에 필수적이지 않기 때문에 SSE를 사용하는게 더 적절해 보입니다. 

이제 본격적으로 구현을 시작해 봅시다.

## 1초마다 사용자에게 타이머 시간 전송하기
“코딩해듀오”는 spring boot를 사용하고 있기 때문에 spring boot를 기반으로 SSE를 구현해 보겠습니다.

### 1 - 커넥션 생성
먼저 페어룸에 접속한 클라이언트와 SSE 커넥션을 맺어야 합니다.

``` java
@RestController
public class SseController implements SseDocs {


   private final SseService sseService;


   @GetMapping("/{key}/connect")
   public ResponseEntity<SseEmitter> createConnection(@PathVariable("key") final String key) {
       final SseEmitter sseEmitter = sseService.connect(key);


       return ResponseEntity.ok(sseEmitter);
   }
```
컨트롤러에 SSE 커넥션 생성 요청 api를 추가해 줍시다.
참고로 엔트포인트에 있는 `{key}`는 페어룸을 식별할 수 있는 코드입니다.

`sseService.connect()` 가 어떻게 구현되어 있는지 자세히 살펴봅시다.

``` java
@Service
public class SseService {


   private final EventStreamsRegistry eventStreamsRegistry;


   public SseEmitter connect(final String key) {
       final SseEmitter emitter = eventStreamsRegistry.register(key);
       return emitter;
   }
```
eventStreamRegistry에서 key(페어룸 식별자)에 해당하는 SseEmitter를 생성하고, 해당 SseEmitter를 반환하고 있습니다.
SseEmitter는 SSE 를 구현할 수 있도록 스프링이 제공하는 클래스로 이 객체를 통해 데이터를 전송할 수 있습니다. SseEmitter를 간단하게 말하면, 클라이언트에게 데이터를 전송하기 위한 커넥션이라고 할 수 있습니다.
구체적으로 어떤 일을 하는지는 뒤에서 자세하게 설명하겠습니다.

EventStreamRegistry를 살펴봅시다.

``` java
@Component
public class EventStreamsRegistry {


   private final Map<String, EventStreams> registry;


   public EventStreamsRegistry() {
       this.registry = new ConcurrentHashMap<>();
   }


   public SseEmitter register(final String key) {
       final EventStreams eventStreams = registry.getOrDefault(key, new EventStreams());
       final EventStream eventStream = new SseEventStream();
       eventStreams.add(eventStream);
       registry.put(key, eventStreams);
       return eventStreams.publish(eventStream);
   }
 ```
register 메서드는 페어룸 식별자를 key로 갖고있는 EventStreams에 새로운 EventStream을 생성하여 추가합니다. 
EventStream은 SseEmitter를 생성하고, SseEmitter를 통해 데이터를 전송하는 역할을 맡고 있는 인터페이스입니다.
EventStreams는 List<EventStream>를 멤버 변수로 갖고있는 일급 컬렉션입니다.
하나의 페어룸에 여러명의 사용자가 존재할 수 있기 때문에 EventStream이 아닌 List<EventStream>을 값으로 갖고 있습니다.

EventStreams에 대해 알아보겠습니다.

``` java
public class EventStreams {


   private final List<EventStream> streams = new CopyOnWriteArrayList<>();


   public SseEmitter publish(final EventStream eventStream) {
       final SseEmitter sseEmitter = eventStream.connect();
       sseEmitter.onTimeout(sseEmitter::complete);
       sseEmitter.onCompletion(() -> streams.remove(eventStream));
       sseEmitter.onError(error -> streams.remove(eventStream));
       return sseEmitter;
   }
```
EventStream은 SseEmitter를 생성하고, 해당 SseEmitter가 타임아웃 되었을 때, 종료되었을 때, 에러가 발생했을 때, 어떤 행동을 할지 콜백 함수를 등록합니다. 코드에서는 위의 세가지 상황이 발생했을 때 streams에서 해당 EventStream을 삭제합니다.

EventStream의 구현체인 SseEventStream에 대해 알아보겠습니다.
``` java
public class SseEventStream implements EventStream {


   private static final Duration TIME_OUT = Duration.ofMinutes(20);
   private static final String CONNECT_NAME = "connect";
   private static final String SUCCESS_MESSAGE = "OK";


   private final AtomicLong id = new AtomicLong(0);
   private final SseEmitter sseEmitter;


   public SseEventStream() {
       this.sseEmitter = new SseEmitter(TIME_OUT.toMillis());
   }


   @Override
   public SseEmitter connect() {
       final String eventId = String.valueOf(id.incrementAndGet());
       try {
           sseEmitter.send(SseEmitter.event()
                   .id(eventId)
                   .name(CONNECT_NAME)
                   .data(SUCCESS_MESSAGE));
       } catch (final IOException e) {
           sseEmitter.complete();
           throw new SseConnectionFailureException("SSE 연결이 실패했습니다.");
       }
       return sseEmitter;
   }
```
기본적으로 SseEmitter의 타임아웃 시간은 20분입니다. 20분 뒤에 SseEmitter는 종료되고, SSE 연결이 끊어지게 됩니다.
connect 메서드를 살펴보면, 커넥션 연결이 성공했을 시 name은 “connect”로, data는 “OK”가 전송됩니다.
개발자 도구를 열어 네트워크 창의 EventStream을 확인하면 아래와 같은 화면을 확인할 수 있습니다.

![image](https://github.com/user-attachments/assets/7ec3aade-bdb2-40be-910d-fb9422b68fe3)
 
SSE 커넥션을 맺는 것에 성공했습니다.
이제 타이머를 시작해 봅시다.
### 2 - 타이머 시작
``` java
@RestController
public class TimerController implements TimerDocs {


   private final SchedulerService schedulerService;


   @PatchMapping("/{accessCode}/timer/start")
   public ResponseEntity<Void> createTimerStart(@PathVariable("accessCode") final String accessCode) {
       schedulerService.start(accessCode);
       return ResponseEntity.noContent()
               .build();
   }
```
컨트롤러에 타이머 시작 요청 api를 추가합니다.
accessCode는 페어룸 식별자를 뜻합니다. SSE 커넥션을 맺을 때 사용되었던 key와 같습니다.

SchedulerService를 살펴봅시다.
``` java
@Component
public class SchedulerService {


   public static final Duration DELAY_SECOND = Duration.of(1, ChronoUnit.SECONDS);


   private final ThreadPoolTaskScheduler taskScheduler;
   private final SchedulerRegistry schedulerRegistry;
   private final TimestampRegistry timestampRegistry;
   private final TimerRepository timerRepository;
   private final SseService sseService;


   public void start(final String key) {
       if (isInitial(key)) {
           final Timer timer = timerRepository.fetchTimerByAccessCode(key)
                   .toDomain();
           scheduling(key, timer);
           timestampRegistry.register(key, timer);
           return;
       }
       final Timer timer = timestampRegistry.get(key);
       scheduling(key, timer);
   }


   private boolean isInitial(final String key) {
       return !schedulerRegistry.has(key) && !timestampRegistry.has(key);
   }


   private void scheduling(final String key, final Timer timer) {
       final Trigger trigger = new PeriodicTrigger(DELAY_SECOND);
       final ScheduledFuture<?> schedule = taskScheduler.schedule(() -> runTimer(key, timer), trigger);
       schedulerRegistry.register(key, schedule);
   }


   private void runTimer(final String key, final Timer timer) {
       if (timer.isTimeUp()) {
           stop(key);
           final Timer initalTimer = new Timer(timer.getAccessCode(), timer.getDuration(), timer.getDuration());
           timestampRegistry.register(key, initalTimer);
           return;
       }
       if (sseService.hasNoConnections(key)) {
           stop(key);
           return;
       }
       timer.decreaseRemainingTime(DELAY_SECOND.toMillis());
       sseService.broadcast(key, "remaining-time", String.valueOf(timer.getRemainingTime()));
   }
```
start 메서드에서는 먼저 타이머가 실행되었다가 중단된 상태인지, 아니면 한번도 실행되지 않은 상태인지를 구분합니다. 
구분하는 이유는, 타이머 총 시간은 db에서 관리하지만, 타이머가 1초씩 줄어든 상태, 즉 타이머 남은 시간은 인메모리로 관리하기 때문입니다. 타이머 남은 시간이 변경될 때마다 db에 업데이트 할 수 없기 때문에 타이머 남은 시간은 TimeStampRegistry에서 관리하고 있습니다.

타이머가 한번도 실행되지 않은 상태라면 db에서 타이머 시간을 가져옵니다.(타이머 초기화 상태) 타이머가 실행되었던 적이 있다면, TimeStampRegistry에서 타이머 현재 시간을 가져옵니다.
이 시간을 이용해 스케줄링을 시작합니다.

1초에 한번씩 타이머 남은 시간을 전송해야 하기 때문에 스케줄러 실행 주기는 1초로 설정했습니다.
ThreadPoolTaskScheduler를 활용해 1초에 한번씩 runTimer가 실행됩니다.

runTimer에서는 타이머 남은 시간을 1초 줄이고, 페어룸에 있는 모든 클라이언트에게 남은 시간을 전송합니다.
타이머 남은 시간이 0초가 된다면(타이머가 한바퀴 돌았을 때), 타이머를 중단하고 타이머 남은 시간을 타이머 총 시간으로 초기화 합니다.
페어룸에 사용자가 아무도 없을 경우에도 타이머를 중단합니다.
타이머를 중단하는 stop 메서드는 타이머 종료 목차에서 설명드리겠습니다.

이제 타이머를 시작하면,
![image](https://github.com/user-attachments/assets/cc13643f-594d-4360-bcc3-154ec1e65d36)


위 사진과 같은 결과를 확인할 수 있습니다.
사진에 있는 타이머 시간은 밀리초 기준으로, 타이머 시간이 1000밀리초(1초)씩 줄어들고 있는 것을 볼 수 있습니다.
3 - 타이머 종료

이제 실행중인 타이머를 중지해 봅시다.
이전 단계와 마찬가지로 타이머 중지 api를 추가합니다.
``` java
@PatchMapping("/{accessCode}/timer/stop")
public ResponseEntity<Void> createTimerStop(@PathVariable("accessCode") final String accessCode) {
   schedulerService.pause(accessCode);


   return ResponseEntity.noContent()
           .build();
}
```
schedulerService.pause 메서드를 살펴봅시다.
``` java
public void pause(final String key) {
   sseService.broadcast(key, "timer", "pause");
   schedulerRegistry.release(key);
}
```
페어룸에 있는 모든 사용자에게 타이머가 중지되었다는 데이터를 전송합니다.
그 후, SchedulerRegistry에서 타이머를 중단하고자 하는 페어룸의 스케줄러를 종료합니다.
``` java
@Component
public class SchedulerRegistry {


   private final Map<String, ScheduledFuture<?>> registry;


   public SchedulerRegistry() {
       this.registry = new ConcurrentHashMap<>();
   }


   public void register(final String key, final ScheduledFuture<?> future) {
       registry.put(key, future);
   }


   public void release(final String key) {
       if (!registry.containsKey(key)) {
           throw new NotFoundScheduledFutureException("키에 해당하는 스케줄러 결과가 존재하지 않습니다.");
       }
       registry.get(key)
               .cancel(false);
       registry.remove(key);
   }
```
SchedulerRegistry는 페어룸 별 스케줄링 결과를 관리합니다.
release 메서드를 살펴보면, registry에서 페어룸을 key로 페어룸 결과를 가져옵니다.
ScheduledFuture는 스케줄러의 현재 상태를 나타내는 객체입니다.
ScheduledFuture를 cancel하여 실행중이던 스케줄러를 종료하고, registry에서 해당 ScheduledFuture를 삭제합니다.

![image](https://github.com/user-attachments/assets/6debba45-a0a2-405f-93d2-8c93a653fcd3)


성공적으로 타이머가 중단되면, 1초마다 전송되던 타이머 남은 시간이 더이상 전송되지 않고, timer pause 메세지가 전송됩니다.
