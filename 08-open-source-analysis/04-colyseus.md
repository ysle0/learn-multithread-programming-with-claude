# Colyseus - 멀티플레이어 게임 서버

## 📌 프로젝트 개요

**Colyseus**는 TypeScript/Node.js로 작성된 멀티플레이어 게임 서버 프레임워크입니다. 실시간 상태 동기화에 특화되어 있습니다.

### 기본 정보
- **저장소**: https://github.com/colyseus/colyseus
- **언어**: TypeScript/Node.js
- **라이선스**: MIT License
- **주요 개발자**: Endel Dreyer
- **활성 상태**: 활발히 개발 중
- **난이도**: ⭐⭐

### 주요 특징
- **간단한 API**: 빠른 프로토타이핑 가능
- **자동 상태 동기화**: Delta 압축으로 효율적 전송
- **Room 기반 아키텍처**: 매치/방 관리 간편
- **다양한 클라이언트 SDK**: Unity, Cocos, Defold 등
- **Node.js 기반**: JavaScript/TypeScript 개발자 친화적

---

## 🎯 왜 Colyseus를 분석해야 하는가?

### 학습 가치

1. **이벤트 기반 동시성 모델**
   - Node.js Event Loop 활용
   - 비동기 I/O 패턴
   - 싱글 스레드의 높은 동시성

2. **상태 동기화 설계**
   - Delta 기반 업데이트
   - 스키마 정의
   - 자동 직렬화

3. **간단한 아키텍처**
   - 멀티스레드보다 이해하기 쉬움
   - 빠른 개발 사이클
   - 실시간 게임에 적합

4. **프로덕션 사용 사례**
   - 수많은 게임에서 사용
   - 검증된 패턴

---

## 🏗️ 아키텍처

### 전체 구조

```
colyseus/
├── src/
│   ├── Room.ts              # Room 기본 클래스
│   ├── Server.ts            # 서버 인스턴스
│   ├── MatchMaker.ts        # 매치메이킹
│   ├── Protocol.ts          # 프로토콜 정의
│   └── serializer/          # 상태 직렬화
│       ├── SchemaSerializer.ts
│       └── FossilDeltaSerializer.ts
└── packages/
    └── schema/              # 스키마 정의
```

### 핵심 개념

#### 1. Room
게임의 독립적인 인스턴스입니다.

```typescript
import { Room, Client } from 'colyseus';

class GameRoom extends Room {
    maxClients = 4;

    onCreate(options: any) {
        this.setState(new GameState());

        this.onMessage("move", (client, data) => {
            this.handleMove(client, data);
        });
    }

    onJoin(client: Client, options: any) {
        console.log(client.sessionId, "joined!");
        this.state.players.set(client.sessionId, new Player());
    }

    onLeave(client: Client, consented: boolean) {
        console.log(client.sessionId, "left!");
        this.state.players.delete(client.sessionId);
    }

    onDispose() {
        console.log("room", this.roomId, "disposing...");
    }
}
```

#### 2. Schema (상태 정의)
타입 안전한 상태 정의와 자동 동기화:

```typescript
import { Schema, type, MapSchema } from '@colyseus/schema';

class Player extends Schema {
    @type("number") x: number = 0;
    @type("number") y: number = 0;
    @type("string") name: string = "";
    @type("number") score: number = 0;
}

class GameState extends Schema {
    @type({ map: Player }) players = new MapSchema<Player>();
    @type("number") round: number = 0;
}
```

---

## 🔬 핵심 기능 분석

### 1. 이벤트 루프 기반 동시성

#### Node.js Event Loop

```
   ┌───────────────────────────┐
┌─>│           timers          │  setTimeout, setInterval
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │  I/O callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │  internal
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           poll            │  retrieve new I/O events
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           check           │  setImmediate
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤      close callbacks      │  socket.on('close', ...)
   └───────────────────────────┘
```

#### Room의 이벤트 처리

