# 开发者文档

## 项目架构

### 技术栈
- **框架**: HarmonyOS ArkTS
- **UI框架**: ArkUI
- **状态管理**: @State, @Prop 装饰器
- **路由**: router 模块
- **构建工具**: Hvigor

### 目录结构
```
entry/src/main/
├── ets/
│   ├── pages/                    # 页面组件
│   │   ├── Index.ets            # 首页
│   │   ├── PresetMode.ets       # 预设模式页面
│   │   ├── CustomMode.ets       # 自定义模式设置页面
│   │   ├── CustomSkillInput.ets # 自定义技能输入页面
│   │   └── ResultPage.ets       # 结果展示页面
│   ├── utils/                   # 工具函数
│   │   └── SkillData.ets        # 技能数据管理
│   └── entryability/            # 应用入口
│       └── EntryAbility.ets     # 应用生命周期管理
├── resources/                   # 资源文件
│   ├── base/
│   │   ├── element/            # 样式资源
│   │   ├── media/              # 媒体资源
│   │   └── profile/            # 配置文件
│   └── rawfile/                # 原始资源文件
└── module.json5                # 模块配置文件
```

## 核心组件

### 1. 首页 (Index.ets)
```typescript
// 主要功能：模式选择
@Component
struct Index {
  build() {
    Column() {
      // 预设模式按钮
      Button('预设模式')
        .onClick(() => router.pushUrl(...))
      
      // 自定义模式按钮  
      Button('自定义模式')
        .onClick(() => router.pushUrl(...))
    }
  }
}
```

### 2. 预设模式页面 (PresetMode.ets)
```typescript
// 主要功能：玩家数量设置，使用预设技能库
@Component
struct PresetMode {
  @State playerCount: number = 5
  @State skillsPerPlayer: number = 2
  
  build() {
    Column() {
      // 玩家数量滑块
      Slider({...})
        .onChange((value) => this.updatePlayerCount(value))
      
      // 开始分配按钮
      Button('开始分配技能')
        .onClick(() => this.startDistribution())
    }
  }
}
```

### 3. 自定义模式设置页面 (CustomMode.ets)
```typescript
// 主要功能：设置技能参数
@Component
struct CustomMode {
  @State totalSkills: number = 10
  @State skillsPerPlayer: number = 5
  
  build() {
    Column() {
      // 总技能数输入
      TextInput({...})
        .onChange((value) => this.handleTotalSkillsChange(value))
      
      // 每人分配数输入
      TextInput({...})
        .onChange((value) => this.handleSkillsPerPlayerChange(value))
      
      // 下一步按钮
      Button('下一步')
        .onClick(() => this.goToSkillInput())
    }
  }
}
```

### 4. 自定义技能输入页面 (CustomSkillInput.ets)
```typescript
// 主要功能：输入自定义技能信息
@Component
struct CustomSkillInput {
  @State skills: CustomSkill[] = []
  @State skillNameInputs: string[] = []    // 本地状态 - 技能名称
  @State skillDescInputs: string[] = []    // 本地状态 - 技能描述
  
  // 技能类型选项
  private skillTypes: SkillTypeOption[] = [
    { label: '主动技能', value: 'active' },
    { label: '被动技能', value: 'passive' },
    { label: '大招技能', value: 'ultimate' },
    { label: '特殊技能', value: 'special' }
  ]
  
  // 构建技能输入卡片
  @Builder
  SkillInputCard(skill: CustomSkill, index: number) {
    Column() {
      // 技能名称输入
      TextInput({
        placeholder: '请输入技能名称',
        text: this.skillNameInputs[index] || skill.name
      })
      .onChange((value) => this.updateSkillNameLocal(index, value))
      .onEditChange((isEditing) => {
        if (!isEditing) this.updateSkillNameGlobal(index)
      })
      
      // 技能描述输入
      TextArea({
        placeholder: '请输入技能描述',
        text: this.skillDescInputs[index] || skill.description
      })
      .maxLines(3)
      .onChange((value) => this.updateSkillDescriptionLocal(index, value))
      .onEditChange((isEditing) => {
        if (!isEditing) this.updateSkillDescriptionGlobal(index)
      })
      
      // 技能类型选择器（可滑动）
      Scroll() {
        Row() {
          ForEach(this.skillTypes, (item) => {
            Button(item.label)
              .onClick(() => this.updateSkillType(index, item.value))
          })
        }
      }
      .scrollable(ScrollDirection.Horizontal)
    }
  }
}
```

