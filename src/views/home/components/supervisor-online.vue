<template>
  <el-card class="box-card" body-style="padding: 0;">
    <template #header>
      <div class="card-header">
        <span>在线督导</span>
        <el-pagination
          v-model:current-page="currentPage"
          v-model:page-size="pageSize"
          background
          layout="prev, pager, next"
          :total="totalPage"
          @current-change="handleCurrentChange"
        />
      </div>
    </template>
    <div style="display: flex; justify-content: space-between">
      <div
        style="
          display: flex;
          flex-direction: column;
          align-items: center;
          width: 50%;
        "
      >
        <div v-for="item in staffs" :key="item.userId" class="online-wrapper">
          <div>{{ item.name }}</div>
          <el-tag :type="item.state == 1 ? '' : 'danger'">{{
            item.state == 1 ? "空闲" : "忙碌"
          }}</el-tag>
        </div>
      </div>
      <div class="right-wrapper">
        <div style="font-size: 1.5em; margin: 10px">正在进行的督导会话</div>
        <div style="font-size: 2em">{{ liveConversations }}</div>
      </div>
    </div>
  </el-card>
</template>

<script setup lang="ts">
import { onMounted, reactive, ref } from "vue";
import { getOnlineSupervisorInfo } from "@/apis/conversation/conversation";

let currentPage = ref(1);
let pageSize = ref(3);
let totalPage = ref(3);
const handleCurrentChange = async (val) => {
  currentPage.value = val;
  await refreshData();
};

let liveConversations = ref(0);
const staffs = reactive<any[]>([]);
const refreshData = async () => {
  const data = await getOnlineSupervisorInfo({
    size: pageSize.value,
    current: currentPage.value
  });
  liveConversations.value = data.liveConversations;
  totalPage.value = data.total;
  staffs.splice(0);
  staffs.push(...data.staffs);
};
onMounted(async () => {
  await refreshData();
});
</script>

<style scoped lang="scss">
.box-card {
  height: 240px;
  min-width: 410px;
  margin: 0 10px 20px 10px;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.online-wrapper {
  display: flex;
  align-content: center;
  justify-content: center;
  margin: 17px 0;
}

.right-wrapper {
  display: flex;
  height: 172px;
  justify-content: center;
  align-items: center;
  flex-direction: column;
  background-color: #409dfd;
  color: white;
  width: 300px;
}
</style>
