<template>
  <div class="welcome-screen">
    <div class="big-avatar">
      <svg width="80" height="80" viewBox="0 0 80 80" fill="none" xmlns="http://www.w3.org/2000/svg">
        <rect width="80" height="80" rx="20" fill="url(#grad)" />
        <text x="40" y="54" text-anchor="middle" fill="white" font-size="42" font-weight="700" font-family="Inter, sans-serif">d</text>
        <defs>
          <linearGradient id="grad" x1="0" y1="0" x2="80" y2="80">
            <stop offset="0%" stop-color="#4F46E5" />
            <stop offset="100%" stop-color="#7C3AED" />
          </linearGradient>
        </defs>
      </svg>
    </div>
    <h3>你好呀！我是 deep</h3>
    <p>我可以帮你写代码、回答问题，或者只是陪你聊聊天 ✨</p>
  </div>
</template>

<script setup lang="ts">
// Simple static component
</script>