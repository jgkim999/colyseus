# Colyseus Room 클래스 분석

## 📋 개요

`Room` 클래스는 Colyseus 프레임워크의 핵심 구성 요소로, 게임 세션을 구현하고 클라이언트 그룹 간의 통신 채널 역할을 합니다.

## 🏗️ 클래스 구조

### 제네릭 타입 매개변수

```typescript
abstract class Room<State extends object = any, Metadata = any, UserData = any, AuthData = any>
```

- `State`: 룸 상태 객체 타입
- `Metadata`: 룸 메타데이터 타입
- `UserData`: 클라이언트 사용자 데이터 타입
- `AuthData`: 인증 데이터 타입

## 🔧 주요 속성

### 상태 관리

```typescript
public state: State;                    // 룸 상태
public listing: RoomCache<Metadata>;    // 룸 캐시 정보
public clients: ClientArray<UserData, AuthData>; // 연결된 클라이언트 배열
```

### 설정 속성

```typescript
public maxClients: number = Infinity;   // 최대 클라이언트 수
public autoDispose: boolean = true;     // 자동 정리 여부
public patchRate: number = 50;          // 패치 전송 주기 (ms)
public clock: Clock;                    // 타이밍 이벤트 관리
```

### 내부 상태

```typescript
enum RoomInternalState {
  CREATING = 0,   // 생성 중
  CREATED = 1,    // 생성 완료
  DISPOSING = 2   // 정리 중
}
```

## 🔄 생명주기 메서드

### 1. onCreate()

```typescript
public onCreate?(options: any): void | Promise<any>;
```

- 룸 생성 시 호출
- 초기 상태 설정
- 게임 로직 초기화

### 2. onAuth()

```typescript
public onAuth(client: Client, options: any, context: AuthContext): any | Promise<any>
```

- 클라이언트 인증 처리
- 기본값: `true` 반환 (모든 클라이언트 허용)

### 3. onJoin()

```typescript
public onJoin?(client: Client, options?: any, auth?: AuthData): void | Promise<any>;
```

- 클라이언트 입장 시 호출
- 클라이언트별 초기화 작업

### 4. onLeave()

```typescript
public onLeave?(client: Client, consented?: boolean): void | Promise<any>;
```

- 클라이언트 퇴장 시 호출
- `consented`: 자발적 퇴장 여부

### 5. onDispose()

```typescript
public onDispose?(): void | Promise<any>;
```

- 룸 정리 시 호출
- 리소스 해제 작업

## 📨 메시지 처리

### 메시지 핸들러 등록

```typescript
public onMessage<T = any>(
  messageType: string | number,
  callback: (client: Client, message: T) => void,
  validate?: (message: unknown) => T
)
```

**사용 예시**:

```typescript
this.onMessage("move", (client, message) => {
  // 이동 메시지 처리
});

// 와일드카드 핸들러
this.onMessage("*", (client, type, message) => {
  // 모든 메시지 처리
});
```

### 메시지 전송

```typescript
// 브로드캐스트
this.broadcast("update", data);

// 특정 클라이언트 제외
this.broadcast("update", data, { except: client });

// 바이트 데이터 전송
this.broadcastBytes("binary_data", uint8Array);
```

## 🔒 룸 잠금 및 관리

### 자동 잠금

- `maxClients` 도달 시 자동 잠금
- 클라이언트 퇴장 시 자동 해제

### 수동 잠금

```typescript
await this.lock();    // 룸 잠금
await this.unlock();  // 룸 잠금 해제
```

### 프라이빗 룸

```typescript
await this.setPrivate(true);  // 프라이빗 설정
await this.setPrivate(false); // 퍼블릭 설정
```

## 🔄 상태 동기화

### 패치 시스템

```typescript
// 패치 주기 설정 (기본: 50ms, 20fps)
this.patchRate = 16; // 60fps

// 수동 패치 전송
this.broadcastPatch();

// 패치 전 콜백
public onBeforePatch?(state: State): void | Promise<any>;
```

### 직렬화 설정

```typescript
// Schema 직렬화 (자동 감지)
this.state = new MyState();

// 커스텀 직렬화
this.setSerializer(new CustomSerializer());
```

## 🔌 재연결 시스템

### 재연결 허용

```typescript
public allowReconnection(client: Client, seconds: number | "manual"): Deferred<Client>
```

**사용 예시**:

```typescript
async onLeave(client: Client, consented: boolean) {
  if (!consented) {
    // 30초 재연결 허용
    const reconnection = this.allowReconnection(client, 30);

    try {
      const newClient = await reconnection;
      console.log("클라이언트 재연결 성공");
    } catch (e) {
      console.log("재연결 실패 또는 타임아웃");
    }
  }
}
```

### 좌석 예약 시스템

