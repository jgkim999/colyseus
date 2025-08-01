# Colyseus Room 파괴 과정 분석

## 📋 개요

Colyseus Room의 파괴는 여러 트리거에 의해 시작되며, 체계적인 정리 과정을 거쳐 완전히 제거됩니다. 이 문서는 Room이 파괴되는 모든 방법과 순서를 상세히 분석합니다.

## 🔥 Room 파괴 트리거

### 1. 자동 파괴 (Auto Dispose)

```typescript
// Room.ts
public autoDispose: boolean = true; // 기본값

protected _disposeIfEmpty() {
  const willDispose = (
    this.#_onLeaveConcurrent === 0 &&     // onLeave 진행 중 없음
    this.#_autoDispose &&                 // 자동 정리 활성화
    this._autoDisposeTimeout === undefined && // 타임아웃 없음
    this.clients.length === 0 &&          // 클라이언트 없음
    Object.keys(this.reservedSeats).length === 0 // 예약 좌석 없음
  );

  if (willDispose) {
    this._events.emit('dispose');
  }

  return willDispose;
}
```

**트리거 조건**:
- 모든 클라이언트가 퇴장
- 예약된 좌석 없음
- `onLeave` 처리 중이 아님
- `autoDispose = true`

### 2. 수동 파괴 (Manual Disconnect)

```typescript
// Room.ts
public disconnect(closeCode: number = Protocol.WS_CLOSE_CONSENTED): Promise<any> {
  if (this._internalState === RoomInternalState.DISPOSING) {
    return Promise.resolve(`disconnect() ignored: room (${this.roomId}) is already disposing.`);
  }

  this._internalState = RoomInternalState.DISPOSING;
  this.listing.remove();
  this.#_autoDispose = true;

  // 모든 클라이언트 강제 종료
  let numClients = this.clients.length;
  if (numClients > 0) {
    while (numClients--) {
      this._forciblyCloseClient(this.clients[numClients], closeCode);
    }
  } else {
    this._events.emit('dispose');
  }

  return delayedDisconnection;
}
```

### 3. 서버 종료 시 파괴

```typescript
// MatchMaker.ts
export async function gracefullyShutdown(): Promise<any> {
  state = MatchMakerState.SHUTTING_DOWN;

  // 모든 룸 잠금 및 정리
  await lockAndDisposeAll();

  // 모든 룸 연결 해제
  return Promise.all(disconnectAll(
    (isDevMode) ? Protocol.WS_CLOSE_DEVMODE_RESTART : undefined
  ));
}

async function lockAndDisposeAll(): Promise<any> {
  // 모든 로컬 룸 잠금 및 onBeforeShutdown 호출
  for (const roomId in rooms) {
    const room = rooms[roomId];
    room.lock();
    room.onBeforeShutdown();
  }
}
```

### 4. 자동 정리 타임아웃

```typescript
// Room.ts
protected resetAutoDisposeTimeout(timeoutInSeconds: number = 1) {
  clearTimeout(this._autoDisposeTimeout);

  if (!this.#_autoDispose) {
    return;
  }

  this._autoDisposeTimeout = setTimeout(() => {
    this._autoDisposeTimeout = undefined;
    this._disposeIfEmpty();
  }, timeoutInSeconds * 1000);
}
```

## 🔄 Room 파괴 순서

### Phase 1: 파괴 시작 (Dispose Trigger)

#### 1-1. 상태 변경

```typescript
// Room.ts - constructor에서 이벤트 바인딩
this._events.once('dispose', () => {
  this._dispose()
    .catch((e) => debugAndPrintError(`onDispose error: ${e}`))
    .finally(() => this._events.emit('disconnect'));
});
```

#### 1-2. 내부 상태 설정

```typescript
this._internalState = RoomInternalState.DISPOSING;
```

### Phase 2: 클라이언트 정리

#### 2-1. 클라이언트 강제 종료

```typescript
// Room.ts
protected _forciblyCloseClient(client: Client & ClientPrivate, closeCode: number) {
  // 메시지 수신 중단
  client.ref.removeAllListeners('message');

  // onLeave 중복 호출 방지
  client.ref.removeListener('close', client.ref['onleave']);

  // onLeave 완료 후 연결 종료
  this._onLeave(client, closeCode).then(() => client.leave(closeCode));
}
```

#### 2-2. onLeave 처리

