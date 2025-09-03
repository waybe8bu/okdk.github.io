+++
title = 'TrinityCore 如何处理怪物 Engage 玩家'
date = 2024-01-14T07:07:07+01:00
+++
分析/Users/10000501/Documents/chaos/c/TrinityCore-3.3.5下的代码，指出触发怪物将玩家设置为target的代码

GitHub Copilot: 我来帮你分析TrinityCore-3.3.5代码库中触发怪物将玩家设置为target的相关代码。首先让我了解一下代码结构。 现在让我搜索与怪物目标选择相关的代码，重点关注AI和Entities目录： 让我搜索更多关于AI和目标选择的代码： 现在让我查看更具体的AI文件来了解目标选择机制： 让我搜索更多关于目标选择的函数： 现在让我查看关键的`SelectVictim`和`EngageWithTarget`函数： 现在让我查看`EngageWithTarget`函数： 现在让我查看`AddThreat`函数，这是关键的函数： 让我继续阅读这个函数的关键部分： 现在让我查看与怪物最初激活攻击相关的代码： 现在让我查看`CanStartAttack`函数，这是决定怪物是否能开始攻击玩家的关键函数： 让我查看怪物AI更新的代码，了解怪物如何处理攻击目标： 让我查看怪物Creature类的Update方法： 现在让我查看一些常见的AI脚本，了解怪物如何选择和攻击目标： 基于我对TrinityCore 3.3.5代码的分析，我可以为你详细解释触发怪物将玩家设置为target的主要代码机制：

## TrinityCore 3.3.5中怪物设置玩家为目标的代码分析

### 1. 主要触发机制

怪物将玩家设置为攻击目标主要通过以下几个关键代码路径：

#### A. 视线检测系统 (`MoveInLineOfSight`)
**文件**: CreatureAI.cpp (第117行)
```cpp
void CreatureAI::MoveInLineOfSight(Unit* who)
{
    if (me->IsEngaged())
        return;

    if (me->HasReactState(REACT_AGGRESSIVE) && me->CanStartAttack(who, false))
        me->EngageWithTarget(who);
}
```

这是最基本的攻击触发机制，当玩家进入怪物的视线范围内时，如果怪物是攻击性反应状态，就会尝试攻击。

#### B. 怪物能否开始攻击的判断 (`CanStartAttack`)
**文件**: Creature.cpp (第1865行)
```cpp
bool Creature::CanStartAttack(Unit const* who, bool force) const
{
    if (IsCivilian())
        return false;

    // 免疫检查
    if ((IsImmuneToNPC() && !who->HasUnitFlag(UNIT_FLAG_PLAYER_CONTROLLED))
        || (IsImmuneToPC() && who->HasUnitFlag(UNIT_FLAG_PLAYER_CONTROLLED)))
        return false;

    // 不攻击非战斗宠物
    if (who->GetTypeId() == TYPEID_UNIT && who->GetCreatureType() == CREATURE_TYPE_NON_COMBAT_PET)
        return false;

    // Z轴距离检查
    if (!CanFly() && (GetDistanceZ(who) > CREATURE_Z_ATTACK_RANGE + m_CombatDistance))
        return false;

    if (!force)
    {
        // 目标可接受性检查
        if (!_IsTargetAcceptable(who))
            return false;

        // 中立或距离检查
        if (IsNeutralToAll() || !IsWithinDistInMap(who, GetAttackDistance(who) + m_CombatDistance))
            return false;
    }

    // 基本攻击能力检查
    if (!CanCreatureAttack(who, force))
        return false;

    // 灰色怪物不主动攻击配置
    if (CheckNoGrayAggroConfig(who->GetLevelForTarget(this), GetLevelForTarget(who)))
        return false;

    // 视线检查
    return IsWithinLOSInMap(who);
}
```

### 2. 开始攻击的核心函数 (`EngageWithTarget`)
**文件**: Unit.cpp (第8289行)
```cpp
void Unit::EngageWithTarget(Unit* enemy)
{
    if (!enemy)
        return;

    if (CanHaveThreatList())
        m_threatManager.AddThreat(enemy, 0.0f, nullptr, true, true);
    else
        SetInCombatWith(enemy);
}
```

这个函数是怪物设置目标的关键入口点，它会将目标添加到威胁列表中。

