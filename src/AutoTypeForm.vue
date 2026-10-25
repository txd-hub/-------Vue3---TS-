<!-- 泛型表单组件：传入 as const 配置数组，自动推导出表单返回类型，无需手写 interface -->
<script lang="ts">
// ===== 类型定义（普通 script 块，可被外部 import）=====
/** 字段配置：input / number / select 三种控件 */
export interface FieldConfig {
  key: string
  label?: string
  type: 'input' | 'number' | 'select'
  /** 仅 select 需要；配合 as const 推导出字面量联合类型 */
  options?: readonly (string | number)[]
}

/** 核心推导：按 type 把每个字段映射为具体的值类型 */
export type FieldValue<F> =
  F extends { type: 'select'; options: readonly (infer O)[] } ? O :
  F extends { type: 'number' } ? number :
  F extends { type: 'input' } ? string :
  never

/** 泛型映射：把配置数组 T 转成 { key: 值类型 } */
export type FormModel<T extends readonly FieldConfig[]> = {
  [F in T[number] as F['key']]: FieldValue<F>
}
</script>

<script setup lang="ts" generic="T extends readonly FieldConfig[]">
import { reactive } from 'vue'

// T 由父组件传入的 as const 配置在编译期自动推断
defineProps<{ config: T }>()
const emit = defineEmits<{ submit: [value: FormModel<T>] }>()

// 内部用宽松类型承接表单值（key 索引在模板中无法按字面量类型收缩），
// 对外通过 emit 暴露精确的 FormModel<T>
const data = reactive<any>({})

function handleSubmit() {
  emit('submit', data)
}
</script>

<template>
  <form class="auto-form" @submit.prevent="handleSubmit">
    <label v-for="field in config" :key="field.key" class="field">
      <span class="name">{{ field.label ?? field.key }}</span>
      <input
        v-if="field.type === 'input'"
        v-model="data[field.key]"
        :placeholder="field.label ?? field.key"
      />
      <input
        v-else-if="field.type === 'number'"
        v-model="data[field.key]"
        type="number"
      />
      <select v-else v-model="data[field.key]">
        <option v-for="opt in field.options ?? []" :key="opt" :value="opt">{{ opt }}</option>
      </select>
    </label>
    <button type="submit">提交并打印</button>
  </form>
</template>

<style scoped>
.auto-form { display: flex; flex-direction: column; gap: 12px; }
.field { display: flex; flex-direction: column; gap: 4px; font-size: 14px; }
.name { color: #555; }
.auto-form :is(input, select) {
  padding: 8px 10px; border: 1px solid #ccc; border-radius: 6px; font-size: 14px;
}
.auto-form input[type='number'] { width: 100px; }
.auto-form button {
  align-self: flex-start; padding: 8px 18px; border: none; border-radius: 6px;
  background: #4f6ef7; color: #fff; font-size: 14px; cursor: pointer;
}
</style>