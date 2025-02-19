<template>
  <div>
    <el-form :model="form">
      <el-form-item label="站点" :label-width="labelWidth">
        <el-select v-model="form.siteId">
          <el-option :label="site.name" :value="site.id" v-for="site of sites"/>
        </el-select>
      </el-form-item>
      <el-form-item label="索引名称" :label-width="labelWidth">
        <span>用于生成文件名，自动添加后缀.txt</span>
        <el-input v-model="form.indexName" autocomplete="off"/>
      </el-form-item>
      <el-form-item v-for="(item, index) in form.paths" :key="item.key" :label="'索引路径'+(index+1)"
                    :label-width="labelWidth">
        <el-input v-model="item.value">
          <template #append>
            <el-button class="mt-2" @click.prevent="removePath(item)">删除</el-button>
          </template>
        </el-input>
      </el-form-item>
      <el-form-item :label-width="labelWidth">
        <el-button @click="addPath">添加路径</el-button>
      </el-form-item>
      <el-form-item label="排除路径" :label-width="labelWidth">
        <span>支持多个路径，逗号分割</span>
        <el-input v-model="form.excludes" autocomplete="off" placeholder="逗号分割"/>
      </el-form-item>
      <el-form-item label="排除关键词" :label-width="labelWidth">
        <span>支持多个关键词，逗号分割</span>
        <el-input v-model="form.stopWords" autocomplete="off" placeholder="逗号分割"/>
      </el-form-item>
      <el-form-item label="排除外部AList站点？" :label-width="labelWidth">
        <el-switch v-model="form.excludeExternal"/>
      </el-form-item>
      <el-form-item label="压缩文件？" :label-width="labelWidth">
        <el-switch v-model="form.compress"/>
      </el-form-item>
      <el-form-item label="最大索引目录层级" :label-width="labelWidth">
        <el-input-number v-model="form.maxDepth" :min="1"/>
      </el-form-item>
      <el-form-item>
        <el-button type="primary" @click="handleform">开始索引</el-button>
      </el-form-item>
    </el-form>
    <div class="space"></div>

    <h2>任务列表</h2>
    <el-row justify="end">
      <el-button @click="loadTasks">刷新</el-button>
    </el-row>
    <el-table :data="tasks.content" border style="width: 100%">
      <el-table-column prop="id" label="ID" sortable width="70"/>
      <el-table-column prop="name" label="名称" sortable width="180"/>
      <el-table-column prop="status" label="状态" sortable width="120">
        <template #default="scope">
          <span>{{ getTaskStatus(scope.row.status) }}</span>
        </template>
      </el-table-column>
      <el-table-column prop="result" label="结果" sortable width="120">
        <template #default="scope">
          <span>{{ getTaskResult(scope.row.result) }}</span>
        </template>
      </el-table-column>
      <el-table-column prop="summary" label="概要"/>
      <el-table-column prop="error" label="错误"/>
      <el-table-column prop="startTime" label="开始时间" sortable width="150"/>
      <el-table-column prop="endTime" label="结束时间" sortable width="150"/>
      <el-table-column fixed="right" label="操作" width="140">
        <template #default="scope">
          <el-button link type="primary" size="small" @click="showDetails(scope.row)">数据</el-button>
          <el-button link type="danger" size="small" @click="handleCancel(scope.row)"
                     :disabled="scope.row.status==='COMPLETED'">取消
          </el-button>
          <el-button link type="danger" size="small" @click="handleDelete(scope.row)">删除</el-button>
        </template>
      </el-table-column>
    </el-table>
    <el-pagination layout="total, prev, pager, next" v-model:current-page="currentPage"
                   @current-change="handleCurrentChange" :total="total"/>

    <el-dialog v-model="dialogVisible" :title="task.name" width="30%">
      <p>{{ task.data }}</p>
      <template #footer>
      <span class="dialog-footer">
        <el-button @click="dialogVisible = false">关闭</el-button>
      </span>
      </template>
    </el-dialog>
  </div>
</template>

<script setup lang="ts">
import {onMounted, reactive, ref} from 'vue'
import axios from "axios";
import type {Site} from "@/model/Site";
import type {TaskPage} from "@/model/Page";
import type {Task} from "@/model/Task";
import {onUnmounted} from "@vue/runtime-core";

interface Item {
  key: number
  value: string
}

let intervalId = 0
const labelWidth = 160
const total = ref(0)
const currentPage = ref(1)
const dialogVisible = ref(false)
const sites = ref([] as Site[])
const tasks = ref({} as TaskPage)
const task = ref({} as Task)
const form = reactive({
  siteId: 0,
  indexName: 'index',
  excludeExternal: false,
  compress: false,
  maxDepth: 10,
  paths: [{
    key: 1,
    value: '/',
  }],
  stopWords: '',
  excludes: '',
})

const loadSites = () => {
  axios.get('/sites').then(({data}) => {
    sites.value = data
    if (sites.value && sites.value.length > 0) {
      form.siteId = sites.value[0].id
    }
  })
}

const loadTasks = () => {
  axios.get('/tasks?sort=id,desc&size=10&page=' + (currentPage.value - 1)).then(({data}) => {
    tasks.value = data
    total.value = data.totalElements
  })
}

const showDetails = (data: any) => {
  task.value = data
  dialogVisible.value = true
}

const handleCurrentChange = (data: number) => {
  currentPage.value = data
  loadTasks()
}

const handleCancel = (data: any) => {
  axios.post('/tasks/' + data.id + '/cancel').then(() => {
    loadTasks()
  })
}

const handleDelete = (data: any) => {
  axios.delete('/tasks/' + data.id).then(() => {
    loadTasks()
  })
}

const addPath = () => {
  form.paths.push({
    key: Date.now(),
    value: '',
  })
}

const removePath = (item: Item) => {
  if (form.paths.length < 2) {
    return
  }
  const index = form.paths.indexOf(item)
  if (index !== -1) {
    form.paths.splice(index, 1)
  }
}

const handleform = () => {
  const request = {
    siteId: form.siteId,
    indexName: form.indexName,
    excludeExternal: form.excludeExternal,
    compress: form.compress,
    maxDepth: form.maxDepth,
    paths: form.paths.map(e => e.value).filter(e => e),
    stopWords: form.stopWords ? form.stopWords.split(/\s*,\s*/) : [],
    excludes: form.excludes ? form.excludes.split(/\s*,\s*/) : [],
  }
  axios.post('/index', request).then(() => {
    currentPage.value = 1
    loadTasks()
  })
}

const getTaskStatus = (status: string) => {
  if (status === 'READY') {
    return '就绪'
  }
  if (status === 'COMPLETED') {
    return '结束'
  }
  if (status === 'RUNNING') {
    return '运行'
  }
  return ''
}

const getTaskResult = (result: string) => {
  if (result === 'OK') {
    return '成功'
  }
  if (result === 'FAILED') {
    return '失败'
  }
  if (result === 'CANCELLED') {
    return '取消'
  }
  return ''
}

onMounted(() => {
  loadSites()
  loadTasks()
  intervalId = setInterval(loadTasks, 30000)
})

onUnmounted(() => {
  clearInterval(intervalId)
})
</script>

<style scoped>
.space {
  margin-bottom: 6px;
}
</style>
