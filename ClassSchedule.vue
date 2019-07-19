<template>
  <div class="table">
    <div class="container">
      <div class="crumbs">
        <el-breadcrumb separator="/">
          <el-breadcrumb-item><i class="el-icon-search"></i> 查询条件：</el-breadcrumb-item>
        </el-breadcrumb>
      </div>
      <el-row :gutter="20">
        <el-col :sm="5">
          <el-select v-model="searchCondition.gradeId"
                     placeholder="请选择年级"
                     class=""
                     @change="changeGradeCopy">
            <el-option v-for="item in selectGradeCopy"
                       :label="item.gradeName"
                       :value="item.id"
                       :key="item.id"></el-option>
          </el-select>
        </el-col>
        <el-col :sm="5">
          <el-select v-model="searchCondition.classId"
                     placeholder="请选择班级"
                     class=""
                     @change="changeClassCopy">
            <el-option v-for="item in selectClassCopy"
                       :label="item.className"
                       :value="item.id"
                       :key="item.id"></el-option>
          </el-select>
        </el-col>
        <el-col :md="11">
          <el-button type="primary"
                     icon="el-icon-search"
                     @click="searchInfo">搜索</el-button>
          <el-button type="primary"
                     icon="el-icon-refresh"
                     @click="resetSearch">重置</el-button>
        </el-col>
      </el-row>
      <div class="crumbs">
        <el-breadcrumb separator="/">
          <el-breadcrumb-item><i class="el-icon-lx-cascades"></i> 查询结果：</el-breadcrumb-item>
        </el-breadcrumb>
      </div>
      <div class="handle-box">
      </div>
      <div class="subject-center">
        <div class="select-subject-title">
          <p>科目</p>
          <ul class="subject-content clearfix"
              id='subject-content'
              @dragover='subjectDragover($event)'
              @drop='subjectDrop($event)'>
            <li v-for="(subject,i) in subjectData"
                :key="i"
                :draggable='true'
                @dragstart='subjectDragstart(i,subject.id,$event)'>{{subject.subjectName}}
            </li>
          </ul>
        </div>
        <div id="courseWrapper"
             class="courseWrapper">
          <div class="courses-head ">
            <ul class="week">
              <li v-for="(week, i) in weeks"
                  :key="i">{{week}}</li>
            </ul>
          </div>
          <div class="courses-content">
            <ul class="subject"
                v-for="(weekNum, i) in courseArray"
                :key="i">
              <li v-for="(course, j) in weekNum"
                  :key="j"
                  :draggable='course.draggable'
                  @dragstart='courseDragstart(i,j,course.subjectId,$event)'
                  @dragover='courseDragover($event)'
                  @drop='courseDrop(i,j,course.subjectId,$event)'>{{ course.courseName }}</li>
            </ul>
          </div>
          <div class="courses-leftHand">
            <ul class="class-time">
              <li>#</li>
              <li>第一节</li>
              <li>第二节</li>
              <li>第三节</li>
              <li>第四节</li>
              <li>第五节</li>
              <li>第六节</li>
              <li>第七节</li>
              <li>第八节</li>
              <li>第九节</li>
              <li>第十节</li>
              <li>第十一节</li>
              <li>第十二节</li>
            </ul>
          </div>
        </div>
      </div>
    </div>

    <!-- 重新登录提示框 -->
    <el-dialog title="提示"
               :visible.sync="loginVisible"
               width="300px"
               :close-on-click-modal='false'
               center>
      <div class="del-dialog-cnt">登录时间过长，请重新登录</div>
      <span slot="footer"
            class="dialog-footer">
        <el-button @click="loginVisible = false">取 消</el-button>
        <el-button type="primary"
                   @click="signOut('/login')">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
