<!-- src/views/ExperimentProjects.vue -->
<template>
  <div class="experiment-projects">
    <!-- 页面标题 -->
    <h2>实验项目</h2>
    <!-- 管理员专用：创建实验项目按钮 -->
    <el-button v-if="isAdmin" type="primary" @click="openCreateModal" style="margin-bottom: 20px;">
      创建实验项目
    </el-button>
    <!-- 项目列表表格 -->
    <el-table :data="projects" style="width: 100%" border>
      <el-table-column prop="e_name" label="项目名称" />
      <el-table-column prop="e_type" label="实验类型" />
      <el-table-column prop="e_level" label="等级" />
      <el-table-column prop="e_description" label="项目描述" width="200" />
      <!-- 管理员操作列 -->
      <el-table-column label="操作" v-if="isAdmin">
        <template #default="{ row }">
          <el-button type="primary" size="small" @click="openDesignModal(row)">设计实验</el-button>
          <el-button type="success" size="small" @click="openDistributeModal(row)">下发实验</el-button>
          <el-button type="info" size="small" @click="enterExperiment(row)">进入实验</el-button>
          <el-button type="warning" size="small" @click="editProject(row)">编辑</el-button>
          <el-button type="danger" size="small" @click="handleDeleteProject(row.e_id)">删除</el-button>
        </template>
      </el-table-column>
      <!-- 学生操作列 -->
      <el-table-column label="操作" v-else>
        <template #default="{ row }">
          <el-button type="primary" size="small" @click="viewGuide(row)">进入实验</el-button>
        </template>
      </el-table-column>
    </el-table>

    <!-- 创建/编辑项目模态框 -->
    <el-dialog :title="editMode ? '编辑实验项目' : '创建实验项目'" v-model="showCreateModal" width="30%">
      <el-form :model="projectForm" :rules="rules" ref="projectFormRef" label-width="100px">
        <el-form-item label="项目名称" prop="name">
          <el-input v-model="projectForm.name" placeholder="请输入项目名称" clearable></el-input>
        </el-form-item>
        <el-form-item label="实验类型" prop="type">
          <el-select v-model="projectForm.type" placeholder="请选择实验类型" clearable>
            <el-option label="语音识别" value="Speech Recognition"></el-option>
            <el-option label="语音合成" value="Speech Synthesis"></el-option>
          </el-select>
        </el-form-item>
        <el-form-item label="等级" prop="level">
          <el-select v-model="projectForm.level" placeholder="请选择等级" clearable>
            <el-option label="初级" value="Beginner"></el-option>
            <el-option label="中级" value="Intermediate"></el-option>
            <el-option label="高级" value="Advanced"></el-option>
          </el-select>
        </el-form-item>
        <el-form-item label="项目描述" prop="description">
          <el-input
            v-model="projectForm.description"
            type="textarea"
            :rows="3"
            placeholder="请输入项目描述（可选）"
            clearable
          ></el-input>
        </el-form-item>
      </el-form>
      <template #footer>
        <span class="dialog-footer">
          <el-button @click="showCreateModal = false">取消</el-button>
          <el-button type="primary" @click="submitProject">确定</el-button>
        </span>
      </template>
    </el-dialog>

    <!-- 设计实验模态框 -->
    <el-dialog title="设计实验指导书" v-model="showDesignModal" width="50%">
      <el-form :model="guideForm" ref="guideFormRef">
        <div v-for="(step, index) in guideForm.steps" :key="index" class="step-container">
          <el-form-item :label="'步骤 ' + (index + 1)" :prop="'steps.' + index + '.title'" :rules="[{ required: true, message: '请输入步骤标题', trigger: 'blur' }]">
            <el-input v-model="step.title" placeholder="请输入步骤标题" clearable></el-input>
          </el-form-item>
          <el-form-item :prop="'steps.' + index + '.content'" :rules="[{ required: true, message: '请输入步骤内容', trigger: 'blur' }]">
            <el-input v-model="step.content" type="textarea" :rows="4" placeholder="请输入步骤内容" clearable></el-input>
          </el-form-item>
          <el-button type="danger" v-if="guideForm.steps.length > 1" @click="removeStep(index)">删除步骤</el-button>
        </div>
        <el-button type="primary" @click="addStep">添加步骤</el-button>
      </el-form>
      <template #footer>
        <span class="dialog-footer">
          <el-button @click="showDesignModal = false">取消</el-button>
          <el-button type="primary" :loading="saveLoading" @click="saveGuide">保存</el-button>
        </span>
      </template>
    </el-dialog>

    <!-- 下发实验模态框 -->
    <el-dialog title="下发实验" v-model="showDistributeModal" width="30%">
      <el-form :model="distributeForm" ref="distributeFormRef">
        <el-form-item label="选择学生IP" prop="ips" :rules="[{ required: true, message: '请选择学生IP', trigger: 'change' }]">
          <el-select v-model="distributeForm.ips" multiple placeholder="请选择学生IP" clearable>
            <el-option v-for="ip in availableIPs" :key="ip" :label="ip" :value="ip"></el-option>
          </el-select>
        </el-form-item>
      </el-form>
      <template #footer>
        <span class="dialog-footer">
          <el-button @click="showDistributeModal = false">取消</el-button>
          <el-button type="primary" @click="distributeProject">确定</el-button>
        </span>
      </template>
    </el-dialog>
  </div>
