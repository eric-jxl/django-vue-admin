<!--
 * @创建文件时间: 2021-06-01 22:41:21
 * @Auther: 猿小天
 * @最后修改人: 猿小天
 * @最后修改时间: 2021-09-26 21:18:29
 * 联系Qq:1638245306
 * @文件介绍:授权管理
-->
<template>
  <d2-container :class="{ 'page-compact': false }">
    <SplitPane :min-percent="20" :default-percent="20" split="vertical">
      <template slot="paneL">
        <div style="margin: 10px">
          <div class="yxt-flex-between">
            <div>
              <el-tag>
                当前选择:{{ roleObj.name ? roleObj.name : "无" }}
              </el-tag>
            </div>
            <div>
              <el-button
                type="primary"
                size="mini"
                @click="submitPermisson"
                v-permission="'Save'"
                >保存
              </el-button>
            </div>
          </div>
          <br />
          <el-tree
            class="filter-tree"
            :data="data"
            :highlight-current="true"
            :props="{ label: 'name' }"
            default-expand-all
            :filter-node-method="filterNode"
            @node-click="nodeClick"
            node-key="node_id"
            ref="tree"
          >
          </el-tree>
        </div>
      </template>
      <template slot="paneR">
        <SplitPane split="horizontal" :min-percent="20" :default-percent="30">
          <template slot="paneL">
            <div style="margin: 10px">
              <div>
                <div style="margin-bottom: 20px">
                  <div class="yxt-flex-align-center">
                    <div class="yxt-divider"></div>
                    <span>数据授权</span>
                    <el-tooltip
                      class="item"
                      effect="dark"
                      :content="dataAuthorizationTips"
                      placement="right"
                    >
                      <el-icon class="el-icon-question"></el-icon>
                    </el-tooltip>
                  </div>
                </div>
                <el-row>
                  <el-col :span="6">
                    <el-select
                      v-show="roleObj.name"
                      v-model="roleObj.data_range"
                      @change="dataScopeSelectChange"
                    >
                      <el-option
                        v-for="item in dataScopeOptions"
                        :key="item.value"
                        :label="item.label"
                        :value="item.value"
                      ></el-option>
                    </el-select>
                  </el-col>
                  <el-col :span="18">
                    <div v-show="roleObj.data_range === 4" class="dept-tree">
                      <el-tree
                        :data="deptOptions"
                        show-checkbox
                        default-expand-all
                        :default-checked-keys="deptCheckedKeys"
                        ref="dept"
                        node-key="id"
                        :check-strictly="true"
                        :props="{ label: 'name' }"
                      ></el-tree>
                    </div>
                  </el-col>
                </el-row>
              </div>
            </div>
          </template>
          <template slot="paneR">
            <div style="margin: 10px">
              <div>
                <div style="margin-bottom: 20px">
                  <div class="yxt-flex-align-center">
                    <div class="yxt-divider"></div>
                    <span>菜单授权</span>
                    <el-tooltip
                      class="item"
                      effect="dark"
                      :content="menuAuthorizationTips"
                      placement="right"
                    >
                      <el-icon class="el-icon-question"></el-icon>
                    </el-tooltip>
                  </div>
                </div>
                <el-tree
                  ref="menuTree"
                  :data="menuOptions"
                  node-key="id"
                  default-expand-all
                  show-checkbox
                  :expand-on-click-node="false"
                  :default-checked-keys="menuCheckedKeys"
                  :check-on-click-node="true"
                  :check-strictly="true"
                  empty-text="请先选择角色"
                >
                  <span class="custom-tree-node" slot-scope="{ node, data }">
                    <div class="yxt-flex-between">
                      <div style="margin-right: 50px">{{ data.name }}</div>
                      <div>
                        <el-checkbox
                          v-for="(item, index) in data.menuPermission"
                          :key="index"
                          v-model="item.checked"
                          >{{ item.name }}</el-checkbox
                        >
                      </div>
                    </div>
                  </span>
                </el-tree>
              </div>
            </div>
          </template>
        </SplitPane>
      </template>
    </SplitPane>
  </d2-container>
</template>

<script>
import * as api from './api'
import * as deptApi from '../dept/api'
import XEUtils from 'xe-utils'
import Vue from 'vue'
import SplitPane from 'vue-splitpane'

