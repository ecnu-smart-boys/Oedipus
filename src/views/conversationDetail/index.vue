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
            咨询记录
          </div>
          <div style="margin: 10px 0; font-size: 20px">总共用时</div>
          <div style="margin: 10px 0; font-size: 30px">
            {{
              parseTime(
                (allInfo?.consultationInfo.lastTime -
                  allInfo?.consultationInfo.startTime) /
                  1000
              )
            }}
          </div>
          <div style="margin: 10px 0; font-size: 20px">咨询者评价</div>
          <el-rate
            v-model="rate"
            size="large"
            disabled
            style="transform: scale(1.5) translateX(18px)"
          />
          <div style="margin: 10px 0; font-size: 15px; color: #d7d7d7">
            {{ allInfo?.visitorText }}
          </div>
          <div style="margin: 10px 0; font-size: 20px">咨询师评价</div>
          <el-tag v-if="allInfo?.tag.length > 0" type="info">{{
            allInfo?.tag
          }}</el-tag>
          <div style="margin: 10px 0; font-size: 15px; color: #d7d7d7">
            {{ allInfo?.consultantText }}
          </div>
        </template>
        <template #bottom>
          <el-divider />
          <el-button
            size="large"
            :icon="Collection"
            style="margin: 0 20px; font-size: 20px"
            color="#337ecc"
            @click="handleExport"
          >
            导出记录
          </el-button>
          <el-divider />
          <el-button
            size="large"
            :icon="Back"
            style="margin: 0 20px; font-size: 20px"
            color="#337ecc"
            @click="handleBack"
          >
            返回列表
          </el-button>
        </template>
      </conversation-info>
    </div>
    <div class="chat-wrapper">
      <div ref="leftChatAreaWrapper" class="chat-list-wrapper">
        <chat-area
          ref="leftChatArea"
          :current-message="
            (allMsg?.consultation ?? []).map((i) =>
              messageAdapter(i, <string>allInfo?.consultationInfo.consultantId)
            )
          "
          :has-revoke="false"
          :should-loop="true"
          :my-avatar="allInfo?.consultationInfo.consultantAvatar"
          :other-avatar="allInfo?.consultationInfo.visitorAvatar"
        />
      </div>
    </div>
    <div v-if="allInfo?.helpInfo !== null" class="chat-wrapper">
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
              {{ allInfo?.helpInfo?.supervisorName }}
            </div>
            <div>求助记录</div>
            <div style="font-size: 30px">
              {{
                parseTime(
                  (allInfo?.helpInfo?.endTime - allInfo?.helpInfo?.startTime) /
                    1000
                )
              }}
            </div>
          </div>
        </template>
      </supervisor-to-consultant>
      <div ref="rightChatAreaWrapper" class="chat-list-wrapper">
        <chat-area
          ref="rightChatArea"
          :current-message="
            (allMsg?.help ?? []).map((i) =>
              messageAdapter(i, <string>allInfo?.consultationInfo.consultantId)
            )
          "
          :has-revoke="false"
          :should-loop="true"
          :my-avatar="allInfo?.consultationInfo.consultantAvatar"
          :other-avatar="allInfo?.helpInfo?.avatar"
        />
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import SupervisorToConsultant from "@/views/conversation/components/supervisor-to-consultant.vue";
import ConversationInfo from "@/views/conversation/components/conversation-info.vue";
import ChatArea from "@/imComponent/components/chatArea/index.vue";
import { nextTick, onMounted, ref, watch, watchEffect } from "vue";
import { Back, Collection } from "@element-plus/icons-vue";
import router from "@/router";
import {
  getAdminConsultationInfo,
  getBoundConsultantsInfo,
  getConsultantOwnConsultationInfo,
  getSupervisorOwnHelpInfo
} from "@/apis/conversation/conversation";
import { useRoute } from "vue-router";
import createStore from "@/store";
import { WebConversationInfoResp } from "@/apis/conversation/conversation-interface";
import { messageAdapter, mosaic, parseTime, preExport } from "@/utils";
import {
  AllMessageReq,
  AllMsgListResp,
  MessageInfo
} from "@/apis/message/message-interface";
import {
  getAdminConsultationMsg,
  getBoundConsultantsMsg,
  getConsultantOwnConsultationMsg,
  getSupervisorOwnHelpMsg
} from "@/apis/message/message";
import useScroll from "@/hooks/useScroll";
import JSZip from "jszip";
import FileSaver from "file-saver";
import { ElMessageBox } from "element-plus";
import { p } from "@/utils/data";

