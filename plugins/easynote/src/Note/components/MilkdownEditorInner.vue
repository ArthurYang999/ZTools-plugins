<template>
  <Milkdown />
</template>

<script setup lang="ts">
import { Milkdown, useEditor } from '@milkdown/vue'
import { Editor, rootCtx, defaultValueCtx, editorViewOptionsCtx } from '@milkdown/kit/core'
import { DOMParser as PMDOMParser } from '@milkdown/prose/model'
import { commonmark } from '@milkdown/kit/preset/commonmark'
import { nord } from '@milkdown/theme-nord'
import '@milkdown/theme-nord/style.css'
import { listener, listenerCtx } from '@milkdown/kit/plugin/listener'

const props = defineProps<{ modelValue: string }>()
const emit = defineEmits<{ (e: 'update:modelValue', v: string): void }>()

// useEditor 必须在 MilkdownProvider 的子组件中调用（inject provider 提供的 context）。
// 回调返回「未 create 的 Editor」，由 Milkdown 组件内部（useGetEditor）负责 create()。
useEditor((root) =>
  Editor.make()
    .config(nord)
    .config((ctx) => {
      ctx.set(rootCtx, root)
      ctx.set(defaultValueCtx, props.modelValue || '')
      // 纯文本粘贴保结构：单换行→硬换行，k 个连续换行→段落边界 + (k-1) 个空段落。
      // ProseMirror 默认把每个 \n 都拆成独立段落，贴 4 行文本会变成 4 个段落（序列化后行间多空行）。
      ctx.update(editorViewOptionsCtx, (prev) => ({
        ...prev,
        clipboardTextParser: (text, $context) => {
          const schema = $context.doc.type.schema
          const dom = document.createElement('div')
          let p: HTMLParagraphElement | null = null
          let pending = 0
          for (const line of text.replace(/\r\n?/g, '\n').split('\n')) {
            if (!p) {
              if (!line) continue // 开头的空行忽略
              p = document.createElement('p')
              p.appendChild(document.createTextNode(line))
              dom.appendChild(p)
              continue
            }
            if (!line) {
              pending++
              continue
            }
            // 本行与前一行之间共 k = pending + 1 个换行
            const k = pending + 1
            if (k === 1) {
              p.appendChild(document.createElement('br'))
            } else {
              for (let i = 0; i < k - 1; i++) dom.appendChild(document.createElement('p'))
              p = document.createElement('p')
              dom.appendChild(p)
            }
            pending = 0
            p.appendChild(document.createTextNode(line))
          }
          return PMDOMParser.fromSchema(schema).parseSlice(dom)
        }
      }))
      ctx.get(listenerCtx).markdownUpdated((_ctx, markdown) => {
        if (markdown !== props.modelValue) emit('update:modelValue', markdown)
      })
    })
    .use(commonmark)
    .use(listener)
)
</script>
