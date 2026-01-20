# Nakama - 게임 서버 프레임워크

## 📌 프로젝트 개요

**Nakama**는 Go로 작성된 오픈소스 게임 서버 프레임워크입니다. 멀티플레이어 게임의 백엔드를 쉽게 구축할 수 있도록 설계되었습니다.

### 기본 정보
- **저장소**: https://github.com/heroiclabs/nakama
- **언어**: Go
- **라이선스**: Apache License 2.0
- **주요 개발사**: Heroic Labs
- **활성 상태**: 매우 활발히 개발 중
- **난이도**: ⭐⭐⭐

### 주요 특징
- **완전한 게임 서버**: 인증, 매치메이킹, 리더보드, 소셜 기능
- **고성능**: Go의 동시성 모델 활용
- **확장성**: 수평적 확장 지원
- **실시간 통신**: WebSocket 기반
- **풍부한 기능**: 사용자 관리, 스토리지, RPC 등

---

## 🎯 왜 Nakama를 분석해야 하는가?

### 학습 가치

1. **Go 동시성 모델의 실전 활용**
   - Goroutine을 활용한 비동기 처리
   - Channel 패턴
   - Context를 통한 취소 및 타임아웃

2. **게임 서버 아키텍처**
   - 실시간 매치메이킹
   - 세션 관리
   - 상태 동기화

3. **확장 가능한 설계**
   - 수평적 확장
   - 분산 시스템 패턴
   - 부하 분산

4. **프로덕션 레벨 코드**
   - 실제 게임에서 사용
   - 철저한 에러 처리
   - 성능 최적화

---

## 🏗️ 아키텍처

### 전체 구조

```
nakama/
├── server/
│   ├── core_*.go         # 핵심 기능 (매치메이킹, 세션 등)
│   ├── pipeline_*.go     # 메시지 처리 파이프라인
│   ├── runtime_*.go      # 사용자 정의 로직 실행
│   └── match_*.go        # 매치 관리
├── console/              # 관리자 콘솔
└── data/                 # 데이터베이스 스키마
```

### 핵심 컴포넌트

#### 1. Session Manager
```go
type SessionRegistry interface {
    Add(session Session)
    Remove(session Session)
    Get(sessionID uuid.UUID) Session
    Count() int
}

type LocalSessionRegistry struct {
    sessions    sync.Map  // sessionID -> Session
    sessionsByUID sync.Map  // userID -> []Session
}
```

#### 2. Match Handler
```go
type Match interface {
    JoinAttempt(ctx context.Context, tick int64, state interface{},
                userID uuid.UUID, username string, vars map[string]string)
                (interface{}, bool, string)

    Join(ctx context.Context, tick int64, state interface{},
         userID uuid.UUID) interface{}

    Loop(ctx context.Context, tick int64, state interface{},
         messages []MatchData) interface{}

    Terminate(ctx context.Context, tick int64, state interface{},
              graceSeconds int) interface{}
}
```

#### 3. Message Router
```go
type Pipeline struct {
    router *mux.Router
    handlers map[string]RuntimeMatchCreateFunction
}
```

---

## 🔬 핵심 기능 분석

### 1. 세션 관리

#### Session 구조

```go
type Session interface {
    UserID() uuid.UUID
    Username() string
    SessionID() uuid.UUID
    Send(envelope *rtapi.Envelope) error
    Close()
}

type sessionWS struct {
    id        uuid.UUID
    userID    uuid.UUID
    username  string
    conn      *websocket.Conn
    ctx       context.Context
    ctxCancel context.CancelFunc

    // 동시성 제어
    sendMu    sync.Mutex
    closeMu   sync.Mutex
    closed    atomic.Bool

    // 메시지 큐
    outgoingCh chan *rtapi.Envelope
}

func (s *sessionWS) Send(envelope *rtapi.Envelope) error {
    if s.closed.Load() {
        return errors.New("session is closed")
    }

    select {
    case s.outgoingCh <- envelope:
        return nil
    case <-s.ctx.Done():
        return s.ctx.Err()
    default:
        return errors.New("outgoing queue full")
    }
}

func (s *sessionWS) processOutgoing() {
    for {
        select {
        case envelope := <-s.outgoingCh:
            s.sendMu.Lock()
            err := s.conn.WriteJSON(envelope)
            s.sendMu.Unlock()

            if err != nil {
                s.Close()
                return
            }

        case <-s.ctx.Done():
            return
        }
    }
}
```