const route = useRoute();
const store = createStore();

const leftChatAreaWrapper: any = ref(null);
const rightChatAreaWrapper: any = ref(null);
// 迭代器用于懒加载
let consultationIterator = ref(-1);
let helpIterator = ref(-1);

const {
  isReachTop: isLeftReachTop,
  clientHeight: leftClientHeight,
  scrollHeight: leftScrollHeight,
  reflow: leftReflow,
  setScrollTop: setLeftScrollTop
} = useScroll(leftChatAreaWrapper);
const {
  isReachTop: isRightReachTop,
  clientHeight: rightClientHeight,
  scrollHeight: rightScrollHeight,
  reflow: rightReflow,
  setScrollTop: setRightScrollTop
} = useScroll(rightChatAreaWrapper);

watchEffect(async () => {
  if (isLeftReachTop.value && consultationIterator.value != 0) {
    // 触发懒加载
    const data = await getMsg();
    allMsg.value?.consultation.unshift(...data.consultation);
    // 保证滚动条还在同一位置
    await nextTick(() => {
      const oldScrollHeight = leftScrollHeight.value;
      leftReflow();
      setLeftScrollTop(leftScrollHeight.value - oldScrollHeight);
    });
  }
});

watchEffect(async () => {
  if (isRightReachTop.value && helpIterator.value != 0) {
    // 触发懒加载
    const data = await getMsg();
    allMsg.value?.help?.unshift(...data.help);
    // 保证滚动条还在同一位置
    await nextTick(() => {
      const oldScrollHeight = rightScrollHeight.value;
      rightReflow();
      setRightScrollTop(rightScrollHeight.value - oldScrollHeight);
    });
  }
});

// 所有信息
let allInfo = ref<WebConversationInfoResp>();
let allMsg = ref<AllMsgListResp>();

const rate = ref(0);
watchEffect(() => {
  if (allInfo.value) {
    const value = allInfo.value as WebConversationInfoResp;
    rate.value = value.visitorScore;
  }
});
const handleExport = async () => {
  await ElMessageBox.confirm(p, "同意数据保密使用协议", {
    confirmButtonText: "同意",
    cancelButtonText: "不同意",
    type: "warning"
  });
  let conversationInfo: WebConversationInfoResp;
  let allMsgList: AllMsgListResp;
  const allMsgReq: AllMessageReq = {
    conversationId: (route.query as any).conversationId,
    consultationIterator: -1,
    helpIterator: -1,
    size: 100000
  };
  if ((route.query as any).from == "help") {
    allMsgList = await getSupervisorOwnHelpMsg(allMsgReq);
    conversationInfo = await getSupervisorOwnHelpInfo(
      (route.query as any).conversationId
    );
  } else if ((route.query as any).from === "list") {
    if (store.role === "supervisor") {
      allMsgList = await getSupervisorOwnHelpMsg(allMsgReq);
      conversationInfo = await getSupervisorOwnHelpInfo(
        (route.query as any).conversationId
      );
    } else if (store.role === "consultant") {
      allMsgList = await getConsultantOwnConsultationMsg(allMsgReq);
      conversationInfo = await getConsultantOwnConsultationInfo(
        (route.query as any).conversationId
      );
    }
  } else {
    if (store.role === "supervisor") {
      allMsgList = await getBoundConsultantsMsg(allMsgReq);
      conversationInfo = await getBoundConsultantsInfo(
        (route.query as any).conversationId
      );
    } else if (store.role === "consultant") {
      allMsgList = await getConsultantOwnConsultationMsg(allMsgReq);
      conversationInfo = await getConsultantOwnConsultationInfo(
        (route.query as any).conversationId
      );
    } else if (store.role === "admin") {
      allMsgList = await getAdminConsultationMsg(allMsgReq);
      conversationInfo = await getAdminConsultationInfo(
        (route.query as any).conversationId
      );
    }
  }
  const data = preExport(conversationInfo, allMsgList);
  const zip = new JSZip();
  zip.file(
    `${conversationInfo.consultationInfo.consultantName}-${conversationInfo.consultationInfo.visitorName}.txt`,
    data[0]
  );
  if (data.length > 1) {
    zip.file(
      `${conversationInfo.consultationInfo.consultantName}-${conversationInfo.helpInfo?.supervisorName}.txt`,
      data[1]
    );
  }
  zip.generateAsync({ type: "blob" }).then((content) => {
    FileSaver(content, `${new Date().getTime()}.zip`);
  });
};

