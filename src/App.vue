<script setup>
import ChineseChess from './components/ChineseChess.vue'
</script>

<template>
  <div class="min-h-screen bg-gradient-to-br from-slate-900 via-slate-800 to-slate-900">
    <div class="container mx-auto px-4 py-8">
      <div class="text-center mb-8">
        <h1 class="text-5xl font-bold text-white mb-2">象棋</h1>
        <p class="text-xl text-slate-300">Chinese Chess (Xiangqi)</p>
      </div>
      <ChineseChess />
    </div>
  </div>
</template>