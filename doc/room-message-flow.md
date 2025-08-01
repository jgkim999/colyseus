# Room 메시지 전달 시스템: Presence를 통한 클라이언트 통신

## 📋 개요

이 문서는 Colyseus Room이 Presence를 통해 클라이언트에게 메시지를 전달하는 과정을 상세히 분석하고 시각화합니다.

## 🔄 전체 메시지 흐름 아키텍처

```mermaid
graph TB
    %% 클라이언트들
    C1[Client 1]
    C2[Client 2]
    C3[Client 3]

    %% 프로세스들
    subgraph "Process A"
        RA[Room A]
        PA[Presence A]
    end

    subgraph "Process B"
        RB[Room B]
        PB[Presence B]
    end

    %% 중앙 메시지 브로커
    Redis[(Redis/Message Broker)]

    %% Transport 계층
    TA[Transport A]
    TB[Transport B]

    %% 연결 관계
    C1 <--> TA
    C2 <--> TA
    C3 <--> TB

    TA <--> RA
    TB <--> RB

    RA --> PA
    RB --> PB

    PA <--> Redis
    PB <--> Redis

    %% 메시지 흐름
    RA -.->|broadcast| PA
    PA -.->|publish| Redis
    Redis -.->|notify| PB
    PB -.->|forward| RB
    RB -.->|send| TB
    TB -.->|websocket| C3

    %% 스타일링
    classDef client fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    classDef room fill:#f3e5f5,stroke:#4a148c,stroke-width:2px
    classDef presence fill:#e8f5e8,stroke:#1b5e20,stroke-width:2px
    classDef transport fill:#fff3e0,stroke:#e65100,stroke-width:2px
    classDef broker fill:#ffebee,stroke:#c62828,stroke-width:3px

    class C1,C2,C3 client
    class RA,RB room
    class PA,PB presence
    class TA,TB transport
    class Redis broker
```

## 📨 Room 내부 메시지 브로드캐스트 흐름

```mermaid
sequenceDiagram
    participant GS as Game State
    participant R as Room
    participant CA as ClientArray
    participant C1 as Client 1
    participant C2 as Client 2
    participant P as Presence
    participant T as Transport

    Note over GS,T: 로컬 클라이언트 브로드캐스트

    GS->>R: State Change
    R->>R: broadcastPatch()
    R->>CA: serialize state
    CA->>C1: send patch
    CA->>C2: send patch
    C1->>T: WebSocket message
    C2->>T: WebSocket message

    Note over GS,T: 커스텀 메시지 브로드캐스트

    R->>R: broadcast("event", data)
    R->>CA: broadcastMessageType()
    CA->>C1: enqueueRaw(message)
    CA->>C2: enqueueRaw(message)

    Note over GS,T: Presence를 통한 통계 업데이트

    R->>P: hset("stats", "ccu", count)
    P->>P: store locally or Redis
```

## 🌐 프로세스 간 메시지 전달 (IPC)

```mermaid
sequenceDiagram
    participant R1 as Room (Process 1)
    participant P1 as Presence (Process 1)
    participant Redis as Redis Broker
    participant P2 as Presence (Process 2)
    participant R2 as Room (Process 2)
    participant C as Client

    Note over R1,C: 원격 룸 호출 시나리오

    R1->>P1: remoteRoomCall(roomId, method, args)
    P1->>Redis: publish("$roomId", {method, args})
    Redis->>P2: notify subscribers
    P2->>R2: invoke method(args)
    R2->>P2: return result
    P2->>Redis: publish response
    Redis->>P1: response received
    P1->>R1: return result

    Note over R1,C: 클라이언트에게 결과 전달

    R2->>C: send result via WebSocket
```

## 🔧 Presence 인터페이스 활용 패턴

