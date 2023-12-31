<template>
  <v-container fluid class="subject-card-wrapper">
    <v-row class="topMenuBar" style="margin-bottom: 0px;">
      <v-col cols="2" class="d-flex justify-start">
        <v-img @click="$router.go(-1)" src="@/assets/back_arrow.png" class="back-arrow"></v-img> 
      </v-col>
      <v-col cols="8" class="text-center">
        <h3 class="grey--text">{{ otherName }}님과의 대화</h3>
      </v-col>
      <v-col cols="2">
      <v-spacer></v-spacer><v-spacer></v-spacer>
      <v-btn @click="openReportModal()" class="report-btn">
          신고하기
        </v-btn>
      </v-col>
    <!-- Add the modal for reporting -->
    <v-dialog v-model="reportDialog" max-width="290" persistent>
      <v-card>
        <v-card-title class="headline">신고 사유 선택</v-card-title>
        <v-card-text>
          <v-radio-group v-model="reportReason" row>
            <v-radio label="부적절한 내용" value="2"></v-radio>
            <v-radio label="욕설" value="3"></v-radio>
            <v-radio label="스팸" value="4"></v-radio>
            <v-radio label="기타" value="5"></v-radio>
          </v-radio-group>
        </v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn color="blue darken-1" text @click="reportDialog = false">취소</v-btn>
          <v-btn color="blue darken-1" text @click="createReport">제출</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
   
     
    </v-row>
    <v-card>
      <v-card-actions class="topic-btn">
        <v-spacer>생각주제</v-spacer>
        <v-btn :icon="show ? 'mdi-chevron-up' : 'mdi-chevron-down'" @click="show = !show"></v-btn>
      </v-card-actions>

      <v-expand-transition>
        <div v-show="show">
          <v-divider></v-divider>
          <v-card-text>
            {{ todaySubjectContent }}
          </v-card-text>
        </div>
      </v-expand-transition>
    </v-card>

   <br>

    <!-- Chat messages -->
    <v-row>
      <v-col cols="12" class="d-flex justify-center">
        <v-card class="chat-card-wrapper" style="overflow-y: auto;">
          <v-card-text>
            <!-- message -->
            <v-row v-for="message in messages" :key="message.id">
              <template v-if="message.content != 'messageDate'">
                <v-col cols="12">
                  <!-- Removed the v-divider element -->
                  <div class="d-flex" style="align-items: center;" :class="message.sender.nickname === myName ? 'justify-end' : 'justify-start'">
                    <div v-if="message.sender.nickname !== myName">
                      <v-img class="align-end text-white" :src=findImage(message.userImgPath) cover rounded
                            style="border-radius: 50%; width: 40px; height: 40px;">
                      </v-img>
                    </div>
                    <div>
                      <v-card class="mx-2"
                        :class="message.sender.nickname === myName ? 'chat-message-yellow' : 'chat-message-mint'" tile>
                        <v-card-text class="text-box">
                          {{ message.content }}
                        </v-card-text>
                      </v-card>
                    </div>
                    <div v-if="message.sender.nickname === myName">
                      <v-img class="align-end text-white" :src=findImage(message.userImgPath) cover rounded
                            style="border-radius: 50%; width: 40px; height: 40px;">
                      </v-img>
                    </div>
                  </div>
                  <div>
                    <div class="d-flex grey--text"
                      :class="message.sender.nickname === myName ? 'justify-end' : 'justify-start'">{{
                        message.sender.nickname }} {{ message.time }}</div>
                  </div>
                </v-col>
              </template>
              <template v-else>
                <v-col cols=12>
                  <div class="date-div">{{ message.time }}</div>
                </v-col>
              </template>
            </v-row>
          </v-card-text>
        </v-card>
      </v-col>
    </v-row>

    <!-- Message input -->
    <v-row class="textArea">
      <v-col cols="12">
        <v-textarea  variant="filled" no-resize="True" v-model="userMessage" outlined placeholder="메시지 입력" class="mb-2 message-input"
        @keyup.shift.enter="inputBlank" @keyup.enter="sendMessage"></v-textarea>
      </v-col>
    </v-row>
    <v-row class="sendButtonRow">
      <v-col cols="12" class="d-flex justify-end">
        <v-btn color="#FBF0A0" dark @click="sendMessage" class="send-btn">전송</v-btn>
      </v-col>
    </v-row>

    <!-- Add this button after the '전송' button -->
    <!-- <v-btn color="#FBF0A0" dark @click="receiveMessage">Receive</v-btn> -->

  </v-container>

  <v-snackbar
  v-model="showSnackbar"
  color="success"
  :timeout="3000"
  class="snackbar-center"
  @click="showSnackbar = false">
  <span class="snackbar-content">신고가 완료되었습니다.</span>
