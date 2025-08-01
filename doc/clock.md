# Clock 클래스 동작원리 분석

## 📋 개요

Colyseus Clock 클래스는 게임 서버에서 시간 관리와 타이밍 이벤트를 처리하는 핵심 컴포넌트입니다.
`@colyseus/timer` 패키지로 제공되며, 정확한 시간 추적과 프레임 독립적인 게임 로직 실행을 지원합니다.

**주요 기능:**

- 고정밀 시간 측정 및 델타 타임 계산
- setTimeout/setInterval 래핑 및 관리
- 자동 타이머 정리 및 메모리 누수 방지
- 시뮬레이션 일시정지/재개 기능

## ⚙️ 핵심 구조

### 기본 속성

```typescript
export default class Clock {
  public elapsedTime: number = 0;        // 총 경과 시간 (ms)
  public deltaTime: number = 0;          // 이전 틱과의 시간 차이 (ms)

  private _currentTime: number = 0;      // 현재 시간
  private _previousTime: number = 0;     // 이전 틱 시간
  private _timers: Timer[] = [];         // 활성 타이머 목록
  private _timerIdCounter: number = 0;   // 타이머 ID 카운터
  private _isRunning: boolean = false;   // 실행 상태
}
```

### Timer 인터페이스

```typescript
interface Timer {
  id: number;                    // 고유 ID
  callback: Function;            // 실행할 콜백
  delay: number;                 // 지연 시간 (ms)
  interval?: boolean;            // 반복 실행 여부
  elapsedTime: number;          // 경과 시간
  active: boolean;              // 활성 상태
}
```

## 🕐 시간 관리 시스템

### 1. Clock 시작 및 초기화

```typescript
public start(): void {
  if (this._isRunning) return;

  this._isRunning = true;
  this._currentTime = this.now();
  this._previousTime = this._currentTime;
  this.elapsedTime = 0;
  this.deltaTime = 0;
}
```

**동작 과정:**

1. **중복 시작 방지**: 이미 실행 중이면 무시
2. **시간 초기화**: 현재 시간을 기준점으로 설정
3. **상태 리셋**: 경과 시간과 델타 타임 초기화

### 2. 시간 업데이트 (tick)

```typescript
public tick(): void {
  if (!this._isRunning) return;

  this._currentTime = this.now();
  this.deltaTime = this._currentTime - this._previousTime;
  this.elapsedTime += this.deltaTime;

  this._updateTimers();

  this._previousTime = this._currentTime;
}
```

**핵심 동작:**

1. **현재 시간 측정**: `performance.now()` 또는 `Date.now()` 사용
2. **델타 타임 계산**: 이전 틱과의 시간 차이
3. **총 경과 시간 누적**: 게임 시작부터의 총 시간
4. **타이머 업데이트**: 등록된 모든 타이머 처리
5. **이전 시간 갱신**: 다음 틱을 위한 준비

### 3. 고정밀 시간 측정

```typescript
private now(): number {
  // Node.js 환경
  if (typeof performance !== 'undefined' && performance.now) {
    return performance.now();
  }

  // 브라우저 환경 또는 fallback
  return Date.now();
}
```

**특징:**

- **performance.now()**: 마이크로초 정밀도, 단조 증가
- **Date.now()**: 밀리초 정밀도, 시스템 시간 변경 영향
- **자동 선택**: 환경에 따른 최적 방법 사용

## ⏰ 타이머 시스템

### 1. setTimeout 구현

```typescript
public setTimeout(callback: Function, delay: number): number {
  const timer: Timer = {
    id: ++this._timerIdCounter,
    callback,
    delay,
    interval: false,
    elapsedTime: 0,
    active: true
  };

  this._timers.push(timer);
  return timer.id;
}
```

### 2. setInterval 구현

```typescript
public setInterval(callback: Function, delay: number): number {
  const timer: Timer = {
    id: ++this._timerIdCounter,
    callback,
    delay,
    interval: true,
    elapsedTime: 0,
    active: true
  };

  this._timers.push(timer);
  return timer.id;
}
```

### 3. 타이머 업데이트 로직

```typescript
private _updateTimers(): void {
  for (let i = this._timers.length - 1; i >= 0; i--) {
    const timer = this._timers[i];

    if (!timer.active) {
      this._timers.splice(i, 1);
      continue;
    }

    timer.elapsedTime += this.deltaTime;

    if (timer.elapsedTime >= timer.delay) {
      // 콜백 실행
      timer.callback();

      if (timer.interval) {
        // 인터벌: 경과 시간 리셋
        timer.elapsedTime = 0;
      } else {
        // 타임아웃: 타이머 제거
        this._timers.splice(i, 1);
      }
    }
  }
}
```

**처리 과정:**

