# Colyseus 클래스 의존성 다이어그램

## 📋 개요

이 문서는 Colyseus 프레임워크의 핵심 클래스들 간의 의존성 관계를 Mermaid 다이어그램으로 시각화합니다.

## 🏗️ 전체 아키텍처 의존성

```mermaid
graph TB
    %% 외부 의존성
    HTTP[HTTP Server]
    WS[WebSocket]
    Redis[(Redis)]
    MongoDB[(MongoDB)]

    %% 핵심 컴포넌트
    Server[Server]
    Transport[Transport]
    MatchMaker[MatchMaker]
    Room[Room]

    %% 인터페이스
    Presence{{"Presence<br/>(Interface)"}}
    Driver{{"MatchMakerDriver<br/>(Interface)"}}

    %% 구현체
    LocalPresence[LocalPresence]
    RedisPresence[RedisPresence]
    LocalDriver[LocalDriver]
    MongooseDriver[MongooseDriver]

    %% 보조 클래스
    RegisteredHandler[RegisteredHandler]
    Clock[Clock]
    ClientArray[ClientArray]
    Stats[Stats]

    %% 의존성 관계
    Server --> Transport
    Server --> Presence
    Server --> Driver
    Server --> MatchMaker

    MatchMaker --> Presence
    MatchMaker --> Driver
    MatchMaker --> RegisteredHandler
    MatchMaker --> Room
    MatchMaker --> Stats

    Room --> Presence
    Room --> Clock
    Room --> ClientArray

    %% 인터페이스 구현
    LocalPresence -.-> Presence
    RedisPresence -.-> Presence
    LocalDriver -.-> Driver
    MongooseDriver -.-> Driver

    %% 외부 의존성
    Transport --> HTTP
    Transport --> WS
    RedisPresence --> Redis
    MongooseDriver --> MongoDB

    %% 스타일링
    classDef interface fill:#e1f5fe,stroke:#01579b,stroke-width:2px
    classDef core fill:#f3e5f5,stroke:#4a148c,stroke-width:2px
    classDef impl fill:#e8f5e8,stroke:#1b5e20,stroke-width:2px
    classDef external fill:#fff3e0,stroke:#e65100,stroke-width:2px

    class Presence,Driver interface
    class Server,MatchMaker,Room,Transport core
    class LocalPresence,RedisPresence,LocalDriver,MongooseDriver impl
    class HTTP,WS,Redis,MongoDB external
```

## 🔗 Server 중심 의존성

```mermaid
graph LR
    Server[Server]

    %% 직접 의존성
    Server --> Transport
    Server --> Presence
    Server --> Driver
    Server --> MatchMaker

    %% 의존성 주입
    Server -.->|"inject"| LocalPresence
    Server -.->|"inject"| RedisPresence
    Server -.->|"inject"| LocalDriver
    Server -.->|"inject"| MongooseDriver

    %% 설정
    ServerOptions[ServerOptions] --> Server

    %% 스타일링
    classDef server fill:#f3e5f5,stroke:#4a148c,stroke-width:3px
    classDef dependency fill:#e8f5e8,stroke:#1b5e20,stroke-width:2px
    classDef injection fill:#fff3e0,stroke:#e65100,stroke-width:1px,stroke-dasharray: 5 5

    class Server server
    class Transport,Presence,Driver,MatchMaker dependency
    class LocalPresence,RedisPresence,LocalDriver,MongooseDriver injection
```

## 🎯 MatchMaker 의존성 상세