### 3. 威胁系统 (`ThreatManager::AddThreat`)
**文件**: ThreatManager.cpp (第356行开始)

这是最核心的目标设置机制：
```cpp
void ThreatManager::AddThreat(Unit* target, float amount, SpellInfo const* spell, bool ignoreModifiers, bool ignoreRedirects)
{
    // 第一步: 检查法术是否有NO_THREAT属性
    if (spell)
    {
        if (spell->HasAttribute(SPELL_ATTR1_NO_THREAT))
            return;
        if (!_owner->IsEngaged() && spell->HasAttribute(SPELL_ATTR3_NO_INITIAL_AGGRO))
            return;
    }

    // 确保进入战斗状态 (威胁意味着战斗!)
    if (!_owner->GetCombatManager().SetInCombatWith(target))
        return;

    // 检查是否已有威胁条目
    auto it = _myThreatListEntries.find(target->GetGUID());
    if (it != _myThreatListEntries.end())
    {
        // 已存在，增加威胁值
        ThreatReference* const ref = it->second;
        if (ref->IsOnline())
            ref->AddThreat(amount);
        return;
    }

    // 创建新的威胁引用并添加到威胁管理器
    ThreatReference* ref = new ThreatReferenceImpl(this, target);
    PutThreatListRef(target->GetGUID(), ref);
    target->GetThreatManager().PutThreatenedByMeRef(_owner->GetGUID(), ref);

    ref->UpdateOffline();
    if (ref->IsOnline())
        ref->AddThreat(amount);

    // 如果没有当前受害者，更新受害者
    if (!_currentVictimRef)
        UpdateVictim();
    else
        ProcessAIUpdates();
}
```

### 4. 目标选择机制 (`SelectVictim`)
**文件**: Creature.cpp (第1124行)
```cpp
Unit* Creature::SelectVictim()
{
    Unit* target = nullptr;

    if (CanHaveThreatList())
        target = GetThreatManager().GetCurrentVictim();
    else if (!HasReactState(REACT_PASSIVE))
    {
        // 宠物逻辑
        target = getAttackerForHelper();
        if (!target && IsSummon())
        {
            if (Unit* owner = ToTempSummon()->GetOwner())
            {
                if (owner->IsInCombat())
                    target = owner->getAttackerForHelper();
                // ... 更多宠物逻辑
            }
        }
    }

    if (target && _IsTargetAcceptable(target) && CanCreatureAttack(target))
    {
        if (!HasSpellFocus())
            SetInFront(target);
        return target;
    }

    return nullptr;
}
```

### 5. AI更新循环中的目标更新
**文件**: CreatureAI.cpp (第240行)
```cpp
bool CreatureAI::UpdateVictim()
{
    if (!IsEngaged())
        return false;

    if (!me->IsAlive())
    {
        EngagementOver();
        return false;
    }

    if (!me->HasReactState(REACT_PASSIVE))
    {
        if (Unit* victim = me->SelectVictim())
            if (victim != me->GetVictim())
                AttackStart(victim);  // 这里会触发目标切换

        return me->GetVictim() != nullptr;
    }
    
    // ... 被动状态处理
    return true;
}
```

### 6. 特殊触发情况

#### A. 副本中的定期战斗脉冲
**文件**: Creature.cpp (第785行)
```cpp
// 如果启用了定期战斗脉冲且怪物在战斗中且在副本中
if (m_combatPulseDelay > 0 && IsEngaged() && GetMap()->IsDungeon())
{
    // ... 定时逻辑
    Map::PlayerList const& players = GetMap()->GetPlayers();
    for (auto& player : players)
    {
        if (player->IsAlive() && IsHostileTo(player))
            EngageWithTarget(player);  // 强制将玩家设为目标
    }
}
```

#### B. 所有者战斗交互
**文件**: CreatureAI.cpp (第126行)
```cpp
void CreatureAI::OnOwnerCombatInteraction(Unit* target)
{
    if (!target || !me->IsAlive())
        return;

    if (!me->HasReactState(REACT_PASSIVE) && me->CanStartAttack(target, true))
        me->EngageWithTarget(target);
}
```

### 总结

怪物将玩家设置为target的主要触发点：

