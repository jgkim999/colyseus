# Colyseus Room 관리 시스템 및 메시지 전달 분석

## 📋 개요

Colyseus의 Room 관리는 **Server**, **MatchMaker**, **Transport** 계층으로 구성된 복합 시스템입니다. 클라이언트 메시지가 Room에 도달하는 과정을 상세히 분석합니다.

## 🏗️ 시스템 아키텍처

```text
Client → Transport → Server → MatchMaker → Room
                ↓
            IPC/Presence
                ↓
        Remote Process Room
```

## 🎯 핵심 관리 컴포넌트

### 1. Server 클래스 - 최상위 관리자

```typescript
export class Server {
  public transport: Transport;
  protected presence: Presence;
  protected driver: MatchMakerDriver;

  constructor(options: ServerOptions = {}) {
    this.presence = options.presence || new LocalPresence();
    this.driver = options.driver || new LocalDriver();

    // MatchMaker 초기화
    matchMaker.setup(
      this.presence,
      this.driver,
      options.publicAddress,
      options.selectProcessIdToCreateRoom,
    );
  }
}
```

**주요 역할**:

- Transport 계층 관리
- MatchMaker 초기화 및 설정
- Room 타입 정의 (`define()` 메서드)
- HTTP 매치메이킹 라우트 처리

### 2. MatchMaker - Room 생명주기 관리

```typescript
// 전역 Room 저장소
const handlers: {[id: string]: RegisteredHandler} = {};
const rooms: {[roomId: string]: Room} = {};

export let presence: Presence;
export let driver: MatchMakerDriver;
export let processId: string;
```

**핵심 기능**:

- Room 생성/삭제/조회
- 클라이언트 매치메이킹
- 프로세스 간 통신 (IPC)
- 좌석 예약 시스템

## 🔄 Room 생명주기 관리

### Room 등록 과정

#### 1. Server.define() - Room 타입 등록

```typescript
public define<T extends Type<Room>>(
  name: string,
  roomClass: T,
  defaultOptions?: any
): RegisteredHandler {
  return matchMaker.defineRoomType(name, roomClass, options);
}
```

#### 2. MatchMaker.defineRoomType() - 핸들러 등록

```typescript
export function defineRoomType<T extends Type<Room>>(
  roomName: string,
  klass: T,
  defaultOptions?: any
) {
  const registeredHandler = new RegisteredHandler(roomName, klass, defaultOptions);
  handlers[roomName] = registeredHandler;
  return registeredHandler;
}
```

### Room 생성 과정

#### 1. 매치메이킹 요청 처리

```typescript
// Server.ts - HTTP 매치메이킹 라우트
protected async handleMatchMakeRequest(req: IncomingMessage, res: ServerResponse) {
  const response = await matchMaker.controller.invokeMethod(
    method,      // 'join', 'create', 'joinOrCreate'
    roomName,
    clientOptions,
    authContext
  );
}
```

#### 2. Room 생성 결정

```typescript
// MatchMaker.ts
export async function createRoom(roomName: string, clientOptions: ClientOptions): Promise<IRoomCache> {
  // 프로세스 선택
  const selectedProcessId = await selectProcessIdToCreateRoom(roomName, clientOptions);

  if (selectedProcessId === processId) {
    // 로컬 프로세스에서 생성
    room = await handleCreateRoom(roomName, clientOptions);
  } else {
    // 원격 프로세스에 생성 요청
    room = await requestFromIPC(
      presence,
      getProcessChannel(selectedProcessId),
      undefined,
      [roomName, clientOptions]
    );
  }
}
```

#### 3. 실제 Room 인스턴스 생성

