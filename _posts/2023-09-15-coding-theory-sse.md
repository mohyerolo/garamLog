---
title: SSE 구현
layout: post
categories: coding
tags: theory
---
프로젝트에서 알림 기능을 구현해야 되면서 폴링이 아닌 다른 방식이 있나 알아보다가 웹 알림 구현 방식을 알게됐다.    
대표적인 웹 알림 구현 방식은 **Polling, Long-Polling, 웹소켓, SSE(Server-Sent Event)** 가 있으나 맨 처음에는 알림을 날릴만한 이벤트가 발생했을 때 db에 저장하고, 프론트에서 주기적으로 서버로 api를 호출해 받아오도록 하는 방식(폴링)을 선택했다.    

그렇게 했던 이유는 우선 알림이 발생할만한 애들을 개발하던 도중이었고 알림 테이블 설계에서 내가 고민이 많았으므로 알림을 주고 받는 행위는 나중에 생각해보려고 했었다.    

프로젝트는 커뮤니티 개발이므로 글 작성이 많다보니 알림 기능이 었어야됐지만 폴링을 쓰는게 좋은 환경과 좋지 않을 환경이 있을테고 그것을 먼저 확인하고자 했다.    

## 폴링
폴링이란 리얼타임 웹을 위한 기법으로 일정한 주기를 가지고 서버와 응답을 주고받는 방식이다.    
우리 프로젝트에서는 10초를 기준으로 정보를 가져오도록 프론트에서 이벤트? 설정을 했던 걸로 기억한다.   
그러나 이렇게 했을 때 만약 업데이트 된 정보가 없다면 리소스를 낭비하는것이며, 주기가 짧으면 서버의 성능에 부담이 가고, 주기가 길면 실시간 성능이 떨어진다는 단점이 있다.    

폴링 방식의 장점은 구현이 쉽다는 것밖에 없는 것 같기는한데 정말 간단하다.    
클라이언트는 몇 초마다 돌아가는 이벤트를 등록하면 되는 것이고, 서버에서는 단순히 Get api를 하나 만들면 끝이다.    

폴링은 실시간성이 떨어진다는 단점이 있고, 이런 단점을 보완하고자 나온 방식이 롱 폴링이다.    

## 롱 폴링
폴링의 보완 버전으로 서버의 접속을 열어두는 시간을 늘려 클라이언트의 요청이 서버에서 답이 올때까지 대기하는 방식이다.    
만약 새로운 데이터가 도착하면 응답을 반환하고, 클라이언트는 다시 새로운 요청을 보내면 된다.    

- 폴링에 비해 서버 <-> 클라이언트 통신 빈도가 줄어들어 서버 부하 감소
- 데이터의 업데이트가 많다면 폴링과 큰 차이가 없음

## SSE
프로젝트에서 이미 웹소켓을 채팅 기능에 사용중이었으나 나는 알림에 더 맞는 새로운 방식을 사용해 보고 싶어서 SSE를 적용해보기로 했다.
알림이라는 것은 클라이언트와 서버의 소통이 아닌 서버가 일방적으로 보내주면 클라이언트에서는 받아서 보여주기만 하면 되는 기능으로 양방향 소통이 필요하지 않다. 웹소켓은 양방향이나 SSE는 __단방향 통신__ 으로 알림이라는 기능에 더 맞다고 생각했다.

폴링을 사용하지 않기로 한 제일 큰 이유는 실시간성으로 SSE는 클라이언트가 서버쪽으로 이벤트를 구독하면, 서버에서 해당 이벤트가 발생할때 데이터를 보내줄 수 있고 폴링과 다르게 한 번만 연결 요청을 보내면 연결이 종료되기 전까지 서버와 클라이언트쪽에서 계속 소통이 가능하다는 장점이 있다.    

- 작성한 글에 댓글이 달림
1. 폴링 방식
    1) 클라이언트가 10초마다 요청을 날리는 이벤트 등록
    2) db에 댓글이 달렸다는 알림 저장
    3) 다른 작업을 하다 클라이언트가 10초에 한 번씩 데이터를 불러오는 api 호출
    4) GET API 요청을 통해 서버에서 db에 저장된 사용자의 알림 리스트를 보내줌
    5) 사이트에 받아온 알림을 표시
2. SSE 방식
    1) 클라이언트가 이벤트 구독
    2) db에 댓글이 달렸다는 알림 저장
    3) 서버에서 알림이 저장되면 클라이언트쪽으로 데이터를 보내줌
    4) 날아온 데이터를 받아서 사이트에 표시

spring boot에서는 SseEmitter 클래스를 통해 SSE를 구현할 수 있다.    

SSE를 사용하기 위해서는 우선 클라이언트쪽에서 서버의 이벤트 구독을 위한 요청을 보내야한다.    
이벤트의 미디어 타입은 text/event-stream으로 정해져있고 연결 요청이 들어오면 서버는 클라이언트와 매핑되는 SSE 통신 객체를 만들어서 보내줘야된다.    