```mermaid
graph TB
    Room[Room]

    %% Presence 기능들
    subgraph "Presence Interface"
        PubSub[Publish/Subscribe]
        KeyValue[Key-Value Store]
        Hash[Hash Operations]
        List[List Operations]
    end

    %% Room의 Presence 사용 패턴
    Room --> PubSub
    Room --> KeyValue
    Room --> Hash
    Room --> List

    %% 구체적 사용 사례
    PubSub --> |"IPC 통신"| IPC[Inter-Process Communication]
    KeyValue --> |"재연결 토큰"| Reconnect[Reconnection Tokens]
    Hash --> |"룸 통계"| Stats[Room Statistics]
    List --> |"대기열 관리"| Queue[Client Queue]

    %% 구현체
    subgraph "Implementations"
        LocalP[LocalPresence<br/>메모리 기반]
        RedisP[RedisPresence<br/>Redis 기반]
    end

    PubSub -.-> LocalP
    PubSub -.-> RedisP
    KeyValue -.-> LocalP
    KeyValue -.-> RedisP
    Hash -.-> LocalP
    Hash -.-> RedisP
    List -.-> LocalP
    List -.-> RedisP

    %% 스타일링
    classDef room fill:#f3e5f5,stroke:#4a148c,stroke-width:3px
    classDef interface fill:#e1f5fe,stroke:#01579b,stroke-width:2px
    classDef usecase fill:#e8f5e8,stroke:#1b5e20,stroke-width:2px
    classDef impl fill:#fff3e0,stroke:#e65100,stroke-width:2px

    class Room room
    class PubSub,KeyValue,Hash,List interface
    class IPC,Reconnect,Stats,Queue usecase
    class LocalP,RedisP impl
```

## 📡 메시지 타입별 전달 경로

```mermaid
flowchart TD
    Room[Room Instance]

    %% 메시지 타입들
    StateSync[State Synchronization]
    CustomMsg[Custom Messages]
    SystemMsg[System Messages]
    ErrorMsg[Error Messages]

    Room --> StateSync
    Room --> CustomMsg
    Room --> SystemMsg
    Room --> ErrorMsg

    %% State Sync 경로
    StateSync --> Serializer[Serializer]
    Serializer --> Patch[Delta Patches]
    Patch --> LocalClients[Local Clients]

    %% Custom Message 경로
    CustomMsg --> Broadcast[broadcast()]
    Broadcast --> MessageType[Message Type Encoding]
    MessageType --> LocalClients

    %% System Message 경로 (IPC)
    SystemMsg --> Presence[Presence]
    Presence --> IPCChannel[IPC Channel]
    IPCChannel --> RemoteProcess[Remote Process]
    RemoteProcess --> RemoteClients[Remote Clients]

    %% Error Message 경로
    ErrorMsg --> ErrorHandler[Error Handler]
    ErrorHandler --> ClientError[Client Error Response]
    ClientError --> Transport[Transport Layer]

    %% Transport to Clients
    LocalClients --> Transport
    RemoteClients --> RemoteTransport[Remote Transport]
    Transport --> WebSocket1[WebSocket 1]
    Transport --> WebSocket2[WebSocket 2]
    RemoteTransport --> WebSocket3[WebSocket 3]

    %% 스타일링
    classDef room fill:#f3e5f5,stroke:#4a148c,stroke-width:3px
    classDef msgtype fill:#e1f5fe,stroke:#01579b,stroke-width:2px
    classDef process fill:#e8f5e8,stroke:#1b5e20,stroke-width:2px
    classDef transport fill:#fff3e0,stroke:#e65100,stroke-width:2px
    classDef client fill:#ffebee,stroke:#c62828,stroke-width:2px

    class Room room
    class StateSync,CustomMsg,SystemMsg,ErrorMsg msgtype
    class Serializer,Broadcast,Presence,ErrorHandler process
    class Transport,RemoteTransport,IPCChannel transport
    class WebSocket1,WebSocket2,WebSocket3,LocalClients,RemoteClients client
```

## 🔄 실시간 상태 동기화 시퀀스

```mermaid
sequenceDiagram
    participant GS as Game State
    participant R as Room
    participant S as Serializer
    participant CA as ClientArray
    participant C1 as Local Client 1
    participant C2 as Local Client 2
    participant P as Presence
    participant RC as Remote Client

    Note over GS,RC: 게임 상태 변경 및 동기화

    GS->>R: state.player.x = 100
    R->>R: setSimulationInterval tick
    R->>R: broadcastPatch()

    alt Has Simulation Interval
        Note over R: clock.tick() called by simulation
    else No Simulation Interval
        R->>R: clock.tick()
    end

    R->>S: applyPatches(clients, state)
    S->>S: calculate delta patches
    S->>CA: send patches to clients

    par Local Clients
        CA->>C1: raw(patchData)
        CA->>C2: raw(patchData)
    and Remote Process Notification
        R->>P: publish("room:update", roomId)
        P->>RC: notify remote process
    end

    Note over GS,RC: 클라이언트 상태 적용

    C1->>C1: apply patch to local state
    C2->>C2: apply patch to local state
    RC->>RC: apply patch to local state
```