</v-snackbar>
</template>
  
<script>
export default {
  name: "ChatRoom"
}
</script>


<script setup>
import { ref, nextTick, onMounted, onBeforeUnmount} from "vue";
import Stomp from 'stompjs';
import { useUserStore } from "@/store/user";
import { useChatStore } from "@/store/chat";
import { $getChatMessages } from '@/api/chat';
import { $createReport } from "@/api/report";
import { useSubjectStore } from "@/store/subject";
import { format } from 'date-fns'
import { findImage } from "@/api/index";
import { getChatSocket } from "@/api/socket";

const userStore = useUserStore()
const subjectStore = useSubjectStore()
const chatStore = useChatStore()
const todaySubjectContent = subjectStore.getSubject.content


const show = ref(false);
const reportDialog = ref(false);
const reportReason = ref('');
const otherName = ref(''); // 상대방 닉네임
const myName = userStore.getLoginUser.nickname; // 나의 닉네임
const messages = ref([]); // 채팅 메시지 배열
const userMessage = ref(""); // 전송하려는 메세지 콘텐츠
const chatRoomNo = ref(chatStore.getChatRoom.chatRoomNo); // 채팅방 이름을 저장할 ref 변수
reportDialog.value = false;


const getCurrentTime = () => {
  return format(new Date(), 'yyyy-MM-dd HH:mm:ss');

  // const hours = now.getHours();
  // const minutes = now.getMinutes();
  // const ampm = hours >= 12 ? "pm" : "am";
  // const hour = hours % 12;
  // const time = `${ampm} ${hour}:${minutes < 10 ? "0" + minutes : minutes}`;
  // return time;
};

async function scrollToLatestMessage() {
  await nextTick()
  const container = document.querySelector(".chat-card-wrapper");
  container.scrollTop = container.scrollHeight;
}

function openReportModal() {
  reportDialog.value = true;
}

const showSnackbar = ref(false); // 스낵바

async function createReport() {
  // console.log("신고하기 값", chatRoomNo.value, reportReason.value);
  await $createReport(chatRoomNo.value, reportReason.value)
    .then(res => {
      // console.log(res.data);
      reportDialog.value = false; // 성공 시 reportDialog 닫기
      showSnackbar.value = true; // 성공 시 Snackbar 표시
    })
    .catch(err => console.log(err));
}

const chatDate = ref('')

const chatHistory = ref(null)

function loadChatHistory() {
  $getChatMessages(chatRoomNo.value)
    .then(res => {
      chatHistory.value = res.data
      // console.log(res.data)
      // for문을 돌면서 해당 메세지의 sendUserNo 이 `나` 일 경우 오른쪽,
      // 상대방일 경우 왼쪽에 추가
      chatHistory.value.forEach(chatMsg => {
        let checkDate = checkedDate(chatMsg.chatCreateAt)
        if(chatDate.value != checkDate) {
          chatDate.value = checkDate
          messages.value.push({
            content: 'messageDate',
            time: checkDate
          });
        }
        if (myName == chatMsg.sendNickname) {
          messages.value.push({
            sender: { nickname: myName, avatarUrl: "" },
            content: chatMsg.chatMsgContent,
            time: formattedDate(chatMsg.chatCreateAt),
            userImgPath : userStore.getLoginUser.userImgPath
          });
        } else {
          // console.log(otherName)
          // otherName.value = chatMsg.sendNickname
          messages.value.push({
            sender: { nickname: chatMsg.sendNickname, avatarUrl: "" },
            content: chatMsg.chatMsgContent,
            time: formattedDate(chatMsg.chatCreateAt),
            userImgPath : chatMsg.userImgPath
          });
        }
      });
      scrollToLatestMessage();
    })
    .catch(err => console.log(err))
}