```typescript
export async function handleCreateRoom(roomName: string, clientOptions: ClientOptions): Promise<IRoomCache> {
  const handler = getHandler(roomName);
  const room = new handler.klass();

  // Room 초기화
  room.roomId = generateId();
  room['__init']();
  room.roomName = roomName;
  room.presence = presence;

  // 드라이버에 캐시 생성
  room.listing = driver.createInstance({
    name: roomName,
    processId,
    ...additionalListingData
  });

  // onCreate 호출
  if (room.onCreate) {
    await room.onCreate(merge({}, clientOptions, handler.options));
  }

  // 이벤트 바인딩
  room._events.on('join', onClientJoinRoom.bind(this, room));
  room._events.on('leave', onClientLeaveRoom.bind(this, room));
  room._events.once('dispose', disposeRoom.bind(this, roomName, room));

  // 전역 rooms 객체에 등록
  rooms[room.roomId] = room;

  return room.listing;
}
```

## 📨 클라이언트 메시지 전달 과정

### 1. Transport 계층에서 메시지 수신

```typescript
// Transport 구현체 (예: WebSocketTransport)
websocket.on('message', (data) => {
  // 클라이언트 식별 및 Room 찾기
  const client = getClientByWebSocket(websocket);
  const room = getRoomByClient(client);

  // Room의 _onMessage 호출
  room._onMessage(client, data);
});
```

### 2. Room._onMessage() - 메시지 파싱

```typescript
// Room.ts
protected _onMessage(client: Client & ClientPrivate, buffer: Buffer) {
  if (client.state === ClientState.LEAVING) { return; }

  const it: Iterator = { offset: 1 };
  const code = buffer[0];

  if (code === Protocol.ROOM_DATA) {
    // 메시지 타입 추출
    const messageType = (decode.stringCheck(buffer, it))
      ? decode.string(buffer, it)
      : decode.number(buffer, it);

    // 메시지 데이터 추출
    const message = (buffer.byteLength > it.offset)
      ? unpack(buffer.subarray(it.offset, buffer.byteLength))
      : undefined;

    // 핸들러 찾기 및 실행
    const messageTypeHandler = this.onMessageHandlers[messageType];

    if (messageTypeHandler) {
      // 검증 (있는 경우)
      if (messageTypeHandler.validate !== undefined) {
        message = messageTypeHandler.validate(message);
      }

      // 핸들러 실행
      messageTypeHandler.callback(client, message);
    } else {
      // 와일드카드 또는 기본 핸들러
      (this.onMessageHandlers['*'] || this.onMessageHandlers['__no_message_handler'])
        .callback(client, messageType, message);
    }
  }
}
```

### 3. 메시지 핸들러 등록 시스템

```typescript
// Room.ts
private onMessageHandlers: {
  [id: string]: {
    callback: (...args: any[]) => void,
    validate?: (data: unknown) => any,
  }
} = {
  '__no_message_handler': {
    callback: (client: Client, messageType: string, _: unknown) => {
      const errorMessage = `room onMessage for "${messageType}" not registered.`;

      if (isDevMode) {
        client.error(ErrorCode.INVALID_PAYLOAD, errorMessage);
      } else {
        client.leave(Protocol.WS_CLOSE_WITH_ERROR, errorMessage);
      }
    }
  }
};

public onMessage<T = any>(
  messageType: string | number,
  callback: (client: Client, message: T) => void,
  validate?: (message: unknown) => T,
) {
  // 예외 처리 래핑
  this.onMessageHandlers[messageType] = (this.onUncaughtException !== undefined)
    ? { validate, callback: wrapTryCatch(callback, this.onUncaughtException.bind(this), OnMessageException, 'onMessage', false, messageType) }
    : { validate, callback };
}
```

## 🔗 클라이언트 연결 과정

### 1. 좌석 예약 (Seat Reservation)

```typescript
// MatchMaker.ts
export async function reserveSeatFor(room: IRoomCache, options: ClientOptions, authData?: any) {
  const sessionId: string = authData?.sessionId || generateId();

  // 원격 또는 로컬 Room에 좌석 예약 요청
  const successfulSeatReservation = await remoteRoomCall(
    room.roomId,
    '_reserveSeat',
    [sessionId, options, authData]
  );

  if (!successfulSeatReservation) {
    throw new SeatReservationError(`${room.roomId} is already full.`);
  }

  return { room, sessionId };
}
```

