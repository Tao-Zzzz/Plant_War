# Plant_War
**Plant_War** 是一个使用 C++ 和 EasyX 图形库开发的简易乱斗游戏，灵感来源于《明星大乱斗》。该项目展示了如何在 Windows 平台上利用 EasyX 实现 2D 游戏开发，涵盖了动画处理、场景管理、碰撞检测等核心游戏机制。
## 游戏截图
![image](https://github.com/user-attachments/assets/bf574390-aafd-4901-948a-7d641e9c1799)

![image](https://github.com/user-attachments/assets/97f95c95-aa2b-452d-918f-6e7d41d08c6e)
以下是一些制作笔记, 笔者表达不清请见谅
## 场景系统
首先需要实现<font color="#ff0000">场景基类</font>，定义 `enter`、`input`、`update`、`draw`、`exit` 五种方法，用于描述一个场景的生命周期。然后由 `SceneManager` 统一管理当前处于激活状态的场景，并在主循环中调用当前场景的输入、更新和绘制等逻辑，同时在切换场景时自动调用当前场景的 `exit` 和新场景的 `enter`，从而形成一个完整的**场景状态机**。
## 受击闪烁
实现方式是将角色素材中每一个像素点全部染成白色，这样看起来就是一个全白闪烁的效果。配合定时器使用，**在受击时替换为闪白贴图，定时器结束后则恢复原贴图**，从而实现受击时的视觉提示。

![image](https://github.com/user-attachments/assets/9cbcbd1c-8ec1-4870-ad0e-eb9cee1baa58)

## 定时器
定时器内部需要实现一个 `update(delta)` 方法，在每一帧传入时间差进行更新。在需要使用定时器的地方调用它的 `update` 即可实现**倒计时逻辑**。定时器可以设置触发的总时长、是否只触发一次、是否启用等状态，非常适合用于动画帧切换、闪白、摄像机摇晃等有时间控制需求的功能。
## 摄像机摇晃
摄像机的默认位置为 (0, 0)，只有在触发摇晃时才会生效。每一帧通过一定范围内的随机数生成摄像机偏移量，配合一个定时器控制持续时间。偏移值可以用 `((-50 + rand() % 100) / 50.0f) * shaking_strength` 表达，控制摇晃强度。**所有非背景图片绘制时加上摄像机偏移量**即可实现整体画面抖动的视觉效果。
```c
void on_update(int delta) {
	timer_shake.on_update(delta);
	if (is_shaking) {
		position.x = ((-50 + rand() % 100) / 50.0f) * shaking_strength;
		position.y = ((-50 + rand() % 100) / 50.0f) * shaking_strength;
	}
}
```
## 抛体运动
向日葵的子弹运动轨迹是斜上抛，所以需要用一个 Vector 类来描述物体在 2D 空间中的位置和速度。速度会随时间更新，比如 y 方向要加上重力加速度 `velocity.y += gravity * delta`，而位置则通过当前速度和时间决定 `position += velocity * delta`。整个运动过程完全依赖传入的 `delta` 来保证在不同帧率下的平滑效果。

![image](https://github.com/user-attachments/assets/f1c4e2ee-6eea-49a6-b547-5a54c76417fa)

![image](https://github.com/user-attachments/assets/94703f77-a2fe-4054-ab8b-817168c05a37)

## 子弹效果
玩家类中定义 `attack` 和 `ex_attack` 两个方法，由**具体植物来实现对应的子弹类型**。每个子弹类都继承统一的 `Bullet` 基类，具有 update、draw、hit 判定等逻辑。在 `main` 中维护一个 `bullet_list` 容器来管理所有场上的子弹，**射出时只需把新子弹加入列表即可**。因为玩家有 P1 和 P2 阵营之分，子弹的伤害目标也要做判断。子弹每帧都会由 `GameScene` 调用进行更新，当击中敌人后**触发命中动画**并标记为删除，同时若子弹超出屏幕也直接标记删除等待回收。
![image](https://github.com/user-attachments/assets/940c3123-f29b-4e5c-ac7d-5eededd3e148)

## 碰撞系统
碰撞检测由玩家类自身实现，包括与平台和子弹的碰撞。**子弹的碰撞主要判断是否与玩家矩形区域重叠**，**平台碰撞则是判断玩家底部是否接触平台顶部**，即 `player_bottom_y >= platform_top_y && player_prev_bottom_y < platform_top_y` 这类逻辑。每一帧只要遍历 `bullet_list` 和 `platform_list` 进行检测即可完成所有碰撞逻辑。
## 粒子效果
玩家在地面上移动时可以在脚底生成粒子，制造出奔跑时的烟尘效果。每个粒子内部会使用定时器进行自销毁逻辑，比如存在 100ms 后自动消失。**同时所有粒子由一个 vector 容器进行管理，绘制时一起绘出**。因为玩家每帧都在移动，所以要频繁生成粒子，同时还要注意优化性能防止生成过多。

![image](https://github.com/user-attachments/assets/1cdf9904-77c5-448f-8360-0a32a757d73c)

## 血条
血条绘制逻辑很简单，先画一个黑色的背景矩形，然后再画一个红色的前景矩形。红色矩形的宽度通过玩家当前 HP 与最大 HP 的比例进行缩放显示。**血条的绘制和管理可以放在 `GameScene` 中**统一处理，这样便于集中管理玩家状态和 UI。
![image](https://github.com/user-attachments/assets/9bac6d23-22c6-4156-8f76-5930363c8c19)

## 动态延时
控制游戏帧率的关键是让**每帧运行时间**控制在一个目标范围内，比如 1000/FPS 毫秒。通过记录本帧开始和结束时间，如果结束时间 - 开始时间小于目标帧时间，就使用 `Sleep` 强制休眠一段时间来释放 CPU 资源。这样能保持画面稳定，避免因为处理速度过快导致画面撕裂或资源浪费。
```c
DWORD frame_end_time = GetTickCount(); 
DWORD frame_delta_time = frame_end_time - frame_start_time; 
if (frame_delta_time < 1000 / FPS) { 
    Sleep(1000 / FPS - frame_delta_time); 
}
```
## 动画系统
动画系统分为两部分，**一是 `Atlas` 用来管理动画帧序列，即所有的图片切图信息**。**二是 `Animation` 类管理动画的播放逻辑，包括当前帧下标、帧切换间隔等**。使用一个内部定时器累加 delta，当累积时间超过一帧间隔就切换到下一帧，如果到最后一帧可以选择循环或者停止播放。帧间隔可以用 `#define` 统一定义，便于控制不同动画的播放速度。

## 项目结构
项目采用模块化设计，主要包括以下文件和目录：
- `main.cpp`：程序入口，初始化游戏并启动主循环。
- `scene_manager.h`：管理不同的游戏场景，如主菜单、游戏进行中等。
- `player.h`、`peashooter_player.h`、`sunflower_player.h`：定义植物角色及其行为。
- `bullet.h`、`pea_bullet.h`、`sun_bullet.h`：定义子弹类型及其运动逻辑。
- `enemy.h`：定义敌人角色及其行为。
- `animation.h`、`particle.h`：处理动画和粒子效果。
- `util.h`、`vector2.h`：提供辅助函数和二维向量类。
## 核心模块文件说明
### 1. `main.cpp` — 游戏入口与主循环
该文件是程序的入口点，主要负责初始化游戏环境、加载资源、设置窗口属性，并启动游戏主循环。
**主要职责：**
- 调用 `initgraph()` 初始化 EasyX 图形窗口。
- 创建并管理 `SceneManager` 实例，控制游戏场景的切换。
- 处理全局事件，如退出游戏、暂停等。
- 维护游戏主循环，持续更新和渲染当前场景。
    
### 2. `scene_manager.h` — 场景管理器
该模块负责管理游戏中的不同场景，如主菜单、游戏进行中、游戏结束等。
**主要职责：**
- 定义场景的生命周期，包括初始化、更新、渲染和销毁。
- 根据游戏状态切换不同的场景。
- 提供接口供其他模块访问当前场景的信息。
### 3. `player.h`、`peashooter_player.h`、`sunflower_player.h` — 植物角色模块
这些文件定义了植物角色的基类和具体实现，管理植物的属性和行为。
**主要职责：**
- 定义植物的基本属性，如生命值、攻击力、攻击范围等。
- 实现植物的攻击逻辑，如发射子弹、生成阳光等。
- 处理植物的动画效果和状态变化。
### 4. `bullet.h`、`pea_bullet.h`、`sun_bullet.h` — 子弹模块
这些文件定义了子弹的基类和具体实现，管理子弹的运动和碰撞检测。
**主要职责：**
- 定义子弹的基本属性，如速度、伤害、方向等。
- 实现子弹的移动逻辑和与敌人的碰撞检测。
- 处理子弹的生命周期，如命中敌人后销毁。
### 5. `enemy.h` — 敌人模块
该文件定义了敌人的属性和行为，管理敌人的生成、移动和攻击逻辑。
**主要职责：**
- 定义敌人的基本属性，如生命值、速度、攻击力等。
- 实现敌人的移动路径和攻击逻辑。
- 处理敌人的动画效果和状态变化。
### 6. `animation.h`、`particle.h` — 动画与粒子效果模块
这些文件处理游戏中的动画和粒子效果，增强视觉表现力。
**主要职责：**
- 管理动画帧的播放和切换。
- 实现粒子效果，如爆炸、光效等。
- 与其他模块协作，触发特定的动画和粒子效果。
### 7. `util.h`、`vector2.h` — 工具模块
这些文件提供了辅助函数和数据结构，支持游戏的各项功能。
**主要职责：**
- 定义二维向量类 `Vector2`，用于表示位置、方向等。
- 提供数学计算、随机数生成等实用函数。
- 简化其他模块的开发，提高代码的可读性和复用性。


