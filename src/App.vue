<template>
  <div class="app-container">
    <!-- 顶部标题 -->
    <div class="header">
      <h1>Torn 帮派实力分析器</h1>
    </div>

    <!-- 主内容区域 -->
    <div class="main-content">
      <!-- 帮派实力分析 -->
      <div class="content-section">
        <FactionComparison />
      </div>
    </div>
  </div>
</template>

<script setup>
import FactionComparison from './components/FactionComparison.vue'
</script>

<style scoped>
.app-container {
  min-height: 100vh;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  padding: 20px;
}

.header {
  text-align: center;
  margin-bottom: 30px;
}

.header h1 {
  color: white;
  font-size: 2.5rem;
  font-weight: bold;
  text-shadow: 2px 2px 4px rgba(0,0,0,0.3);
  margin: 0;
}

.main-content {
  max-width: 1400px;
  margin: 0 auto;
}

.content-section {
  background: white;
  border-radius: 15px;
  box-shadow: 0 8px 32px rgba(0,0,0,0.1);
  overflow: hidden;
}
</style>
