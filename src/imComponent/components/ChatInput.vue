<template>
  <div class="ChatInput">
    <div class="header">
      <SendFace @on-face="handleFace" />
      <SendImage :to-id="toId" @on-send="handleSend" />
      <SendAudio :to-id="toId" @on-send="handleSend" />
    </div>
    <div class="body">
      <div class="input-wrapper">
        <textarea v-model="currentMessage"></textarea>
      </div>
      <div class="btn-wrapper">
        <el-button class="btn" type="primary" @click="submitTextMsg"
          >发送</el-button
        >
      </div>
    </div>
  </div>
</template>
<script lang="ts" setup>
import SendAudio from "@/imComponent/components/SendAudio.vue";
import SendFace from "./SendFace.vue";
import SendImage from "@/imComponent/components/SendImage.vue";
import { ref } from "vue";
import { imSendMessage } from "@/apis/im/im";
import { createTextMessage } from "@/imComponent/utils/createMessage";
import { ElMessage } from "element-plus";

const props = defineProps<{
  toId: string;
}>();

let currentMessage = ref("");
const emits = defineEmits<{
  (event: "onSend", data: any): void;
}>();
const submitTextMsg = async () => {
  try {
    const data = await imSendMessage(
      createTextMessage(props.toId, currentMessage.value)
    );
    emits("onSend", data);
    currentMessage.value = "";
  } catch (error) {
    ElMessage({
      message: error,
      type: "error",
      duration: 5 * 1000
    });
  }
};
const handleFace = (item) => {
  currentMessage.value = currentMessage.value + item;
};

const handleSend = (data) => {
  emits("onSend", data);
};
</script>
<style lang="scss" scoped>
.ChatInput {
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
  .header {
    height: 40px;
    border-bottom: 1px solid #eee;
    display: flex;
    > div {
      width: 40px;
      line-height: 40px;
      text-align: center;
      cursor: pointer;
      color: #999;
    }
  }
  .body {
    flex-grow: 1;
    position: relative;
    display: flex;
    .input-wrapper {
      flex: 1;
    }
    .btn-wrapper {
      width: 100px;
      position: relative;
      .btn {
        position: absolute;
        right: 10px;
        bottom: 10px;
      }
    }
    .input-wrapper {
      position: relative;
      textarea {
        font-family: "Helvetica Neue", Helvetica, "PingFang SC",
          "Hiragino Sans GB", "Microsoft YaHei", "微软雅黑", Arial, sans-serif;
        position: absolute;
        top: 0;
        bottom: 0;
        left: 0;
        right: 0;
        resize: none;
        width: 100%;
        border: none;
        outline: none;
        padding: 8px;
        color: #3d3d3d;
        font-size: 16px;
      }
    }
  }
}
</style>
