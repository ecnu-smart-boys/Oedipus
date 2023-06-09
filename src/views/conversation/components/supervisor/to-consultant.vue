<template>
  <div style="display: flex">
    <div>
      <conversation-info>
        <template #head>
          <img
            :src="allInfo?.consultationInfo.visitorAvatar"
            class="avatar"
            alt=""
            onerror="this.src='/defaultAvatar.jpg'"
          />
          <div
            style="
              margin-left: 20px;
              display: flex;
              flex-direction: column;
              justify-content: center;
            "
          >
            <div style="font-size: 25px">
              {{ allInfo?.consultationInfo.visitorName }}
            </div>
            <div>{{ mosaic(allInfo?.consultationInfo.phone) }}</div>
          </div>
        </template>
        <template #middle>
          <div style="margin: 10px 0; font-size: 20px; font-weight: bold">
            {{ allInfo?.consultationInfo.end ? "咨询已完成" : "正在咨询中" }}
          </div>
          <div style="margin: 10px 0; font-size: 20px">已咨询时间</div>
          <div style="margin: 10px 0; font-size: 40px">
            {{
              parseTime(
                (currentTime - allInfo?.consultationInfo.startTime) / 1000
              )
            }}
          </div>
        </template>
      </conversation-info>
    </div>
    <im-component
      v-if="!leftIsEnd"
      :is-left="true"
      :is-end="leftIsEnd"
      :to-id="<string>allInfo?.consultationInfo.consultantId"
      :other-avatar="allInfo?.consultationInfo.consultantAvatar"
      :my-avatar="allInfo?.helpInfo?.avatar"
    >
      <template #left>
        <supervisor-to-consultant
          :is-show-btn="leftHelpBtnShown"
          @on-stop="handleStopHelp"
        >
          <template #left>
            <img
              :src="allInfo?.helpInfo?.avatar"
              class="avatar"
              alt=""
              onerror="this.src='/defaultAvatar.jpg'"
            />
            <div
              style="
                margin-left: 20px;
                display: flex;
                flex-direction: column;
                justify-content: center;
              "
            >
              <div style="font-size: 25px">
                {{ allInfo?.consultationInfo.consultantName }}
              </div>
              <div>
                {{ allInfo?.helpInfo?.end ? "求助已完成" : "正在求助中" }}
              </div>
              <div style="font-size: 30px">
                {{
                  parseTime(
                    (currentHelpTime - allInfo?.helpInfo?.startTime) / 1000
                  )
                }}
              </div>
            </div>
          </template>
        </supervisor-to-consultant>
      </template>
    </im-component>
    <div v-if="leftIsEnd" class="chat-wrapper">
      <supervisor-to-consultant :is-show-btn="false">
        <template #left>
          <img
            :src="allInfo?.helpInfo?.avatar"
            class="avatar"
            alt=""
            onerror="this.src='/defaultAvatar.jpg'"
          />
          <div
            style="
              margin-left: 20px;
              display: flex;
              flex-direction: column;
              justify-content: center;
            "
          >
            <div style="font-size: 25px">
              {{ allInfo?.consultationInfo.consultantName }}
            </div>
            <div>
              {{ allInfo?.helpInfo?.end ? "求助已完成" : "正在求助中" }}
            </div>
            <div style="font-size: 30px">
              {{
                parseTime(
                  (currentHelpTime - allInfo?.helpInfo?.startTime) / 1000
                )
              }}
            </div>
          </div>
        </template>
      </supervisor-to-consultant>
      <div class="chat-list-wrapper">
        <chat-area
          ref="leftChatArea"
          :current-message="
            (leftMsg?.help ?? []).map((i) =>
              messageAdapter(i, <string>allInfo?.helpInfo?.supervisorId)
            )
          "
          :has-revoke="false"
          :should-loop="true"
          :other-avatar="allInfo?.consultationInfo.consultantAvatar"
          :my-avatar="allInfo?.helpInfo?.avatar"
        />
      </div>
    </div>
    <div class="chat-wrapper">
      <div class="chat-list-wrapper">
        <chat-area
          ref="rightChatArea"
          :current-message="(allMsg?.consultation ?? []).map((i) =>
              messageAdapter(i, <string>allInfo?.consultationInfo.consultantId)
            )"
          :has-revoke="true"
          :should-loop="true"
          :my-avatar="allInfo?.consultationInfo.consultantAvatar"
          :other-avatar="allInfo?.consultationInfo.visitorAvatar"
        />
      </div>
    </div>
  </div>
