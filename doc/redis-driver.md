# @colyseus/redis-driver 동작 분석

## 📋 개요

`@colyseus/redis-driver`는 Colyseus 프레임워크에서 Redis를 사용한 매치메이킹 드라이버입니다. 룸 캐시 데이터를 Redis에 저장하고 관리하여 분산 환경에서의 매치메이킹을 지원합니다.

## 🏗️ 아키텍처

### 핵심 클래스

```
RedisDriver (MatchMakerDriver 구현)
├── RoomData (RoomCache 구현)
├── Query (쿼리 처리)
└── Redis/Cluster 클라이언트
```

## 🔧 RedisDriver 클래스

### 주요 기능

- Redis 단일 인스턴스 또는 클러스터 지원
- 룸 캐시 데이터 저장/조회/삭제
- 동시 요청 최적화
- 프로세스 정리 기능

### 생성자

```typescript
constructor(options?: number | string | RedisOptions | ClusterNode[], clusterOptions?: ClusterOptions)
```

**동작 방식**:

- `Array.isArray(options)` 체크로 클러스터/단일 인스턴스 구분
- 클러스터: `new Cluster(options, clusterOptions)`
- 단일: `new Redis(options as RedisOptions)`

### 핵심 메서드

#### 1. createInstance()

```typescript
public createInstance(initialValues: Partial<IRoomCache> = {}) {
  return new RoomData(initialValues, this._client);
}
```

- 새로운 RoomData 인스턴스 생성
- Redis 클라이언트 참조 전달

#### 2. has()

```typescript
public async has(roomId: string) {
  return await this._client.hexists(ROOMCACHES_KEY, roomId) === 1;
}
```

- Redis HEXISTS 명령어 사용
- `roomcaches` 해시에서 roomId 존재 여부 확인

#### 3. query()

```typescript
public async query(conditions: Partial<IRoomCache>, sortOptions?: SortOptions) {
  const rooms = await this.getRooms();
  return rooms.filter((room) => {
    // 조건 필터링 로직
  });
}
```

- 모든 룸 데이터 조회 후 메모리에서 필터링
- 조건별 매칭 검사

#### 4. findOne()

```typescript
public findOne(conditions: Partial<IRoomCache>, sortOptions?: SortOptions): Promise<RoomData>
```

**동작 분기**:

1. **roomId 조건**: 직접 Redis HGET 사용
2. **기타 조건**: Query 클래스로 필터링

#### 5. getRooms() - 최적화 핵심

```typescript
private _concurrentRoomCacheRequest?: Promise<Record<string, string>>;
private _roomCacheRequestByName: {[roomName: string]: Promise<RoomData[]>} = {};
```

**동시 요청 최적화**:

- `_concurrentRoomCacheRequest`: 전체 룸 캐시 요청 공유
- `_roomCacheRequestByName`: 룸 이름별 요청 캐싱
- 동일한 요청이 진행 중이면 기존 Promise 재사용

**미세 최적화**:

```typescript
if (roomName !== undefined) {
  const roomNameField = `"name":"${roomName}"`;
  roomcaches = roomcaches.filter(([, roomcache]) => roomcache.includes(roomNameField));
}
```

- JSON 파싱 전에 문자열 검색으로 필터링

#### 6. cleanup()

```typescript
public async cleanup(processId: string) {
  const cachedRooms = await this.query({ processId });
  const itemsPerCommand = 500;

  // 500개씩 배치 삭제
  for (let i = 0; i < cachedRooms.length; i += itemsPerCommand) {
    await this._client.hdel(ROOMCACHES_KEY, ...cachedRooms.slice(i, i + itemsPerCommand).map((room) => room.roomId));
  }
}
```

- 프로세스 종료 시 해당 프로세스의 룸들 정리
- 배치 처리로 성능 최적화

## 💾 RoomData 클래스

### 데이터 구조

```typescript
public clients: number = 0;
public locked: boolean = false;
public private: boolean = false;
public maxClients: number = Infinity;
public metadata: any;
public name: string;
public publicAddress: string;
public processId: string;
public roomId: string;
public createdAt: Date;
public unlisted: boolean = false;
```

### 핵심 메서드

#### 1. save()

```typescript
public async save() {
  if (this.#removed) { return; }

  if (this.roomId) {
    const toJSON = this.toJSON;
    this.toJSON = undefined;

    const roomcache = JSON.stringify(this);
    this.toJSON = toJSON;

    await this.hset('roomcaches', this.roomId, roomcache);
  }
}
```

