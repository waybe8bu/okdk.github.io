+++
title = 'TrinityCore 如何处理怪物 Engage 玩家：CreatureAI::MoveInLineOfSight的调用链路'
date = 2024-01-02T00:00:00+00:00
+++
指出CreatureAI::MoveInLineOfSight的调用链路

我来帮你分析`CreatureAI::MoveInLineOfSight`的调用链路。让我搜索相关的调用代码。 现在让我查看Grid通知机制，这是`MoveInLineOfSight`被调用的关键地方： 现在让我查看`CreatureUnitRelocationWorker`函数： 现在让我查看这些Grid通知器是在哪里被调用的： 让我查看这些通知器在哪里被触发： 现在让我搜索这些通知器是在什么时候被调用的： 现在让我查看具体的调用点，特别是Unit和Player中的调用： 现在让我查看Object类中的调用： 现在让我查看Map的Relocation方法： 让我查看Map的Update方法，那里应该有DelayedUnitRelocation的调用： 现在让我搜索哪里调用了Map的Update方法： 让我查看World或者MapManager中Map的Update调用： 现在让我查看MapManager的Update方法： 现在让我查看World类中哪里调用了MapManager的Update： 现在让我搜索World的Update在哪里被调用： 现在让我找一下具体的强制重新定位调用，比如SetPosition等函数调用： 基于我的分析，现在我可以给你完整的`CreatureAI::MoveInLineOfSight`调用链路：

## CreatureAI::MoveInLineOfSight 调用链路分析

### 完整调用链路

```
1. WorldServer 主循环
   ↓
2. World::Update(diff)
   ↓
3. MapManager::Update(diff)
   ↓
4. Map::Update(diff) [对每个地图]
   ↓
5. DelayedUnitRelocation 通知器处理
   ↓
6. PlayerRelocationNotifier/CreatureRelocationNotifier::Visit()
   ↓
7. CreatureUnitRelocationWorker(Creature* c, Unit* u)
   ↓
8. CreatureAI::MoveInLineOfSight_Safe(Unit* who)
   ↓
9. CreatureAI::MoveInLineOfSight(Unit* who)
```

### 详细分析

#### 1. 主要触发路径

**路径A: 定期地图更新**
- **文件**: Main.cpp (第531行)
```cpp
void WorldUpdateLoop()
{
    // ...
    sWorld->Update(diff);  // 每次世界更新循环
}
```

**路径B: 单位强制更新可见性**
- **文件**: Unit.cpp (第12221行)
```cpp
void Unit::UpdateObjectVisibility(bool forced)
{
    // ...
    if (forced)
    {
        // call MoveInLineOfSight for nearby creatures
        Trinity::AIRelocationNotifier notifier(*this);
        Cell::VisitAllObjects(this, notifier, GetVisibilityRange());
    }
}
```

**路径C: 玩家进入PvP状态**
- **文件**: Player.cpp (第20381行)
```cpp
void Player::UpdatePvPState()
{
    // ...
    // call MoveInLineOfSight for nearby contested guards
    Trinity::AIRelocationNotifier notifier(*this);
    Cell::VisitWorldObjects(this, notifier, GetVisibilityRange());
}
```

**路径D: 生物召唤时**
- **文件**: Object.cpp (第1946行)
```cpp
TempSummon* Map::SummonCreature(...)
{
    // ...
    // call MoveInLineOfSight for nearby creatures
    Trinity::AIRelocationNotifier notifier(*summon);
    Cell::VisitAllObjects(summon, notifier, GetVisibilityRange());
}
```

#### 2. 核心处理函数

**GridNotifiers.cpp** (第128行)
```cpp
inline void CreatureUnitRelocationWorker(Creature* c, Unit* u)
{
    if (!u->IsAlive() || !c->IsAlive() || c == u || u->IsInFlight())
        return;

    if (!c->HasUnitState(UNIT_STATE_SIGHTLESS))
    {
        if (c->IsAIEnabled() && c->CanSeeOrDetect(u, false, true))
            c->AI()->MoveInLineOfSight_Safe(u);  // 调用安全版本
        else
            if (u->GetTypeId() == TYPEID_PLAYER && u->HasStealthAura() && c->IsAIEnabled() && c->CanSeeOrDetect(u, false, true, true))
                c->AI()->TriggerAlert(u);
    }
}
```

#### 3. 安全包装函数

**CreatureAI.cpp** (第108行)
```cpp
void CreatureAI::MoveInLineOfSight_Safe(Unit* who)
{
    if (_moveInLOSLocked == true)  // 防止递归调用
        return;
    _moveInLOSLocked = true;
    MoveInLineOfSight(who);  // 调用实际的处理函数
    _moveInLOSLocked = false;
}
```

#### 4. 实际处理函数

**CreatureAI.cpp** (第117行)
```cpp
void CreatureAI::MoveInLineOfSight(Unit* who)
{
    if (me->IsEngaged())  // 已经在战斗中则不处理
        return;

    if (me->HasReactState(REACT_AGGRESSIVE) && me->CanStartAttack(who, false))
        me->EngageWithTarget(who);  // 开始攻击目标
}
```

### 调用频率和时机

1. **定期调用**: 每次地图更新循环中，处理需要重新定位的单位
2. **强制调用**: 当单位位置发生重大变化时立即触发
3. **特殊事件**: PvP状态变化、生物召唤等特殊情况

### 性能优化措施

1. **安全锁**: `_moveInLOSLocked` 防止递归调用造成栈溢出
2. **条件检查**: 在`CreatureUnitRelocationWorker`中进行多重条件检查，避免不必要的AI调用
3. **延迟处理**: 使用`DelayedUnitRelocation`将重定位处理延迟到适当时机

这个调用链路确保了怪物能够及时感知到附近玩家的存在，并根据其AI设置做出相应的反应（攻击、警戒等）。
