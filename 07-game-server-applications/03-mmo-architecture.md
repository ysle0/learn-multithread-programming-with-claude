# MMO Architecture: Sharding and World Partitioning

## Overview

Massively Multiplayer Online (MMO) games present unique challenges in server architecture due to the need to support thousands of concurrent players in a persistent, shared world. This document covers advanced architectural patterns including sharding, world partitioning, zone management, and cross-server communication strategies.

## Table of Contents

1. [Core Concepts](#core-concepts)
2. [World Partitioning](#world-partitioning)
3. [Sharding Strategies](#sharding-strategies)
4. [Zone Management](#zone-management)
5. [Cross-Zone Communication](#cross-zone-communication)
6. [Load Balancing](#load-balancing)
7. [Case Studies](#case-studies)

## Core Concepts

### MMO Server Challenges

```
Challenge Areas:
┌────────────────────────────────────────────────┐
│ 1. Scale: 10,000+ concurrent players           │
│ 2. Persistence: 24/7 uptime, shared world      │
│ 3. Consistency: Synchronized game state        │
│ 4. Performance: Low latency despite scale      │
│ 5. Fairness: No advantages from server issues  │
└────────────────────────────────────────────────┘
```

### Architecture Layers

```
┌─────────────────────────────────────────────────────────┐
│                  MMO Server Stack                        │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  Layer 1: Gateway / Connection Layer                    │
│  ┌────────────────────────────────────────────────┐     │
│  │  • Client connections (WebSocket/TCP)          │     │
│  │  • Authentication                               │     │
│  │  • Protocol handling                            │     │
│  │  • Rate limiting                                │     │
│  └────────────────────────────────────────────────┘     │
│                         │                                │
│                         ▼                                │
│  Layer 2: World Servers (Game Logic)                    │
│  ┌────────────────────────────────────────────────┐     │
│  │  • Zone simulation                              │     │
│  │  • Entity management                            │     │
│  │  • Combat/skill processing                      │     │
│  │  • AI (NPCs, monsters)                          │     │
│  └────────────────────────────────────────────────┘     │
│                         │                                │
│                         ▼                                │
│  Layer 3: Shared Services                               │
│  ┌────────────────────────────────────────────────┐     │
│  │  • Database (player data, inventory)           │     │
│  │  • Cache (Redis - hot data)                    │     │
│  │  • Chat/social systems                          │     │
│  │  • Guild/party management                       │     │
│  │  • Auction house                                │     │
│  │  • Leaderboards                                 │     │
│  └────────────────────────────────────────────────┘     │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

## World Partitioning

### Grid-Based Partitioning

**Concept:** Divide the world into a uniform grid, each cell managed by a zone server.

```
World Map (Grid Partitioning)
┌─────────┬─────────┬─────────┬─────────┐
│  Zone   │  Zone   │  Zone   │  Zone   │
│  (0,0)  │  (1,0)  │  (2,0)  │  (3,0)  │
│ Server1 │ Server2 │ Server3 │ Server4 │
├─────────┼─────────┼─────────┼─────────┤
│  Zone   │  Zone   │  Zone   │  Zone   │
│  (0,1)  │  (1,1)  │  (2,1)  │  (3,1)  │
│ Server5 │ Server6 │ Server1 │ Server2 │
├─────────┼─────────┼─────────┼─────────┤
│  Zone   │  Zone   │  Zone   │  Zone   │
│  (0,2)  │  (1,2)  │  (2,2)  │  (3,2)  │
│ Server3 │ Server4 │ Server5 │ Server6 │
├─────────┼─────────┼─────────┼─────────┤
│  Zone   │  Zone   │  Zone   │  Zone   │
│  (0,3)  │  (1,3)  │  (2,3)  │  (3,3)  │
│ Server1 │ Server2 │ Server3 │ Server4 │
└─────────┴─────────┴─────────┴─────────┘

Grid Size: 1000m x 1000m per zone
Servers: 6 (zones distributed across servers)
Players per Zone: 100-500
```

**Implementation:**

```cpp
// Grid-based world partitioning
class GridWorldPartitioner {
public:
    struct GridCell {
        int x;
        int y;
        std::string server_id;
        std::unordered_set<EntityID> entities;
    };

    GridWorldPartitioner(float cell_size, int grid_width, int grid_height)
        : cell_size_(cell_size),
          grid_width_(grid_width),
          grid_height_(grid_height) {

        // Initialize grid
        grid_.resize(grid_width * grid_height);

        for (int y = 0; y < grid_height; ++y) {
            for (int x = 0; x < grid_width; ++x) {
                int index = y * grid_width + x;
                grid_[index].x = x;
                grid_[index].y = y;

                // Assign server (round-robin distribution)
                grid_[index].server_id = "server_" +
                    std::to_string(index % num_servers_);
            }
        }
    }

    // Get grid cell from world position
    GridCell* GetCell(float world_x, float world_y) {
        int grid_x = static_cast<int>(world_x / cell_size_);
        int grid_y = static_cast<int>(world_y / cell_size_);

        if (grid_x < 0 || grid_x >= grid_width_ ||
            grid_y < 0 || grid_y >= grid_height_) {
            return nullptr; // Out of bounds
        }

        return &grid_[grid_y * grid_width_ + grid_x];
    }

    // Get neighboring cells (for cross-zone visibility)
    std::vector<GridCell*> GetNeighborCells(int grid_x, int grid_y) {
        std::vector<GridCell*> neighbors;

        for (int dy = -1; dy <= 1; ++dy) {
            for (int dx = -1; dx <= 1; ++dx) {
                int nx = grid_x + dx;
                int ny = grid_y + dy;

                if (nx >= 0 && nx < grid_width_ &&
                    ny >= 0 && ny < grid_height_) {
                    neighbors.push_back(&grid_[ny * grid_width_ + nx]);
                }
            }
        }

        return neighbors;
    }

    // Handle entity movement between cells
    void MoveEntity(EntityID entity_id, float old_x, float old_y,
                    float new_x, float new_y) {
        auto* old_cell = GetCell(old_x, old_y);
        auto* new_cell = GetCell(new_x, new_y);

        if (!old_cell || !new_cell) return;

        // Same cell - no action needed
        if (old_cell == new_cell) return;

        // Remove from old cell
        old_cell->entities.erase(entity_id);

        // Add to new cell
        new_cell->entities.insert(entity_id);

        // Check if crossing server boundary
        if (old_cell->server_id != new_cell->server_id) {
            HandleServerTransition(entity_id,
                old_cell->server_id,
                new_cell->server_id);
        }
    }

private:
    void HandleServerTransition(EntityID entity_id,
                               const std::string& from_server,
                               const std::string& to_server) {
        // Serialize entity state
        auto entity_state = SerializeEntity(entity_id);

        // Send migration request to target server
        SendMigrationRequest(to_server, entity_id, entity_state);

        // Remove entity from source server (after confirmation)
        // This is handled asynchronously to prevent data loss
    }

private:
    float cell_size_;
    int grid_width_;
    int grid_height_;
    int num_servers_ = 6;
    std::vector<GridCell> grid_;
};
```

### Hierarchical Partitioning (Quadtree)

**Concept:** Dynamically subdivide space based on entity density.

```
Quadtree Partitioning
┌─────────────────────────────────────┐
│                                     │
│  ┌───────────┬────────────────────┐ │
│  │           │                    │ │
│  │  Zone A   │                    │ │
│  │  (Low     │     Zone B         │ │
│  │  density) │     (Medium)       │ │
│  │           │                    │ │
│  ├─────┬─────┤                    │ │
│  │ A-1 │ A-2 │                    │ │
│  │     │     │                    │ │
│  └─────┴─────┴────────────────────┘ │
│  ┌──────┬──────┬──────┬──────────┐  │
│  │ C-1  │ C-2  │ C-3  │          │  │
│  ├──────┼──────┼──────┤  Zone D  │  │
│  │ C-4  │ C-5  │ C-6  │  (Low)   │  │
│  ├──────┴──────┴──────┤          │  │
│  │     Zone C         │          │  │
│  │     (High density) │          │  │
│  └────────────────────┴──────────┘  │
│                                     │
└─────────────────────────────────────┘

Splits when: entities > threshold (e.g., 500)
Merges when: entities < threshold / 4
```

**Implementation:**

```cpp
// Quadtree-based dynamic partitioning
class QuadtreeZone {
public:
    struct Bounds {
        float min_x, min_y;
        float max_x, max_y;

        bool Contains(float x, float y) const {
            return x >= min_x && x <= max_x && y >= min_y && y <= max_y;
        }
    };

    QuadtreeZone(Bounds bounds, int max_entities = 500, int max_depth = 8)
        : bounds_(bounds),
          max_entities_(max_entities),
          max_depth_(max_depth),
          depth_(0),
          is_leaf_(true) {}

    void Insert(EntityID entity_id, float x, float y) {
        if (!bounds_.Contains(x, y)) return;

        if (is_leaf_) {
            entities_.insert(entity_id);
            entity_positions_[entity_id] = {x, y};

            // Check if we need to subdivide
            if (entities_.size() > max_entities_ && depth_ < max_depth_) {
                Subdivide();
            }
        } else {
            // Insert into appropriate child
            for (auto& child : children_) {
                if (child->bounds_.Contains(x, y)) {
                    child->Insert(entity_id, x, y);
                    return;
                }
            }
        }
    }

    void Remove(EntityID entity_id) {
        if (is_leaf_) {
            entities_.erase(entity_id);
            entity_positions_.erase(entity_id);
        } else {
            for (auto& child : children_) {
                child->Remove(entity_id);
            }

            // Check if we should merge
            size_t total_entities = GetTotalEntityCount();
            if (total_entities < max_entities_ / 4) {
                Merge();
            }
        }
    }

    // Query entities in radius
    std::vector<EntityID> QueryRadius(float x, float y, float radius) {
        std::vector<EntityID> result;

        if (!BoundsIntersectsCircle(bounds_, x, y, radius)) {
            return result; // No intersection
        }

        if (is_leaf_) {
            // Check all entities in this leaf
            for (auto entity_id : entities_) {
                auto [ex, ey] = entity_positions_[entity_id];
                float dist_sq = (ex - x) * (ex - x) + (ey - y) * (ey - y);
                if (dist_sq <= radius * radius) {
                    result.push_back(entity_id);
                }
            }
        } else {
            // Recursively query children
            for (auto& child : children_) {
                auto child_result = child->QueryRadius(x, y, radius);
                result.insert(result.end(), child_result.begin(), child_result.end());
            }
        }

        return result;
    }

private:
    void Subdivide() {
        float mid_x = (bounds_.min_x + bounds_.max_x) / 2.0f;
        float mid_y = (bounds_.min_y + bounds_.max_y) / 2.0f;

        // Create 4 children (NW, NE, SW, SE)
        children_.push_back(std::make_unique<QuadtreeZone>(
            Bounds{bounds_.min_x, mid_y, mid_x, bounds_.max_y},
            max_entities_, max_depth_));

        children_.push_back(std::make_unique<QuadtreeZone>(
            Bounds{mid_x, mid_y, bounds_.max_x, bounds_.max_y},
            max_entities_, max_depth_));

        children_.push_back(std::make_unique<QuadtreeZone>(
            Bounds{bounds_.min_x, bounds_.min_y, mid_x, mid_y},
            max_entities_, max_depth_));

        children_.push_back(std::make_unique<QuadtreeZone>(
            Bounds{mid_x, bounds_.min_y, bounds_.max_x, mid_y},
            max_entities_, max_depth_));

        // Set depth for children
        for (auto& child : children_) {
            child->depth_ = depth_ + 1;
        }

        // Redistribute entities to children
        for (auto entity_id : entities_) {
            auto [x, y] = entity_positions_[entity_id];
            for (auto& child : children_) {
                if (child->bounds_.Contains(x, y)) {
                    child->Insert(entity_id, x, y);
                    break;
                }
            }
        }

        // Clear local storage
        entities_.clear();
        entity_positions_.clear();
        is_leaf_ = false;
    }

    void Merge() {
        if (is_leaf_) return;

        // Collect all entities from children
        for (auto& child : children_) {
            if (child->is_leaf_) {
                for (auto entity_id : child->entities_) {
                    entities_.insert(entity_id);
                    entity_positions_[entity_id] = child->entity_positions_[entity_id];
                }
            }
        }

        // Remove children
        children_.clear();
        is_leaf_ = true;
    }

    size_t GetTotalEntityCount() const {
        if (is_leaf_) {
            return entities_.size();
        }

        size_t count = 0;
        for (const auto& child : children_) {
            count += child->GetTotalEntityCount();
        }
        return count;
    }

    bool BoundsIntersectsCircle(const Bounds& bounds, float cx, float cy, float radius) {
        float closest_x = std::clamp(cx, bounds.min_x, bounds.max_x);
        float closest_y = std::clamp(cy, bounds.min_y, bounds.max_y);

        float dist_sq = (cx - closest_x) * (cx - closest_x) +
                        (cy - closest_y) * (cy - closest_y);

        return dist_sq <= radius * radius;
    }

private:
    Bounds bounds_;
    int max_entities_;
    int max_depth_;
    int depth_;
    bool is_leaf_;

    // Leaf node data
    std::unordered_set<EntityID> entities_;
    std::unordered_map<EntityID, std::pair<float, float>> entity_positions_;

    // Internal node data
    std::vector<std::unique_ptr<QuadtreeZone>> children_;
};
```

## Sharding Strategies

### Database Sharding

**Horizontal Sharding (by Player ID):**

```
Player ID Range Sharding
┌────────────────────────────────────────┐
│  Shard 1: Player IDs 0 - 999,999       │
│  Server: db-shard-1                    │
│  Storage: 500GB                        │
└────────────────────────────────────────┘
┌────────────────────────────────────────┐
│  Shard 2: Player IDs 1M - 1,999,999    │
│  Server: db-shard-2                    │
│  Storage: 500GB                        │
└────────────────────────────────────────┘
┌────────────────────────────────────────┐
│  Shard 3: Player IDs 2M - 2,999,999    │
│  Server: db-shard-3                    │
│  Storage: 500GB                        │
└────────────────────────────────────────┘
┌────────────────────────────────────────┐
│  Shard N: Player IDs ...               │
│  Server: db-shard-N                    │
│  Storage: 500GB                        │
└────────────────────────────────────────┘
```

**Implementation:**

```cpp
// Shard router - routes database queries to correct shard
class DatabaseShardRouter {
public:
    DatabaseShardRouter() {
        // Initialize shard connections
        shards_.push_back(std::make_unique<DatabaseConnection>("db-shard-1"));
        shards_.push_back(std::make_unique<DatabaseConnection>("db-shard-2"));
        shards_.push_back(std::make_unique<DatabaseConnection>("db-shard-3"));
        shards_.push_back(std::make_unique<DatabaseConnection>("db-shard-4"));
    }

    // Get shard for player ID (consistent hashing)
    DatabaseConnection* GetShardForPlayer(uint64_t player_id) {
        size_t shard_index = player_id % shards_.size();
        return shards_[shard_index].get();
    }

    // Get player data
    std::optional<PlayerData> GetPlayerData(uint64_t player_id) {
        auto* shard = GetShardForPlayer(player_id);

        auto result = shard->Query(
            "SELECT * FROM players WHERE player_id = ?",
            player_id
        );

        if (result.empty()) {
            return std::nullopt;
        }

        return PlayerData::FromRow(result[0]);
    }

    // Update player data
    bool UpdatePlayerData(uint64_t player_id, const PlayerData& data) {
        auto* shard = GetShardForPlayer(player_id);

        return shard->Execute(
            "UPDATE players SET level = ?, xp = ?, gold = ? WHERE player_id = ?",
            data.level, data.xp, data.gold, player_id
        );
    }

    // Multi-shard query (e.g., leaderboard)
    std::vector<PlayerData> GetTopPlayers(int limit) {
        std::vector<PlayerData> all_players;

        // Query each shard in parallel
        std::vector<std::future<std::vector<PlayerData>>> futures;

        for (auto& shard : shards_) {
            futures.push_back(std::async(std::launch::async, [&shard, limit]() {
                std::vector<PlayerData> players;

                auto result = shard->Query(
                    "SELECT * FROM players ORDER BY xp DESC LIMIT ?",
                    limit
                );

                for (auto& row : result) {
                    players.push_back(PlayerData::FromRow(row));
                }

                return players;
            }));
        }

        // Collect results
        for (auto& future : futures) {
            auto shard_players = future.get();
            all_players.insert(all_players.end(),
                shard_players.begin(),
                shard_players.end());
        }

        // Sort and return top N
        std::sort(all_players.begin(), all_players.end(),
            [](const PlayerData& a, const PlayerData& b) {
                return a.xp > b.xp;
            });

        if (all_players.size() > limit) {
            all_players.resize(limit);
        }

        return all_players;
    }

private:
    std::vector<std::unique_ptr<DatabaseConnection>> shards_;
};
```

### Realm/Server Sharding

**Concept:** Separate game worlds (realms) with independent state.

```
Realm Architecture
┌──────────────────────────────────────────────────┐
│              Login Server (Global)               │
│  • Authentication                                 │
│  • Realm selection                                │
│  • Character list (cross-realm)                   │
└─────────┬────────────────────────────────────────┘
          │
    ┌─────┴──────┬──────────┬──────────┐
    │            │          │          │
    ▼            ▼          ▼          ▼
┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐
│ Realm 1 │ │ Realm 2 │ │ Realm 3 │ │ Realm N │
│ "East"  │ │ "West"  │ │ "EU"    │ │ "Asia"  │
├─────────┤ ├─────────┤ ├─────────┤ ├─────────┤
│ 5000    │ │ 4500    │ │ 6000    │ │ 3000    │
│ players │ │ players │ │ players │ │ players │
├─────────┤ ├─────────┤ ├─────────┤ ├─────────┤
│ Own DB  │ │ Own DB  │ │ Own DB  │ │ Own DB  │
└─────────┘ └─────────┘ └─────────┘ └─────────┘

Independent game worlds
Players cannot interact across realms (without special features)
```

**Cross-Realm Features:**

```cpp
// Cross-realm system for matchmaking, auction house, etc.
class CrossRealmService {
public:
    // Find match across all realms
    std::optional<MatchInfo> FindMatch(uint64_t player_id,
                                       const MatchmakingCriteria& criteria) {
        // Get player's home realm
        auto home_realm = GetPlayerRealm(player_id);

        // Search for match on home realm first
        auto match = home_realm->FindMatch(criteria);
        if (match.has_value()) {
            return match;
        }

        // Search other realms
        for (auto& realm : realms_) {
            if (realm.get() == home_realm) continue;

            match = realm->FindMatch(criteria);
            if (match.has_value()) {
                // Create cross-realm instance
                return CreateCrossRealmMatch(match.value(), player_id);
            }
        }

        return std::nullopt;
    }

    // Create instance server that players from different realms can join
    MatchInfo CreateCrossRealmMatch(const MatchInfo& match, uint64_t player_id) {
        // Allocate instance server
        auto instance_server = instance_pool_.Allocate();

        // Migrate players to instance
        for (auto pid : match.player_ids) {
            MigratePlayerToInstance(pid, instance_server->id);
        }

        MigratePlayerToInstance(player_id, instance_server->id);

        return match;
    }

private:
    std::vector<std::unique_ptr<RealmServer>> realms_;
    InstanceServerPool instance_pool_;
};
```

## Zone Management

### Zone Server Architecture

```cpp
// Zone server manages a portion of the game world
class ZoneServer {
public:
    ZoneServer(ZoneID zone_id, const ZoneBounds& bounds)
        : zone_id_(zone_id),
          bounds_(bounds),
          tick_rate_(20), // 20 Hz
          running_(false) {}

    void Start() {
        running_ = true;

        // Main simulation thread
        simulation_thread_ = std::thread([this]() {
            SimulationLoop();
        });

        // Network I/O thread
        network_thread_ = std::thread([this]() {
            NetworkLoop();
        });

        // Database persistence thread
        db_thread_ = std::thread([this]() {
            PersistenceLoop();
        });
    }

    void Stop() {
        running_ = false;
        if (simulation_thread_.joinable()) simulation_thread_.join();
        if (network_thread_.joinable()) network_thread_.join();
        if (db_thread_.joinable()) db_thread_.join();
    }

    // Add player to zone
    void AddPlayer(uint64_t player_id, const PlayerState& state) {
        std::lock_guard<std::mutex> lock(players_mutex_);

        players_[player_id] = state;

        // Add to spatial index
        spatial_index_.Insert(player_id, state.position.x, state.position.y);

        // Notify nearby players
        NotifyNearbyPlayers(player_id, PlayerEvent::Enter);
    }

    // Remove player from zone
    void RemovePlayer(uint64_t player_id) {
        std::lock_guard<std::mutex> lock(players_mutex_);

        auto it = players_.find(player_id);
        if (it == players_.end()) return;

        // Remove from spatial index
        spatial_index_.Remove(player_id);

        // Notify nearby players
        NotifyNearbyPlayers(player_id, PlayerEvent::Leave);

        players_.erase(it);
    }

    // Handle player movement
    void UpdatePlayerPosition(uint64_t player_id, const Vector3& new_position) {
        std::lock_guard<std::mutex> lock(players_mutex_);

        auto it = players_.find(player_id);
        if (it == players_.end()) return;

        auto& player = it->second;
        Vector3 old_position = player.position;
        player.position = new_position;

        // Update spatial index
        spatial_index_.Update(player_id, old_position.x, old_position.y,
                            new_position.x, new_position.y);

        // Check if player left zone bounds
        if (!bounds_.Contains(new_position)) {
            HandleZoneTransition(player_id, new_position);
        }
    }

private:
    void SimulationLoop() {
        auto tick_interval = std::chrono::milliseconds(1000 / tick_rate_);
        auto next_tick = std::chrono::steady_clock::now();

        while (running_) {
            // Process player actions
            ProcessPlayerActions();

            // Update NPCs
            UpdateNPCs();

            // Update monsters
            UpdateMonsters();

            // Process combat
            ProcessCombat();

            // Update world events
            UpdateWorldEvents();

            // Interest management
            UpdatePlayerInterests();

            next_tick += tick_interval;
            std::this_thread::sleep_until(next_tick);
        }
    }

    void NetworkLoop() {
        while (running_) {
            // Receive packets from players
            ReceivePackets();

            // Send state updates
            SendStateUpdates();

            std::this_thread::sleep_for(std::chrono::milliseconds(10));
        }
    }

    void PersistenceLoop() {
        while (running_) {
            // Periodic save of player states
            SavePlayerStates();

            // Save every 5 minutes
            std::this_thread::sleep_for(std::chrono::minutes(5));
        }
    }

    void HandleZoneTransition(uint64_t player_id, const Vector3& position) {
        // Determine target zone
        auto target_zone = world_->GetZoneAtPosition(position);
        if (!target_zone) {
            // Out of bounds - teleport back
            TeleportToSafePosition(player_id);
            return;
        }

        if (target_zone->zone_id == zone_id_) {
            // Still in same zone (just outside bounds temporarily)
            return;
        }

        // Initiate zone transfer
        auto player_state = players_[player_id];

        // Send transfer request to target zone
        TransferPlayerToZone(player_id, player_state, target_zone->zone_id);

        // Remove from this zone
        RemovePlayer(player_id);
    }

    void TransferPlayerToZone(uint64_t player_id,
                             const PlayerState& state,
                             ZoneID target_zone_id) {
        // Serialize player state
        auto serialized_state = SerializePlayerState(state);

        // Send to zone coordinator
        zone_coordinator_->RequestPlayerTransfer(
            player_id,
            zone_id_,
            target_zone_id,
            serialized_state
        );
    }

    void UpdatePlayerInterests() {
        std::lock_guard<std::mutex> lock(players_mutex_);

        for (auto& [player_id, player] : players_) {
            // Query nearby entities
            auto nearby = spatial_index_.QueryRadius(
                player.position.x,
                player.position.y,
                player.visibility_radius
            );

            // Calculate priority for each entity
            std::vector<InterestEntity> interests;
            for (auto entity_id : nearby) {
                float distance = CalculateDistance(player.position,
                    GetEntityPosition(entity_id));

                float priority = CalculateInterestPriority(entity_id, distance);

                interests.push_back({entity_id, priority});
            }

            // Sort by priority
            std::sort(interests.begin(), interests.end(),
                [](const InterestEntity& a, const InterestEntity& b) {
                    return a.priority > b.priority;
                });

            // Limit to max visible entities
            if (interests.size() > max_visible_entities_) {
                interests.resize(max_visible_entities_);
            }

            player.interest_set = std::move(interests);
        }
    }

    void NotifyNearbyPlayers(uint64_t player_id, PlayerEvent event) {
        auto it = players_.find(player_id);
        if (it == players_.end()) return;

        const auto& player = it->second;

        // Find nearby players
        auto nearby = spatial_index_.QueryRadius(
            player.position.x,
            player.position.y,
            100.0f // notification radius
        );

        // Send notification to each nearby player
        for (auto nearby_id : nearby) {
            if (nearby_id == player_id) continue;

            SendPlayerEvent(nearby_id, player_id, event);
        }
    }

private:
    ZoneID zone_id_;
    ZoneBounds bounds_;
    int tick_rate_;
    std::atomic<bool> running_;

    // Threading
    std::thread simulation_thread_;
    std::thread network_thread_;
    std::thread db_thread_;

    // Entities
    std::mutex players_mutex_;
    std::unordered_map<uint64_t, PlayerState> players_;
    std::vector<NPC> npcs_;
    std::vector<Monster> monsters_;

    // Spatial indexing
    SpatialHashGrid spatial_index_;

    // Configuration
    size_t max_visible_entities_ = 200;

    // External services
    WorldCoordinator* world_;
    ZoneCoordinator* zone_coordinator_;
};
```

## Cross-Zone Communication

### Message Passing Between Zones

```cpp
// Zone coordinator manages communication between zones
class ZoneCoordinator {
public:
    void Start() {
        // Start message processing thread
        message_thread_ = std::thread([this]() {
            MessageProcessingLoop();
        });
    }

    // Send message to another zone
    void SendMessageToZone(ZoneID target_zone, const ZoneMessage& message) {
        auto* target = GetZoneServer(target_zone);
        if (!target) {
            LogError("Zone not found: " + std::to_string(target_zone));
            return;
        }

        // Use lock-free queue for cross-zone messages
        target->message_queue_.push(message);
    }

    // Broadcast message to all zones
    void BroadcastMessage(const ZoneMessage& message) {
        for (auto& [zone_id, zone_server] : zone_servers_) {
            zone_server->message_queue_.push(message);
        }
    }

    // Handle player transfer between zones
    void RequestPlayerTransfer(uint64_t player_id,
                               ZoneID from_zone,
                               ZoneID to_zone,
                               const std::string& serialized_state) {
        // Add to transfer queue
        TransferRequest request{
            player_id,
            from_zone,
            to_zone,
            serialized_state,
            std::chrono::steady_clock::now()
        };

        transfer_queue_.push(request);
    }

private:
    void MessageProcessingLoop() {
        while (running_) {
            // Process zone messages
            ProcessZoneMessages();

            // Process player transfers
            ProcessPlayerTransfers();

            std::this_thread::sleep_for(std::chrono::milliseconds(10));
        }
    }

    void ProcessPlayerTransfers() {
        TransferRequest request;
        while (transfer_queue_.try_pop(request)) {
            // Get target zone server
            auto* target_zone = GetZoneServer(request.to_zone);
            if (!target_zone) {
                LogError("Target zone not found");
                continue;
            }

            // Deserialize player state
            auto player_state = DeserializePlayerState(request.serialized_state);

            // Add player to target zone
            target_zone->AddPlayer(request.player_id, player_state);

            // Notify player of successful transfer
            NotifyPlayerTransferComplete(request.player_id, request.to_zone);

            // Log transfer
            LogPlayerTransfer(request);
        }
    }

private:
    std::atomic<bool> running_{true};
    std::thread message_thread_;

    std::unordered_map<ZoneID, ZoneServer*> zone_servers_;
    LockFreeQueue<TransferRequest> transfer_queue_;
};
```

## Load Balancing

### Dynamic Zone Splitting

```cpp
// Dynamic zone splitting when player density is too high
class DynamicZoneManager {
public:
    void MonitorZoneLoad() {
        for (auto& [zone_id, zone] : zones_) {
            size_t player_count = zone->GetPlayerCount();

            // Check if zone is overloaded
            if (player_count > zone_split_threshold_) {
                SplitZone(zone_id);
            }
            // Check if zone can be merged
            else if (player_count < zone_merge_threshold_) {
                ConsiderZoneMerge(zone_id);
            }
        }
    }

    void SplitZone(ZoneID zone_id) {
        auto* zone = zones_[zone_id];
        auto bounds = zone->GetBounds();

        // Split zone into 4 quadrants
        auto quadrants = SplitBounds(bounds, 2, 2);

        std::vector<ZoneID> new_zone_ids;

        for (size_t i = 0; i < quadrants.size(); ++i) {
            // Create new zone
            ZoneID new_zone_id = AllocateZoneID();
            auto new_zone = std::make_unique<ZoneServer>(new_zone_id, quadrants[i]);

            // Start new zone
            new_zone->Start();

            new_zone_ids.push_back(new_zone_id);
            zones_[new_zone_id] = std::move(new_zone);
        }

        // Migrate players to new zones based on position
        auto players = zone->GetAllPlayers();
        for (const auto& [player_id, player_state] : players) {
            // Find which new zone this player belongs to
            for (auto new_zone_id : new_zone_ids) {
                auto* new_zone = zones_[new_zone_id];
                if (new_zone->GetBounds().Contains(player_state.position)) {
                    // Transfer player
                    new_zone->AddPlayer(player_id, player_state);
                    break;
                }
            }
        }

        // Shutdown old zone
        zone->Stop();
        zones_.erase(zone_id);

        LogInfo("Split zone " + std::to_string(zone_id) +
                " into " + std::to_string(new_zone_ids.size()) + " zones");
    }

private:
    std::unordered_map<ZoneID, std::unique_ptr<ZoneServer>> zones_;
    size_t zone_split_threshold_ = 1000;   // Split when > 1000 players
    size_t zone_merge_threshold_ = 100;    // Merge when < 100 players
};
```

## Case Studies

### World of Warcraft (Classic)

**Architecture:**
- Realm-based sharding (isolated game worlds)
- Zone-based world servers (Continent per server initially)
- Later: Dynamic zone instancing (phasing)
- Cross-realm zones (CRZ) for low-population areas

**Key Techniques:**
- Interest management (players only see nearby entities)
- Area-of-Interest (AOI) updates
- Database sharding by realm
- Layer technology for launch (temporary sharding)

**Scaling Challenges:**
- Launch congestion (queues, layers)
- Popular zone overload (Barrens, Stranglethorn Vale)
- World boss contention (single spawn, hundreds of players)

### EVE Online

**Architecture:**
- Single-shard universe (all players in one world)
- Solar system = zone (one thread per system)
- Dynamic load balancing (move busy systems to dedicated hardware)
- Time dilation (slow down time in overloaded systems)

**Key Innovations:**
- Stackless Python for massive concurrency
- Time dilation (10% speed when 2000+ players in system)
- Reinforced nodes (prepare hardware for planned battles)
- CREST API for third-party tools

**Record Battle:**
- Battle of B-R5RB (2014): 7,548 players, 21 hours, $300k+ destroyed

### Final Fantasy XIV

**Architecture:**
- Data center > World (server) > Instance
- Instanced zones for main story
- Shared world for open areas
- Cross-world party finder
- Cross-data-center travel

**Techniques:**
- Dynamic instancing (create new instance when zone is full)
- Instance merging (merge low-population instances)
- Server tick: 3Hz (yes, really!)
- Aggressive client-side prediction
- Snapshot interpolation

## Conclusion

MMO architecture requires careful balance between consistency, scalability, and performance. Modern MMOs use hybrid approaches combining static partitioning (grids, zones) with dynamic techniques (instancing, sharding, load balancing). The choice of architecture depends on game design (open world vs instanced, PvP vs PvE), player count, and budget.

## Further Reading

- [How Game Servers Work (MMO Architecture)](https://www.gabrielgambetta.com/client-server-game-architecture.html)
- [EVE Online Server Architecture](https://www.eveonline.com/news/view/tranquility-tech-3)
- [WoW Server Architecture (Interview)](https://www.youtube.com/watch?v=U3dNKk1gKQQ)
- [Spatial Partitioning in Games](https://gameprogrammingpatterns.com/spatial-partition.html)
- [Database Sharding Strategies](https://www.digitalocean.com/community/tutorials/understanding-database-sharding)
