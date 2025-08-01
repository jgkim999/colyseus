# Colyseus Room 생명주기: 생성과 파괴 과정

## 📋 개요

이 문서는 Colyseus Room의 전체 생명주기를 상세히 설명합니다. Room이 생성되어 클라이언트가 입장하고, 최종적으로 파괴되는 모든 과정을 단계별로 분석합니다.

## 🔄 Room 생명주기 전체 플로우

```mermaid
graph TD
    A[클라이언트 매치메이킹 요청] --> B{기존 Room 존재?}
    B -->|Yes| C[기존 Room 입장]
    B -->|No| D[새 Room 생성 필요]

    D --> E[selectProcessIdToCreateRoom 호출]
    E --> F{로컬 프로세스?}
    F -->|Yes| G[로컬에서 Room 생성]
    F -->|No| H[원격 프로세스에 생성 요청]

    G --> I[handleCreateRoom 실행]
    H --> I

    I --> J[Room 인스턴스 생성]
    J --> K[Room.__init__ 호출]
    K --> L[onCreate 실행]
    L --> M[Room 상태: CREATED]
    M --> N[IPC 채널 구독]
    N --> O[Room 캐시 저장]
    O --> P[통계 업데이트]

    C --> Q[좌석 예약]
    P --> Q
    Q --> R[클라이언트 입장 처리]

    R --> S[Room 활성 상태]
    S --> T{클라이언트 퇴장?}
    T -->|Yes| U[onLeave 처리]
    T -->|No| S

    U --> V{마지막 클라이언트?}
    V -->|No| S
    V -->|Yes| W{autoDispose?}
    W -->|No| S
    W -->|Yes| X[Room 파괴 시작]

    X --> Y[dispose 이벤트 발생]
    Y --> Z[_dispose 실행]
    Z --> AA[onDispose 호출]
    AA --> BB[리소스 정리]
    BB --> CC[MatchMaker에서 제거]
    CC --> DD[Room 완전 파괴]
```

## 🚀 Room 생성 과정

### 1. 매치메이킹 요청 처리

```mermaid
sequenceDiagram
    participant Client as 클라이언트
    participant Server as 서버
    participant MM as MatchMaker
    participant Stats as 통계 시스템 Redis
    participant Process as 선택된 프로세스

    Client->>Server: joinOrCreate 요청
    Server->>MM: createRoom 호출
    MM->>Stats: fetchAll() - 프로세스 통계 조회 stats
    Stats-->>MM: 모든 프로세스 통계 반환
    MM->>MM: selectProcessIdToCreateRoom 실행
    MM->>Process: Room 생성 요청 (Redis pub/sub)
    Process->>Process: handleCreateRoom 실행
    Process-->>MM: Room 캐시 정보 반환
    MM-->>Server: SeatReservation 반환
    Server-->>Client: 입장 정보 전송
```

### 2. Room 인스턴스 생성

```mermaid
flowchart TD
    A[handleCreateRoom 시작] --> B[RegisteredHandler에서 Room 클래스 가져오기]
    B --> C[new handler.klass 실행]
    C --> D[roomId 생성]
    D --> E[room.__init__ 호출]

    E --> F[상태 관리 프로퍼티 설정]
    F --> G[patchRate 인터벌 설정]
    G --> H[clock.start 실행]
    H --> I[autoDisposeTimeout 설정]

    I --> J[roomName, presence 할당]
    J --> K[listing 캐시 생성]
    K --> L{onCreate 존재?}
    L -->|Yes| M[onCreate 실행]
    L -->|No| N[상태를 CREATED로 변경]
    M --> N

    N --> O[통계 업데이트]
    O --> P[이벤트 리스너 바인딩]
    P --> Q[IPC 채널 구독]
    Q --> R[Room 캐시 저장]
    R --> S[Room 생성 완료]
```

### 3. 이벤트 바인딩 및 초기화