```typescript
// 좌석 예약 시간 설정 (기본: 15초)
this.setSeatReservationTime(30);

// 예약된 좌석 확인
this.hasReservedSeat(sessionId, reconnectionToken);
```

## ⏰ 시뮬레이션 루프

### 게임 루프 설정

```typescript
this.setSimulationInterval((deltaTime) => {
  // 게임 로직 업데이트
  this.updateGameLogic(deltaTime);
}, 16); // 60fps
```

### 타이머 사용

```typescript
// 일회성 타이머
this.clock.setTimeout(() => {
  // 실행할 코드
}, 1000);

// 반복 타이머
this.clock.setInterval(() => {
  // 반복 실행할 코드
}, 5000);
```

## 🛡️ 예외 처리

### 전역 예외 핸들러

```typescript
public onUncaughtException(
  error: RoomException,
  methodName: string
): void {
  console.error(`${methodName}에서 오류 발생:`, error);
  // 오류 처리 로직
}
```

**자동 래핑되는 메서드**:

- `onMessage`, `onAuth`, `onJoin`, `onLeave`, `onCreate`, `onDispose`
- `clock.setTimeout`, `clock.setInterval`
- `setSimulationInterval`

## 📊 메타데이터 관리

### 메타데이터 설정

```typescript
await this.setMetadata({
  gameMode: "battle",
  mapName: "desert",
  difficulty: "hard"
});

// 부분 업데이트
await this.setMetadata({ difficulty: "normal" });
```

## 🔧 내부 동작 원리

### 클라이언트 연결 과정

1. **좌석 예약**: `_reserveSeat()` 호출
2. **인증**: `onAuth()` 실행
3. **입장**: `onJoin()` 실행
4. **상태 전송**: 전체 상태 전송
5. **메시지 바인딩**: 메시지 핸들러 연결

### 상태 동기화 과정

1. **패치 생성**: `_serializer.applyPatches()` 호출
2. **변경 감지**: 상태 변경 사항 추출
3. **전송**: 모든 클라이언트에게 패치 전송
4. **큐 처리**: `afterNextPatch` 메시지 처리

### 자동 정리 시스템

```typescript
private _disposeIfEmpty(): boolean {
  return (
    this.#_onLeaveConcurrent === 0 &&     // onLeave 진행 중 없음
    this.#_autoDispose &&                 // 자동 정리 활성화
    this._autoDisposeTimeout === undefined && // 타임아웃 없음
    this.clients.length === 0 &&          // 클라이언트 없음
    Object.keys(this.reservedSeats).length === 0 // 예약 좌석 없음
  );
}
```

## 🎯 실제 사용 예시

### 기본 룸 구현

```typescript
import { Room, Client } from '@colyseus/core';

class GameRoom extends Room {
  onCreate(options: any) {
    this.setState(new GameState());
    this.maxClients = 4;

    this.onMessage("move", (client, message) => {
      // 플레이어 이동 처리
      this.state.movePlayer(client.sessionId, message);
    });
  }

  onJoin(client: Client, options: any) {
    console.log(`${client.sessionId} 입장`);
    this.state.addPlayer(client.sessionId);
  }

  onLeave(client: Client, consented: boolean) {
    console.log(`${client.sessionId} 퇴장`);
    this.state.removePlayer(client.sessionId);
  }

  onDispose() {
    console.log("룸 정리");
  }
}
```

### 고급 기능 활용

```typescript
class AdvancedRoom extends Room {
  onCreate() {
    // 60fps 시뮬레이션
    this.setSimulationInterval(this.update.bind(this), 16);

    // 30fps 패치 전송
    this.patchRate = 33;

    // 재연결 허용 시간 설정
    this.setSeatReservationTime(30);
  }

  update(deltaTime: number) {
    // 게임 로직 업데이트
    this.state.update(deltaTime);
  }

  async onLeave(client: Client, consented: boolean) {
    if (!consented) {
      // 비정상 종료 시 재연결 허용
      try {
        await this.allowReconnection(client, 60);
      } catch (e) {
        // 재연결 실패 처리
      }
    }
  }
}
```

## 🔍 주요 특징

1. **타입 안전성**: TypeScript 제네릭으로 타입 안전성 보장
2. **자동 상태 동기화**: 상태 변경 자동 감지 및 전송
3. **유연한 메시지 시스템**: 타입별 메시지 핸들러 지원
4. **재연결 지원**: 네트워크 끊김 시 자동 재연결
5. **성능 최적화**: 패치 기반 델타 압축 전송
6. **예외 처리**: 전역 예외 핸들러로 안정성 향상
7. **개발 편의성**: 개발 모드 지원 및 디버깅 기능

---
*이 문서는 @colyseus/core Room 클래스 분석을 바탕으로 작성되었습니다.*