1. **视线检测** - 玩家进入怪物攻击范围触发`MoveInLineOfSight`
2. **威胁系统** - 通过`AddThreat`函数将玩家添加到威胁列表
3. **目标选择** - `SelectVictim`根据威胁列表选择最高威胁目标
4. **AI更新** - 在每次AI更新中通过`UpdateVictim`刷新当前目标
5. **特殊机制** - 副本脉冲、宠物辅助、法术伤害等特殊情况

这整个系统确保了怪物能够智能地选择和切换攻击目标，同时保持了战斗的流畅性和逻辑性。

找到具有 1 个许可证类型的类似代码

10000501_shein: 指出CreatureAI::MoveInLineOfSight的调用链路

GitHub Copilot: 我来帮你分析`CreatureAI::MoveInLineOfSight`的调用链路。让我搜索相关的调用代码。 现在让我查看Grid通知机制，这是`MoveInLineOfSight`被调用的关键地方： 现在让我查看`CreatureUnitRelocationWorker`函数： 现在让我查看这些Grid通知器是在哪里被调用的： 让我查看这些通知器在哪里被触发： 现在让我搜索这些通知器是在什么时候被调用的： 现在让我查看具体的调用点，特别是Unit和Player中的调用： 现在让我查看Object类中的调用： 现在让我查看Map的Relocation方法： 让我查看Map的Update方法，那里应该有DelayedUnitRelocation的调用： 现在让我搜索哪里调用了Map的Update方法： 让我查看World或者MapManager中Map的Update调用： 现在让我查看MapManager的Update方法： 现在让我查看World类中哪里调用了MapManager的Update： 现在让我搜索World的Update在哪里被调用： 现在让我找一下具体的强制重新定位调用，比如SetPosition等函数调用： 基于我的分析，现在我可以给你完整的`CreatureAI::MoveInLineOfSight`调用链路：

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

找到具有 1 个许可证类型的类似代码

10000501_shein: 怪物选择到合适的victim，完成了路径规划之后，是将路径信息发给客户端还是定时向附近玩家广播位置？

GitHub Copilot: 我来帮你分析怪物在选择到victim并完成路径规划后，是如何向客户端同步位置信息的。让我搜索相关的代码。 现在让我查看Movement包的相关代码： 现在让我查看MoveSplineInit.cpp文件，这应该是发送移动包的关键代码： 现在让我看看`SendMessageToSet`函数： 现在让我查看MessageDistDeliverer的实现： 现在让我看看定时位置更新的机制： 让我查看Unit的UpdatePosition方法： 现在让我看看Map的CreatureRelocation方法： 现在让我看看Movement相关的MotionMaster如何工作： 现在让我查看Spline移动系统，这是怪物移动的核心： 让我看一下Unit类中位置同步相关的代码： 基于我的分析，现在我可以全面回答你的问题：

## 怪物移动同步机制分析

### 答案：发送路径信息给客户端

怪物在选择到合适的victim并完成路径规划后，**主要是将完整的路径信息发送给客户端**，而不是定时广播位置。

### 详细机制

#### 1. 路径规划完成后的处理

**文件**: MoveSplineInit.cpp (第130行)
```cpp
uint32 MoveSplineInit::Launch()
{
    // 验证参数并初始化移动样条
    move_spline.Initialize(args);

    // 创建SMSG_MONSTER_MOVE包
    WorldPacket data(SMSG_MONSTER_MOVE, 64);
    data << unit->GetPackGUID();
    
    // 写入完整的移动路径信息
    PacketBuilder::WriteMonsterMove(move_spline, data);
    
    // 向附近玩家广播路径包
    unit->SendMessageToSet(&data, true);
    
    return move_spline.Duration();
}
```

#### 2. 路径包的内容

移动包包含以下信息：
- **起始位置**: 怪物当前位置
- **路径点序列**: 完整的移动路径waypoints
- **移动类型**: 走路/跑步/飞行等
- **移动速度**: 移动速度参数
- **面向信息**: 面向目标、角度或位置
- **移动标志**: 特殊移动状态

#### 3. 客户端预测机制

```cpp
void Unit::UpdateSplinePosition()
{
    // 服务器端根据样条曲线计算当前位置
    Movement::Location loc = movespline->ComputePosition();
    
    // 更新服务器端位置
    UpdatePosition(loc.x, loc.y, loc.z, loc.orientation);
}
```

**工作原理**:
1. **一次性发送**: 服务器将完整路径一次性发送给客户端
2. **客户端插值**: 客户端根据路径和时间进行平滑插值
3. **服务器计算**: 服务器同时进行相同的位置计算
4. **同步检查**: 定期验证客户端和服务器位置是否一致

