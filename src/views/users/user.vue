<template>
  <div class="app-container">
    <div class="filter-container">
      <el-input
        v-model="listQuery.username"
        placeholder="用户名"
        style="width: 200px;"
        class="filter-item"
        @keyup.enter.native="handleFilter"
      />
      <el-button
        v-waves
        class="filter-item"
        type="primary"
        icon="el-icon-search"
        @click="handleFilter"
      >
        搜索
      </el-button>
      <el-button
        class="filter-item"
        style="margin-left: 10px;"
        type="primary"
        icon="el-icon-edit"
        @click="handleCreate"
      >
        新增
      </el-button>
    </div>

    <el-table
      :key="tableKey"
      v-loading="listLoading"
      :data="list"
      border
      fit
      highlight-current-row
      style="width: 100%;"
      @sort-change="sortChange"
    >
      <el-table-column label="ID" prop="id" sortable="custom" align="center" width="80"
                       :class-name="getSortClass('id')">
        <template slot-scope="{row}">
          <span>{{ row.id }}</span>
        </template>
      </el-table-column>
      <el-table-column label="用户" min-width="50px" align="center">
        <template slot-scope="{row}">
          <span class="link-type">{{ row.username }}</span>
        </template>
      </el-table-column>
      <el-table-column label="邮箱" width="110px" align="center">
        <template slot-scope="{row}">
          <span>{{ row.email }}</span>
        </template>
      </el-table-column>
      <el-table-column label="角色" width="110px" align="center">
        <template slot-scope="{row}">
          <span>{{ row.roles }}</span>
        </template>
      </el-table-column>
      <el-table-column label="超级用户" width="110px" align="center">
        <template slot-scope="{row}">
          {{ row.is_superuser|statusChoice }}
        </template>
      </el-table-column>
      <el-table-column label="状态" width="110px" align="center">
        <template slot-scope="{row}">
          <el-tag :type="row.is_active | statusFilter">{{ row.is_active|statusChoice }}</el-tag>
          <!--          <el-tag :type="scope.row.is_active | statusFilter">{{ scope.row.is_active }}</el-tag>-->
        </template>
      </el-table-column>
      <el-table-column label="创建日期" width="110px" align="center">
        <template slot-scope="{row}">
          <i class="el-icon-time"/>
          <span>{{ row.create_date }}</span>
        </template>
      </el-table-column>

      <el-table-column label="操作" align="center" width="230" class-name="small-padding fixed-width">
        <template slot-scope="{row,$index}">
          <el-button type="primary" size="mini" @click="handleUpdate(row)">
            修改
          </el-button>
          <el-button size="mini" type="danger" @click="handleDelete(row,$index)">
            删除
          </el-button>
        </template>
      </el-table-column>
    </el-table>

    <pagination v-show="total>0" :total="total" :page.sync="listQuery.page" :limit.sync="listQuery.limit"
                @pagination="getList"/>
    <el-dialog :title="textMap[dialogStatus]" :visible.sync="dialogFormVisible">
      <el-form ref="dataForm" :rules="rules" :model="temp" label-position="left" label-width="70px"
               style="width: 400px; margin-left:50px;">
        <el-form-item label="用户" prop="username">
          <el-input v-model="temp.username"/>
        </el-form-item>
        <el-form-item label="姓名" prop="name">
          <el-input v-model="temp.name"/>
        </el-form-item>
        <el-form-item label="角色" prop="roles">
          <el-select ref="select" v-model="temp.roles" placeholder="请选择">
            <el-option v-for="item in this.roles" :key="item.id" :value="item.id" :label="item.name"/>
          </el-select>
        </el-form-item>
        <el-form-item label="电话" prop="mobile">
          <el-input v-model="temp.mobile"/>
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button @click="dialogFormVisible = false">
          取消
        </el-button>
        <el-button type="primary" @click="dialogStatus==='create'?createData():updateData()">
          确认
        </el-button>
      </div>
    </el-dialog>
  </div>
</template>

<script>
import {getUsersInfo, deleteUser, updateUser, updateUserRoles, updateUserState, addUser} from '@/api/user'
import waves from '@/directive/waves' // waves directive
import {getRoles} from '@/api/role'
import Pagination from '@/components/Pagination'


