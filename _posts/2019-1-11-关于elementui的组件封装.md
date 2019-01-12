---
layout:     post
title:      关于elementui的组件封装
subtitle:   提醒插件封装
date:       2019-1-11
author:     czk
header-img: img/my/img1.jpg
catalog: true
tags:
    - js
    - vue
    - 开发技巧
---
### 工作一段时间后大多应用于elementui进行页面的搭建

- 先看一下平时elementui分页器的使用
``` JavaScript
<div class="block">
    <span class="demonstration">完整功能</span>
    <el-pagination
      @size-change="handleSizeChange"
      @current-change="handleCurrentChange"
      :current-page="currentPage"
      :page-sizes="[100, 200, 300, 400]"
      :page-size="100"
      layout="total, sizes, prev, pager, next, jumper"
      :total="400">
    </el-pagination>
  </div>
```
```JAVASCRIPT
<script>
  export default {
    methods: {
      handleSizeChange(val) {
        console.log(`每页 ${val} 条`);
      },
      handleCurrentChange(val) {
        console.log(`当前页: ${val}`);
      }
    },
    data() {
      return {
        currentPage: 5,
      };
    }
  }
</script>
```



- 但是在实际的与后台交互的过程中不单只是将从后台获取到的数据单纯的绑定到table上还要考虑当table的数据过多时是否应该限制获取的数据而不是全部获取，按需加载，当用户点击下一页或者改变表单时再进行数据的请求
##### 总之大概就是数据不一次获取而是根据用户的点击选择来进行获取减轻服务器的压力及等待时间

- 首先设置分页器
``` JavaScript
 <el-pagination
      :current-page="currentPage"
      :page-sizes="[5,10]"
      :page-size="pagesize"
      :total="total"
      layout="total, sizes, prev, pager, next, jumper"
      @size-change="handleSizeChange"
      @current-change="handleCurrentChange"
    />
```
```JAVASCRIPT
 // 传第几页，页数限制
    PageAxios (page, limit) {
      let _this = this
      this.$axios
        .post(this.QueryUrl, {
          page,
          limit
        })
        .then(res => {
          console.log(res.data)
          console.log(res.data.rows)
          let total = res.data.total // 返回页面总数
          let limit = res.data.limit // 返回页数限制
          // let page = res.data.page // 返回第几页
          _this.tableData = res.data.rows
          // _this.currentPage = page // 初始页
          _this.total = total // 获取页面总数
          _this.pagesize = limit // 初始页
        })
```
##### 封装后的方法调用就十分的简单,只需要传入页面的初始页，和每一页的限制就可以了
```
 // 页面加载获取初始值
    handleDataGet () {
      this.PageAxios(1, this.pagesize)
    },
    // 控制页面页数
    handleSizeChange: function (size) {
      this.PageAxios(1, size)
    },
    // 点击第几页
    handleCurrentChange: function (currentPage) {
      this.PageAxios(currentPage, this.pagesize)
    }
```


#### 可以通过控制currentPage，pagesize 这2个变量来控制每次请求的页数及其请求的条数

- 附带一个elementui的用户提示框 可以根据与后台约定的code参数完成提示
```JAVASCRIPT
 // 封装用户提示
    msgalert (res, _this) {
        //请求成功
      if (res.data.code === 200) {
        this.$message({
          message: res.data.msg,
          type: 'success'
        })
        this.httpGet()
      } else {
        // 提示用户
        console.log(res.data.msg)
        this.$message.error(res.data.msg)
      }
    }
```
```JS
  handle () {
      let _this = this
      this.$axios
        .post(this.$store.state.BaseUrl)
        .then(res => {
          console.log(res.data)
          _this.msgalert(res, _this)
        })
        .catch(e => {
          console.log(e)
        })
    },
```