#### 4. 位置同步机制

**定时同步的情况**:
```cpp
void WorldObject::SendMessageToSet(WorldPacket const* data, bool self) const
{
    if (IsInWorld())
        SendMessageToSetInRange(data, GetVisibilityRange(), self);
}
```

**同步时机**:
- **立即同步**: 开始新的移动时发送完整路径
- **定期校验**: 移动过程中偶尔发送位置校正
- **状态变化**: 停止、转向、速度变化时更新
- **异常处理**: 检测到不同步时强制校正

### 优势分析

#### 路径方式的优点:
1. **网络效率**: 一次发送完整路径，减少网络包数量
2. **流畅性**: 客户端可以进行平滑插值，移动更流畅
3. **预测性**: 客户端可以预测移动轨迹
4. **延迟容忍**: 网络延迟不会影响移动流畅度

#### 定时位置方式的缺点:
1. **网络压力**: 需要频繁发送位置包
2. **不流畅**: 移动可能出现跳跃感
3. **延迟敏感**: 网络延迟直接影响移动表现

### 特殊情况

```cpp
void MoveSplineInit::Stop()
{
    // 停止移动时发送停止包
    WorldPacket data(SMSG_MONSTER_MOVE, 64);
    PacketBuilder::WriteStopMovement(loc, args.splineId, data);
    unit->SendMessageToSet(&data, true);
}
```

**需要立即更新的情况**:
- 移动路径被打断
- 目标发生变化
- 遇到障碍物
- 进入战斗状态
- 施放法术需要停止

### 总结

TrinityCore使用的是**基于路径的移动同步机制**，这是现代MMO游戏的标准做法。服务器在怪物开始移动时发送完整的移动路径信息，客户端根据这些信息进行本地插值计算，实现流畅的移动效果。只有在特殊情况下才会发送额外的位置校正包。

10000501_shein: 怪物在移动时，如果移动到原本没有怪物视野的玩家的视野内，如果让这位玩家接受到路径信息？

GitHub Copilot: 这是一个很好的问题！当怪物移动到一个原本没有看到它的玩家视野内时，需要特殊的处理机制。让我分析一下TrinityCore是如何处理这种情况的。 现在让我查看Player的UpdateVisibilityOf函数的具体实现： 现在让我查看`SendUpdateToPlayer`函数，这应该是发送创建包给玩家的关键函数： 现在让我查看怪物的移动更新机制，特别是与视野变化相关的部分： 让我查看Unit的BuildCreateUpdateBlockForPlayer函数： 让我直接查看Unit类的继承关系，它应该继承自Object： 现在让我查看BuildMovementUpdate函数，这应该包含移动信息： 现在让我查看关键的Spline信息写入部分： 现在让我查看当怪物进入新的Grid Cell时是如何处理的： 现在我来查看关键的`UpdateObjectVisibility`调用： 现在让我查看WorldObject的UpdateObjectVisibility： 现在让我查看VisibleChangesNotifier： 现在我有了完整的理解！让我详细回答你的问题：

## 怪物移动到新玩家视野内的路径信息处理机制

### 核心机制：创建更新包含当前移动状态

当怪物移动到一个原本没有看到它的玩家视野内时，TrinityCore使用一个非常聪明的机制来解决这个问题：

### 1. 触发时机

**文件**: Map.cpp (第1290行)
```cpp
void Map::MoveAllCreaturesInMoveList()
{
    // 处理跨Grid/Cell移动的怪物
    for (std::vector<Creature*>::iterator itr = _creaturesToMove.begin(); itr != _creaturesToMove.end(); ++itr)
    {
        // 更新怪物位置后
        c->Relocate(c->_newPosition);
        c->UpdatePositionData();
        
        // 关键：更新可见性
        c->UpdateObjectVisibility(false);
    }
}
```

### 2. 可见性更新机制

**文件**: Object.cpp (第3479行)
```cpp
void WorldObject::UpdateObjectVisibility(bool /*forced*/)
{
    // 通知附近玩家可见性变化
    Trinity::VisibleChangesNotifier notifier(*this);
    Cell::VisitWorldObjects(this, notifier, GetVisibilityRange());
}
```

### 3. 玩家视野检测

