<template>
  <div class="app-container">
    <!--工具栏-->
    <div class="head-container">
      <div style="margin-bottom: 20px;">
        <div>
          <el-button @click="setViewerRole">访问者端</el-button>
          <el-button @click="setEditorRole">用户编辑端</el-button>
          <el-button @click="setAdminRole">管理员端</el-button>

          <!-- Conditionally render el-descriptions based on user permissions -->
          <div v-if="checkPer(['admin', 'viewer'])" style="margin-top: 20px;">
            <el-descriptions title="用户信息" direction="vertical" :column="2 " border>
              <el-descriptions-item label="用户名">测试员1</el-descriptions-item>
              <el-descriptions-item label="手机号">18100000000</el-descriptions-item>
              <el-descriptions-item label="居住地" :span="2">苏州市</el-descriptions-item>
              <el-descriptions-item label="备注">
                <el-tag size="small">学校</el-tag>
                <el-tag size="small" color="pink" style="margin-left: 10px;">公园</el-tag>
              </el-descriptions-item>
              <el-descriptions-item label="联系地址">地址</el-descriptions-item>
            </el-descriptions>
          </div>
          <div v-else>
            <p>您没有权限查看此信息。</p>
          </div>
        </div>
      </div>
      <!-- <el-collapse v-model="activeName" accordion>
        <el-collapse-item title="一致性 Consistency" name="1">
          <div>与现实生活一致：与现实生活的流程、逻辑保持一致，遵循用户习惯的语言和概念；</div>
          <div>在界面中一致：所有的元素和结构需保持一致，比如：设计样式、图标和文本、元素的位置等。</div>
        </el-collapse-item>
        <el-collapse-item title="反馈 Feedback" name="2">
          <div>控制反馈：通过界面样式和交互动效让用户可以清晰的感知自己的操作；</div>
          <div>页面反馈：操作后，通过页面元素的变化清晰地展现当前状态。</div>
        </el-collapse-item>
        <el-collapse-item title="效率 Efficiency" name="3">
          <div>简化流程：设计简洁直观的操作流程；</div>
          <div>清晰明确：语言表达清晰且表意明确，让用户快速理解进而作出决策；</div>
          <div>帮助用户识别：界面简单直白，让用户快速识别而非回忆，减少用户记忆负担。</div>
        </el-collapse-item>
        <el-collapse-item title="可控 Controllability" name="4">
          <div>用户决策：根据场景可给予用户操作建议或安全提示，但不能代替用户进行决策；</div>
          <div>结果可控：用户可以自由的进行操作，包括撤销、回退和终止当前操作等。</div>
        </el-collapse-item>
      </el-collapse> -->
      <div style="margin: 20px;" />
      <div v-if="crud.props.searchToggle">
        <!-- 搜索 -->
        <el-input v-model="query.name" clearable size="small" placeholder="输入部门名称搜索" style="width: 200px;" class="filter-item" @keyup.enter.native="crud.toQuery" />
        <!-- <date-range-picker v-model="query.createTime" class="date-item" /> -->
        <el-select v-model="query.enabled" clearable size="small" placeholder="状态" class="filter-item" style="width: 90px" @change="crud.toQuery">
          <el-option v-for="item in enabledTypeOptions" :key="item.key" :label="item.display_name" :value="item.key" />
        </el-select>
        <rrOperation />
      </div>
      <crudOperation :permission="permission" />
    </div>
    <!--表单组件-->
    <el-dialog append-to-body :close-on-click-modal="false" :before-close="crud.cancelCU" :visible.sync="crud.status.cu > 0" :title="crud.status.title" width="500px">
      <el-form ref="form" inline :model="form" :rules="rules" size="small" label-width="80px">
        <el-form-item label="部门名称" prop="name">
          <el-input v-model="form.name" style="width: 370px;" />
        </el-form-item>
        <el-form-item label="部门排序" prop="deptSort">
          <el-input-number
            v-model.number="form.deptSort"
            :min="0"
            :max="999"
            controls-position="right"
            style="width: 370px;"
          />
        </el-form-item>
        <el-form-item label="顶级部门">
          <el-radio-group v-model="form.isTop" style="width: 140px">
            <el-radio label="1">是</el-radio>
            <el-radio label="0">否</el-radio>
          </el-radio-group>
        </el-form-item>
        <el-form-item label="状态" prop="enabled">
          <el-radio v-for="item in dict.dept_status" :key="item.id" v-model="form.enabled" :label="item.value">{{ item.label }}</el-radio>
        </el-form-item>
        <el-form-item v-if="form.isTop === '0'" style="margin-bottom: 0;" label="上级部门" prop="pid">
          <treeselect
            v-model="form.pid"
            :load-options="loadDepts"
            :options="depts"
            style="width: 370px;"
            placeholder="选择上级类目"
          />
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button type="text" @click="crud.cancelCU">取消</el-button>
        <el-button :loading="crud.status.cu === 2" type="primary" @click="crud.submitCU">确认</el-button>
      </div>
    </el-dialog>
    <!--表格渲染-->
    <el-table
      ref="table"
      v-loading="crud.loading"
      lazy
      :load="getDeptDatas"
      :tree-props="{children: 'children', hasChildren: 'hasChildren'}"
      :data="crud.data"
      row-key="id"
      @select="crud.selectChange"
      @select-all="crud.selectAllChange"
      @selection-change="crud.selectionChangeHandler"
    >
      <el-table-column :selectable="checkboxT" type="selection" width="55" />
      <el-table-column label="名称" prop="name" />
      <el-table-column label="排序" prop="deptSort" />
      <el-table-column label="状态" align="center" prop="enabled">
        <template slot-scope="scope">
          <el-switch
            v-model="scope.row.enabled"
            :disabled="scope.row.id === 1"
            active-color="#409EFF"
            inactive-color="#F56C6C"
            @change="changeEnabled(scope.row, scope.row.enabled,)"
          />
        </template>
      </el-table-column>
      <el-table-column prop="createTime" label="创建日期" />
      <el-table-column v-if="checkPer(['admin','dept:edit','dept:del'])" label="操作" width="130px" align="center" fixed="right">
        <template slot-scope="scope">
          <udOperation
            :data="scope.row"
            :permission="permission"
            :disabled-dle="scope.row.id === 1"
            msg="确定删除吗,如果存在下级节点则一并删除，此操作不能撤销！"
          />
        </template>
      </el-table-column>
    </el-table>
  </div>
