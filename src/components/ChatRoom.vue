<template>
  <div class="body">
    <div v-if="login" class="common-layout">
      <div class="title">
        <span>当前用户: {{userData.userName}}</span>
      </div>
      <el-row>
        <el-col :span="6">
      <el-page-header icon="ArrowLeft" @back="handleLogout()"/>
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
              <el-menu-item v-for="(item,index) in array" :index="String(item)" :key="index">{{item}}</el-menu-item>
            </el-menu-item-group>
          </el-sub-menu>
        </el-menu>
      </el-col>
      <el-col :span="18">
      <el-card>
        <template #header>
          <span v-if="tab==='public'">公聊💫</span>
          <span v-else>对{{tab}}✨</span>
        </template>
        <div height="30vh">
          <el-scrollbar height="20vh">
          </el-scrollbar>
          <el-divider></el-divider>
          
          <div min-height="10vh">
            <el-input type="text">
                <template #prepend>
                    <el-upload>
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
  <div v-else class="common-layout">
    <el-input v-model="userData.userName" placeholder="请输入用户名" type="text" @keyup.enter="handleLogin()">
    <template #append>
      <el-button type="primary" @click="handleLogin()">进入聊天室</el-button>
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

<style scoped>
*{
    margin: 0;
    padding: 0;
    }
.body{
    width:60vw;
    margin:0 auto;
    margin-top:25vh;
}
.body .title{
    background-color: rgb(155, 127, 255);
    height:3vh;
    text-align: center;
    border-radius: 5px;
}
</style>