```java
@Operation(description = "알람 구독")
@GetMapping(value = "/subscribe", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public SseEmitter subscribe(@AuthenticationPrincipal UserDetails userDetails,
                            @RequestParam(required = false, defaultValue = "") String lastEventId,
                            HttpServletResponse response) {
    response.setHeader("X-Accel-Buffering", "no");
    return sseEmitters.subscribe(userDetails.getUsername(), lastEventId);
}
```
- X-Accel-Buffering: 이거는 클라이언트에서 설정해줘도 되는건데 나는 서버에서 직접 설정해줬다. Nginx로 통신하다보니 Nginx의 버퍼링 기능으로 실시간성이 떨어질 수 있다. X-Accel-Buffering을 여기에서만 넣어주면 이 헤더가 있는 것만 버퍼링을 하지 않도록 설정할 수 있다.    

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class SseEmitters {
    private final Map<String, SseEmitter> emitters = new ConcurrentHashMap<>();
    private final Map<String, Object> eventCache = new ConcurrentHashMap<>();
    private final NotificationRepository notificationRepository;

//    private Long TIMEOUT = 1000L * 60L * 20L;
    private Long TIMEOUT = 1000L * 60L * 3;

    public SseEmitter subscribe(String userid, String lastEventId) {
        log.info("[SSE] - subscribe: " + userid);
        String emitterId = makeTimeIncludeId(userid);
        SseEmitter emitter = save(emitterId, new SseEmitter(TIMEOUT));
        emitter.onCompletion(() -> {
            log.info("[SSE] - onCompletion()");
            deleteAllEmitterStartWithId(userid);
        });
        emitter.onTimeout(() -> {
            log.info("[SSE] - timeout: " + userid);
            this.emitters.remove(emitterId);
        });
        emitter.onError(throwable -> {
            log.error("[SSE] - subscribe error");
            log.error("", throwable);
            emitter.complete();
        });

        // 503 에러 방지하기 위해 더미 이벤트 전송
        String eventId = makeTimeIncludeId(userid);
        sendNotification(emitter, eventId, emitterId, "EventStream Created. [userid=" + userid + "]");

        // 미수신한 Event 목록이 존재할 경우 전송하여 Event 유실을 예방 (Last-Event-ID는 프론트에서 보내주는 것)
        if (hasLostData(lastEventId)) {
            sendLostData(lastEventId, userid, emitterId, emitter);
        }

        return emitter;
    }

    private String makeTimeIncludeId(String userid) {
        return userid + "_" + System.currentTimeMillis();
    }

    private void sendNotification(SseEmitter emitter, String eventId, String emitterId, Object data) {
        try {
            emitter.send(SseEmitter.event()
                    .id(eventId)
                    .data(data, MediaType.APPLICATION_JSON)
                    .reconnectTime(0));
        } catch (IOException exception) {
            emitters.remove(emitterId);
        }
    }

    private boolean hasLostData(String lastEventId) {
        return !lastEventId.isEmpty();
    }

    private void sendLostData(String lastEventId, String userid, String emitterId, SseEmitter emitter) {
        Map<String, Object> eventCaches = findAllEventCacheStartWithByUserid(userid);
        eventCaches.entrySet().stream()
                .filter(entry -> lastEventId.compareTo(entry.getKey()) < 0)
                .forEach(entry -> sendNotification(emitter, entry.getKey(), emitterId, entry.getValue()));
    }

    public void send(Long refId, User receiver, NotificationType type, String message, String url) {
        Notification notification = notificationRepository.save(Notification.builder().refId(refId).receiver(receiver).type(type).message(message).url(url).build());

        String receiverid = receiver.getUserid();
        String eventId = receiverid + "_" + System.currentTimeMillis();

        Map<String, SseEmitter> emitters = findAllEmitterStartWithByUserid(receiverid);
        emitters.forEach(
                (key, emitter) -> {
                    saveEventCache(key, notification);
                    sendNotification(emitter, eventId, key, NotificationDto.toDto(notification));
                }
        );
    }

    public SseEmitter save(String emitterId, SseEmitter sseEmitter) {
        emitters.put(emitterId, sseEmitter);
        return sseEmitter;
    }

    public void saveEventCache(String eventCacheId, Object event) {
        eventCache.put(eventCacheId, event);
    }

    public Map<String, SseEmitter> findAllEmitterStartWithByUserid(String userid) {
        return emitters.entrySet().stream()
                .filter(entry -> entry.getKey().startsWith(userid))
                .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue));
    }

    public Map<String, Object> findAllEventCacheStartWithByUserid(String userid) {
        return eventCache.entrySet().stream()
                .filter(entry -> entry.getKey().startsWith(userid))
                .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue));
    }

    public void deleteAllEmitterStartWithId(String userid) {
        log.info("[SSE] - deleteAllEmitterStartWithId: " + userid);
        emitters.forEach(
                (key, emitter) -> {
                    if (key.startsWith(userid)) {
                        emitters.remove(key);
                    }
                }
        );
    }

    public void deleteAllEventCacheStartWithId(String userid) {
        eventCache.forEach(
                (key, emitter) -> {
                    if (key.startsWith(userid)) {
                        eventCache.remove(key);
                    }
                }
        );
    }
}
```

1. subscribe 요청 들어옴
2. 사용자의 id로 구분이 가능한 emitter id를 만듬
3. 멀티스레드 환경에서 사용할 수 있는 ConcurrentHashMap을 사용해서 타임아웃이 설정된 SseEmitter를 emitterId 값이랑 저장. sseEmitter를 관리하는 스레드들이 콜백할 때 스레드가 다를 수 있기 때문에 ThreadSafe한 구조를 사용하는 것.
4. 해당 SseEmitter에게 이벤트 등록
    1) onCompletion(): 비동기 요청이 완료됐을 때 호출할 코드로 시간 초과 및 네트워크 오류를 포함한 어떤 이유로든 비동기 요청이 완료되면 컨테이너 스레드에서 호출된다.
    2) onTimeout(): 비동기 요청이 시간 초과 됐을 때 발생한다.
    3) ontError(): 비동기 요청 진행 중에 에러가 발생했을 때 실행된다.
5. 등록을 진행한 뒤, 유효 시간동안 어느 데이터도 전송되지 않는다면 503 에러가 발생하므로 더미데이터를 보내준다.    
6. 연결이 끊어지고 다시 재연결이 될 때까지 미수신한 이벤트가 존재할 경우 클라이언트에서 보내준 lastEventId를 기준으로 전송해 이벤트 유실을 예방한다.
7. 등록한 sseEmitter를 클라이언트에게 반환

*만약 스케일 아웃을 생각한다면 ConcurrentHashMap이 아닌 Redis pub/sub 방식 필요*

이 이후에는 알림이 발생할 상황에 SseEmitter의 send를 활용하면 되는데 이보다는 이벤트 리스너를 활용하는게 졸을 것 같아서 나는 ApplicationEventListener를 사용했다.    

```java
private final ApplicationEventPublisher applicationEventPublisher;