```mermaid
graph TB
    MatchMaker[MatchMaker]

    %% 핵심 의존성
    MatchMaker --> Presence
    MatchMaker --> Driver

    %% 룸 관리
    MatchMaker --> RegisteredHandler
    MatchMaker --> Room

    %% 통계 및 모니터링
    MatchMaker --> Stats
    MatchMaker --> IPC[IPC]

    %% 보조 기능
    MatchMaker --> Utils[Utils]
    MatchMaker --> Debug[Debug]

    %% RegisteredHandler 의존성
    RegisteredHandler --> Room
    RegisteredHandler --> Lobby[Lobby]

    %% Driver 구현체
    LocalDriver -.-> Driver
    MongooseDriver -.-> Driver

    %% Presence 구현체
    LocalPresence -.-> Presence
    RedisPresence -.-> Presence

    %% 스타일링
    classDef matchmaker fill:#f3e5f5,stroke:#4a148c,stroke-width:3px
    classDef core fill:#e1f5fe,stroke:#01579b,stroke-width:2px
    classDef support fill:#e8f5e8,stroke:#1b5e20,stroke-width:2px

    class MatchMaker matchmaker
    class Presence,Driver,Room core
    class RegisteredHandler,Stats,IPC,Utils,Debug,Lobby support
```

## 🏠 Room 생명주기 의존성

```mermaid
graph TB
    Room[Room]

    %% 핵심 의존성
    Room --> Presence
    Room --> Clock
    Room --> ClientArray

    %% 직렬화
    Room --> Serializer
    Serializer --> SchemaSerializer
    Serializer --> NoneSerializer

    %% 클라이언트 관리
    ClientArray --> Client
    Client --> Transport

    %% 타이밍 시스템
    Clock --> Timer[@colyseus/timer]

    %% 상태 관리
    Room --> State[Game State]

    %% 이벤트 시스템
    Room --> EventEmitter

    %% 예외 처리
    Room --> RoomExceptions

    %% 스타일링
    classDef room fill:#f3e5f5,stroke:#4a148c,stroke-width:3px
    classDef core fill:#e1f5fe,stroke:#01579b,stroke-width:2px
    classDef client fill:#e8f5e8,stroke:#1b5e20,stroke-width:2px
    classDef external fill:#fff3e0,stroke:#e65100,stroke-width:2px

    class Room room
    class Presence,Clock,ClientArray,Serializer core
    class Client,Transport,EventEmitter client
    class Timer external
```

## 🔄 데이터 흐름 의존성

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server
    participant M as MatchMaker
    participant D as Driver
    participant P as Presence
    participant R as Room

    C->>S: Join Request
    S->>M: joinOrCreate()
    M->>D: findOne()
    D-->>M: Room Data

    alt Room Not Found
        M->>M: createRoom()
        M->>P: IPC Subscribe
        M->>R: new Room()
        R->>P: Store Data
    end

    M->>R: reserveSeat()
    R->>P: Check Availability
    P-->>R: Seat Reserved
    R-->>M: Reservation
    M-->>S: Room Info
    S-->>C: Connection Details

    C->>R: WebSocket Connect
    R->>R: onJoin()
    R->>P: Update Stats
```

## 🏭 팩토리 패턴 의존성

```mermaid
graph TB
    %% 팩토리들
    PresenceFactory[PresenceFactory]
    DriverFactory[DriverFactory]
    TransportFactory[TransportFactory]

    %% 설정
    Config[Configuration]

    %% 생성되는 객체들
    Config --> PresenceFactory
    Config --> DriverFactory
    Config --> TransportFactory

    PresenceFactory --> LocalPresence
    PresenceFactory --> RedisPresence

    DriverFactory --> LocalDriver
    DriverFactory --> MongooseDriver
    DriverFactory --> RedisDriver

    TransportFactory --> WebSocketTransport
    TransportFactory --> TCPTransport
    TransportFactory --> H3Transport

    %% Server에서 사용
    Server --> PresenceFactory
    Server --> DriverFactory
    Server --> TransportFactory

    %% 스타일링
    classDef factory fill:#fff3e0,stroke:#e65100,stroke-width:2px
    classDef product fill:#e8f5e8,stroke:#1b5e20,stroke-width:2px
    classDef config fill:#f3e5f5,stroke:#4a148c,stroke-width:2px

    class PresenceFactory,DriverFactory,TransportFactory factory
    class LocalPresence,RedisPresence,LocalDriver,MongooseDriver,RedisDriver,WebSocketTransport,TCPTransport,H3Transport product
    class Config,Server config
