<template>
  <div class="app-container">
    <!-- 搜索表单 -->
    <el-form :model="queryParams" ref="queryRef" :inline="true" v-show="showSearch">
      <el-form-item label="课程名称" prop="courseName">
        <el-input
            v-model="queryParams.courseName"
            placeholder="请输入课程名称"
            clearable
            style="width: 200px"
            @keyup.enter="handleQuery"
        />
      </el-form-item>
      <el-form-item label="课程作者" prop="courseAuthor">
        <el-input
            v-model="queryParams.courseAuthor"
            placeholder="请输入课程作者"
            clearable
            style="width: 200px"
            @keyup.enter="handleQuery"
        />
      </el-form-item>
      <el-form-item>
        <el-button type="primary" icon="Search" @click="handleQuery">搜索</el-button>
        <el-button icon="Refresh" @click="resetQuery">重置</el-button>
      </el-form-item>
    </el-form>

    <el-row :gutter="10" class="mb8">
      <el-col :span="1.5">
        <el-button
            type="primary"
            plain
            icon="Plus"
            @click="handleAdd"
            v-hasPermi="['system:course:add']"
        >新增</el-button>
      </el-col>
      <el-col :span="1.5">
        <el-button
            type="success"
            plain
            icon="Edit"
            :disabled="single"
            @click="handleUpdate"
            v-hasPermi="['system:course:edit']"
        >修改</el-button>
      </el-col>
      <el-col :span="1.5">
        <el-button
            type="danger"
            plain
            icon="Delete"
            :disabled="multiple"
            @click="handleDelete"
            v-hasPermi="['system:course:remove']"
        >删除</el-button>
      </el-col>
      <el-col :span="1.5">
        <el-button
            type="warning"
            plain
            icon="Download"
            @click="handleExport"
            v-hasPermi="['system:course:export']"
        >导出</el-button>
      </el-col>
      <right-toolbar v-model:showSearch="showSearch" @queryTable="getList"></right-toolbar>
    </el-row>

    <el-table v-loading="loading" :data="courseList" @selection-change="handleSelectionChange">
      <el-table-column type="selection" width="55" align="center" />
      <el-table-column label="课程编号" align="center" prop="courseId" />
      <el-table-column label="课程名称" align="center" prop="courseName" />
      <el-table-column label="课程作者" align="center" prop="courseAuthor" />
      <el-table-column label="课程排序" align="center" prop="courseSort" />
      <el-table-column label="创建时间" align="center" prop="createTime" width="180">
        <template #default="scope">
          <span>{{ parseTime(scope.row.createTime) }}</span>
        </template>
      </el-table-column>
      <el-table-column label="操作" width="180" align="center" class-name="small-padding fixed-width">
        <template #default="scope">
          <el-button link type="primary" icon="Edit" @click="handleUpdate(scope.row)" v-hasPermi="['system:course:edit']">修改</el-button>
          <el-button link type="primary" icon="Delete" @click="handleDelete(scope.row)" v-hasPermi="['system:course:remove']">删除</el-button>
        </template>
      </el-table-column>
    </el-table>

    <pagination
        v-show="total > 0"
        :total="total"
        v-model:page="queryParams.pageNum"
        v-model:limit="queryParams.pageSize"
        @pagination="getList"
    />

    <!-- 添加或修改课程对话框 -->
    <el-dialog :title="title" v-model="open" width="500px" append-to-body>
      <el-form ref="courseRef" :model="form" :rules="rules" label-width="80px">
        <el-form-item label="课程名称" prop="courseName">
          <el-input v-model="form.courseName" placeholder="请输入课程名称" />
        </el-form-item>
        <el-form-item label="课程简介" prop="courseDescription">
          <el-input v-model="form.courseDescription" placeholder="请输入课程简介" />
        </el-form-item>
        <el-form-item label="课程作者" prop="courseAuthor">
          <el-input v-model="form.courseAuthor" placeholder="请输入课程作者" />
        </el-form-item>
        <el-form-item label="课程封面" prop="courseCover">
          <!-- 课程封面上传组件 -->
          <el-upload
              multiple
              :before-upload="handleImageUpload"
              list-type="picture-card"
              :file-list="imageFileList"
              :on-remove="handleImageRemove"
              :auto-upload="true"
              :on-preview="handlePictureCardPreview"
          >
            <el-icon class="avatar-uploader-icon"><plus /></el-icon>
          </el-upload>
        </el-form-item>
        <el-form-item label="课程视频" prop="courseVideo">
          <!-- 课程视频上传组件 -->
          <el-upload
              multiple
              :before-upload="handleVideoUpload"
              list-type="picture-card"
              :file-list="videoFileList"
              :on-remove="handleVideoRemove"
              :auto-upload="true"
          >
            <el-icon class="avatar-uploader-icon"><plus /></el-icon>
          </el-upload>
        </el-form-item>
        <el-form-item label="课程排序" prop="courseSort">
          <el-input-number v-model="form.courseSort" controls-position="right" :min="0" />
        </el-form-item>
        <el-form-item label="备注" prop="remark">
          <!-- 富文本编辑器组件 -->
          <quill-editor
              ref="quillEditorRef"
              v-model:content="form.remark"
              contentType="html"
              :options="options"
              :style="styles"
          />
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

