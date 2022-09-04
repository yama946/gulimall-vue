<template>
  <!-- 多个element组件，需要一个根元素进行包裹，否则会报错，这里使用div标签包裹 -->
  <div>
    <el-switch
    style="margin-bottom: 20px"
        v-model="draggable"
        active-color="#13ce66"
        inactive-color="#ff4949"
        active-text="开启拖拽"
        inactive-text="关闭拖拽"
      >
      </el-switch>
      <el-button type="primary" v-if="draggable" @click="batchSave" style="float: right" round>批量更新</el-button>
      <el-button type="danger" @click="batchDelete" style="float: right" round>批量删除</el-button>

    <el-tree
      :data="menus"
      :props="defaultProps"
      :default-expanded-keys="expandedKey"
      show-checkbox
      node-key="catId"
      :expand-on-click-node="false"
      :draggable="draggable"
      :allow-drop="allowDrop"
      @node-drop="handleDrop"
      ref="menuTree"
    >
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            v-if="data.catLevel <= 2"
            type="text"
            size="mini"
            @click="() => append(data)"
          >
            添加
          </el-button>
          <el-button type="text" size="mini" @click="edit(data)">
            修改
          </el-button>
          <el-button
            v-if="data.children.length == 0"
            type="text"
            size="mini"
            @click="() => remove(node, data)"
          >
            删除
          </el-button>
        </span>
      </span>
    </el-tree>

    <el-dialog
      :title="title"
      :visible.sync="dialogVisible"
      width="30%"
      :before-close="handleClose"
    >
      <!-- 绑定category对象，作为表单对象进行获取发送 -->
      <el-form :model="category">
        <el-form-item label="分类名称">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图标地址">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input
            v-model="category.productCount"
            autocomplete="off"
          ></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="submitData">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
//这里可以导入其他文件（比如：组件，工具 js，第三方插件 js，json文件，图片文件等等）
//例如：import 《组件名称》 from '《组件路径》';