const handleBack = () => {
  router.go(-1);
};

const pageSize = 15;

const getInfo = async () => {
  let conversationInfo: WebConversationInfoResp;
  if ((route.query as any).from === "help") {
    conversationInfo = await getSupervisorOwnHelpInfo(
      (route.query as any).conversationId
    );
  } else if ((route.query as any).from === "list") {
    if (store.role === "supervisor") {
      conversationInfo = await getSupervisorOwnHelpInfo(
        (route.query as any).conversationId
      );
    } else if (store.role === "consultant") {
      conversationInfo = await getConsultantOwnConsultationInfo(
        (route.query as any).conversationId
      );
    }
  } else {
    if (store.role === "supervisor") {
      conversationInfo = await getBoundConsultantsInfo(
        (route.query as any).conversationId
      );
    } else if (store.role === "consultant") {
      conversationInfo = await getConsultantOwnConsultationInfo(
        (route.query as any).conversationId
      );
    } else if (store.role === "admin") {
      conversationInfo = await getAdminConsultationInfo(
        (route.query as any).conversationId
      );
    }
  }
  allInfo.value = conversationInfo;
};

const getMsg = async (): Promise<AllMsgListResp> => {
  let allMsgList: AllMsgListResp;
  const allMsgReq: AllMessageReq = {
    conversationId: (route.query as any).conversationId,
    consultationIterator: consultationIterator.value,
    helpIterator: helpIterator.value,
    size: pageSize
  };
  if ((route.query as any).from === "help") {
    allMsgList = await getSupervisorOwnHelpMsg(allMsgReq);
  } else if ((route.query as any).from === "list") {
    if (store.role === "supervisor") {
      allMsgList = await getSupervisorOwnHelpMsg(allMsgReq);
    } else if (store.role === "consultant") {
      allMsgList = await getConsultantOwnConsultationMsg(allMsgReq);
    }
  } else {
    if (store.role === "supervisor") {
      allMsgList = await getBoundConsultantsMsg(allMsgReq);
    } else if (store.role === "consultant") {
      allMsgList = await getConsultantOwnConsultationMsg(allMsgReq);
    } else if (store.role === "admin") {
      allMsgList = await getAdminConsultationMsg(allMsgReq);
    }
  }
  if (allMsgList.consultation.length > 0) {
    consultationIterator.value = allMsgList.consultation[0].iterator;
  }

  if (allMsgList.callHelp) {
    const help = <MessageInfo[]>allMsgList.help;
    if (help.length > 0) {
      helpIterator.value = help[0].iterator;
    }
  }
  return allMsgList;
};

watch(route, async () => {
  if (!(route.query as any).conversationId) return;
  if (route.path !== "/conversation-detail") return;
  await getInfo();
  allMsg.value = await getMsg();
  // 滑动到最底部
  await nextTick(() => {
    leftReflow();
    setLeftScrollTop(leftScrollHeight.value - leftClientHeight.value);
    rightReflow();
    setRightScrollTop(rightScrollHeight.value - rightClientHeight.value);
  });
});

watch(route, async () => {
  if (!(route.query as any).conversationId) return;
  if (route.path !== "/conversation-detail") return;
  consultationIterator.value = -1;
  helpIterator.value = -1;
});

onMounted(async () => {
  await getInfo();
  allMsg.value = await getMsg();
  // 滑动到最底部
  await nextTick(() => {
    leftReflow();
    setLeftScrollTop(leftScrollHeight.value - leftClientHeight.value);
    rightReflow();
    setRightScrollTop(rightScrollHeight.value - rightClientHeight.value);
  });
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
<style>
.el-divider--horizontal {
  margin: 10px 0;
}
</style>