export default {
  name: 'classScheduleInfo',
  data () {
    return {
      Request: this.$api.api.prototype,/* 用于获取请求地址 */
      Common: this.$common.common.prototype,/* 用于获取公共方法 */
      classInfoUrl: '',
      subjectInfo: '',
      courseInfo: '',
      selectGradeCopy: [],
      selectClassCopy: [],
      selectGrade: [],
      selectClass: [],

      limitShow: localStorage.getItem('userType'),

      tableData: [],
      selectTableData: [],/* 选中行数据 */
      subjectData: [],

      dialogTitle: '',
      editVisible: false,
      delVisible: false,
      loginVisible: false,/* 提示需要登录的对话框 */
      resetVisible: false,
      delVisibleType: 'delete',
      disabledbtn: false,/* 可用与不可用 */
      disabledbtnExport: false,/* 可用与不可用 */
      isDisabledbtn: false,/* 显示或隐藏 */

      importVisible: false,
      fileList: [],
      warningInfo: '',
      headers: {
        Authorization: localStorage.getItem('token')
      },

      loading: true,
      loadingDialog: false,/* 对话框中数据提交加载 */
      propTypeCodeDisabled: false,/* 是否可编辑 */
      classCodeDisabled: true,/* 是否可编辑 */
      showType: 'add',/* 用于对话框显示 类型标记 */
      methodType: 'post',/* 请求类型 */

      weeks: ['周一', '周二', '周三', '周四', '周五', '周六', '周日'],
      courseNums: [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11],
      courseWeeks: [0, 1, 2, 3, 4, 5, 6],
      courseData: [],
      courseArray: [],

      /* 查询条件 */
      searchCondition: {
        classId: '',
        // classId: 746,
        gradeId: ''
      },

      /* 表单中需要提交的数据 */
      courseForm: {
        classId: '',
        gradeId: '',
        courseNum: '',
        courseWeek: '',
        subjectId: ''
      },
      /* 删除时所需的编号 删除时的id */
      idx: -1,
      id: -1
    }
  },
  created () {
    this.gradeInfoUrl = this.Request.getGradeInfoUrl();
    this.classInfoUrl = this.Request.getClassInfoUrl();
    this.subjectInfo = this.Request.getSubjectInfoUrl();
    this.courseInfo = this.Request.getCourseInfoUrl();

    /* 获取年级信息 */
    this.getGradeInfoDataRequest();
    /* 获取学科信息 */
    this.getSubjectDataRequest();

    /* 初始化二维数组 */
    this.courseArray = new Array();
    for (var i = 0; i < 12; i++) {
      this.courseArray[i] = new Array();
      for (var j = 0; j < 7; j++) {
        let _name = {
          id: '',
          draggable: false,
          courseName: '',
          subjectId: ''
        }
        this.courseArray[i][j] = _name;
      }
    }
  },
  computed: {},
  methods: {

    /* 取消拖拽的默认行为 */
    subjectDragover (e) {
      e.preventDefault();
      e.stopPropagation();
      return false;
    },

    /* 课程表-->学科 当放置被拖数据时，会发生 drop 事件。放置课程表中的数据时 */
    subjectDrop (e) {
      const dt = e.dataTransfer;
      let i = dt.getData("i");
      let j = dt.getData("j");
      /* 此处进行删除操作 
      方法1：
        1.删除对应二维数组的值，即可
        2.请求后端删除对应的值
      方法2：请求后端删除对应的值，再进行数据的初始化
      */
      this.deleteCourse(i, j);
    },

    /* 学科 当开始拖拽的时候触发 */
    subjectDragstart (index, id, e) {
      const dt = e.dataTransfer;
      dt.setData("key", index);
      dt.setData("subjectId", id);
    },

    /* 课程表 当开始拖拽的时候触发 */
    courseDragstart (i, j, subjectId, e) {
      const dt = e.dataTransfer;
      console.log(subjectId);
      dt.setData("i", i);
      dt.setData("j", j);
      dt.setData("subjectId", subjectId);
    },

    /* 取消默认行为 */
    courseDragover (e) {
      e.preventDefault();
      e.stopPropagation();
      return false;
    },

    /* 科目--->课程 当放置被拖数据时，会发生 drop 事件。 */
    courseDrop (i, j, subjectId, e) {
      if (this.searchCondition.gradeId === '') {
        this.$message.warning('请先选择年级');
      } else if (this.searchCondition.classId === '') {
        this.$message.warning('请先选择班级');
      } else {
        const dt = e.dataTransfer;
        console.log(subjectId, dt.getData("i"));
        console.log(subjectId, dt.getData("j"));
        console.log(subjectId, dt.getData("subjectId"));
        let temp = this.courseArray[i];
        temp[j].courseName = this.subjectData[dt.getData("key")].subjectName;
        temp[j].draggable = true;
        this.$set(this.courseArray, i, temp);
        /* 此处进行修改操作 
            方法1：向后端发送数据，并进行二维数组赋值
            方法2：向后端发送数据，并初始化数据(使用这种，但查询请求会很多)
        */
        this.courseForm.courseNum = i;
        this.courseForm.courseWeek = j;
        this.courseForm.subjectId = dt.getData("subjectId");
        this.updateCourse(this.courseInfo, 'post', this.courseForm);
      }
    },

    /* 删除某一课程 */
    deleteCourse (i, j) {
      /* 删除一条或多条数据 */
      let idArray = [];
      idArray.push(this.courseArray[i][j].id);
      this.Common.deleteAxios(this.courseInfo, {
        idArray: idArray
      }, this.deleteCourseInfo);
    },

    /* 修改某一课程 */
    updateCourse (url, type, formData) {
      this.Common.commonAxios(url, type, formData, this.updateCourseInfo);
    },

    /* 删除后的信息 */
    deleteCourseInfo (data) {
      if (data.status === 200) {
        let tempData = data.data;
        if (!this.validToken(tempData)) return;
        if (tempData.code === 0) {
          this.$message.success(tempData.message);
          /* 重新刷新数据 */
          this.initCourseArray();
          this.getCourseDataRequest();
        } else {
          this.$message.error(tempData.message);
        }
      } else {
        this.$message.error('服务器繁忙，请稍后再试');
      }
    },

    /* 初始化二维数组 */
    initCourseArray () {
      for (var i = 0; i < 12; i++) {    //一维长度为i,i为变量，可以根据实际情况改变
        for (var j = 0; j < 7; j++) {   //一维数组里面每个元素数组可以包含的数量p，p也是一个变量；
          let _name = {
            id: '',
            draggable: false,
            courseName: '',
            subjectId: ''
          }
          let temp = this.courseArray[i];
          temp[j] = _name
          // this.courseArray[i][j] = _name;    //这里将变量初始化，我这边统一初始化为空，后面在用所需的值覆盖里面的值
          this.$set(this.courseArray, i, temp);
        }
      }
    },

    /* 修改后的信息 */
    updateCourseInfo (data) {
      if (data.status === 200) {
        let tempData = data.data;
        if (!this.validToken(tempData)) return;
        if (tempData.code === 0) {
          this.$message.success(tempData.message);
          /* 重新刷新数据 */
          this.initCourseArray();
          this.getCourseDataRequest();
        } else {
          this.$message.error(tempData.message);
        }
      } else {
        this.$message.error('服务器繁忙，请稍后再试');
      }
    },

    /* 请求：获取年级信息 */
    getGradeInfoDataRequest () {
      this.Common.getAxios(this.gradeInfoUrl, {
      }, this.getGradeInfoData);
    },
    /* 请求：获取学科中数据 */
    getSubjectDataRequest () {
      this.Common.getAxios(this.subjectInfo, this.searchCondition, this.getSubjectData);
    },
    /* 请求：获取课程表中数据 */
    getCourseDataRequest () {
      this.Common.getAxios(this.courseInfo, this.searchCondition, this.getCourseData);
    },

    /* 获取学科数据  */
    getSubjectData (data) {
      if (data.status === 200) {
        let tempData = data.data;
        if (!this.validToken(tempData)) return;
        if (tempData.code === 0 && tempData.result !== "") {
          this.subjectData = tempData.result;
        }
      } else {
        this.$message.error('服务器繁忙，请稍后再试');
      }
    },
    /* 获取课程表数据  */
    getCourseData (data) {
      if (data.status === 200) {
        let tempData = data.data;
        this.loading = false;
        if (!this.validToken(tempData)) return;
        if (tempData.code === 0 && tempData.result !== null) {
          this.courseData = tempData.result;
          for (let i = 0; i < this.courseData.length; i++) {
            let courseNum = this.courseData[i].courseNum;
            let courseWeek = this.courseData[i].courseWeek;
            if (courseNum < 12 && courseWeek < 7) {
              let temp = this.courseArray[courseNum];
              temp[courseWeek].subjectId = this.courseData[i].subjectId;
              temp[courseWeek].courseName = this.courseData[i].subjectName;
              temp[courseWeek].draggable = true;
              temp[courseWeek].id = this.courseData[i].id;
              this.$set(this.courseArray, courseNum, temp)
              /* es6不支持  直接修改数组中的值  进行监听 */
              // this.courseArray[courseNum][courseWeek] = this.courseData[i].subjectName;
            }
          }
        }
      } else {
        this.$message.error('服务器繁忙，请稍后再试');
        this.loading = false;
      }
    },

    /* 获取年级信息 */
    getGradeInfoData (data) {
      if (data.status === 200) {
        let tempData = data.data;
        if (!this.validToken(tempData)) return;
        if (tempData.code === 0 && tempData.result !== "") {
          this.selectGradeCopy = tempData.result;
        }
      } else {
        this.$message.error('服务器繁忙，请稍后再试');
      }
    },

    /* 查询区域：获取班级信息 */
    getClassInfoCopyData (data) {
      if (data.status === 200) {
        let tempData = data.data;
        if (!this.validToken(tempData)) return;
        if (tempData.code === 0 && tempData.result !== "") {
          this.searchCondition.classId = '';
          this.selectClassCopy = tempData.result;
        }
      } else {
        this.$message.error('服务器繁忙，请稍后再试');
      }
    },

    /* 查询区域事件:  年级下拉框 */
    changeGradeCopy (data) {
      this.Common.getAxios(this.classInfoUrl, {
        gradeId: data
      }, this.getClassInfoCopyData);
      this.courseForm.gradeId = data;
    },

    /* 查询区域事件:  班级下拉框 */
    changeClassCopy (data) {
      this.courseForm.classId = data;
      this.searchInfo();
    },

    /* 搜索 */
    searchInfo () {
      /* 重置到第一页 */
      if (this.searchCondition.gradeId === '') {
        this.$message.warning('请先选择年级');
      } else if (this.searchCondition.classId === '') {
        this.$message.warning('请先选择班级');
      } else {
        this.initCourseArray();
        this.getCourseDataRequest();
      }
    },

    /* 重置搜索内容 */
    resetSearch () {
      this.searchCondition.gradeId = '';
      this.searchCondition.classId = '';
      this.courseForm.gradeId = '';
      this.courseForm.classId = '';
      this.courseForm.courseNum = '';
      this.courseForm.courseWeek = '';
      this.courseForm.subjectId = '';
      this.initCourseArray();
    },

    /* 确定删除 */
    deleteRow (deleteUrl) {
      if (this.delVisibleType !== 'delete' && this.delVisibleType !== 'batchDelete') {
        this.delVisible = false;
        this.$message.error(`别偷偷的修改特定的参数哦`);
        return;
      }
      let selection = [];
      let length = 0;
      /* 获取要删除数据的id值数组 (单个与批量删除相同) */
      let idArray = [];
      if (this.delVisibleType === 'delete') {
        idArray.push(this.id);
      } else if (this.delVisibleType === 'batchDelete') {
        /*获取选中的值 */
        selection = this.$refs.multipleTable.store.states.selection;
        length = selection.length;
        for (let i = 0; i < length; i++) {
          idArray.push(selection[i].id);
        }
      }
      this.loading = true;
      this.disabledbtn = true;
      /* 删除一条或多条数据 */
      this.Common.deleteAxios(deleteUrl, {
        idArray: idArray
      }, this.deleteInfo, idArray);
      this.delVisible = false;
    },

    /* 校验token */
    validToken (data) {
      if (data.code === 2001 || data.code === 2101 || data.code === 2102) {
        this.loginVisible = true;
        return false;
      }
      return true;
    },

    /* 退出登录 */
    signOut (data) {
      localStorage.removeItem('token');
      this.$router.push(data);
    },

    /* 关闭上传 弹出框事件 */
    closedImportDialog () {
      this.warningInfo = '';
      this.fileList = [];
    }
  }
}