export default {
  name: 'ComplexTable',
  components: {Pagination},
  directives: {waves},
  filters: {
    statusFilter(status) {
      const statusMap = {
        'true': 'success',
        // draft: 'gray',
        'false': 'danger'
      }
      return statusMap[status]
    },
    statusChoice(status) {
      const valueMap = {
        'true': '是',
        'false': '否'
      }
      return valueMap[status]
    }
  },
  data() {
    return {
      tableKey: 0,
      roles: null,
      total: 0,
      listLoading: true,
      listQuery: {
        page: 1,
        size: 10,
        limit: this.size,
        level: undefined,
        sort: '+id'
      },
      sortOptions: [{label: 'ID 正序', key: '+id'}, {label: 'ID 逆序', key: '-id'}],
      showReviewer: false,
      iconOptions: [
        "404", "bug", "chart", "clipboard", "component", "dashboard", "documentation", "drag", "edit", "education", "email", "example", "excel", "exit-fullscreen", "eye-open", "eye", "form", "fullscreen", "guide", "icon", "international", "language", "link", "list", "lock", "message", "money", "nested", "password", "pdf", "people", "peoples", "qq", "search", "shopping", "size", "skill", "star", "tab", "table", "theme", "tree-table", "tree", "user", "wechat", "zip"
      ],
      temp: {
        id: undefined,
        username: '',
        name: '',
        mobile: '',
        roles: '',
        email: '',
        is_active: "",
        is_staff: 0
      },
      dialogFormVisible: false,
      dialogStatus: '',
      textMap: {
        update: '修改信息',
        create: '添加用户'
      },
      dialogPvVisible: false,
      pvData: [],
      rules: {
        username: [{required: true, message: '用户名称(账号)必须填写!', trigger: 'blur'}],
        name: [{required: true, message: '姓名名称必须填写！', trigger: 'blur'}],
        model: [{required: true, message: '资源必须填写！', trigger: 'blur'}],
        roles: [{required: true, message: '角色必须填写！', trigger: 'blur'}],
        mobile: [{required: true, message: '电话必须填写！', trigger: 'blur'}]
      },
      downloadLoading: false
    }
  },
  created() {
    this.getList()
    this.getRoles()
  },
  methods: {
    getRoles(){
      getRoles().then(response=>{
        this.roles = response.data
      })
    },
    getList() {
      this.listLoading = true
      this.listQuery.size = this.listQuery.limit
      // 重置选择上的空
      if (this.listQuery.level === '') {
        this.listQuery.level = undefined
      }
      getUsersInfo().then(response => {
        this.list = response.data
        this.total = response.total

        // Just to simulate the time of the request
        setTimeout(() => {
          this.listLoading = false
        }, 1.5 * 1000)
      })
    },
    handleFilter() {
      this.listQuery.page = 1
      this.getList()
    },
    handleModifyStatus(row, status) {
      this.$message({
        message: '操作Success',
        type: 'success'
      })
      row.status = status
    },
    sortChange(data) {
      const {prop, order} = data
      if (prop === 'id') {
        this.sortByID(order)
      }
    },
    sortByID(order) {
      if (order === 'ascending') {
        this.listQuery.sort = '+id'
      } else {
        this.listQuery.sort = '-id'
      }
      this.handleFilter()
    },
    resetTemp() {
      this.temp = {
        id: undefined,
        username: '',
        name: '',
        mobile: '',
        roles: '',
        email: '',
        is_active: '',
        is_staff: '',
        create_date: '',
        update_date: '',
        last_login: ''
      }
    },
    handleCreate() {
      this.resetTemp()
      this.dialogStatus = 'create'
      this.dialogFormVisible = true
      this.$nextTick(() => {
        this.$refs['dataForm'].clearValidate()
      })
    },
    createData() {
      this.$refs['dataForm'].validate((valid) => {
        if (valid) {
          this.temp.id = parseInt(Math.random() * 100) + 1024 // mock a id
          addUser(this.temp).then(() => {
            this.list.unshift(this.temp)
            this.dialogFormVisible = false
            this.$notify({
              title: 'Success',
              message: 'Created Successfully',
              type: 'success',
              duration: 2000
            })
          })
        }
      })
    },
    handleUpdate(row) {
      this.temp = Object.assign({}, row) // copy obj
      // this.temp.timestamp = new Date(this.temp.timestamp)
      this.dialogStatus = 'update'
      this.dialogFormVisible = true
      this.$nextTick(() => {
        this.$refs['dataForm'].clearValidate()
      })
    },
    updateData() {
      this.$refs['dataForm'].validate((valid) => {
        if (valid) {
          const tempData = Object.assign({}, this.temp)
          updateUser(tempData.id, tempData).then(() => {
            const index = this.list.findIndex(v => v.id === this.temp.id)
            this.list.splice(index, 1, this.temp)
            this.dialogFormVisible = false
            this.$notify({
              title: '修改权限成功：' + tempData.name,
              message: '修改权限成功',
              type: 'success',
              duration: 2000
            })
          })
        }
      })
    },
    handleDelete(row, index) {
      deletePermission(row.id).then(response => {
        const {meta} = response.data
        this.list.splice(index, 1)
        // this.total = response.data.total
        this.$notify({
          title: 'Success',
          message: meta.message,
          type: 'success',
          duration: 2000
        })

        // Just to simulate the time of the request
        setTimeout(() => {
          this.listLoading = false
        }, 1.5 * 1000)
      })
    },
    handleFetchPv(pv) {
      fetchPv(pv).then(response => {
        this.pvData = response.data.pvData
        this.dialogPvVisible = true
      })
    },
    getSortClass: function (key) {
      const sort = this.listQuery.sort
      return sort === `+${key}` ? 'ascending' : 'descending'
    }
  }
}
</script>
