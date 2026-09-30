# ros2-academic-skills

个人维护的两套 Agent Skill，面向 ROS2 机器人开发与学术论文写作场景，可直接用于豆包 / WorkBuddy 等支持 SKILL.md 规范的 Agent 客户端。

## 包含的 Skill

| Skill | 路径 | 作用 |
|---|---|---|
| `ros2-dev` | `skills/ros2-dev/` | ROS2 (Humble/Jazzy) 环境搭建、功能包开发、节点模板、launch、调试三板斧、rosbag2、Nav2/MoveIt2/Gazebo 入门 |
| `academic-paper-writing` | `skills/academic-paper-writing/` | 学位论文/期刊论文全流程：IMRAD 结构、摘要/绪论/方法/实验模板、参考文献管理、降重润色、答辩衔接 |

## 安装方法

### 豆包（Doubao）

把 `skills/` 下两个目录整体复制到：

```
C:\Users\<你>\AppData\Local\Doubao\User Data\Default\.doubao\agent_mode\workspace\.user_skills\
```

最终结构：

```
.user_skills/
├── ros2-dev/SKILL.md
└── academic-paper-writing/SKILL.md
```

重启豆包即可自动加载。

### WorkBuddy

WorkBuddy 的 skill 目录在首次登录后生成（通常位于 `%APPDATA%\WorkBuddy\` 下的 skills 目录），同样把两个 skill 文件夹放进去即可。

## 使用

在对话中自然提到 ROS2 命令、写论文、改摘要、降重等需求时，Agent 会自动匹配对应 skill。也可以显式说"用 ros2-dev skill 帮我写个发布者节点"。

## License

MIT
