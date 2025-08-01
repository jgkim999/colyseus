# Colyseus 아키텍처: Presence, MatchMaker, Room 의존성 관리 및 분리 구조

## 📋 개요

Colyseus는 **Presence**, **MatchMaker**, **Room** 세 가지 핵심 컴포넌트를 통해 멀티플레이어 게임 서버를 구성합니다. 각 컴포넌트는 명확한 책임 분리와 의존성 주입을 통해 확장 가능하고 유지보수가 용이한 구조를 제공합니다.

## 🏗️ 전체 아키텍처 구조

```
┌─────────────────────────────────────────────────────────────┐
│                        Server                               │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │  Transport  │  │ MatchMaker  │  │      Presence       │  │
│  │             │  │             │  │                     │  │
│  │ - WebSocket │  │ - Room 생성 │  │ - 프로세스간 통신   │  │
│  │ - HTTP      │  │ - 매칭 로직 │  │ - 데이터 공유       │  │
│  │ - TCP       │  │ - 로드밸런싱│  │ - 이벤트 발행/구독  │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
│                           │                    │            │
│  ┌─────────────────────────┼────────────────────┼──────────┐ │
│  │                    Room │ Management         │          │ │
│  │  ┌─────────────┐        │        ┌─────────────────────┐│ │
│  │  │    Room     │◄───────┘        │   RegisteredHandler ││ │
│  │  │             │                 │                     ││ │
│  │  │ - 게임 로직 │                 │ - 룸 타입 관리      ││ │
│  │  │ - 상태 관리 │                 │ - 필터링/정렬       ││ │
│  │  │ - 클라이언트│                 │ - 이벤트 처리       ││ │
│  │  │   관리      │                 │                     ││ │
│  │  └─────────────┘                 └─────────────────────┘│ │
│  └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## 🔗 의존성 관계 분석

### 1. Server → 모든 컴포넌트 (의존성 주입)

```typescript
export class Server {
  protected presence: Presence;
  protected driver: MatchMakerDriver;
  public transport: Transport;

  constructor(options: ServerOptions = {}) {
    // 의존성 주입 또는 기본값 사용
    this.presence = options.presence || new LocalPresence();
    this.driver = options.driver || new LocalDriver();
    this.transport = options.transport || this.getDefaultTransport();
    
    // MatchMaker에 의존성 전달
    matchMaker.setup(
      this.presence,
      this.driver,
      options.publicAddress,
      options.selectProcessIdToCreateRoom,
    );
  }
}
```

**특징:**
- **의존성 주입 패턴**: 인터페이스 기반으로 구현체를 주입
- **기본 구현체 제공**: Local 환경용 기본 구현
- **확장성**: Redis, MongoDB 등 다양한 구현체로 교체 가능

### 2. MatchMaker → Presence + Driver (강한 의존성)

```typescript
// MatchMaker.ts
export let presence: Presence;
export let driver: MatchMakerDriver;

export async function setup(
  _presence?: Presence,
  _driver?: MatchMakerDriver,
  _publicAddress?: string,
  _selectProcessIdToCreateRoom?: SelectProcessIdCallback,
) {
  presence = _presence || new LocalPresence();
  driver = _driver || new LocalDriver();
  // ...
}
```

**의존성 사용 패턴:**

#### Presence 활용
```typescript
// 프로세스간 통신
await subscribeIPC(presence, processId, getProcessChannel(), handler);

// 룸 생성 동시성 제어
const concurrency = await presence.hincrbyex(hkey, concurrencyKey, 1, timeout);

// 헬스체크
await requestFromIPC(presence, getProcessChannel(processId), 'healthcheck');
```

#### Driver 활용
```typescript
// 룸 검색
const room = await driver.findOne({ locked: false, name: roomName });

// 룸 데이터 생성
room.listing = driver.createInstance({ name: roomName, processId });

