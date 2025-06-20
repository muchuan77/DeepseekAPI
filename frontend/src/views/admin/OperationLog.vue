<template>
  <div class="operation-log">
    <el-card>
      <template #header>
        <div class="card-header">
          <span>操作日志</span>
        </div>
      </template>

      <el-form
        :inline="true"
        :model="searchForm"
        class="search-form"
      >
        <el-form-item label="操作类型">
          <el-select
            v-model="searchForm.operationType"
            placeholder="请选择操作类型"
          >
            <el-option
              label="全部"
              value=""
            />
            <el-option
              label="登录"
              value="LOGIN"
            />
            <el-option
              label="登出"
              value="LOGOUT"
            />
            <el-option
              label="创建"
              value="CREATE"
            />
            <el-option
              label="更新"
              value="UPDATE"
            />
            <el-option
              label="删除"
              value="DELETE"
            />
          </el-select>
        </el-form-item>
        <el-form-item label="用户名">
          <el-input
            v-model="searchForm.username"
            placeholder="请输入用户名"
          />
        </el-form-item>
        <el-form-item label="时间范围">
          <el-date-picker
            v-model="searchForm.dateRange"
            type="daterange"
            range-separator="至"
            start-placeholder="开始日期"
            end-placeholder="结束日期"
            value-format="YYYY-MM-DD"
          />
        </el-form-item>
        <el-form-item>
          <el-button
            type="primary"
            @click="handleSearch"
          >
            搜索
          </el-button>
          <el-button @click="resetSearch">
            重置
          </el-button>
        </el-form-item>
      </el-form>

      <el-table
        v-loading="loading"
        :data="logs"
        style="width: 100%"
      >
        <el-table-column
          prop="timestamp"
          label="时间"
          width="180"
        >
          <template #default="{ row }">
            {{ formatDate(row.timestamp) }}
          </template>
        </el-table-column>
        <el-table-column
          prop="username"
          label="用户名"
          width="120"
        />
        <el-table-column
          prop="operationType"
          label="操作类型"
          width="100"
        >
          <template #default="{ row }">
            <el-tag :type="getOperationType(row.operationType)">
              {{ getOperationTypeText(row.operationType) }}
            </el-tag>
          </template>
        </el-table-column>
        <el-table-column
          prop="module"
          label="操作模块"
          width="120"
        />
        <el-table-column
          prop="description"
          label="操作描述"
        />
        <el-table-column
          prop="ip"
          label="IP地址"
          width="150"
        />
        <el-table-column
          prop="status"
          label="状态"
          width="100"
        >
          <template #default="{ row }">
            <el-tag :type="row.status === 'SUCCESS' ? 'success' : 'danger'">
              {{ row.status === 'SUCCESS' ? '成功' : '失败' }}
            </el-tag>
          </template>
        </el-table-column>
      </el-table>

      <div class="pagination">
        <el-pagination
          v-model:current-page="currentPage"
          v-model:page-size="pageSize"
          :page-sizes="[10, 20, 50, 100]"
          :total="total"
          layout="total, sizes, prev, pager, next"
          @size-change="handleSizeChange"
          @current-change="handleCurrentChange"
        />
      </div>
    </el-card>
  </div>
</template>

<script>
import { ref, reactive, onMounted, computed } from 'vue'
import { ElMessage } from 'element-plus'
import { useLogStore } from '@/stores/log'
import { format } from 'date-fns'