</template>

<script lang="ts" setup>
import ChatArea from "@/imComponent/components/chatArea/index.vue";
import ConversationInfo from "@/views/conversation/components/conversation-info.vue";
import SupervisorToConsultant from "@/views/conversation/components/supervisor-to-consultant.vue";
import { nextTick, onMounted, onUnmounted, ref, watch, watchEffect } from "vue";
import ImComponent from "@/imComponent/im-component.vue";
import { messageAdapter, mosaic, parseTime } from "@/utils";
import { WebConversationInfoResp } from "@/apis/conversation/conversation-interface";
import { endHelp, getOnlineHelpInfo } from "@/apis/conversation/conversation";
import { AllMsgListResp, MsgListResp } from "@/apis/message/message-interface";
import { useRoute } from "vue-router";
import {
  EndConsultationNotification,
  EndHelpNotification,
  newMessageNotification,
  WebSocketResponse
} from "@/apis/schema";
import createStore from "@/store";
import {
  getSupervisorOwnHelpMsg,
  synchronizeMsg
} from "@/apis/message/message";
import useScroll from "@/hooks/useScroll";
import { deleteConversation, setMessageRead } from "@/apis/im/im";
import router from "@/router";

const route = useRoute();
const store = createStore();

const rightChatArea = ref(null);
let leftHelpBtnShown = ref(false);

const {
  isReachTop: isRightReachTop,
  clientHeight: rightClientHeight,
  scrollHeight: rightScrollHeight,
  reflow: rightReflow,
  setScrollTop: setRightScrollTop
} = useScroll(rightChatArea);

let timer;
let helpTimer;
let leftIsEnd = ref(false);
// 迭代器用于懒加载
let consultationIterator = ref(-1);
// 所有信息
let allInfo = ref<WebConversationInfoResp>();
let allMsg = ref<MsgListResp>();
let leftMsg = ref<AllMsgListResp>();

let currentTime = ref(new Date().getTime());
let currentHelpTime = ref(new Date().getTime());

const getMsg = async () => {
  const data = await synchronizeMsg({
    conversationId: <string>allInfo.value?.consultationInfo.consultationId,
    iterator: consultationIterator.value,
    size: 15
  });
  if (data.consultation.length > 0) {
    consultationIterator.value = data.consultation[0].iterator;
  }
  return data;
};

const getLeftMsg = async () => {
  return await getSupervisorOwnHelpMsg({
    conversationId: (route.query as any).conversationId,
    consultationIterator: -1,
    helpIterator: -1,
    size: 10000
  });
};
const refreshData = async (isEnd = false) => {
  timer && clearInterval(timer);
  helpTimer && clearInterval(helpTimer);
  try {
    const data = await getOnlineHelpInfo((route.query as any).conversationId);
    allInfo.value = data;
    if (data.helpInfo != null) {
      if (!data.helpInfo.end) {
        // 还没结束
        leftHelpBtnShown.value = true;
      } else {
        // 结束了
        leftHelpBtnShown.value = false;
        currentHelpTime.value = <number>data.helpInfo?.endTime;
        if (leftMsg.value) {
          const data = await getLeftMsg();
          if (data.help && leftMsg.value?.help) {
            (leftMsg.value as AllMsgListResp).help?.push(...data.help);
          }
          (leftMsg.value as AllMsgListResp).callHelp = data.callHelp;
        } else {
          leftMsg.value = await getLeftMsg();
        }
      }
    }
    if (data.consultationInfo.end) {
      // 结束了啥也不展示
      currentTime.value = data.consultationInfo.lastTime;
      currentHelpTime.value = <number>data.helpInfo?.endTime;
      leftIsEnd.value = true;
    } else {
      if (isEnd) {
        // 不需要开计时器了
        return;
      }
      timer = setInterval(() => {
        currentTime.value = new Date().getTime();
      }, 1000);
      if (!data.helpInfo?.end) {
        helpTimer = setInterval(() => {
          currentHelpTime.value = new Date().getTime();
        }, 1000);
      }
    }
  } catch (e) {
    leftIsEnd.value = true;
    currentTime.value = 0;
    currentHelpTime.value = 0;
  }
  allMsg.value = await getMsg();
};

watchEffect(async () => {
  if (isRightReachTop.value && consultationIterator.value != 0) {
    // 触发懒加载
    const data = await getMsg();
    allMsg.value?.consultation.unshift(...data.consultation);
    // 保证滚动条还在同一位置
    await nextTick(() => {
      const oldScrollHeight = rightScrollHeight.value;
      rightReflow();
      setRightScrollTop(rightScrollHeight.value - oldScrollHeight);
    });
  }
});