<script setup name="Course">
import { listCourse, addCourse, delCourse, getCourse, updateCourse } from "@/api/system/course";
import { ref, reactive, toRefs, getCurrentInstance } from "vue";
import { QuillEditor } from '@vueup/vue-quill';
import '@vueup/vue-quill/dist/vue-quill.snow.css';

const { proxy } = getCurrentInstance();

const courseList = ref([]);
const open = ref(false);
const loading = ref(true);
const showSearch = ref(true);
const ids = ref([]);
const single = ref(true);
const multiple = ref(true);
const total = ref(0);
const title = ref("");
const imageFileList = ref([]);
const videoFileList = ref([]);
const quillEditorRef = ref(null);
const styles = ref({
  height: '400px',
});

const data = reactive({
  form: {
    courseId: null,
    courseName: '',
    courseDescription: '',
    courseCover: '',
    courseVideo: '',
    courseAuthor: '',
    courseSort: 0,
    remark: ''
  },
  queryParams: {
    pageNum: 1,
    pageSize: 10,
    courseName: undefined,
    courseAuthor: undefined
  },
  rules: {
    courseName: [{ required: true, message: "课程名称不能为空", trigger: "blur" }],
    courseAuthor: [{ required: true, message: "课程作者不能为空", trigger: "blur" }],
  }
});

const { queryParams, form, rules } = toRefs(data);

/** 查询课程列表 */
function getList() {
  loading.value = true;
  listCourse(queryParams.value).then(response => {
    courseList.value = response.rows;
    total.value = response.total;
    loading.value = false;
  });
}
/** 取消按钮 */
function cancel() {
  open.value = false;
  reset();
}
/** 表单重置 */
function reset() {
  form.value = {
    courseId: null,
    courseName: '',
    courseDescription: '',
    courseCover: '',
    courseVideo: '',
    courseAuthor: '',
    courseSort: 0,
    remark: ''
  };
  imageFileList.value = [];
  videoFileList.value = [];
  proxy.resetForm("courseRef");
}
function handleTextChange({ editor }) {
  form.value.remark = editor.root.innerHTML;  // 将编辑器内容转换为HTML格式
}

