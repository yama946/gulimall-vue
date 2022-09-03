<template>
<el-tree 
  :data="menus" 
  :props="defaultProps" 
  show-checkbox
  node-key="catId"
  :expand-on-click-node="false"
  >
    <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            v-if="data.catLevel<=2"
            type="text"
            size="mini"
            @click="() => append(data)">
            添加
          </el-button>
          <el-button
            v-if="data.children.length==0"
            type="text"
            size="mini"
            @click="() => remove(node, data)">
            删除
          </el-button>
        </span>
      </span>
</el-tree>
</template>

<script>
//这里可以导入其他文件（比如：组件，工具 js，第三方插件 js，json文件，图片文件等等）
//例如：import 《组件名称》 from '《组件路径》';

export default {
  
  //import 引入的组件需要注入到对象中才能使用
    data() {
        return {
            menus: [],
            defaultProps: {
                children: 'children',
                label: 'name'
            }
        };
    },
    //方法集合
    methods: {
        getMenu(){
            this.$http({
            url: this.$http.adornUrl('/product/category/list/menu'),
            method: 'get'
            }).then(({data}) => {
                console.log("获取到菜单数据:",data.data)
                this.menus = data.data
            })
        },
        append(data) {
          console.log("add:",data)
      },
      remove(node, data) {
        console.log("remove:",node,data)
      }
    },
  //计算属性 类似于 data 概念
  computed: {},
  //监控 data 中的数据变化
  watch: {},
  //生命周期 - 创建完成（可以访问当前 this 实例）
  created() {
      this.getMenu()
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