</script>

<style scoped>
/* layout-布局 */
.el-row {
  margin-bottom: 20px;
}
.el-row:last-child {
  margin-bottom: 0;
}
.el-col {
  border-radius: 4px;
}
.bg-purple-dark {
  background: #99a9bf;
}
.bg-purple {
  background: #d3dce6;
}
.bg-purple-light {
  background: #e5e9f2;
}
.grid-content {
  border-radius: 4px;
  min-height: 36px;
}
.row-bg {
  padding: 10px 0;
  background-color: #f9fafc;
}

.handle-box {
  margin-bottom: 20px;
}

.handle-select {
  width: 120px;
}

.handle-input {
  width: 300px;
  display: inline-block;
}
.del-dialog-cnt {
  font-size: 16px;
  text-align: center;
}
.table {
  width: 100%;
  font-size: 14px;
}
.red {
  color: #ff0000;
}
.mr10 {
  margin-right: 10px;
}

.textAlignCenter {
  text-align: center;
}

/* 修改select的盒模型 */
.el-select {
  display: block;
}

/* 确定按钮的显示与隐藏 */
.disabledbtn {
  display: none;
}

/* 修改该属性值  查询条件，查询结果得样式 */
.el-breadcrumb {
  border-bottom: 1px solid #ebeef5;
  padding-bottom: 10px;
}

