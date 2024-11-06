<template>
  <div class="app-container">
    <div class="flex flex-wrap flex-between mb10">
      <div>
        <el-form ref="queryForm" :inline="true" :model="queryParams" size="small">
          <el-form-item label="" prop="type">
            <el-select v-model="queryParams.type" clearable placeholder="服务器类型">
              <el-option v-for="dict in dict.type.server_type" :key="dict.value" :label="dict.label"
                :value="Number(dict.value)"></el-option>
            </el-select>
          </el-form-item>
          <el-form-item label="" prop="name">
            <el-input v-model="queryParams.name" clearable maxlength="200" placeholder="请输入你要搜索的服务器名称或 IP 地址"
              style="width: 290px;" oninput="value = value.trim()" @keyup.enter.native="handleQuery" />
          </el-form-item>
          <el-form-item>
            <el-button icon="el-icon-search" size="mini" type="primary" @click="handleQuery">搜索</el-button>
          </el-form-item>
        </el-form>
      </div>
      <div>
        <el-button size="mini" type="warning" @click="refresh">刷新
        </el-button>
        <el-button v-has-permi="['pt:testserver:add']" icon="el-icon-plus" size="mini" type="primary"
          @click="handleAdd">新建服务器
        </el-button>
      </div>
    </div>

    <el-table v-loading="loading" :data="dataList">
      <!--<el-table-column label="ID" align="center" prop="serverid"/>-->
      <el-table-column align="left" label="服务器名称" prop="name" show-overflow-tooltip width="400" />
      <el-table-column align="left" label="用户名" prop="username" width="130" />
      <el-table-column align="left" label="IP地址" prop="ip" width="160" />
      <el-table-column align="left" label="服务器类型" prop="type" width="120">
        <template slot-scope="scope">
          <dict-tag :options="dict.type.server_type" :value="scope.row.type" />
        </template>
      </el-table-column>
      <el-table-column align="left" label="关键进程名称" min-width="100" prop="keyProcessName" show-overflow-tooltip />
      <el-table-column align="center" class-name="small-padding fixed-width" fixed="right" label="操作" width="260">
        <template slot-scope="scope">
          <el-button v-has-permi="['pt:testserver:edit']" icon="el-icon-edit" size="mini" type="text"
            @click="handleUpdate(scope.row)">编辑
          </el-button>
          <el-button v-has-permi="['pt:testserver:remove']" icon="el-icon-delete" size="mini" type="text"
            @click="handleDelete(scope.row)">删除
          </el-button>
          <el-button v-hasPermi="['pt:testserver:test']" icon="el-icon-document-copy" size="mini" type="text"
            @click="handleTest(scope.row)">联通性测试
          </el-button>
        </template>
      </el-table-column>
    </el-table>

    <pagination v-show="total > 0" :limit.sync="queryParams.pageSize" :page.sync="queryParams.pageNum" :total="total"
      @pagination="getList" />


    <!-- 添加或修改服务器对话框 -->
    <my-popup :title="title" :visible.sync="open" destroy-on-close type="dialog" width="700px"
      @handleClose="handleClose">
      <div style="position: relative">
        <el-button :disabled="loadingConnectivity" :icon="loadingConnectivity ? 'el-icon-loading' : ''"
          class="connectivity-btn" size="mini" type="primary" @click="handleConnectivityTest">{{ loadingConnectivity ?
            '连通测试中...' : '连通性测试' }}
        </el-button>
      </div>
      <el-form ref="form" :model="form" :rules="isedit ? rulesedit : rules" label-position="top">
        <el-form-item label="服务器名称" prop="name">
          <el-input oninput="value = value.replace(/\s/g, '')" maxlength="50" v-model="form.name" placeholder="请输入服务器名称"
            type="text" />
        </el-form-item>
        <el-row :gutter="20" :span="15" :xs="24">
          <el-col :span="12">
            <el-form-item label="服务器类型" prop="type">
              <el-select v-model="form.type" :disabled="form.serverid != null" clearable placeholder="请选择服务器类型">
                <el-option v-for="dict in dict.type.server_type" :key="dict.value" :label="dict.label"
                  :value="Number(dict.value)"></el-option>
              </el-select>
            </el-form-item>
          </el-col>
          <el-col :span="12">
            <el-form-item label="用户名" prop="username">
              <el-input oninput="value = value.replace(/\s/g, '')" maxlength="50" v-model.trim="form.username"
                placeholder="请输入用户名" type="text" />
            </el-form-item>
          </el-col>
        </el-row>
        <el-row :gutter="20" :span="15" :xs="24">
          <el-col :span="12">
            <el-form-item label="IP地址" prop="ip">
              <el-input maxlength="16" oninput="value = value.replace(/\s/g, '')" v-model="form.ip"
                placeholder="请输入IP地址" type="text" />
            </el-form-item>
          </el-col>
          <el-col :span="12">
            <el-form-item label="SSH 密码" prop="sshPassword">
              <el-input maxlength="50" oninput="value = value.replace(/\s/g, '')" v-model.trim="form.sshPassword"
                placeholder="请输入SSH 密码" type="text" />
            </el-form-item>
          </el-col>
        </el-row>
        <el-row :gutter="20" :span="15" :xs="24">
          <el-col :span="12">
            <el-form-item label="SSH 端口" prop="sshPort">
              <el-input oninput="value = value.replace(/\s/g, '')" v-model.number="form.sshPort" class="no-number"
                maxlength="6" placeholder="请输入SSH 端口" />
            </el-form-item>
          </el-col>
          <el-col :span="12">
            <el-form-item label="Trader_port" prop="traderPo">
              <el-input oninput="value = value.replace(/\s/g, '')" v-model.number="form.traderPo" class="no-number"
                maxlength="5" placeholder="请输入traderPo" />
            </el-form-item>
          </el-col>
        </el-row>
        <el-row :gutter="20" :span="15" :xs="24">
          <el-col :span="12">
            <el-form-item label="业务端口" prop="businessPort">
              <el-input oninput="value = value.replace(/\s/g, '')" v-model.number="form.businessPort" class="no-number"
                maxlength="5" placeholder="请输入业务端口" />
            </el-form-item>
          </el-col>
        </el-row>
        <!---BUGUD 003487 数据库链接字符串原来做的空格过滤 oninput="value = value.replace(/\s/g, '')" 现在去掉 -->
        <el-form-item label="链接字符串" prop="connectionString">
          <el-input maxlength="250" v-model="form.connectionString" placeholder="请输入链接字符串" type="textarea" />

          <el-row :gutter="0" :span="24">
            <el-col :span="4">
              当交易前置时：
            </el-col>
            <el-col :span="10">
              交易控制台用户|密码<br />
              技术控制台用户|密码
            </el-col>
            <el-col :span="2">
              其他：
            </el-col>
            <el-col :span="7">采用链接字符串</el-col>
          </el-row>
        </el-form-item>
        <el-form-item label="关键进程名称" prop="keyProcessName">
          <el-input maxlength="1000" v-model="form.keyProcessName" placeholder="请输入关键进程名称" type="textarea" />
        </el-form-item>
        <el-form-item label="日志地址配置" prop="logAddressConfig">
          <el-input maxlength="1000" v-model="form.logAddressConfig" placeholder="多个地址用 | 分割" type="textarea" />
        </el-form-item>
      </el-form>
      <span slot="footer">
        <el-button @click="cancel">取 消</el-button>
        <el-button type="primary" @click="submitForm">确 定</el-button>
      </span>
    </my-popup>
  </div>