```typescript
class GameRoom extends Room {
    onCreate(options: any) {
        // 게임 루프 설정 (30 FPS)
        this.setSimulationInterval((deltaTime) => {
            this.update(deltaTime);
        }, 1000 / 30);

        // 메시지 핸들러 등록
        this.onMessage("input", (client, message) => {
            // 싱글 스레드에서 순차 처리
            this.processInput(client, message);
        });
    }

    update(deltaTime: number) {
        // 게임 로직 업데이트
        this.state.players.forEach((player, sessionId) => {
            player.x += player.velocityX * deltaTime;
            player.y += player.velocityY * deltaTime;
        });

        // 상태는 자동으로 동기화됨 (다음 틱에)
    }
}
```

#### 핵심 특징

**1. 싱글 스레드 = No Race Conditions**
```typescript
// 동기화 필요 없음!
this.state.players.forEach((player) => {
    player.score += 10;  // 안전함
});
```

**2. 비동기 I/O는 콜백으로**
```typescript
async onJoin(client: Client) {
    // 데이터베이스 조회 (비동기)
    const userData = await database.getUser(client.auth.userId);

    // Event Loop로 돌아와서 처리
    this.state.players.set(client.sessionId, new Player(userData));
}
```

**3. CPU-bound 작업은 Worker로**
```typescript
import { Worker } from 'worker_threads';

class GameRoom extends Room {
    async computePathfinding(start: Point, end: Point) {
        // 별도 스레드에서 실행
        const worker = new Worker('./pathfinding-worker.js');

        return new Promise((resolve) => {
            worker.on('message', (path) => {
                resolve(path);
                worker.terminate();
            });

            worker.postMessage({ start, end });
        });
    }
}
```

---

### 2. Delta 기반 상태 동기화

#### Schema의 변경 감지

```typescript
class GameState extends Schema {
    @type({ map: Player }) players = new MapSchema<Player>();

    // 내부적으로:
    // 1. 모든 변경 추적
    // 2. Delta 생성
    // 3. 클라이언트로 전송
}

// Room에서 상태 변경
this.state.players.get(sessionId).x = 100;
// -> 자동으로 Delta 생성 및 전송!
```

#### Delta 인코딩

```
전체 상태 (처음):
{
  players: {
    "abc": { x: 0, y: 0, score: 0 },
    "def": { x: 10, y: 10, score: 5 }
  }
}
크기: ~100 bytes

Delta 업데이트 (이후):
{
  players: {
    "abc": { x: 5 }  // x만 변경
  }
}
크기: ~20 bytes (80% 절약!)
```

#### 구현 원리

```typescript
// 간소화된 버전
class Schema {
    private _changes: Map<string, any> = new Map();
    private _listeners: Set<Function> = new Set();

    set(key: string, value: any) {
        if (this[key] !== value) {
            this[key] = value;
            this._changes.set(key, value);
            this._notifyChanges();
        }
    }

    encode(): Uint8Array {
        // 변경된 필드만 인코딩
        const changes = Array.from(this._changes);
        this._changes.clear();

        return encode(changes);  // Binary format
    }

    private _notifyChanges() {
        // Room에 변경 통지
        this._listeners.forEach(listener => listener());
    }
}

// Room 내부
class Room {
    private broadcastPatch() {
        const encoded = this.state.encode();

        // 모든 클라이언트에 전송
        this.clients.forEach(client => {
            client.send(encoded);
        });
    }
}
```

---

### 3. 매치메이킹

#### 자동 매칭

