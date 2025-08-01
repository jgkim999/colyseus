# Colyseus Presence 시스템 분석

## 📋 개요

Presence는 Colyseus에서 **프로세스 간 통신(IPC)**과 **데이터 공유**를 담당하는 핵심 인터페이스입니다. 단일 프로세스에서는 LocalPresence를, 분산 환경에서는 RedisPresence를 사용하여 확장성을 제공합니다.

## 🏗️ Presence 인터페이스

### 핵심 개념

```typescript
/**
 * 다중 프로세스/머신에서 서버를 확장할 때 필요한 인터페이스
 * 프로세스 간 통신과 데이터 공유를 위한 목적
 * 특히 매치메이킹 과정에서 중요한 역할
 */
export interface Presence {
  // Pub/Sub 시스템
  subscribe(topic: string, callback: Function);
  unsubscribe(topic: string, callback?: Function);
  publish(topic: string, data: any);
  channels(pattern?: string): Promise<string[]>;

  // Key-Value 저장소
  exists(key: string): Promise<boolean>;
  set(key: string, value: string);
  setex(key: string, value: string, seconds: number);
  expire(key: string, seconds: number);
  get(key: string);
  del(key: string): void;

  // Set 연산
  sadd(key: string, value: any);
  smembers(key: string): Promise<string[]>;
  sismember(key: string, field: string);
  srem(key: string, value: any);
  scard(key: string);
  sinter(...keys: string[]): Promise<string[]>;

  // Hash 연산
  hset(key: string, field: string, value: string): Promise<boolean>;
  hincrby(key: string, field: string, value: number): Promise<number>;
  hget(key: string, field: string): Promise<string>;
  hgetall(key: string): Promise<{ [key: string]: string }>;
  hdel(key: string, field: string): Promise<boolean>;
  hlen(key: string): Promise<number>;

  // 카운터
  incr(key: string): Promise<number>;
  decr(key: string): Promise<number>;

  // List 연산
  llen(key: string): Promise<number>;
  rpush(key: string, ...values: string[]): Promise<number>;
  lpush(key: string, ...values: string[]): Promise<number>;
  rpop(key: string): Promise<string | null>;
  lpop(key: string): Promise<string | null>;
  brpop(...args: [...keys: string[], timeoutInSeconds: number]): Promise<[string, string] | null>;

  // 관리
  setMaxListeners(number: number): void;
  shutdown(): void;
}
```

## 🔧 LocalPresence 구현

### 단일 프로세스용 구현체

```typescript
export class LocalPresence implements Presence {
  public subscriptions = new EventEmitter();
  public data: {[roomName: string]: string[]} = {};      // Set 데이터
  public hash: {[roomName: string]: {[key: string]: string}} = {}; // Hash 데이터
  public keys: {[name: string]: string | number} = {};   // Key-Value 데이터
  private timeouts: {[name: string]: NodeJS.Timeout} = {}; // 만료 타이머
}
```

### 주요 특징

#### 1. 메모리 기반 저장

```typescript
// Set 연산 예시
public sadd(key: string, value: any) {
  if (!this.data[key]) {
    this.data[key] = [];
  }

  if (this.data[key].indexOf(value) === -1) {
    this.data[key].push(value);
  }
}

// Hash 연산 예시
public hset(key: string, field: string, value: string) {
  if (!this.hash[key]) { this.hash[key] = {}; }
  this.hash[key][field] = value;
  return Promise.resolve(true);
}
```

#### 2. EventEmitter 기반 Pub/Sub

```typescript
public subscribe(topic: string, callback: (...args: any[]) => void) {
  this.subscriptions.on(topic, callback);
  return this;
}

public publish(topic: string, data: any) {
  this.subscriptions.emit(topic, data);
  return this;
}
```

#### 3. 개발 모드 캐싱

```typescript
constructor() {
  // devMode에서 로컬 캐시 복원
  if (isDevMode && fs.existsSync(DEVMODE_CACHE_FILE_PATH)) {
    const cache = fs.readFileSync(DEVMODE_CACHE_FILE_PATH).toString('utf-8');
    const parsed = JSON.parse(cache);
    if (parsed.data) { this.data = parsed.data; }
    if (parsed.hash) { this.hash = parsed.hash; }
    if (parsed.keys) { this.keys = parsed.keys; }
  }
}

public shutdown() {
  if (isDevMode) {
    // 종료 시 캐시 저장
    const cache = JSON.stringify({
      data: this.data,
      hash: this.hash,
      keys: this.keys
    });
    fs.writeFileSync(DEVMODE_CACHE_FILE_PATH, cache);
  }
}
```

## 🌐 RedisPresence 구현

### 분산 환경용 Redis 기반 구현체

```typescript
export class RedisPresence implements Presence {
  protected sub: Redis | Cluster;  // 구독용 연결
  protected pub: Redis | Cluster;  // 발행용 연결
  protected subscriptions = new EventEmitter();
}
```

### 주요 특징