### 5. 结果页面 (ResultPage.ets)
```typescript
// 主要功能：展示分配结果
@Component
struct ResultPage {
  @State players: Player[] = []  // 玩家技能分配结果
  
  build() {
    Column() {
      // 玩家列表
      ForEach(this.players, (player, index) => {
        PlayerCard(player, index)
      })
      
      // 操作按钮
      Row() {
        Button('重新分配')
          .onClick(() => this.redistribute())
        
        Button('返回首页')
          .onClick(() => router.back())
      }
    }
  }
}
```

## 状态管理

### 数据模型
```typescript
// 技能接口
interface CustomSkill {
  id: number;
  name: string;
  description: string;
  type: 'active' | 'passive' | 'ultimate' | 'special';
}

// 玩家接口
interface Player {
  id: number;
  name: string;
  skills: CustomSkill[];
}

// 技能类型选项
interface SkillTypeOption {
  label: string;
  value: 'active' | 'passive' | 'ultimate' | 'special';
}
```

### 状态更新策略
```typescript
// 双层状态管理解决焦点问题
@Component
struct CustomSkillInput {
  @State skills: CustomSkill[] = [];          // 全局状态
  @State skillNameInputs: string[] = [];      // 本地状态 - 名称
  @State skillDescInputs: string[] = [];      // 本地状态 - 描述
  
  // 实时更新本地状态（不触发全局重渲染）
  updateSkillNameLocal(index: number, name: string): void {
    const newNames = this.skillNameInputs.slice();
    newNames[index] = name;
    this.skillNameInputs = newNames;
  }
  
  // 失去焦点时更新全局状态
  updateSkillNameGlobal(index: number): void {
    const name = this.skillNameInputs[index] || '';
    const newSkills = this.skills.slice();
    const oldSkill = newSkills[index];
    newSkills[index] = {
      ...oldSkill,
      name: name
    };
    this.skills = newSkills;
  }
}
```

## 关键算法

### 随机分配算法
```typescript
// 技能随机分配
function distributeSkills(
  skills: CustomSkill[], 
  playerCount: number, 
  skillsPerPlayer: number
): Player[] {
  // 1. 随机打乱技能顺序
  const shuffledSkills = [...skills].sort(() => Math.random() - 0.5);
  
  // 2. 创建玩家数组
  const players: Player[] = [];
  for (let i = 0; i < playerCount; i++) {
    players.push({
      id: i + 1,
      name: `玩家 ${i + 1}`,
      skills: []
    });
  }
  
  // 3. 平均分配技能
  let skillIndex = 0;
  for (let i = 0; i < skillsPerPlayer; i++) {
    for (let j = 0; j < playerCount; j++) {
      if (skillIndex < shuffledSkills.length) {
        players[j].skills.push(shuffledSkills[skillIndex]);
        skillIndex++;
      }
    }
  }
  
  return players;
}
```

### 验证算法
```typescript
// 验证技能输入完整性
function validateSkills(skills: CustomSkill[]): { valid: boolean, message: string } {
  for (let i = 0; i < skills.length; i++) {
    const skill = skills[i];
    
    if (!skill.name.trim()) {
      return {
        valid: false,
        message: `请填写技能 ${i + 1} 的名称`
      };
    }
    
    if (!skill.description.trim()) {
      return {
        valid: false,
        message: `请填写技能 ${i + 1} 的描述`
      };
    }
  }
  
  return { valid: true, message: '' };
}
```

