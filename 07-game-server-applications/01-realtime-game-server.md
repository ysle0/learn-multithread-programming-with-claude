# 실시간 게임 서버

## 개요

실시간 게임 서버는 FPS, MOBA, MMO와 같은 경쟁 멀티플레이어 게임의 핵심입니다. 공정하고 반응성 있는 게임플레이를 제공하기 위해 낮은 지연 시간, 높은 tick rate, 결정론적 시뮬레이션이 필요합니다. 이 문서에서는 고성능 실시간 게임 서버를 구축하기 위한 아키텍처, 스레딩 모델 및 구현 전략을 다룹니다.

## 목차

1. [서버 유형](#서버-유형)
2. [스레딩 아키텍처](#스레딩-아키텍처)
3. [Tick Rate와 시뮬레이션](#tick-rate와-시뮬레이션)
4. [네트워크 동기화](#네트워크-동기화)
5. [성능 최적화](#성능-최적화)
6. [사례 연구](#사례-연구)

## 서버 유형

### FPS (1인칭 슈팅)

**특성:**
- 높은 tick rate (60-128 Hz)
- 낮은 지연 시간 필수 (20ms 미만 권장)
- 빠른 액션에 정밀한 피격 판정 필요
- 클라이언트 측 예측과 서버 조정

**아키텍처:**
```
┌────────────────────────────────────────────────┐
│         FPS Server Architecture                │
├────────────────────────────────────────────────┤
│                                                 │
│  ┌───────────────┐        ┌──────────────────┐│
│  │  Network I/O  │◄──────►│  Input Buffer    ││
│  │   Thread      │        │  (Lock-free Q)   ││
│  │  (Recv/Send)  │        └──────────────────┘│
│  └───────────────┘                             │
│         │                                       │
│         ▼                                       │
│  ┌────────────────────────────────────┐        │
│  │       Game Simulation Thread       │        │
│  ├────────────────────────────────────┤        │
│  │ • Process Inputs (Player Movement) │        │
│  │ • Physics (Projectiles, Collision) │        │
│  │ • Game Logic (Health, Weapons)     │        │
│  │ • AI (Bots)                        │        │
│  │ • Prepare Snapshots                │        │
│  └────────────────────────────────────┘        │
│         │                                       │
│         ▼                                       │
│  ┌──────────────────┐                          │
│  │  Output Buffer   │                          │
│  │  (Lock-free Q)   │                          │
│  └──────────────────┘                          │
│         │                                       │
│         ▼                                       │
│  ┌───────────────┐                             │
│  │  Replication  │                             │
│  │    Thread     │                             │
│  │  (Delta Comp) │                             │
│  └───────────────┘                             │
│                                                 │
└────────────────────────────────────────────────┘

Tick Rate: 60-128 Hz
Max Players: 32-64
Latency Target: < 20ms
```

**구현 예시:**

```cpp
// FPS 서버 - 높은 tick rate 시뮬레이션
class FPSGameServer {
public:
    FPSGameServer(int tick_rate = 128)
        : tick_rate_(tick_rate),
          tick_duration_(1000000 / tick_rate),  // 마이크로초
          running_(false) {}

    void Start() {
        running_ = true;

        // 네트워크 I/O 스레드 시작
        network_thread_ = std::thread(&FPSGameServer::NetworkThread, this);

        // 메인 시뮬레이션 스레드 시작
        simulation_thread_ = std::thread(&FPSGameServer::SimulationThread, this);

        // 복제 스레드 시작
        replication_thread_ = std::thread(&FPSGameServer::ReplicationThread, this);
    }

    void Stop() {
        running_ = false;
        if (network_thread_.joinable()) network_thread_.join();
        if (simulation_thread_.joinable()) simulation_thread_.join();
        if (replication_thread_.joinable()) replication_thread_.join();
    }

private:
    void NetworkThread() {
        while (running_) {
            // 클라이언트로부터 패킷 수신
            ReceivePackets();

            // 송신 패킷 전송
            SendPackets();

            // 바쁜 대기를 방지하기 위한 짧은 sleep
            std::this_thread::sleep_for(std::chrono::microseconds(100));
        }
    }

    void SimulationThread() {
        using clock = std::chrono::high_resolution_clock;
        auto next_tick = clock::now();

        uint32_t tick_number = 0;

        while (running_) {
            auto tick_start = clock::now();

            // 대기 중인 모든 입력 처리
            ProcessInputs(tick_number);

            // 물리 업데이트
            UpdatePhysics(tick_duration_.count() / 1000000.0f);

            // 게임 로직 업데이트
            UpdateGameLogic();

            // AI 업데이트
            UpdateAI();

            // 이 tick의 스냅샷 생성
            CreateSnapshot(tick_number);

            tick_number++;

            // 다음 tick까지 sleep
            next_tick += tick_duration_;
            std::this_thread::sleep_until(next_tick);

            // 실제 tick 시간 측정
            auto tick_end = clock::now();
            auto tick_time = std::chrono::duration_cast<std::chrono::microseconds>(
                tick_end - tick_start);

            if (tick_time > tick_duration_) {
                // 서버가 지연되고 있음!
                LogWarning("Tick overrun: " + std::to_string(tick_time.count()) + "us");
            }
        }
    }

    void ReplicationThread() {
        while (running_) {
            // 최신 스냅샷 가져오기
            auto snapshot = snapshot_queue_.pop();

            // 연결된 각 플레이어에 대해
            for (auto& [player_id, connection] : connections_) {
                // 마지막으로 확인된 스냅샷과의 차이 계산
                auto delta = CalculateDelta(player_id, snapshot);

                // 압축 및 전송
                auto packet = CompressSnapshot(delta);
                outgoing_queue_.push({connection, packet});
            }

            // 다음 스냅샷 대기
            std::this_thread::sleep_for(std::chrono::milliseconds(15)); // ~60 Hz 업데이트
        }
    }

    void ProcessInputs(uint32_t tick_number) {
        // 큐의 모든 입력 처리
        PlayerInput input;
        while (input_queue_.try_pop(input)) {
            // 입력 유효성 검사
            if (!ValidateInput(input)) continue;

            // 플레이어에 입력 적용
            auto& player = players_[input.player_id];
            player.ApplyInput(input, tick_number);
        }
    }

    void UpdatePhysics(float delta_time) {
        // 발사체 업데이트
        for (auto& projectile : projectiles_) {
            projectile.position += projectile.velocity * delta_time;

            // 충돌 검사
            if (CheckCollision(projectile)) {
                HandleHit(projectile);
            }
        }

        // 플레이어 위치 업데이트
        for (auto& [id, player] : players_) {
            player.UpdateMovement(delta_time);

            // 서버 측 피격 유효성 검사
            if (player.IsShooting()) {
                ValidateShot(player);
            }
        }
    }

    void UpdateGameLogic() {
        // 체력, 탄약, 무기 등 업데이트
        for (auto& [id, player] : players_) {
            player.UpdateLogic();

            // 플레이어 사망 확인
            if (player.health <= 0 && player.alive) {
                HandlePlayerDeath(id);
            }
        }

        // 게임 모드별 로직 업데이트
        UpdateGameMode();
    }

    void CreateSnapshot(uint32_t tick_number) {
        Snapshot snapshot;
        snapshot.tick_number = tick_number;
        snapshot.timestamp = GetCurrentTime();

        // 모든 엔티티 상태 캡처
        for (const auto& [id, player] : players_) {
            snapshot.player_states.push_back(player.GetState());
        }

        for (const auto& projectile : projectiles_) {
            snapshot.projectile_states.push_back(projectile.GetState());
        }

        // 복제 스레드로 전달
        snapshot_queue_.push(std::move(snapshot));
    }

    // 피격 판정을 위한 지연 보상
    bool ValidateShot(const Player& shooter) {
        // 사격자의 시점 시간으로 월드 상태 되감기
        uint32_t rewind_tick = shooter.last_acknowledged_tick;

        // 과거 월드 상태 가져오기
        auto past_state = GetHistoricalState(rewind_tick);

        // 되감은 상태에서 레이캐스트 수행
        RaycastResult hit = past_state.Raycast(
            shooter.position,
            shooter.aim_direction,
            shooter.weapon_range
        );

        if (hit.entity_type == EntityType::Player) {
            // 피격이 합리적인지 검증 (안티 치트)
            if (ValidateHitReason(shooter, hit)) {
                // 현재 상태에서 데미지 적용
                ApplyDamage(hit.entity_id, shooter.weapon_damage);
                return true;
            }
        }

        return false;
    }

private:
    int tick_rate_;
    std::chrono::microseconds tick_duration_;
    std::atomic<bool> running_;

    // 스레딩
    std::thread network_thread_;
    std::thread simulation_thread_;
    std::thread replication_thread_;

    // Lock-free 큐
    LockFreeQueue<PlayerInput> input_queue_;
    LockFreeQueue<Snapshot> snapshot_queue_;
    LockFreeQueue<OutgoingPacket> outgoing_queue_;

    // 게임 상태
    std::unordered_map<uint32_t, Player> players_;
    std::vector<Projectile> projectiles_;
    std::unordered_map<uint32_t, Connection> connections_;

    // 지연 보상을 위한 과거 상태 (순환 버퍼)
    std::array<Snapshot, 256> historical_states_;
    uint8_t history_index_ = 0;
};
```

### MOBA (멀티플레이어 온라인 배틀 아레나)

**특성:**
- 중간 tick rate (20-30 Hz)
- 결정론적 시뮬레이션 필수
- 일반적으로 10명의 플레이어 (5v5)
- 복잡한 게임 로직 (스킬, 아이템, AI)

**아키텍처:**
```
┌────────────────────────────────────────────────┐
│           MOBA Server Architecture              │
├────────────────────────────────────────────────┤
│                                                 │
│  ┌─────────────────────────────────────────┐   │
│  │      Deterministic Game Loop             │   │
│  │         (30 Hz Fixed Step)               │   │
│  ├─────────────────────────────────────────┤   │
│  │                                           │   │
│  │  1. Collect Inputs (Command Pattern)    │   │
│  │  2. Validate Commands                   │   │
│  │  3. Execute Commands (Deterministic)    │   │
│  │  4. Update Abilities/Cooldowns          │   │
│  │  5. Update AI (Minions, Jungle)         │   │
│  │  6. Update Physics/Movement             │   │
│  │  7. Resolve Collisions                  │   │
│  │  8. Calculate Damage/Healing            │   │
│  │  9. Update Game State                   │   │
│  │ 10. Broadcast State Changes             │   │
│  │                                           │   │
│  └─────────────────────────────────────────┘   │
│                                                 │
│  ┌─────────────┐      ┌─────────────────────┐  │
│  │  Worker     │      │  Ability System     │  │
│  │  Thread     │──────│  (Spell Queue)      │  │
│  │  Pool       │      └─────────────────────┘  │
│  │             │                                │
│  │ • Events    │      ┌─────────────────────┐  │
│  │ • Effects   │──────│  AI Processing      │  │
│  │ • Pathfind  │      │  (Minions/Jungle)   │  │
│  └─────────────┘      └─────────────────────┘  │
│                                                 │
└────────────────────────────────────────────────┘

Tick Rate: 20-30 Hz
Max Players: 10
Latency Target: < 50ms
```

**구현 예시:**

```cpp
// MOBA 서버 - 결정론적 lockstep 시뮬레이션
class MOBAGameServer {
public:
    MOBAGameServer() : tick_rate_(30), running_(false) {}

    void Start() {
        running_ = true;

        // 메인 시뮬레이션 스레드 (결정론적)
        simulation_thread_ = std::thread(&MOBAGameServer::SimulationThread, this);

        // 병렬 처리를 위한 워커 스레드
        for (int i = 0; i < 4; ++i) {
            worker_threads_.emplace_back(&MOBAGameServer::WorkerThread, this);
        }

        // 네트워크 I/O 스레드
        network_thread_ = std::thread(&MOBAGameServer::NetworkThread, this);
    }

    void Stop() {
        running_ = false;
        work_available_.notify_all();

        if (simulation_thread_.joinable()) simulation_thread_.join();
        for (auto& thread : worker_threads_) {
            if (thread.joinable()) thread.join();
        }
        if (network_thread_.joinable()) network_thread_.join();
    }

private:
    void SimulationThread() {
        constexpr auto tick_duration = std::chrono::milliseconds(33); // ~30 Hz
        auto next_tick = std::chrono::steady_clock::now();

        uint32_t tick_number = 0;

        while (running_) {
            auto tick_start = std::chrono::steady_clock::now();

            // 1. 이 tick의 모든 명령 수집
            std::vector<Command> commands = CollectCommands(tick_number);

            // 2. 결정론적으로 명령 정렬 (플레이어 ID, 타임스탬프 순)
            std::sort(commands.begin(), commands.end(),
                [](const Command& a, const Command& b) {
                    if (a.player_id != b.player_id)
                        return a.player_id < b.player_id;
                    return a.timestamp < b.timestamp;
                });

            // 3. 명령 검증 및 실행 (결정론적 순서)
            for (const auto& cmd : commands) {
                if (ValidateCommand(cmd)) {
                    ExecuteCommand(cmd);
                }
            }

            // 4. 스킬과 쿨다운 업데이트
            UpdateAbilities(tick_duration.count() / 1000.0f);

            // 5. AI 업데이트 (미니언, 정글, 타워)
            UpdateAI(tick_duration.count() / 1000.0f);

            // 6. 물리 및 이동 업데이트 (결정론적)
            UpdatePhysics(tick_duration.count() / 1000.0f);

            // 7. 데미지 계산 및 효과 적용
            ProcessCombat();

            // 8. 게임 상태 업데이트 (골드, 경험치, 아이템)
            UpdateGameState();

            // 9. 클라이언트에 상태 변경 방송
            BroadcastStateChanges(tick_number);

            tick_number++;

            // 다음 tick까지 sleep
            next_tick += tick_duration;
            std::this_thread::sleep_until(next_tick);

            // tick 성능 모니터링
            auto tick_time = std::chrono::steady_clock::now() - tick_start;
            if (tick_time > tick_duration) {
                LogWarning("Tick overrun: " +
                    std::to_string(
                        std::chrono::duration_cast<std::chrono::milliseconds>(tick_time).count()
                    ) + "ms");
            }
        }
    }

    void WorkerThread() {
        while (running_) {
            std::unique_lock<std::mutex> lock(work_mutex_);
            work_available_.wait(lock, [this] {
                return !work_queue_.empty() || !running_;
            });

            if (!running_) break;

            if (!work_queue_.empty()) {
                auto work = work_queue_.front();
                work_queue_.pop();
                lock.unlock();

                // 작업 처리
                work();
            }
        }
    }

    void UpdateAbilities(float delta_time) {
        // 워커 스레드에 스킬 업데이트 분배
        for (auto& [id, hero] : heroes_) {
            SubmitWork([&hero, delta_time]() {
                hero.UpdateAbilities(delta_time);
            });
        }

        // 모든 스킬 업데이트 완료 대기
        WaitForWorkCompletion();
    }

    void UpdateAI(float delta_time) {
        // 미니언 AI를 병렬로 업데이트
        SubmitWork([this, delta_time]() {
            UpdateMinionWave(LaneType::Top, delta_time);
        });

        SubmitWork([this, delta_time]() {
            UpdateMinionWave(LaneType::Mid, delta_time);
        });

        SubmitWork([this, delta_time]() {
            UpdateMinionWave(LaneType::Bot, delta_time);
        });

        SubmitWork([this, delta_time]() {
            UpdateJungleMonsters(delta_time);
        });

        WaitForWorkCompletion();
    }

    void ProcessCombat() {
        // 스킬, 공격 등에서 모든 데미지 이벤트 수집
        std::vector<DamageEvent> damage_events;

        for (auto& [id, hero] : heroes_) {
            auto events = hero.GetPendingDamageEvents();
            damage_events.insert(damage_events.end(), events.begin(), events.end());
        }

        // 결정론적으로 데미지 이벤트 정렬
        std::sort(damage_events.begin(), damage_events.end(),
            [](const DamageEvent& a, const DamageEvent& b) {
                return a.timestamp < b.timestamp;
            });

        // 결정론적 순서로 데미지 적용
        for (const auto& event : damage_events) {
            ApplyDamage(event);
        }
    }

    void ExecuteCommand(const Command& cmd) {
        switch (cmd.type) {
            case CommandType::Move:
                ExecuteMoveCommand(cmd);
                break;
            case CommandType::Attack:
                ExecuteAttackCommand(cmd);
                break;
            case CommandType::CastAbility:
                ExecuteCastAbilityCommand(cmd);
                break;
            case CommandType::BuyItem:
                ExecuteBuyItemCommand(cmd);
                break;
            default:
                LogError("Unknown command type");
        }
    }

    void ExecuteCastAbilityCommand(const Command& cmd) {
        auto& hero = heroes_[cmd.player_id];

        // 스킬 사용 가능 여부 확인
        if (!hero.CanCastAbility(cmd.ability_index)) {
            return; // 쿨다운 또는 마나 부족
        }

        // 스킬 가져오기
        auto& ability = hero.GetAbility(cmd.ability_index);

        // 타겟팅 검증
        if (!ability.ValidateTarget(cmd.target_position, cmd.target_entity_id)) {
            return;
        }

        // 스킬 시전
        ability.Cast(cmd.target_position, cmd.target_entity_id);

        // 마나 소모 및 쿨다운 시작
        hero.ConsumeMana(ability.GetManaCost());
        ability.StartCooldown();

        // 스킬 효과 생성 (투사체, 범위 효과 등)
        CreateAbilityEffect(hero, ability, cmd);
    }

    // 결정론적 물리 업데이트
    void UpdatePhysics(float delta_time) {
        // 모든 엔티티 위치 업데이트
        for (auto& [id, hero] : heroes_) {
            UpdateEntityPhysics(hero, delta_time);
        }

        for (auto& minion : minions_) {
            UpdateEntityPhysics(minion, delta_time);
        }

        for (auto& projectile : projectiles_) {
            projectile.position += projectile.velocity * delta_time;

            // 충돌 검사
            if (CheckProjectileCollision(projectile)) {
                HandleProjectileHit(projectile);
            }
        }
    }

    void UpdateEntityPhysics(Entity& entity, float delta_time) {
        if (entity.has_movement_target) {
            // 목표까지의 방향 계산
            glm::vec3 direction = glm::normalize(
                entity.movement_target - entity.position
            );

            // 목표를 향해 이동
            glm::vec3 displacement = direction * entity.movement_speed * delta_time;

            // 목표에 도달했는지 확인
            if (glm::length(displacement) >= glm::length(entity.movement_target - entity.position)) {
                entity.position = entity.movement_target;
                entity.has_movement_target = false;
            } else {
                entity.position += displacement;
            }
        }
    }

    void BroadcastStateChanges(uint32_t tick_number) {
        // 상태 업데이트 패킷 생성
        StateUpdate update;
        update.tick_number = tick_number;

        // 영웅 상태 추가
        for (const auto& [id, hero] : heroes_) {
            update.hero_states.push_back(hero.GetState());
        }

        // 미니언 상태 추가 (변경된 미니언만)
        for (const auto& minion : minions_) {
            if (minion.state_changed) {
                update.minion_states.push_back(minion.GetState());
            }
        }

        // 이벤트 추가 (킬, 어시스트, 골드 등)
        update.events = std::move(pending_events_);
        pending_events_.clear();

        // 연결된 모든 클라이언트에 전송
        for (auto& [player_id, connection] : connections_) {
            SendStateUpdate(connection, update);
        }
    }

    void SubmitWork(std::function<void()> work) {
        {
            std::lock_guard<std::mutex> lock(work_mutex_);
            work_queue_.push(std::move(work));
            pending_work_++;
        }
        work_available_.notify_one();
    }

    void WaitForWorkCompletion() {
        std::unique_lock<std::mutex> lock(work_mutex_);
        work_completed_.wait(lock, [this] {
            return pending_work_ == 0;
        });
    }

private:
    int tick_rate_;
    std::atomic<bool> running_;

    // 스레딩
    std::thread simulation_thread_;
    std::vector<std::thread> worker_threads_;
    std::thread network_thread_;

    // 병렬 처리를 위한 작업 큐
    std::queue<std::function<void()>> work_queue_;
    std::mutex work_mutex_;
    std::condition_variable work_available_;
    std::condition_variable work_completed_;
    std::atomic<int> pending_work_{0};

    // 게임 상태
    std::unordered_map<uint32_t, Hero> heroes_;
    std::vector<Minion> minions_;
    std::vector<Projectile> projectiles_;
    std::vector<GameEvent> pending_events_;

    // 네트워크
    std::unordered_map<uint32_t, Connection> connections_;
};
```

### MMO (대규모 다중 접속 온라인)

**특성:**
- 가변 tick rate (지역에 따라 5-30 Hz)
- Zone 기반 아키텍처
- 서버당 수백에서 수천 명의 동시 접속 플레이어
- 관심 영역 관리 필수

**아키텍처:**
```
┌────────────────────────────────────────────────────────┐
│              MMO Server Architecture                    │
├────────────────────────────────────────────────────────┤
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │   Zone 1     │  │   Zone 2     │  │   Zone 3     │ │
│  │  (Combat)    │  │   (City)     │  │  (Dungeon)   │ │
│  │  Thread 1    │  │  Thread 2    │  │  Thread 3    │ │
│  │  30 Hz       │  │  10 Hz       │  │  20 Hz       │ │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘ │
│         │                  │                  │         │
│         └──────────────────┼──────────────────┘         │
│                            │                            │
│  ┌─────────────────────────┴──────────────────────┐    │
│  │         Interest Management System              │    │
│  ├────────────────────────────────────────────────┤    │
│  │ • Spatial hashing (Grid/Octree)                │    │
│  │ • Visibility culling                           │    │
│  │ • Priority-based updates                       │    │
│  │ • Distance-based update frequency              │    │
│  └────────────────────────────────────────────────┘    │
│                                                         │
│  ┌────────────────────────────────────────────────┐    │
│  │          Shared Services (Thread Pool)          │    │
│  ├────────────────────────────────────────────────┤    │
│  │ • Database I/O (Async)                         │    │
│  │ • Chat System                                  │    │
│  │ • Guild/Party Management                       │    │
│  │ • Auction House                                │    │
│  │ • Mail System                                  │    │
│  └────────────────────────────────────────────────┘    │
│                                                         │
└────────────────────────────────────────────────────────┘

Zones: Multiple per server
Tick Rate: 5-30 Hz (variable)
Max Players per Zone: 100-500
Latency Target: < 100ms
```

**구현 예시:**

```cpp
// MMO 서버 - 관심 영역 관리를 포함한 zone 기반 아키텍처
class MMOZoneServer {
public:
    MMOZoneServer(ZoneConfig config)
        : config_(config),
          tick_rate_(config.tick_rate),
          running_(false) {}

    void Start() {
        running_ = true;

        // 메인 zone 시뮬레이션 스레드
        simulation_thread_ = std::thread(&MMOZoneServer::SimulationThread, this);

        // 관심 영역 관리 스레드
        interest_thread_ = std::thread(&MMOZoneServer::InterestManagementThread, this);

        // 데이터베이스 I/O thread pool
        for (int i = 0; i < 2; ++i) {
            db_threads_.emplace_back(&MMOZoneServer::DatabaseThread, this);
        }

        // 네트워크 I/O 스레드
        network_thread_ = std::thread(&MMOZoneServer::NetworkThread, this);
    }

    void Stop() {
        running_ = false;
        // 모든 스레드 join...
    }

private:
    void SimulationThread() {
        auto tick_duration = std::chrono::milliseconds(1000 / tick_rate_);
        auto next_tick = std::chrono::steady_clock::now();

        while (running_) {
            // 플레이어 액션 처리
            ProcessPlayerActions();

            // NPC와 몬스터 업데이트
            UpdateNPCs();

            // 전투 업데이트
            UpdateCombat();

            // 월드 이벤트 업데이트 (스폰, 디스폰 등)
            UpdateWorldEvents();

            // 이동 처리
            UpdateMovement();

            // 공간 인덱스 업데이트
            UpdateSpatialIndex();

            next_tick += tick_duration;
            std::this_thread::sleep_until(next_tick);
        }
    }

    void InterestManagementThread() {
        // 시뮬레이션보다 낮은 빈도로 실행 (예: 10 Hz)
        auto update_interval = std::chrono::milliseconds(100);
        auto next_update = std::chrono::steady_clock::now();

        while (running_) {
            // 모든 플레이어의 가시성 세트 업데이트
            for (auto& [id, player] : players_) {
                UpdatePlayerVisibility(player);
            }

            // 우선순위 기반 상태 업데이트 전송
            SendPrioritizedUpdates();

            next_update += update_interval;
            std::this_thread::sleep_until(next_update);
        }
    }

    void UpdatePlayerVisibility(Player& player) {
        // 공간 인덱스에서 근처 엔티티 조회
        auto visible_entities = spatial_index_.QueryRadius(
            player.position,
            player.visibility_radius
        );

        // 각 엔티티에 대한 관심도 계산
        std::vector<InterestEntry> interests;
        for (auto entity_id : visible_entities) {
            float distance = glm::distance(
                player.position,
                GetEntityPosition(entity_id)
            );

            // 거리와 엔티티 유형에 따른 우선순위 계산
            float priority = CalculateInterestPriority(distance, entity_id);

            interests.push_back({entity_id, priority});
        }

        // 우선순위로 정렬
        std::sort(interests.begin(), interests.end(),
            [](const InterestEntry& a, const InterestEntry& b) {
                return a.priority > b.priority;
            });

        // 상위 N개 엔티티로 제한
        if (interests.size() > config_.max_visible_entities) {
            interests.resize(config_.max_visible_entities);
        }

        // 플레이어의 관심 세트 업데이트
        player.interest_set = std::move(interests);
    }

    float CalculateInterestPriority(float distance, EntityID entity_id) {
        // 거리 기반 기본 우선순위 (가까울수록 높은 우선순위)
        float priority = 1.0f / (1.0f + distance);

        // 특정 엔티티 유형에 대해 우선순위 가중치
        auto entity_type = GetEntityType(entity_id);
        switch (entity_type) {
            case EntityType::Player:
                priority *= 2.0f;  // 다른 플레이어는 중요함
                break;
            case EntityType::Boss:
                priority *= 3.0f;  // 보스는 매우 중요함
                break;
            case EntityType::QuestNPC:
                priority *= 1.5f;
                break;
            default:
                break;
        }

        return priority;
    }

    void SendPrioritizedUpdates() {
        for (auto& [player_id, player] : players_) {
            StateUpdate update;
            update.player_id = player_id;

            // 우선순위와 대역폭 예산에 따라 엔티티 추가
            size_t bandwidth_budget = config_.max_update_size_bytes;

            for (const auto& interest : player.interest_set) {
                auto entity_state = GetEntityState(interest.entity_id);
                size_t state_size = entity_state.GetSize();

                if (state_size > bandwidth_budget) {
                    break;  // 예산 소진
                }

                // 마지막 업데이트 이후 상태가 변경되었는지 확인
                if (HasStateChanged(player_id, interest.entity_id)) {
                    update.entity_states.push_back(entity_state);
                    bandwidth_budget -= state_size;
                }
            }

            // 플레이어에게 업데이트 전송
            if (!update.entity_states.empty()) {
                SendUpdate(player.connection, update);
            }
        }
    }

    void UpdateSpatialIndex() {
        // 공간 인덱스 재구축 (점진적 업데이트로 최적화 가능)
        spatial_index_.Clear();

        // 모든 플레이어 추가
        for (const auto& [id, player] : players_) {
            spatial_index_.Insert(id, player.position);
        }

        // 모든 NPC 추가
        for (const auto& npc : npcs_) {
            spatial_index_.Insert(npc.id, npc.position);
        }

        // 모든 몬스터 추가
        for (const auto& monster : monsters_) {
            spatial_index_.Insert(monster.id, monster.position);
        }
    }

private:
    ZoneConfig config_;
    int tick_rate_;
    std::atomic<bool> running_;

    // 스레딩
    std::thread simulation_thread_;
    std::thread interest_thread_;
    std::vector<std::thread> db_threads_;
    std::thread network_thread_;

    // 게임 상태
    std::unordered_map<uint32_t, Player> players_;
    std::vector<NPC> npcs_;
    std::vector<Monster> monsters_;

    // 공간 인덱싱
    SpatialHashGrid spatial_index_;
};
```

## 성능 최적화

### 1. Lock-free 통신
```cpp
// 스레드 간 통신을 위한 SPSC 큐 사용
template<typename T>
class SPSCQueue {
    std::vector<T> buffer_;
    std::atomic<size_t> write_pos_{0};
    std::atomic<size_t> read_pos_{0};
    size_t capacity_;

public:
    SPSCQueue(size_t capacity)
        : capacity_(capacity),
          buffer_(capacity) {}

    bool try_push(T value) {
        size_t write = write_pos_.load(std::memory_order_relaxed);
        size_t next_write = (write + 1) % capacity_;

        if (next_write == read_pos_.load(std::memory_order_acquire)) {
            return false;  // 큐 가득 참
        }

        buffer_[write] = std::move(value);
        write_pos_.store(next_write, std::memory_order_release);
        return true;
    }

    bool try_pop(T& value) {
        size_t read = read_pos_.load(std::memory_order_relaxed);

        if (read == write_pos_.load(std::memory_order_acquire)) {
            return false;  // 큐 비어 있음
        }

        value = std::move(buffer_[read]);
        read_pos_.store((read + 1) % capacity_, std::memory_order_release);
        return true;
    }
};
```

### 2. 오브젝트 풀링
```cpp
// 빈번하게 할당/해제되는 객체를 위한 오브젝트 풀
template<typename T>
class ObjectPool {
    std::vector<std::unique_ptr<T>> pool_;
    std::atomic<size_t> next_index_{0};
    std::mutex mutex_;

public:
    ObjectPool(size_t initial_size) {
        pool_.reserve(initial_size);
        for (size_t i = 0; i < initial_size; ++i) {
            pool_.push_back(std::make_unique<T>());
        }
    }

    T* Acquire() {
        std::lock_guard<std::mutex> lock(mutex_);
        if (pool_.empty()) {
            return new T();  // 힙 할당으로 폴백
        }
        auto obj = pool_.back().release();
        pool_.pop_back();
        return obj;
    }

    void Release(T* obj) {
        obj->Reset();  // 객체 상태 초기화
        std::lock_guard<std::mutex> lock(mutex_);
        pool_.push_back(std::unique_ptr<T>(obj));
    }
};

// 사용 예시
ObjectPool<Projectile> projectile_pool(1000);

void SpawnProjectile() {
    auto* proj = projectile_pool.Acquire();
    proj->Initialize(/* ... */);
    projectiles_.push_back(proj);
}

void DestroyProjectile(Projectile* proj) {
    projectile_pool.Release(proj);
}
```

### 3. 일괄 처리
```cpp
// 오버헤드를 줄이기 위해 이벤트를 일괄 처리
void ProcessNetworkEvents() {
    constexpr size_t BATCH_SIZE = 64;
    std::array<NetworkEvent, BATCH_SIZE> batch;

    while (true) {
        size_t count = 0;

        // 배치 수집
        while (count < BATCH_SIZE) {
            NetworkEvent event;
            if (!event_queue_.try_pop(event)) {
                break;
            }
            batch[count++] = event;
        }

        if (count == 0) break;

        // 배치 처리
        for (size_t i = 0; i < count; ++i) {
            ProcessEvent(batch[i]);
        }
    }
}
```

## 사례 연구

### Counter-Strike: Global Offensive (CS:GO)

**아키텍처:**
- 전용 서버 모델
- 64 Hz 또는 128 Hz tick rate
- Source 엔진
- 클라이언트 측 예측과 서버 조정
- 피격 판정을 위한 지연 보상

**핵심 기법:**
- 명령 버퍼링
- 델타 압축
- 우선순위 기반 엔티티 업데이트
- 지연 보상을 위한 과거 상태 스냅샷

### 리그 오브 레전드

**아키텍처:**
- 결정론적 lockstep 시뮬레이션
- 30 Hz tick rate
- 지역 데이터 센터
- 보조 기능을 위한 마이크로서비스

**핵심 기법:**
- 모든 플레이어 액션에 대한 Command 패턴
- 예측 가능한 시뮬레이션 순서
- 재접속을 위한 상태 롤백
- 부정 방지를 위한 관전자 딜레이 (3분)

### 월드 오브 워크래프트 (클래식)

**아키텍처:**
- Zone 기반 월드 서버
- ~50ms 스펠 일괄 처리 윈도우
- 약 10-20 Hz 유효 tick rate
- 인스턴싱을 위한 페이징 기술

**핵심 기법:**
- 관심 영역 관리 (가시성 버블)
- 데이터베이스 sharding
- Cross-realm 기술
- 레이어링을 통한 부하 분산

## 결론

실시간 게임 서버는 스레딩 모델, tick rate, 네트워크 동기화 전략에 대한 신중한 고려가 필요합니다. 아키텍처 선택은 게임 장르, 플레이어 수, 성능 요구사항에 따라 달라집니다. 현대 게임 서버는 최적의 성능과 확장성을 달성하기 위해 여러 스레딩 패턴을 결합한 하이브리드 접근 방식을 자주 사용합니다.

## 추가 참고 자료

- [Gaffer on Games - State Synchronization](https://gafferongames.com/post/state_synchronization/)
- [Valve Developer Wiki - Source Multiplayer Networking](https://developer.valvesoftware.com/wiki/Source_Multiplayer_Networking)
- [Riot Games Engineering - Determinism in League of Legends](https://technology.riotgames.com/)
- [Overwatch Gameplay Architecture](https://www.youtube.com/watch?v=W3aieHjyNvw)
