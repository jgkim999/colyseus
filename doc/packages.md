# Colyseus Packages 구성 및 사용법

이 문서는 Colyseus 프로젝트의 `packages/` 폴더 내 패키지들의 구성과 사용법을 설명합니다.

## 📁 패키지 구조

```text
packages/
├── auth/                    # 인증 유틸리티
├── core/                    # 핵심 프레임워크
├── drivers/                 # 데이터베이스 드라이버
│   ├── mikro-orm-driver/
│   ├── mongoose-driver/
│   └── redis-driver/
├── example/                 # 예제 코드
├── greeting-banner/         # 시작 배너
├── loadtest/               # 부하 테스트 도구
├── presence/               # 프레즌스 관리
│   └── redis-presence/
├── serializer/             # 직렬화 도구
│   ├── fossil-delta-serializer/
│   └── legacy-schema-serializer/
├── testing/                # 테스트 유틸리티
├── tools/                  # 개발/배포 도구
└── transport/              # 전송 계층
    ├── bun-websockets/
    ├── h3-transport/
    ├── tcp-transport/
    ├── uwebsockets-transport/
    └── ws-transport/
```

## 🔧 핵심 패키지

### @colyseus/core

**설명**: Colyseus 멀티플레이어 프레임워크의 핵심 기능
**버전**: 0.16.19

**주요 기능**:

- Server, Room 클래스
- MatchMaker 시스템
- 프로토콜 및 에러 처리
- 직렬화 및 전송 계층

**사용법**:

```typescript
import { Server, Room } from '@colyseus/core';

class MyRoom extends Room {
  onCreate(options: any) {
    // 룸 생성 로직
  }
}

const server = new Server();
server.define('my_room', MyRoom);
```

### @colyseus/auth

**설명**: Colyseus용 인증 유틸리티
**버전**: 0.16.6

**주요 기능**:

- JWT 토큰 관리
- OAuth 인증
- 비밀번호 해싱
- 세션 관리

**사용법**:

```typescript
import { auth, JWT } from '@colyseus/auth';

// JWT 토큰 생성
const token = JWT.sign({ userId: 123 });

// 인증 미들웨어 설정
app.use('/auth', auth({
  // 인증 설정
}));
```

## 🚀 전송 계층 (Transport)

### @colyseus/ws-transport

**설명**: WebSocket 기반 전송 계층 (기본)
**버전**: 0.16.5

**사용법**:

```typescript
import { Server } from '@colyseus/core';
import { WebSocketTransport } from '@colyseus/ws-transport';

const server = new Server({
  transport: new WebSocketTransport()
});
```

### @colyseus/uwebsockets-transport

**설명**: 고성능 uWebSockets.js 기반 전송 계층
**버전**: 0.16.9

**사용법**:

```typescript
import { uWebSocketsTransport } from '@colyseus/uwebsockets-transport';

const server = new Server({
  transport: new uWebSocketsTransport()
});
```

### @colyseus/h3-transport

**설명**: HTTP/3 WebTransport 기반 전송 계층
**버전**: 0.16.3

**사용법**:

```typescript
import { H3Transport } from '@colyseus/h3-transport';

const server = new Server({
  transport: new H3Transport()
});
```

## 💾 데이터베이스 드라이버 (Drivers)

### @colyseus/redis-driver

**설명**: Redis 기반 매치메이킹 드라이버
**버전**: 0.16.1

**사용법**:

```typescript
import { RedisDriver } from '@colyseus/redis-driver';

const server = new Server({
  driver: new RedisDriver({
    host: 'localhost',
    port: 6379
  })
});
```

### @colyseus/mongoose-driver

**설명**: MongoDB/Mongoose 기반 드라이버
**버전**: 0.16.1

**사용법**:

```typescript
import { MongooseDriver } from '@colyseus/mongoose-driver';

const server = new Server({
  driver: new MongooseDriver('mongodb://localhost:27017/colyseus')
});
```

## 👥 프레즌스 관리 (Presence)

### @colyseus/redis-presence

**설명**: Redis 기반 프레즌스 관리
**버전**: 0.16.4

**사용법**:

```typescript
import { RedisPresence } from '@colyseus/redis-presence';

const server = new Server({
  presence: new RedisPresence({
    host: 'localhost',
    port: 6379
  })
});
```

## 📦 직렬화 (Serializers)

### @colyseus/fossil-delta-serializer

**설명**: Fossil Delta 알고리즘 기반 직렬화
**버전**: 0.16.0

**사용법**:

```typescript
import { FossilDeltaSerializer } from '@colyseus/fossil-delta-serializer';

class MyRoom extends Room {
  onCreate() {
    this.setSerializer(new FossilDeltaSerializer());
  }
}
```

## 🛠️ 개발 도구

### @colyseus/tools

**설명**: 개발 및 배포를 위한 도구 모음
**버전**: 0.16.12

**주요 기능**:

- 개발 서버 설정
- PM2 배포 스크립트
- 통계 리포팅

**사용법**:

```typescript
import { listen } from '@colyseus/tools';

listen({
  port: 2567,
  server: gameServer
});
```

### @colyseus/loadtest

**설명**: 부하 테스트 도구
**버전**: 0.16.1

**사용법**:

```bash
npx @colyseus/loadtest --endpoint ws://localhost:2567 --room my_room --clients 100
```

### @colyseus/testing

**설명**: 테스트 유틸리티
**버전**: 0.16.3

**사용법**:

```typescript
import { ColyseusTestServer } from '@colyseus/testing';

const server = new ColyseusTestServer();
await server.listen(2567);
```

## 📋 설치 및 사용 가이드

### 기본 설치

```bash
npm install @colyseus/core @colyseus/ws-transport
```

### 고성능 설정

```bash
npm install @colyseus/uwebsockets-transport @colyseus/redis-driver @colyseus/redis-presence
```

### 인증 기능 추가

```bash
npm install @colyseus/auth
```

### 개발 도구 설치

```bash
npm install @colyseus/tools @colyseus/testing @colyseus/loadtest
```

## 🔗 의존성 관계

- **@colyseus/core**: 모든 패키지의 기본 의존성
- **Transport 패키지들**: @colyseus/core에 의존
- **Driver 패키지들**: @colyseus/core에 의존
- **@colyseus/auth**: Express와 함께 사용
- **@colyseus/tools**: 개발 환경 설정에 필요

## 📝 참고사항

- 모든 패키지는 Node.js 18.x 이상 필요
- TypeScript 지원
- ESM/CommonJS 모두 지원
- 각 패키지는 독립적으로 설치 가능
- workspace 환경에서 개발됨

---
*이 문서는 Colyseus 0.16.x 버전 기준으로 작성되었습니다.*
