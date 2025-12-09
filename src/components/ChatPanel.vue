<script setup>
import { ref, onMounted, nextTick } from 'vue'

const messages = ref([{ role: 'system', text: '欢迎来到棋局聊天。' }])
const input = ref('')
const listEl = ref(null)

const scrollToBottom = () => {
  nextTick(() => {
    if (listEl.value) {
      listEl.value.scrollTop = listEl.value.scrollHeight
    }
  })
}

const send = () => {
  const text = input.value.trim()
  if (!text) return
  messages.value.push({ role: 'me', text })
  input.value = ''
  scrollToBottom()
}

onMounted(scrollToBottom)
</script>

<template>
  <div class="chat-panel">
    <div class="chat-header">对局聊天</div>
    <div class="chat-messages" ref="listEl">
      <div
        v-for="(msg, idx) in messages"
        :key="idx"
        :class="['chat-message', msg.role]"
      >
        <span class="chat-role">
          {{ msg.role === 'me' ? '我' : '系统' }}
        </span>
        <span class="chat-text">{{ msg.text }}</span>
      </div>
    </div>
    <div class="chat-input">
      <textarea
        v-model="input"
        rows="3"
        placeholder="输入聊天内容，按发送"
        @keydown.enter.exact.prevent="send"
      ></textarea>
      <button type="button" class="send-btn" @click="send">发送</button>
    </div>
  </div>
</template>