// 정리 작업
await driver.cleanup(processId);
```

### 3. Room → Presence (약한 의존성)

```typescript
export abstract class Room {
  public presence: Presence;
  
  // MatchMaker에서 주입
  // room.presence = presence;
}
```

**사용 목적:**
- **재연결 관리**: 클라이언트 재연결 토큰 저장
- **개발 모드**: 룸 복원 데이터 캐싱
- **통계**: 룸 상태 정보 공유

## 🔄 컴포넌트별 책임 분리

### 1. Presence (데이터 계층)

```typescript
export interface Presence {
  // 발행/구독 패턴
  subscribe(topic: string, callback: Function);
  publish(topic: string, data: any);
  
  // 키-값 저장소
  set(key: string, value: string);
  get(key: string);
  
  // 해시 테이블
  hset(key: string, field: string, value: string);
  hget(key: string, field: string);
  
  // 리스트/큐
  lpush(key: string, ...values: string[]);
  brpop(...args: [...keys: string[], timeout: number]);
}
```

**책임:**
- 프로세스간 데이터 공유
- 이벤트 발행/구독
- 분산 락 메커니즘
- 임시 데이터 저장 (TTL 지원)

**구현체:**
- `LocalPresence`: 단일 프로세스용 메모리 기반
- `RedisPresence`: 다중 프로세스용 Redis 기반

### 2. MatchMaker (비즈니스 로직 계층)

```typescript
export interface MatchMakerDriver {
  createInstance(initialValues: any): IRoomCache;
  findOne(conditions: Partial<IRoomCache>): Promise<RoomCache>;
  query(conditions: Partial<IRoomCache>): Promise<RoomCache[]>;
  cleanup(processId: string): Promise<void>;
}
```

**책임:**
- 룸 생성/매칭 로직
- 로드 밸런싱
- 프로세스 헬스체크
- 동시성 제어

**핵심 기능:**
```typescript
// 룸 생성 또는 참가
export async function joinOrCreate(roomName: string, options: ClientOptions) {
  let room = await findOneRoomAvailable(roomName, options);
  
  if (!room) {
    // 동시성 제어를 통한 룸 생성
    await concurrentJoinOrCreateRoomLock(handler, concurrencyKey, async () => {
      room = await createRoom(roomName, options);
    });
  }
  
  return await reserveSeatFor(room, options);
}
```

### 3. Room (애플리케이션 계층)

```typescript
export abstract class Room {
  public clock: Clock;
  public clients: ClientArray;
  public state: State;
  public presence: Presence;
  
  // 생명주기 메서드
  public onCreate?(options: any): void | Promise<any>;
  public onJoin?(client: Client): void | Promise<any>;
  public onLeave?(client: Client): void | Promise<any>;
  public onDispose?(): void | Promise<any>;
}
```

**책임:**
- 게임 로직 실행
- 클라이언트 상태 관리
- 실시간 동기화
- 타이밍 이벤트 처리

## 🔧 의존성 주입 패턴

### 1. 인터페이스 기반 설계

```typescript
// 추상화된 인터페이스
interface Presence { /* ... */ }
interface MatchMakerDriver { /* ... */ }

// 구체적인 구현
class LocalPresence implements Presence { /* ... */ }
class RedisPresence implements Presence { /* ... */ }
class LocalDriver implements MatchMakerDriver { /* ... */ }
class MongooseDriver implements MatchMakerDriver { /* ... */ }
```

### 2. 런타임 의존성 주입

```typescript
// 개발 환경
const server = new Server({
  presence: new LocalPresence(),
  driver: new LocalDriver(),
});