#### 핵심 패턴

**1. Context를 활용한 생명주기 관리**
```go
ctx, cancel := context.WithCancel(context.Background())
session := &sessionWS{
    ctx:       ctx,
    ctxCancel: cancel,
}

// 세션 종료
func (s *sessionWS) Close() {
    if s.closed.CompareAndSwap(false, true) {
        s.ctxCancel()  // 모든 고루틴에 취소 신호
        close(s.outgoingCh)
        s.conn.Close()
    }
}
```

**2. Channel을 활용한 메시지 큐**
```go
outgoingCh := make(chan *rtapi.Envelope, 64)

// Producer (게임 로직)
session.Send(message)  // Channel에 메시지 추가

// Consumer (WebSocket 전송)
go session.processOutgoing()  // Goroutine에서 처리
```

**3. Atomic 연산으로 상태 관리**
```go
type sessionWS struct {
    closed atomic.Bool
}

if s.closed.CompareAndSwap(false, true) {
    // 정확히 한 번만 실행됨
}
```

---

### 2. 매치메이킹 시스템

#### Matchmaker 구조

```go
type MatchmakerIndex struct {
    activeIndexCount int64
    indexMu          sync.RWMutex
    indexes          map[string]*MatchmakerIndexEntry

    ticketsMu        sync.Mutex
    tickets          map[string]*MatchmakerEntry

    revThresholdMu   sync.RWMutex
    revThresholds    map[string]map[string]int64
}

type MatchmakerEntry struct {
    Ticket     string
    Properties map[string]interface{}
    PartyID    string
    StringProperties  map[string]string
    NumericProperties map[string]float64
    CreatedAt  int64
}
```

#### 매칭 알고리즘

```go
func (mi *MatchmakerIndex) Process(ctx context.Context) {
    ticker := time.NewTicker(1 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            mi.processTick()
        case <-ctx.Done():
            return
        }
    }
}

func (mi *MatchmakerIndex) processTick() {
    mi.ticketsMu.Lock()
    defer mi.ticketsMu.Unlock()

    now := time.Now().Unix()
    matches := make([][]*MatchmakerEntry, 0)

    // 매칭 가능한 티켓들 찾기
    for ticket, entry := range mi.tickets {
        if now-entry.CreatedAt > matchTimeout {
            delete(mi.tickets, ticket)
            continue
        }

        // 호환되는 다른 티켓 찾기
        compatibleEntries := mi.findCompatible(entry)

        if len(compatibleEntries) >= minPlayers {
            matches = append(matches, compatibleEntries)

            // 매칭된 티켓 제거
            for _, e := range compatibleEntries {
                delete(mi.tickets, e.Ticket)
            }
        }
    }

    // 매치 생성
    for _, matchEntries := range matches {
        go mi.createMatch(matchEntries)
    }
}

func (mi *MatchmakerIndex) findCompatible(entry *MatchmakerEntry) []*MatchmakerEntry {
    compatible := make([]*MatchmakerEntry, 0)

    for _, other := range mi.tickets {
        if entry.Ticket == other.Ticket {
            continue
        }

        // 수치 속성 매칭 (예: 레벨)
        if !mi.matchNumericProperties(entry, other) {
            continue
        }

        // 문자열 속성 매칭 (예: 지역)
        if !mi.matchStringProperties(entry, other) {
            continue
        }

        compatible = append(compatible, other)

        if len(compatible) >= maxPlayers-1 {
            break
        }
    }

    return compatible
}
```