...

applicationEventPublisher.publishEvent(NotificationRequestDto.toDto(answer.getId(), question.getId(), question.getUser(),
                    NotificationType.ANSWER, messageSource.getMessage("notification.answer", new Object[]{question.getTitle(), user.getNickname()}, null)));
```

이런 식으로 이벤트 리스너를 등록하고 publishEvent로 이벤트를 발생시키면 Listener 클래스를 통해 send를 발생시키도록 했다.

```java
@Component
@RequiredArgsConstructor
public class NotificationListener {
    private final SseEmitters sseEmitters;

//    @TransactionalEventListener
//    @Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT) //커밋완료 후 작업
    @Transactional(propagation = Propagation.REQUIRES_NEW)// 새로운 트랜잭션으로 구성
    public void handleNotification(NotificationRequestDto dto) {
        sseEmitters.send(dto.getRefId(), dto.getReceiver(), dto.getType(), dto.getMessage(), dto.getUrl());
    }
}
```

원래는 @Async를 이용해 비동기적으로 처리하려고 했으나 SseEmitter에서는 이걸 이용하면 어째서인지 이벤트가 발생되지 않았다. 그래서 어쩔 수 없이 트랜잭션을 끊고 새로 만드는 방식을 활용할 수 밖에 없었다.    

- @TransactionalEventListener: 트랜잭션이 사용되는 상황에서 이벤트 발생이 중간에 껴있을 때 해당 작업 이후의 작업이 롤백되거나 상황이 발생됐을 때 이벤트의 처리를 설정하기 위해 사용하는 어노테이션

<br>
<hr>
이렇게 했을 때 로컬에서는 문제 없이 작업이 돌아가고 알림을 주고받는 것을 확인할 수 있었다.    
그러나 문제는 Nginx를 끼고 통신을 하면 SSE 연결을 1분마다 닫고 재연결이 발생한다는 점이었다.    
이것을 위해    

```java
proxy_set_header Connection '';
proxy_http_version 1.1;
```
위와 같은 설정을 해주고 read_timeout 시간을 변경해줘도 어째서인지 우리 프로젝트에서는 고쳐지지 않아 안타깝게도 SSE는 사용할 수가 없었다.    

결국 새로운 방식을 찾기 시작했고 단방향 통신을 이어가고 싶던 점에서 웹소켓이 아닌 주로 웹보다는 앱에서 쓰이는 FCM 방식을 채택했다.