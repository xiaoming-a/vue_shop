<template>
  <div>
    <!-- 面包屑导航区 -->
    <el-breadcrumb separator-class="el-icon-arrow-right">
      <el-breadcrumb-item :to="{ path: '/home' }">首页</el-breadcrumb-item>
      <el-breadcrumb-item>权限管理</el-breadcrumb-item>
      <el-breadcrumb-item>角色列表</el-breadcrumb-item>
    </el-breadcrumb>
    <!-- 卡片视图区域 -->
    <el-card>
      <!-- 添加角色按钮区域 -->
      <el-row>
        <el-col>
          <el-button type="primary">添加角色</el-button>
        </el-col>
      </el-row>

      <!-- 角色列表区域 -->
      <el-table :data="rolesList" border stripe>
        <!-- 展开列 -->
        <el-table-column type="expand">
          <template slot-scope="scope">
            <el-row
              :class="['bdbottom',i1 ===0 ? 'bdtop':'','vcenter']"
              v-for="(item1,i1) in scope.row.children"
              :key="item1.id"
            >
              <!-- 一级权限 -->
              <el-col :span="5">
                <el-tag closable @close="removeRightById(scope.row,item1.id)">{{item1.authName}}</el-tag>
                <i class="el-icon-caret-right"></i>
              </el-col>
              <!-- 二级、三级权限 -->
              <el-col :span="19">
                <el-row
                  :class="[i2 ===0 ? '':'bdtop','vcenter']"
                  v-for="(item2, i2) in item1.children"
                  :key="item2.id"
                >
                  <el-col :span="6">
                    <el-tag
                      type="success"
                      closable
                      @close="removeRightById(scope.row,item2.id)"
                    >{{item2.authName}}</el-tag>
                    <i class="el-icon-caret-right"></i>
                  </el-col>
                  <el-col :span="18">
                    <el-tag
                      type="warning"
                      v-for="(item3) in item2.children"
                      :key="item3.id"
                      closable
                      @close="removeRightById(scope.row,item3.id)"
                    >{{item3.authName}}</el-tag>
                  </el-col>
                </el-row>
              </el-col>
            </el-row>
          </template>
        </el-table-column>
        <!-- 索引列 -->
        <el-table-column type="index"></el-table-column>
        <el-table-column lable="角色名称" prop="roleName"></el-table-column>
        <el-table-column lable="角色描述" prop="roleDesc"></el-table-column>
        <el-table-column lable="操作" width="300px">
          <template slot-scope="scope">
            <el-button type="primary" icon="el-icon-edit" size="mini">编辑</el-button>
            <el-button type="danger" icon="el-icon-delete" size="mini">删除</el-button>
            <el-button
              type="warning"
              icon="el-icon-setting"
              size="mini"
              @click="showSetRightDialog(scope.row)"
            >分配权限</el-button>
          </template>
        </el-table-column>
      </el-table>
    </el-card>

    <!-- 分配权限对话框提示 -->
    <el-dialog
      title="分配权限"
      :visible.sync="setRightDialogVisible"
      width="50%"
      @close="setRightDialogClosed"
    >
      <!-- 树形控件 -->
      <el-tree
        :data="rightsList"
        :props="treeProps"
        show-checkbox
        node-key="id"
        default-expand-all
        :default-checked-keys="defkeys"
        ref="treeRef"
      ></el-tree>
      <span slot="footer" class="dialog-footer">
        <el-button @click="setRightDialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="allotRights">确 定</el-button>
      </span>
    </el-dialog>

    <!--  -->
  </div>
</template>

<script>
export default {
  data() {
    return {
      // 所有角色列表数据
      rolesList: [],
      //分配权限对话框的显示与隐藏
      setRightDialogVisible: false,
      //所有权限的数据
      rightsList: [],
      // 遍历树形
      treeProps: {
        label: "authName",
        children: "children",
      },
      // 默认选中的节点id值数组
      defkeys: [],
      // 当前即将分配权限的ID
      roleId: "",
    };
  },
  created() {
    this.getRolesList();
  },
  methods: {
    // 获取所有角色列表
    async getRolesList() {
      const { data: result } = await this.$http.get("roles");
      if (result.meta.status !== 200) {
        return this.$message.error(" 获取角色列表失败");
      }
      this.rolesList = result.data;
    },
    // 根据id删除对应的权限
    async removeRightById(role, rightId) {
      // 提示
      const confirmResult = await this.$confirm(
        "此操作将永久删除该权限, 是否继续?",
        "提示",
        {
          confirmButtonText: "确定",
          cancelButtonText: "取消",
          type: "warning",
        }
      ).catch((err) => err);
      if (confirmResult !== "confirm") {
        return this.$message.info("取消了删除");
      }
      const { data: result } = await this.$http.delete(
        `roles/${role.id}/rights/${rightId}`
      );
      if (result.meta.status !== 200) {
        return this.$message.error("删除权限失败");
      }
      // this.getRolesList();
      role.children = result.data;
    },
    // 分配权限管理
    async showSetRightDialog(role) {
      this.roleId = role.id;
      // 获取所有权限额数据
      const { data: result } = await this.$http.get("rights/tree");

      if (result.meta.status !== 200) {
        return this.$message.error("获取失败");
      }
      // 权限数据存储
      this.rightsList = result.data;
      // 递归获取三级id
      this.getLeafKeys(role, this.defkeys);
      this.setRightDialogVisible = true;
    },
    // 通过递归获取角色三级权限
    getLeafKeys(node, arr) {
      // 如果当前节点没有下级权限，就赋值给arr数组
      if (!node.children) {
        return arr.push(node.id);
      }
      node.children.forEach((item) => this.getLeafKeys(item, arr));
    },
    // 监听权限对话框的关闭事件
    setRightDialogClosed() {
      this.defkeys = [];
    },
    // 点击确认分配权限
    async allotRights() {
      const keys = [
        ...this.$refs.treeRef.getCheckedKeys(),
        ...this.$refs.treeRef.getHalfCheckedKeys(),
      ];
      // console.log(keys);
      const idStr = keys.join(",");
      const {
        data: result,
      } = await this.$http.post(`roles/${this.roleId}/rights`, { rids: idStr });
      if (result.meta.status !== 200) {
        this.$message.error("分配权限失败");
      }
      this.$message.success("分配权限成功");
      this.getRolesList();
      this.setRightDialogVisible = false;
    },
  },
};
</script>


<style lang="less" scoped>
.el-tag {
  margin: 7px;
}

.bdtop {
  border-top: 1px solid #eeeeee;
}

.bdbottom {
  border-bottom: 1px solid #eeeeee;
}

.vcenter {
  display: flex;
  align-items: center;
}
</style>