```typescript
// MatchMaker.ts - handleCreateRoom()
room._events.on('lock', lockRoom.bind(this, room));
room._events.on('unlock', unlockRoom.bind(this, room));
room._events.on('join', onClientJoinRoom.bind(this, room));
room._events.on('leave', onClientLeaveRoom.bind(this, room));
room._events.on('visibility-change', onVisibilityChange.bind(this, room));
room._events.once('dispose', disposeRoom.bind(this, roomName, room));
```

## 👥 클라이언트 입장 과정

```mermaid
sequenceDiagram
    participant Client as 클라이언트
    participant Room as Room
    participant Auth as 인증 시스템

    Client->>Room: 입장 요청 (sessionId)
    Room->>Room: 좌석 예약 확인
    Room->>Auth: onAuth 실행
    Auth-->>Room: 인증 결과

    alt 인증 성공
        Room->>Room: clients 배열에 추가
        Room->>Room: onJoin 실행
        Room->>Room: 메시지 핸들러 바인딩
        Room->>Client: JOIN_ROOM 확인 메시지
        Room->>Client: 전체 상태 전송
        Room->>Room: join 이벤트 발생
    else 인증 실패
        Room->>Client: 연결 종료
    end
```

## 🔥 Room 파괴 과정

### 1. 파괴 트리거

```mermaid
graph TD
    A[파괴 트리거] --> B{트리거 유형}

    B -->|자동 파괴| C[모든 클라이언트 퇴장]
    B -->|수동 파괴| D[disconnect 호출]
    B -->|서버 종료| E[gracefullyShutdown]
    B -->|타임아웃| F[autoDisposeTimeout]

    C --> G[_disposeIfEmpty 체크]
    D --> H[강제 파괴 시작]
    E --> I[onBeforeShutdown 호출]
    F --> G

    G --> J{파괴 조건 만족?}
    J -->|Yes| K[dispose 이벤트 발생]
    J -->|No| L[파괴 취소]

    H --> K
    I --> K
    K --> M[_dispose 실행]
```

### 2. 클라이언트 정리 과정

```mermaid
sequenceDiagram
    participant Room as Room
    participant Client as 클라이언트
    participant Stats as 통계

    Note over Room: 클라이언트 퇴장 시작
    Room->>Client: 연결 종료 신호
    Room->>Room: _onLeave 실행
    Room->>Room: onLeave 호출 (사용자 정의)
    Room->>Room: clients 배열에서 제거
    Room->>Stats: CCU 감소
    Room->>Room: leave 이벤트 발생

    alt 재연결 대기
        Room->>Room: allowReconnection 처리
    else 일반 퇴장
        Room->>Room: _onAfterLeave 실행
        Room->>Room: _disposeIfEmpty 체크
    end
```

### 3. 리소스 정리 과정

```mermaid
flowchart TD
    A[_dispose 시작] --> B[상태를 DISPOSING으로 변경]
    B --> C[listing.remove 호출]
    C --> D{onDispose 존재?}

    D -->|Yes| E[onDispose 실행]
    D -->|No| F[인터벌 정리 시작]
    E --> F

    F --> G[patchInterval 정리]
    G --> H[simulationInterval 정리]
    H --> I[autoDisposeTimeout 정리]
    I --> J[clock.clear & stop]

    J --> K[disconnect 이벤트 발생]
    K --> L[disposeRoom 호출 - MatchMaker]

    L --> M[통계 업데이트]
    M --> N[핸들러 이벤트 발생]
    N --> O[IPC 구독 해제]
    O --> P[rooms 객체에서 제거]
    P --> Q[Room 완전 파괴]
```

## 📊 상태 전이 다이어그램

```mermaid
stateDiagram-v2
    [*] --> CREATING: Room 생성 시작
    CREATING --> CREATED: onCreate 완료
    CREATED --> DISPOSING: 파괴 트리거
    DISPOSING --> [*]: 정리 완료

    CREATED --> CREATED: 클라이언트 입장/퇴장
    CREATING --> [*]: onCreate 실패

    note right of CREATING
        - Room 인스턴스 생성
        - __init__ 실행
        - onCreate 호출
    end note

    note right of CREATED
        - 클라이언트 수용 가능
        - 메시지 처리 활성
        - 상태 동기화 진행
    end note

    note right of DISPOSING
        - 새 클라이언트 거부
        - 기존 클라이언트 정리
        - 리소스 해제
    end note
```

