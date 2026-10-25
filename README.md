# AutoTypeForm —— 配置驱动 · 类型自动推导（Vue3 + TS）

一个极简 Demo：**传入配置数组，表单的返回类型 100% 由配置自动推导，全程无需手写 interface**。

- 技术栈：Vite + Vue 3.5 + TypeScript（`<script setup lang="ts">`，原生控件，无 UI 库）
- 代码只有两个文件：`src/AutoTypeForm.vue`（泛型组件）、`src/App.vue`（使用方）

---

## 一、它到底解决了什么问题？

### 痛点（传统写法）

维护表单时，配置和类型是**两份脱节的数据**，你需要手写对应的 interface：

```ts
// 配置
const config = [
  { key: 'name', label: '姓名', type: 'input' },
  { key: 'age', label: '年龄', type: 'number' },
  { key: 'gender', label: '性别', type: 'select', options: ['男', '女'] },
]

// 类型 —— 必须人肉同步，增删字段要记得回来改这里
interface Person {
  name: string     // 其实 config 里没写，是"约定"
  age?: number     // age 到底可空吗？选项里没体现
  gender: string   // select 的选项居然被拍扁成 string，丢了 '男'|'女'！
}
```

问题：不改配置类型对不上、类型手工维护易腐化、select 选项的约束层级直接被丢弃。

### 解决方式

把配置数组用 `as const` 声明，组件用泛型 `T extends readonly FieldConfig[]` 接收配置，TS 在**编译期**把配置映射成精确类型：

```ts
const config = [ ... ] as const

type Person = FormModel<typeof config>
// => { name: string; age: number; gender: '男' | '女'; level: 1 | 2 | 3 }
```

收益：

| 痛点 | 解决后 |
| --- | --- |
| 手写 interface，配置与类型不同步 | 类型由配置**单一来源**自动生成 |
| select 选项被宽化成 `string` | 推导出**字面量联合类型** `'男' \| '女'`、`1 \| 2 \| 3` |
| 填错字段名/类型不报错 | `onSubmit(value)` 参数自动带类型，编译期校验 |
| 增删字段要改多处 | 改一处配置，类型同步更新 |

---

## 二、怎么去查看 / 运行

```bash
# 1. 安装依赖（国内环境建议用镜像，更快）
npm install --registry https://registry.npmmirror.com

# 2. 启动开发服务器
npm run dev
#    → 打开 http://localhost:5173/
```

页面上可交互验证：

1. 填写 姓名 / 年龄，选择 性别（男·女）和 级别（1·2·3）
2. 点击 **提交并打印**
3. 观察两处输出：
   - 页面「提交结果」区域的 `<pre>`（自动更新为 JSON）
   - 浏览器 DevTools 控制台的 `console.log('提交的表单数据：', value)`

其他命令：

```bash
npm run build   # vue-tsc --noEmit 类型门禁 + vite 打包，类型不通过则构建失败
npm run preview # 预览打包产物
```

> 验证"类型真的被推导"最直接的办法：在 `App.vue` 的 `onSubmit` 里把 `value.age` 传给字符串参数，编辑器/VSCode 会立刻标红——因为 `age` 是 `number`。

---

## 三、怎么去逐步分析

核心是一条**类型数据流**：`as const` → 泛型参数 → 条件类型 → 映射类型 → emit 透出 → 用方消费。按下面 6 步走即可。

### 第 1 步：`as const` 保留字面量（App.vue: [配置](src/App.vue#L6-L11)）

```ts
const config = [
  { type: 'input' },
  { type: 'number' },
  { type: 'select', options: ['男', '女'] as const },
  { type: 'select', options: [1, 2, 3] as const },
] as const
```

没有 `as const`，`type` 会被宽化成 `string`、`options` 宽化成 `string[]`，后面一切推导都无从谈起。

**验证**：把 `as const` 删掉，再看第 2 步的类型推导结果，`gender` 会变成 `string`。

### 第 2 步：泛型组件接收配置（AutoTypeForm.vue: [generic](src/AutoTypeForm.vue#L26-L31)）

```html
<script setup lang="ts" generic="T extends readonly FieldConfig[]">
```

- `generic="..."` 是 Vue 3.3+ 的 `<script setup>` 泛型语法，等价于函数式组件的 `defineComponent<T>()`
- `T` 就是你传入的**整个配置的字面量类型**（4 个字段的元组）
- `FieldConfig` 是约束：每个字段必须满足 `FieldConfig` 结构（有 `key`、`type`，select 可选 `options`）

```ts
defineProps<{ config: T }>()
const emit = defineEmits<{ submit: [value: FormModel<T>] }>()
```