## 🎯 메시지 큐잉 및 배치 처리

```mermaid
graph TB
    Room[Room]

    %% 메시지 큐 시스템
    subgraph "Message Queuing"
        Queue[Message Queue]
        Batch[Batch Processor]
        Priority[Priority Handler]
    end

    %% 메시지 타입별 처리
    Room --> Queue
    Queue --> Priority
    Priority --> Batch

    %% 우선순위별 분류
    Priority --> HighPrio[High Priority<br/>- Error Messages<br/>- System Events]
    Priority --> MedPrio[Medium Priority<br/>- Game State<br/>- Custom Messages]
    Priority --> LowPrio[Low Priority<br/>- Statistics<br/>- Logs]

    %% 배치 처리
    Batch --> LocalBatch[Local Batch<br/>Direct WebSocket]
    Batch --> RemoteBatch[Remote Batch<br/>via Presence]

    %% 전송 계층
    LocalBatch --> Transport[Transport Layer]
    RemoteBatch --> Presence[Presence Layer]

    Presence --> IPC[IPC Channel]
    IPC --> RemoteTransport[Remote Transport]

    %% 최종 클라이언트 전달
    Transport --> LocalClients[Local Clients]
    RemoteTransport --> RemoteClients[Remote Clients]

    %% 스타일링
    classDef room fill:#f3e5f5,stroke:#4a148c,stroke-width:3px
    classDef queue fill:#e1f5fe,stroke:#01579b,stroke-width:2px
    classDef priority fill:#e8f5e8,stroke:#1b5e20,stroke-width:2px
    classDef transport fill:#fff3e0,stroke:#e65100,stroke-width:2px
    classDef client fill:#ffebee,stroke:#c62828,stroke-width:2px

    class Room room
    class Queue,Batch,Priority queue
    class HighPrio,MedPrio,LowPrio priority
    class Transport,RemoteTransport,Presence,IPC transport
    class LocalClients,RemoteClients client
```

## 🛡️ 에러 처리 및 복구 메커니즘

```mermaid
sequenceDiagram
    participant R as Room
    participant P as Presence
    participant C as Client
    participant ER as Error Recovery

    Note over R,ER: 정상 메시지 전달

    R->>P: send message via presence
    P->>C: deliver to client
    C->>P: acknowledge receipt
    P->>R: confirm delivery

    Note over R,ER: 전달 실패 시나리오

    R->>P: send message via presence
    P->>C: attempt delivery
    C--xP: connection lost
    P->>ER: delivery failed

    ER->>ER: check retry policy

    alt Retry Available
        ER->>P: retry delivery
        P->>C: reattempt delivery

        alt Success
            C->>P: acknowledge
            P->>R: confirm delivery
        else Max Retries Exceeded
            ER->>R: notify permanent failure
            R->>R: handle client disconnect
        end

    else No Retry Policy
        ER->>R: immediate failure notification
        R->>R: remove client from room
    end
```

## 📊 성능 최적화 패턴

```mermaid
graph TB
    Room[Room]

    %% 최적화 전략들
    subgraph "Performance Optimizations"
        Batching[Message Batching]
        Compression[Data Compression]
        Filtering[Client Filtering]
        Caching[Message Caching]
    end

    Room --> Batching
    Room --> Compression
    Room --> Filtering
    Room --> Caching

    %% 배치 처리
    Batching --> TimeBatch[Time-based Batching<br/>10ms windows]
    Batching --> SizeBatch[Size-based Batching<br/>Max 1KB per batch]

    %% 압축
    Compression --> DeltaComp[Delta Compression<br/>Only changed data]
    Compression --> BinaryComp[Binary Compression<br/>MessagePack]

    %% 필터링
    Filtering --> InterestMgmt[Interest Management<br/>Spatial filtering]
    Filtering --> PermissionFilter[Permission Filtering<br/>Role-based access]

    %% 캐싱
    Caching --> StateCache[State Caching<br/>Avoid recalculation]
    Caching --> MessageCache[Message Caching<br/>Reuse common messages]

    %% Presence 활용
    TimeBatch --> Presence[Presence]
    SizeBatch --> Presence
    DeltaComp --> Presence
    BinaryComp --> Presence

    %% 최종 전달
    Presence --> OptimizedDelivery[Optimized Message Delivery]

    %% 스타일링
    classDef room fill:#f3e5f5,stroke:#4a148c,stroke-width:3px
    classDef optimization fill:#e1f5fe,stroke:#01579b,stroke-width:2px
    classDef technique fill:#e8f5e8,stroke:#1b5e20,stroke-width:2px
    classDef presence fill:#fff3e0,stroke:#e65100,stroke-width:2px
    classDef delivery fill:#ffebee,stroke:#c62828,stroke-width:2px

    class Room room
    class Batching,Compression,Filtering,Caching optimization
    class TimeBatch,SizeBatch,DeltaComp,BinaryComp,InterestMgmt,PermissionFilter,StateCache,MessageCache technique
    class Presence presence
    class OptimizedDelivery delivery
```