/* 批量导入告警信息提示 */
.warning-info {
  padding-top: 20px;
  color: #f00;
}

/* 课程表 */
li {
  list-style-type: none;
}
.courseWrapper {
  position: relative;
  width: 70%;
  padding-left: 50px;
  font: 14px/1.5;
}

.courses-head {
  width: 100%;
  background-color: #16baef;
  border-top-right-radius: 10px;
}

.courses-head .week {
  display: flex;
  justify-content: space-around;
  width: 100%;
}

.courses-head .week li {
  width: 14.28%;
  padding: 9px 0;
  border-left: 2px solid;
  text-align: center;
  color: #fff;
}

.subject {
  display: flex;
  justify-content: space-around;
  background-color: rgba(22, 186, 239, 0.1);
  border-bottom: 1px solid #fff;
  color: #000;
}
.courses-content {
  /* font: 14px/1.5; */
  line-height: 136%;
}
.courses-content ul:first-child {
  border-top: 1px solid rgb(219, 219, 219);
}

.courses-content ul:last-child {
  border-bottom-right-radius: 10px;
}

.subject-first-top-border {
  border-top: 1px solid rgb(219, 219, 219);
}

.subject li {
  width: 14.28%;
  height: 19px;
  padding: 10px 0;
  border-left: 2px solid #fff;
  text-align: center;
  color: #000;
}

