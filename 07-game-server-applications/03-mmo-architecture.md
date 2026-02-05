# MMO 아키텍처: Sharding과 월드 파티셔닝

## 개요

대규모 다중 접속 온라인(MMO) 게임은 영속적이고 공유된 세계에서 수천 명의 동시 접속 플레이어를 지원해야 하기 때문에 서버 아키텍처에 있어 고유한 과제를 제시합니다. 이 문서에서는 sharding, 월드 파티셔닝, zone 관리, 크로스 서버 통신 전략을 포함한 고급 아키텍처 패턴을 다룹니다.

## 목차

1. [핵심 개념](#핵심-개념)
2. [월드 파티셔닝](#월드-파티셔닝)
3. [Sharding 전략](#sharding-전략)
4. [Zone 관리](#zone-관리)
5. [크로스 Zone 통신](#크로스-zone-통신)
6. [부하 분산](#부하-분산)
7. [사례 연구](#사례-연구)

## 핵심 개념

### MMO 서버 과제

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

### 아키텍처 계층

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

## 월드 파티셔닝

### 그리드 기반 파티셔닝

**개념:** 월드를 균일한 그리드로 분할하고, 각 셀을 zone 서버가 관리합니다.

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

**구현:**

```cpp
// 그리드 기반 월드 파티셔닝
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

        // 그리드 초기화
        grid_.resize(grid_width * grid_height);

        for (int y = 0; y < grid_height; ++y) {
            for (int x = 0; x < grid_width; ++x) {
                int index = y * grid_width + x;
                grid_[index].x = x;
                grid_[index].y = y;

                // 서버 할당 (라운드 로빈 분배)
                grid_[index].server_id = "server_" +
                    std::to_string(index % num_servers_);
            }
        }
    }

    // 월드 좌표로 그리드 셀 가져오기
    GridCell* GetCell(float world_x, float world_y) {
        int grid_x = static_cast<int>(world_x / cell_size_);
        int grid_y = static_cast<int>(world_y / cell_size_);

        if (grid_x < 0 || grid_x >= grid_width_ ||
            grid_y < 0 || grid_y >= grid_height_) {
            return nullptr; // 범위 밖
        }

        return &grid_[grid_y * grid_width_ + grid_x];
    }

    // 인접 셀 가져오기 (크로스 zone 가시성 용)
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

    // 셀 간 엔티티 이동 처리
    void MoveEntity(EntityID entity_id, float old_x, float old_y,
                    float new_x, float new_y) {
        auto* old_cell = GetCell(old_x, old_y);
        auto* new_cell = GetCell(new_x, new_y);

        if (!old_cell || !new_cell) return;

        // 같은 셀 - 처리 불필요
        if (old_cell == new_cell) return;

        // 이전 셀에서 제거
        old_cell->entities.erase(entity_id);

        // 새 셀에 추가
        new_cell->entities.insert(entity_id);

        // 서버 경계를 넘는지 확인
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
        // 엔티티 상태 직렬화
        auto entity_state = SerializeEntity(entity_id);

        // 대상 서버에 마이그레이션 요청 전송
        SendMigrationRequest(to_server, entity_id, entity_state);

        // 확인 후 소스 서버에서 엔티티 제거
        // 데이터 손실 방지를 위해 비동기로 처리
    }

private:
    float cell_size_;
    int grid_width_;
    int grid_height_;
    int num_servers_ = 6;
    std::vector<GridCell> grid_;
};
```

### 계층적 파티셔닝 (Quadtree)

**개념:** 엔티티 밀도에 따라 공간을 동적으로 세분화합니다.

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

**구현:**

```cpp
// Quadtree 기반 동적 파티셔닝
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

            // 세분화 필요 여부 확인
            if (entities_.size() > max_entities_ && depth_ < max_depth_) {
                Subdivide();
            }
        } else {
            // 적절한 자식에 삽입
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

            // 병합 여부 확인
            size_t total_entities = GetTotalEntityCount();
            if (total_entities < max_entities_ / 4) {
                Merge();
            }
        }
    }

    // 반경 내 엔티티 조회
    std::vector<EntityID> QueryRadius(float x, float y, float radius) {
        std::vector<EntityID> result;

        if (!BoundsIntersectsCircle(bounds_, x, y, radius)) {
            return result; // 교차 없음
        }

        if (is_leaf_) {
            // 이 리프의 모든 엔티티 확인
            for (auto entity_id : entities_) {
                auto [ex, ey] = entity_positions_[entity_id];
                float dist_sq = (ex - x) * (ex - x) + (ey - y) * (ey - y);
                if (dist_sq <= radius * radius) {
                    result.push_back(entity_id);
                }
            }
        } else {
            // 자식 노드를 재귀적으로 조회
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

        // 4개의 자식 생성 (NW, NE, SW, SE)
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

        // 자식 노드의 깊이 설정
        for (auto& child : children_) {
            child->depth_ = depth_ + 1;
        }

        // 엔티티를 자식 노드로 재분배
        for (auto entity_id : entities_) {
            auto [x, y] = entity_positions_[entity_id];
            for (auto& child : children_) {
                if (child->bounds_.Contains(x, y)) {
                    child->Insert(entity_id, x, y);
                    break;
                }
            }
        }

        // 로컬 스토리지 정리
        entities_.clear();
        entity_positions_.clear();
        is_leaf_ = false;
    }

    void Merge() {
        if (is_leaf_) return;

        // 자식에서 모든 엔티티 수집
        for (auto& child : children_) {
            if (child->is_leaf_) {
                for (auto entity_id : child->entities_) {
                    entities_.insert(entity_id);
                    entity_positions_[entity_id] = child->entity_positions_[entity_id];
                }
            }
        }

        // 자식 제거
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

    // 리프 노드 데이터
    std::unordered_set<EntityID> entities_;
    std::unordered_map<EntityID, std::pair<float, float>> entity_positions_;

    // 내부 노드 데이터
    std::vector<std::unique_ptr<QuadtreeZone>> children_;
};
```

## Sharding 전략

### 데이터베이스 Sharding

**수평 Sharding (플레이어 ID 기준):**

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

**구현:**

```cpp
// Shard 라우터 - 데이터베이스 쿼리를 올바른 shard로 라우팅
class DatabaseShardRouter {
public:
    DatabaseShardRouter() {
        // shard 연결 초기화
        shards_.push_back(std::make_unique<DatabaseConnection>("db-shard-1"));
        shards_.push_back(std::make_unique<DatabaseConnection>("db-shard-2"));
        shards_.push_back(std::make_unique<DatabaseConnection>("db-shard-3"));
        shards_.push_back(std::make_unique<DatabaseConnection>("db-shard-4"));
    }

    // 플레이어 ID에 대한 shard 가져오기 (consistent hashing)
    DatabaseConnection* GetShardForPlayer(uint64_t player_id) {
        size_t shard_index = player_id % shards_.size();
        return shards_[shard_index].get();
    }

    // 플레이어 데이터 가져오기
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

    // 플레이어 데이터 업데이트
    bool UpdatePlayerData(uint64_t player_id, const PlayerData& data) {
        auto* shard = GetShardForPlayer(player_id);

        return shard->Execute(
            "UPDATE players SET level = ?, xp = ?, gold = ? WHERE player_id = ?",
            data.level, data.xp, data.gold, player_id
        );
    }

    // 다중 shard 쿼리 (예: 리더보드)
    std::vector<PlayerData> GetTopPlayers(int limit) {
        std::vector<PlayerData> all_players;

        // 각 shard를 병렬로 쿼리
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

        // 결과 수집
        for (auto& future : futures) {
            auto shard_players = future.get();
            all_players.insert(all_players.end(),
                shard_players.begin(),
                shard_players.end());
        }

        // 정렬하고 상위 N개 반환
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

### Realm/서버 Sharding

**개념:** 독립적인 상태를 가진 별도의 게임 월드(realm)를 운영합니다.

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

**Cross-Realm 기능:**

```cpp
// 매치메이킹, 거래소 등을 위한 cross-realm 시스템
class CrossRealmService {
public:
    // 모든 realm에서 매치 찾기
    std::optional<MatchInfo> FindMatch(uint64_t player_id,
                                       const MatchmakingCriteria& criteria) {
        // 플레이어의 홈 realm 가져오기
        auto home_realm = GetPlayerRealm(player_id);

        // 홈 realm에서 먼저 매치 검색
        auto match = home_realm->FindMatch(criteria);
        if (match.has_value()) {
            return match;
        }

        // 다른 realm 검색
        for (auto& realm : realms_) {
            if (realm.get() == home_realm) continue;

            match = realm->FindMatch(criteria);
            if (match.has_value()) {
                // Cross-realm 인스턴스 생성
                return CreateCrossRealmMatch(match.value(), player_id);
            }
        }

        return std::nullopt;
    }

    // 다른 realm의 플레이어들이 참가할 수 있는 인스턴스 서버 생성
    MatchInfo CreateCrossRealmMatch(const MatchInfo& match, uint64_t player_id) {
        // 인스턴스 서버 할당
        auto instance_server = instance_pool_.Allocate();

        // 플레이어를 인스턴스로 마이그레이션
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

## Zone 관리

### Zone 서버 아키텍처

```cpp
// Zone 서버는 게임 월드의 일부를 관리
class ZoneServer {
public:
    ZoneServer(ZoneID zone_id, const ZoneBounds& bounds)
        : zone_id_(zone_id),
          bounds_(bounds),
          tick_rate_(20), // 20 Hz
          running_(false) {}

    void Start() {
        running_ = true;

        // 메인 시뮬레이션 스레드
        simulation_thread_ = std::thread([this]() {
            SimulationLoop();
        });

        // 네트워크 I/O 스레드
        network_thread_ = std::thread([this]() {
            NetworkLoop();
        });

        // 데이터베이스 영속화 스레드
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

    // zone에 플레이어 추가
    void AddPlayer(uint64_t player_id, const PlayerState& state) {
        std::lock_guard<std::mutex> lock(players_mutex_);

        players_[player_id] = state;

        // 공간 인덱스에 추가
        spatial_index_.Insert(player_id, state.position.x, state.position.y);

        // 근처 플레이어에게 알림
        NotifyNearbyPlayers(player_id, PlayerEvent::Enter);
    }

    // zone에서 플레이어 제거
    void RemovePlayer(uint64_t player_id) {
        std::lock_guard<std::mutex> lock(players_mutex_);

        auto it = players_.find(player_id);
        if (it == players_.end()) return;

        // 공간 인덱스에서 제거
        spatial_index_.Remove(player_id);

        // 근처 플레이어에게 알림
        NotifyNearbyPlayers(player_id, PlayerEvent::Leave);

        players_.erase(it);
    }

    // 플레이어 이동 처리
    void UpdatePlayerPosition(uint64_t player_id, const Vector3& new_position) {
        std::lock_guard<std::mutex> lock(players_mutex_);

        auto it = players_.find(player_id);
        if (it == players_.end()) return;

        auto& player = it->second;
        Vector3 old_position = player.position;
        player.position = new_position;

        // 공간 인덱스 업데이트
        spatial_index_.Update(player_id, old_position.x, old_position.y,
                            new_position.x, new_position.y);

        // 플레이어가 zone 경계를 벗어났는지 확인
        if (!bounds_.Contains(new_position)) {
            HandleZoneTransition(player_id, new_position);
        }
    }

private:
    void SimulationLoop() {
        auto tick_interval = std::chrono::milliseconds(1000 / tick_rate_);
        auto next_tick = std::chrono::steady_clock::now();

        while (running_) {
            // 플레이어 액션 처리
            ProcessPlayerActions();

            // NPC 업데이트
            UpdateNPCs();

            // 몬스터 업데이트
            UpdateMonsters();

            // 전투 처리
            ProcessCombat();

            // 월드 이벤트 업데이트
            UpdateWorldEvents();

            // 관심 영역 관리
            UpdatePlayerInterests();

            next_tick += tick_interval;
            std::this_thread::sleep_until(next_tick);
        }
    }

    void NetworkLoop() {
        while (running_) {
            // 플레이어로부터 패킷 수신
            ReceivePackets();

            // 상태 업데이트 전송
            SendStateUpdates();

            std::this_thread::sleep_for(std::chrono::milliseconds(10));
        }
    }

    void PersistenceLoop() {
        while (running_) {
            // 플레이어 상태 주기적 저장
            SavePlayerStates();

            // 5분마다 저장
            std::this_thread::sleep_for(std::chrono::minutes(5));
        }
    }

    void HandleZoneTransition(uint64_t player_id, const Vector3& position) {
        // 대상 zone 결정
        auto target_zone = world_->GetZoneAtPosition(position);
        if (!target_zone) {
            // 범위 밖 - 안전한 위치로 텔레포트
            TeleportToSafePosition(player_id);
            return;
        }

        if (target_zone->zone_id == zone_id_) {
            // 아직 같은 zone (일시적으로 경계 밖)
            return;
        }

        // zone 전환 시작
        auto player_state = players_[player_id];

        // 대상 zone에 전환 요청 전송
        TransferPlayerToZone(player_id, player_state, target_zone->zone_id);

        // 이 zone에서 제거
        RemovePlayer(player_id);
    }

    void TransferPlayerToZone(uint64_t player_id,
                             const PlayerState& state,
                             ZoneID target_zone_id) {
        // 플레이어 상태 직렬화
        auto serialized_state = SerializePlayerState(state);

        // zone 코디네이터에 전송
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
            // 근처 엔티티 조회
            auto nearby = spatial_index_.QueryRadius(
                player.position.x,
                player.position.y,
                player.visibility_radius
            );

            // 각 엔티티에 대한 우선순위 계산
            std::vector<InterestEntity> interests;
            for (auto entity_id : nearby) {
                float distance = CalculateDistance(player.position,
                    GetEntityPosition(entity_id));

                float priority = CalculateInterestPriority(entity_id, distance);

                interests.push_back({entity_id, priority});
            }

            // 우선순위로 정렬
            std::sort(interests.begin(), interests.end(),
                [](const InterestEntity& a, const InterestEntity& b) {
                    return a.priority > b.priority;
                });

            // 최대 표시 엔티티 수로 제한
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

        // 근처 플레이어 찾기
        auto nearby = spatial_index_.QueryRadius(
            player.position.x,
            player.position.y,
            100.0f // 알림 반경
        );

        // 근처 각 플레이어에게 알림 전송
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

    // 스레딩
    std::thread simulation_thread_;
    std::thread network_thread_;
    std::thread db_thread_;

    // 엔티티
    std::mutex players_mutex_;
    std::unordered_map<uint64_t, PlayerState> players_;
    std::vector<NPC> npcs_;
    std::vector<Monster> monsters_;

    // 공간 인덱싱
    SpatialHashGrid spatial_index_;

    // 설정
    size_t max_visible_entities_ = 200;

    // 외부 서비스
    WorldCoordinator* world_;
    ZoneCoordinator* zone_coordinator_;
};
```

## 크로스 Zone 통신

### Zone 간 메시지 전달

```cpp
// Zone 코디네이터는 zone 간 통신을 관리
class ZoneCoordinator {
public:
    void Start() {
        // 메시지 처리 스레드 시작
        message_thread_ = std::thread([this]() {
            MessageProcessingLoop();
        });
    }

    // 다른 zone에 메시지 전송
    void SendMessageToZone(ZoneID target_zone, const ZoneMessage& message) {
        auto* target = GetZoneServer(target_zone);
        if (!target) {
            LogError("Zone not found: " + std::to_string(target_zone));
            return;
        }

        // 크로스 zone 메시지에 lock-free 큐 사용
        target->message_queue_.push(message);
    }

    // 모든 zone에 메시지 방송
    void BroadcastMessage(const ZoneMessage& message) {
        for (auto& [zone_id, zone_server] : zone_servers_) {
            zone_server->message_queue_.push(message);
        }
    }

    // zone 간 플레이어 전환 처리
    void RequestPlayerTransfer(uint64_t player_id,
                               ZoneID from_zone,
                               ZoneID to_zone,
                               const std::string& serialized_state) {
        // 전환 큐에 추가
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
            // zone 메시지 처리
            ProcessZoneMessages();

            // 플레이어 전환 처리
            ProcessPlayerTransfers();

            std::this_thread::sleep_for(std::chrono::milliseconds(10));
        }
    }

    void ProcessPlayerTransfers() {
        TransferRequest request;
        while (transfer_queue_.try_pop(request)) {
            // 대상 zone 서버 가져오기
            auto* target_zone = GetZoneServer(request.to_zone);
            if (!target_zone) {
                LogError("Target zone not found");
                continue;
            }

            // 플레이어 상태 역직렬화
            auto player_state = DeserializePlayerState(request.serialized_state);

            // 대상 zone에 플레이어 추가
            target_zone->AddPlayer(request.player_id, player_state);

            // 플레이어에게 전환 완료 알림
            NotifyPlayerTransferComplete(request.player_id, request.to_zone);

            // 전환 로그
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

## 부하 분산

### 동적 Zone 분할

```cpp
// 플레이어 밀도가 너무 높을 때 동적 zone 분할
class DynamicZoneManager {
public:
    void MonitorZoneLoad() {
        for (auto& [zone_id, zone] : zones_) {
            size_t player_count = zone->GetPlayerCount();

            // zone 과부하 확인
            if (player_count > zone_split_threshold_) {
                SplitZone(zone_id);
            }
            // zone 병합 가능 여부 확인
            else if (player_count < zone_merge_threshold_) {
                ConsiderZoneMerge(zone_id);
            }
        }
    }

    void SplitZone(ZoneID zone_id) {
        auto* zone = zones_[zone_id];
        auto bounds = zone->GetBounds();

        // zone을 4개 사분면으로 분할
        auto quadrants = SplitBounds(bounds, 2, 2);

        std::vector<ZoneID> new_zone_ids;

        for (size_t i = 0; i < quadrants.size(); ++i) {
            // 새 zone 생성
            ZoneID new_zone_id = AllocateZoneID();
            auto new_zone = std::make_unique<ZoneServer>(new_zone_id, quadrants[i]);

            // 새 zone 시작
            new_zone->Start();

            new_zone_ids.push_back(new_zone_id);
            zones_[new_zone_id] = std::move(new_zone);
        }

        // 위치에 따라 플레이어를 새 zone으로 마이그레이션
        auto players = zone->GetAllPlayers();
        for (const auto& [player_id, player_state] : players) {
            // 이 플레이어가 속하는 새 zone 찾기
            for (auto new_zone_id : new_zone_ids) {
                auto* new_zone = zones_[new_zone_id];
                if (new_zone->GetBounds().Contains(player_state.position)) {
                    // 플레이어 전환
                    new_zone->AddPlayer(player_id, player_state);
                    break;
                }
            }
        }

        // 이전 zone 종료
        zone->Stop();
        zones_.erase(zone_id);

        LogInfo("Split zone " + std::to_string(zone_id) +
                " into " + std::to_string(new_zone_ids.size()) + " zones");
    }

private:
    std::unordered_map<ZoneID, std::unique_ptr<ZoneServer>> zones_;
    size_t zone_split_threshold_ = 1000;   // 1000명 초과 시 분할
    size_t zone_merge_threshold_ = 100;    // 100명 미만 시 병합
};
```

## 사례 연구

### 월드 오브 워크래프트 (클래식)

**아키텍처:**
- Realm 기반 sharding (독립된 게임 월드)
- Zone 기반 월드 서버 (초기에는 대륙당 서버)
- 이후: 동적 zone 인스턴싱 (페이징)
- 저인구 지역을 위한 Cross-realm zone (CRZ)

**핵심 기법:**
- 관심 영역 관리 (플레이어는 근처 엔티티만 봄)
- Area-of-Interest (AOI) 업데이트
- Realm별 데이터베이스 sharding
- 출시를 위한 레이어 기술 (임시 sharding)

**확장 과제:**
- 출시 혼잡 (대기열, 레이어)
- 인기 zone 과부하 (불모의 땅, 가시덤불 골짜기)
- 월드 보스 경쟁 (단일 스폰, 수백 명의 플레이어)

### EVE Online

**아키텍처:**
- 단일 shard 우주 (모든 플레이어가 하나의 세계에)
- 태양계 = zone (시스템당 하나의 스레드)
- 동적 부하 분산 (바쁜 시스템을 전용 하드웨어로 이동)
- 시간 팽창 (과부하된 시스템에서 시간을 늦춤)

**핵심 혁신:**
- 대규모 동시성을 위한 Stackless Python
- 시간 팽창 (2000명 이상 시 10% 속도)
- 강화 노드 (계획된 전투를 위한 하드웨어 준비)
- 서드파티 도구를 위한 CREST API

**기록적 전투:**
- B-R5RB 전투 (2014): 7,548명 참여, 21시간, $300k 이상 파괴

### 파이널 판타지 XIV

**아키텍처:**
- 데이터 센터 > 월드 (서버) > 인스턴스
- 메인 스토리를 위한 인스턴스 zone
- 오픈 지역을 위한 공유 월드
- Cross-world 파티 파인더
- Cross-data-center 이동

**기법:**
- 동적 인스턴싱 (zone이 가득 차면 새 인스턴스 생성)
- 인스턴스 병합 (저인구 인스턴스 병합)
- 서버 tick: 3Hz (정말로!)
- 적극적인 클라이언트 측 예측
- 스냅샷 보간

## 결론

MMO 아키텍처는 일관성, 확장성, 성능 간의 신중한 균형을 필요로 합니다. 현대 MMO는 정적 파티셔닝(그리드, zone)과 동적 기법(인스턴싱, sharding, 부하 분산)을 결합한 하이브리드 접근 방식을 사용합니다. 아키텍처 선택은 게임 설계(오픈 월드 vs 인스턴스, PvP vs PvE), 플레이어 수, 예산에 따라 달라집니다.

## 추가 참고 자료

- [How Game Servers Work (MMO Architecture)](https://www.gabrielgambetta.com/client-server-game-architecture.html)
- [EVE Online Server Architecture](https://www.eveonline.com/news/view/tranquility-tech-3)
- [WoW Server Architecture (Interview)](https://www.youtube.com/watch?v=U3dNKk1gKQQ)
- [Spatial Partitioning in Games](https://gameprogrammingpatterns.com/spatial-partition.html)
- [Database Sharding Strategies](https://www.digitalocean.com/community/tutorials/understanding-database-sharding)