// WebSocket 연결을 담을 ref 변수
const stompClient = ref(null);

// 채팅방에 대한 구독(subscribe)을 담을 ref 변수
const subscription = ref(null);
const socket = getChatSocket()
const stomp = Stomp.over(socket);

function formattedDate(createAt) {
  return format(new Date(createAt), 'HH:mm');
}

function checkedDate(createAt) {
  return format(new Date(createAt), 'yyyy년 MM월 dd일');
}

// WebSocket 연결 생성 함수
function createWebSocketConnection() {

  stomp.connect({}, () => {
    stompClient.value = stomp;

    if (myName == chatStore.getChatRoom.fromNickname) {
      otherName.value = chatStore.getChatRoom.toNickname
    } else {
      otherName.value = chatStore.getChatRoom.fromNickname
    }
    console.log(myName, otherName.value)
    // 과거의 채팅 기록 조회
    loadChatHistory()


    // 채팅방 구독(subscribe) 요청
    subscription.value = stomp.subscribe(`/sub/chat/room/${chatRoomNo.value}`, (res) => {
      // console.log(res);
      const chatMsg = JSON.parse(res.body); // 구독하게 되면 받아오게 되는 메세지
      // console.log(chatMsg);
      const writer = chatMsg.sendNickname;

      let checkDate = checkedDate(chatMsg.chatCreateAt)
        if(chatDate.value != checkDate) {
          chatDate.value = checkDate
          messages.value.push({
            content: 'messageDate',
            time: checkDate
          });
        }
      if (myName == writer) {
        const currentTime = getCurrentTime();
        messages.value.push({
          sender: { nickname: myName, avatarUrl: "" },
          content: chatMsg.chatMsgContent,
          time: formattedDate(currentTime),
          userImgPath : userStore.getLoginUser.userImgPath
        });
      } else {
        otherName.value = chatMsg.sendNickname
        const currentTime = getCurrentTime();
        messages.value.push({
          sender: { nickname: writer, avatarUrl: "" },
          content: chatMsg.chatMsgContent,
          time: formattedDate(currentTime),
          userImgPath : chatMsg.userImgPath
        });
      }
      scrollToLatestMessage();
    });

    //3. send(path, header, message)로 메세지를 보낼 수 있음
    const currentTime = getCurrentTime();
    const sendData = JSON.stringify({
      chatRoomNo: chatRoomNo.value,
      chatSendUserNo: userStore.getLoginUser.userNo,
      sendNickname: myName,
      chatCreateAt: currentTime
    })
    stomp.send('/pub/chat/enter', {}, sendData)
  });
}

const enterSwitch = ref(true)
function inputBlank() {
  enterSwitch.value = false
}

const sendMessage = () => {
  if(!enterSwitch.value) {
        enterSwitch.value = true
        return 
  }

  // userMessage가 비어있으면 함수를 종료합니다.
  if (userMessage.value == null || userMessage.value.trim() == "") {
    userMessage.value = ''
    // console.log(userMessage.value)
    return;

  } else {
    // console.log(userMessage.value)
    // messages 배열에 새로운 메시지를 추가합니다.
    // const currentTime = getCurrentTime();
    // console.log(myName + ":" + userMessage.value);
    const sendData = JSON.stringify({
      chatRoomNo: chatRoomNo.value,
      chatSendUserNo: userStore.getLoginUser.userNo,
      sendNickname: myName,
      chatMsgContent: userMessage.value,
      userImgPath : userStore.getLoginUser.userImgPath
    })
    userMessage.value = '';
    // console.log(sendData)
    stomp.send('/pub/chat/message', {}, sendData)
  }
  // 스크롤을 최신 메시지로 이동시킵니다.
  scrollToLatestMessage();
};


