# Room 업데이트 시스템 분석: setSimulationInterval과 clock.setTimeout

## 📋 개요

Colyseus Room 클래스의 업데이트 시스템은 두 가지 주요 메커니즘으로 구성됩니다:

1. **setSimulationInterval**: 게임 루프 시뮬레이션
2. **clock.setTimeout/setInterval**: 타이밍 이벤트 관리

## ⚙️ setSimulationInterval 동작 원리

### 기본 구조

```typescript
public setSimulationInterval(onTickCallback?: SimulationCallback, delay: number = DEFAULT_SIMULATION_INTERVAL): void {
  // 기존 인터벌 정리
  if (this._simulationInterval) {
    clearInterval(this._simulationInterval);
  }

  if (onTickCallback) {
    // 예외 처리 래핑
    if (this.onUncaughtException !== undefined) {
      onTickCallback = wrapTryCatch(onTickCallback, this.onUncaughtException.bind(this), SimulationIntervalException, 'setSimulationInterval');
    }

    // 시뮬레이션 루프 시작
    this._simulationInterval = setInterval(() => {
      this.clock.tick();                    // 시간 업데이트
      onTickCallback(this.clock.deltaTime); // 콜백 실행
    }, delay);
  }
}
```

### 핵심 동작 과정

#### 1. 인터벌 관리

```typescript
private _simulationInterval: NodeJS.Timeout;
```

- **단일 인터벌**: 한 번에 하나의 시뮬레이션 인터벌만 활성화
- **자동 정리**: 새로운 인터벌 설정 시 기존 인터벌 자동 해제
- **룸 정리 시 해제**: `_dispose()` 메서드에서 자동 정리

#### 2. Clock 시스템 통합

```typescript
this._simulationInterval = setInterval(() => {
  this.clock.tick();                    // ← 시간 업데이트
  onTickCallback(this.clock.deltaTime); // ← 델타 타임 전달
}, delay);
```

**Clock.tick() 역할**:

- 현재 시간 측정
- 이전 틱과의 시간 차이 계산 (deltaTime)
- 내부 타이머들 업데이트

#### 3. 예외 처리 시스템

```typescript
if (this.onUncaughtException !== undefined) {
  onTickCallback = wrapTryCatch(onTickCallback, this.onUncaughtException.bind(this), SimulationIntervalException, 'setSimulationInterval');
}
```

**wrapTryCatch 동작**:

- 콜백 함수를 try-catch로 래핑
- 예외 발생 시 `onUncaughtException` 호출
- 시뮬레이션 중단 방지

### 상태 동기화와의 연관성

#### broadcastPatch()와의 관계

```typescript
public broadcastPatch() {
  if (this.onBeforePatch) {
    this.onBeforePatch(this.state);
  }

  // 시뮬레이션 인터벌이 없으면 수동으로 tick
  if (!this._simulationInterval) {
    this.clock.tick();
  }

  if (!this.state) {
    return false;
  }

  const hasChanges = this._serializer.applyPatches(this.clients, this.state);
  this._dequeueAfterPatchMessages();
  return hasChanges;
}
```

**동작 원리**:

- **시뮬레이션 있음**: `setSimulationInterval`에서 `clock.tick()` 호출
- **시뮬레이션 없음**: `broadcastPatch()`에서 `clock.tick()` 호출
- **중복 방지**: 시뮬레이션이 활성화되면 패치에서 tick 생략

## 🕐 Clock 시스템 동작 원리

### Clock 초기화 및 시작

```typescript
public clock: Clock = new Clock();

protected __init() {
  // ... 기타 초기화 코드
  this.clock.start(); // ← 클록 시작
}
```

### Clock의 생명주기 관리

```typescript
protected async _dispose(): Promise<any> {
  // ... 기타 정리 코드

  // 모든 타이머 정리 및 클록 중지
  this.clock.clear();
  this.clock.stop();

  return await (userReturnData || Promise.resolve());
}
```

