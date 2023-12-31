<!-- 广告管理 -->
<script setup lang="ts">
defineOptions({
  name: "Advert",
  inheritAttrs: false,
});

import {
  getAdvertPage,
  getAdvertForm,
  updateAdvert,
  addAdvert,
  deleteAdverts,
} from "@/api/sms/advert";
import { AdvertQuery, Advert, AdvertForm } from "@/api/sms/advert/types";

const queryFormRef = ref(ElForm); // 属性名必须和元素的ref属性值一致
const dataFormRef = ref(ElForm); // 属性名必须和元素的ref属性值一致

const state = reactive({
  loading: true,
  // 选中ID数组
  ids: [],
  // 非单个禁用
  single: true,
  // 非多个禁用
  multiple: true,
  queryParams: { pageNum: 1, pageSize: 10 } as AdvertQuery,
  advertList: [] as Advert[],
  total: 0,
  dialog: { title: "", visible: false } as DialogType,
  formData: {
    status: 1,
    sort: 100,
  } as AdvertForm,
  rules: {
    title: [{ required: true, message: "请输入广告名称", trigger: "blur" }],
    picUrl: [{ required: true, message: "请上传广告图片", trigger: "blur" }],
  },
  validityPeriod: "" as any,
});

const {
  loading,
  multiple,
  queryParams,
  advertList,
  total,
  dialog,
  formData,
  rules,
  validityPeriod,
} = toRefs(state);

function handleQuery() {
  state.loading = true;
  getAdvertPage(state.queryParams).then(({ data }) => {
    state.advertList = data.list;
    state.total = data.total;
    state.loading = false;
  });
}

function resetQuery() {
  queryFormRef.value.resetFields();
  handleQuery();
}

function handleSelectionChange(selection: any) {
  state.ids = selection.map((item: any) => item.id);
  state.single = selection.length !== 1;
  state.multiple = !selection.length;
}

function handleAdd() {
  state.dialog = {
    title: "添加广告",
    visible: true,
  };
}

function handleUpdate(row: any) {
  state.dialog = {
    title: "修改广告",
    visible: true,
  };
  const advertId = row.id || state.ids;
  getAdvertForm(advertId).then(({ data }) => {
    state.formData = data;
    validityPeriod.value = [data.beginTime, data.endTime];
  });
}

function submitForm() {
  dataFormRef.value.validate((valid: any) => {
    if (valid) {
      const avertId = state.formData.id;
      if (avertId) {
        // 有效期转换
        if (validityPeriod.value) {
          formData.value.beginTime = validityPeriod.value[0];
          formData.value.endTime = validityPeriod.value[1];
        }

        updateAdvert(avertId, state.formData).then(() => {
          ElMessage.success("修改成功");
          cancel();
          handleQuery();
        });
      } else {
        addAdvert(state.formData).then(() => {
          ElMessage.success("新增成功");
          cancel();
          handleQuery();
        });
      }
    }
  });
}

function cancel() {
  state.formData.id = undefined;
  dataFormRef.value.resetFields();
  state.dialog.visible = false;
}

function handleDelete(row: any) {
  const ids = [row.id || state.ids].join(",");
  ElMessageBox.confirm("确认删除已选中的数据项?", "警告", {
    confirmButtonText: "确定",
    cancelButtonText: "取消",
    type: "warning",
  })
    .then(() => {
      deleteAdverts(ids).then(() => {
        ElMessage.success("删除成功");
        handleQuery();
      });
    })
    .catch(() => ElMessage.info("已取消删除"));
}

onMounted(() => {
  handleQuery();
});
</script>