## 🔍 디버깅 및 모니터링

```mermaid
graph LR
    Room[Room]

    %% 모니터링 포인트들
    subgraph "Monitoring Points"
        MsgCount[Message Count]
        Latency[Message Latency]
        Errors[Error Rate]
        Throughput[Throughput]
    end

    Room --> MsgCount
    Room --> Latency
    Room --> Errors
    Room --> Throughput

    %% Presence를 통한 메트릭 수집
    MsgCount --> Presence[Presence Metrics]
    Latency --> Presence
    Errors --> Presence
    Throughput --> Presence

    %% 메트릭 저장
    Presence --> MetricsDB[(Metrics Database)]

    %% 알림 시스템
    Presence --> AlertSystem[Alert System]
    AlertSystem --> Dashboard[Monitoring Dashboard]
    AlertSystem --> Notifications[Error Notifications]

    %% 로그 시스템
    Room --> Logger[Debug Logger]
    Logger --> LogFiles[Log Files]
    Logger --> LogAggregator[Log Aggregator]

    %% 스타일링
    classDef room fill:#f3e5f5,stroke:#4a148c,stroke-width:3px
    classDef monitoring fill:#e1f5fe,stroke:#01579b,stroke-width:2px
    classDef presence fill:#e8f5e8,stroke:#1b5e20,stroke-width:2px
    classDef storage fill:#fff3e0,stroke:#e65100,stroke-width:2px
    classDef output fill:#ffebee,stroke:#c62828,stroke-width:2px

    class Room room
    class MsgCount,Latency,Errors,Throughput monitoring
    class Presence,AlertSystem,Logger presence
    class MetricsDB,LogFiles storage
    class Dashboard,Notifications,LogAggregator output
```

## 🎯 핵심 메시지 전달 패턴 요약

### 1. **직접 전달 (Local Clients)**

```typescript
// Room 내 로컬 클라이언트들에게 직접 전달
room.broadcast("message", data);
// → ClientArray → Transport → WebSocket
```

### 2. **Presence를 통한 IPC (Remote Clients)**

```typescript
// 다른 프로세스의 클라이언트들에게 전달
await remoteRoomCall(roomId, "broadcast", ["message", data]);
// → Presence → IPC Channel → Remote Room → Remote Clients
```

### 3. **상태 동기화 (State Patches)**

```typescript
// 게임 상태 변경 시 자동 동기화
room.state.player.x = 100;
// → broadcastPatch() → Serializer → Delta Patches → All Clients
```

### 4. **통계 및 메타데이터 (Statistics)**

```typescript
// 룸 통계 정보 공유
room.presence.hset("room:stats", "ccu", clientCount);
// → Presence → Redis/Local Storage → Other Processes
```

## 🔧 구현 세부사항

### Presence 인터페이스 활용

- **publish/subscribe**: 프로세스 간 실시간 메시지 전달
- **key-value**: 재연결 토큰, 임시 데이터 저장
- **hash**: 룸 통계, 메타데이터 관리
- **list**: 클라이언트 대기열, 메시지 큐

### 메시지 최적화

- **배치 처리**: 여러 메시지를 묶어서 전송
- **압축**: Delta 압축으로 대역폭 절약
- **필터링**: 관심 영역 기반 선택적 전송
- **캐싱**: 반복 메시지 재사용

이러한 구조를 통해 Colyseus는 확장 가능하고 효율적인 실시간 메시지 전달 시스템을 제공합니다.

---
*이 문서는 Colyseus 0.16.x 버전을 기준으로 작성되었습니다.*