## 路由配置

### main_pages.json
```json
{
  "src": [
    "pages/Index",
    "pages/PresetMode", 
    "pages/CustomMode",
    "pages/CustomSkillInput",
    "pages/ResultPage"
  ]
}
```

### 路由跳转示例
```typescript
// 跳转到自定义模式设置
router.pushUrl({
  url: 'pages/CustomMode'
});

// 跳转到技能输入页面（带参数）
router.pushUrl({
  url: 'pages/CustomSkillInput',
  params: {
    totalSkills: this.totalSkills.toString(),
    skillsPerPlayer: this.skillsPerPlayer.toString()
  }
});

// 跳转到结果页面（带数据）
router.pushUrl({
  url: 'pages/ResultPage',
  params: {
    players: JSON.stringify(this.players)
  }
});
```

## 样式系统

### 颜色定义
```typescript
// 技能类型颜色
const SkillTypeColors = {
  active: {
    text: '#007DFF',
    bg: '#F0F7FF'
  },
  passive: {
    text: '#34C759', 
    bg: '#F0FFF4'
  },
  ultimate: {
    text: '#FF9500',
    bg: '#FFF8F0'
  },
  special: {
    text: '#AF52DE',
    bg: '#F9F0FF'
  }
};

// 获取技能类型颜色
getSkillTypeColor(type: string): ResourceColor {
  switch (type) {
    case 'active': return '#007DFF';
    case 'passive': return '#34C759';
    case 'ultimate': return '#FF9500';
    case 'special': return '#AF52DE';
    default: return '#8E8E93';
  }
}
```

### 组件样式
```typescript
// 按钮样式
Button('开始分配技能', { type: ButtonType.Capsule })
  .width(280)
  .height(56)
  .fontSize(18)
  .fontWeight(FontWeight.Medium)
  .backgroundColor('#007DFF')
  .fontColor(Color.White)

// 输入框样式
TextInput({
  placeholder: '请输入技能名称',
  text: this.skillName
})
.width('100%')
.height(44)
.backgroundColor(Color.White)
.border({
  width: 1,
  color: '#E5E5E5',
  radius: 8
})
.padding({ left: 12, right: 12 })
.fontSize(16)
.fontColor('#1A1A1A')
```

## 性能优化

### 1. 减少重新渲染
```typescript
// 使用本地状态减少全局状态更新
@State skillNameInputs: string[] = [];  // 本地状态
@State skillDescInputs: string[] = [];  // 本地状态

// 只在失去焦点时更新全局状态
.onEditChange((isEditing: boolean) => {
  if (!isEditing) {
    this.updateSkillNameGlobal(index);
  }
})
```

### 2. 列表渲染优化
```typescript
// 使用ForEach渲染列表
ForEach(this.skills, (skill: CustomSkill, index: number) => {
  this.SkillInputCard(skill, index)
})

// 使用@Builder构建可复用组件
@Builder
SkillInputCard(skill: CustomSkill, index: number) {
  // 组件内容
}
```

### 3. 内存管理
```typescript
// 使用slice()创建新数组而不是直接修改
updateSkillNameGlobal(index: number): void {
  const newSkills = this.skills.slice();  // 创建新数组
  // ... 修改新数组
  this.skills = newSkills;  // 替换整个数组
}
```

## 测试要点

### 单元测试
```typescript
// 测试随机分配算法
test('distributeSkills should allocate skills evenly', () => {
  const skills = generateTestSkills(10);
  const players = distributeSkills(skills, 4, 2);
  
  expect(players.length).toBe(4);
  players.forEach(player => {
    expect(player.skills.length).toBe(2);
  });
});

// 测试验证逻辑
test('validateSkills should detect empty fields', () => {
  const skills = [
    { id: 1, name: '技能1', description: '描述1', type: 'active' },
    { id: 2, name: '', description: '描述2', type: 'passive' }
  ];
  
  const result = validateSkills(skills);
  expect(result.valid).toBe(false);
  expect(result.message).toContain('技能 2');
});
```