```typescript
// Room.ts
protected async _onLeave(client: Client, code?: number): Promise<any> {
  client.state = ClientState.LEAVING;

  // 클라이언트 배열에서 제거
  if (!this.clients.delete(client)) {
    return; // 이미 제거된 경우
  }

  // 사용자 정의 onLeave 호출
  if (this.onLeave) {
    try {
      this.#_onLeaveConcurrent++;
      await this.onLeave(client, (code === Protocol.WS_CLOSE_CONSENTED));
    } finally {
      this.#_onLeaveConcurrent--;
    }
  }

  // 재연결 처리 또는 후처리
  if (this._reconnections[client.reconnectionToken]) {
    // 재연결 대기 중
  } else if (client.state !== ClientState.RECONNECTED) {
    await this._onAfterLeave(client);
  }
}
```

#### 2-3. 클라이언트 수 감소

```typescript
// Room.ts
protected async _decrementClientCount() {
  const willDispose = this._disposeIfEmpty();

  if (!willDispose && this._internalState !== RoomInternalState.DISPOSING) {
    // 룸 캐시 업데이트
    await this.listing.updateOne({
      $inc: { clients: -1 },
      $set: { locked: this.#_locked },
    });
  }

  return willDispose;
}
```

### Phase 3: 리소스 정리 (_dispose)

#### 3-1. 사용자 정의 onDispose 호출

```typescript
// Room.ts
protected async _dispose(): Promise<any> {
  this._internalState = RoomInternalState.DISPOSING;

  // 룸 캐시에서 제거
  this.listing.remove();

  // 사용자 정의 정리 로직
  let userReturnData;
  if (this.onDispose) {
    userReturnData = this.onDispose();
  }
}
```

#### 3-2. 인터벌 및 타이머 정리

```typescript
// Room.ts - _dispose() 계속
// 패치 인터벌 정리
if (this.#_patchInterval) {
  clearInterval(this.#_patchInterval);
  this.#_patchInterval = undefined;
}

// 시뮬레이션 인터벌 정리
if (this._simulationInterval) {
  clearInterval(this._simulationInterval);
  this._simulationInterval = undefined;
}

// 자동 정리 타임아웃 정리
if (this._autoDisposeTimeout) {
  clearInterval(this._autoDisposeTimeout);
  this._autoDisposeTimeout = undefined;
}
```

#### 3-3. Clock 정리

```typescript
// Room.ts - _dispose() 계속
// 모든 타이머 정리 및 클록 중지
this.clock.clear();
this.clock.stop();
```

### Phase 4: MatchMaker 정리 (disposeRoom)

#### 4-1. MatchMaker에서 Room 제거

```typescript
// MatchMaker.ts
async function disposeRoom(roomName: string, room: Room) {
  debugMatchMaking('disposing \'%s\' (%s) on processId \'%s\'',
    roomName, room.roomId, processId);

  // 룸 캐시 제거 (안전장치)
  room.listing.remove();

  // 통계 업데이트
  stats.local.roomCount--;
}
```

#### 4-2. 통계 및 캐시 정리

```typescript
// MatchMaker.ts - disposeRoom() 계속
if (state !== MatchMakerState.SHUTTING_DOWN) {
  stats.persist();

  // devMode 복원 목록에서 제거
  if (isDevMode) {
    await presence.hdel(getRoomRestoreListKey(), room.roomId);
  }
}
```

#### 4-3. 이벤트 및 참조 정리

```typescript
// MatchMaker.ts - disposeRoom() 계속
// 핸들러 이벤트 발생
handlers[roomName].emit('dispose', room);

// IPC 구독 해제
presence.unsubscribe(getRoomChannel(room.roomId));

// 전역 rooms 객체에서 제거
delete rooms[room.roomId];
```

### Phase 5: 최종 정리

#### 5-1. 연결 해제 이벤트

```typescript
// Room.ts - constructor의 이벤트 핸들러
this._events.once('dispose', () => {
  this._dispose()
    .finally(() => this._events.emit('disconnect')); // ← 최종 이벤트
});
```

#### 5-2. 프로세스 레벨 정리

```typescript
// MatchMaker.ts - handleCreateRoom()에서 바인딩된 이벤트
room._events.once('disconnect', () => {
  // 이벤트 리스너 정리
  room._events.removeAllListeners('lock');
  room._events.removeAllListeners('unlock');
  room._events.removeAllListeners('visibility-change');
  room._events.removeAllListeners('dispose');

  // 활성 룸이 없으면 이벤트 발생
  if (stats.local.roomCount <= 0) {
    events.emit('no-active-rooms');
  }
});
```

## 📊 파괴 과정 플로우차트