#### 1. Redis 클러스터 지원

```typescript
constructor(options?: number | string | RedisOptions | ClusterNode[], clusterOptions?: ClusterOptions) {
  if (Array.isArray(options)) {
    // 클러스터 모드
    this.sub = new Cluster(options, clusterOptions);
    this.pub = new Cluster(options, clusterOptions);
  } else {
    // 단일 인스턴스 모드
    this.sub = new Redis(options as RedisOptions);
    this.pub = new Redis(options as RedisOptions);
  }

  this.sub.setMaxListeners(0); // 무제한 리스너
}
```

#### 2. Pub/Sub 시스템

```typescript
public async subscribe(topic: string, callback: Callback) {
  this.subscriptions.addListener(topic, callback);

  // 첫 번째 구독 시 메시지 핸들러 등록
  if (this.sub.listeners('message').length === 0) {
    this.sub.on('message', this.handleSubscription);
  }

  await this.sub.subscribe(topic);
  return this;
}

public async publish(topic: string, data: any) {
  if (data === undefined) {
    data = false;
  }
  await this.pub.publish(topic, JSON.stringify(data));
}

protected handleSubscription = (channel, message) => {
  this.subscriptions.emit(channel, JSON.parse(message));
}
```

#### 3. Redis 네이티브 연산 활용

```typescript
// Hash 증가 연산 + 만료 시간 설정
public async hincrbyex(key: string, field: string, value: number, expireInSeconds: number) {
  return new Promise<number>((resolve, reject) => {
    this.pub.multi()
      .hincrby(key, field, value)
      .expire(key, expireInSeconds)
      .exec((err, results) => {
        if (err) return reject(err);
        resolve(results[0][1] as number);
      });
  });
}

// 블로킹 팝 연산
public async brpop(...args: [...keys: string[], timeoutInSeconds: number]): Promise<[string, string] | null> {
  return await this.pub.brpop.apply(this.pub, args);
}
```

## 🔗 Presence 사용처 분석

### 1. IPC (Inter-Process Communication)

#### requestFromIPC - 원격 프로세스 호출

```typescript
export async function requestFromIPC<T>(
  presence: Presence,
  publishToChannel: string,
  method: string,
  args: any[]
): Promise<T> {
  return new Promise<T>((resolve, reject) => {
    const requestId = generateId();
    const channel = `ipc:${requestId}`;

    // 응답 구독
    presence.subscribe(channel, (message) => {
      const [code, data] = message;
      if (code === IpcProtocol.SUCCESS) {
        resolve(data);
      } else {
        reject(new Error(data));
      }
      presence.unsubscribe(channel);
    });

    // 요청 발행
    presence.publish(publishToChannel, [method, requestId, args]);

    // 타임아웃 설정
    setTimeout(() => {
      presence.unsubscribe(channel);
      reject(new Error("ipc_timeout"));
    }, rejectionTimeout);
  });
}
```

#### subscribeIPC - 원격 요청 처리

```typescript
export async function subscribeIPC(
  presence: Presence,
  processId: string,
  channel: string,
  replyCallback: (method: string, args: any[]) => any
) {
  await presence.subscribe(channel, (message) => {
    const [method, requestId, args] = message;

    const reply = (code, data) => {
      presence.publish(`ipc:${requestId}`, [code, data]);
    };

    try {
      const response = replyCallback(method, args);

      if (response instanceof Promise) {
        response
          .then(result => reply(IpcProtocol.SUCCESS, result))
          .catch(err => reply(IpcProtocol.ERROR, err));
      } else {
        reply(IpcProtocol.SUCCESS, response);
      }
    } catch (e) {
      reply(IpcProtocol.ERROR, e.message);
    }
  });
}
```

### 2. MatchMaker에서의 활용

#### 프로세스 간 Room 생성

```typescript
// MatchMaker.ts
export async function createRoom(roomName: string, clientOptions: ClientOptions) {
  const selectedProcessId = await selectProcessIdToCreateRoom(roomName, clientOptions);

  if (selectedProcessId === processId) {
    // 로컬 생성
    room = await handleCreateRoom(roomName, clientOptions);
  } else {
    // 원격 프로세스에 생성 요청
    room = await requestFromIPC<IRoomCache>(
      presence,
      getProcessChannel(selectedProcessId),
      undefined,
      [roomName, clientOptions]
    );
  }
}
```

#### Room 채널 구독

```typescript
// MatchMaker.ts - handleCreateRoom()
await subscribeIPC(
  presence,
  processId,
  getRoomChannel(room.roomId), // `$${roomId}`
  (method, args) => {
    return (!args && typeof (room[method]) !== 'function')
      ? room[method]
      : room[method].apply(room, args);
  }
);
```

### 3. 통계 및 상태 관리

#### 프로세스 통계 저장

```typescript
// Stats.ts (추정)
await presence.hset('stats', processId, JSON.stringify({
  roomCount: local.roomCount,
  ccu: local.ccu,
  timestamp: Date.now()
}));
```