### 集成测试
1. **页面跳转测试**：验证路由是否正确
2. **数据传递测试**：验证参数传递是否完整
3. **状态同步测试**：验证本地状态和全局状态同步
4. **用户体验测试**：验证焦点管理、输入体验

### 性能测试
1. **渲染性能**：测试大量技能（50+）的渲染速度
2. **内存使用**：监控内存占用情况
3. **响应时间**：测试用户操作的响应延迟

## 扩展开发

### 添加新功能
1. **技能库管理**：
   ```typescript
   // 添加技能保存功能
   saveSkillLibrary(skills: CustomSkill[]): void {
     // 保存到本地存储
   }
   
   // 加载技能库
   loadSkillLibrary(): CustomSkill[] {
     // 从本地存储加载
   }
   ```

2. **分配策略扩展**：
   ```typescript
   // 添加权重分配
   distributeSkillsWithWeight(
     skills: WeightedSkill[], 
     players: Player[]
   ): Player[] {
     // 根据权重分配技能
   }
   ```

3. **结果导出**：
   ```typescript
   // 导出为JSON
   exportResultsAsJson(players: Player[]): string {
     return JSON.stringify(players, null, 2);
   }
   
   // 导出为文本
   exportResultsAsText(players: Player[]): string {
     // 格式化为文本
   }
   ```

### 国际化支持
```typescript
// 添加多语言支持
const Strings = {
  zh: {
    presetMode: '预设模式',
    customMode: '自定义模式',
    startDistribution: '开始分配技能'
  },
  en: {
    presetMode: 'Preset Mode',
    customMode: 'Custom Mode', 
    startDistribution: 'Start Distribution'
  }
};
```

## 部署指南

### 构建配置
```json
// build-profile.json5
{
  "app": {
    "signingConfigs": [],
    "products": [
      {
        "name": "default",
        "signingConfig": "default",
        "compileSdkVersion": 11,
        "compatibleSdkVersion": 11
      }
    ]
  },
  "modules": [
    {
      "name": "entry",
      "srcPath": "./entry",
      "targets": [
        {
          "name": "default",
          "applyToProducts": ["default"]
        }
      ]
    }
  ]
}
```

### 发布流程
1. **代码检查**：运行构建和测试
2. **版本更新**：更新版本号和构建号
3. **签名配置**：配置应用签名
4. **构建发布**：生成HAP文件
5. **上架审核**：提交到应用市场

## 故障排除

### 常见问题
1. **构建失败**：检查依赖版本和配置
2. **运行时错误**：查看控制台日志
3. **性能问题**：优化状态管理和渲染
4. **兼容性问题**：检查API版本和设备支持

### 调试技巧
```typescript
// 添加调试日志
console.log('当前技能数量:', this.skills.length);
console.log('玩家数量:', this.playerCount);
console.log('分配结果:', this.players);

// 使用断点调试
debugger; // 在DevTools中暂停执行
```

## 贡献指南

### 代码规范
1. **命名规范**：使用驼峰命名法
2. **类型定义**：为所有变量和函数添加类型
3. **注释规范**：为复杂逻辑添加注释
4. **组件拆分**：保持组件单一职责

### 提交规范
```
feat: 添加新功能
fix: 修复问题
docs: 更新文档
style: 代码格式调整
refactor: 代码重构
test: 添加测试
chore: 构建或工具更新
```

### 分支管理
- `main`：主分支，稳定版本
- `develop`：开发分支，功能集成
- `feature/*`：功能开发分支
- `bugfix/*`：问题修复分支
- `release/*`：发布分支

---

**Happy Coding!** 🚀

*如有问题，请查看代码注释或提交Issue*