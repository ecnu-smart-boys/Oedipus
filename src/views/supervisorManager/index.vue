<template>
  <div>
    <div
      style="display: flex; justify-content: space-between; align-items: center"
    >
      <div
        style="
          display: flex;
          flex-direction: column;
          align-items: center;
          margin: 0 20px;
        "
      >
        <div style="margin-bottom: 10px">搜索姓名</div>
        <el-input
          v-model="searchName"
          placeholder="输入姓名进行搜索"
          :prefix-icon="Search"
        />
      </div>
      <el-button type="primary" @click="addDialogVisible = true"
        >添加督导</el-button
      >
    </div>
    <el-table v-loading="isLoading" :data="tableData" style="width: 100%">
      <el-table-column fixed prop="name" label="姓名" width="150">
        <template #default="scope">
          <div style="display: flex; align-items: center">
            <img
              :src="scope.row.avatar"
              class="avatar"
              onerror="this.src='/defaultAvatar.jpg'"
            />
            <div style="margin-left: 10px">{{ scope.row.name }}</div>
          </div>
        </template>
      </el-table-column>
      <el-table-column prop="gender" label="性别" width="100" />
      <el-table-column prop="duration" label="总咨询时长" width="150" />
      <el-table-column prop="count" label="总咨询次数" width="150" />
      <el-table-column prop="consultant" label="绑定咨询师" width="280" />
      <el-table-column prop="schedule" label="排班" width="280" />
      <el-table-column prop="state" label="状态" width="120" />
      <el-table-column fixed="right" label="操作" width="180">
        <template #default="scope">
          <el-button
            link
            type="primary"
            size="small"
            @click="handleEdit(scope.row)"
            >修改</el-button
          >
          <el-button
            link
            type="primary"
            size="small"
            @click="handleSchedule(scope.row)"
            >排班</el-button
          >
          <el-button
            link
            type="primary"
            size="small"
            @click="handleDisable(scope.row)"
            >{{ scope.row.state == "正常" ? "禁用" : "启用" }}</el-button
          >
        </template>
      </el-table-column>
    </el-table>
    <el-pagination
      v-model:current-page="currentPage"
      v-model:page-size="pageSize"
      style="float: right; margin: 20px"
      background
      layout="prev, pager, next, jumper"
      :total="totalPage"
      @current-change="handleCurrentChange"
    />
    <el-dialog v-model="addDialogVisible" title="添加督导" width="700px">
      <form-add-employee
        ref="addSupervisorForm"
        :is-consultant="false"
        @on-submit="handleAddSubmit"
      />
      <template #footer>
        <span class="dialog-footer">
          <el-button @click="addDialogVisible = false"> 取消 </el-button>
          <el-button type="primary" @click="submitAddForm"> 确定 </el-button>
        </span>
      </template>
    </el-dialog>
    <el-dialog v-model="editDialogVisible" title="修改督导" width="700px">
      <form-edit-employee
        ref="editSupervisorForm"
        :edit-data="editData"
        :is-consultant="false"
        @on-submit="handleEditSubmit"
      />
      <template #footer>
        <span class="dialog-footer">
          <el-button @click="editDialogVisible = false"> 取消 </el-button>
          <el-button type="primary" @click="submitEditForm"> 确定 </el-button>
        </span>
      </template>
    </el-dialog>
    <el-dialog v-model="scheduleDialogVisible" title="修改排班" width="700px">
      <form-schedule
        ref="scheduleSupervisorForm"
        :schedule-data="scheduleData"
      />
      <template #footer>
        <span class="dialog-footer">
          <el-button @click="scheduleDialogVisible = false"> 取消 </el-button>
          <el-button type="primary" @click="submitScheduleForm">
            确定
          </el-button>
        </span>
      </template>
    </el-dialog>
  </div>
</template>

<script setup lang="ts">
import { Search } from "@element-plus/icons-vue";
import FormAddEmployee from "@/components/form-add-employee.vue";
import { onMounted, reactive, ref, watch } from "vue";
import FormSchedule from "@/components/form-schedule.vue";
import FormEditEmployee from "@/components/form-edit-employee.vue";
import { ElMessage, ElMessageBox } from "element-plus";
import {
  addSupervisor,
  getSupervisors,
  updateSupervisor
} from "@/apis/userArrange/supervisor/superivisor";
import { md5, parseSchedule, parseTime, toBoolArraySchedule } from "@/utils";
import { FormData } from "@/components/schema";
import { disable, enable, updateArrangement } from "@/apis/userArrange/user";

let isLoading = ref(false);
let searchName = ref("");

interface FormSupervisor {
  id: string;
  name: string;
  avatar: string;
  gender: string;
  count: number;
  duration: string;
  consultant: string;
  _schedule: number;
  schedule: string;
  state: string;

  age: number;
  idNumber: string;
  workPlace: string;
  title: string;
  qualification: string;
  qualificationNumber: string;
}
watch(
  () => searchName.value,
  async () => {
    currentPage.value = 1;
    await refreshData();
  }
);