```typescript
// 클라이언트
const room = await client.joinOrCreate("game_room", {
    mode: "battle_royale",
    region: "us-west"
});

// 서버
class GameRoom extends Room {
    static LOBBY_CHANNEL = "game_lobby";

    onCreate(options: any) {
        // Room 메타데이터 설정
        this.setMetadata({
            mode: options.mode,
            region: options.region
        });
    }

    // 참가 조건 검증
    onAuth(client: Client, options: any) {
        // 인증 로직
        return { userId: "abc123" };
    }

    onJoin(client: Client) {
        if (this.clients.length >= this.maxClients) {
            // 방이 꽉 차면 더 이상 참가 불가
            throw new Error("Room is full");
        }
    }
}

// 매치메이커 필터
gameServer.define("game_room", GameRoom)
    .filterBy(['mode', 'region']);
```

#### 커스텀 매칭 로직

```typescript
import { matchMaker } from 'colyseus';

class MatchMakingRoom extends Room {
    async onCreate(options: any) {
        // 플레이어 수 대기
        await this.waitForPlayers(4);

        // 게임 시작
        this.lock();  // 더 이상 참가 불가
        this.startGame();
    }

    private async waitForPlayers(count: number): Promise<void> {
        return new Promise((resolve) => {
            const interval = setInterval(() => {
                if (this.clients.length >= count) {
                    clearInterval(interval);
                    resolve();
                }
            }, 100);
        });
    }
}
```

---

### 4. 메시지 처리

#### 타입 안전한 메시지

```typescript
// 메시지 타입 정의
type MessageTypes = {
    "move": { x: number, y: number },
    "shoot": { angle: number, power: number },
    "chat": { text: string }
}

class GameRoom extends Room<GameState> {
    onCreate() {
        this.onMessage("move", (client, message: MessageTypes["move"]) => {
            const player = this.state.players.get(client.sessionId);
            player.x = message.x;
            player.y = message.y;
        });

        this.onMessage("shoot", (client, message: MessageTypes["shoot"]) => {
            this.createProjectile(client.sessionId, message.angle, message.power);
        });

        this.onMessage("chat", (client, message: MessageTypes["chat"]) => {
            this.broadcast("chat", {
                sender: client.sessionId,
                text: message.text
            });
        });
    }

    createProjectile(ownerId: string, angle: number, power: number) {
        const projectile = new Projectile();
        projectile.ownerId = ownerId;
        projectile.angle = angle;
        projectile.power = power;

        this.state.projectiles.set(generateId(), projectile);
    }
}
```

---

## 🚀 실전 예제

### 완전한 게임 서버