watch(route, async () => {
  if (!(route.query as any).conversationId) return;
  if (route.path !== "/conversation") return;
  consultationIterator.value = -1;
  await refreshData();
  await setMessageRead(
    `C2C${<string>allInfo.value?.consultationInfo.consultantId}`
  );
});

onMounted(async () => {
  await refreshData();
  // 滑动到最底部
  await nextTick(() => {
    rightReflow();
    setRightScrollTop(rightScrollHeight.value - rightClientHeight.value);
  });
  // await setMessageRead(
  //   `C2C${<string>allInfo.value?.consultationInfo.consultantId}`
  // );
  store.setLeftId(
    <string>allInfo.value?.helpInfo?.supervisorId,
    <string>allInfo.value?.consultationInfo.consultantId
  );
  store.setRightId(
    <string>allInfo.value?.consultationInfo.consultantId,
    <string>allInfo.value?.consultationInfo.visitorId
  );
});

const handleStopHelp = async () => {
  helpTimer && clearInterval(helpTimer);
  await endHelp({
    conversationId: <string>allInfo.value?.helpInfo?.helpId
  });
  // 剩下的工作交给ws
};

// ws相关
watchEffect(async () => {
  // 是对方结束会话
  if (store.websocketMessage != null) {
    const msg = store.websocketMessage as WebSocketResponse;
    if (msg.type === "endConsultation") {
      const content = msg.content as EndConsultationNotification;
      if (
        content.consultationId != allInfo.value?.consultationInfo.consultationId
      ) {
        return;
      }
      leftIsEnd.value = true;
      // 直接跳转到detail中
      await router.push({
        path: "/conversation-detail",
        query: {
          conversationId: allInfo.value?.helpInfo?.helpId,
          from: "help"
        }
      });
    } else if (msg.type === "endHelp") {
      const content = msg.content as EndHelpNotification;
      if (content.helpId != allInfo.value?.helpInfo?.helpId) {
        return;
      }
      await refreshData();
      leftIsEnd.value = true;
      leftHelpBtnShown.value = false;
      // 删除求助会话
      try {
        await deleteConversation(
          `C2C${allInfo?.value?.consultationInfo.consultantId}`
        );
      } catch (e) {
        /* empty */
      }
      store.setWebSocketMessage(null);
    } else if (msg.type === "newMsg") {
      const content = msg.content as newMessageNotification;
      if (content.helpId != allInfo.value?.helpInfo?.helpId) {
        return;
      }
      allMsg.value?.consultation.push(content.info);
      store.setWebSocketMessage(null);
    } else if (msg.type === "revoke") {
      const content = msg.content as newMessageNotification;
      if (content.helpId != allInfo.value?.helpInfo?.helpId) {
        return;
      }
      const it = content.info.iterator;
      for (let i = 0; i < allMsg.value?.consultation.length; i++) {
        if (allMsg.value?.consultation[i].iterator === it) {
          (allMsg.value as MsgListResp).consultation[i].revoked = true;
          break;
        }
      }
      store.setWebSocketMessage(null);
    } else if (msg.type === "comment") {
      const content = msg.content as string;
      if (content != allInfo.value?.consultationInfo?.consultationId) {
        return;
      }
      store.setWebSocketMessage(null);
      // 有comment说明已经结束了，直接跳转到detail中
      await router.push({
        path: "/conversation-detail",
        query: {
          conversationId: allInfo.value?.helpInfo?.helpId,
          from: "help"
        }
      });
    }
  }
});

onUnmounted(() => {
  timer && clearInterval(timer);
  helpTimer && clearInterval(helpTimer);
});
</script>

<style scoped lang="scss">
.chat-wrapper {
  display: flex;
  flex-direction: column;
  flex: 1;
  height: calc(100vh - 100px);
  border-right: 1px #e7e7e7 solid;
}
.chat-list-wrapper {
  background-color: white;
  flex: 8;
  overflow: auto;
  height: 500px;
  min-width: 300px;
  transition: all 1s;
  position: relative;
}
.chat-input-wrapper {
  flex: 1;
  min-height: 150px;
  position: relative;
  border-top: 1px solid #eee;
}
.avatar {
  width: 80px;
  height: 80px;
  border-radius: 50%;
}
</style>
