<template>
  <div id="body">
    <div v-if="login" class="glass" style="opacity:0.9;">
      <el-row>
        <el-col :span="6">
      <el-page-header icon="ArrowLeft" style="background-color:white;" @back="handleLogout()"/>
        <el-menu
          :default-active="tab"
          class="el-menu-vertical-demo"
          @select="handleSelect"
        >
          <el-menu-item index="public">
              <el-icon><Comment /></el-icon>
              <span>公聊</span>
          </el-menu-item>
          <el-sub-menu index="self">
            <template #title>
              <el-icon><ChatDotRound /></el-icon>
              <span>私聊</span>
            </template>
            <el-menu-item-group>          
              <el-menu-item id="selectMenu" v-for="(item,index) in array" :index="String(item)" :key="index">{{item}}</el-menu-item>
            </el-menu-item-group>
          </el-sub-menu>
        </el-menu>
      </el-col>
      <el-col :span="18">
      <el-card height="50vh">
        <template #header>
          <span v-if="tab==='public'">公聊💫</span>
          <span v-else>对{{tab}}✨</span>
        </template>
        <div>
          <el-scrollbar height="30vh">
          </el-scrollbar>
          <el-divider></el-divider>
          
          <div min-height="10vh">
            <el-input type="text" v-model="userData.message">
                <template #prepend>
                    <el-upload id="upload">
                        <el-button type="primary">选择文件</el-button>
                    </el-upload>
                </template>
                <template #append>
                    <el-button>发送</el-button>
                </template>
            </el-input>
          </div>
        </div>    
      </el-card>
      </el-col>
      </el-row>
  </div>
  <div v-else class="frosted-glass">
    <h1 class="title">SAKURA</h1>
    <el-input v-model="userData.userName" placeholder="请输入用户名" type="text" @keyup.enter="handleLogin()">
    <template #append>
      <el-button type="primary" id="enter" @click="handleLogin()">进入聊天室</el-button>
    </template>
    </el-input>
  </div>
  </div>
</template>
<script setup>
import { ref, unref, reactive, onMounted, onUnmounted } from 'vue'
import { useStorage } from '@vueuse/core'
import { over } from 'stompjs';
import  SockJS  from "sockjs-client/dist/sockjs"
import { ElMessage } from 'element-plus'
let stompClient = null;
let login = ref(false)
const userData = ref({
  userName:'',
  receiverName: '',
  connected: false,
  message: ''
})
const defaultData = {}
onMounted(()=>{
  Object.assign(defaultData,userData.value);
})
const array = [1,2,3];
const tab = ref("public");
// menu 自带事件
const handleSelect = (index, keyPath, item) => {
  console.log("select",index, keyPath, item);
  tab.value = index;
}
// 登录
const handleLogin = ()=>{
  if(userData.value.userName===''){
    ElMessage({
    message: `请输入用户名 💦💦💦`,
    type: 'error',
  })
    return;
  }
  login.value = true;
  connect();
  ElMessage({
    message: `欢迎你，${userData.value.userName} 🚀🚀🚀`,
    type: 'success',
  })
}
// 登出
const handleLogout = ()=>{
  ElMessage({
    message: `再见，${userData.value.userName} 💨💨💨`
  })
  if(stompClient){
    let message = {
        senderName: userData.value.userName,
        messageType:"text",
        status:"LEAVE"
    };
    stompClient.send("/app/message",{},JSON.stringify(message));
    stompClient.disconnect();
  }
  userData.value = defaultData;
  login.value = false;
  tab.value = "public"
}
// 链接websocket
const connect = ()=>{
    let sock = new SockJS('http://localhost:8080/ws');
    stompClient = over(sock);
    stompClient.connect({}, onConnected, onError);
}
// 链接成功回调
const onConnected = () => {
    userData.value.connected=true;
    console.log("🚀 成功连接", userData.value);
    // 公聊
    stompClient.subscribe('/chatroom/public', onPublicMessageReceived);
    // 私聊
    stompClient.subscribe('/user/'+userData.value.username+'/private', onPrivateMessageReceived);
    // 用户加入聊天室
    userJoin();
}
// 链接失败回调
const onError = (err) => {
    console.log(err);    
}
// 用户进入聊天室
const userJoin = ()=>{
  console.log("用户加入聊天室");
}
// 收到公聊消息
const onPublicMessageReceived = (payload)=>{
  let payloadData = JSON.parse(payload.body);
  console.log(payloadData);
}
// 收到私聊信息
const onPrivateMessageReceived = (payload)=>{
  let payloadData = JSON.parse(payload.body);
  console.log(payloadData);
}