### setTimeout/setInterval 래핑

#### 예외 처리 래핑 시스템

```typescript
#registerUncaughtExceptionHandlers() {
  const onUncaughtException = this.onUncaughtException.bind(this);

  // setTimeout 래핑
  const originalSetTimeout = this.clock.setTimeout;
  this.clock.setTimeout = (cb, timeout, ...args) => {
    return originalSetTimeout.call(
      this.clock,
      wrapTryCatch(cb, onUncaughtException, TimedEventException, 'setTimeout'),
      timeout,
      ...args
    );
  };

  // setInterval 래핑
  const originalSetInterval = this.clock.setInterval;
  this.clock.setInterval = (cb, timeout, ...args) => {
    return originalSetInterval.call(
      this.clock,
      wrapTryCatch(cb, onUncaughtException, TimedEventException, 'setInterval'),
      timeout,
      ...args
    );
  };
}
```

**래핑 과정**:

1. **원본 메서드 저장**: `originalSetTimeout`, `originalSetInterval`
2. **콜백 래핑**: `wrapTryCatch`로 예외 처리 추가
3. **메서드 교체**: 래핑된 버전으로 교체

## 🔄 업데이트 루프 패턴

### 일반적인 게임 루프 구현

```typescript
class GameRoom extends Room {
  onCreate() {
    this.setState(new GameState());

    // 60fps 시뮬레이션 루프
    this.setSimulationInterval(this.update.bind(this), 16);

    // 20fps 상태 동기화
    this.patchRate = 50;
  }

  update(deltaTime: number) {
    // 물리 시뮬레이션
    this.state.physics.update(deltaTime);

    // 게임 로직 업데이트
    this.state.gameLogic.update(deltaTime);

    // AI 업데이트
    this.state.ai.update(deltaTime);
  }
}
```

### 타이머 기반 이벤트 처리

```typescript
class EventRoom extends Room {
  onCreate() {
    // 5초 후 이벤트 시작
    this.clock.setTimeout(() => {
      this.startEvent();
    }, 5000);

    // 30초마다 보상 지급
    this.clock.setInterval(() => {
      this.giveRewards();
    }, 30000);
  }

  startEvent() {
    this.broadcast("event_start", { type: "boss_battle" });
  }

  giveRewards() {
    this.clients.forEach(client => {
      // 보상 지급 로직
    });
  }
}
```

## ⚡ 성능 최적화 전략

### 1. 적응형 업데이트 주기

```typescript
class OptimizedRoom extends Room {
  private activeClients = 0;

  onCreate() {
    this.adaptiveUpdateRate();
  }

  onJoin(client: Client) {
    this.activeClients++;
    this.adaptiveUpdateRate();
  }

  onLeave(client: Client) {
    this.activeClients--;
    this.adaptiveUpdateRate();
  }

  private adaptiveUpdateRate() {
    if (this.activeClients <= 2) {
      // 적은 클라이언트: 30fps
      this.setSimulationInterval(this.update.bind(this), 33);
    } else if (this.activeClients <= 10) {
      // 중간 클라이언트: 20fps
      this.setSimulationInterval(this.update.bind(this), 50);
    } else {
      // 많은 클라이언트: 10fps
      this.setSimulationInterval(this.update.bind(this), 100);
    }
  }
}
```

### 2. 조건부 업데이트

```typescript
class ConditionalRoom extends Room {
  private needsUpdate = false;

  update(deltaTime: number) {
    // 변경사항이 있을 때만 업데이트
    if (this.needsUpdate) {
      this.state.update(deltaTime);
      this.needsUpdate = false;
    }
  }

  onMessage(client: Client, message: any) {
    // 메시지 처리 후 업데이트 플래그 설정
    this.handleMessage(client, message);
    this.needsUpdate = true;
  }
}
```

## 🛡️ 예외 처리 및 안정성

### 예외 처리 시스템

