<template>
  <div class="app-container">
    <div style="margin-top:20px;margin-bottom: 20px">
      <el-row :gutter="10">
        <el-col :span="4">
          <el-input v-model="listQuery.name" placeholder="账号" />
        </el-col>
        <el-col :span="4">
          <el-input v-model="listQuery.vserName" placeholder="真实姓名" />
        </el-col>
        <el-col :span="4">
          <el-select v-model="listQuery.state" placeholder="用户状态" clearable>
            <el-option
              v-for="item  in stateOptions"
              :key="item.key"
              :label="item.label"
              :value="item.key"
            />
          </el-select>
        </el-col>
        <el-col :span="4">
          <treeselect
            v-model="listQuery.deptid"
            :options="departments"
            :normalizer="normalizer"
            placeholder="请输入部门名称"
          />
        </el-col>
        <el-col :span="8">
          <el-button
            class="filter-item"
            type="primary"
            icon="el-icon-search"
            @click="handleSearch"
            v-permission="['/rest/user/list']"
          >查找</el-button>
          <el-button
            class="filter-item"
            style="margin-left: 10px;"
            type="primary"
            @click="handleCreate"
            v-permission="['/rest/user/add']"
          >
            <i class="el-icon-plus" />新增
          </el-button>
        </el-col>
      </el-row>
    </div>

    <div>
      <el-table
        :key="tableKey"
        :data="list"
        v-loading="listLoading"
        style="width: 100%;"
        height="450"
        border
      >
        <el-table-column width="50">
          <template slot-scope="scope">
            <span>{{scope.$index+(listQuery.page - 1) * listQuery.limit + 1}}</span>
          </template>
        </el-table-column>
        <el-table-column prop="name" label="账号" width="120"></el-table-column>
        <el-table-column prop="vserName" label="真实姓名" width="90"></el-table-column>
        <el-table-column prop="deptName" label="部门" width="120"></el-table-column>
        <el-table-column prop="mobile" label="手机" width="180"></el-table-column>
        <el-table-column prop="email" label="邮箱" width="180"></el-table-column>
        <el-table-column prop="roleName" label="角色" width="150" :formatter="formatRole"></el-table-column>

        <el-table-column label="禁用/启用" width="85">
          <template slot-scope="scope">
            <el-switch
              v-model="scope.row.state"
              :active-value="1"
              :inactive-value="0"
              @change="handleStateChange(scope.row)"
            ></el-switch>
          </template>
        </el-table-column>
        <el-table-column label="操作">
          <template slot-scope="scope">
            <el-button
              @click="handleEdit(scope)"
              type="text"
              size="small"
              v-permission="['/rest/user/update']"
            >编辑</el-button>
          </template>
        </el-table-column>
      </el-table>
      <pagination
        v-show="total>0"
        :total="total"
        :page.sync="listQuery.page"
        :limit.sync="listQuery.limit"
        @pagination="getList"
      />
    </div>
    <el-dialog :visible.sync="dialogVisible" :title="dialogType==='edit'?'编辑':'新增'">
      <el-form :model="user" label-width="80px" label-position="left" style="height: 410px;">
        <el-tabs v-model="activeName">
          <el-tab-pane label="用户信息" name="first">
            <el-form-item label="账号">
              <el-input v-model="user.name" placeholder="账号" />
            </el-form-item>
            <el-form-item label="密码">
              <el-input v-model="user.password" placeholder="密码" />
            </el-form-item>
            <el-form-item label="真实姓名">
              <el-input v-model="user.vserName" placeholder="真实姓名" />
            </el-form-item>
            <el-form-item label="手机号">
              <el-input v-model="user.mobile" placeholder="手机号" />
            </el-form-item>
            <el-form-item label="邮箱">
              <el-input v-model="user.email" placeholder="邮箱" />
            </el-form-item>
            <el-form-item label="状态">
              <el-select v-model="user.state" style="width: 140px" class="filter-item">
                <el-option
                  v-for="item in stateOptions"
                  :key="item.key"
                  :label="item.label"
                  :value="item.key"
                />
              </el-select>
            </el-form-item>
          </el-tab-pane>
          <el-tab-pane label="部门" name="second">
            <el-form-item>
              <el-tree
                ref="tree"
                :check-strictly="true"
                :data="departments"
                :props="defaultProps"
                :default-expanded-keys="defaultExpandeds"
                 default-expand-all
                show-checkbox
                node-key="id"
                class="permission-tree"
                @check="checkDeptTreeNode"
              />
            </el-form-item>
          </el-tab-pane>
          <el-tab-pane label="角色" name="third">
            <el-checkbox-group v-model="user.roleIds">
              <el-checkbox
                v-for="item in roles"
                :key="item.id"
                :label="item.id"
                style="padding-top:20px"
              >{{item.name}}</el-checkbox>
            </el-checkbox-group>
          </el-tab-pane>
        </el-tabs>
      </el-form>

      <div style="text-align:right;">
        <el-button type="danger" @click="dialogVisible=false">取消</el-button>
        <el-button type="primary" @click="confirmUser">确定</el-button>
      </div>
    </el-dialog>
  </div>
