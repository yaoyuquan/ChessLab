<script setup>
import { ref } from 'vue'
import ChessBoard from './components/ChessBoard.vue'
import ChatPanel from './components/ChatPanel.vue'
import './App.css'

// 执棋方（白/黑）
const playerColor = ref('white')

// 用于触发子组件重置
const resetKey = ref(0)
const hasStarted = ref(false)

// 开始/重置：切换 key 让子组件恢复初始布局
const startGame = () => {
  resetKey.value++
  hasStarted.value = true
}

const resign = (color) => {
  // 直接通过事件让子组件判定胜负
  hasStarted.value = false
  // 发送自定义事件到 ChessBoard，通过 resetKey 让其感知并重置状态消息
  resetKey.value++
  // 简单提示
  // 这里仅做前端提示，聊天逻辑已独立到 ChatPanel，可在其中扩展监听
}
</script>

<template>
  <div class="page">
    <div class="chess-container">
      <div class="controls">
        <span>执棋方：</span>
        <button
          type="button"
          :class="['color-btn', playerColor === 'white' ? 'active' : '']"
          :disabled="hasStarted"
          @click="playerColor = 'white'"
        >
          执白
        </button>
        <button
          type="button"
          :class="['color-btn', playerColor === 'black' ? 'active' : '']"
          :disabled="hasStarted"
          @click="playerColor = 'black'"
        >
          执黑
        </button>
        <button
          type="button"
          class="start-btn"
          @click="startGame"
        >
          开始
        </button>
      <button
        type="button"
        class="resign-btn"
        @click="resign(playerColor)"
        :disabled="!hasStarted"
      >
        认输
      </button>
      </div>

      <ChessBoard
        :player-color="playerColor"
        :reset-key="resetKey"
        @gameEnded="hasStarted = false"
      />
    </div>

    <ChatPanel />
  </div>
</template>