```typescript
import { Server, Room, Client } from 'colyseus';
import { Schema, type, MapSchema } from '@colyseus/schema';
import http from 'http';
import express from 'express';

// ========== 상태 정의 ==========
class Player extends Schema {
    @type("number") x: number = Math.random() * 800;
    @type("number") y: number = Math.random() * 600;
    @type("number") velocityX: number = 0;
    @type("number") velocityY: number = 0;
    @type("number") health: number = 100;
    @type("string") name: string = "";
}

class Bullet extends Schema {
    @type("number") x: number;
    @type("number") y: number;
    @type("number") velocityX: number;
    @type("number") velocityY: number;
    @type("string") ownerId: string;
}

class GameState extends Schema {
    @type({ map: Player }) players = new MapSchema<Player>();
    @type({ map: Bullet }) bullets = new MapSchema<Bullet>();
    @type("number") worldTime: number = 0;
}

// ========== Room 구현 ==========
class BattleRoom extends Room<GameState> {
    maxClients = 10;
    private bulletIdCounter = 0;

    onCreate(options: any) {
        this.setState(new GameState());

        // 게임 루프 (60 FPS)
        this.setSimulationInterval((deltaTime) => {
            this.update(deltaTime);
        }, 1000 / 60);

        // 메시지 핸들러
        this.onMessage("input", (client, input) => {
            this.handleInput(client, input);
        });

        this.onMessage("shoot", (client, data) => {
            this.handleShoot(client, data);
        });

        console.log("BattleRoom created!", this.roomId);
    }

    onJoin(client: Client, options: any) {
        const player = new Player();
        player.name = options.name || `Player${this.clients.length}`;

        this.state.players.set(client.sessionId, player);

        console.log(client.sessionId, "joined as", player.name);

        // 환영 메시지
        client.send("welcome", {
            message: `Welcome ${player.name}!`
        });
    }

    onLeave(client: Client, consented: boolean) {
        this.state.players.delete(client.sessionId);
        console.log(client.sessionId, "left");
    }

    onDispose() {
        console.log("BattleRoom disposed", this.roomId);
    }

    // ===== 게임 로직 =====
    private update(deltaTime: number) {
        const dt = deltaTime / 1000;  // seconds

        this.state.worldTime += deltaTime;

        // 플레이어 이동
        this.state.players.forEach((player) => {
            player.x += player.velocityX * dt;
            player.y += player.velocityY * dt;

            // 경계 체크
            player.x = Math.max(0, Math.min(800, player.x));
            player.y = Math.max(0, Math.min(600, player.y));
        });

        // 총알 이동
        this.state.bullets.forEach((bullet, id) => {
            bullet.x += bullet.velocityX * dt;
            bullet.y += bullet.velocityY * dt;

            // 화면 밖으로 나가면 제거
            if (bullet.x < 0 || bullet.x > 800 ||
                bullet.y < 0 || bullet.y > 600) {
                this.state.bullets.delete(id);
            }
        });

        // 충돌 검사
        this.checkCollisions();
    }

    private handleInput(client: Client, input: any) {
        const player = this.state.players.get(client.sessionId);
        if (!player) return;

        const speed = 200;

        player.velocityX = input.horizontal * speed;
        player.velocityY = input.vertical * speed;
    }

    private handleShoot(client: Client, data: any) {
        const player = this.state.players.get(client.sessionId);
        if (!player) return;

        const bullet = new Bullet();
        bullet.x = player.x;
        bullet.y = player.y;
        bullet.velocityX = Math.cos(data.angle) * 500;
        bullet.velocityY = Math.sin(data.angle) * 500;
        bullet.ownerId = client.sessionId;

        this.state.bullets.set(`bullet_${this.bulletIdCounter++}`, bullet);
    }

    private checkCollisions() {
        this.state.bullets.forEach((bullet, bulletId) => {
            this.state.players.forEach((player, playerId) => {
                // 자기 자신의 총알은 맞지 않음
                if (playerId === bullet.ownerId) return;

                // 거리 계산
                const dx = bullet.x - player.x;
                const dy = bullet.y - player.y;
                const distance = Math.sqrt(dx * dx + dy * dy);

                if (distance < 20) {  // 충돌!
                    player.health -= 10;

                    // 총알 제거
                    this.state.bullets.delete(bulletId);

                    // 체력 0이면 제거
                    if (player.health <= 0) {
                        this.broadcast("player_died", {
                            victim: playerId,
                            killer: bullet.ownerId
                        });

                        this.state.players.delete(playerId);
                    }
                }
            });
        });
    }
}

// ========== 서버 시작 ==========
const app = express();
const server = http.createServer(app);
const gameServer = new Server({
    server: server
});

// Room 등록
gameServer.define("battle", BattleRoom);

// 정적 파일 제공 (클라이언트)
app.use(express.static('public'));

gameServer.listen(2567);
console.log("Game server listening on ws://localhost:2567");
```

### 클라이언트 (Unity C#)