<template>
  <div class="app-container">
    <!-- 搜索表单 -->
    <div class="search-container">
      <el-form ref="queryFormRef" :model="queryParams" :inline="true">
        <el-form-item>
          <el-button type="success" @click="handleAdd"
            ><i-ep-plus />新增</el-button
          >
          <el-button type="danger" :disabled="multiple" @click="handleDelete"
            ><i-ep-delete />删除</el-button
          >
        </el-form-item>

        <el-form-item prop="title">
          <el-input
            v-model="queryParams.keywords"
            placeholder="标题"
            clearable
            @keyup.enter="handleQuery"
          />
        </el-form-item>
        <el-form-item>
          <el-button type="primary" @click="handleQuery">
            <i-ep-search /> 搜索</el-button
          >
          <el-button @click="resetQuery"><i-ep-refresh /> 重置</el-button>
        </el-form-item>
      </el-form>
    </div>

    <el-table
      v-loading="loading"
      :data="advertList"
      border
      @selection-change="handleSelectionChange"
    >
      <el-table-column type="selection" min-width="5" align="center" />
      <el-table-column type="index" label="序号" width="80" align="center" />
      <el-table-column prop="title" min-width="100" label="广告标题" />
      <el-table-column label="广告图片" width="100">
        <template #default="scope">
          <el-popover placement="right" :width="400" trigger="hover">
            <img :src="scope.row.picUrl" width="400" height="400" />
            <template #reference>
              <img
                :src="scope.row.picUrl"
                style="max-width: 60px; max-height: 60px"
              />
            </template>
          </el-popover>
        </template>
      </el-table-column>
      <el-table-column prop="beginTime" label="开始时间" width="150" />
      <el-table-column prop="endTime" label="结束时间" width="150" />
      <el-table-column prop="status" label="状态" width="100">
        <template #default="scope">
          <el-tag v-if="scope.row.status === 1" type="success">开启</el-tag>
          <el-tag v-else type="info">关闭</el-tag>
        </template>
      </el-table-column>
      <el-table-column prop="sort" label="排序" width="80" />
      <el-table-column label="操作" align="center" width="150">
        <template #default="scope">
          <el-button
            type="primary"
            circle
            plain
            @click.stop="handleUpdate(scope.row)"
            ><i-ep-edit
          /></el-button>
          <el-button
            type="danger"
            circle
            plain
            @click.stop="handleDelete(scope.row)"
            ><i-ep-delete />
          </el-button>
        </template>
      </el-table-column>
    </el-table>

    <!-- 分页工具条 -->
    <pagination
      v-if="total > 0"
      v-model:page="queryParams.pageNum"
      v-model:limit="queryParams.pageSize"
      :total="total"
      @pagination="handleQuery"
    />

    <!-- 表单弹窗 -->
    <el-dialog v-model="dialog.visible" :title="dialog.title" width="700px">
      <el-form
        ref="dataFormRef"
        :model="formData"
        :rules="rules"
        label-width="100px"
      >
        <el-form-item label="广告标题" prop="title">
          <el-input v-model="formData.title" />
        </el-form-item>

        <el-form-item label="有效期">
          <el-date-picker
            v-model="validityPeriod"
            type="daterange"
            range-separator="~"
            start-placeholder="起始时间"
            end-placeholder="截止时间"
            format="YYYY-MM-DD"
            value-format="YYYY-MM-DD"
          />
        </el-form-item>

        <el-form-item label="广告图片" prop="picUrl">
          <single-upload v-model="formData.picUrl" />
        </el-form-item>

        <el-form-item label="排序" prop="sort">
          <el-input v-model="formData.sort" style="width: 200px" />
        </el-form-item>

        <el-form-item label="状态" prop="status">
          <el-radio-group v-model="formData.status">
            <el-radio :label="1">开启</el-radio>
            <el-radio :label="0">关闭</el-radio>
          </el-radio-group>
        </el-form-item>

        <el-form-item label="跳转链接" prop="url">
          <el-input v-model="formData.url" />
        </el-form-item>

        <el-form-item label="备注" prop="remark">
          <el-input v-model="formData.remark" type="textarea" />
        </el-form-item>
      </el-form>

      <template #footer>
        <div class="dialog-footer">
          <el-button type="primary" @click="submitForm">确 定</el-button>
          <el-button @click="cancel">取 消</el-button>
        </div>
      </template>
    </el-dialog>
  </div>
</template>