#### 핵심 개념

**1. 시간 기반 처리**
```go
ticker := time.NewTicker(1 * time.Second)
for {
    select {
    case <-ticker.C:
        processMatchmaking()
    }
}
```
- 주기적으로 매칭 시도
- CPU 사용량 제어

**2. 속성 기반 매칭**
```go
type MatchmakerEntry struct {
    StringProperties  map[string]string    // 지역, 게임모드 등
    NumericProperties map[string]float64   // 레벨, MMR 등
}
```
- 유연한 매칭 기준
- 확장 가능한 설계

**3. 비동기 매치 생성**
```go
go mi.createMatch(matchEntries)
```
- 매칭 로직과 매치 생성 분리
- 논블로킹 처리

---

### 3. 실시간 매치 관리

#### Match Handler 구현

```go
type MatchHandler struct {
    matchID       uuid.UUID
    state         *MatchState
    dispatcher    MatchDispatcher
    tickRate      int

    joinCh        chan *joinRequest
    leaveCh       chan uuid.UUID
    messageCh     chan *MatchMessage
    stopCh        chan struct{}
}

type MatchState struct {
    players       map[uuid.UUID]*Player
    gameData      interface{}
    tickNumber    int64
}

func (m *MatchHandler) Run(ctx context.Context) {
    ticker := time.NewTicker(time.Second / time.Duration(m.tickRate))
    defer ticker.Stop()

    for {
        select {
        case req := <-m.joinCh:
            m.handleJoin(req)

        case userID := <-m.leaveCh:
            m.handleLeave(userID)

        case msg := <-m.messageCh:
            m.handleMessage(msg)

        case <-ticker.C:
            m.tick()

        case <-m.stopCh:
            m.cleanup()
            return

        case <-ctx.Done():
            m.cleanup()
            return
        }
    }
}

func (m *MatchHandler) tick() {
    m.state.tickNumber++

    // 게임 로직 실행
    m.updateGameState()

    // 클라이언트에 상태 브로드캐스트
    m.broadcastState()
}

func (m *MatchHandler) handleJoin(req *joinRequest) {
    // 참가 검증
    if len(m.state.players) >= maxPlayers {
        req.responseCh <- &joinResponse{allowed: false}
        return
    }

    // 플레이어 추가
    player := &Player{
        UserID:   req.userID,
        Username: req.username,
        Session:  req.session,
    }
    m.state.players[req.userID] = player

    // 다른 플레이어들에게 알림
    m.dispatcher.BroadcastMessage(OpCodePlayerJoined, &PlayerJoined{
        UserID:   req.userID,
        Username: req.username,
    }, nil)

    req.responseCh <- &joinResponse{allowed: true}
}

func (m *MatchHandler) handleMessage(msg *MatchMessage) {
    // 메시지 타입에 따라 처리
    switch msg.OpCode {
    case OpCodePlayerMove:
        m.handlePlayerMove(msg)
    case OpCodePlayerAction:
        m.handlePlayerAction(msg)
    default:
        // 다른 플레이어들에게 릴레이
        m.dispatcher.BroadcastMessage(msg.OpCode, msg.Data, []uuid.UUID{msg.UserID})
    }
}

func (m *MatchHandler) broadcastState() {
    state := m.state.serialize()

    m.dispatcher.BroadcastMessage(OpCodeStateUpdate, state, nil)
}
```

#### 핵심 패턴

**1. Actor 모델 패턴**
```go
// 각 매치는 독립적인 Goroutine에서 실행
go match.Run(ctx)

// Channel을 통해서만 통신
match.joinCh <- joinRequest
```
- 상태 격리
- Race Condition 방지
- 간단한 동시성 제어

