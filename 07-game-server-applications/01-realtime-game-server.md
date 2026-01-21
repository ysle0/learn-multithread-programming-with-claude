# Real-time Game Servers

## Overview

Real-time game servers are the backbone of competitive multiplayer games like FPS, MOBA, and MMO titles. They require low latency, high tick rates, and deterministic simulation to provide fair and responsive gameplay. This document covers the architecture, threading models, and implementation strategies for building high-performance real-time game servers.

## Table of Contents

1. [Server Types](#server-types)
2. [Threading Architecture](#threading-architecture)
3. [Tick Rate and Simulation](#tick-rate-and-simulation)
4. [Network Synchronization](#network-synchronization)
5. [Performance Optimization](#performance-optimization)
6. [Case Studies](#case-studies)

## Server Types

### FPS (First-Person Shooter)

**Characteristics:**
- High tick rate (60-128 Hz)
- Low latency critical (< 20ms desirable)
- Fast-paced action requires precise hit detection
- Client-side prediction with server reconciliation

**Architecture:**
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

**Implementation Example:**

```cpp
// FPS Server - High tick rate simulation
class FPSGameServer {
public:
    FPSGameServer(int tick_rate = 128)
        : tick_rate_(tick_rate),
          tick_duration_(1000000 / tick_rate),  // microseconds
          running_(false) {}

    void Start() {
        running_ = true;

        // Start network I/O thread
        network_thread_ = std::thread(&FPSGameServer::NetworkThread, this);

        // Start main simulation thread
        simulation_thread_ = std::thread(&FPSGameServer::SimulationThread, this);

        // Start replication thread
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
            // Receive packets from clients
            ReceivePackets();

            // Send outgoing packets
            SendPackets();

            // Small sleep to prevent busy waiting
            std::this_thread::sleep_for(std::chrono::microseconds(100));
        }
    }

    void SimulationThread() {
        using clock = std::chrono::high_resolution_clock;
        auto next_tick = clock::now();

        uint32_t tick_number = 0;

        while (running_) {
            auto tick_start = clock::now();

            // Process all pending inputs
            ProcessInputs(tick_number);

            // Update physics
            UpdatePhysics(tick_duration_.count() / 1000000.0f);

            // Update game logic
            UpdateGameLogic();

            // AI updates
            UpdateAI();

            // Create snapshot for this tick
            CreateSnapshot(tick_number);

            tick_number++;

            // Sleep until next tick
            next_tick += tick_duration_;
            std::this_thread::sleep_until(next_tick);

            // Measure actual tick time
            auto tick_end = clock::now();
            auto tick_time = std::chrono::duration_cast<std::chrono::microseconds>(
                tick_end - tick_start);

            if (tick_time > tick_duration_) {
                // Server is lagging!
                LogWarning("Tick overrun: " + std::to_string(tick_time.count()) + "us");
            }
        }
    }

    void ReplicationThread() {
        while (running_) {
            // Get latest snapshot
            auto snapshot = snapshot_queue_.pop();

            // For each connected player
            for (auto& [player_id, connection] : connections_) {
                // Calculate delta from last acknowledged snapshot
                auto delta = CalculateDelta(player_id, snapshot);

                // Compress and send
                auto packet = CompressSnapshot(delta);
                outgoing_queue_.push({connection, packet});
            }

            // Wait for next snapshot
            std::this_thread::sleep_for(std::chrono::milliseconds(15)); // ~60 Hz updates
        }
    }

    void ProcessInputs(uint32_t tick_number) {
        // Process all inputs in the queue
        PlayerInput input;
        while (input_queue_.try_pop(input)) {
            // Validate input
            if (!ValidateInput(input)) continue;

            // Apply input to player
            auto& player = players_[input.player_id];
            player.ApplyInput(input, tick_number);
        }
    }

    void UpdatePhysics(float delta_time) {
        // Update projectiles
        for (auto& projectile : projectiles_) {
            projectile.position += projectile.velocity * delta_time;

            // Check collisions
            if (CheckCollision(projectile)) {
                HandleHit(projectile);
            }
        }

        // Update player positions
        for (auto& [id, player] : players_) {
            player.UpdateMovement(delta_time);

            // Server-side hit validation
            if (player.IsShooting()) {
                ValidateShot(player);
            }
        }
    }

    void UpdateGameLogic() {
        // Update health, ammo, weapons, etc.
        for (auto& [id, player] : players_) {
            player.UpdateLogic();

            // Check if player died
            if (player.health <= 0 && player.alive) {
                HandlePlayerDeath(id);
            }
        }

        // Update game mode specific logic
        UpdateGameMode();
    }

    void CreateSnapshot(uint32_t tick_number) {
        Snapshot snapshot;
        snapshot.tick_number = tick_number;
        snapshot.timestamp = GetCurrentTime();

        // Capture all entity states
        for (const auto& [id, player] : players_) {
            snapshot.player_states.push_back(player.GetState());
        }

        for (const auto& projectile : projectiles_) {
            snapshot.projectile_states.push_back(projectile.GetState());
        }

        // Push to replication thread
        snapshot_queue_.push(std::move(snapshot));
    }

    // Lag compensation for hit detection
    bool ValidateShot(const Player& shooter) {
        // Rewind world state to shooter's view time
        uint32_t rewind_tick = shooter.last_acknowledged_tick;

        // Get historical world state
        auto past_state = GetHistoricalState(rewind_tick);

        // Perform ray cast in rewound state
        RaycastResult hit = past_state.Raycast(
            shooter.position,
            shooter.aim_direction,
            shooter.weapon_range
        );

        if (hit.entity_type == EntityType::Player) {
            // Validate hit is reasonable (anti-cheat)
            if (ValidateHitReason(shooter, hit)) {
                // Apply damage in current state
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

    // Threading
    std::thread network_thread_;
    std::thread simulation_thread_;
    std::thread replication_thread_;

    // Lock-free queues
    LockFreeQueue<PlayerInput> input_queue_;
    LockFreeQueue<Snapshot> snapshot_queue_;
    LockFreeQueue<OutgoingPacket> outgoing_queue_;

    // Game state
    std::unordered_map<uint32_t, Player> players_;
    std::vector<Projectile> projectiles_;
    std::unordered_map<uint32_t, Connection> connections_;

    // Historical states for lag compensation (circular buffer)
    std::array<Snapshot, 256> historical_states_;
    uint8_t history_index_ = 0;
};
```

### MOBA (Multiplayer Online Battle Arena)

**Characteristics:**
- Medium tick rate (20-30 Hz)
- Deterministic simulation crucial
- 10 players typical (5v5)
- Complex game logic (abilities, items, AI)

**Architecture:**
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

**Implementation Example:**

```cpp
// MOBA Server - Deterministic lockstep simulation
class MOBAGameServer {
public:
    MOBAGameServer() : tick_rate_(30), running_(false) {}

    void Start() {
        running_ = true;

        // Main simulation thread (deterministic)
        simulation_thread_ = std::thread(&MOBAGameServer::SimulationThread, this);

        // Worker threads for parallel processing
        for (int i = 0; i < 4; ++i) {
            worker_threads_.emplace_back(&MOBAGameServer::WorkerThread, this);
        }

        // Network I/O thread
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

            // 1. Collect all commands for this tick
            std::vector<Command> commands = CollectCommands(tick_number);

            // 2. Sort commands deterministically (by player ID, then timestamp)
            std::sort(commands.begin(), commands.end(),
                [](const Command& a, const Command& b) {
                    if (a.player_id != b.player_id)
                        return a.player_id < b.player_id;
                    return a.timestamp < b.timestamp;
                });

            // 3. Validate and execute commands (DETERMINISTIC ORDER)
            for (const auto& cmd : commands) {
                if (ValidateCommand(cmd)) {
                    ExecuteCommand(cmd);
                }
            }

            // 4. Update abilities and cooldowns
            UpdateAbilities(tick_duration.count() / 1000.0f);

            // 5. Update AI (minions, jungle, towers)
            UpdateAI(tick_duration.count() / 1000.0f);

            // 6. Update physics and movement (deterministic)
            UpdatePhysics(tick_duration.count() / 1000.0f);

            // 7. Calculate damage and apply effects
            ProcessCombat();

            // 8. Update game state (gold, XP, items)
            UpdateGameState();

            // 9. Broadcast state changes to clients
            BroadcastStateChanges(tick_number);

            tick_number++;

            // Sleep until next tick
            next_tick += tick_duration;
            std::this_thread::sleep_until(next_tick);

            // Monitor tick performance
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

                // Process work
                work();
            }
        }
    }

    void UpdateAbilities(float delta_time) {
        // Dispatch ability updates to worker threads
        for (auto& [id, hero] : heroes_) {
            SubmitWork([&hero, delta_time]() {
                hero.UpdateAbilities(delta_time);
            });
        }

        // Wait for all ability updates
        WaitForWorkCompletion();
    }

    void UpdateAI(float delta_time) {
        // Update minion AI in parallel
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
        // Collect all damage events from abilities, attacks, etc.
        std::vector<DamageEvent> damage_events;

        for (auto& [id, hero] : heroes_) {
            auto events = hero.GetPendingDamageEvents();
            damage_events.insert(damage_events.end(), events.begin(), events.end());
        }

        // Sort damage events deterministically
        std::sort(damage_events.begin(), damage_events.end(),
            [](const DamageEvent& a, const DamageEvent& b) {
                return a.timestamp < b.timestamp;
            });

        // Apply damage in deterministic order
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

        // Check if ability is available
        if (!hero.CanCastAbility(cmd.ability_index)) {
            return; // Cooldown or insufficient mana
        }

        // Get ability
        auto& ability = hero.GetAbility(cmd.ability_index);

        // Validate targeting
        if (!ability.ValidateTarget(cmd.target_position, cmd.target_entity_id)) {
            return;
        }

        // Cast ability
        ability.Cast(cmd.target_position, cmd.target_entity_id);

        // Deduct mana and start cooldown
        hero.ConsumeMana(ability.GetManaCost());
        ability.StartCooldown();

        // Create ability effect (projectile, area of effect, etc.)
        CreateAbilityEffect(hero, ability, cmd);
    }

    // Deterministic physics update
    void UpdatePhysics(float delta_time) {
        // Update all entity positions
        for (auto& [id, hero] : heroes_) {
            UpdateEntityPhysics(hero, delta_time);
        }

        for (auto& minion : minions_) {
            UpdateEntityPhysics(minion, delta_time);
        }

        for (auto& projectile : projectiles_) {
            projectile.position += projectile.velocity * delta_time;

            // Check collision
            if (CheckProjectileCollision(projectile)) {
                HandleProjectileHit(projectile);
            }
        }
    }

    void UpdateEntityPhysics(Entity& entity, float delta_time) {
        if (entity.has_movement_target) {
            // Calculate direction to target
            glm::vec3 direction = glm::normalize(
                entity.movement_target - entity.position
            );

            // Move towards target
            glm::vec3 displacement = direction * entity.movement_speed * delta_time;

            // Check if we reached target
            if (glm::length(displacement) >= glm::length(entity.movement_target - entity.position)) {
                entity.position = entity.movement_target;
                entity.has_movement_target = false;
            } else {
                entity.position += displacement;
            }
        }
    }

    void BroadcastStateChanges(uint32_t tick_number) {
        // Create state update packet
        StateUpdate update;
        update.tick_number = tick_number;

        // Add hero states
        for (const auto& [id, hero] : heroes_) {
            update.hero_states.push_back(hero.GetState());
        }

        // Add minion states (only changed minions)
        for (const auto& minion : minions_) {
            if (minion.state_changed) {
                update.minion_states.push_back(minion.GetState());
            }
        }

        // Add events (kills, assists, gold, etc.)
        update.events = std::move(pending_events_);
        pending_events_.clear();

        // Send to all connected clients
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

    // Threading
    std::thread simulation_thread_;
    std::vector<std::thread> worker_threads_;
    std::thread network_thread_;

    // Work queue for parallel processing
    std::queue<std::function<void()>> work_queue_;
    std::mutex work_mutex_;
    std::condition_variable work_available_;
    std::condition_variable work_completed_;
    std::atomic<int> pending_work_{0};

    // Game state
    std::unordered_map<uint32_t, Hero> heroes_;
    std::vector<Minion> minions_;
    std::vector<Projectile> projectiles_;
    std::vector<GameEvent> pending_events_;

    // Network
    std::unordered_map<uint32_t, Connection> connections_;
};
```

### MMO (Massively Multiplayer Online)

**Characteristics:**
- Variable tick rate (5-30 Hz depending on area)
- Zone-based architecture
- Hundreds to thousands of concurrent players per server
- Interest management crucial

**Architecture:**
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

**Implementation Example:**

```cpp
// MMO Server - Zone-based architecture with interest management
class MMOZoneServer {
public:
    MMOZoneServer(ZoneConfig config)
        : config_(config),
          tick_rate_(config.tick_rate),
          running_(false) {}

    void Start() {
        running_ = true;

        // Main zone simulation thread
        simulation_thread_ = std::thread(&MMOZoneServer::SimulationThread, this);

        // Interest management thread
        interest_thread_ = std::thread(&MMOZoneServer::InterestManagementThread, this);

        // Database I/O thread pool
        for (int i = 0; i < 2; ++i) {
            db_threads_.emplace_back(&MMOZoneServer::DatabaseThread, this);
        }

        // Network I/O thread
        network_thread_ = std::thread(&MMOZoneServer::NetworkThread, this);
    }

    void Stop() {
        running_ = false;
        // Join all threads...
    }

private:
    void SimulationThread() {
        auto tick_duration = std::chrono::milliseconds(1000 / tick_rate_);
        auto next_tick = std::chrono::steady_clock::now();

        while (running_) {
            // Process player actions
            ProcessPlayerActions();

            // Update NPCs and monsters
            UpdateNPCs();

            // Update combat
            UpdateCombat();

            // Update world events (spawns, despawns, etc.)
            UpdateWorldEvents();

            // Process movement
            UpdateMovement();

            // Update spatial index
            UpdateSpatialIndex();

            next_tick += tick_duration;
            std::this_thread::sleep_until(next_tick);
        }
    }

    void InterestManagementThread() {
        // Runs at lower frequency than simulation (e.g., 10 Hz)
        auto update_interval = std::chrono::milliseconds(100);
        auto next_update = std::chrono::steady_clock::now();

        while (running_) {
            // Update visibility sets for all players
            for (auto& [id, player] : players_) {
                UpdatePlayerVisibility(player);
            }

            // Send state updates based on priority
            SendPrioritizedUpdates();

            next_update += update_interval;
            std::this_thread::sleep_until(next_update);
        }
    }

    void UpdatePlayerVisibility(Player& player) {
        // Query spatial index for nearby entities
        auto visible_entities = spatial_index_.QueryRadius(
            player.position,
            player.visibility_radius
        );

        // Calculate interest level for each entity
        std::vector<InterestEntry> interests;
        for (auto entity_id : visible_entities) {
            float distance = glm::distance(
                player.position,
                GetEntityPosition(entity_id)
            );

            // Calculate priority based on distance and entity type
            float priority = CalculateInterestPriority(distance, entity_id);

            interests.push_back({entity_id, priority});
        }

        // Sort by priority
        std::sort(interests.begin(), interests.end(),
            [](const InterestEntry& a, const InterestEntry& b) {
                return a.priority > b.priority;
            });

        // Limit to top N entities
        if (interests.size() > config_.max_visible_entities) {
            interests.resize(config_.max_visible_entities);
        }

        // Update player's interest set
        player.interest_set = std::move(interests);
    }

    float CalculateInterestPriority(float distance, EntityID entity_id) {
        // Base priority on distance (closer = higher priority)
        float priority = 1.0f / (1.0f + distance);

        // Boost priority for certain entity types
        auto entity_type = GetEntityType(entity_id);
        switch (entity_type) {
            case EntityType::Player:
                priority *= 2.0f;  // Other players are important
                break;
            case EntityType::Boss:
                priority *= 3.0f;  // Bosses are very important
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

            // Add entities based on priority and bandwidth budget
            size_t bandwidth_budget = config_.max_update_size_bytes;

            for (const auto& interest : player.interest_set) {
                auto entity_state = GetEntityState(interest.entity_id);
                size_t state_size = entity_state.GetSize();

                if (state_size > bandwidth_budget) {
                    break;  // Budget exhausted
                }

                // Check if state changed since last update
                if (HasStateChanged(player_id, interest.entity_id)) {
                    update.entity_states.push_back(entity_state);
                    bandwidth_budget -= state_size;
                }
            }

            // Send update to player
            if (!update.entity_states.empty()) {
                SendUpdate(player.connection, update);
            }
        }
    }

    void UpdateSpatialIndex() {
        // Rebuild spatial index (could be optimized with incremental updates)
        spatial_index_.Clear();

        // Add all players
        for (const auto& [id, player] : players_) {
            spatial_index_.Insert(id, player.position);
        }

        // Add all NPCs
        for (const auto& npc : npcs_) {
            spatial_index_.Insert(npc.id, npc.position);
        }

        // Add all monsters
        for (const auto& monster : monsters_) {
            spatial_index_.Insert(monster.id, monster.position);
        }
    }

private:
    ZoneConfig config_;
    int tick_rate_;
    std::atomic<bool> running_;

    // Threading
    std::thread simulation_thread_;
    std::thread interest_thread_;
    std::vector<std::thread> db_threads_;
    std::thread network_thread_;

    // Game state
    std::unordered_map<uint32_t, Player> players_;
    std::vector<NPC> npcs_;
    std::vector<Monster> monsters_;

    // Spatial indexing
    SpatialHashGrid spatial_index_;
};
```

## Performance Optimization

### 1. Lock-free Communication
```cpp
// Use SPSC queue for inter-thread communication
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
            return false;  // Queue full
        }

        buffer_[write] = std::move(value);
        write_pos_.store(next_write, std::memory_order_release);
        return true;
    }

    bool try_pop(T& value) {
        size_t read = read_pos_.load(std::memory_order_relaxed);

        if (read == write_pos_.load(std::memory_order_acquire)) {
            return false;  // Queue empty
        }

        value = std::move(buffer_[read]);
        read_pos_.store((read + 1) % capacity_, std::memory_order_release);
        return true;
    }
};
```

### 2. Object Pooling
```cpp
// Object pool for frequently allocated/deallocated objects
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
            return new T();  // Fallback to heap allocation
        }
        auto obj = pool_.back().release();
        pool_.pop_back();
        return obj;
    }

    void Release(T* obj) {
        obj->Reset();  // Reset object state
        std::lock_guard<std::mutex> lock(mutex_);
        pool_.push_back(std::unique_ptr<T>(obj));
    }
};

// Usage
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

### 3. Batch Processing
```cpp
// Process events in batches to reduce overhead
void ProcessNetworkEvents() {
    constexpr size_t BATCH_SIZE = 64;
    std::array<NetworkEvent, BATCH_SIZE> batch;

    while (true) {
        size_t count = 0;

        // Collect batch
        while (count < BATCH_SIZE) {
            NetworkEvent event;
            if (!event_queue_.try_pop(event)) {
                break;
            }
            batch[count++] = event;
        }

        if (count == 0) break;

        // Process batch
        for (size_t i = 0; i < count; ++i) {
            ProcessEvent(batch[i]);
        }
    }
}
```

## Case Studies

### Counter-Strike: Global Offensive (CS:GO)

**Architecture:**
- Dedicated server model
- 64 Hz or 128 Hz tick rate
- Source Engine
- Client-side prediction with server reconciliation
- Lag compensation for hit detection

**Key Techniques:**
- Command buffering
- Delta compression
- Priority-based entity updates
- Historical state snapshots for lag compensation

### League of Legends

**Architecture:**
- Deterministic lockstep simulation
- 30 Hz tick rate
- Regional data centers
- Microservices for auxiliary features

**Key Techniques:**
- Command pattern for all player actions
- Predictable simulation order
- State rollback for reconnection
- Spectator delay (3 minutes) for cheating prevention

### World of Warcraft (Classic)

**Architecture:**
- Zone-based world servers
- ~50ms spell batching window
- Approximately 10-20 Hz effective tick rate
- Phasing technology for instancing

**Key Techniques:**
- Interest management (visibility bubbles)
- Database sharding
- Cross-realm technology
- Load balancing via layering

## Conclusion

Real-time game servers require careful consideration of threading models, tick rates, and network synchronization strategies. The choice of architecture depends on the game genre, player count, and performance requirements. Modern game servers often use hybrid approaches, combining multiple threading patterns to achieve optimal performance and scalability.

## Further Reading

- [Gaffer on Games - State Synchronization](https://gafferongames.com/post/state_synchronization/)
- [Valve Developer Wiki - Source Multiplayer Networking](https://developer.valvesoftware.com/wiki/Source_Multiplayer_Networking)
- [Riot Games Engineering - Determinism in League of Legends](https://technology.riotgames.com/)
- [Overwatch Gameplay Architecture](https://www.youtube.com/watch?v=W3aieHjyNvw)