**동작 방식**:

- `toJSON` 메서드 임시 제거로 전체 객체 직렬화
- Redis HSET으로 `roomcaches` 해시에 저장

#### 2. updateOne()

```typescript
public updateOne(operations: any) {
  if (operations.$set) {
    // 필드 설정
  }

  if (operations.$inc) {
    // 필드 증감
  }

  return this.save();
}
```

- MongoDB 스타일 업데이트 연산자 지원
- `$set`: 값 설정
- `$inc`: 값 증감

#### 3. remove()

```typescript
public remove() {
  if (this.roomId) {
    this.#removed = true;
    return this.hdel('roomcaches', this.roomId);
  }
}
```

- 삭제 플래그 설정
- Redis HDEL로 제거

## 🔍 Query 클래스

### 정렬 기능

```typescript
public sort(options: SortOptions) {
  this.order.clear();

  const fields = Object.entries(options);

  for (const [field, direction] of fields) {
    if (direction === 1 || direction === 'asc' || direction === 'ascending') {
      this.order.set(field, 1);
    } else {
      this.order.set(field, -1);
    }
  }

  return this;
}
```

### Promise 체이닝

```typescript
public then(resolve, reject) {
  return this.rooms.then(rooms => {
    if (this.order.size) {
      rooms.sort((room1, room2) => {
        // 다중 필드 정렬 로직
      });
    }

    // 조건 필터링 후 첫 번째 결과 반환
    return resolve(rooms.find((room) => {
      // 조건 매칭 로직
    }));
  })
}
```

## 📊 Redis 데이터 구조

### 키 구조

```text
roomcaches (Hash)
├── roomId1 → JSON 문자열
├── roomId2 → JSON 문자열
└── roomIdN → JSON 문자열
```

### JSON 데이터 예시

```json
{
  "clients": 2,
  "createdAt": "2024-01-01T00:00:00.000Z",
  "maxClients": 10,
  "metadata": {},
  "name": "game_room",
  "publicAddress": "localhost:2567",
  "processId": "process_123",
  "roomId": "room_456"
}
```

## ⚡ 성능 최적화

### 1. 동시 요청 최적화

- 동일한 Redis 요청 중복 방지
- Promise 재사용으로 네트워크 호출 최소화

### 2. 배치 처리

- cleanup 시 500개씩 배치 삭제
- 대량 데이터 처리 시 성능 향상

### 3. 문자열 필터링

- JSON 파싱 전 문자열 검색
- CPU 사용량 감소

### 4. 메모리 관리

- 요청 완료 후 캐시 정리
- 메모리 누수 방지

## 🧪 테스트 케이스

### 동시 쿼리 테스트

```typescript
it("should allow concurrent queries to multiple room names", async () => {
  // 여러 룸 생성
  for (let i=0; i<10; i++) {
    await driver.createInstance({ name: "one", roomId: "x" + i }).save();
  }

  // 동시 요청
  const req1 = driver.findOne({ name: "one" });
  const req2 = driver.findOne({ name: "two" });

  // 동일한 Promise 재사용 확인
  assert.strictEqual(concurrent, driver['_concurrentRoomCacheRequest']);
});
```

## 🚀 사용 예시

### 기본 설정

```typescript
import { RedisDriver } from '@colyseus/redis-driver';

const driver = new RedisDriver({
  host: 'localhost',
  port: 6379,
  db: 0
});
```

### 클러스터 설정

```typescript
const driver = new RedisDriver([
  { host: 'redis-1', port: 6379 },
  { host: 'redis-2', port: 6379 },
  { host: 'redis-3', port: 6379 }
]);
```

### 서버 통합

```typescript
import { Server } from '@colyseus/core';

const server = new Server({
  driver: new RedisDriver()
});
```

## 🔧 주요 특징

1. **확장성**: Redis 클러스터 지원으로 수평 확장 가능
2. **성능**: 동시 요청 최적화 및 배치 처리
3. **안정성**: 프로세스 정리 및 에러 처리
4. **호환성**: MongoDB 스타일 쿼리 지원
5. **효율성**: 메모리 및 네트워크 사용량 최적화

---
*이 문서는 @colyseus/redis-driver 0.16.1 버전 기준으로 작성되었습니다.*
