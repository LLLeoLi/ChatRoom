<template>
  <div id="body">
    <div v-if="login" class="glass" style="opacity:0.9;">
      <el-row>
      <el-col :span="6" style="opacity:0.8;">
      <el-page-header icon="ArrowLeft"  @back="handleLogout()"/>
        <el-menu
          :default-active="tab"
          class="el-menu-vertical-demo"
          @select="handleSelect"
        >
          <el-menu-item index="public" style>
              <el-icon><Comment /></el-icon>
              <span>公聊</span>
          </el-menu-item>
          <el-sub-menu index="self">
            <template #title>
              <el-icon><ChatDotRound /></el-icon>
              <span>私聊</span>
            </template>
            <el-menu-item-group>          
              <el-menu-item  v-for="(item,index) in userList.keys()" :index="String(item)" :key="index">
              <span>{{item}}</span>
              <span v-if="item==userData.userName">🎇</span>
              <span v-else>{{userList.get(item)}}</span>
              </el-menu-item>
            </el-menu-item-group>
          </el-sub-menu>
        </el-menu>
      </el-col>
      <el-col :span="18">
      <el-card id="chat">
        <template #header>
          <div class="card-title">
            <span v-if="tab==='public'">公聊💫</span>
            <span v-else>对{{tab}}✨</span>
          </div>
        </template>
          <el-scrollbar height="30vh">
            <div v-if="tab==='public'">
              <div v-for="(chat,index) in publicChats" :key="index">
            <div v-if="chat.senderName ===userData.userName" class="avatar-me">{{chat.senderName}}</div>
            <div v-else class="avatar">{{chat.senderName}}</div>
                    <div class="message-data" v-if="chat.messageType=='text'">{{chat.message}}</div>
                    <div class="message-data" v-if="chat.messageType=='image'">
                          <el-image
                          style="width: 10vw; height: 12vh"
                          :initial-index="0"
                          :src="chat.message"
                          :preview-src-list="[chat.message]"
                          :alt="chat.messageName"
                          fit="cover"
                          />
                    </div>
                    <div class="message-data" v-if="chat.messageType=='file'">
                        <el-button size="large" round @click="downloadFile(chat.message, chat.messageName)">下载 {{chat.messageName}}</el-button>
                    </div>
                    <div class="message-data" v-if="chat.messageType=='video'">
                        <video alt="chat.messageName" width="150" height="150" controls>
                            <source :src="chat.message" type="video" />
                        </video>
                    </div>
                    <div class="message-data" v-if="chat.messageType=='audio'">
                        <audio alt="chat.messageName" controls>
                            <source :src="chat.message" type="audio" />
                        </audio>
                    </div>
            </div>
            </div>
            <div v-else>
              <div v-for="(chat,index) in privateChats.get(tab)" :key="index">
            <div v-if="chat.senderName ===userData.userName" class="avatar-me">{{chat.senderName}}</div>
            <div v-else class="avatar">{{chat.senderName}}</div>
                    <div class="message-data" v-if="chat.messageType=='text'">{{chat.message}}</div>
                    <div class="message-data" v-if="chat.messageType=='image'">
                          <el-image
                          style="width: 10vw; height: 12vh"
                          :initial-index="0"
                          :src="chat.message"
                          :preview-src-list="[chat.message]"
                          :alt="chat.messageName"
                          fit="cover"
                          />
                    </div>
                    <div class="message-data" v-if="chat.messageType=='file'">
                        <el-button size="large" round @click="downloadFile(chat.message, chat.messageName)">下载 {{chat.messageName}}</el-button>
                    </div>
                    <div class="message-data" v-if="chat.messageType=='video'">
                        <video alt="chat.messageName" width="150" height="150" controls>
                            <source :src="chat.message" type="video" />
                        </video>
                    </div>
                    <div class="message-data" v-if="chat.messageType=='audio'">
                        <audio alt="chat.messageName" controls>
                            <source :src="chat.message" type="audio" />
                        </audio>
                    </div>
            </div>
            </div>
          </el-scrollbar>
          <div>
            <el-input type="text" v-model="userData.message" @keyup.enter="sendMessage()">
                <template #append>
                    <el-button @click="sendMessage()">发送</el-button>
                </template>
            </el-input>
             <el-upload
             action="https://element-plus.gitee.io/zh-CN/" 
             v-model:file-list="fileList"
             :limit="1"
             :on-exceed="handleExceed"
             :auto-upload="false"
             >
                <el-button>上传文件</el-button>
            </el-upload>
          </div>
            
      </el-card>
      </el-col>
      </el-row>
  </div>
  <div v-else class="frosted-glass">
    <div class="title">
        SAKURA
        <br/>
        <input id="login" v-model="userData.userName" type="text" @keyup.enter="handleLogin()"/>
    </div>
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
// 用户信息
const userData = ref({
  userName:'',
  receiverName: '',
  connected: false,
  message: ''
})
// 上传文件
const fileList = ref([]);
// 默认用户信息
const defaultData = {}
onMounted(()=>{
  Object.assign(defaultData,userData.value);
})
// 用户列表
const userList = useStorage('userList',new Map());
// 私聊信息
const privateChats = new Map();
// 公聊信息
const publicChats = useStorage('publicChats',[]);
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
    // 订阅公聊
    stompClient.subscribe('/chatroom/public', onPublicMessageReceived);
    // 订阅私聊
    stompClient.subscribe('/user/'+userData.value.userName+'/private', onPrivateMessageReceived);
    // 用户加入聊天室
    userJoin();
}
// 链接失败回调
const onError = (err) => {
    console.log(err);    
}
// 用户进入聊天室
const userJoin = ()=>{
  let message = {
    senderName: userData.value.userName,
    messageType: "text",
    status:"JOIN"
  };
  userList.value.set(userData.value.userName,"🈺");
  // 初始化私聊的用户列表
  for(const[key,value] of userList.value.entries()){
    if(!privateChats.get(key)) privateChats.set(key,[]); 
  }
  // 通知WebSockert
  stompClient.send("/app/message",{},JSON.stringify(message));
}
// 收到公聊消息
const onPublicMessageReceived = (payload)=>{
  let payloadData = JSON.parse(payload.body);
  console.log("收到公聊消息",payloadData);
  switch(payloadData.status){
    case "JOIN":
      // 初始化私聊用户列表
      if(!privateChats.get(payloadData.senderName)){
        privateChats.set(payloadData.senderName,[]);
      }
      userList.value.set(payloadData.senderName,"🈺");
      break;
    case "MESSAGE":
      publicChats.value.push(payloadData);
      break;
    case "LEAVE":
      console.log("改变状态")
      userList.value.set(payloadData.senderName,"");
  }
}
// 收到私聊信息
const onPrivateMessageReceived = (payload)=>{
  let payloadData = JSON.parse(payload.body);
  console.log("收到私聊消息",payloadData);
  if(privateChats.get(payloadData.senderName)){
    privateChats.set(payloadData.senderName, [...privateChats.get(payloadData.senderName)])
  }else{
    let list = [];
    list.push(payloadData);
    privateChats.set(payload.senderName,list);
  }
}
// 发送消息
const sendMessage = ()=>{
    if(!userData.value.message&&fileList.value[0]===undefined){
      ElMessage.warning("消息不能为空");
      return;
    }
    if(stompClient){
      // 消息
        if(userData.value.message){
            let message = {
                senderName: userData.value.userName,
                message: userData.value.message,
                messageType: "text",
                status:"MESSAGE"
            };
            console.log("onSendMessage",message);
            if(tab.value=="public"){
              stompClient.send("/app/message",{},JSON.stringify(message));
            }
            else{
              let sendMessage = message;
              sendMessage.receiverName = tab.value;
              privateChats.set(tab.value,[...privateChats.get(tab.value),sendMessage]);
              stompClient.send("/app/private-message",{},JSON.stringify(sendMessage));
            }
            userData.value.message = "";
        }
        // 文件
        if(fileList.value[0]===undefined){
            console.log("没有上传文件");
        }
        else{
          let fileMessage={
          senderName: userData.value.userName,
          status:"MESSAGE",
          messageName:fileList.value[0].raw.name,
          }
          const reader = new FileReader();
          // 读取后回调
          reader.onload = (evt)=>{
            const content = evt.target.result;
            console.log("CONTENT",content);
            fileMessage.message = content;
            if(fileList.value[0].raw.type.match('image')){
              fileMessage.messageType = "image";
            console.log("图片已上传",fileMessage);
            }
            else if(fileList.value[0].raw.type.match('video')){
              fileMessage.messageType = "video";
              console.log("视频已上传",fileMessage);
            }
            else if(fileList.value[0].raw.type.match('audio')){
              fileMessage.messageType = "audio";
              console.log("音频已上传",fileMessage);
            }
            else{
              fileMessage.messageType = "file";
              console.log("文件已上传",fileMessage);
            }
            // 公聊私聊
            if(tab.value=="public"){
              stompClient.send("/app/message", {}, JSON.stringify(fileMessage));
            }
            else{
              fileMessage.receiverName = tab.value;
              console.log("私聊上传",tab.value);
              privateChats.set(tab.value,[...privateChats.get(tab.value),fileMessage]);
              stompClient.send("/app/private-message", {}, JSON.stringify(fileMessage));
            }
          }
          reader.readAsDataURL(fileList.value[0].raw)
        }
    }
}
// 处理超过一个文件上传的情况
const handleExceed = ()=>{
    ElMessage.warning("一次只能发送一个文件")
}
// 下载文件
const downloadFile = (content, name)=>{
    const url = window.URL.createObjectURL(new Blob([JSON.stringify(content)]));
    const fileLink = document.createElement('a');
    fileLink.href = url;
    fileLink.setAttribute('download', name); 
    document.body.appendChild(fileLink);
    fileLink.click();
}
// reset便于测试
const reset = ()=>{
  publicChats.value = null;
  publicChats = useStorage('publicChats',[]);
}
</script>

<style>
#body{
    width:60vw;
    margin:0 auto;
    margin-top:10vh;
}

#login{
    outline-style: none ;
    border:none;
    border-bottom: 3px solid rgb(255, 255, 255);
    font-size:4vh;
    color: rgb(255, 255, 255);
    text-align: center;
    font-family: "Microsoft soft";
    background-color: transparent;
    width:90%;
}

.message{
  padding:5px;
  width: auto;
  display: flex;
  flex-direction: row;
  box-shadow: 0 3px 10px rgb(0 0 0 / 0.2);
  margin: 5px 10px;
}
.message-data{
  padding:5px;
}
.message.self{
  justify-content: end;
}
.avatar{
  background-color: cornflowerblue;
  padding: 3px 5px;
  border-radius: 5px;
  color:#fff;
  width:30%;
}

.avatar-me{
  background-color: gold;
  padding: 3px 5px;
  border-radius: 5px;
  color:#fff;
  width:30%;
}
.card-title{
  text-align: center;
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
    width: 72vw;
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
:deep() .el-input__inner{
    border: none;
    background-color: transparent;
}
:deep()  .el-card__body {
  height: 40vh;
}

</style>