Vue.component('SplitPane', SplitPane)
export default {
  name: 'rolePermission',
  data () {
    return {
      filterText: '',
      data: [],
      roleObj: {
        name: null,
        data_range: null
      },
      menuOptions: [],
      permissionData: [],
      menuCheckedKeys: [], // 菜单默认选中的节点
      deptOptions: [],
      deptCheckedKeys: [],
      dataScopeOptions: [
        {
          value: 0,
          label: '仅本人数据权限'
        },
        {
          value: 1,
          label: '本部门数据权限'
        },
        {
          value: 2,
          label: '本部门及以下数据权限'
        },
        {
          value: 3,
          label: '全部数据权限'
        },
        {
          value: 4,
          label: '自定数据权限'
        }
      ],
      dataAuthorizationTips: '授权用户可操作的数据范围',
      menuAuthorizationTips: '授权用户在菜单中可操作的范围'
    }
  },
  watch: {
    filterText (val) {
      this.$refs.tree.filter(val)
    }
  },
  methods: {
    filterNode (value, data) {
      if (!value) return true
      return data.label.indexOf(value) !== -1
    },
    getCrudOptions () {
      // eslint-disable-next-line no-undef
      return crudOptions(this)
    },
    pageRequest (query) {
      return api.GetList(query).then(res => {
        res.map((value, index) => {
          value.node_id = index
        })
        this.data = res
        this.$nextTick().then(() => {
          this.initNode()
        })
      })
    },
    initNode () {
      if (this.$route.params.id && this.$refs.tree) {
        this.data.map(value => {
          if (this.$route.params.id === value.id) {
            this.node_id = value.node_id
          }
        })
        const node = this.$refs.tree.getNode(this.node_id)
        this.$refs.tree.setCurrentKey(node)
        this.nodeClick(node.data, node)
      }
    },
    addRequest (row) {
      return api.createObj(row)
    },
    updateRequest (row) {
      return api.UpdateObj(row)
    },
    delRequest (row) {
      return api.DelObj(row.id)
    },
    // 获取部门数据
    getDeptData () {
      deptApi.GetList({ status: 1 }).then(ret => {
        this.deptOptions = ret.data.data
      })
    },
    // 获取菜单数据
    getMenuData (data) {
      api.GetMenuData(data).then(res => {
        res.forEach(x => {
          // 根据当前角色的permission,对menuPermisson进行勾选处理
          x.menuPermission.forEach(a => {
            if (data.permission.indexOf(a.id) > -1) {
              this.$set(a, 'checked', true)
            } else {
              this.$set(a, 'checked', false)
            }
          })
        })
        // 将菜单列表转换为树形列表
        this.menuOptions = XEUtils.toArrayTree(res, {
          parentKey: 'parent',
          strict: true
        })
      })
    },
    // 角色树被点击
    nodeClick (data, node, self) {
      this.roleObj = data
      this.getDeptData()
      this.getMenuData(data)
      this.menuCheckedKeys = data.menu // 加载已勾选的菜单
      this.deptCheckedKeys = data.dept
    },
    // 所有勾选菜单节点数据
    getMenuAllCheckedKeys () {
      // 目前被选中的菜单节点
      const checkedKeys = this.$refs.menuTree.getCheckedKeys()
      // 半选中的菜单节点
      const halfCheckedKeys = this.$refs.menuTree.getHalfCheckedKeys()
      checkedKeys.unshift.apply(checkedKeys, halfCheckedKeys)
      return checkedKeys
    },
    // 所有自定义权限时,勾选的部门节点数据
    getDeptAllCheckedKeys () {
      // 目前被选中的部门节点
      const checkedKeys = this.$refs.dept.getCheckedKeys()
      // 半选中的部门节点
      const halfCheckedKeys = this.$refs.dept.getHalfCheckedKeys()
      checkedKeys.unshift.apply(checkedKeys, halfCheckedKeys)
      return checkedKeys
    },
    // 提交修改
    submitPermisson () {
      this.roleObj.menu = this.getMenuAllCheckedKeys() // 获取选中的菜单
      this.roleObj.dept = this.getDeptAllCheckedKeys() // 获取选中的部门
      const menuData = XEUtils.toTreeArray(this.menuOptions)
      const permissionData = []
      menuData.forEach(x => {
        const checkedPermission = x.menuPermission.filter(f => {
          return f.checked
        })

        if (checkedPermission.length > 0) {
          for (const item of checkedPermission) {
            permissionData.push(item.id)
          }
        }
      })
      this.roleObj.permission = permissionData
      return this.updateRequest(this.roleObj).then(res => {
        this.$message.success('更新成功')
      })
    },
    /** 选择角色权限范围触发 */
    dataScopeSelectChange (value) {
      if (value !== 4) {
        // this.$refs.dept.setCheckedKeys([]);
      }
    }
  },
  created () {
    this.pageRequest()
  }
}
</script>

<style lang="scss">
.yxtInput {
  .el-form-item__label {
    color: #49a1ff;
  }
}

.dept-tree::-webkit-scrollbar {
  display: none; /* Chrome Safari */
}
.dept-tree {
  height: 160px;
  overflow-y: scroll;
  scrollbar-width: none; /* firefox */
  -ms-overflow-style: none; /* IE 10+ */
}
</style>