</template>
<script>
// import the component
import Treeselect from "@riophae/vue-treeselect";
// import the styles
import "@riophae/vue-treeselect/dist/vue-treeselect.css";
import permission from "@/directive/permission/index.js"; // 权限判断指令
import {
  userList,
  addUser,
  updateUser,
  updateState
} from "@/api/permission/user";
import { departments } from "@/api/permission/department";
import { roles } from "@/api/permission/role";
import { deepCloneAttributes } from "@/utils";
import { isEmpty, isString, isArray } from "@/utils/validate";
import Pagination from "@/components/Pagination"; // Secondary package based on el-pagination
const defaultUser = {
  uid: "",
  name: "",
  password: "",
  vserName: "",
  mobile: "",
  state: undefined,
  email: "",
  deptid: undefined,
  deptName: "",
  roleIds: []
};

export default {
  name: "User",
  components: { Pagination, Treeselect },
  directives: { permission },
  data() {
    return {
      searchOptions: [
        { label: "登录名", key: "name" },
        { label: "真实姓名", key: "vserName" },
        { label: "手机号", key: "mobile" }
      ],
      tableKey: 0,
      list: null,
      total: 0,
      listLoading: false,
      listQuery: {
        page: 1,
        limit: 10,
        vserName: "",
        name: "",
        mobile: "",
        deptid: undefined,
        deptName: "",
        state: undefined
      },
      searchDeptName: "",
      user: Object.assign({}, defaultUser),
      defaultProps: {
        children: "childrens",
        label: "name"
      },
      defaultExpandeds: [],
      stateOptions: [{ label: "禁用", key: 0 }, { label: "启用", key: 1 }],
      departments: [],
      roles: [],
      activeName: "first",
      dialogVisible: false,
      dialogType: "new"
    };
  },
  created() {
    this.getList();
    this.getDepartments();
    this.getRoles();
  },
  watch: {
    searchDeptName(val) {
      this.$refs.serchDeptTree.filter(val);
    }
  },
  methods: {
    async getList() {
      this.listLoading = true;
      //If the Promise is rejected, the rejected value is thrown.
      try {
        const res = await userList(this.listQuery);
        this.listLoading = false;
        this.list = res.result.rows;
        this.total = res.result.records;
      } catch (e) {
        this.listLoading = false;
      }
    },
    handleSearch() {
      this.getList();
    },
    formatRole(row, column) {
      var roleNames = [];
      row.roles.forEach(role => {
        roleNames.push(role.name);
      });
      return roleNames.join(" , ");
    },
    normalizer(node) {
      return {
        id: node.id,
        label: node.name,
        children: node.childrens
      };
    },
    // 用户状态修改
    handleStateChange(row) {
      let text = row.state == 1 ? "启用" : "停用";
      this.$confirm(
        "确认要 [" + text + "] [" + row.name + "] 用户吗?",
        "警告",
        {
          confirmButtonText: "确定",
          cancelButtonText: "取消",
          type: "warning"
        }
      )
        .then(async () => {
          await updateState({ uid: row.uid, state: row.state });
          this.$message({
            message: text + "成功",
            type: "success"
          });
        })
        .catch(err => {
          console.error(err);
          row.state = row.state == 0 ? 1 : 0;
        });
    },
    async getDepartments() {
      const res = await departments();
      let result = res.result;
      this.diGuiTree(result);
      this.departments = result;
    },
    diGuiTree(item) {
      //递归便利树结构
      item.forEach(item => {
        item.childrens === "" ||
        item.childrens === undefined ||
        item.childrens === null
          ? delete item.childrens
          : this.diGuiTree(item.childrens);
      });
    },
    checkDeptTreeNode(a, b) {
      if (b.checkedKeys.length > 0) {
        this.$refs.tree.setCheckedKeys([a.id]);
      }
    },
    async getRoles() {
      const res = await roles();
      this.roles = res.result;
    },
    handleCreate() {
      this.dialogType = "new";
      this.activeName = "first";
      this.dialogVisible = true;
      this.user = Object.assign({}, defaultUser);
      if (this.$refs.tree) {
        this.$refs.tree.setCheckedKeys([], true);
      }
    },
    handleEdit(scope) {
      this.dialogType = "edit";
      this.activeName = "first";
      this.dialogVisible = true;
      deepCloneAttributes(this.user, scope.row);
      this.$nextTick(() => {
        this.$refs.tree.setCheckedKeys([scope.row.deptid], true);
        this.defaultExpandeds = [scope.row.deptid];
        if (scope.row.roles) {
          let roleIds = [];
          scope.row.roles.forEach(role => {
            roleIds.push(role.id);
          });
          this.user.roleIds = roleIds;
        }
      });
    },
    async confirmUser() {
      const isEdit = this.dialogType === "edit";
      var checkedKeys = this.$refs.tree.getCheckedKeys();
      if (checkedKeys) {
        this.user.deptid = Number(checkedKeys.join(","));
      } else {
        this.user.deptid = -1;
      }
      if (isEdit) {
        await updateUser(this.user);
      } else {
        await addUser(this.user);
      }
      this.dialogVisible = false;
      this.$message({
        showClose: true,
        message: "保存成功",
        type: "success"
      });
      this.getList();
    },
    // 节点单击事件
    treeSearchChange(node) {
      this.isShowSelect = false;
      this.listQuery.deptid = node.id;
      this.getList();
    }
  }
};
</script>
<style lang="scss">
</style>