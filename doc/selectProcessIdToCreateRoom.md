# selectProcessIdToCreateRoom 함수 분석

## 📋 개요

`selectProcessIdToCreateRoom`은 Colyseus에서 **로드 밸런싱**을 담당하는 핵심 함수입니다. 새로운 Room을 생성할 때 어느 프로세스에서 생성할지 결정하여 서버 리소스를 효율적으로 분산시킵니다.

## 🏗️ 함수 정의 및 타입

### 타입 정의

```typescript
export type SelectProcessIdCallback = (roomName: string, clientOptions: ClientOptions) => Promise<string>;
```

### 전역 변수

```typescript
// MatchMaker.ts
export let selectProcessIdToCreateRoom: SelectProcessIdCallback;
```

## 🔧 기본 구현 (Default Implementation)

### setup() 함수에서 초기화

```typescript
// MatchMaker.ts - setup()
export async function setup(
  _presence?: Presence,
  _driver?: MatchMakerDriver,
  _publicAddress?: string,
  _selectProcessIdToCreateRoom?: SelectProcessIdCallback,
) {
  // ... 기타 초기화 코드

  /**
   * Define default `assignRoomToProcessId` method.
   * By default, return the process with least amount of rooms created
   */
  selectProcessIdToCreateRoom = _selectProcessIdToCreateRoom || async function () {
    return (await stats.fetchAll())
      .sort((p1, p2) => p1.roomCount > p2.roomCount ? 1 : -1)[0]?.processId || processId;
  };

  // ... 기타 초기화 코드
}
```

### 기본 로직 분석

```typescript
// 기본 구현의 상세 분석
async function defaultSelectProcessId() {
  // 1. 모든 프로세스 통계 조회
  const allStats = await stats.fetchAll();

  // 2. roomCount 기준 오름차순 정렬
  const sortedStats = allStats.sort((p1, p2) =>
    p1.roomCount > p2.roomCount ? 1 : -1
  );

  // 3. 가장 적은 Room을 가진 프로세스 선택
  const selectedProcess = sortedStats[0];

  // 4. 프로세스가 없으면 현재 프로세스 사용
  return selectedProcess?.processId || processId;
}
```

## 📊 통계 시스템 (Stats System)

### Stats 데이터 구조

```typescript
// Stats.ts
export type Stats = {
  roomCount: number;  // 프로세스가 관리하는 Room 수
  ccu: number;        // 동시 접속자 수 (Concurrent Users)
}

export let local: Stats = {
  roomCount: 0,
  ccu: 0,
};
```

### fetchAll() - 모든 프로세스 통계 조회

```typescript
// Stats.ts
export async function fetchAll() {
  const allStats: Array<Stats & { processId: string }> = [];

  // Redis에서 모든 프로세스 통계 조회
  const allProcesses = await presence.hgetall(getRoomCountKey()); // 'roomcount'

  for (let remoteProcessId in allProcesses) {
    if (remoteProcessId === processId) {
      // 현재 프로세스: 로컬 통계 사용
      allStats.push({
        processId,
        roomCount: local.roomCount,
        ccu: local.ccu
      });
    } else {
      // 원격 프로세스: Redis에서 파싱
      const [roomCount, ccu] = allProcesses[remoteProcessId].split(',').map(Number);
      allStats.push({
        processId: remoteProcessId,
        roomCount,
        ccu
      });
    }
  }

  return allStats;
}
```

### persist() - 통계 저장

```typescript
// Stats.ts
export function persist(forceNow: boolean = false) {
  if (state === MatchMakerState.SHUTTING_DOWN) {
    return;
  }

  const now = Date.now();

  // 1초에 한 번만 저장 (성능 최적화)
  if (forceNow || (now - lastPersisted > persistInterval)) {
    lastPersisted = now;

    // Redis에 "roomCount,ccu" 형태로 저장
    return presence.hset(
      getRoomCountKey(),        // 'roomcount'
      processId,
      `${local.roomCount},${local.ccu}`
    );
  } else {
    // 지연 저장
    clearTimeout(persistTimeout);
    persistTimeout = setTimeout(persist, persistInterval);
  }
}
```

## 🔄 동작 과정 분석

### 1. Room 생성 요청 시

```typescript
// MatchMaker.ts - createRoom()
export async function createRoom(roomName: string, clientOptions: ClientOptions): Promise<IRoomCache> {
  // 프로세스 선택
  const selectedProcessId = (state === MatchMakerState.READY)
    ? await selectProcessIdToCreateRoom(roomName, clientOptions)  // ← 여기서 호출
    : processId;

  let room: IRoomCache;

  if (selectedProcessId === processId) {
    // 로컬 프로세스에서 생성
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

  return room;
}
```