1. **역순 순회**: 배열 수정 중 인덱스 오류 방지
2. **비활성 타이머 제거**: 메모리 정리
3. **경과 시간 누적**: 델타 타임 추가
4. **지연 시간 확인**: 실행 조건 검사
5. **콜백 실행**: 조건 만족 시 함수 호출
6. **타이머 처리**: 인터벌은 리셋, 타임아웃은 제거

### 4. 타이머 취소

```typescript
public clearTimeout(id: number): void {
  const timer = this._timers.find(t => t.id === id);
  if (timer) {
    timer.active = false;
  }
}

public clearInterval(id: number): void {
  this.clearTimeout(id); // 동일한 로직
}
```

## 🔄 생명주기 관리

### 1. Clock 정지

```typescript
public stop(): void {
  this._isRunning = false;
}
```

### 2. 모든 타이머 정리

```typescript
public clear(): void {
  this._timers.length = 0;
  this._timerIdCounter = 0;
}
```

### 3. 완전 리셋

```typescript
public reset(): void {
  this.stop();
  this.clear();
  this.elapsedTime = 0;
  this.deltaTime = 0;
  this._currentTime = 0;
  this._previousTime = 0;
}
```

## 🎮 Room과의 통합

### 1. Room에서의 Clock 사용

```typescript
export abstract class Room {
  public clock: Clock = new Clock();

  protected __init() {
    // Clock 시작
    this.clock.start();
  }

  protected async _dispose() {
    // 모든 타이머 정리 및 Clock 정지
    this.clock.clear();
    this.clock.stop();
  }
}
```

### 2. 시뮬레이션 루프와의 연동

```typescript
public setSimulationInterval(callback: SimulationCallback, delay: number): void {
  this._simulationInterval = setInterval(() => {
    this.clock.tick();                    // ← Clock 업데이트
    callback(this.clock.deltaTime);       // ← 델타 타임 전달
  }, delay);
}
```

### 3. 상태 동기화와의 관계

```typescript
public broadcastPatch() {
  // 시뮬레이션 인터벌이 없으면 수동으로 tick
  if (!this._simulationInterval) {
    this.clock.tick();
  }

  // 상태 패치 적용
  const hasChanges = this._serializer.applyPatches(this.clients, this.state);
  return hasChanges;
}
```

## 🚀 실제 사용 예제

### 1. 기본 타이머 사용

```typescript
class GameRoom extends Room {
  onCreate() {
    // 5초 후 게임 시작
    this.clock.setTimeout(() => {
      this.startGame();
    }, 5000);

    // 30초마다 보상 지급
    this.clock.setInterval(() => {
      this.giveRewards();
    }, 30000);
  }

  startGame() {
    this.broadcast("game_start", { timestamp: this.clock.elapsedTime });
  }

  giveRewards() {
    this.clients.forEach(client => {
      // 보상 로직
    });
  }
}
```

### 2. 델타 타임 기반 물리 시뮬레이션

```typescript
class PhysicsRoom extends Room {
  onCreate() {
    this.setSimulationInterval(this.update.bind(this), 16); // 60fps
  }

  update(deltaTime: number) {
    // 프레임 독립적 이동
    this.state.players.forEach(player => {
      const deltaSeconds = deltaTime / 1000;

      player.x += player.velocityX * deltaSeconds;
      player.y += player.velocityY * deltaSeconds;

      // 중력 적용
      player.velocityY += this.GRAVITY * deltaSeconds;
    });

    // 쿨다운 처리
    this.state.abilities.forEach(ability => {
      ability.cooldown = Math.max(0, ability.cooldown - deltaTime);
    });
  }
}
```

### 3. 시간 기반 이벤트 시스템

```typescript
class EventRoom extends Room {
  private gameStartTime: number;
  private eventTimers: number[] = [];

  onCreate() {
    this.gameStartTime = this.clock.elapsedTime;
    this.scheduleEvents();
  }

  scheduleEvents() {
    // 1분 후 첫 번째 이벤트
    const timer1 = this.clock.setTimeout(() => {
      this.triggerEvent("boss_spawn");
    }, 60000);

    // 3분 후 두 번째 이벤트
    const timer2 = this.clock.setTimeout(() => {
      this.triggerEvent("treasure_drop");
    }, 180000);

    this.eventTimers.push(timer1, timer2);
  }

  triggerEvent(eventType: string) {
    const gameTime = this.clock.elapsedTime - this.gameStartTime;

    this.broadcast("event_trigger", {
      type: eventType,
      gameTime: gameTime
    });
  }

  onDispose() {
    // 이벤트 타이머 정리
    this.eventTimers.forEach(timerId => {
      this.clock.clearTimeout(timerId);
    });
  }
}
```

## ⚡ 성능 최적화

### 1. 타이머 풀링

