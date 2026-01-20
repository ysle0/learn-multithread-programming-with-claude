# Non-Real-time Game Servers

## Overview

Non-real-time game servers handle turn-based games, asynchronous multiplayer, social features, and web-based gaming experiences. Unlike real-time servers, they don't require strict tick rates or low-latency synchronization. Instead, they focus on scalability, reliability, and efficient handling of concurrent requests. This document covers architectures, threading models, and implementation strategies for non-real-time game servers.

## Table of Contents

1. [Server Types](#server-types)
2. [Threading Architecture](#threading-architecture)
3. [Request Processing Models](#request-processing-models)
4. [Database Integration](#database-integration)
5. [Scaling Strategies](#scaling-strategies)
6. [Case Studies](#case-studies)

## Server Types

### Turn-based Strategy Games

**Characteristics:**
- Event-driven architecture
- No strict latency requirements
- State persisted in database
- Asynchronous gameplay

**Architecture:**
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

**Implementation Example:**

```cpp
// Turn-based game server using HTTP REST API
class TurnBasedGameServer {
public:
    TurnBasedGameServer(int num_threads = 8)
        : io_context_(num_threads),
          thread_pool_(num_threads) {
        // Setup routes
        SetupRoutes();
    }

    void Start(uint16_t port) {
        // Start HTTP server
        server_ = std::make_unique<HttpServer>(io_context_, port);

        // Start worker threads
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
            // Parse request body
            auto body = json::parse(req.body);
            std::string player1_id = body["player1_id"];
            std::string player2_id = body["player2_id"];
            std::string game_type = body["game_type"];

            // Validate players exist
            if (!ValidatePlayer(player1_id) || !ValidatePlayer(player2_id)) {
                res.status = 400;
                res.body = R"({"error": "Invalid player ID"})";
                return;
            }

            // Create new game
            std::string game_id = GenerateGameId();

            // Initialize game state
            GameState initial_state = InitializeGameState(game_type);

            // Save to database (async)
            auto db_future = std::async(std::launch::async, [&]() {
                return database_.CreateGame(
                    game_id,
                    player1_id,
                    player2_id,
                    game_type,
                    initial_state
                );
            });

            // Wait for database operation (with timeout)
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

            // Cache initial state
            cache_.Set("game:" + game_id, initial_state.ToJson());

            // Send notifications to both players (async)
            message_queue_.Publish("game.created", {
                {"game_id", game_id},
                {"player1_id", player1_id},
                {"player2_id", player2_id}
            });

            // Return response
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

            // Try to get game state from cache first
            std::optional<std::string> cached_state = cache_.Get("game:" + game_id);

            GameState state;
            if (cached_state.has_value()) {
                // Cache hit
                state = GameState::FromJson(cached_state.value());
            } else {
                // Cache miss - load from database
                auto db_state = database_.GetGameState(game_id);
                if (!db_state.has_value()) {
                    res.status = 404;
                    res.body = R"({"error": "Game not found"})";
                    return;
                }
                state = db_state.value();
            }

            // Validate it's player's turn
            if (state.current_player_id != player_id) {
                res.status = 403;
                res.body = R"({"error": "Not your turn"})";
                return;
            }

            // Validate move
            if (!state.ValidateMove(move_data)) {
                res.status = 400;
                res.body = R"({"error": "Invalid move"})";
                return;
            }

            // Apply move
            state.ApplyMove(move_data);

            // Check for game end
            bool game_ended = state.IsGameEnded();
            if (game_ended) {
                state.DetermineWinner();
            }

            // Update database (async with future)
            auto update_future = std::async(std::launch::async, [&]() {
                database_.UpdateGameState(game_id, state);

                // Record move in history
                database_.RecordMove(game_id, player_id, move_data, state.turn_number);

                return true;
            });

            // Update cache
            cache_.Set("game:" + game_id, state.ToJson(), 3600); // 1 hour TTL

            // Wait for database update
            update_future.wait();

            // Notify opponent
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

            // Return updated state
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

            // Try cache first
            auto cached_state = cache_.Get("game:" + game_id);
            if (cached_state.has_value()) {
                res.status = 200;
                res.body = cached_state.value();
                res.headers["X-Cache"] = "HIT";
                return;
            }

            // Cache miss - query database
            auto state = database_.GetGameState(game_id);
            if (!state.has_value()) {
                res.status = 404;
                res.body = R"({"error": "Game not found"})";
                return;
            }

            std::string state_json = state.value().ToJson();

            // Update cache
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

    // Database connection pool
    DatabaseConnectionPool database_{10}; // 10 connections

    // Cache client
    RedisClient cache_;

    // Message queue
    RabbitMQClient message_queue_;
};
```

### Card Games / Deck Builders

**Implementation Example:**

```cpp
// Card game server with matchmaking
class CardGameServer {
public:
    void Start() {
        // Matchmaking thread
        matchmaking_thread_ = std::thread([this]() {
            MatchmakingLoop();
        });

        // Game room manager threads
        for (int i = 0; i < 4; ++i) {
            room_threads_.emplace_back([this]() {
                GameRoomLoop();
            });
        }

        // REST API server
        api_server_.Start(8080);
    }

private:
    void MatchmakingLoop() {
        while (running_) {
            // Get waiting players
            auto waiting_players = GetWaitingPlayers();

            if (waiting_players.size() >= 2) {
                // Match players based on rating
                auto matches = CreateMatches(waiting_players);

                for (auto& match : matches) {
                    // Create game room
                    std::string room_id = CreateGameRoom(match);

                    // Notify players
                    for (auto player_id : match.player_ids) {
                        NotifyPlayerMatchFound(player_id, room_id);
                    }
                }
            }

            // Sleep before next matchmaking iteration
            std::this_thread::sleep_for(std::chrono::seconds(1));
        }
    }

    void GameRoomLoop() {
        while (running_) {
            // Get next game room to process
            auto room = pending_rooms_.pop();

            // Run game until completion
            RunGame(room);

            // Update player stats
            UpdatePlayerStats(room.player_ids, room.results);

            // Clean up room
            CleanupRoom(room.room_id);
        }
    }

    void RunGame(GameRoom& room) {
        // Initialize game state
        GameState state = InitializeCardGame(room);

        // Game loop
        while (!state.IsGameEnded()) {
            // Wait for current player's action
            auto action = WaitForPlayerAction(
                state.current_player_id,
                std::chrono::seconds(30) // 30s timeout
            );

            if (!action.has_value()) {
                // Timeout - player loses
                state.DeclareWinner(state.GetOpponentId(state.current_player_id));
                break;
            }

            // Validate action
            if (!state.ValidateAction(action.value())) {
                // Invalid action - send error to player
                SendError(state.current_player_id, "Invalid action");
                continue;
            }

            // Apply action
            state.ApplyAction(action.value());

            // Broadcast state update to both players
            BroadcastGameState(room, state);

            // Check for game end
            if (state.IsGameEnded()) {
                state.DetermineWinner();
                room.results = state.GetResults();
            }
        }

        // Notify players of game end
        NotifyGameEnd(room);
    }

    std::optional<Action> WaitForPlayerAction(
        const std::string& player_id,
        std::chrono::seconds timeout) {

        // Wait for action with timeout
        std::unique_lock<std::mutex> lock(action_mutex_);
        auto result = action_cv_.wait_for(lock, timeout, [&]() {
            return player_actions_.count(player_id) > 0;
        });

        if (!result) {
            return std::nullopt; // Timeout
        }

        // Get and remove action
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

    // Threading
    std::thread matchmaking_thread_;
    std::vector<std::thread> room_threads_;

    // Game rooms
    ConcurrentQueue<GameRoom> pending_rooms_;
    std::unordered_map<std::string, GameRoom> active_rooms_;

    // Player actions
    std::mutex action_mutex_;
    std::condition_variable action_cv_;
    std::unordered_map<std::string, Action> player_actions_;

    // API server
    RestAPIServer api_server_;
};
```

### Social / Casual Games

**Architecture:**
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

**Implementation Example (Microservices):**

```cpp
// Player service - handles player profiles and progression
class PlayerService {
public:
    PlayerService(int num_threads = 4)
        : thread_pool_(num_threads) {}

    // gRPC service methods
    grpc::Status GetPlayerProfile(
        grpc::ServerContext* context,
        const GetPlayerProfileRequest* request,
        PlayerProfile* response) {

        std::string player_id = request->player_id();

        // Try cache first
        auto cached = cache_.Get("player:" + player_id);
        if (cached.has_value()) {
            response->ParseFromString(cached.value());
            return grpc::Status::OK;
        }

        // Load from database
        auto profile = database_.GetPlayerProfile(player_id);
        if (!profile.has_value()) {
            return grpc::Status(grpc::StatusCode::NOT_FOUND, "Player not found");
        }

        *response = profile.value();

        // Update cache
        std::string serialized;
        response->SerializeToString(&serialized);
        cache_.Set("player:" + player_id, serialized, 600); // 10 min TTL

        return grpc::Status::OK;
    }

    grpc::Status UpdatePlayerProgress(
        grpc::ServerContext* context,
        const UpdateProgressRequest* request,
        UpdateProgressResponse* response) {

        std::string player_id = request->player_id();
        int xp_gained = request->xp_gained();
        int coins_earned = request->coins_earned();

        // Use optimistic locking to prevent race conditions
        int max_retries = 3;
        for (int retry = 0; retry < max_retries; ++retry) {
            // Load current state
            auto profile = database_.GetPlayerProfile(player_id);
            if (!profile.has_value()) {
                return grpc::Status(grpc::StatusCode::NOT_FOUND, "Player not found");
            }

            int current_version = profile->version();

            // Apply updates
            profile->set_xp(profile->xp() + xp_gained);
            profile->set_coins(profile->coins() + coins_earned);

            // Check for level up
            int new_level = CalculateLevel(profile->xp());
            if (new_level > profile->level()) {
                profile->set_level(new_level);

                // Publish level up event
                PublishEvent("player.levelup", {
                    {"player_id", player_id},
                    {"new_level", new_level}
                });
            }

            profile->set_version(current_version + 1);

            // Try to update with version check (optimistic locking)
            bool success = database_.UpdatePlayerProfileIfVersion(
                player_id,
                *profile,
                current_version
            );

            if (success) {
                *response->mutable_profile() = *profile;

                // Invalidate cache
                cache_.Delete("player:" + player_id);

                return grpc::Status::OK;
            }

            // Version conflict - retry
            if (retry == max_retries - 1) {
                return grpc::Status(
                    grpc::StatusCode::ABORTED,
                    "Too many concurrent updates"
                );
            }

            // Exponential backoff
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

## Request Processing Models

### 1. Thread Pool Model
```cpp
// Fixed-size thread pool for handling requests
class ThreadPoolServer {
    boost::asio::io_context io_context_;
    boost::asio::thread_pool thread_pool_;

public:
    ThreadPoolServer(size_t pool_size)
        : thread_pool_(pool_size) {}

    void HandleRequest(Request req) {
        // Post work to thread pool
        boost::asio::post(thread_pool_, [this, req = std::move(req)]() {
            ProcessRequest(req);
        });
    }

    void ProcessRequest(const Request& req) {
        try {
            // Parse request
            auto params = ParseRequest(req);

            // Execute business logic
            auto result = ExecuteLogic(params);

            // Send response
            SendResponse(req.connection_id, result);

        } catch (const std::exception& e) {
            SendError(req.connection_id, e.what());
        }
    }
};
```

### 2. Async I/O Model (Coroutines)
```cpp
// Using C++20 coroutines for async I/O
class AsyncServer {
public:
    Task<void> HandleRequest(Request req) {
        try {
            // Async database query
            auto player_data = co_await database_.GetPlayerAsync(req.player_id);

            // Async cache update
            co_await cache_.SetAsync("player:" + req.player_id, player_data);

            // Async response
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

### 3. Actor Model (Message Passing)
```cpp
// Actor-based request handling
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

            // Process message
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
            // ... other message types
        }
    }
};
```

## Database Integration

### Connection Pooling
```cpp
class DatabaseConnectionPool {
    std::queue<std::unique_ptr<DatabaseConnection>> available_;
    std::mutex mutex_;
    std::condition_variable cv_;
    size_t pool_size_;

public:
    DatabaseConnectionPool(size_t pool_size, const std::string& connection_string)
        : pool_size_(pool_size) {
        // Create initial connections
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

// RAII wrapper for automatic connection release
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

// Usage
void HandleRequest(const Request& req) {
    ScopedConnection conn(db_pool_);
    auto result = conn->Query("SELECT * FROM players WHERE id = ?", req.player_id);
    // Connection automatically released when going out of scope
}
```

## Scaling Strategies

### Horizontal Scaling
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

### Stateless Services
```cpp
// Stateless service - can scale horizontally
class StatelessGameService {
public:
    Response HandleGameAction(const Request& req) {
        // 1. Load state from database/cache
        auto game_state = LoadGameState(req.game_id);

        // 2. Validate action
        if (!game_state.ValidateAction(req.action)) {
            return Response::Error("Invalid action");
        }

        // 3. Apply action
        game_state.ApplyAction(req.action);

        // 4. Save state
        SaveGameState(req.game_id, game_state);

        // 5. Return result
        return Response::Success(game_state);
    }

private:
    GameState LoadGameState(const std::string& game_id) {
        // Try cache first
        auto cached = cache_.Get("game:" + game_id);
        if (cached.has_value()) {
            return GameState::Deserialize(cached.value());
        }

        // Load from database
        auto state = database_.GetGameState(game_id);

        // Update cache
        cache_.Set("game:" + game_id, state.Serialize());

        return state;
    }
};
```

## Case Studies

### Hearthstone (Blizzard)

**Architecture:**
- REST API for game client
- Turn-based gameplay
- Client-server model (no peer-to-peer)
- Deterministic game logic
- Server-authoritative

**Key Technologies:**
- Stateless application servers
- MySQL for persistent storage
- Redis for caching and matchmaking
- RabbitMQ for async processing

**Scaling:**
- Horizontal scaling of API servers
- Database read replicas
- CDN for static assets
- Regional deployments

### Clash of Clans (Supercell)

**Architecture:**
- Asynchronous multiplayer
- Client-server model
- Attack replays stored and played back
- Server validates all actions

**Key Features:**
- Optimistic locking for resource updates
- Event sourcing for attack replays
- Queue-based async multiplayer
- Offline gameplay with server sync

### Words with Friends (Zynga)

**Architecture:**
- Turn-based word game
- Push notifications for turn alerts
- Social features (chat, friends)
- Cross-platform (mobile, web)

**Technology Stack:**
- Node.js for API servers
- MongoDB for game state
- Redis for sessions and caching
- AWS SNS for push notifications

**Scaling:**
- Microservices architecture
- Auto-scaling based on load
- Multi-region deployment
- Database sharding by user ID

## Performance Metrics

### Target SLAs

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

### Monitoring
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

## Conclusion

Non-real-time game servers prioritize scalability, reliability, and developer productivity over strict latency requirements. Modern architectures use stateless services, caching layers, message queues, and database optimization to handle millions of concurrent players. The choice between monolithic and microservices architecture depends on team size, complexity, and scalability requirements.

## Further Reading

- [AWS Game Tech Blog](https://aws.amazon.com/blogs/gametech/)
- [Scaling to Millions of Simultaneous Connections](https://www.slideshare.net/slideshow/scaling-to-millions-of-simultaneous-connections/17820191)
- [Microservices for Gaming](https://cloud.google.com/blog/products/gaming/building-a-multiplayer-game-with-microservices)
- [Database Sharding Strategies](https://www.digitalocean.com/community/tutorials/understanding-database-sharding)