let currentPage = ref(1);
let pageSize = ref(10);
let totalPage = ref(10);
const handleCurrentChange = async (val) => {
  currentPage.value = val;
  await refreshData();
};

const addDialogVisible = ref(false);
const editDialogVisible = ref(false);
const scheduleDialogVisible = ref(false);

const tableData: FormSupervisor[] = reactive([]);

const addSupervisorForm: any = ref(null);
const editSupervisorForm: any = ref(null);
const scheduleSupervisorForm: any = ref(null);

const submitAddForm = async () => {
  try {
    await addSupervisorForm.value.submitForm();
  } catch (e) {
    return;
  }
};

const handleAddSubmit = async (data: FormData) => {
  await addSupervisor({
    age: Number(data.age),
    department: data.workPlace,
    email: data.email ? data.email : "",
    gender: data.gender == "男" ? 1 : 2,
    idNumber: data.idNumber,
    name: data.name,
    password: md5(data.password ? data.password : ""),
    phone: data.phone ? data.phone : "",
    qualification: data.qualification,
    qualificationCode: data.qualificationNumber,
    title: data.title,
    username: data.username ? data.username : ""
  });
  addDialogVisible.value = false;
  addSupervisorForm.value.changeToDefault();
  ElMessage({
    message: "添加成功",
    type: "success",
    duration: 5 * 1000
  });
  await refreshData();
};

let editData: FormData = reactive({
  id: "",
  name: "",
  gender: "",
  age: "",
  idNumber: "",
  supervisor: null,
  workPlace: "",
  title: "",
  qualification: "",
  qualificationNumber: ""
});

const handleEdit = (row) => {
  editData.id = row.id;
  editData.name = row.name;
  editData.gender = row.gender;
  editData.age = row.age;
  editData.idNumber = row.idNumber;
  editData.workPlace = row.workPlace;
  editData.title = row.title;
  editData.qualification = row.qualification;
  editData.qualificationNumber = row.qualificationNumber;
  editDialogVisible.value = true;
};

const submitEditForm = async () => {
  try {
    await editSupervisorForm.value.submitForm();
  } catch (e) {
    return;
  }
};

const handleEditSubmit = async (data) => {
  await updateSupervisor({
    id: data.id,
    age: Number(data.age),
    department: data.workPlace,
    gender: data.gender == "男" ? 1 : 2,
    idNumber: data.idNumber,
    name: data.name,
    qualification: data.qualification,
    qualificationCode: data.qualificationNumber,
    title: data.title
  });
  editDialogVisible.value = false;
  ElMessage({
    message: "修改成功",
    type: "success",
    duration: 5 * 1000
  });
  await refreshData();
};

let scheduleData = reactive<{
  schedule: any[];
  id: string;
}>({
  schedule: [],
  id: ""
});
const handleSchedule = (row) => {
  scheduleData = {
    schedule: toBoolArraySchedule(row._schedule),
    id: row.id
  };
  scheduleDialogVisible.value = true;
};

const submitScheduleForm = async () => {
  await updateArrangement({
    id: scheduleData.id,
    arrangement: scheduleSupervisorForm.value.getScheduleData()
  });
  ElMessage({
    message: "修改成功",
    type: "success",
    duration: 5 * 1000
  });
  scheduleDialogVisible.value = false;
  await refreshData();
};

const handleDisable = async (row) => {
  await ElMessageBox.confirm(
    `确定${row.state == "正常" ? "禁用" : "启用"}该咨询师吗？`,
    "警告",
    {
      confirmButtonText: "确定",
      cancelButtonText: "取消",
      type: "warning"
    }
  );
  if (row.state == "正常") {
    await disable({
      id: row.id
    });
  } else {
    await enable({
      id: row.id
    });
  }
  await refreshData();
};

const refreshData = async () => {
  isLoading.value = true;
  const data = await getSupervisors({
    current: currentPage.value,
    name: searchName.value,
    size: pageSize.value
  });
  totalPage.value = data.total;
  tableData.splice(0);
  data.supervisors.forEach((c) => {
    const i = {
      id: c.id,
      name: c.name,
      avatar: c.avatar,
      gender: c.gender == 1 ? "男" : "女",
      count: c.consultTimes,
      duration: parseTime(c.totalTime),
      consultant: c.consultantList.map((i) => i.name).join(", "),
      _schedule: c.arrangement,
      schedule: parseSchedule(c.arrangement),
      state: c.disabled ? "禁用" : "正常",
      age: c.age,
      idNumber: c.idNumber,
      workPlace: c.department,
      title: c.title,
      qualification: c.qualification,
      qualificationNumber: c.qualificationCode
    };
    tableData.push(i);
  });
  isLoading.value = false;
};
onMounted(async () => {
  await refreshData();
});
</script>

<style scoped lang="scss">
.avatar {
  border-radius: 50%;
  height: 30px;
}
</style>