/** 搜索按钮操作 */
function handleQuery() {
  queryParams.value.pageNum = 1;
  getList();
}
/** 重置按钮操作 */
function resetQuery() {
  proxy.resetForm("queryRef");
  handleQuery();
}
/** 多选框选中数据 */
function handleSelectionChange(selection) {
  ids.value = selection.map(item => item.courseId);
  single.value = selection.length != 1;
  multiple.value = !selection.length;
}
/** 新增按钮操作 */
function handleAdd() {
  reset();
  open.value = true;
  title.value = "添加课程";
}
/** 修改按钮操作 */
function handleUpdate(row) {
  reset();
  const courseId = row.courseId || ids.value;
  getCourse(courseId).then(response => {
    form.value = response.data;
    open.value = true;
    title.value = "修改课程";
  });
}
/** 提交按钮 */
/** 提交按钮 */
function submitForm() {
  proxy.$refs["courseRef"].validate(valid => {
    if (valid) {
      // 确保备注内容为HTML字符串
      const quillInstance = quillEditorRef.value && quillEditorRef.value.getQuill();
      if (quillInstance) {
        form.value.remark = quillInstance.root.innerHTML;
      }
      console.log("Form data being submitted:", form.value);  // 检查提交的数据

      // 构建简化的数据对象，只包含必要的属性
      const submitData = {
        courseId: form.value.courseId,
        courseName: form.value.courseName,
        courseDescription: form.value.courseDescription,
        courseCover: form.value.courseCover,
        courseVideo: form.value.courseVideo,
        courseAuthor: form.value.courseAuthor,
        courseSort: form.value.courseSort,
        remark: form.value.remark
      };

      let jsonForm = JSON.stringify(submitData);
      if (form.value.courseId != null) {
        updateCourse(jsonForm).then(response => {
          proxy.$modal.msgSuccess("修改成功");
          open.value = false;
          getList();
        });
      } else {
        addCourse(jsonForm).then(response => {
          proxy.$modal.msgSuccess("新增成功");
          open.value = false;
          getList();
        });
      }
    }
  });
}


/** 删除按钮操作 */
function handleDelete(row) {
  const courseIds = row.courseId || ids.value;
  proxy.$modal.confirm('是否确认删除课程编号为"' + courseIds + '"的数据项？').then(function() {
    return delCourse(courseIds);
  }).then(() => {
    getList();
    proxy.$modal.msgSuccess("删除成功");
  }).catch(() => {});
}
/** 导出按钮操作 */
function handleExport() {
  proxy.download("system/course/export", {
    ...queryParams.value
  }, `course_${new Date().getTime()}.xlsx`);
}
/** 图片上传处理 */
function handleImageUpload(file) {
  const reader = new FileReader();
  reader.onload = (e) => {
    // 删除旧的封面
    imageFileList.value = [];
    form.value.courseCover = '';

    // 添加新的封面
    form.value.courseCover = e.target.result;  // 将Base64字符串存储到表单字段中
    imageFileList.value.push({ name: file.name, url: e.target.result });
    console.log('Image uploaded, form.courseCover:', form.value.courseCover);  // 添加调试代码
  };
  reader.readAsDataURL(file);
  return false;  // 阻止默认的上传行为
}

/** 图片移除处理 */
function handleImageRemove(file, fileList) {
  imageFileList.value = fileList;
  if (fileList.length === 0) {
    form.value.courseCover = '';  // 如果移除所有图片，清空表单字段
  }
}

/** 图片预览处理 */
function handlePictureCardPreview(file) {
  const imgWindow = window.open(file.url);
  imgWindow.document.write('<img src="' + file.url + '" />');
}

/** 视频上传处理 */
function handleVideoUpload(file) {
  const reader = new FileReader();
  reader.onload = (e) => {
    // 删除旧的视频
    videoFileList.value = [];
    form.value.courseVideo = '';

    // 添加新的视频
    form.value.courseVideo = e.target.result;  // 将Base64字符串存储到表单字段中
    videoFileList.value.push({ name: file.name, url: e.target.result });
    console.log('Video uploaded, form.courseVideo:', form.value.courseVideo);  // 添加调试代码
  };
  reader.readAsDataURL(file);
  return false;  // 阻止默认的上传行为
}

/** 视频移除处理 */
function handleVideoRemove(file, fileList) {
  videoFileList.value = fileList;
  if (fileList.length === 0) {
    form.value.courseVideo = '';  // 如果移除所有视频，清空表单字段
  }
}



getList();
</script>

<style scoped>
.upload-demo .el-icon-plus {
  font-size: 28px;
}
</style>