```typescript
class SafeRoom extends Room {
  onUncaughtException(error: RoomException, methodName: string) {
    console.error(`[${this.roomId}] ${methodName}에서 오류:`, error);

    // 심각한 오류 시 룸 정리
    if (error.critical) {
      this.disconnect();
    }
  }

  update(deltaTime: number) {
    try {
      // 안전하지 않은 코드
      this.riskyOperation();
    } catch (error) {
      // 로컬 예외 처리
      console.warn("업데이트 중 경고:", error);
    }
  }
}
```

### 메모리 누수 방지

```typescript
class MemorySafeRoom extends Room {
  private timers: NodeJS.Timeout[] = [];

  onCreate() {
    // 타이머 추적
    const timer = this.clock.setTimeout(() => {
      this.doSomething();
    }, 5000);

    this.timers.push(timer);
  }

  onDispose() {
    // 수동 타이머 정리 (필요한 경우)
    this.timers.forEach(timer => clearTimeout(timer));
    this.timers = [];
  }
}
```

## 📊 시간 관리 패턴

### 델타 타임 활용

```typescript
class PhysicsRoom extends Room {
  update(deltaTime: number) {
    // 프레임 독립적 이동
    this.state.players.forEach(player => {
      player.x += player.velocityX * deltaTime / 1000; // 초당 픽셀
      player.y += player.velocityY * deltaTime / 1000;
    });

    // 시간 기반 쿨다운
    this.state.abilities.forEach(ability => {
      ability.cooldown = Math.max(0, ability.cooldown - deltaTime);
    });
  }
}
```

### 경과 시간 추적

```typescript
class TimedRoom extends Room {
  private gameStartTime: number;

  onCreate() {
    this.gameStartTime = this.clock.elapsedTime;

    this.setSimulationInterval(this.update.bind(this), 16);
  }

  update(deltaTime: number) {
    const gameTime = this.clock.elapsedTime - this.gameStartTime;

    // 게임 시간 기반 로직
    if (gameTime > 300000) { // 5분 후
      this.endGame();
    }
  }
}
```

## 🔧 디버깅 및 모니터링

### 성능 모니터링

```typescript
class MonitoredRoom extends Room {
  private updateTimes: number[] = [];

  update(deltaTime: number) {
    const startTime = Date.now();

    // 게임 로직 실행
    this.gameLogic(deltaTime);

    const endTime = Date.now();
    const updateTime = endTime - startTime;

    // 성능 추적
    this.updateTimes.push(updateTime);
    if (this.updateTimes.length > 100) {
      this.updateTimes.shift();
    }

    // 평균 업데이트 시간 계산
    const avgTime = this.updateTimes.reduce((a, b) => a + b, 0) / this.updateTimes.length;

    if (avgTime > 10) { // 10ms 초과 시 경고
      console.warn(`업데이트 시간 초과: ${avgTime}ms`);
    }
  }
}
```

## 🎯 주요 특징 요약

### setSimulationInterval

1. **단일 인터벌**: 한 번에 하나만 활성화
2. **자동 정리**: 새 설정 시 기존 인터벌 해제
3. **예외 안전**: wrapTryCatch로 예외 처리
4. **Clock 통합**: tick()과 deltaTime 자동 관리

### clock.setTimeout/setInterval

1. **예외 래핑**: onUncaughtException과 통합
2. **자동 정리**: 룸 dispose 시 모든 타이머 해제
3. **원본 보존**: 기존 Clock API 유지
4. **메서드 교체**: 런타임에 안전한 버전으로 교체

### 통합 시스템

1. **이중 tick 방지**: 시뮬레이션 유무에 따른 조건부 tick
2. **생명주기 관리**: 룸 생성부터 정리까지 자동 관리
3. **성능 최적화**: 필요에 따른 적응형 업데이트
4. **안정성**: 다층 예외 처리 시스템

---
*이 문서는 Colyseus Room 클래스의 업데이트 시스템 분석을 바탕으로 작성되었습니다.*