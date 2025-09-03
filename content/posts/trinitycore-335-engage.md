+++
title = 'TrinityCore 如何处理怪物 Engage 玩家'
date = 2024-01-01T00:00:00+00:00
+++
分析TrinityCore-3.3.5下的代码，指出触发怪物将玩家设置为target的代码

我来帮你分析TrinityCore-3.3.5代码库中触发怪物将玩家设置为target的相关代码。首先让我了解一下代码结构。 现在让我搜索与怪物目标选择相关的代码，重点关注AI和Entities目录： 让我搜索更多关于AI和目标选择的代码： 现在让我查看更具体的AI文件来了解目标选择机制： 让我搜索更多关于目标选择的函数： 现在让我查看关键的`SelectVictim`和`EngageWithTarget`函数： 现在让我查看`EngageWithTarget`函数： 现在让我查看`AddThreat`函数，这是关键的函数： 让我继续阅读这个函数的关键部分： 现在让我查看与怪物最初激活攻击相关的代码： 现在让我查看`CanStartAttack`函数，这是决定怪物是否能开始攻击玩家的关键函数： 让我查看怪物AI更新的代码，了解怪物如何处理攻击目标： 让我查看怪物Creature类的Update方法： 现在让我查看一些常见的AI脚本，了解怪物如何选择和攻击目标： 基于我对TrinityCore 3.3.5代码的分析，我可以为你详细解释触发怪物将玩家设置为target的主要代码机制：

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