export default {
  //import 引入的组件需要注入到对象中才能使用
  data() {
    //注意data中参数后一定要添加空格赋值时，否则报错！！！！！！！！！！！！
    return {
      pCid: 0,
      draggable: false,
      updateNodes: [],
      maxLevel: 1,
      title: " ",
      //复用对话框，此属性用来决定调用那个方法
      dialogType: " ",
      category: {
        name: " ",
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        icon: "",
        productCount: 0,
        catId: null,
      },
      dialogVisible: false,
      menus: [],
      expandedKey: [],
      defaultProps: {
        children: "children",
        label: "name",
      },
    };
  },
  //方法集合
  methods: {
    //获取菜单内容
    getMenu() {
      this.$http({
        url: this.$http.adornUrl("/product/category/list/menu"),
        method: "get",
      }).then(({ data }) => {
        console.log("获取到菜单数据:", data.data);
        this.menus = data.data;
      });
    },
    //批量删除功能
    batchDelete(){
      let catIds = [];
      let names= [];
      //getCheckedNodes为vue的树形控件中提供的方法
      let checkedNodes = this.$refs.menuTree.getCheckedNodes();
      console.log("被选中的节点：",checkedNodes);
      //收集节点的catId，发送后端进行删除
      for(let i=0;i<checkedNodes.length;i++){
        catIds.push(checkedNodes[i].catId);
        names.push(checkedNodes[i].name);
      }
      //消息提示框
      this.$confirm(`是否批量删除【${names}】菜单`, "提示", {
        //点击确定会调用then中的代码段
        confirmButtonText: "确定",
        //点击取消会调用catch中的代码段，catch部分可以置空，不可删除
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(catIds, false),
          }).then(({ data }) => {
            console.log("批量删除成功后返回的数据：", data);
            //删除成功消息通知
            this.$notify({
              message: "菜单批量删除成功",
              type: "success",
            });
            //刷新出新的菜单
            this.getMenu();
          });
        })
        .catch(() => {
          console.log("取消，不做任何操作");
        });
    },
    //批量保存拖拽数据
    batchSave(){
      this.$http({
        url: this.$http.adornUrl("/product/category/update/sort"),
        method: "post",
        data: this.$http.adornData(this.updateNodes, false),
      }).then(({ data }) => {
        console.log("拖拽数据更新后返回的数据：", data);
        //更新成功消息通知
        this.$notify({
          message: "菜单顺序成功",
          type: "success",
        });
        //刷新菜单
        this.getMenu();
        this.expandedKey = [this.pCid];
        this.pCid= 0;
        // this.updateNodes=[];
        // this.maxLevel=1;
      });
    },
    //拖拽成功绑定事件,之后需要将改变的pid,cid,sort数据进行更新
    handleDrop(draggingNode, dropNode, dropType, ev) {
      console.log(
        "拖拽成功事件传递的参数: ",
        draggingNode,
        dropNode,
        dropType,
        ev
      );
      //1.获取最新的pid数据
      let pCid = 0;
      let siblings = null;
      if (dropType == "before" || dropType == "after") {
        pCid =
          dropNode.parent.data.catId == undefined
            ? 0
            : dropNode.parent.data.catId;
        siblings = dropNode.parent.childNodes;
      } else {
        pCid = dropNode.data.catId;
        siblings = dropNode.childNodes;
      }

      this.pCid = pCid;

      console.log("收集的兄弟节点数据:", siblings, siblings.length);
      //2.当前拖拽节点的最新顺序
      //拖拽后更新所有的兄弟顺序
      for (let i = 0; i < siblings.length; i++) {
        if (siblings[i].data.catId == draggingNode.data.catId) {
          //如果遍历的是正在拖拽的节点
          let catLevel = draggingNode.level;
          if (siblings[i].level != draggingNode.level) {
            //当前节点的层级发生变化,并修改当前节点的层级
            catLevel = siblings[i].level;

            //当前节点的层级发生变化，字节点的层级也会发生变化
            //修改其子节点的层级
            this.updateChildNodeLevel(siblings[i]);
          }
          //3.当前拖拽节点的最新层级
          this.updateNodes.push({
            catId: siblings[i].data.catId,
            sort: i,
            parentCid: pCid,
            catLevel: catLevel,
          });
        } else {
          //updateNodes为data中定义的参数，进行组装数据进行为发送请求做准备
          this.updateNodes.push({ catId: siblings[i].data.catId, sort: i });
        }
      }
      //4.发送请求进行更新拖拽数据
      // this.$http({
      //   url: this.$http.adornUrl("/product/category/update/sort"),
      //   method: "post",
      //   data: this.$http.adornData(this.updateNodes, false),
      // }).then(({ data }) => {
      //   console.log("拖拽数据更新后返回的数据：", data);
      //   //删除成功消息通知
      //   this.$notify({
      //     message: "菜单顺序成功",
      //     type: "success",
      //   });
      //   //刷新菜单
      //   this.getMenu();
      //   this.expandedKey = [pCid];
      //   // this.updateNodes=[];
      //   // this.maxLevel=1;
      // });

      //不刷新第二次拖拽会积累siblings数量，原因据说是未刷新？？？？？
      console.log("拖拽后收集数据结果：", this.updateNodes);
    },
    //更新子节点的层级
    updateChildNodeLevel(node) {
      if (node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          var cNode = node.childNodes[i].data;
          this.updateNodes.push({
            catId: cNode.catId,
            catLevel: node.childNodes[i].level,
          });
          this.updateChildNodeLevel(node.childNodes[i]);
        }
      }
    },
    //统计当前被拖动节点的总层数
    countNodeLevel(node) {
      //1,找到所有子节点，求出最大深度,递归操作
      if (node.children != null && node.children.length > 0) {
        for (let i = 0; i < node.children.length; i++) {
          if (node.children[i].catLevel > this.maxLevel) {
            this.maxLevel = node.children[i].catLevel;
          }
          this.countNodeLevel(node.children[i]);
        }
      } else {
        this.maxLevel = node.catLevel;
      }
    },
    //拖拽时判定目标节点能否被放置，被拖拽后层级不能超过3级
    allowDrop(draggingNode, dropNode, type) {
      //1、被拖动节点以及拖动后所在节点总层级不能超过3

      //2、被拖动节点的总层数
      console.log("当前拖拽节点函数参数数据：", draggingNode, dropNode, type);
      this.countNodeLevel(draggingNode.data);
      //当前正在拖动的节点+父节点所在的深度不大于3即可
      let deep = this.maxLevel - draggingNode.data.catLevel + 1;
      console.log("深度：", deep);
      //this.maxLevel
      if (type == "inner") {
        return deep + dropNode.level <= 3;
      } else {
        return deep + dropNode.parent.level <= 3;
      }
    },
    //点击确定按扭，调用修改方法还是添加方法
    submitData() {
      if (this.dialogType == "add") {
        this.addCategory();
      }
      if (this.dialogType == "edit") {
        this.editCategory();
      }
    },
    //修改菜单,当前方法要放置到获取menu方法之后，原因未知？？？？
    edit(data) {
      console.log("修改的菜单数据为：", data);
      this.title = "修改分类";
      this.dialogType = "edit";
      //开启对话框
      this.dialogVisible = true;
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: "get",
        params: this.$http.adornParams({}),
      }).then(({ data }) => {
        console.log("修改前请求最新数据：", data);
        this.category.name = data.data.name;
        this.category.catId = data.data.catId;
        this.category.icon = data.data.icon;
        this.category.productCount = data.data.productCount;
        this.category.parentCid = data.data.parentCid;
      });
      //回显数据,这样回显数据如果多个用户同时操作无法显示最新的数据，我们通过请求进行回显
      // this.category.name = data.name;
      // this.category.catId = data.catId;
      // this.category.parentCid = data.parentCid;
      // this.category.catLevel = data.catLevel;
    },
    //发送修改请求，修改菜单
    editCategory() {
      //获取到category的cid进行更新数据
      console.log("修改的三级分类数据", this.category);
      const { name, catId, icon, productCount } = this.category;
      let data = { name, catId, icon, productCount };
      //发送修改请求
      this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "post",
        data: this.$http.adornData(data, false),
      }).then(({ data }) => {
        console.log("修改成功后返回的数据：", data);
        //删除成功消息通知
        this.$notify({
          message: "菜单修改成功",
          type: "success",
        });
        //刷新菜单
        this.getMenu();
        this.expandedKey = [this.category.parentCid];
      });
      //对话框关闭
      this.dialogVisible = false;
    },
    //添加菜单
    append(data) {
      console.log("添加提示信息:", data);
      this.title = "添加分类";
      //显示对话框
      this.dialogVisible = true;
      this.category.name = "";
      this.category.catId = null;
      (this.category.icon = ""),
        (this.category.productCount = null),
        //初始换category数据
        (this.category.parentCid = data.catId);
      //此时的数值乘1，是为了防止类型为字符串，加操作后变成拼串而不是算数运算
      this.category.catLevel = data.catLevel * 1 + 1;
      this.dialogType = "add";
    },
    //添加category
    addCategory() {
      console.log("提交的三级分类数据", this.category);
      //发送保存请求
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "post",
        data: this.$http.adornData(this.category, false),
      }).then(({ data }) => {
        console.log("添加成功后返回的数据：", data);
        //删除成功消息通知
        this.$notify({
          message: "菜单添加成功",
          type: "success",
        });
        //刷新菜单
        this.getMenu();
        this.expandedKey = [this.category.parentCid];
      });
      //对话框关闭
      this.dialogVisible = false;
    },
    //关闭对话框调用方法
    handleClose(done) {
      this.$confirm("确认关闭？")
        .then((_) => {
          done();
        })
        .catch((_) => {});
    },
    // 删除菜单请求，vue传递节点信息，和数据信息
    remove(node, data) {
      console.log("remove提示信息:", node, data);
      var ids = [data.catId];
      this.$confirm(`是否删除【${data.name}】菜单`, "提示", {
        //点击确定会调用then中的代码段
        confirmButtonText: "确定",
        //点击取消会调用catch中的代码段，catch部分可以置空，不可删除
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(ids, false),
          }).then(({ data }) => {
            console.log("删除成功后返回的数据：", data);
            //删除成功消息通知
            this.$notify({
              message: "菜单删除成功",
              type: "success",
            });
            //刷新出新的菜单
            this.getMenu();
            //当前删除的子菜单的父菜单默认展开。
            //设置需要默认展开的菜单,通过node获取父节点的id
            this.expandedKey = [node.parent.data.catId];
          });
        })
        .catch(() => {
          console.log("取消，不做任何操作");
        });
    },
  },
  //计算属性 类似于 data 概念
  computed: {},
  //监控 data 中的数据变化
  watch: {},
  //生命周期 - 创建完成（可以访问当前 this 实例）
  created() {
    this.getMenu();
  },
  //生命周期 - 挂载完成（可以访问 DOM 元素）
  mounted() {},
  beforeCreate() {}, //生命周期 - 创建之前
  beforeMount() {}, //生命周期 - 挂载之前
  beforeUpdate() {}, //生命周期 - 更新之前
  updated() {}, //生命周期 - 更新之后
  beforeDestroy() {}, //生命周期 - 销毁之前
  destroyed() {}, //生命周期 - 销毁完成
  activated() {}, //如果页面有 keep-alive 缓存功能，这个函数会触发
};
</script>
<style>
.custom-tree-node {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: space-between;
  font-size: 14px;
  padding-right: 8px;
}
</style>