</template>

<script>
import crudDept from '@/api/system/dept'
import Treeselect from '@riophae/vue-treeselect'
import '@riophae/vue-treeselect/dist/vue-treeselect.css'
import { LOAD_CHILDREN_OPTIONS } from '@riophae/vue-treeselect'
import CRUD, { presenter, header, form, crud } from '@crud/crud'
import rrOperation from '@crud/RR.operation'
import crudOperation from '@crud/CRUD.operation'
import udOperation from '@crud/UD.operation'
// import DateRangePicker from '@/components/DateRangePicker'

const defaultForm = { id: null, name: null, isTop: '1', subCount: 0, pid: null, deptSort: 999, enabled: 'true' }
export default {
  name: 'Dept',
  components: { Treeselect, crudOperation, rrOperation, udOperation },
  cruds() {
    return CRUD({ title: '部门', url: 'api/dept', crudMethod: { ...crudDept }})
  },
  mixins: [presenter(), header(), form(defaultForm), crud()],
  // 设置数据字典
  dicts: ['dept_status'],
  data() {
    return {
      activeName: '1',
      depts: [],
      rules: {
        name: [
          { required: true, message: '请输入名称', trigger: 'blur' }
        ],
        deptSort: [
          { required: true, message: '请输入序号', trigger: 'blur', type: 'number' }
        ]
      },
      permission: {
        add: ['admin', 'dept:add'],
        edit: ['admin', 'dept:edit'],
        del: ['admin', 'dept:del']
      },
      enabledTypeOptions: [
        { key: 'true', display_name: '正常' },
        { key: 'false', display_name: '禁用' }
      ]
    }
  },
  methods: {
    getDeptDatas(tree, treeNode, resolve) {
      const params = { pid: tree.id }
      setTimeout(() => {
        crudDept.getDepts(params).then(res => {
          resolve(res.content)
        })
      }, 100)
    },
    // 新增与编辑前做的操作
    [CRUD.HOOK.afterToCU](crud, form) {
      if (form.pid !== null) {
        form.isTop = '0'
      } else if (form.id !== null) {
        form.isTop = '1'
      }
      form.enabled = `${form.enabled}`
      if (form.id != null) {
        this.getSupDepts(form.id)
      } else {
        this.getDepts()
      }
    },
    getSupDepts(id) {
      crudDept.getDeptSuperior(id, true).then(res => {
        const data = res.content
        this.buildDepts(data)
        this.depts = data
      })
    },
    buildDepts(depts) {
      depts.forEach(data => {
        if (data.children) {
          this.buildDepts(data.children)
        }
        if (data.hasChildren && !data.children) {
          data.children = null
        }
      })
    },
    getDepts() {
      crudDept.getDepts({ enabled: true }).then(res => {
        this.depts = res.content.map(function(obj) {
          if (obj.hasChildren) {
            obj.children = null
          }
          return obj
        })
      })
    },
    // 获取弹窗内部门数据
    loadDepts({ action, parentNode, callback }) {
      console.log('loadDepts', action, parentNode, callback)
      if (action === LOAD_CHILDREN_OPTIONS) {
        crudDept.getDepts({ enabled: true, pid: parentNode.id }).then(res => {
          parentNode.children = res.content.map(function(obj) {
            if (obj.hasChildren) {
              obj.children = null
            }
            return obj
          })
          setTimeout(() => {
            callback()
          }, 100)
        })
      }
    },
    // 提交前的验证
    [CRUD.HOOK.afterValidateCU]() {
      if (this.form.pid !== null && this.form.pid === this.form.id) {
        this.$message({
          message: '上级部门不能为空',
          type: 'warning'
        })
        return false
      }
      if (this.form.isTop === '1') {
        this.form.pid = null
      }
      return true
    },
    // 改变状态
    changeEnabled(data, val) {
      this.$confirm('此操作将 "' + this.dict.label.dept_status[val] + '" ' + data.name + '部门, 是否继续？', '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        crudDept.edit(data).then(res => {
          this.crud.notify(this.dict.label.dept_status[val] + '成功', CRUD.NOTIFICATION_TYPE.SUCCESS)
        }).catch(err => {
          data.enabled = !data.enabled
          console.log(err.response.data.message)
        })
      }).catch(() => {
        data.enabled = !data.enabled
      })
    },
    checkboxT(row, rowIndex) {
      return row.id !== 1
    },
    setViewerRole() {
      this.$store.dispatch('toggleRoles', ['viewer'])
    },
    setEditorRole() {
      this.$store.dispatch('toggleRoles', ['editor'])
    },
    setAdminRole() {
      this.$store.dispatch('toggleRoles', ['admin'])
    }
  }
}
</script>

<style rel="stylesheet/scss" lang="scss" scoped>
 ::v-deep .vue-treeselect__control,::v-deep .vue-treeselect__placeholder,::v-deep .vue-treeselect__single-value {
    height: 30px;
    line-height: 30px;
  }
</style>
<style rel="stylesheet/scss" lang="scss" scoped>
 ::v-deep .el-input-number .el-input__inner {
    text-align: left;
  }
</style>