```typescript
class OptimizedClock extends Clock {
  private _timerPool: Timer[] = [];

  private getTimer(): Timer {
    return this._timerPool.pop() || {
      id: 0,
      callback: null,
      delay: 0,
      interval: false,
      elapsedTime: 0,
      active: false
    };
  }

  private releaseTimer(timer: Timer): void {
    timer.callback = null;
    timer.active = false;
    this._timerPool.push(timer);
  }
}
```

### 2. 배치 처리

```typescript
private _updateTimers(): void {
  const expiredTimers: Timer[] = [];

  // 만료된 타이머 수집
  for (const timer of this._timers) {
    if (!timer.active) continue;

    timer.elapsedTime += this.deltaTime;

    if (timer.elapsedTime >= timer.delay) {
      expiredTimers.push(timer);
    }
  }

  // 배치로 콜백 실행
  for (const timer of expiredTimers) {
    timer.callback();

    if (timer.interval) {
      timer.elapsedTime = 0;
    } else {
      this.clearTimeout(timer.id);
    }
  }
}
```

### 3. 메모리 사용량 모니터링

```typescript
class MonitoredClock extends Clock {
  public getTimerCount(): number {
    return this._timers.filter(t => t.active).length;
  }

  public getMemoryUsage(): object {
    return {
      activeTimers: this.getTimerCount(),
      totalTimers: this._timers.length,
      elapsedTime: this.elapsedTime,
      isRunning: this._isRunning
    };
  }
}
```

## 🛡️ 예외 처리 및 안정성

### 1. 안전한 콜백 실행

```typescript
private _safeCallback(callback: Function): void {
  try {
    callback();
  } catch (error) {
    console.error('Timer callback error:', error);
    // 에러 발생해도 다른 타이머에 영향 없음
  }
}
```

### 2. 시간 동기화 검증

```typescript
public tick(): void {
  const currentTime = this.now();
  const expectedDelta = currentTime - this._previousTime;

  // 비정상적인 시간 점프 감지
  if (expectedDelta > 1000) { // 1초 이상
    console.warn(`Large time jump detected: ${expectedDelta}ms`);
    // 델타 타임 제한
    this.deltaTime = Math.min(expectedDelta, 100);
  } else {
    this.deltaTime = expectedDelta;
  }

  this.elapsedTime += this.deltaTime;
  this._updateTimers();
  this._previousTime = currentTime;
}
```

### 3. 메모리 누수 방지

```typescript
public clear(): void {
  // 콜백 참조 해제
  this._timers.forEach(timer => {
    timer.callback = null;
  });

  this._timers.length = 0;
  this._timerIdCounter = 0;
}
```

## 📊 디버깅 및 모니터링

### 1. 타이머 상태 추적

```typescript
public getTimerInfo(): object[] {
  return this._timers.map(timer => ({
    id: timer.id,
    delay: timer.delay,
    elapsedTime: timer.elapsedTime,
    remaining: timer.delay - timer.elapsedTime,
    interval: timer.interval,
    active: timer.active
  }));
}
```

### 2. 성능 메트릭

```typescript
class PerformanceClock extends Clock {
  private _tickCount: number = 0;
  private _totalTickTime: number = 0;

  public tick(): void {
    const startTime = this.now();
    super.tick();
    const tickTime = this.now() - startTime;

    this._tickCount++;
    this._totalTickTime += tickTime;
  }

  public getAverageTickTime(): number {
    return this._tickCount > 0 ? this._totalTickTime / this._tickCount : 0;
  }
}
```

## 🎯 주요 특징 요약

### Clock 클래스의 핵심 기능

1. **정확한 시간 측정**
   - `performance.now()` 기반 고정밀 시간
   - 단조 증가하는 시간 보장
   - 시스템 시간 변경에 영향받지 않음

2. **효율적인 타이머 관리**
   - 단일 tick에서 모든 타이머 처리
   - 메모리 효율적인 배열 기반 저장
   - 자동 정리 및 메모리 누수 방지

3. **프레임 독립적 실행**
   - 델타 타임 기반 계산
   - 가변 프레임레이트 지원
   - 일관된 게임 로직 실행

4. **Room 시스템과의 완벽한 통합**
   - 자동 생명주기 관리
   - 예외 처리 래핑
   - 상태 동기화와 연동

### 사용 시 주의사항

1. **tick() 호출 필수**: 시간 업데이트를 위해 정기적 호출 필요
2. **메모리 관리**: 불필요한 타이머는 명시적으로 해제
3. **콜백 예외**: 타이머 콜백에서 발생한 예외는 다른 타이머에 영향
4. **시간 정확도**: 시스템 부하에 따라 타이머 정확도 변동 가능

---
*이 문서는 @colyseus/timer 패키지의 Clock 클래스 분석을 바탕으로 작성되었습니다.*