**2. Tick-based 게임 루프**
```go
ticker := time.NewTicker(time.Second / 20)  // 20 TPS
for {
    case <-ticker.C:
        updateGame()
        broadcastState()
}
```
- 예측 가능한 업데이트 주기
- 클라이언트 동기화 용이

**3. Select를 활용한 멀티플렉싱**
```go
select {
case join := <-joinCh:
    handleJoin(join)
case leave := <-leaveCh:
    handleLeave(leave)
case msg := <-messageCh:
    handleMessage(msg)
case <-ticker.C:
    tick()
}
```
- 여러 이벤트 동시 처리
- 논블로킹 I/O

---

### 4. 메시지 라우팅 파이프라인

#### Pipeline 구조

```go
type MessageRouter struct {
    sessionRegistry SessionRegistry
    matchRegistry   MatchRegistry
    logger          *zap.Logger
}

func (mr *MessageRouter) ProcessMessage(session Session, envelope *rtapi.Envelope) {
    switch envelope.Message.(type) {
    case *rtapi.Envelope_MatchDataSend:
        mr.routeMatchData(session, envelope.GetMatchDataSend())

    case *rtapi.Envelope_MatchJoin:
        mr.routeMatchJoin(session, envelope.GetMatchJoin())

    case *rtapi.Envelope_MatchLeave:
        mr.routeMatchLeave(session, envelope.GetMatchLeave())

    case *rtapi.Envelope_StatusUpdate:
        mr.routeStatusUpdate(session, envelope.GetStatusUpdate())

    default:
        mr.logger.Warn("Unknown message type",
            zap.Any("envelope", envelope))
    }
}

func (mr *MessageRouter) routeMatchData(session Session, data *rtapi.MatchDataSend) {
    match := mr.matchRegistry.GetMatch(uuid.FromStringOrNil(data.MatchId))
    if match == nil {
        session.Send(&rtapi.Envelope{
            Message: &rtapi.Envelope_Error{
                Error: &rtapi.Error{
                    Code:    int32(rtapi.Error_MATCH_NOT_FOUND),
                    Message: "Match not found",
                },
            },
        })
        return
    }

    // 매치에 메시지 전달
    match.QueueData(&MatchDataMessage{
        UserID:   session.UserID(),
        Username: session.Username(),
        OpCode:   data.OpCode,
        Data:     data.Data,
    })
}
```

#### 비동기 처리

```go
type SessionHandler struct {
    session  Session
    pipeline *MessageRouter

    incomingCh chan *rtapi.Envelope
}

func (sh *SessionHandler) Start(ctx context.Context) {
    // 수신 처리 Goroutine
    go sh.processIncoming(ctx)

    // 송신 처리는 Session 내부에서
}

func (sh *SessionHandler) processIncoming(ctx context.Context) {
    for {
        select {
        case envelope := <-sh.incomingCh:
            // 각 메시지를 별도 Goroutine에서 처리
            go sh.pipeline.ProcessMessage(sh.session, envelope)

        case <-ctx.Done():
            return
        }
    }
}
```

---

## 🚀 실전 예제

### 간단한 매치 핸들러