// 프로덕션 환경
const server = new Server({
  presence: new RedisPresence({ host: 'redis-cluster' }),
  driver: new MongooseDriver({ uri: 'mongodb://cluster' }),
});
```

### 3. 팩토리 패턴 활용

```typescript
class PresenceFactory {
  static create(type: 'local' | 'redis', options?: any): Presence {
    switch (type) {
      case 'local': return new LocalPresence();
      case 'redis': return new RedisPresence(options);
      default: throw new Error(`Unknown presence type: ${type}`);
    }
  }
}
```

## 🚀 확장성 및 분리 이점

### 1. 수평 확장 지원

```typescript
// 단일 프로세스 → 다중 프로세스 전환
const server = new Server({
  presence: new RedisPresence({
    host: process.env.REDIS_HOST,
    port: process.env.REDIS_PORT,
  }),
  driver: new RedisDriver({
    // 룸 메타데이터도 Redis에 저장
  }),
});
```

### 2. 테스트 용이성

```typescript
// 테스트용 Mock 구현
class MockPresence implements Presence {
  private data = new Map();
  
  async set(key: string, value: string) {
    this.data.set(key, value);
  }
  
  async get(key: string) {
    return this.data.get(key);
  }
}

// 테스트에서 사용
const testServer = new Server({
  presence: new MockPresence(),
  driver: new MockDriver(),
});
```

### 3. 기능별 독립 배포

```typescript
// 매치메이킹 전용 서버
const matchmakingServer = new Server({
  presence: new RedisPresence(),
  driver: new DatabaseDriver(),
  // Room 인스턴스는 생성하지 않음
});

// 게임 로직 전용 서버
const gameServer = new Server({
  presence: new RedisPresence(),
  driver: new LocalDriver(), // 로컬 캐시만 사용
});
```

## 🔄 데이터 흐름 및 통신 패턴

### 1. 클라이언트 연결 흐름

```
Client Request → Server → MatchMaker → Driver → Room Creation
                    ↓
                Presence ← Room ← MatchMaker ← Server
```

```typescript
// 1. 클라이언트 요청 수신
server.handleMatchMakeRequest(req, res);

// 2. MatchMaker에서 룸 검색/생성
const room = await matchMaker.joinOrCreate(roomName, options);

// 3. Driver를 통한 룸 데이터 조회
const availableRoom = await driver.findOne({ name: roomName, locked: false });

// 4. Presence를 통한 프로세스간 통신
const roomInstance = await remoteRoomCall(room.roomId, '_reserveSeat', [sessionId]);

// 5. Room에서 클라이언트 수락
await room.onJoin(client, options);
```

### 2. 프로세스간 통신 패턴

```typescript
// IPC 채널 구독
await subscribeIPC(presence, processId, getProcessChannel(), (method, args) => {
  if (method === 'healthcheck') {
    return true;
  } else {
    return handleCreateRoom.apply(undefined, args);
  }
});

// 원격 룸 호출
export async function remoteRoomCall(roomId: string, method: string, args?: any[]) {
  const room = rooms[roomId];
  
  if (!room) {
    // 다른 프로세스의 룸 호출
    return await requestFromIPC(presence, getRoomChannel(roomId), method, args);
  } else {
    // 로컬 룸 직접 호출
    return room[method].apply(room, args);
  }
}
```

### 3. 상태 동기화 패턴

```typescript
// Room에서 상태 변경 시
room._events.on('join', onClientJoinRoom.bind(this, room));
room._events.on('leave', onClientLeaveRoom.bind(this, room));
room._events.on('lock', lockRoom.bind(this, room));

