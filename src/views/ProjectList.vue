<!-- src/views/ProjectList.vue -->
<template>
  <div class="project-list-container">
    <h1>实验项目列表</h1>

    <!-- 加载状态 -->
    <div v-if="loading" class="loading">
      <el-spinner size="large" />
      <span>加载中...</span>
    </div>

    <!-- 空状态 -->
    <div v-else-if="projects.length === 0" class="empty">
      <el-empty description="暂无项目" />
    </div>

    <!-- 项目列表 -->
    <div v-else class="project-grid">
      <div 
        v-for="project in projects" 
        :key="project.id" 
        class="project-card"
      >
        <h3>{{ project.name }}</h3>
        <p><strong>类型：</strong>{{ project.type }}</p>
        <p><strong>难度：</strong>{{ project.level }}</p>
        <p><strong>创建时间：</strong>{{ formatDate(project.created_at) }}</p>

        <!-- 关键：带 projectId 的链接 -->
        <router-link 
          :to="{ name: 'AsrFinetune', params: { projectId: project.id } }"
          class="el-button el-button--primary el-button--small"
        >
          进入 ASR 模型微调
        </router-link>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import { ElMessage } from 'element-plus'
import axios from 'axios'

// 状态
const projects = ref([])
const loading = ref(true)

// 获取项目列表（调用后端 /api/projects）
const fetchProjects = async () => {
  try {
    const response = await axios.get('/api/projects')
    projects.value = response.data.projects || []
  } catch (error) {
    console.error('获取项目失败:', error)
    ElMessage.error('加载项目失败：' + error.message)
  } finally {
    loading.value = false
  }
}

// 格式化时间
const formatDate = (dateString) => {
  const date = new Date(dateString)
  return date.toLocaleString('zh-CN', {
    year: 'numeric',
    month: '2-digit',
    day: '2-digit',
    hour: '2-digit',
    minute: '2-digit'
  })
}

// 页面加载时获取数据
onMounted(() => {
  fetchProjects()
})
</script>

<style scoped>
.project-list-container {
  padding: 20px;
  max-width: 1200px;
  margin: 0 auto;
}

.project-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 20px;
  margin-top: 20px;
}

.project-card {
  border: 1px solid #e4e7ed;
  border-radius: 8px;
  padding: 16px;
  background: #fff;
  box-shadow: 0 2px 4px rgba(0,0,0,0.05);
  transition: all 0.3s;
}

.project-card:hover {
  box-shadow: 0 4px 12px rgba(0,0,0,0.1);
  transform: translateY(-2px);
}

.project-card h3 {
  margin: 0 0 12px 0;
  color: #303133;
  font-size: 18px;
}

.project-card p {
  margin: 6px 0;
  color: #606266;
  font-size: 14px;
}

.loading, .empty {
  text-align: center;
  padding: 40px;
  color: #909399;
}

.el-button {
  margin-top: 12px;
  width: 100%;
}
</style>