**文件**: GridNotifiers.cpp (第88行)
```cpp
void VisibleChangesNotifier::Visit(PlayerMapType &m)
{
    for (PlayerMapType::iterator iter = m.begin(); iter != m.end(); ++iter)
    {
        // 让每个附近玩家检查对这个对象的可见性
        iter->GetSource()->UpdateVisibilityOf(&i_object);
    }
}
```

### 4. 关键的视野判断和包发送

**文件**: Player.cpp (第22328行)
```cpp
void Player::UpdateVisibilityOf(WorldObject* target)
{
    if (HaveAtClient(target))
    {
        // 玩家已经看到过这个对象，检查是否应该隐藏
        if (!CanSeeOrDetect(target, false, true))
        {
            target->DestroyForPlayer(this);
            m_clientGUIDs.erase(target->GetGUID());
        }
    }
    else
    {
        // 关键：玩家还没看到过这个对象
        if (CanSeeOrDetect(target, false, true))
        {
            // 发送创建更新包，包含当前移动状态！
            target->SendUpdateToPlayer(this);
            m_clientGUIDs.insert(target->GetGUID());
        }
    }
}
```

### 5. 创建包包含移动信息

**文件**: Object.cpp (第201行)
```cpp
void Object::SendUpdateToPlayer(Player* player)
{
    UpdateData upd;
    WorldPacket packet;

    if (player->HaveAtClient(this))
        BuildValuesUpdateBlockForPlayer(&upd, player);  // 已知对象，只发送数值更新
    else
        BuildCreateUpdateBlockForPlayer(&upd, player);  // 新对象，发送完整创建包
    
    upd.BuildPacket(&packet);
    player->SendDirectMessage(&packet);
}
```

### 6. 移动信息的包含机制

**文件**: Object.cpp (第288行)
```cpp
void Object::BuildMovementUpdate(ByteBuffer* data, uint16 flags) const
{
    // 如果单位正在移动（有样条启用）
    if (unit->m_movementInfo.GetMovementFlags() & MOVEMENTFLAG_SPLINE_ENABLED)
    {
        // 关键：包含当前的移动样条信息
        Movement::PacketBuilder::WriteCreate(*unit->movespline, *data);
    }
}
```

### 7. 样条信息写入

**文件**: MovementPacketBuilder.cpp (第147行)
```cpp
void PacketBuilder::WriteCreate(MoveSpline const& move_spline, ByteBuffer& data)
{
    // 写入完整的移动路径信息
    data << move_spline.timePassed();     // 已经过去的时间
    data << move_spline.Duration();       // 总持续时间
    data << move_spline.GetId();          // 移动ID
    
    uint32 nodes = move_spline.getPath().size();
    data << nodes;
    // 写入剩余的路径点
    data.append<G3D::Vector3>(&move_spline.getPath()[0], nodes);
    
    // 写入最终目标点
    data << (move_spline.isCyclic() ? G3D::Vector3::zero() : move_spline.FinalDestination());
}
```

### 工作原理总结

1. **怪物开始移动**: 使用`SMSG_MONSTER_MOVE`包发送路径给当前可见的玩家

2. **移动过程中**: 当怪物跨越Grid/Cell边界时，系统会调用`UpdateObjectVisibility`

3. **检测新玩家**: `VisibleChangesNotifier`遍历新区域的所有玩家

4. **发送创建包**: 对于能看到怪物但还没收到过它信息的玩家，发送`UPDATETYPE_CREATE_OBJECT`包

5. **包含移动状态**: 创建包中包含完整的当前移动样条信息，包括：
   - 当前路径的剩余路径点
   - 已经移动的时间比例
   - 移动速度和参数
   - 最终目标位置

6. **客户端处理**: 新玩家的客户端收到创建包后，会：
   - 立即创建怪物
   - 根据已过去的时间计算当前应该在的位置
   - 开始跟随剩余的路径继续移动

### 优势

这种机制确保了：
- **无缝体验**: 玩家看到怪物时，它已经在正确的位置和移动状态
- **时间同步**: 通过`timePassed()`和`Duration()`，客户端可以准确计算当前位置
- **路径连续**: 新玩家看到的移动是连续的，不会有跳跃感
- **性能优化**: 只在需要时发送创建包，避免不必要的网络开销

这就是TrinityCore处理怪物进入新玩家视野时路径信息同步的完整机制！
