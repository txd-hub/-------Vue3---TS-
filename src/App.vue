<script setup lang="ts">
import { ref } from 'vue'
import AutoTypeForm, { type FormModel, type FieldConfig } from './AutoTypeForm.vue'

// as const：让 type / options 保持字面量类型，供组件泛型 T 推导
const config = [
  { key: 'name', label: '姓名', type: 'input' },
  { key: 'age', label: '年龄', type: 'number' },
  { key: 'gender', label: '性别', type: 'select', options: ['男', '女'] as const },
  { key: 'level', label: '级别', type: 'select', options: [1, 2, 3] as const },
] as const

/** 零手写：表单返回类型完全由配置推导
 *  => { name: string; age: number; gender: '男' | '女'; level: 1 | 2 | 3 } */
type Person = FormModel<typeof config>

// 编译期自测（若推导错误会直接报错）：gender 必须是字面量联合
type Assert<T extends true> = T
type _checkGender = Assert<'男' extends Person['gender'] ? true : false>
type _checkLevel = Assert<2 extends Person['level'] ? true : false>

const result = ref<Person | null>(null)

// value 参数无需手写类型注解，自动获得推导结果
function onSubmit(value: Person) {
  console.log('提交的表单数据：', value)
  result.value = value
}
</script>

<template>
  <main class="page">
    <h1>AutoTypeForm<span class="sub">— 配置驱动 · 类型自动推导</span></h1>
    <AutoTypeForm :config="config" @submit="onSubmit" />
    <section class="result">
      <h2>提交结果（控制台已打印）</h2>
      <pre v-if="result">{{ result }}</pre>
      <p v-else class="empty">尚未提交</p>
    </section>
  </main>
</template>

<style scoped>
.page { max-width: 420px; margin: 40px auto; font-family: system-ui, sans-serif; }
h1 { font-size: 20px; display: flex; align-items: baseline; gap: 8px; }
.sub { font-size: 13px; color: #888; font-weight: 400; }
.result { margin-top: 24px; border-top: 1px dashed #ddd; padding-top: 12px; }
.result h2 { font-size: 14px; color: #888; }
pre { background: #f6f8fa; padding: 10px; border-radius: 6px; font-size: 13px; }
.empty { font-size: 13px; color: #aaa; }
</style>