</template>
<script>
import {
  addTestServer,
  connectTestServer,
  delTestServer,
  getTestServer,
  listTestServer,
  updateTestServer,
  nameoriplist,
  connectTestServerA
} from '@/api/pt/testserver'
import { isIp } from '@/utils/validate'
export default {
  name: 'testServer',
  dicts: ['server_type'],
  data() {
    const validatorPositive = (rule, value, callback) => {
      if (value) {
        if (!Number.isInteger(Number(value))) {
          return callback(new Error('必须正整数'))
        }
      }
      return callback()
    }
    const validatorTraderPo = (rule, value, callback) => {
      if (value) {
        if (!Number.isInteger(Number(value))) {
          return callback(new Error('Trader_port必须是数字类型'))
        }
      }
      return callback()
    }
    const validatorSshPort = (rule, value, callback) => {
      if (value) {
        if (!Number.isInteger(Number(value))) {
          return callback(new Error('SSH端口必须是数字类型'))
        }
      }
      return callback()
    }
    const validatorBusinessPort = (rule, value, callback) => {
      if (value) {
        if (!Number.isInteger(Number(value))) {
          return callback(new Error('业务端口必须是数字类型'))
        }
      }
      return callback()
    }
    return {
      // 遮罩层
      loading: false,
      loadingConnectivity: false,
      // 总条数
      total: 0,
      // 场景表格数据
      dataList: [],
      // 弹出层标题
      title: '',
      // 是否显示弹出层
      open: false,
      //联通测试按钮默认显示
      showbutton: true,
      // 查询参数
      queryParams: {
        pageNum: 1,
        pageSize: 10,
        type: null,
        name: null
      },
      isedit: false,
      // 表单参数
      form: {},
      // 表单校验
      rules: {
        name: [
          { required: true, message: '服务器名称不能为空', trigger: ['change', 'blur'] },
          { max: 200, message: '服务器名称长度不能大于 200 个字符', trigger: ['change', 'blur'] }
        ],
        type: [
          { required: true, message: '服务器类型不能为空', trigger: ['change', 'blur'] }
        ],
        username: [
          { required: true, message: '用户名不能为空', trigger: ['change', 'blur'] },
          { max: 50, message: '用户名长度不能大于 50 个字符', trigger: ['change', 'blur'] }
        ],
        sshPassword: [
          { required: true, message: '密码必填', trigger: ['change', 'blur'] },
        ],
        ip: [
          { required: true, message: 'IP地址不能为空', trigger: ['change', 'blur'] },
          { max: 20, message: 'IP地址长度不能大于 20 个字符', trigger: ['change', 'blur'] },
          {
            validator: (rule, value, callback) => {
              if (!isIp(value)) {
                callback(new Error('请输入正确的ip地址'))
              } else {
                callback()
              }
            }, trigger: ['change', 'blur']
          }
        ],
        traderPo: [
          { validator: validatorTraderPo, trigger: ['change', 'blur'] },
          { type: 'number', message: '输入必须是整数', trigger: ['change', 'blur'] },
          { validator: validatorPositive, trigger: ['change', 'blur'] }
        ],
        sshPort: [
          { required: true, message: 'SSH端口不能为空', trigger: ['change', 'blur'] },
          { validator: validatorSshPort, trigger: ['change', 'blur'] },
          { type: 'number', message: '输入必须是整数', trigger: ['change', 'blur'] },
          { validator: validatorPositive, trigger: ['change', 'blur'] }
        ],
        businessPort: [
          { validator: validatorBusinessPort, trigger: ['change', 'blur'] },
          { type: 'number', message: '输入必须是整数', trigger: ['change', 'blur'] },
          { validator: validatorPositive, trigger: ['change', 'blur'] }
        ],
        connectionString: [
          // { required: true, message: '链接字符串不能为空', trigger: 'blur' },
          { max: 250, message: '链接字符串长度不能大于 250 个字符', trigger: 'change' }
        ],
        keyProcessName: [
          // { required: true, message: '关键进程名称不能为空', trigger: 'blur' },
          { max: 1000, message: '关键进程名称长度不能大于 1000 个字符', trigger: 'change' }
        ],
        logAddressConfig: [
          // { required: true, message: '日志地址配置不能为空', trigger: 'blur' },
          { max: 1000, message: '日志地址配置长度不能大于 1000 个字符', trigger: 'change' }
        ]
      },
      ruleseidt: {
        name: [
          { required: true, message: '服务器名称不能为空', trigger: ['change', 'blur'] },
          { max: 200, message: '服务器名称长度不能大于 200 个字符', trigger: ['change', 'blur'] }
        ],
        type: [
          { required: true, message: '服务器类型不能为空', trigger: ['change', 'blur'] }
        ],
        username: [
          { required: true, message: '用户名不能为空', trigger: ['change', 'blur'] },
          { max: 50, message: '用户名长度不能大于 50 个字符', trigger: ['change', 'blur'] }
        ],
        ip: [
          { required: true, message: 'IP地址不能为空', trigger: ['change', 'blur'] },
          { max: 20, message: 'IP地址长度不能大于 20 个字符', trigger: ['change', 'blur'] },
          {
            validator: (rule, value, callback) => {
              if (!isIp(value)) {
                callback(new Error('请输入正确的ip地址'))
              } else {
                callback()
              }
            }, trigger: ['change', 'blur']
          }
        ],
        traderPo: [
          { validator: validatorTraderPo, trigger: ['change', 'blur'] },
          { type: 'number', message: '输入必须是整数', trigger: ['change', 'blur'] },
          { validator: validatorPositive, trigger: ['change', 'blur'] }
        ],
        sshPort: [
          { required: true, message: 'SSH端口不能为空', trigger: ['change', 'blur'] },
          { validator: validatorSshPort, trigger: ['change', 'blur'] },
          { type: 'number', message: '输入必须是整数', trigger: ['change', 'blur'] },
          { validator: validatorPositive, trigger: ['change', 'blur'] }
        ],
        businessPort: [
          { validator: validatorBusinessPort, trigger: ['change', 'blur'] },
          { type: 'number', message: '输入必须是整数', trigger: ['change', 'blur'] },
          { validator: validatorPositive, trigger: ['change', 'blur'] }
        ],
        connectionString: [
          // { required: true, message: '链接字符串不能为空', trigger: 'blur' },
          { max: 250, message: '链接字符串长度不能大于 250 个字符', trigger: 'change' }
        ],
        keyProcessName: [
          // { required: true, message: '关键进程名称不能为空', trigger: 'blur' },
          { max: 1000, message: '关键进程名称长度不能大于 1000 个字符', trigger: 'change' }
        ],
        logAddressConfig: [
          // { required: true, message: '日志地址配置不能为空', trigger: 'blur' },
          { max: 1000, message: '日志地址配置长度不能大于 1000 个字符', trigger: 'change' }
        ]
      }
    }
  },
  created() {
    this.getList()
  },
  methods: {
    /** 查询列表 */
    /*BUGID 003486 原来的查询接口更改， 查询服务器列表接口变更为nameoriplist */
    getList() {
      this.loading = true
      let query = this.queryParams
      query.ip = query.name
      nameoriplist(query).then(response => {
        this.dataList = response.rows
        this.total = response.total
        this.loading = false
      })
    },
    /** 搜索按钮操作 */
    handleQuery() {
      this.queryParams.pageNum = 1
      this.getList()
    },
    // 刷新
    refresh() {
      this.resetForm('queryForm')
      this.handleQuery()
    },
    /** 新增执行机组 */
    handleAdd() {
      this.reset()
      this.title = '新建服务器'
      this.open = true;
      this.showbutton = true;
      this.isedit = false;
    },
    /** 修改执行机组 */
    handleUpdate(row) {
      this.reset()
      this.isedit = true;
      getTestServer(row.serverid).then(response => {
        this.form = response.data
        this.open = true
        this.title = '修改服务器'
        this.showbutton = false;
      })
    },
    /*关闭抽屉*/
    handleClose() {
      this.cancel()
    },
    // 取消按钮
    cancel() {
      this.reset()
      this.open = false;
      this.loadingConnectivity = false
    },
    // 表单重置
    reset() {
      this.form = {
        serverid: null,
        name: null,
        type: null,
        username: null,
        ip: null,
        sshPassword: "",
        sshPort: null,
        traderPo: null,
        businessPort: null,
        connectionString: null,
        keyProcessName: null,
        logAddressConfig: null,
        status: null,
        appPath: ''
      }
      this.resetForm('form')
    },
    /** 提交按钮 */
    submitForm() {
      let form = this.form
      this.$refs['form'].validate(valid => {
        if (valid) {
          if (form.serverid) {
            updateTestServer(form).then(res => {
              this.$modal.msgSuccess('修改成功')
              this.open = false
              this.getList()
            })
          } else {
            addTestServer(form).then(res => {
              this.$modal.msgSuccess('新建成功')
              this.open = false
              this.getList()
            })
          }
        }
      })
    },
    // 删除执行机组
    handleDelete(row) {
      this.$modal.confirm('是否确认删除名称为"' + row.name + '"的服务器？').then(function () {
        return delTestServer(row.serverid)
      }).then(() => {
        this.getList()
        this.$modal.msgSuccess('删除成功')
      }).catch(() => {
      })
    },
    handleTest(row) {
      let formpost = {
        serverid: row.serverid
      }
      connectTestServerA(formpost).then(res => {
        this.$modal.msgSuccess('连接正常')
      }).catch(() => {
        this.$modal.msgError('连接异常')
      })
    },
    // 连通性测试
    //BUGID003699 链接成功的返回和测试连通性
    handleConnectivityTest() {
      if (this.showbutton) {
        this.$refs['form'].validate(valid => {
          if (valid) {
            this.loadingConnectivity = true
            let formpost = {
              serverid: 0,
              ip: this.form.ip,
              sshPort: this.form.sshPort,
              username: this.form.username,
              sshPassword: this.form.sshPassword
            }
            connectTestServerA(formpost).then(res => {
              this.$modal.msgSuccess(res.msg)
              this.form.status = 0
              this.loadingConnectivity = false
            }).catch(() => {
              this.form.status = 1
              this.loadingConnectivity = false
            })
          }
          else {
            this.$modal.msgError('请注意数据填写！');
          }
        })
      }
      else {
        let formpost = {
          serverid: this.form.serverid
        }
        this.loadingConnectivity = true
        connectTestServerA(formpost).then(res => {
          this.$modal.msgSuccess(res.msg)
          this.loadingConnectivity = false
        }).catch(() => {
          this.$modal.msgError('连接异常')
        })
      }
    }
  }
}
</script>

<style lang="scss" scoped>
.connectivity-btn {
  position: absolute;
  right: 35px;
  top: 5px;
}
</style>
