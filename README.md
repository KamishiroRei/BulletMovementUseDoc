# BulletMovementComponent 用户手册

基于UE5.5开发的子弹运动组件，支持弹道跟踪、重力模拟、折射反射、归位系统、加速/减速曲线等特性，适用于弹幕射击类游戏开发。

---

## 组件参数 (Editable in Editor)
| **参数分类** | **显示名称** | **C++类型** | **默认值** | **说明** |
|--------------|--------------|-------------|------------|----------|
| **Bullet Performance** |  |  |  |  |
|  | `Initial Speed` | `float` | 0 | 子弹初始速度（单位/秒） |
|  | `Max Speed` | `float` | 0 | 最大速度限制（0=无限制） |
|  | `Min Speed` | `float` | 0 | 触发停止的最小速度阈值 |
|  | `Max Move Distance` | `float` | 0 | 最大移动距离限制（0=无限制） |
|  | `Gravity Scale` | `float` | 0 | 重力缩放系数（0=禁用重力） |
|  | `Initial Follow AimPoint` | `uint8` | false | 初始速度是否朝向瞄准点（AimPoint） |
|  | `Aiming Point` | `FVector` | (0,0,0) | 初始瞄准点坐标（仅在`bFollowAim`启用时生效） |
| **Bullet Trail Effect** |  |  |  |  |
|  | `Trail Effect` | `UNiagaraSystem*` | nullptr | 拖尾粒子特效资产 |
|  | `Effect Scale` | `float` | 1 | 拖尾特效缩放系数 |
| **Bullet Refract** |  |  |  |  |
|  | `Refract Type` | `E_BulletRefractType` | None | 子弹折射类型：<br>- **No Refract**：碰撞后停止<br>- **Standard**：镜面反射<br>- **Targeted**：导向目标折射 |
|  | `Refract Count` | `int32` | 0 | 最大折射次数（-1=无限） |
|  | `Max Detection Range for Directional Refract` | `float` | 0 |