.subjectRadio {
  border-bottom-right-radius: 10px;
}

/* 左侧 */
.courses-leftHand {
  position: absolute;
  left: -42px;
  top: 0px;
  width: 96px;
}

.class-time li {
  /* width: 90px; */
  padding: 10px 18px;
  background-color: #16baef;
  border-right: 1px solid;
  text-align: center;
  border-bottom: 1px solid #fff;
  color: #fff;
}

.class-time li:first-child {
  border-top-left-radius: 10px;
}

.class-time li:last-child {
  border-bottom-left-radius: 10px;
}

.subject-center {
  display: flex;
  justify-content: space-around;
}

/* 拖拽选择学科的样式 */
.select-subject-title {
  width: 233px;
  height: 100%;
  margin-right: 50px;
  font-size: 17px;
  text-align: center;
  color: #fff;
}

.subject-center p {
  margin-bottom: 10px;
  padding: 8px 15px;
  background-color: #16baef;
  border-radius: 20px;
  font-size: 22px;
  font-weight: bold;
}

.subject-center .subject-content li {
  float: left;
  width: 97px;
  margin: 6px;
  padding: 3px 0;
  background-color: #16baef;
  border-radius: 10px;
}

/* 清除浮动 */
.clearfix:after {
  display: block;
  clear: both;
  content: "";
}

.clearfix {
  *zoom: 1;
}
</style>
