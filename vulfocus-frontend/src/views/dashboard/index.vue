<template>

  <div class="dashboard-container">
    <el-dialog :visible.sync="centerDialogVisible" :before-close="handleDialogClose" title="镜像信息" >
      <div class="text item" v-loading="startCon" element-loading-text="环境启动中">
        <div class="text item" >
          访问地址: {{vul_host}}
        </div>
        <div class="text item">
          映射端口：
          <el-tag v-for="(value, key) in vul_port" :key="key" style="margin-right: 5px;">
            {{key}}:{{value}}
          </el-tag>
        </div>
        <div class="text item">
          名称: {{images_name}}
        </div>
        <div class="text item">
          描述: {{images_desc}}
        </div>
        <el-form>
          <el-form-item label="Flag">
            <el-input v-model="input" placeholder="请输入Flag：格式flag-{xxxxxxxx}"></el-input>
          </el-form-item>
          <el-form-item>
            <el-button type="primary" @click="subFlag(container_id,input.trim())" :disabled="cStatus">提 交</el-button>
            <!--</div>-->
          </el-form-item>
        </el-form>
      </div>
    </el-dialog>
    <el-row :gutter="6" >
      <el-col :span="18" v-if="this.countlist.length===0">
        <el-input v-model="search" style="width: 230px;" size="medium"></el-input>
        <el-select v-model="searchForm.time_img_type" @change="getselectdata" multiple filterable allow-create default-first-option placeholder="请选择漏洞类型" style="left: 5px">
          <el-option v-for="item in degreeList" :key="item.value" :label="item.value" :value="item.value"></el-option>
        </el-select>
        <el-input-number @change="getselectdata" v-model="searchForm.rank_range" style="left: 10px" :precision="1" :step="0.5" :max="5" :min="0"></el-input-number>
        <el-button class="filter-item" size="medium" style="margin-left: 10px;margin-bottom: 10px" type="primary" icon="el-icon-search" @click="handleQuery(1)">
          查询
        </el-button>
      </el-col>
    </el-row>
    <el-row :gutter="24" >
      <el-col :span="6" v-for="(item,index) in listdata" :key="index"  style="padding-bottom: 18px;">
        <el-card :body-style="{ padding: '8px' }" shadow="hover"
                 @click.native=" item.status.status === 'running' && open(item.image_id,item.image_vul_name,item.image_desc,item.status.status,item.status.container_id,item)" >
          <div class="clearfix" >
            <div style="display: inline-block;height: 20px;line-height: 20px;min-height: 20px;max-height: 20px;">
              <svg-icon icon-class="bug"  style="font-size: 20px;"/>
              <el-tooltip v-if="(item.status.status === 'stop' || item.status.status === 'delete') && item.status.is_check === true" content="已通过" placement="top">
                <i style="color: #20a0ff;" class="el-icon-check"></i>
              </el-tooltip>
              <el-tooltip v-else-if="item.status.status === 'running'" content="运行中" placement="top">
                <i style="color: #20a0ff;" class="el-icon-loading"></i>
              </el-tooltip>
              <el-tooltip v-else-if="item.status.status === 'stop' && item.status.is_check === false" content="暂停中" placement="top">
                <svg-icon style="color: #20a0ff;" icon-class="stop" />
              </el-tooltip>
              <div style="display: inline-block;margin: 0;" v-if="item.status.status === 'running' && item.status.start_date !== null && item.status.start_date !=='' && item.status.end_date !== null && item.status.end_date !== '' && item.status.end_date !== 0">
                <el-tooltip content="容器剩余时间，0 为用不过期" placement="top">
                  <i class="el-icon-time"></i>
                </el-tooltip>
                <count-down style="display: inline-block;height: 20px;line-height: 20px;size: 20px;margin-block-start: 0em;margin-block-end: 0em;" v-on:end_callback="stop(item.status.container_id, item)" :currentTime="item.status.now" :startTime=item.status.now :endTime=item.status.end_date :secondsTxt="''"></count-down>
              </div>
              <div style="display: inline-block;" v-else-if="item.status.status === 'running' && item.status.start_date !== null && item.status.start_date !=='' && item.status.end_date !== null && item.status.end_date !== '' && item.status.end_date === 0">
                <el-tooltip content="容器剩余时间，0 为用不过期" placement="top">
                  <i class="el-icon-time"></i>
                </el-tooltip>
                <p style="display: inline-block;">-1</p>
              </div>
              <div v-else style="display: inline-block;">
                <p style="display: inline-block;margin-block-start: 1em;margin-block-end: 1em"></p>
              </div>
            </div>
            <div style="margin-top: 7px;">
              <el-rate v-model=item.rank disabled show-score text-color="#ff9900" score-template={value}></el-rate>
            </div>
          </div>
          <div style="padding: 5px;" >
            <div class="container-title">
            <span>{{item.image_vul_name}}</span>
            </div>
            <div class="bottom clearfix">
              <div class="time container-title">{{ item.image_desc }}</div>
            </div>
            <el-row>
              <el-button type="primary" @click.stop="stop(item.status.container_id,item)" :disabled="item.status.stop_flag" size="mini" v-if="item.status.status === 'running'">停止</el-button>
              <el-button type="primary" @click.stop="open(item.image_id,item.image_vul_name,item.image_desc,item.status.status,item.status.container_id,item)" :disabled="item.status.start_flag" size="mini" v-else="item.status.status === '' || item.status.status === 'stop'">启动</el-button>
              <el-button type="primary" @click.stop="deleteContainer(item.status.container_id,item)" v-if="item.status.status === 'running' || item.status.status === 'stop'" :disabled="item.status.delete_flag" size="mini" icon="el-icon-stopwatch">删除</el-button>
            </el-row>
          </div>
        </el-card>
      </el-col>
    </el-row>
    <div style="margin-top: 20px">
      <el-pagination
        :page-size="page.size"
        @current-change="handleQuery"
        layout="total, prev, pager, next, jumper"
        :total="page.total">
      </el-pagination>
    </div>

  </div>