```go
package main

import (
    "context"
    "database/sql"
    "github.com/heroiclabs/nakama-common/runtime"
)

// 게임 상태
type GameState struct {
    Players map[string]*Player
    Round   int
}

type Player struct {
    UserID   string
    Username string
    Score    int
    Position Position
}

type Position struct {
    X, Y float64
}

// Match Handler 구현
func MakeMatch(ctx context.Context, logger runtime.Logger, db *sql.DB,
               nk runtime.NakamaModule) (runtime.Match, error) {
    return &Match{
        logger: logger,
    }, nil
}

type Match struct {
    logger runtime.Logger
}

// 매치 초기화
func (m *Match) MatchInit(ctx context.Context, logger runtime.Logger,
                          db *sql.DB, nk runtime.NakamaModule,
                          params map[string]interface{}) (interface{}, int, string) {
    state := &GameState{
        Players: make(map[string]*Player),
        Round:   0,
    }

    tickRate := 10  // 10 ticks per second
    label := "skill-based-match"

    return state, tickRate, label
}

// 참가 시도
func (m *Match) MatchJoinAttempt(ctx context.Context, logger runtime.Logger,
                                  db *sql.DB, nk runtime.NakamaModule,
                                  dispatcher runtime.MatchDispatcher,
                                  tick int64, state interface{},
                                  presence runtime.Presence,
                                  metadata map[string]string) (interface{}, bool, string) {
    gameState := state.(*GameState)

    // 최대 4명까지만
    if len(gameState.Players) >= 4 {
        return state, false, "match is full"
    }

    return state, true, ""
}

// 참가 완료
func (m *Match) MatchJoin(ctx context.Context, logger runtime.Logger,
                          db *sql.DB, nk runtime.NakamaModule,
                          dispatcher runtime.MatchDispatcher,
                          tick int64, state interface{},
                          presences []runtime.Presence) interface{} {
    gameState := state.(*GameState)

    for _, presence := range presences {
        player := &Player{
            UserID:   presence.GetUserId(),
            Username: presence.GetUsername(),
            Score:    0,
            Position: Position{X: 0, Y: 0},
        }
        gameState.Players[presence.GetUserId()] = player

        logger.Info("Player joined",
            "user_id", presence.GetUserId(),
            "username", presence.GetUsername())
    }

    return gameState
}

// 매치 루프 (Tick마다 실행)
func (m *Match) MatchLoop(ctx context.Context, logger runtime.Logger,
                          db *sql.DB, nk runtime.NakamaModule,
                          dispatcher runtime.MatchDispatcher,
                          tick int64, state interface{},
                          messages []runtime.MatchData) interface{} {
    gameState := state.(*GameState)

    // 플레이어 메시지 처리
    for _, message := range messages {
        switch message.GetOpCode() {
        case 1: // Player Move
            m.handlePlayerMove(gameState, message)
        case 2: // Player Action
            m.handlePlayerAction(gameState, message)
        }
    }

    // 게임 로직 업데이트
    gameState.Round++

    // 상태 브로드캐스트 (매 10틱마다)
    if tick%10 == 0 {
        m.broadcastState(dispatcher, gameState)
    }

    return gameState
}

func (m *Match) handlePlayerMove(state *GameState, message runtime.MatchData) {
    player := state.Players[message.GetUserId()]
    if player == nil {
        return
    }

    // 위치 업데이트 (메시지 데이터 파싱 생략)
    player.Position.X += 1.0
    player.Position.Y += 1.0
}

func (m *Match) broadcastState(dispatcher runtime.MatchDispatcher, state *GameState) {
    // 상태를 JSON으로 직렬화하여 브로드캐스트
    // (실제 구현에서는 protobuf 등 사용)
    data := serializeState(state)
    dispatcher.BroadcastMessage(99, data, nil, nil, true)
}

// 매치 종료
func (m *Match) MatchTerminate(ctx context.Context, logger runtime.Logger,
                                db *sql.DB, nk runtime.NakamaModule,
                                dispatcher runtime.MatchDispatcher,
                                tick int64, state interface{},
                                graceSeconds int) interface{} {
    logger.Info("Match terminating")
    return state
}
```

### 클라이언트 연동 (Unity C# 예제)

