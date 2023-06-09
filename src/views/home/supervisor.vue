<template>
  <div style="display: flex; flex-direction: column">
    <div style="display: flex">
      <div>
        <div style="max-width: 700px; display: flex">
          <supervisor-info
            :current-count="currentMaxConsultantCount"
            @on-change="handleChange"
          />
          <supervisor-statistic
            :today-consultant-number="todayConsultantNumber"
            :today-consultant-time="todayConsultantTime"
          />
        </div>
        <consultant-online />
      </div>
      <schedule-calendar :arrangement-info="arrangementInfo" />
    </div>
    <div style="display: flex">
      <supervisor-recently :help-records="helpRecords" />
    </div>
  </div>
</template>

<script setup lang="ts">
import ScheduleCalendar from "@/views/home/components/schedule-calendar.vue";
import SupervisorInfo from "@/views/home/components/supervisor/supervisor-info.vue";
import SupervisorStatistic from "@/views/home/components/supervisor/supervisor-statistic.vue";
import SupervisorRecently from "@/views/home/components/supervisor/supervisor-recently.vue";
import { computed, onMounted, ref } from "vue";
import {
  ConversationInfo,
  HelpRecordsResp
} from "@/apis/conversation/conversation-interface";
import {
  getMaxConversations,
  getRecentHelps,
  getTodayHelps
} from "@/apis/conversation/conversation";
import { personalArrangement } from "@/apis/arrangement/arrangement";
import { parseTime } from "@/utils";
import ConsultantOnline from "@/views/home/components/consultant-online.vue";

const conversationInfo = ref<ConversationInfo[]>([]);
const helpRecords = ref<HelpRecordsResp>();
const arrangementInfo = ref<number[]>([]);
let currentMaxConsultantCount = ref(0);

onMounted(async () => {
  conversationInfo.value = await getTodayHelps();
  helpRecords.value = await getRecentHelps();
  arrangementInfo.value = await personalArrangement();
  currentMaxConsultantCount.value = await getMaxConversations();
});

let todayConsultantNumber = computed(() => {
  return conversationInfo.value.length;
});

let todayConsultantTime = computed(() => {
  const rawTime = conversationInfo.value.reduce((before, item) => {
    return before + item.endTime - item.startTime;
  }, 0);
  return parseTime(rawTime / 1000);
});

const handleChange = async () => {
  currentMaxConsultantCount.value = await getMaxConversations();
};
</script>

<style scoped lang="scss"></style>