// MatchMaker에서 통계 업데이트
function onClientJoinRoom(room: Room, client: Client) {
  stats.local.ccu++;
  stats.persist(); // Presence를 통해 전역 통계 업데이트
  
  handlers[room.roomName].emit('join', room, client);
}
```

## 🛡️ 장애 격리 및 복구

### 1. 프로세스 헬스체크

```typescript
export async function healthCheckProcessId(processId: string) {
  try {
    await requestFromIPC(
      presence,
      getProcessChannel(processId),
      'healthcheck',
      [],
      REMOTE_ROOM_SHORT_TIMEOUT,
    );
    return true;
  } catch (e) {
    // 응답 실패 시 프로세스 제외
    await stats.excludeProcess(processId);
    await removeRoomsByProcessId(processId);
    return false;
  }
}
```

### 2. 자동 복구 메커니즘

```typescript
// 룸 생성 실패 시 대체 프로세스 선택
try {
  room = await requestFromIPC(presence, getProcessChannel(selectedProcessId), ...);
} catch (e) {
  if (e.message === "ipc_timeout") {
    // 헬스체크 후 로컬에서 생성
    if (enableHealthChecks) {
      await stats.excludeProcess(selectedProcessId);
    }
    room = await handleCreateRoom(roomName, clientOptions);
  }
}
```

### 3. 우아한 종료

```typescript
export async function gracefullyShutdown() {
  // 1. 새로운 룸 생성 중단
  await stats.excludeProcess(processId);
  
  // 2. 기존 룸 잠금
  for (const roomId in rooms) {
    const room = rooms[roomId];
    room.lock();
    room.onBeforeShutdown();
  }
  
  // 3. 모든 룸 정리 대기
  await noActiveRooms;
  
  // 4. 리소스 정리
  presence.unsubscribe(getProcessChannel());
  return Promise.all(disconnectAll());
}
```

## 📊 성능 최적화 전략

### 1. 연결 풀링

```typescript
class RedisPresence implements Presence {
  private connectionPool: Redis[];
  
  constructor(options: RedisOptions) {
    this.connectionPool = Array.from({ length: options.poolSize }, 
      () => new Redis(options)
    );
  }
  
  private getConnection(): Redis {
    return this.connectionPool[Math.floor(Math.random() * this.connectionPool.length)];
  }
}
```

### 2. 캐싱 전략

```typescript
class CachedDriver implements MatchMakerDriver {
  private cache = new Map<string, IRoomCache>();
  private cacheTimeout = 5000; // 5초
  
  async findOne(conditions: Partial<IRoomCache>): Promise<RoomCache> {
    const cacheKey = JSON.stringify(conditions);
    
    if (this.cache.has(cacheKey)) {
      return this.cache.get(cacheKey);
    }
    
    const result = await this.driver.findOne(conditions);
    this.cache.set(cacheKey, result);
    
    setTimeout(() => this.cache.delete(cacheKey), this.cacheTimeout);
    return result;
  }
}
```

### 3. 배치 처리

```typescript
class BatchedPresence implements Presence {
  private batchQueue: Array<{ method: string, args: any[] }> = [];
  private batchTimeout: NodeJS.Timeout;
  
  hset(key: string, field: string, value: string) {
    this.batchQueue.push({ method: 'hset', args: [key, field, value] });
    this.scheduleBatch();
  }
  
  private scheduleBatch() {
    if (this.batchTimeout) return;
    
    this.batchTimeout = setTimeout(() => {
      this.processBatch();
      this.batchTimeout = null;
    }, 10); // 10ms 배치 윈도우
  }
}
```

## 🎯 주요 설계 원칙 요약

### 1. 단일 책임 원칙 (SRP)
- **Presence**: 데이터 저장 및 통신만 담당
- **MatchMaker**: 매칭 로직만 담당  
- **Room**: 게임 로직만 담당

### 2. 의존성 역전 원칙 (DIP)
- 인터페이스에 의존, 구현체에 의존하지 않음
- 런타임 의존성 주입으로 유연성 확보

### 3. 개방-폐쇄 원칙 (OCP)
- 새로운 Presence/Driver 구현체 추가 가능
- 기존 코드 수정 없이 확장 가능

### 4. 인터페이스 분리 원칙 (ISP)
- 각 컴포넌트는 필요한 인터페이스만 의존
- 불필요한 의존성 제거

이러한 설계를 통해 Colyseus는 확장 가능하고 유지보수가 용이한 멀티플레이어 게임 서버 프레임워크를 제공합니다.

---
*이 문서는 Colyseus 0.16.x 버전을 기준으로 작성되었습니다.*