// 컴포넌트가 마운트되면 WebSocket 연결 생성 함수 실행
onMounted(async () => {
  await createWebSocketConnection();
});


onBeforeUnmount(() => {
    if (stompClient.value) {
      stompClient.value.disconnect();
    }
})

// function findImage(userImg) {
//   if (userImg) {
//     // console.log(userImg);
//     const convertedPath = userImg.replace(/\\/g, '/');
//     return `http://localhost:8080/onedaythink/api/v1/imgfind/userImg?userImgPath=${convertedPath}`;
//   } else {
//     const defaultImg = 'src/main/resources/static/profileImages/default.png'
//     return `http://localhost:8080/onedaythink/api/v1/imgfind/userImg?userImgPath=${defaultImg}`;
//   }
// }

</script>

<style scoped>
.nickname {
  font-family: "IBM Plex Sans", sans-serif;
  font-size: 13px;
  line-height: 17px;
  color: #000000;
}

.textArea{
  margin-right: 1px;
}
.chat-card-wrapper {
  width: 100%;
  height: 300px;
  overflow-y: auto;
  margin-right: 12px;
}

.chat-message-mint {
  background-color: #d3f1ef;
  border-radius: 6px;
}

.chat-message-yellow {
  background-color: #fff8abe6;
  border-radius: 6px;
}

.chat-card-wrapper::-webkit-scrollbar {
  width: 8px;
}

.chat-card-wrapper::-webkit-scrollbar-thumb {
  background-color: #B5B5B5;
  border-radius: 4px;
}

.chat-card-wrapper::-webkit-scrollbar-track {
  background-color: #F5F5F5;
}

/* 뒤로가기 버튼 스타일  */
.back-arrow {
  width: 10px;
  height: 20px;
  cursor: pointer;
}

/* 신고하기 버튼 스타일 */
.report-btn {
  background-color: #ffa1a1;
  color: white;
  border-radius: 5px;
}

/* 생각주제 버튼 스타일 */
.topic-btn {
  background-color: #fdfdef;
  color: #2C2C2C;
  border-radius: 3px;
}

/* 메시지 입력창 스타일 */
.message-input {
  background-color: #F5F5F5;
  border: 1px solid #E0E0E0;
  border-radius: 10px;
}

/* 전송 버튼 스타일 */
.send-btn {
  background-color: #feed00;
  color: #2C2C2C;
  border-radius: 5px;
  margin-right: 10px;
}

.topMenuBar{
  margin-right: 30px;
}

.sendButtonRow{
    margin-top: -25px;
 }

 /* 스크롤바 관련 설정 */
.subject-card-wrapper {
  height: 600px;
  overflow-y: auto;
  margin-top: 10px;
  margin-bottom: 60px;
}

/* 스크롤바 숨기기 */
.subject-card-wrapper::-webkit-scrollbar {
  display: none;
}
/* .v-main{
    --v-layout-top: 0px !important; 
} */

.date-div {
  display: flex;
  justify-content: center;
  align-items: center;
  background-color: #f2f2f2;
  font-weight: bold;
  font-size: 14px;
  color: #555555;
  border-radius:80px;
}

.custom-snackbar {
    background-color: #43a047; 
    color: white;  
    position: absolute;  
    top: 50%;  
    left: 50%; 
    transform: translate(-50%, -50%); 
    padding: 16px;
    border-radius: 5px;
    margin-bottom: 20px;
    z-index: 1000; 
    white-space: pre-wrap; 
    text-align: center; 
    display: flex;       
    align-items: center; 
    justify-content: center; 
}

.slide-enter-active, .slide-leave-active {
  transition: all .3s ease;
}
.slide-enter, .slide-leave-to {
  opacity: 0;
  transform: translateY(30px);
}
.snackbar-content {
  margin: auto;
}
.snackbar-center {
  position: fixed;
  left: 50%;
  transform: translateX(-50%);
  bottom: auto !important;
  top: 50%;
}
.text-box {
  padding: 0.5rem;
}

</style>