## 🔄 프로세스 간 통신 (IPC)

```mermaid
sequenceDiagram
    participant MM1 as MatchMaker 1
    participant Presence as Presence/Redis
    participant MM2 as MatchMaker 2
    participant Room as Room Instance

    MM1->>MM1: selectProcessIdToCreateRoom
    MM1->>Presence: publish to process channel
    Presence->>MM2: IPC 메시지 전달
    MM2->>MM2: handleCreateRoom 실행
    MM2->>Room: Room 인스턴스 생성
    Room->>MM2: Room 캐시 정보
    MM2->>Presence: 응답 publish
    Presence->>MM1: 응답 전달
    MM1->>MM1: Room 생성 완료 처리
```

## 🎯 핵심 코드 흐름

### Room 생성
```typescript
// 1. 매치메이킹 요청
const room = await matchMaker.createRoom(roomName, clientOptions);

// 2. 프로세스 선택
const selectedProcessId = await selectProcessIdToCreateRoom(roomName, clientOptions);

// 3. Room 생성 (로컬 또는 원격)
if (selectedProcessId === processId) {
  room = await handleCreateRoom(roomName, clientOptions);
} else {
  room = await requestFromIPC(presence, getProcessChannel(selectedProcessId), undefined, [roomName, clientOptions]);
}

// 4. 좌석 예약
return await reserveSeatFor(room, clientOptions, authData);
```

### Room 파괴
```typescript
// 1. 파괴 조건 체크
protected _disposeIfEmpty() {
  const willDispose = (
    this.#_onLeaveConcurrent === 0 &&
    this.#_autoDispose &&
    this._autoDisposeTimeout === undefined &&
    this.clients.length === 0 &&
    Object.keys(this.reservedSeats).length === 0
  );

  if (willDispose) {
    this._events.emit('dispose');
  }
}

// 2. 리소스 정리
protected async _dispose() {
  this._internalState = RoomInternalState.DISPOSING;
  this.listing.remove();

  if (this.onDispose) {
    await this.onDispose();
  }

  // 모든 인터벌/타이머 정리
  this.clock.clear();
  this.clock.stop();
}
```

## 📈 성능 모니터링

```mermaid
graph LR
    A[Room 생성] --> B[stats.local.roomCount++]
    C[클라이언트 입장] --> D[stats.local.ccu++]
    E[클라이언트 퇴장] --> F[stats.local.ccu--]
    G[Room 파괴] --> H[stats.local.roomCount--]

    B --> I[stats.persist]
    D --> I
    F --> I
    H --> I

    I --> J[Redis에 통계 저장]
    J --> K[다른 프로세스에서 조회 가능]
```

## 🛡️ 에러 처리 및 복구

```mermaid
flowchart TD
    A[Room 생성 실패] --> B{실패 원인}
    B -->|onCreate 에러| C[Room 인스턴스 정리]
    B -->|IPC 타임아웃| D[로컬에서 재시도]
    B -->|프로세스 불가용| E[헬스체크 실행]

    C --> F[에러 전파]
    D --> G[handleCreateRoom 실행]
    E --> H[프로세스 제외]

    H --> I[다른 프로세스 선택]
    I --> G
    G --> J[Room 생성 성공]

    F --> K[클라이언트에 에러 응답]
```

## 🎯 주요 특징

1. **비동기 처리**: 모든 생명주기 단계가 비동기로 처리
2. **분산 지원**: 여러 프로세스 간 Room 생성 분산
3. **자동 정리**: autoDispose를 통한 자동 리소스 관리
4. **상태 추적**: 실시간 통계 및 상태 모니터링
5. **에러 복구**: 프로세스 장애 시 자동 복구 메커니즘
6. **확장성**: 프로세스 추가/제거 시 자동 적응
7. **개발 편의성**: devMode에서 Room 상태 복원

---
*이 문서는 Colyseus Room의 전체 생명주기 분석을 바탕으로 작성되었습니다.*