export default {
  name: 'OperationLog',
  setup() {
    const logStore = useLogStore()
    const loading = ref(false)
    const currentPage = ref(1)
    const pageSize = ref(10)
    const total = ref(0)

    const searchForm = reactive({
      operationType: '',
      username: '',
      dateRange: []
    })

    const fetchLogs = async () => {
      loading.value = true
      try {
        let startTime = null
        let endTime = null
        if (searchForm.dateRange && searchForm.dateRange.length === 2) {
          startTime = searchForm.dateRange[0] ? `${searchForm.dateRange[0]}T00:00:00.000Z` : null
          endTime = searchForm.dateRange[1] ? `${searchForm.dateRange[1]}T23:59:59.999Z` : null
        }

        const params = {
          page: currentPage.value - 1,
          size: pageSize.value,
          operationType: searchForm.operationType || null,
          username: searchForm.username || null,
          startTime,
          endTime
        }

        // 移除空值参数
        Object.keys(params).forEach(key => {
          if (params[key] === null || params[key] === undefined || params[key] === '') {
            delete params[key]
          }
        })

        const result = await logStore.fetchOperationLogs(params)
        total.value = result.totalElements || 0
        
        if (logStore.operationLogs.length === 0) {
          ElMessage.info('暂无操作日志数据')
        }
      } catch (error) {
        if (error.response) {
          const status = error.response.status
          switch (status) {
            case 400:
              ElMessage.error('请求参数错误，请检查输入')
              break
            case 401:
              ElMessage.error('未授权，请重新登录')
              break
            case 403:
              ElMessage.error('没有权限访问此资源')
              break
            case 404:
              ElMessage.info('暂无操作日志数据')
              logStore.operationLogs = []
              total.value = 0
              break
            case 500:
              ElMessage.error('服务器内部错误，请稍后重试')
              break
            default:
              ElMessage.error('获取操作日志失败')
          }
          console.error(`请求失败 (${status}):`, error)
        } else if (error.request) {
          ElMessage.error('网络连接失败，请检查网络设置')
          console.error('网络错误:', error)
        } else {
          ElMessage.error('获取操作日志失败')
          console.error('请求错误:', error)
        }
      } finally {
        loading.value = false
      }
    }

    const handleSearch = () => {
      currentPage.value = 1
      fetchLogs()
    }

    const resetSearch = () => {
      searchForm.operationType = ''
      searchForm.username = ''
      searchForm.dateRange = []
      handleSearch()
    }

    const handleSizeChange = (val) => {
      pageSize.value = val
      fetchLogs()
    }

    const handleCurrentChange = (val) => {
      currentPage.value = val
      fetchLogs()
    }

    const formatDate = (timestamp) => {
      return format(new Date(timestamp), 'yyyy-MM-dd HH:mm:ss')
    }

    const getOperationType = (type) => {
      switch (type) {
        case 'LOGIN':
          return 'success'
        case 'LOGOUT':
          return 'info'
        case 'CREATE':
          return 'primary'
        case 'UPDATE':
          return 'warning'
        case 'DELETE':
          return 'danger'
        default:
          return ''
      }
    }

    const getOperationTypeText = (type) => {
      switch (type) {
        case 'LOGIN':
          return '登录'
        case 'LOGOUT':
          return '登出'
        case 'CREATE':
          return '创建'
        case 'UPDATE':
          return '更新'
        case 'DELETE':
          return '删除'
        default:
          return type
      }
    }

    onMounted(() => {
      fetchLogs()
    })

    return {
      loading,
      logs: computed(() => logStore.operationLogs),
      currentPage,
      pageSize,
      total,
      searchForm,
      handleSearch,
      resetSearch,
      handleSizeChange,
      handleCurrentChange,
      formatDate,
      getOperationType,
      getOperationTypeText
    }
  }
}
</script>

<style scoped>
.operation-log {
  padding: 20px;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.search-form {
  margin-bottom: 20px;
}

.pagination {
  margin-top: 20px;
  display: flex;
  justify-content: flex-end;
}

/* 添加下拉框样式 */
:deep(.el-select) {
  width: 200px;
}

:deep(.el-select-dropdown__item) {
  white-space: normal;
  height: auto;
  padding: 8px 10px;
  line-height: 1.5;
}

:deep(.el-select-dropdown__item span) {
  display: inline-block;
  width: 100%;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

:deep(.el-select-dropdown__item.selected) {
  color: #409EFF;
  font-weight: bold;
}
</style> 