+++
title = 'TrinityCore 如何处理怪物 Engage 玩家：路径规划同步-新玩家加入'
date = 2024-01-04T00:00:00+00:00
+++
怪物在移动时，如果移动到原本没有怪物视野的玩家的视野内，如果让这位玩家接受到路径信息？

这是一个很好的问题！当怪物移动到一个原本没有看到它的玩家视野内时，需要特殊的处理机制。让我分析一下TrinityCore是如何处理这种情况的。 现在让我查看Player的UpdateVisibilityOf函数的具体实现： 现在让我查看`SendUpdateToPlayer`函数，这应该是发送创建包给玩家的关键函数： 现在让我查看怪物的移动更新机制，特别是与视野变化相关的部分： 让我查看Unit的BuildCreateUpdateBlockForPlayer函数： 让我直接查看Unit类的继承关系，它应该继承自Object： 现在让我查看BuildMovementUpdate函数，这应该包含移动信息： 现在让我查看关键的Spline信息写入部分： 现在让我查看当怪物进入新的Grid Cell时是如何处理的： 现在我来查看关键的`UpdateObjectVisibility`调用： 现在让我查看WorldObject的UpdateObjectVisibility： 现在让我查看VisibleChangesNotifier： 现在我有了完整的理解！让我详细回答你的问题：

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