</template>

<script>
import { ref, onMounted, watch } from 'vue';
import { ElMessage, ElMessageBox } from 'element-plus';
import api, {
  getUser,
  getProjects,
  createProject,
  updateProject,
  deleteProject,
  saveGuideAPI,
  getStudentIPs,
  distributeProject,
} from '@/api/index.js';
import io from 'socket.io-client';
import { API_BASE_URL } from '@/api/index.js';

export default {
  name: 'ExperimentProjects',
  setup() {
    const saveLoading = ref(false);   // 2. 保存按钮 loading
    const projects = ref([]);
    const isAdmin = ref(false);
    const availableIPs = ref([]);
    const showCreateModal = ref(false);
    const editMode = ref(false);
    const currentProjectId = ref(null);
    const projectForm = ref({
      name: '',
      type: '',
      level: '',
      description: '',
    });
    const projectFormRef = ref(null);
    const showDesignModal = ref(false);
    const guideForm = ref({
      project_id: null,
      steps: [{ title: '', content: '' }],
    });
    const guideFormRef = ref(null);
    const showDistributeModal = ref(false);
    const distributeForm = ref({
      ips: [],
    });
    const distributeFormRef = ref(null);
  

    const rules = {
      name: [{ required: true, message: '请输入项目名称', trigger: 'blur' }],
      type: [{ required: true, message: '请选择实验类型', trigger: 'change' }],
      level: [{ required: true, message: '请选择等级', trigger: 'change' }],
    };

    // WebSocket 连接
    const socket = io(API_BASE_URL, {
      transports: ['websocket'],
      path: '/socket.io',          // 默认即可，显式写出来防止 nginx 转发异常
      withCredentials: true,       // 允许携带 cookie（JWT 需要）
    });

    socket.on('connect', () => {
      console.log('WebSocket connected:', socket.id);
    });

    socket.on('connect_error', (error) => {
      console.error('WebSocket connect error:', error);
    });

    socket.on('disconnect', () => {
      console.log('WebSocket disconnected');
    });

    socket.on('project_distributed', (project) => {
      if (project && !projects.value.find((p) => p.id === project.id)) {
        projects.value.push(project);
        ElMessage.success(`收到新项目：${project.name || '未知项目'}`);
      }
    });

    // 调试 projectForm 变化
    watch(projectForm, (newVal) => {
      console.log('projectForm updated:', newVal);
    }, { deep: true });

    // 打开创建模态框并重置表单
    const openCreateModal = () => {
      editMode.value = false;
      currentProjectId.value = null;
      projectForm.value = { name: '', type: '', level: '', description: '' };
      showCreateModal.value = true;
    };

    onMounted(async () => {
      try {
        const userRes = await getUser();
        console.log('getUser 返回：', userRes);
        const role = userRes.data.role;
        isAdmin.value = role === 'admin';
        console.log('当前角色：', role, 'isAdmin=', isAdmin.value);

        await loadProjects();  // 使用统一函数
      } catch (e) {
        console.error('初始化失败', e);
        ElMessage.error('登录状态异常');
      }
    });

    // 加载项目列表
    const loadProjects = async () => {
      try {
        const { data } = await getProjects();
        projects.value = data || [];
      } catch (error) {
        console.error('加载项目失败', error);
        ElMessage.error('加载项目失败');
      }
    };

    // 提交项目（创建 + 编辑，都传 description）
    const submitProject = async () => {
      if (!projectFormRef.value) {
        console.error('projectFormRef is not defined');
        return;
      }
      try {
        await projectFormRef.value.validate();
        console.log('Submitting projectForm:', projectForm.value);

        if (editMode.value) {
          // 编辑项目：传 name, type, level, description
          await updateProject(currentProjectId.value, {
            name: projectForm.value.name,
            type: projectForm.value.type,
            level: projectForm.value.level,
            description: projectForm.value.description
          });
          // 更新本地列表
          projects.value = projects.value.map(p =>
            p.id === currentProjectId.value
              ? { ...p, ...projectForm.value }
              : p
          );
          ElMessage.success('项目更新成功');
        } else {
          // 创建项目：传完整对象
          const response = await createProject({
            name: projectForm.value.name,
            type: projectForm.value.type,
            level: projectForm.value.level,
            description: projectForm.value.description
          });
          projects.value.push(response.data);
          ElMessage.success('项目创建成功');
        }

        // 关闭弹窗 + 重置表单（包括 description）
        showCreateModal.value = false;
        projectForm.value = { name: '', type: '', level: '', description: '' };
        if (projectFormRef.value) {
          projectFormRef.value.resetFields();
        }
        editMode.value = false;
      } catch (error) {
        console.error('Submit project error:', error.response || error);
        ElMessage.error('提交项目失败: ' + (error.message || '未知错误'));
      }
    };

    // 编辑项目
    const editProject = (project) => {
      editMode.value = true;
      currentProjectId.value = project.e_id;
      projectForm.value = {
        name: project.e_name,
        type: project.e_type,
        level: project.e_level,
        description: project.e_description || ''
      };
      showCreateModal.value = true;
    };

    // 删除项目
    const isProcessing = ref(false); // 移到 setup 作用域，确保状态持久

    // 删除项目
    const handleDeleteProject = async (projectId) => {
      if (!projectId) return;

      try {
        const result = await ElMessageBox.confirm(
          '确定要删除此实验项目吗？删除后无法恢复！',
          '警告',
          {
            confirmButtonText: '确定',
            cancelButtonText: '取消',
            type: 'warning'
          }
        );

        if (result === 'confirm') {
          await api.delete(`/projects/${projectId}`);
          ElMessage.success('项目删除成功');
          await loadProjects();  // 关键：重新加载列表
        }
      } catch (error) {
        if (error !== 'cancel') {
          ElMessage.error('删除失败: ' + (error.response?.data?.error || error.message));
        }
      }
    };

    // 打开设计实验模态框
    const openDesignModal = (project) => {
      currentProjectId.value = project.e_id;
      guideForm.value.steps = (project.e_list || []).map(item => ({
        title: item.title || '',
        content: item.content || ''
      }));
      if (guideForm.value.steps.length === 0) {
        addStep(); // 至少一个步骤
      }
      showDesignModal.value = true;
    };

    // 添加指导书步骤
    const addStep = () => {
      guideForm.value.steps.push({ title: '', content: '' });
    };

    // 删除指导书步骤
    const removeStep = (index) => {
      guideForm.value.steps.splice(index, 1);
    };

    // 保存指导书（异步 + loading）
    const saveGuide = async () => {
      try {
        await guideFormRef.value.validate();
        const steps = guideForm.value.steps.map(s => ({
          title: s.title,
          content: s.content
        }));

        await api.put(`/projects/${currentProjectId.value}/guide`, { steps });
        
        ElMessage.success('指导书保存成功');
        showDesignModal.value = false;
        await loadProjects();  // 刷新列表
      } catch (error) {
        if (error !== 'cancel') {
          ElMessage.error('保存失败: ' + (error.response?.data?.error || error.message));
        }
      }
    };

    // 进入实验（管理员预览）
    const enterExperiment = async (row) => {
      try {
        const res = await api.get(`/admin/guide/${row.id}`);
        const g = res.data;

        const win = window.open('', '_blank');
        win.document.write(`
    <html>
      <head>
        <meta charset="utf-8"/>
        <title>${g.project_name} - 指导书预览</title>
        <style>
          body { font-family: Arial, sans-serif; margin: 40px; line-height: 1.6; }
          h1 { color: #409EFF; }
          ol { padding-left: 20px; }
          li { margin: 20px 0; }
          strong { color: #303133; }
          pre { white-space: pre-wrap; background: #f5f5f5; padding: 10px; border-radius: 4px; }
        </style>
      </head>
      <body>
        <h1>${g.project_name}</h1>
        <p><strong>类型：</strong>${g.project_type} | <strong>等级：</strong>${g.project_level} | <strong>创建时间：</strong>${g.created_at || '未知'}</p>
        <hr>
        <ol>
          ${g.steps.map(s => `
            <li>
              <strong>${s.title}</strong><br/>
              <pre>${s.content}</pre>
            </li>`).join('')}
        </ol>
      </body>
    </html>`);
        win.document.close();
      } catch (e) {
        ElMessage.error('加载指导书失败：' + (e.response?.data?.error || e.message));
      }
    };

    // 打开下发实验模态框
    const openDistributeModal = (project) => {
      currentProjectId.value = project.id;
      distributeForm.value.ips = [];
      showDistributeModal.value = true;
    };

    // 下发项目
    const distributeProject = async () => {
      console.log('[1] 进入函数');          // 断点 1
      if (!distributeFormRef.value) return;
      try {
        await distributeFormRef.value.validate();
        console.log('[2] 表单校验通过');     // 断点 2

        console.log('[3] 准备调接口', currentProjectId.value, distributeForm.value);
        const res = await api.post(`/projects/${currentProjectId.value}/distribute`, distributeForm.value);
        console.log('[4] 接口返回', res);

        ElMessage.success('项目下发成功');
        showDistributeModal.value = false;
        distributeFormRef.value.resetFields();
      } catch (error) {
        console.error('[5] 捕获错误', error);
        ElMessage.error('下发项目失败: ' + (error.message || '未知错误'));
      }
    };

    // 查看指导书
    const viewGuide = async (row) => {
      try {
        const res = await api.get(`/student/guide/${row.id}`);
        const g = res.data;
        const win = window.open('', '_blank');
        win.document.write(`
    <html>
      <head><meta charset="utf-8"/><title>${g.project_name}</title></head>
      <body style="font-family:Arial,Helvetica,sans-serif; margin:40px;">
        <h1>${g.project_name}</h1>
        <p>类型：${g.project_type} | 等级：${g.project_level} | 创建时间：${g.created_at}</p>
        <ol>
          ${g.steps.map(s => `
            <li>
              <strong>${s.title}</strong><br/>
              <pre style="white-space: pre-wrap; margin-top:4px;">${s.content}</pre>
            </li>`).join('')}
        </ol>
      </body>
    </html>`);
        win.document.close();
      } catch (e) {
        ElMessage.error('加载指导书失败：' + (e.response?.data?.error || e.message));
      }
    };

    return {
      projects,
      isAdmin,
      availableIPs,
      showCreateModal,
      editMode,
      currentProjectId,
      projectForm,
      projectFormRef,
      showDesignModal,
      guideForm,
      guideFormRef,
      showDistributeModal,
      distributeForm,
      distributeFormRef,
      rules,
      openCreateModal,
      submitProject,
      editProject,
      handleDeleteProject,  // ← 正确：返回你定义的函数
      openDesignModal,
      addStep,
      removeStep,
      saveGuide,
      openDistributeModal,
      distributeProject,
      viewGuide,
      enterExperiment,
      isProcessing,
    };
  },
};
</script>

<style scoped>
.experiment-projects {
  padding: 20px;
}
.step-container {
  margin-bottom: 20px;
  border-bottom: 1px solid #eee;
  padding-bottom: 10px;
}
.dialog-footer {
  display: flex;
  justify-content: flex-end;
  gap: 10px;
}
</style>