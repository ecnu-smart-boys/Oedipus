<template>
  <el-card class="box-card">
    <el-row style="height: 200px; align-items: center">
      <el-col :span="8">
        <el-statistic title="今日咨询数" :value="todayConsultantNumber" />
      </el-col>
      <el-col :span="8">
        <el-statistic title="今日咨询时长" :value="todayConsultantTime" />
      </el-col>
      <el-col :span="8">
        <el-statistic title="当前会话数" :value="todayConsultantSession" />
      </el-col>
    </el-row>
  </el-card>
</template>

<script setup lang="ts">
defineProps<{
  todayConsultantNumber: number;
  todayConsultantTime: string;
  todayConsultantSession: number;
}>();
</script>

<style scoped lang="scss">
.box-card {
  min-width: 500px;
  margin: 20px 10px 20px 0;

  .el-col {
    text-align: center;
  }

  .el-statistic {
    --el-statistic-content-font-size: 2.5em;
    --el-statistic-title-font-size: 1em;
  }
}
</style>
<style>
.el-statistic__head {
  margin-bottom: 15px;
}
</style>