</template>

<script>
import { ImgList,SubFlag,ContainerSTART,ContainerDelete,ContainerStop } from '@/api/docker'
import { publicMethod,gettimetemp } from '@/api/timemoudel'
import { getTask } from '@/api/tasks'
import CountDown from 'vue2-countdown'
import { Notification } from 'element-ui'
import {stoptimetemp} from "@/api/timemoudel";
export default {
  inject: ['reload'],
  name: 'Dashboard',
  components: {
    CountDown
  },
  replace:true,
  data() {
    return {
      page:{
        total: 0,
        size: 20,
      },
      listdata: [],
      vul_host: "",
      centerDialogVisible: false,
      startCon:false,
      startTime:(new Date()).getTime(),
      input: "",
      images_id: "",
      container_id: "",
      images_name: "",
      images_desc: "",
      item_raw_data: "",
      cStatus: true,
      search: "",
      searchForm:{
        time_img_type:"",
        rank_range:0
      },
      vul_port:{},
      countlist:[],
      notifications: {},
      degreeList:[
          {value:"命令执行", lable:"命令执行"},
          {value:"代码执行", lable:"代码执行"},
          {value:"文件写入", lable:"文件写入"},
          {value:"文件上传", lable:"文件上传"},
          {value:"后门", lable:"后门"},
          {value:"默认口令", lable:"默认口令"},
          {value:"弱口令", lable:"弱口令"},
          {value:"权限绕过", lable:"权限绕过"},
          {value:"未授权访问", lable:"未授权访问"},
          {value:"XXE漏洞", lable:"XXE漏洞"},
          {value:"SQL注入", lable:"SQL注入"},
          {value:"文件读取", lable:"文件读取"},
          {value:"文件下载", lable:"文件下载"},
          {value:"文件包含", lable:"文件包含"},
          {value:"文件删除", lable:"文件删除"},
          {value:"目录遍历", lable:"目录遍历"},
          {value:"信息泄漏", lable:"信息泄漏"},
          {value:"任意账户操作", lable:"任意账户操作"},
          {value:"XSS漏洞", lable:"XSS漏洞"},
          {value:"SSRF漏洞", lable:"SSRF漏洞"},
          {value:"CSRF漏洞", lable:"CSRF漏洞"},
        ],
      };
    },
  created() {
    this.listData(1)
    this.timeData()
  },
  beforeDestroy(){
    Notification.closeAll()
  },
  methods:{
      timeData(){
        gettimetemp().then(response => {
          this.countlist = response.data.results
          if (this.countlist.length===0){
          }else {
            this.countlist[0].end_date = publicMethod.getTimestamp(this.countlist[0].end_date)
            this.countlist[0].start_date = publicMethod.getTimestamp(this.get_time)
            this.$notify({
              title: '计时模式',
              message:<count-down currentTime={this.countlist[0].start_date} startTime={this.countlist[0].start_date} endTime={this.countlist[0].end_date} dayTxt={"天"} hourTxt={"小时"} minutesTxt={"分钟"} secondsTxt={"秒"}></count-down>,
              duration: 0,
              position: 'bottom-right',
              showClose: false,
              dangerouslyUseHTMLString:true,
            });
          }})
      },
      listData() {
          ImgList().then(response => {
            this.listdata = response.data.results
            this.page.total = response.data.count
            for (let i = 0; i <this.listdata.length ; i++) {
              this.listdata[i].status.start_flag = false
              this.listdata[i].status.stop_flag = false
              this.listdata[i].status.delete_flag = false
            }
          })
      },
      getselectdata(){
        ImgList(undefined,undefined,undefined,true,this.searchForm.time_img_type,this.searchForm.rank_range).then(response =>{
            this.listdata = response.data.results
            this.page.total = response.data.count
            for (let i = 0; i <this.listdata.length ; i++) {
            this.listdata[i].status.start_flag = false
            this.listdata[i].status.stop_flag = false
            this.listdata[i].status.delete_flag = false
          }
        }).catch((e)=>{})
      },
      open(id,images_name,images_desc,status,container_id,raw_data) {
        this.images_id = ""
        this.images_name = ""
        this.images_desc = ""
        this.container_id = ""
        this.item_raw_data = ""
        this.vul_host = ""
        this.startCon = "loading"
        this.cStatus = true
        this.item_raw_data = raw_data
        this.images_id = id
        this.images_name = images_name
        this.images_desc = images_desc
        this.centerDialogVisible = true
        this.$set(raw_data.status, "start_flag", true)
        this.$forceUpdate();
        if(raw_data.status.is_check === true){
          this.$message({
            message:  "该题目已经通过，重复答题分数不会累计",
            type: "success",
          })
          // this.centerDialogVisible = false
        }
        if (raw_data.status.status === "running"){
          this.vul_host = raw_data.status.host
          this.vul_port = JSON.parse(raw_data.status.port)
          this.container_id = raw_data.status.container_id
          this.startCon = false
          this.cStatus = false
        }else{
          ContainerSTART(id).then(response=>{
          let taskId = response.data["data"]
          let tmpRunContainerInterval = window.setInterval(() => {
            setTimeout(()=>{
              getTask(taskId).then(response=>{
                let responseStatus = response.data["status"]
                let responseData = response.data
                if (responseStatus === 1001){
                  // 一直轮训
                }else{
                  clearInterval(tmpRunContainerInterval)
                  raw_data.status.start_flag = false
                  if (responseStatus === 200){
                    container_id = responseData["data"]["id"]
                    this.container_id = container_id
                    this.vul_host = responseData["data"]["host"]
                    this.vul_port = responseData["data"]["port"]
                    raw_data.status.now = responseData["data"]["_now"]
                    raw_data.status.start_date = responseData["data"]["start_date"]
                    raw_data.status.end_date = responseData["data"]["end_date"]
                    raw_data.status.status = responseData["data"]["status"]
                    raw_data.status.container_id = container_id
                    this.startCon = false
                    this.cStatus = false
                  }else if (responseStatus === 201){
                    this.$message({
                      message: response.data["msg"],
                      type: "error",
                    })
                    this.listData(1)
                    this.timeData()
                    this.centerDialogVisible = false
                  }else{
                    this.$message({message:  response.data["msg"],
                      type: "error",
                    })
                    this.listData(1)
                    this.timeData()
                    this.centerDialogVisible = false
                  }
                }
              })
            },1)
          },2000)
        })
        }
      },
      subFlag(id,flag) {
          SubFlag(id,flag).then(response => {
            this.input = ""
            let responseData = response.data
            if(responseData["status"] === 200){
              this.$message({
                message:  "恭喜！通过",
                type: "success",
              })
              this.listData(1)
            }else if(responseData.status === 201){
              this.$message({
                message: responseData["msg"],
                type: "error",
              })
            }else{
              this.$message({
                message:  responseData["msg"],
                type: "error",
              })
            }
            this.centerDialogVisible = false
            this.reload()
            this.item_raw_data.status.status = 'stop'
          })
      },
      stop(container_id,raw) {
        /**
         * 停止容器运行
         */
        this.$set(raw.status, "stop_flag", true)
        this.$forceUpdate();
        ContainerStop(container_id).then(response=>{
          let taskId = response.data["data"]
          let tmpStopContainerInterval = window.setInterval(() => {
            setTimeout(()=>{
              getTask(taskId).then(response=>{
                let responseStatus = response.data["status"]
                let responseData = response.data
                if (responseStatus === 1001){
                  // 一直轮训
                }else{
                  clearInterval(tmpStopContainerInterval)
                  if (responseStatus === 200){
                    this.$message({
                      message: responseData["msg"],
                      type: "success",
                    })
                    raw.status.status = "stop"
                    raw.status.start_date = ""
                    raw.status.stop_flag = false
                  }else{
                    this.$message({
                      message: responseData["msg"],
                      type: "error",
                    })
                  }
                }
              })
            },1)
          },2000)
        })
      },
      deleteContainer(container_id,raw){
        /**
         * 删除容器
         */
        this.$set(raw.status, "delete_flag", true)
        this.$forceUpdate();
        ContainerDelete(container_id).then(response=>{
          let taskId = response.data["data"]
          let tmpDeleteContainerInterval = window.setInterval(() => {
            setTimeout(()=>{
              getTask(taskId).then(response=>{
                let responseStatus = response.data["status"]
                let responseData = response.data
                if (responseStatus === 1001){
                  // 一直轮训
                }else{
                  clearInterval(tmpDeleteContainerInterval)
                  raw.status.delete_flag = false
                  if (responseStatus === 200){
                    // 清空状态码
                    raw.status.status = ""
                    // 清空 image_id
                    this.images_id = ""
                    // 清空 image_name
                    this.images_name = ""
                    // 清空 image_desc
                    this.images_desc = ""
                    // 清空 container_id
                    this.container_id = ""
                    // 清空 item_raw_data
                    this.item_raw_data = ""
                    raw.status.container_id = ""
                    this.$message({
                      message: responseData["msg"],
                      type: "success",
                    })
                    this.listData(1)
                  }else{
                    this.$message({
                      message: responseData["msg"],
                      type: "error",
                    })
                  }
                }
              })
            },1)
          },2000)
        })
    },
      handleQuery(page){
        ImgList(this.search,false,page,true,this.searchForm.time_img_type,this.searchForm.rank_range).then(response => {
          this.listdata = response.data.results
          this.page.total = response.data.count
        })
      },
      autoStop(){
        stoptimetemp().then(response => {
          const data = response.data;
          let msgType = 'success';
          let msg = '';
          if('2000'===data.code){
            msg = '计时模式已经关闭！'
          }else{
            msgType = 'error';
            msg = '关闭失败,内部错误';
          }
          this.$message({
            type: msgType,
            message: msg,
          });
        })
    },
      handleDialogClose(){
        this.reload()
      }
  },
  mounted: function() {
      var _this = this;
      let yy = new Date().getFullYear();
      let mm = new Date().getMonth()+1;
      let dd = new Date().getDate();
      let hh = new Date().getHours();
      let mf = new Date().getMinutes()<10 ? '0'+new Date().getMinutes() : new Date().getMinutes();
      let ss = new Date().getSeconds()<10 ? '0'+new Date().getSeconds() : new Date().getSeconds();
      _this.get_time = yy+'-'+mm+'-'+dd+' '+hh+':'+mf+':'+ss;
  },
}

</script>

<style lang="scss" scoped>
.dashboard {
  &-container {
    margin: 30px;
  }
  &-text {
    font-size: 30px;
    line-height: 46px;
  }
}
.time {
  font-size: 13px;
  color: #999;
}

.bottom {
  margin-top: 5px;
  margin-bottom: 13px;
  line-height: 12px;
}

.button {
  padding: 5px;
  float: right;
}

.image {
  width: 100%;
  display: block;
}

.clearfix:before,
.clearfix:after {
  display: table;
  content: "";
}

.clearfix:after {
  clear: both
}

.text {
  font-size: 14px;
}

.item {
  margin-bottom: 18px;
}

.container-title{
  width: 100%;    /*根据自己项目进行定义宽度*/
  overflow: hidden;     /*设置超出的部分进行影藏*/
  text-overflow: ellipsis;     /*设置超出部分使用省略号*/
  white-space:nowrap ;    /*设置为单行*/
}

.date {

}
.date p{
  height: 20px;
  line-height: 20px;
  margin: 0;

  margin-block-end: 0em;
}

.el-row {
  display: flex;
  flex-wrap: wrap;
}

/*p {*/
/*  height: 20px;*/
/*  line-height: 20px;*/
/*}*/
</style>