```
[Trigger Event]
       ↓
[_disposeIfEmpty() 또는 disconnect()]
       ↓
[_internalState = DISPOSING]
       ↓
[_events.emit('dispose')]
       ↓
[_dispose() 호출]
       ↓
┌─────────────────────────────┐
│ 1. listing.remove()         │
│ 2. onDispose() 호출         │
│ 3. 인터벌/타이머 정리       │
│ 4. clock.clear() & stop()   │
└─────────────────────────────┘
       ↓
[_events.emit('disconnect')]
       ↓
[disposeRoom() 호출 - MatchMaker]
       ↓
┌─────────────────────────────┐
│ 1. stats.local.roomCount--  │
│ 2. handlers[].emit('dispose')│
│ 3. presence.unsubscribe()   │
│ 4. delete rooms[roomId]     │
└─────────────────────────────┘
       ↓
[Room 완전 제거]
```

## 🛡️ 안전장치 및 예외 처리

### 중복 파괴 방지

```typescript
// Room.ts - disconnect()
if (this._internalState === RoomInternalState.DISPOSING) {
  return Promise.resolve(`disconnect() ignored: room (${this.roomId}) is already disposing.`);
}
```

### 비동기 onLeave 처리

```typescript
// Room.ts
#_onLeaveConcurrent: number = 0;

// _disposeIfEmpty()에서 체크
const willDispose = (
  this.#_onLeaveConcurrent === 0 && // onLeave 진행 중이 아님
  // ... 기타 조건들
);
```

### 예외 처리

```typescript
// Room.ts - constructor
this._events.once('dispose', () => {
  this._dispose()
    .catch((e) => debugAndPrintError(`onDispose error: ${e}`))
    .finally(() => this._events.emit('disconnect'));
});
```

## 🔧 개발자 제어 가능한 부분

### 1. autoDispose 설정

```typescript
class MyRoom extends Room {
  onCreate() {
    this.autoDispose = false; // 수동 관리
  }

  // 조건부 파괴
  checkGameEnd() {
    if (this.gameEnded) {
      this.disconnect();
    }
  }
}
```

### 2. onDispose 커스터마이징

```typescript
class MyRoom extends Room {
  async onDispose() {
    // 게임 결과 저장
    await this.saveGameResults();

    // 외부 서비스 정리
    await this.cleanupExternalResources();

    console.log(`Room ${this.roomId} disposed`);
  }
}
```

### 3. onBeforeShutdown 처리

```typescript
class MyRoom extends Room {
  onBeforeShutdown() {
    // 클라이언트에게 서버 종료 알림
    this.broadcast("server_shutdown", {
      message: "Server is restarting..."
    });

    // 기본 동작 (모든 클라이언트 연결 해제)
    super.onBeforeShutdown();
  }
}
```

### 4. 재연결 고려 파괴

```typescript
class MyRoom extends Room {
  async onLeave(client: Client, consented: boolean) {
    if (!consented) {
      // 비정상 종료 시 재연결 허용
      try {
        await this.allowReconnection(client, 30);
        return; // 파괴 방지
      } catch (e) {
        // 재연결 실패 시 정상 파괴 진행
      }
    }

    // 자발적 퇴장 시 정상 파괴 진행
  }
}
```

## ⚠️ 주의사항

### 1. 비동기 onDispose

- `onDispose`는 비동기 메서드 가능
- 완료될 때까지 대기 후 최종 정리

### 2. 메모리 누수 방지

- 모든 인터벌/타이머 자동 정리
- 이벤트 리스너 자동 해제
- 외부 참조는 `onDispose`에서 수동 정리

### 3. 재연결 중 파괴

- 재연결 대기 중인 클라이언트가 있으면 파괴 지연
- `allowReconnection` 타임아웃 후 자동 파괴

### 4. 프로세스 종료 시

- `onBeforeShutdown` 호출 후 강제 파괴
- devMode에서는 룸 상태 캐싱

## 🎯 핵심 특징

1. **단계적 파괴**: 클라이언트 → 리소스 → 캐시 → 참조 순서
2. **안전장치**: 중복 파괴 방지 및 상태 체크
3. **비동기 처리**: onLeave, onDispose 비동기 지원
4. **자동 정리**: 모든 타이머, 인터벌, 이벤트 자동 해제
5. **통계 관리**: 룸 수, CCU 자동 업데이트
6. **재연결 고려**: 재연결 대기 중 파괴 지연
7. **개발자 제어**: autoDispose, onDispose 커스터마이징

---
*이 문서는 Colyseus Room 파괴 과정 분석을 바탕으로 작성되었습니다.*