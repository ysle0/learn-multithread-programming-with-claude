# 비실시간 게임 서버

## 개요

비실시간 게임 서버는 턴 기반 게임, 비동기 멀티플레이어, 소셜 기능, 웹 기반 게임 경험을 처리합니다. 실시간 서버와 달리 엄격한 tick rate나 낮은 지연 시간 동기화가 필요하지 않습니다. 대신 확장성, 신뢰성, 동시 요청의 효율적 처리에 중점을 둡니다. 이 문서에서는 비실시간 게임 서버의 아키텍처, 스레딩 모델 및 구현 전략을 다룹니다.

## 목차

1. [서버 유형](#서버-유형)
2. [스레딩 아키텍처](#스레딩-아키텍처)
3. [요청 처리 모델](#요청-처리-모델)
4. [데이터베이스 통합](#데이터베이스-통합)
5. [확장 전략](#확장-전략)
6. [사례 연구](#사례-연구)

## 서버 유형

### 턴 기반 전략 게임

**특성:**
- 이벤트 기반 아키텍처
- 엄격한 지연 시간 요구 없음
- 데이터베이스에 상태 영속화
- 비동기 게임플레이

**아키텍처:**
```
┌─────────────────────────────────────────────────┐
│      Turn-based Strategy Server                 │
├─────────────────────────────────────────────────┤
│                                                  │
│  ┌────────────────────────────────────────────┐ │
│  │         HTTP/WebSocket Server              │ │
│  │      (Nginx, Node.js, or similar)          │ │
│  └──────────────────┬─────────────────────────┘ │
│                     │                            │
│                     ▼                            │
│  ┌────────────────────────────────────────────┐ │
│  │         Load Balancer (Round Robin)        │ │
│  └─────────────┬──────────────────────────────┘ │
│                │                                 │
│    ┌───────────┼───────────┬──────────┐         │
│    │           │           │          │         │
│    ▼           ▼           ▼          ▼         │
│  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐        │
│  │ App  │  │ App  │  │ App  │  │ App  │        │
│  │Server│  │Server│  │Server│  │Server│        │
│  │  1   │  │  2   │  │  3   │  │  N   │        │
│  └──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘        │
│     │         │         │         │             │
│     └─────────┼─────────┴─────────┘             │
│               │                                  │
│               ▼                                  │
│  ┌────────────────────────────────────────────┐ │
│  │         Database Cluster                   │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐ │ │
│  │  │ Primary  │→ │Secondary │→ │Secondary │ │ │
│  │  │  (Write) │  │  (Read)  │  │  (Read)  │ │ │
│  │  └──────────┘  └──────────┘  └──────────┘ │ │
│  └────────────────────────────────────────────┘ │
│               │                                  │
│               ▼                                  │
│  ┌────────────────────────────────────────────┐ │
│  │         Cache Layer (Redis/Memcached)      │ │
│  └────────────────────────────────────────────┘ │
│               │                                  │
│               ▼                                  │
│  ┌────────────────────────────────────────────┐ │
│  │      Message Queue (RabbitMQ/SQS)          │ │
│  │  (For async notifications, matchmaking)    │ │
│  └────────────────────────────────────────────┘ │
│                                                  │
└─────────────────────────────────────────────────┘

Request Model: REST API
Concurrency: Thread pool per app server
Players per Server: 10,000+
Response Time: < 200ms
```

**구현 예시:**

```cpp
// HTTP REST API를 사용하는 턴 기반 게임 서버
class TurnBasedGameServer {
public:
    TurnBasedGameServer(int num_threads = 8)
        : io_context_(num_threads),
          thread_pool_(num_threads) {
        // 라우트 설정
        SetupRoutes();
    }

    void Start(uint16_t port) {
        // HTTP 서버 시작
        server_ = std::make_unique<HttpServer>(io_context_, port);

        // 워커 스레드 시작
        for (int i = 0; i < thread_pool_.size(); ++i) {
            thread_pool_[i] = std::thread([this]() {
                io_context_.run();
            });
        }

        std::cout << "Server started on port " << port << std::endl;
    }

    void Stop() {
        io_context_.stop();
        for (auto& thread : thread_pool_) {
            if (thread.joinable()) thread.join();
        }
    }

private:
    void SetupRoutes() {
        // POST /api/game/create
        server_->RegisterRoute("POST", "/api/game/create",
            [this](const Request& req, Response& res) {
                HandleCreateGame(req, res);
            });

        // POST /api/game/{game_id}/move
        server_->RegisterRoute("POST", "/api/game/{game_id}/move",
            [this](const Request& req, Response& res) {
                HandleMove(req, res);
            });

        // GET /api/game/{game_id}/state
        server_->RegisterRoute("GET", "/api/game/{game_id}/state",
            [this](const Request& req, Response& res) {
                HandleGetGameState(req, res);
            });

        // GET /api/player/{player_id}/games
        server_->RegisterRoute("GET", "/api/player/{player_id}/games",
            [this](const Request& req, Response& res) {
                HandleGetPlayerGames(req, res);
            });
    }

    void HandleCreateGame(const Request& req, Response& res) {
        try {
            // 요청 본문 파싱
            auto body = json::parse(req.body);
            std::string player1_id = body["player1_id"];
            std::string player2_id = body["player2_id"];
            std::string game_type = body["game_type"];

            // 플레이어 존재 여부 확인
            if (!ValidatePlayer(player1_id) || !ValidatePlayer(player2_id)) {
                res.status = 400;
                res.body = R"({"error": "Invalid player ID"})";
                return;
            }

            // 새 게임 생성
            std::string game_id = GenerateGameId();

            // 게임 상태 초기화
            GameState initial_state = InitializeGameState(game_type);

            // 데이터베이스에 저장 (비동기)
            auto db_future = std::async(std::launch::async, [&]() {
                return database_.CreateGame(
                    game_id,
                    player1_id,
                    player2_id,
                    game_type,
                    initial_state
                );
            });

            // 데이터베이스 작업 대기 (타임아웃 포함)
            auto status = db_future.wait_for(std::chrono::seconds(5));
            if (status == std::future_status::timeout) {
                res.status = 503;
                res.body = R"({"error": "Database timeout"})";
                return;
            }

            bool success = db_future.get();
            if (!success) {
                res.status = 500;
                res.body = R"({"error": "Failed to create game"})";
                return;
            }

            // 초기 상태 캐시
            cache_.Set("game:" + game_id, initial_state.ToJson());

            // 양 플레이어에게 알림 전송 (비동기)
            message_queue_.Publish("game.created", {
                {"game_id", game_id},
                {"player1_id", player1_id},
                {"player2_id", player2_id}
            });

            // 응답 반환
            res.status = 201;
            res.body = json({
                {"game_id", game_id},
                {"state", initial_state.ToJson()}
            }).dump();

        } catch (const std::exception& e) {
            res.status = 500;
            res.body = json({{"error", e.what()}}).dump();
        }
    }

    void HandleMove(const Request& req, Response& res) {
        try {
            std::string game_id = req.params["game_id"];
            auto body = json::parse(req.body);

            std::string player_id = body["player_id"];
            json move_data = body["move"];

            // 캐시에서 게임 상태를 먼저 가져오기 시도
            std::optional<std::string> cached_state = cache_.Get("game:" + game_id);

            GameState state;
            if (cached_state.has_value()) {
                // 캐시 히트
                state = GameState::FromJson(cached_state.value());
            } else {
                // 캐시 미스 - 데이터베이스에서 로드
                auto db_state = database_.GetGameState(game_id);
                if (!db_state.has_value()) {
                    res.status = 404;
                    res.body = R"({"error": "Game not found"})";
                    return;
                }
                state = db_state.value();
            }

            // 플레이어의 턴인지 확인
            if (state.current_player_id != player_id) {
                res.status = 403;
                res.body = R"({"error": "Not your turn"})";
                return;
            }

            // 수 검증
            if (!state.ValidateMove(move_data)) {
                res.status = 400;
                res.body = R"({"error": "Invalid move"})";
                return;
            }

            // 수 적용
            state.ApplyMove(move_data);

            // 게임 종료 확인
            bool game_ended = state.IsGameEnded();
            if (game_ended) {
                state.DetermineWinner();
            }

            // 데이터베이스 업데이트 (future를 사용한 비동기)
            auto update_future = std::async(std::launch::async, [&]() {
                database_.UpdateGameState(game_id, state);

                // 이동 기록에 수 기록
                database_.RecordMove(game_id, player_id, move_data, state.turn_number);

                return true;
            });

            // 캐시 업데이트
            cache_.Set("game:" + game_id, state.ToJson(), 3600); // 1시간 TTL

            // 데이터베이스 업데이트 대기
            update_future.wait();

            // 상대방에게 알림
            std::string opponent_id = state.GetOpponentId(player_id);
            message_queue_.Publish("game.move", {
                {"game_id", game_id},
                {"player_id", player_id},
                {"opponent_id", opponent_id},
                {"move", move_data}
            });

            if (game_ended) {
                message_queue_.Publish("game.ended", {
                    {"game_id", game_id},
                    {"winner_id", state.winner_id}
                });
            }

            // 업데이트된 상태 반환
            res.status = 200;
            res.body = json({
                {"state", state.ToJson()},
                {"game_ended", game_ended}
            }).dump();

        } catch (const std::exception& e) {
            res.status = 500;
            res.body = json({{"error", e.what()}}).dump();
        }
    }

    void HandleGetGameState(const Request& req, Response& res) {
        try {
            std::string game_id = req.params["game_id"];

            // 캐시를 먼저 확인
            auto cached_state = cache_.Get("game:" + game_id);
            if (cached_state.has_value()) {
                res.status = 200;
                res.body = cached_state.value();
                res.headers["X-Cache"] = "HIT";
                return;
            }

            // 캐시 미스 - 데이터베이스 조회
            auto state = database_.GetGameState(game_id);
            if (!state.has_value()) {
                res.status = 404;
                res.body = R"({"error": "Game not found"})";
                return;
            }

            std::string state_json = state.value().ToJson();

            // 캐시 업데이트
            cache_.Set("game:" + game_id, state_json, 3600);

            res.status = 200;
            res.body = state_json;
            res.headers["X-Cache"] = "MISS";

        } catch (const std::exception& e) {
            res.status = 500;
            res.body = json({{"error", e.what()}}).dump();
        }
    }

private:
    boost::asio::io_context io_context_;
    std::vector<std::thread> thread_pool_;
    std::unique_ptr<HttpServer> server_;

    // 데이터베이스 커넥션 풀
    DatabaseConnectionPool database_{10}; // 10개 연결

    // 캐시 클라이언트
    RedisClient cache_;

    // 메시지 큐
    RabbitMQClient message_queue_;
};
```

### 카드 게임 / 덱 빌더

**구현 예시:**

```cpp
// 매치메이킹이 포함된 카드 게임 서버
class CardGameServer {
public:
    void Start() {
        // 매치메이킹 스레드
        matchmaking_thread_ = std::thread([this]() {
            MatchmakingLoop();
        });

        // 게임 룸 관리 스레드
        for (int i = 0; i < 4; ++i) {
            room_threads_.emplace_back([this]() {
                GameRoomLoop();
            });
        }

        // REST API 서버
        api_server_.Start(8080);
    }

private:
    void MatchmakingLoop() {
        while (running_) {
            // 대기 중인 플레이어 가져오기
            auto waiting_players = GetWaitingPlayers();

            if (waiting_players.size() >= 2) {
                // 레이팅 기반으로 플레이어 매칭
                auto matches = CreateMatches(waiting_players);

                for (auto& match : matches) {
                    // 게임 룸 생성
                    std::string room_id = CreateGameRoom(match);

                    // 플레이어에게 알림
                    for (auto player_id : match.player_ids) {
                        NotifyPlayerMatchFound(player_id, room_id);
                    }
                }
            }

            // 다음 매치메이킹 반복 전 sleep
            std::this_thread::sleep_for(std::chrono::seconds(1));
        }
    }

    void GameRoomLoop() {
        while (running_) {
            // 처리할 다음 게임 룸 가져오기
            auto room = pending_rooms_.pop();

            // 완료될 때까지 게임 실행
            RunGame(room);

            // 플레이어 통계 업데이트
            UpdatePlayerStats(room.player_ids, room.results);

            // 룸 정리
            CleanupRoom(room.room_id);
        }
    }

    void RunGame(GameRoom& room) {
        // 게임 상태 초기화
        GameState state = InitializeCardGame(room);

        // 게임 루프
        while (!state.IsGameEnded()) {
            // 현재 플레이어의 액션 대기
            auto action = WaitForPlayerAction(
                state.current_player_id,
                std::chrono::seconds(30) // 30초 타임아웃
            );

            if (!action.has_value()) {
                // 타임아웃 - 플레이어 패배
                state.DeclareWinner(state.GetOpponentId(state.current_player_id));
                break;
            }

            // 액션 검증
            if (!state.ValidateAction(action.value())) {
                // 유효하지 않은 액션 - 플레이어에게 오류 전송
                SendError(state.current_player_id, "Invalid action");
                continue;
            }

            // 액션 적용
            state.ApplyAction(action.value());

            // 양 플레이어에게 상태 업데이트 방송
            BroadcastGameState(room, state);

            // 게임 종료 확인
            if (state.IsGameEnded()) {
                state.DetermineWinner();
                room.results = state.GetResults();
            }
        }

        // 플레이어에게 게임 종료 알림
        NotifyGameEnd(room);
    }

    std::optional<Action> WaitForPlayerAction(
        const std::string& player_id,
        std::chrono::seconds timeout) {

        // 타임아웃과 함께 액션 대기
        std::unique_lock<std::mutex> lock(action_mutex_);
        auto result = action_cv_.wait_for(lock, timeout, [&]() {
            return player_actions_.count(player_id) > 0;
        });

        if (!result) {
            return std::nullopt; // 타임아웃
        }

        // 액션 가져오기 및 제거
        auto action = player_actions_[player_id];
        player_actions_.erase(player_id);

        return action;
    }

    void HandlePlayerAction(const std::string& player_id, const Action& action) {
        {
            std::lock_guard<std::mutex> lock(action_mutex_);
            player_actions_[player_id] = action;
        }
        action_cv_.notify_all();
    }

private:
    std::atomic<bool> running_{true};

    // 스레딩
    std::thread matchmaking_thread_;
    std::vector<std::thread> room_threads_;

    // 게임 룸
    ConcurrentQueue<GameRoom> pending_rooms_;
    std::unordered_map<std::string, GameRoom> active_rooms_;

    // 플레이어 액션
    std::mutex action_mutex_;
    std::condition_variable action_cv_;
    std::unordered_map<std::string, Action> player_actions_;

    // API 서버
    RestAPIServer api_server_;
};
```

### 소셜 / 캐주얼 게임

**아키텍처:**
```
┌─────────────────────────────────────────────────────┐
│         Social Game Server Architecture              │
├─────────────────────────────────────────────────────┤
│                                                       │
│  ┌─────────────────────────────────────────────────┐│
│  │              CDN (Static Assets)                 ││
│  │  • Game client (HTML/JS/CSS)                     ││
│  │  • Images, sounds, sprites                       ││
│  └─────────────────────────────────────────────────┘│
│                                                       │
│  ┌─────────────────────────────────────────────────┐│
│  │         API Gateway (GraphQL/REST)               ││
│  │  • Authentication (JWT/OAuth)                    ││
│  │  • Rate limiting                                 ││
│  │  • Request validation                            ││
│  └─────────────┬───────────────────────────────────┘│
│                │                                      │
│    ┌───────────┼───────────┬────────────┐            │
│    │           │           │            │            │
│    ▼           ▼           ▼            ▼            │
│  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────────────┐    │
│  │Player│  │Social│  │ Game │  │ Monetization │    │
│  │ Svc  │  │ Svc  │  │ Logic│  │   Service    │    │
│  │      │  │      │  │  Svc │  │              │    │
│  └──┬───┘  └──┬───┘  └──┬───┘  └──────┬───────┘    │
│     │         │         │             │             │
│     └─────────┴─────────┴─────────────┘             │
│                         │                            │
│                         ▼                            │
│  ┌─────────────────────────────────────────────────┐│
│  │         Event Bus (Pub/Sub)                      ││
│  │  • Player events                                 ││
│  │  • Social events (friend requests, messages)    ││
│  │  • Game events                                   ││
│  └─────────────────────────────────────────────────┘│
│                         │                            │
│                         ▼                            │
│  ┌─────────────────────────────────────────────────┐│
│  │         Data Layer                               ││
│  │  • PostgreSQL (transactional data)              ││
│  │  • MongoDB (game state, player data)            ││
│  │  • Redis (sessions, leaderboards)               ││
│  │  • S3 (file storage)                            ││
│  └─────────────────────────────────────────────────┘│
│                                                       │
└─────────────────────────────────────────────────────┘
```

**구현 예시 (마이크로서비스):**

```cpp
// 플레이어 서비스 - 플레이어 프로필과 진행도 처리
class PlayerService {
public:
    PlayerService(int num_threads = 4)
        : thread_pool_(num_threads) {}

    // gRPC 서비스 메서드
    grpc::Status GetPlayerProfile(
        grpc::ServerContext* context,
        const GetPlayerProfileRequest* request,
        PlayerProfile* response) {

        std::string player_id = request->player_id();

        // 캐시를 먼저 확인
        auto cached = cache_.Get("player:" + player_id);
        if (cached.has_value()) {
            response->ParseFromString(cached.value());
            return grpc::Status::OK;
        }

        // 데이터베이스에서 로드
        auto profile = database_.GetPlayerProfile(player_id);
        if (!profile.has_value()) {
            return grpc::Status(grpc::StatusCode::NOT_FOUND, "Player not found");
        }

        *response = profile.value();

        // 캐시 업데이트
        std::string serialized;
        response->SerializeToString(&serialized);
        cache_.Set("player:" + player_id, serialized, 600); // 10분 TTL

        return grpc::Status::OK;
    }

    grpc::Status UpdatePlayerProgress(
        grpc::ServerContext* context,
        const UpdateProgressRequest* request,
        UpdateProgressResponse* response) {

        std::string player_id = request->player_id();
        int xp_gained = request->xp_gained();
        int coins_earned = request->coins_earned();

        // 경쟁 조건 방지를 위해 낙관적 잠금 사용
        int max_retries = 3;
        for (int retry = 0; retry < max_retries; ++retry) {
            // 현재 상태 로드
            auto profile = database_.GetPlayerProfile(player_id);
            if (!profile.has_value()) {
                return grpc::Status(grpc::StatusCode::NOT_FOUND, "Player not found");
            }

            int current_version = profile->version();

            // 업데이트 적용
            profile->set_xp(profile->xp() + xp_gained);
            profile->set_coins(profile->coins() + coins_earned);

            // 레벨업 확인
            int new_level = CalculateLevel(profile->xp());
            if (new_level > profile->level()) {
                profile->set_level(new_level);

                // 레벨업 이벤트 발행
                PublishEvent("player.levelup", {
                    {"player_id", player_id},
                    {"new_level", new_level}
                });
            }

            profile->set_version(current_version + 1);

            // 버전 확인과 함께 업데이트 시도 (낙관적 잠금)
            bool success = database_.UpdatePlayerProfileIfVersion(
                player_id,
                *profile,
                current_version
            );

            if (success) {
                *response->mutable_profile() = *profile;

                // 캐시 무효화
                cache_.Delete("player:" + player_id);

                return grpc::Status::OK;
            }

            // 버전 충돌 - 재시도
            if (retry == max_retries - 1) {
                return grpc::Status(
                    grpc::StatusCode::ABORTED,
                    "Too many concurrent updates"
                );
            }

            // 지수 백오프
            std::this_thread::sleep_for(
                std::chrono::milliseconds(10 * (1 << retry))
            );
        }

        return grpc::Status(grpc::StatusCode::INTERNAL, "Update failed");
    }

private:
    ThreadPool thread_pool_;
    DatabaseClient database_;
    RedisClient cache_;
    EventBus event_bus_;
};
```

## 요청 처리 모델

### 1. Thread Pool 모델
```cpp
// 요청 처리를 위한 고정 크기 thread pool
class ThreadPoolServer {
    boost::asio::io_context io_context_;
    boost::asio::thread_pool thread_pool_;

public:
    ThreadPoolServer(size_t pool_size)
        : thread_pool_(pool_size) {}

    void HandleRequest(Request req) {
        // thread pool에 작업 게시
        boost::asio::post(thread_pool_, [this, req = std::move(req)]() {
            ProcessRequest(req);
        });
    }

    void ProcessRequest(const Request& req) {
        try {
            // 요청 파싱
            auto params = ParseRequest(req);

            // 비즈니스 로직 실행
            auto result = ExecuteLogic(params);

            // 응답 전송
            SendResponse(req.connection_id, result);

        } catch (const std::exception& e) {
            SendError(req.connection_id, e.what());
        }
    }
};
```

### 2. 비동기 I/O 모델 (코루틴)
```cpp
// C++20 코루틴을 사용한 비동기 I/O
class AsyncServer {
public:
    Task<void> HandleRequest(Request req) {
        try {
            // 비동기 데이터베이스 조회
            auto player_data = co_await database_.GetPlayerAsync(req.player_id);

            // 비동기 캐시 업데이트
            co_await cache_.SetAsync("player:" + req.player_id, player_data);

            // 비동기 응답
            co_await SendResponseAsync(req.connection_id, player_data);

        } catch (const std::exception& e) {
            co_await SendErrorAsync(req.connection_id, e.what());
        }
    }

private:
    AsyncDatabaseClient database_;
    AsyncRedisClient cache_;
};
```

### 3. Actor 모델 (메시지 전달)
```cpp
// Actor 기반 요청 처리
class PlayerActor {
    std::string player_id_;
    std::queue<Message> mailbox_;
    std::mutex mailbox_mutex_;
    std::condition_variable mailbox_cv_;
    std::thread actor_thread_;

public:
    PlayerActor(std::string player_id)
        : player_id_(std::move(player_id)) {
        actor_thread_ = std::thread([this]() { Run(); });
    }

    void Send(Message msg) {
        {
            std::lock_guard<std::mutex> lock(mailbox_mutex_);
            mailbox_.push(std::move(msg));
        }
        mailbox_cv_.notify_one();
    }

private:
    void Run() {
        while (true) {
            Message msg;
            {
                std::unique_lock<std::mutex> lock(mailbox_mutex_);
                mailbox_cv_.wait(lock, [this]() {
                    return !mailbox_.empty();
                });
                msg = std::move(mailbox_.front());
                mailbox_.pop();
            }

            // 메시지 처리
            HandleMessage(msg);
        }
    }

    void HandleMessage(const Message& msg) {
        switch (msg.type) {
            case MessageType::GetState:
                SendState(msg.sender);
                break;
            case MessageType::UpdateState:
                UpdateState(msg.data);
                break;
            // ... 기타 메시지 유형
        }
    }
};
```

## 데이터베이스 통합

### 커넥션 풀링
```cpp
class DatabaseConnectionPool {
    std::queue<std::unique_ptr<DatabaseConnection>> available_;
    std::mutex mutex_;
    std::condition_variable cv_;
    size_t pool_size_;

public:
    DatabaseConnectionPool(size_t pool_size, const std::string& connection_string)
        : pool_size_(pool_size) {
        // 초기 연결 생성
        for (size_t i = 0; i < pool_size; ++i) {
            available_.push(std::make_unique<DatabaseConnection>(connection_string));
        }
    }

    std::unique_ptr<DatabaseConnection> Acquire() {
        std::unique_lock<std::mutex> lock(mutex_);
        cv_.wait(lock, [this]() { return !available_.empty(); });

        auto conn = std::move(available_.front());
        available_.pop();
        return conn;
    }

    void Release(std::unique_ptr<DatabaseConnection> conn) {
        {
            std::lock_guard<std::mutex> lock(mutex_);
            available_.push(std::move(conn));
        }
        cv_.notify_one();
    }
};

// 자동 연결 반환을 위한 RAII 래퍼
class ScopedConnection {
    DatabaseConnectionPool& pool_;
    std::unique_ptr<DatabaseConnection> conn_;

public:
    ScopedConnection(DatabaseConnectionPool& pool)
        : pool_(pool), conn_(pool.Acquire()) {}

    ~ScopedConnection() {
        if (conn_) {
            pool_.Release(std::move(conn_));
        }
    }

    DatabaseConnection* operator->() { return conn_.get(); }
};

// 사용 예시
void HandleRequest(const Request& req) {
    ScopedConnection conn(db_pool_);
    auto result = conn->Query("SELECT * FROM players WHERE id = ?", req.player_id);
    // 스코프를 벗어나면 연결이 자동으로 반환됨
}
```

## 확장 전략

### 수평 확장
```
┌───────────────────────────────────────┐
│         Load Balancer (Nginx)         │
└─────────────┬─────────────────────────┘
              │
    ┌─────────┼─────────┬─────────┐
    │         │         │         │
    ▼         ▼         ▼         ▼
┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
│ App    │ │ App    │ │ App    │ │ App    │
│Server 1│ │Server 2│ │Server 3│ │Server N│
└────┬───┘ └────┬───┘ └────┬───┘ └────┬───┘
     │          │          │          │
     └──────────┴──────────┴──────────┘
                │
                ▼
        ┌───────────────┐
        │   Database    │
        │   Cluster     │
        └───────────────┘
```

### Stateless 서비스
```cpp
// Stateless 서비스 - 수평 확장 가능
class StatelessGameService {
public:
    Response HandleGameAction(const Request& req) {
        // 1. 데이터베이스/캐시에서 상태 로드
        auto game_state = LoadGameState(req.game_id);

        // 2. 액션 검증
        if (!game_state.ValidateAction(req.action)) {
            return Response::Error("Invalid action");
        }

        // 3. 액션 적용
        game_state.ApplyAction(req.action);

        // 4. 상태 저장
        SaveGameState(req.game_id, game_state);

        // 5. 결과 반환
        return Response::Success(game_state);
    }

private:
    GameState LoadGameState(const std::string& game_id) {
        // 캐시를 먼저 확인
        auto cached = cache_.Get("game:" + game_id);
        if (cached.has_value()) {
            return GameState::Deserialize(cached.value());
        }

        // 데이터베이스에서 로드
        auto state = database_.GetGameState(game_id);

        // 캐시 업데이트
        cache_.Set("game:" + game_id, state.Serialize());

        return state;
    }
};
```

## 사례 연구

### 하스스톤 (Blizzard)

**아키텍처:**
- 게임 클라이언트용 REST API
- 턴 기반 게임플레이
- 클라이언트-서버 모델 (P2P 없음)
- 결정론적 게임 로직
- 서버 권위적

**핵심 기술:**
- Stateless 애플리케이션 서버
- 영구 저장을 위한 MySQL
- 캐싱과 매치메이킹을 위한 Redis
- 비동기 처리를 위한 RabbitMQ

**확장:**
- API 서버의 수평 확장
- 데이터베이스 읽기 복제본
- 정적 자산을 위한 CDN
- 지역별 배포

### 클래시 오브 클랜 (Supercell)

**아키텍처:**
- 비동기 멀티플레이어
- 클라이언트-서버 모델
- 공격 리플레이 저장 및 재생
- 서버가 모든 액션 검증

**핵심 기능:**
- 자원 업데이트를 위한 낙관적 잠금
- 공격 리플레이를 위한 이벤트 소싱
- 큐 기반 비동기 멀티플레이어
- 서버 동기화가 포함된 오프라인 게임플레이

### 워드 위드 프렌즈 (Zynga)

**아키텍처:**
- 턴 기반 단어 게임
- 턴 알림을 위한 푸시 알림
- 소셜 기능 (채팅, 친구)
- 크로스 플랫폼 (모바일, 웹)

**기술 스택:**
- API 서버를 위한 Node.js
- 게임 상태를 위한 MongoDB
- 세션과 캐싱을 위한 Redis
- 푸시 알림을 위한 AWS SNS

**확장:**
- 마이크로서비스 아키텍처
- 부하 기반 자동 확장
- 다중 리전 배포
- 사용자 ID 기반 데이터베이스 sharding

## 성능 지표

### 목표 SLA

```
Metric                   | Target     | Acceptable
-------------------------|------------|------------
API Response Time (p50)  | < 50ms     | < 100ms
API Response Time (p99)  | < 200ms    | < 500ms
Database Query (p50)     | < 10ms     | < 50ms
Cache Hit Rate           | > 80%      | > 60%
Throughput               | 10K req/s  | 5K req/s
Uptime                   | 99.9%      | 99.5%
```

### 모니터링
```cpp
class MetricsCollector {
public:
    void RecordRequest(const std::string& endpoint, int latency_ms) {
        histogram_[endpoint].Observe(latency_ms);
        counter_[endpoint]++;
    }

    void RecordDatabaseQuery(int latency_ms) {
        db_latency_histogram_.Observe(latency_ms);
    }

    void RecordCacheHit(bool hit) {
        if (hit) {
            cache_hits_++;
        } else {
            cache_misses_++;
        }
    }

    double GetCacheHitRate() const {
        long total = cache_hits_ + cache_misses_;
        if (total == 0) return 0.0;
        return static_cast<double>(cache_hits_) / total;
    }

private:
    std::unordered_map<std::string, Histogram> histogram_;
    std::unordered_map<std::string, Counter> counter_;
    Histogram db_latency_histogram_;
    std::atomic<long> cache_hits_{0};
    std::atomic<long> cache_misses_{0};
};
```

## 결론

비실시간 게임 서버는 엄격한 지연 시간 요구사항보다 확장성, 신뢰성, 개발자 생산성을 우선시합니다. 현대 아키텍처는 stateless 서비스, 캐싱 계층, 메시지 큐, 데이터베이스 최적화를 사용하여 수백만 명의 동시 접속 플레이어를 처리합니다. 모놀리식과 마이크로서비스 아키텍처 사이의 선택은 팀 규모, 복잡성, 확장성 요구사항에 따라 달라집니다.

## 추가 참고 자료

- [AWS Game Tech Blog](https://aws.amazon.com/blogs/gametech/)
- [Scaling to Millions of Simultaneous Connections](https://www.slideshare.net/slideshow/scaling-to-millions-of-simultaneous-connections/17820191)
- [Microservices for Gaming](https://cloud.google.com/blog/products/gaming/building-a-multiplayer-game-with-microservices)
- [Database Sharding Strategies](https://www.digitalocean.com/community/tutorials/understanding-database-sharding)