#### 동시성 제어

```typescript
// MatchMaker.ts - concurrentJoinOrCreateRoomLock()
const concurrency = await presence.hincrbyex(
  hkey,
  concurrencyKey,
  1, // increment by 1
  MAX_CONCURRENT_CREATE_ROOM_WAIT_TIME * 2 // expire time
) - 1;

if (concurrency > 0) {
  // 대기열에서 결과 기다리기
  const result = await presence.brpop(
    `l:${handler.name}:${concurrencyKey}`,
    MAX_CONCURRENT_CREATE_ROOM_WAIT_TIME
  );
}
```

### 4. Room 상태 캐싱

#### Room 목록 관리

```typescript
// RedisDriver.ts
const ROOMCACHES_KEY = 'roomcaches';

// Room 저장
await presence.hset(ROOMCACHES_KEY, roomId, JSON.stringify(roomData));

// Room 조회
const roomcache = await presence.hget(ROOMCACHES_KEY, roomId);

// Room 삭제
await presence.hdel(ROOMCACHES_KEY, roomId);
```

## 🔄 채널 명명 규칙

### IPC 채널

```typescript
// 프로세스 채널
function getProcessChannel(id: string = processId) {
  return `p:${id}`;
}

// Room 채널
function getRoomChannel(roomId: string) {
  return `$${roomId}`;
}

// 임시 응답 채널
const responseChannel = `ipc:${requestId}`;

// 동시성 제어 해시
function getConcurrencyHashKey(roomName: string) {
  return `ch:${roomName}`;
}

// 대기열 리스트
const queueKey = `l:${handler.name}:${concurrencyKey}`;
```

## 🚀 사용 예시

### 서버 설정

```typescript
import { Server } from '@colyseus/core';
import { RedisPresence } from '@colyseus/redis-presence';

// Redis Presence 사용
const server = new Server({
  presence: new RedisPresence({
    host: 'localhost',
    port: 6379,
    db: 0
  })
});

// Redis 클러스터 사용
const server = new Server({
  presence: new RedisPresence([
    { host: 'redis-1', port: 6379 },
    { host: 'redis-2', port: 6379 },
    { host: 'redis-3', port: 6379 }
  ])
});
```

### Room에서 Presence 활용

```typescript
class MyRoom extends Room {
  onCreate() {
    // 커스텀 데이터 저장
    this.presence.set(`room:${this.roomId}:created`, Date.now().toString());

    // 룸 통계 업데이트
    this.presence.hincrby('room_stats', 'total_created', 1);

    // 이벤트 발행
    this.presence.publish('room_created', {
      roomId: this.roomId,
      roomName: this.roomName,
      timestamp: Date.now()
    });
  }

  onDispose() {
    // 정리 작업
    this.presence.del(`room:${this.roomId}:created`);
    this.presence.hincrby('room_stats', 'total_disposed', 1);
  }
}
```

### 외부 서비스와 통합

```typescript
// 별도 모니터링 서비스
class MonitoringService {
  constructor(private presence: Presence) {
    // 룸 생성 이벤트 구독
    this.presence.subscribe('room_created', this.onRoomCreated.bind(this));
  }

  async onRoomCreated(data: any) {
    // 외부 모니터링 시스템에 전송
    await this.sendToMonitoring(data);
  }

  async getRoomStats() {
    return await this.presence.hgetall('room_stats');
  }
}
```

## ⚡ 성능 고려사항

### LocalPresence

- **장점**: 지연시간 없음, 설정 불필요
- **단점**: 단일 프로세스만 지원, 메모리 사용량 증가
- **적합한 경우**: 개발 환경, 소규모 서비스

### RedisPresence

- **장점**: 다중 프로세스 지원, 데이터 영속성, 확장성
- **단점**: 네트워크 지연, Redis 의존성, 설정 복잡성
- **적합한 경우**: 프로덕션 환경, 대규모 서비스

### 최적화 팁

```typescript
// 연결 풀링
const redisPresence = new RedisPresence({
  host: 'localhost',
  port: 6379,
  maxRetriesPerRequest: 3,
  retryDelayOnFailover: 100,
  lazyConnect: true
});

// 배치 연산 활용
const multi = presence.pub.multi();
multi.hset('key1', 'field1', 'value1');
multi.hset('key2', 'field2', 'value2');
await multi.exec();
```

## 🎯 핵심 특징

1. **추상화**: 단일/분산 환경을 동일한 인터페이스로 처리
2. **확장성**: LocalPresence → RedisPresence 쉬운 전환
3. **IPC 지원**: 프로세스 간 투명한 통신
4. **Redis 호환**: Redis의 모든 주요 데이터 타입 지원
5. **개발 편의성**: devMode에서 자동 캐싱
6. **성능**: 비동기 처리 및 배치 연산 지원
7. **안정성**: 타임아웃, 에러 처리, 재시도 로직

---
*이 문서는 Colyseus Presence 시스템 분석을 바탕으로 작성되었습니다.*