### 2. 통계 업데이트

```typescript
// MatchMaker.ts - handleCreateRoom()
export async function handleCreateRoom(roomName: string, clientOptions: ClientOptions): Promise<IRoomCache> {
  // ... Room 생성 로직

  // Room 수 증가
  stats.local.roomCount++;
  stats.persist();

  // ... 기타 로직
}

// MatchMaker.ts - onClientJoinRoom()
function onClientJoinRoom(room: Room, client: Client) {
  // CCU 증가
  stats.local.ccu++;
  stats.persist();
}

// MatchMaker.ts - onClientLeaveRoom()
function onClientLeaveRoom(room: Room, client: Client, willDispose: boolean) {
  // CCU 감소
  stats.local.ccu--;
  stats.persist();
}
```

### 3. Redis 데이터 구조

```
Redis Hash: 'roomcount'
├── process_1 → "5,23"    (roomCount=5, ccu=23)
├── process_2 → "3,15"    (roomCount=3, ccu=15)
├── process_3 → "7,31"    (roomCount=7, ccu=31)
└── process_4 → "2,8"     (roomCount=2, ccu=8)
```

## 🎯 커스텀 구현 예시

### 1. CCU 기반 로드 밸런싱

```typescript
const server = new Server({
  selectProcessIdToCreateRoom: async (roomName, clientOptions) => {
    const allStats = await stats.fetchAll();

    // CCU가 가장 적은 프로세스 선택
    return allStats
      .sort((p1, p2) => p1.ccu - p2.ccu)[0]?.processId || processId;
  }
});
```

### 2. 룸 타입별 분산

```typescript
const server = new Server({
  selectProcessIdToCreateRoom: async (roomName, clientOptions) => {
    const allStats = await stats.fetchAll();

    if (roomName === 'battle') {
      // 배틀 룸은 CPU 집약적이므로 roomCount 기준
      return allStats
        .sort((p1, p2) => p1.roomCount - p2.roomCount)[0]?.processId || processId;
    } else if (roomName === 'chat') {
      // 채팅 룸은 네트워크 집약적이므로 CCU 기준
      return allStats
        .sort((p1, p2) => p1.ccu - p2.ccu)[0]?.processId || processId;
    }

    // 기본: roomCount 기준
    return allStats
      .sort((p1, p2) => p1.roomCount - p2.roomCount)[0]?.processId || processId;
  }
});
```

### 3. 가중치 기반 분산

```typescript
const server = new Server({
  selectProcessIdToCreateRoom: async (roomName, clientOptions) => {
    const allStats = await stats.fetchAll();

    // 가중치 계산 (roomCount * 0.7 + ccu * 0.3)
    const processesWithWeight = allStats.map(stat => ({
      ...stat,
      weight: stat.roomCount * 0.7 + stat.ccu * 0.3
    }));

    // 가중치가 가장 낮은 프로세스 선택
    return processesWithWeight
      .sort((p1, p2) => p1.weight - p2.weight)[0]?.processId || processId;
  }
});
```

### 4. 지역 기반 분산

```typescript
const server = new Server({
  selectProcessIdToCreateRoom: async (roomName, clientOptions) => {
    const allStats = await stats.fetchAll();
    const userRegion = clientOptions.region || 'us-east';

    // 지역별 프로세스 매핑
    const regionProcesses = {
      'us-east': ['process_1', 'process_2'],
      'us-west': ['process_3', 'process_4'],
      'eu': ['process_5', 'process_6']
    };

    const availableProcesses = allStats.filter(stat =>
      regionProcesses[userRegion]?.includes(stat.processId)
    );

    if (availableProcesses.length === 0) {
      // 해당 지역에 프로세스가 없으면 기본 로직
      return allStats
        .sort((p1, p2) => p1.roomCount - p2.roomCount)[0]?.processId || processId;
    }

    // 해당 지역에서 roomCount가 가장 적은 프로세스
    return availableProcesses
      .sort((p1, p2) => p1.roomCount - p2.roomCount)[0].processId;
  }
});
```

### 5. 시간대별 분산