```

## 🔧 인터페이스 분리 원칙

```mermaid
graph LR
    %% 인터페이스들
    IPresence{{"IPresence"}}
    IDriver{{"IMatchMakerDriver"}}
    ITransport{{"ITransport"}}
    ISerializer{{"ISerializer"}}

    %% 세분화된 인터페이스들
    IPresence --> IPubSub{{"IPubSub"}}
    IPresence --> IKeyValue{{"IKeyValue"}}
    IPresence --> IHash{{"IHash"}}
    IPresence --> IList{{"IList"}}

    IDriver --> IQuery{{"IQuery"}}
    IDriver --> ICache{{"ICache"}}
    IDriver --> ICleanup{{"ICleanup"}}

    %% 구현체들
    LocalPresence -.-> IPresence
    RedisPresence -.-> IPresence

    LocalDriver -.-> IDriver
    MongooseDriver -.-> IDriver

    %% 클라이언트들
    MatchMaker --> IPubSub
    MatchMaker --> IQuery
    MatchMaker --> ICache

    Room --> IKeyValue
    Room --> IHash

    %% 스타일링
    classDef interface fill:#e1f5fe,stroke:#01579b,stroke-width:2px
    classDef subinterface fill:#f1f8e9,stroke:#33691e,stroke-width:1px
    classDef impl fill:#e8f5e8,stroke:#1b5e20,stroke-width:2px
    classDef client fill:#f3e5f5,stroke:#4a148c,stroke-width:2px

    class IPresence,IDriver,ITransport,ISerializer interface
    class IPubSub,IKeyValue,IHash,IList,IQuery,ICache,ICleanup subinterface
    class LocalPresence,RedisPresence,LocalDriver,MongooseDriver impl
    class MatchMaker,Room client
```

## 📦 모듈 의존성

```mermaid
graph TB
    %% 패키지들
    Core["@colyseus/core"]
    Timer["@colyseus/timer"]
    Schema["@colyseus/schema"]
    Transport["@colyseus/ws-transport"]
    Redis["@colyseus/redis-presence"]
    Mongoose["@colyseus/mongoose-driver"]

    %% 외부 의존성
    NodeJS[Node.js]
    WebSocket[ws]
    RedisLib[redis]
    MongooseLib[mongoose]

    %% 의존성 관계
    Core --> Timer
    Core --> Schema
    Core --> Transport

    Transport --> WebSocket
    Redis --> RedisLib
    Mongoose --> MongooseLib

    %% 선택적 의존성
    Core -.->|optional| Redis
    Core -.->|optional| Mongoose

    %% 런타임 의존성
    Core --> NodeJS

    %% 스타일링
    classDef core fill:#f3e5f5,stroke:#4a148c,stroke-width:3px
    classDef colyseus fill:#e1f5fe,stroke:#01579b,stroke-width:2px
    classDef external fill:#fff3e0,stroke:#e65100,stroke-width:2px
    classDef optional fill:#e8f5e8,stroke:#1b5e20,stroke-width:1px,stroke-dasharray: 5 5

    class Core core
    class Timer,Schema,Transport colyseus
    class NodeJS,WebSocket,RedisLib,MongooseLib external
    class Redis,Mongoose optional
```

## 🎯 주요 의존성 특징

### 1. **의존성 주입 (Dependency Injection)**
- Server가 모든 핵심 컴포넌트의 의존성을 관리
- 인터페이스 기반으로 구현체를 런타임에 주입
- 테스트와 확장성을 위한 유연한 구조

### 2. **인터페이스 분리 (Interface Segregation)**
- Presence와 Driver는 인터페이스로 추상화
- 클라이언트는 필요한 기능만 의존
- 구현체 교체가 용이한 구조

### 3. **단방향 의존성 (Unidirectional Dependencies)**
- 상위 레벨이 하위 레벨에 의존
- 순환 의존성 방지
- 명확한 계층 구조

### 4. **느슨한 결합 (Loose Coupling)**
- 인터페이스를 통한 간접 의존
- 구현체 변경이 클라이언트에 영향 없음
- 독립적인 테스트와 개발 가능

---
*이 다이어그램들은 Colyseus 0.16.x 버전을 기준으로 작성되었습니다.*