</script>

<style>

/* *{
    margin: 0;
    padding: 0;
} */
#body{
    width:60vw;
    margin:0 auto;
    margin-top:10vh;
}

.frosted-glass {
    display: flex;
    justify-content: center;
    align-items: center;
    width: 72vw;
    height: 36vh;
    box-shadow: 0 0.3px 0.7px rgba(0, 0, 0, 0.126), 0 0.9px 1.7px rgba(0, 0, 0, 0.179), 0 1.8px 3.5px rgba(0, 0, 0, 0.224), 0 3.7px 7.3px rgba(0, 0, 0, 0.277), 0 10px 20px rgba(0, 0, 0, 0.4);
    backdrop-filter: blur(20px);
    transition: 0.5s ease;
}
.frosted-glass:hover {
    box-shadow: 0 0.7px 1px rgba(0, 0, 0, 0.157), 0 1.7px 2.6px rgba(0, 0, 0, 0.224), 0 3.5px 5.3px rgba(0, 0, 0, 0.28), 0 7.3px 11px rgba(0, 0, 0, 0.346), 0 20px 30px rgba(0, 0, 0, 0.5);
}
.frosted-glass .title {
    padding-left: 0.375em;
    font-size: 3.6em;
    font-family: Lato, sans-serif;
    font-weight: 200;
    letter-spacing: 0.75em;
    color: white;
}
@media (max-width: 640px) {
    .frosted-glass .title {
        font-size: 2em;
    }
}

body {
    height: 100vh;
}
.frosted-bg, .glass {
    background: url(https://i.loli.net/2019/11/17/GAYyzeKsiWjP5qO.jpg);
    background-position: center;
    background-repeat: no-repeat;
}
.frosted-bg {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    transition: 1s;
}
.glass {
    position: absolute;
    top: 50%;
    left: 50%;
    z-index: 1;
    width: 600px;
    height: 450px;
    transform: translate(-50%, -50%);
    transition: 1s;
}
.glass body {
    height: 100vh;
}
.glass .frosted-bg, .glass .glass {
    background-position: center;
    background-repeat: no-repeat;
}
.glass .frosted-bg {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    transition: 1s;
}
.glass .glass {
    position: absolute;
    top: 50%;
    left: 50%;
    z-index: 1;
    width: 600px;
    height: 450px;
    transform: translate(-50%, -50%);
    transition: 1s;
}
.glass .glass:hover {
    box-shadow: 0 0.7px 1.7px rgba(0, 0, 0, 0.248), 0 1.7px 4.3px rgba(0, 0, 0, 0.355), 0 3.5px 8.9px rgba(0, 0, 0, 0.445), 0 7.3px 18.3px rgba(0, 0, 0, 0.552), 0 20px 50px rgba(0, 0, 0, 0.8);
}
.glass .glass:hover ~ .frosted-bg {
    filter: blur(10px);
}
.glass:hover {
    box-shadow: 0 0.7px 1.7px rgba(0, 0, 0, 0.248), 0 1.7px 4.3px rgba(0, 0, 0, 0.355), 0 3.5px 8.9px rgba(0, 0, 0, 0.445), 0 7.3px 18.3px rgba(0, 0, 0, 0.552), 0 20px 50px rgba(0, 0, 0, 0.8);
}
.glass:hover ~ .frosted-bg {
    filter: blur(10px);
}
body {
    height: 100vh;
}
.frosted-bg, .glass {
    background-position: center;
    background-repeat: no-repeat;
}
.frosted-bg {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    transition: 1s;
}
.glass {
    position: absolute;
    top: 50%;
    left: 50%;
    z-index: 1;
    width: 600px;
    height: 450px;
    transform: translate(-50%, -50%);
    transition: 1s;
}
.glass:hover {
    box-shadow: 0 0.7px 1.7px rgba(0, 0, 0, 0.248), 0 1.7px 4.3px rgba(0, 0, 0, 0.355), 0 3.5px 8.9px rgba(0, 0, 0, 0.445), 0 7.3px 18.3px rgba(0, 0, 0, 0.552), 0 20px 50px rgba(0, 0, 0, 0.8);
}
.glass:hover ~ .frosted-bg {
    filter: blur(10px);
}


:deep() .el-input-group__prepend {
    background-color: rgb(255, 255, 255);
    color: black;
    border-radius: 5px;
}
:deep() .el-input-group__append{
    background-color: rgb(59, 144, 217);
    color: white;
    border-radius: 5px;
}
:deep() #enter{
    border-width: 0;
}
</style>