### 2. Room._reserveSeat() - 좌석 예약 처리

```typescript
// Room.ts
protected async _reserveSeat(
  sessionId: string,
  joinOptions: any = true,
  authData: any = undefined,
  seconds: number = this.seatReservationTime,
  allowReconnection: boolean = false
) {
  if (!allowReconnection && this.hasReachedMaxClients()) {
    return false;
  }

  // 좌석 예약 정보 저장
  this.reservedSeats[sessionId] = [joinOptions, authData, false, allowReconnection];

  if (!allowReconnection) {
    // 클라이언트 수 증가
    await this._incrementClientCount();

    // 타임아웃 설정
    this.reservedSeatTimeouts[sessionId] = setTimeout(async () => {
      delete this.reservedSeats[sessionId];
      delete this.reservedSeatTimeouts[sessionId];
      await this._decrementClientCount();
    }, seconds * 1000);
  }

  return true;
}
```

### 3. Room._onJoin() - 클라이언트 입장 처리

```typescript
// Room.ts
public async ['_onJoin'](client: Client & ClientPrivate, authContext: AuthContext) {
  const sessionId = client.sessionId;

  // 재연결 토큰 생성
  client.reconnectionToken = generateId();

  // 좌석 예약 정보 가져오기
  const [joinOptions, authData, isConsumed, isWaitingReconnection] = this.reservedSeats[sessionId];

  // 예약 소비 표시
  this.reservedSeats[sessionId][2] = true;

  // 인증 처리
  if (this.onAuth !== Room.prototype.onAuth) {
    client.auth = await this.onAuth(client, joinOptions, authContext);
  }

  // 클라이언트 배열에 추가
  this.clients.push(client);

  // onJoin 호출
  if (this.onJoin) {
    await this.onJoin(client, joinOptions, client.auth);
  }

  // 메시지 핸들러 바인딩
  client.ref.on('message', this._onMessage.bind(this, client));

  // 룸 입장 확인 메시지 전송
  client.raw(getMessageBytes[Protocol.JOIN_ROOM](
    client.reconnectionToken,
    this._serializer.id,
    this._serializer.handshake && this._serializer.handshake(),
  ));
}
```

## 🌐 프로세스 간 통신 (IPC)

### IPC 채널 시스템

```typescript
// MatchMaker.ts
function getRoomChannel(roomId: string) {
  return `$${roomId}`;
}

function getProcessChannel(id: string = processId) {
  return `p:${id}`;
}

// Room별 IPC 구독
await subscribeIPC(
  presence,
  processId,
  getRoomChannel(room.roomId),
  (method, args) => {
    return (!args && typeof (room[method]) !== 'function')
      ? room[method]
      : room[method].apply(room, args);
  },
);
```

### 원격 Room 호출

```typescript
// MatchMaker.ts
export async function remoteRoomCall<R= any>(
  roomId: string,
  method: string,
  args?: any[]
): Promise<R> {
  const room = rooms[roomId];

  if (!room) {
    // 원격 프로세스의 Room 호출
    return await requestFromIPC<R>(
      presence,
      getRoomChannel(roomId),
      method,
      args
    );
  } else {
    // 로컬 Room 직접 호출
    return (!args && typeof (room[method]) !== 'function')
      ? room[method]
      : (await room[method].apply(room, args));
  }
}
```

## 📊 Room 상태 관리

### 드라이버 시스템

```typescript
// MatchMaker.ts
export let driver: MatchMakerDriver;

// Room 캐시 생성
room.listing = driver.createInstance({
  name: roomName,
  processId,
  roomId: room.roomId,
  maxClients: room.maxClients,
  clients: 0,
  locked: false,
  private: false
});

// 캐시 저장
await room.listing.save();
```