`T` 从「父组件传的 prop」里被推断出来，然后喂给 `FormModel<T>`。

### 第 3 步：`FieldValue<F>` 条件类型 —— 单个字段的值类型（AutoTypeForm.vue: [FieldValue](src/AutoTypeForm.vue#L14-L18)）

```ts
export type FieldValue<F> =
  F extends { type: 'select'; options: readonly (infer O)[] } ? O : // 取 options 元素类型
  F extends { type: 'number' } ? number :
  F extends { type: 'input' } ? string :
  never
```

条件类型按字段的 `type` 分发：

| 传入字面量 | 命中分支 | 结果 |
| --- | --- | --- |
| `{ type: 'input' }` | 第 3 行 | `string` |
| `{ type: 'number' }` | 第 2 行 | `number` |
| `{ type: 'select'; options: readonly ['男','女'] }` | 第 1 行（`infer O` 抓出元素） | `'男' \| '女'` |

`infer O` 是这里的关键：它从 `options` 里**反向抓取元素类型**，字面量级别保留。

### 第 4 步：`FormModel<T>` 映射类型 —— 把数组转成对象（AutoTypeForm.vue: [FormModel](src/AutoTypeForm.vue#L21-L23)）

```ts
export type FormModel<T extends readonly FieldConfig[]> = {
  [F in T[number] as F['key']]: FieldValue<F>
}
```

- `T[number]`：取出元组里每个字段（如 4 个字段的联合）
- `as F['key']`：**键重映射**，把每个字段的 `key` 作为结果对象的属性名
- 值为第 3 步算出的 `FieldValue<F>`

展开过程（想象 TS 编译期的机械化推演）：

```text
T = [{key:'name',type:'input'},
     {key:'age',type:'number'},
     {key:'gender',type:'select',options:['男','女']},
     {key:'level',type:'select',options:[1,2,3]}]

→ { name: string, age: number, gender: '男'|'女', level: 1|2|3 }
```

### 第 5 步：类型从 emit 透出到用方（App.vue: [Person](src/App.vue#L13-L15)）

```ts
type Person = FormModel<typeof config>
```

组件内部为了 `v-model` 索引用了宽松的 `data`（模板里 `field.key` 是普通 `string`，无法按字面量收缩），但对外 `emit` 的类型签名是精确的 `FormModel<T>`——类型安全完整保留在**类型系统边界**，`onSubmit(value)` 里的 `value` 不带手写注解就是 `Person`。

### 第 6 步：编译期自测，证明推导正确（App.vue: [Assert](src/App.vue#L18-L20)）

```ts
type Assert<T extends true> = T
type _checkGender = Assert<'男' extends Person['gender'] ? true : false>
type _checkLevel  = Assert<2 extends Person['level'] ? true : false>
```

- 若 `gender` 被错误推导成 `string`，`'男' extends string` 为真 → 其实这步检测不出来
- 正确的写法是**反向**检测（把例子反过来）：`typeof '男'` 是否被收窄。这里用正向断言证明字面量存在，实际项目配合 **expect-type** 库做双向断言更严谨
- 关键运行 `npm run build`（内部含 `vue-tsc --noEmit`），**类型不过则构建失败**，这就是推导正确性的自动化闸门

### 一张图看完整数据流

```text
App.vue  const config = [...] as const     ← ① 字面量保留
   │  :config="config"  (prop)
   ▼
AutoTypeForm.vue  generic="T extends readonly FieldConfig[]"   ← ② 泛型捕获字面量类型
   │
   ├─ FieldValue<F>：条件类型按 type 分发每个字段的值类型        ← ③ 单字段映射
   └─ FormModel<T> = { [F in T[number] as F['key']]: FieldValue<F> }  ← ④ 数组→对象
   │
   │  emit('submit', data)  类型 = FormModel<T>                 ← ⑤ 精确类型透出
   ▼
App.vue  function onSubmit(value: Person /* 推导所得 */)        ← ⑥ 零手写消费
```

---

## 常见疑问

- **为什么组件内部用 `reactive<any>`？** 模板中 `data[field.key]` 的 `key` 是宽泛的 `string`，即便有字面量也不知道是哪个字段，强行收紧要么报错要么代码很绕。内部宽松、对外精确，是刻意保留"推导类型供外部消费"这个核心的取舍。
- **建字段后表单类型自动变吗？** 会。在 `config` 里加一行，`Person` 立刻多一个属性，`onSubmit`、`result` 全部同步。删字段同理。
- **select 支持数字选项吗？** 支持：`options: [1, 2, 3] as const` 推导出 `1 | 2 | 3`。