```csharp
using Nakama;
using System.Threading.Tasks;

public class GameClient
{
    private IClient client;
    private ISocket socket;
    private IMatch currentMatch;

    public async Task ConnectAsync()
    {
        client = new Client("http", "localhost", 7350, "defaultkey");

        // 인증
        var session = await client.AuthenticateDeviceAsync(GetDeviceId());

        // 소켓 연결
        socket = client.NewSocket();
        await socket.ConnectAsync(session);

        // 매치 메시지 수신
        socket.ReceivedMatchState += OnMatchState;
    }

    public async Task JoinMatchAsync()
    {
        // 매치메이킹
        var matchmakerTicket = await socket.AddMatchmakerAsync(
            "*",              // Query
            2,                // Min players
            4,                // Max players
            new Dictionary<string, string> { { "region", "us" } }
        );

        // 매칭 완료 대기
        socket.ReceivedMatchmakerMatched += async matched =>
        {
            // 매치 참가
            currentMatch = await socket.JoinMatchAsync(matched);
        };
    }

    public async Task SendMoveAsync(float x, float y)
    {
        var data = new { x, y };
        var json = JsonUtility.ToJson(data);

        await socket.SendMatchStateAsync(
            currentMatch.Id,
            1,  // OpCode for move
            json
        );
    }

    private void OnMatchState(IMatchState matchState)
    {
        if (matchState.OpCode == 99)  // State update
        {
            var state = JsonUtility.FromJson<GameState>(
                System.Text.Encoding.UTF8.GetString(matchState.State)
            );

            UpdateGameState(state);
        }
    }
}
```

---

## 📊 성능 특성

### 동시성 패턴

**1. Goroutine per Session**
```
10,000 동시 접속 = 10,000 Goroutines
메모리: ~2KB per goroutine = 20MB
```

**2. Goroutine per Match**
```
1,000 동시 매치 = 1,000 Goroutines
CPU: Tick rate에 따라 다름 (보통 10-60 TPS)
```

### 확장성

- **단일 서버**: ~10,000 CCU (Concurrent Users)
- **수평 확장**: 무제한 (로드 밸런서 + 여러 서버)
- **데이터베이스**: CockroachDB로 분산 가능

---

## 🎓 학습 포인트

### 1. 설치 및 실행

```bash
# Docker로 실행
docker run --name nakama \
  -p 7349:7349 \
  -p 7350:7350 \
  -p 7351:7351 \
  heroiclabs/nakama:3.16.0

# 소스에서 빌드
git clone https://github.com/heroiclabs/nakama.git
cd nakama
go build -o nakama ./server
./nakama
```

### 2. 커스텀 로직 작성

```bash
# 런타임 모듈 작성 (Go)
# nakama/data/modules/example.go

package main

import (
    "context"
    "database/sql"
    "github.com/heroiclabs/nakama-common/runtime"
)

func InitModule(ctx context.Context, logger runtime.Logger,
                db *sql.DB, nk runtime.NakamaModule,
                initializer runtime.Initializer) error {

    // RPC 등록
    if err := initializer.RegisterRpc("hello", RpcHello); err != nil {
        return err
    }

    // Match Handler 등록
    if err := initializer.RegisterMatch("example", MakeMatch); err != nil {
        return err
    }

    return nil
}

func RpcHello(ctx context.Context, logger runtime.Logger,
              db *sql.DB, nk runtime.NakamaModule,
              payload string) (string, error) {
    return "Hello, " + payload, nil
}
```

---

## ⚠️ 주의사항

1. **메모리 관리**: 많은 Goroutine은 메모리를 소비함
2. **Context 사용**: 항상 context로 생명주기 관리
3. **에러 처리**: Goroutine 내부 에러도 반드시 처리
4. **데이터베이스**: 연결 풀 크기 조정 필요

---

## 🔗 관련 리소스

### 공식 문서
- GitHub: https://github.com/heroiclabs/nakama
- 문서: https://heroiclabs.com/docs/

### 클라이언트 SDK
- Unity: https://github.com/heroiclabs/nakama-unity
- Unreal: https://github.com/heroiclabs/nakama-unreal
- Godot: https://github.com/heroiclabs/nakama-godot

---

## 📚 다음 단계

1. **Nakama 로컬 실행**
2. **간단한 매치 핸들러 작성**
3. **클라이언트 연동 테스트**
4. **다음: [Colyseus](./04-colyseus.md) 분석**

---

*이 문서는 학습 목적으로 작성되었습니다.*