```csharp
using Colyseus;
using UnityEngine;
using System.Collections.Generic;

public class GameManager : MonoBehaviour
{
    private ColyseusClient client;
    private ColyseusRoom<GameState> room;

    private Dictionary<string, GameObject> playerObjects = new Dictionary<string, GameObject>();

    async void Start()
    {
        client = new ColyseusClient("ws://localhost:2567");

        room = await client.JoinOrCreate<GameState>("battle", new Dictionary<string, object>
        {
            { "name", "Player" + Random.Range(0, 1000) }
        });

        // 상태 변경 리스너
        room.State.players.OnAdd += OnPlayerAdded;
        room.State.players.OnRemove += OnPlayerRemoved;
        room.State.players.OnChange += OnPlayerChanged;

        // 메시지 리스너
        room.OnMessage<string>("welcome", (message) =>
        {
            Debug.Log(message);
        });

        Debug.Log("Joined room: " + room.Id);
    }

    void Update()
    {
        // 입력 전송
        float horizontal = Input.GetAxis("Horizontal");
        float vertical = Input.GetAxis("Vertical");

        if (horizontal != 0 || vertical != 0)
        {
            room.Send("input", new
            {
                horizontal = horizontal,
                vertical = vertical
            });
        }

        // 발사
        if (Input.GetMouseButtonDown(0))
        {
            Vector3 mousePos = Camera.main.ScreenToWorldPoint(Input.mousePosition);
            float angle = Mathf.Atan2(mousePos.y, mousePos.x);

            room.Send("shoot", new { angle = angle });
        }
    }

    void OnPlayerAdded(string sessionId, Player player)
    {
        GameObject playerObj = Instantiate(playerPrefab);
        playerObj.transform.position = new Vector3(player.x, player.y, 0);
        playerObjects[sessionId] = playerObj;

        // 플레이어 상태 변경 리스너
        player.OnChange += (changes) =>
        {
            playerObj.transform.position = new Vector3(player.x, player.y, 0);
            // 체력 UI 업데이트 등
        };
    }

    void OnPlayerRemoved(string sessionId, Player player)
    {
        Destroy(playerObjects[sessionId]);
        playerObjects.Remove(sessionId);
    }

    void OnApplicationQuit()
    {
        room?.Leave();
    }
}
```

---

## 📊 성능 특성

### 싱글 스레드 한계

```
단일 Room:
- 최대 ~10,000 ops/sec
- ~100-200 클라이언트 (게임에 따라 다름)

서버:
- 여러 Room을 동시에 실행 가능
- Room마다 독립적 (서로 영향 없음)
- 클러스터링으로 수평 확장
```

### 네트워크 효율

```
전체 상태 전송: 1KB - 10KB per tick
Delta 전송:     50B - 500B per tick

압축 비율: ~90-95% 절감
```

---

## 🎓 학습 포인트

### 1. 설치 및 실행

```bash
# 프로젝트 생성
npm init colyseus-app ./my-game-server

cd my-game-server
npm install

# 개발 모드 실행
npm start

# 프로덕션 빌드
npm run build
```

### 2. 간단한 Room

```typescript
import { Room, Client } from 'colyseus';

export class MyRoom extends Room {
    onCreate(options: any) {
        console.log("Room created!", options);
    }

    onJoin(client: Client, options: any) {
        console.log(client.sessionId, "joined!");
    }

    onLeave(client: Client, consented: boolean) {
        console.log(client.sessionId, "left!");
    }

    onDispose() {
        console.log("Room disposed!");
    }
}
```

---

## ⚠️ 주의사항

1. **CPU-bound 작업**: Worker Thread 사용 필수
2. **메모리 관리**: 상태가 클수록 메모리 증가
3. **네트워크**: Delta가 작아도 빈번한 업데이트는 부담
4. **확장성**: 단일 서버 한계 고려

---

## 🔗 관련 리소스

### 공식 문서
- GitHub: https://github.com/colyseus/colyseus
- 문서: https://docs.colyseus.io/

### 클라이언트 SDK
- Unity: https://github.com/colyseus/colyseus-unity3d
- JavaScript: https://github.com/colyseus/colyseus.js

---

## 📚 다음 단계

1. **Colyseus 설치 및 예제 실행**
2. **간단한 게임 Room 작성**
3. **클라이언트 연동**
4. **다음: [xsync](./05-xsync.md) 분석**

---

*이 문서는 학습 목적으로 작성되었습니다.*