### 통계 관리

```typescript
// MatchMaker.ts
function onClientJoinRoom(room: Room, client: Client) {
  // 로컬 CCU 증가
  stats.local.ccu++;
  stats.persist();

  // 핸들러 이벤트 발생
  handlers[room.roomName].emit('join', room, client);
}

function onClientLeaveRoom(room: Room, client: Client, willDispose: boolean) {
  // 로컬 CCU 감소
  stats.local.ccu--;
  stats.persist();

  // 핸들러 이벤트 발생
  handlers[room.roomName].emit('leave', room, client, willDispose);
}
```

## 🔄 메시지 전달 플로우 요약

### 전체 메시지 전달 과정

```
1. Client sends message
   ↓
2. Transport receives WebSocket message
   ↓
3. Transport identifies Client and Room
   ↓
4. Room._onMessage(client, buffer) called
   ↓
5. Message parsed (Protocol.ROOM_DATA)
   ↓
6. Message type extracted
   ↓
7. Message data unpacked (msgpack)
   ↓
8. Handler lookup in onMessageHandlers
   ↓
9. Validation (if defined)
   ↓
10. Handler callback executed
    ↓
11. Game logic processes message
    ↓
12. State changes (if any)
    ↓
13. broadcastPatch() sends updates to clients
```

### 코드 예시 - 완전한 메시지 처리

```typescript
class GameRoom extends Room {
  onCreate() {
    // 메시지 핸들러 등록
    this.onMessage("move", (client, message) => {
      // 1. 메시지 검증
      if (!this.validateMoveMessage(message)) {
        client.error(ErrorCode.INVALID_PAYLOAD, "Invalid move");
        return;
      }

      // 2. 게임 로직 처리
      const player = this.state.players.get(client.sessionId);
      if (player) {
        player.x = message.x;
        player.y = message.y;

        // 3. 상태 변경 (자동으로 클라이언트에 동기화됨)
        // broadcastPatch()가 patchRate 주기로 호출됨
      }
    });
  }

  validateMoveMessage(message: any): boolean {
    return message &&
           typeof message.x === 'number' &&
           typeof message.y === 'number';
  }
}
```

## 🛡️ 에러 처리 및 안정성

### 메시지 처리 에러

```typescript
// Room.ts - _onMessage()
try {
  message = (buffer.byteLength > it.offset)
    ? unpack(buffer.subarray(it.offset, buffer.byteLength))
    : undefined;

  // 커스텀 검증
  if (messageTypeHandler?.validate !== undefined) {
    message = messageTypeHandler.validate(message);
  }
} catch (e) {
  debugAndPrintError(e);
  client.leave(Protocol.WS_CLOSE_WITH_ERROR);
  return;
}
```

### 예외 처리 래핑

```typescript
// Room.ts - onMessage 등록 시
this.onMessageHandlers[messageType] = (this.onUncaughtException !== undefined)
  ? {
      validate,
      callback: wrapTryCatch(
        callback,
        this.onUncaughtException.bind(this),
        OnMessageException,
        'onMessage',
        false,
        messageType
      )
    }
  : { validate, callback };
```

## 🎯 핵심 특징

1. **계층화된 아키텍처**: Transport → Server → MatchMaker → Room
2. **프로세스 간 통신**: IPC를 통한 분산 Room 관리
3. **좌석 예약 시스템**: 동시성 제어 및 타임아웃 관리
4. **메시지 타입 시스템**: 타입별 핸들러 등록 및 검증
5. **자동 상태 동기화**: 상태 변경 자동 감지 및 패치 전송
6. **예외 안전성**: 다층 예외 처리 및 에러 복구
7. **확장성**: 멀티 프로세스 지원 및 로드 밸런싱

---
*이 문서는 Colyseus Room 관리 시스템과 메시지 전달 과정 분석을 바탕으로 작성되었습니다.*