```typescript
const server = new Server({
  selectProcessIdToCreateRoom: async (roomName, clientOptions) => {
    const allStats = await stats.fetchAll();
    const hour = new Date().getHours();

    // 피크 시간대 (18-22시)에는 더 엄격한 분산
    if (hour >= 18 && hour <= 22) {
      // 피크 시간: CCU 기준으로 엄격하게 분산
      return allStats
        .sort((p1, p2) => p1.ccu - p2.ccu)[0]?.processId || processId;
    } else {
      // 일반 시간: roomCount 기준
      return allStats
        .sort((p1, p2) => p1.roomCount - p2.roomCount)[0]?.processId || processId;
    }
  }
});
```

## 🔍 헬스 체크 시스템

### 프로세스 상태 확인

```typescript
// MatchMaker.ts
export function healthCheckProcessId(processId: string) {
  return new Promise<boolean>(async (resolve, reject) => {
    try {
      await requestFromIPC<IRoomCache>(
        presence,
        getProcessChannel(processId),
        'healthcheck',
        []
      );

      resolve(true); // 응답 성공
    } catch (e) {
      // 응답 실패 - 프로세스 제외
      await stats.excludeProcess(processId);
      resolve(false);
    }
  });
}
```

### 자동 정리 시스템

```typescript
// Stats.ts
export function excludeProcess(_processId: string) {
  // Redis에서 해당 프로세스 통계 제거
  return presence.hdel(getRoomCountKey(), _processId);
}
```

## ⚡ 성능 최적화

### 1. 통계 캐싱

```typescript
let statsCache = null;
let lastCacheTime = 0;
const CACHE_DURATION = 5000; // 5초

const server = new Server({
  selectProcessIdToCreateRoom: async (roomName, clientOptions) => {
    const now = Date.now();

    // 캐시된 통계 사용 (5초 이내)
    if (statsCache && (now - lastCacheTime < CACHE_DURATION)) {
      return statsCache
        .sort((p1, p2) => p1.roomCount - p2.roomCount)[0]?.processId || processId;
    }

    // 새로운 통계 조회
    statsCache = await stats.fetchAll();
    lastCacheTime = now;

    return statsCache
      .sort((p1, p2) => p1.roomCount - p2.roomCount)[0]?.processId || processId;
  }
});
```

### 2. 배치 처리

```typescript
const server = new Server({
  selectProcessIdToCreateRoom: async (roomName, clientOptions) => {
    // 여러 요청을 배치로 처리
    if (!pendingRequests) {
      pendingRequests = [];

      // 다음 틱에서 배치 처리
      process.nextTick(async () => {
        const allStats = await stats.fetchAll();
        const selectedProcess = allStats
          .sort((p1, p2) => p1.roomCount - p2.roomCount)[0]?.processId || processId;

        // 모든 대기 중인 요청에 동일한 프로세스 반환
        pendingRequests.forEach(resolve => resolve(selectedProcess));
        pendingRequests = null;
      });
    }

    return new Promise(resolve => {
      pendingRequests.push(resolve);
    });
  }
});
```

## 🎯 핵심 특징

1. **로드 밸런싱**: 프로세스 간 Room 분산으로 부하 균등화
2. **실시간 통계**: Redis 기반 실시간 프로세스 상태 추적
3. **커스터마이징**: 비즈니스 로직에 맞는 분산 전략 구현 가능
4. **헬스 체크**: 비정상 프로세스 자동 감지 및 제외
5. **성능 최적화**: 통계 캐싱 및 배치 처리 지원
6. **확장성**: 프로세스 추가/제거 시 자동 적응
7. **안정성**: 프로세스 장애 시 자동 복구

## 🔧 모니터링 및 디버깅

### 통계 조회

```typescript
// 전체 프로세스 통계 조회
const allStats = await stats.fetchAll();
console.log('All processes:', allStats);

// 글로벌 CCU 조회
const globalCCU = await stats.getGlobalCCU();
console.log('Global CCU:', globalCCU);

// 로컬 통계 조회
console.log('Local stats:', stats.local);
```

### PM2 메트릭 연동

```typescript
// Stats.ts - PM2 메트릭 자동 등록
import('@pm2/io')
  .then((io) => {
    io.default.metric({
      id: 'app/stats/ccu',
      name: 'ccu',
      value: () => local.ccu
    });
    io.default.metric({
      id: 'app/stats/roomcount',
      name: 'roomcount',
      value: () => local.roomCount
    });
  })
  .catch(() => { });
```

---
*이 문서는 Colyseus selectProcessIdToCreateRoom 함수 분석을 바탕으로 작성되었습니다.*