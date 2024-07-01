<template>
   <div class="app-container">
     <el-form :model="queryParams" ref="queryRef" :inline="true" v-show="showSearch">
       <el-form-item label="租户名称" prop="tenantName">
         <el-input
           v-model="queryParams.tenantName"
           placeholder="请输入租户名称"
           clearable
           style="width: 200px"
           @keyup.enter="handleQuery"
         />
       </el-form-item>
       <el-form-item label="联系人" prop="contactPerson">
         <el-input
           v-model="queryParams.contactPerson"
           placeholder="请输入联系人"
           clearable
           style="width: 200px"
           @keyup.enter="handleQuery"
         />
       </el-form-item>
       <el-form-item label="联系电话" prop="phoneNumber">
         <el-input
           v-model="queryParams.phoneNumber"
           placeholder="请输入联系电话"
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
           v-hasPermi="['system:tenant:add']"
         >新增</el-button>
       </el-col>
       <el-col :span="1.5">
         <el-button
           type="success"
           plain
           icon="Edit"
           :disabled="single"
           @click="handleUpdate"
           v-hasPermi="['system:tenant:edit']"
         >修改</el-button>
       </el-col>
       <el-col :span="1.5">
         <el-button
           type="danger"
           plain
           icon="Delete"
           :disabled="multiple"
           @click="handleDelete"
           v-hasPermi="['system:tenant:remove']"
         >删除</el-button>
       </el-col>
       <el-col :span="1.5">
         <el-button
           type="warning"
           plain
           icon="Download"
           @click="handleExport"
           v-hasPermi="['system:tenant:export']"
         >导出</el-button>
       </el-col>
       <right-toolbar v-model:showSearch="showSearch" @queryTable="getList"></right-toolbar>
     </el-row>
 
     <el-table v-loading="loading" :data="tenantList" @selection-change="handleSelectionChange">
       <el-table-column type="selection" width="55" align="center" />
       <el-table-column label="租户编号" align="center" prop="tenantId" />
       <el-table-column label="租户名称" align="center" prop="tenantName" />
       <el-table-column label="联系人" align="center" prop="contactPerson" />
       <el-table-column label="联系电话" align="center" prop="phoneNumber" />
       <el-table-column label="管理员ID" align="center" prop="admin" />
       <el-table-column label="备注" align="center" prop="remark" />
       <el-table-column label="操作" width="180" align="center" class-name="small-padding fixed-width">
         <template #default="scope">
           <el-button link type="primary" icon="Edit" @click="handleUpdate(scope.row)" v-hasPermi="['system:tenant:edit']">修改</el-button>
           <el-button link type="primary" icon="Delete" @click="handleDelete(scope.row)" v-hasPermi="['system:tenant:remove']">删除</el-button>
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
 
     <!-- 添加或修改租户对话框 -->
     <el-dialog :title="title" v-model="open" width="500px" append-to-body>
       <el-form ref="tenantRef" :model="form" :rules="rules" label-width="80px">
         <el-form-item label="租户名称" prop="tenantName">
           <el-input v-model="form.tenantName" placeholder="请输入租户名称" />
         </el-form-item>
         <el-form-item label="联系人" prop="contactPerson">
           <el-input v-model="form.contactPerson" placeholder="请输入联系人" />
         </el-form-item>
         <el-form-item label="联系电话" prop="phoneNumber">
           <el-input v-model="form.phoneNumber" placeholder="请输入联系电话" />
         </el-form-item>
         <el-form-item label="管理员ID" prop="adminUserId">
           <el-input v-model="form.admin" placeholder="请输入管理员ID" />
         </el-form-item>
         <el-form-item label="备注" prop="remark">
           <el-input v-model="form.remark" type="textarea" placeholder="请输入内容" />
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
 
 <script setup name="Tenant">
 import { listTenant, addTenant, delTenant, getTenant, updateTenant } from "@/api/system/tenant";
 
 const { proxy } = getCurrentInstance();
 
 const tenantList = ref([]);
 const open = ref(false);
 const loading = ref(true);
 const showSearch = ref(true);
 const ids = ref([]);
 const single = ref(true);
 const multiple = ref(true);
 const total = ref(0);
 const title = ref("");
 
 const data = reactive({
   form: {},
   queryParams: {
     pageNum: 1,
     pageSize: 10,
     tenantName: undefined,
     contactPerson: undefined,
     phoneNumber: undefined
   },
   rules: {
     tenantName: [{ required: true, message: "租户名称不能为空", trigger: "blur" }],
     contactPerson: [{ required: true, message: "联系人不能为空", trigger: "blur" }],
     phoneNumber: [{ required: true, message: "联系电话不能为空", trigger: "blur" }],
     admin: [{ required: true, message: "管理员ID不能为空", trigger: "blur" }]
   }
 });
 
 const { queryParams, form, rules } = toRefs(data);
 
 /** 查询租户列表 */
 function getList() {
   loading.value = true;
   listTenant(queryParams.value).then(response => {
     tenantList.value = response.rows;
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
     tenantId: undefined,
     tenantName: undefined,
     contactPerson: undefined,
     phoneNumber: undefined,
     admin: undefined,
     remark: undefined
   };
   proxy.resetForm("tenantRef");
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
   ids.value = selection.map(item => item.tenantId);
   single.value = selection.length != 1;
   multiple.value = !selection.length;
 }
 
 /** 新增按钮操作 */
 function handleAdd() {
   reset();
   open.value = true;
   title.value = "添加租户";
 }
 
 /** 修改按钮操作 */
 function handleUpdate(row) {
   reset();
   const tenantId = row.tenantId || ids.value;
   getTenant(tenantId).then(response => {
     form.value = response.data;
     open.value = true;
     title.value = "修改租户";
   });
 }
 
 /** 提交按钮 */
 function submitForm() {
   proxy.$refs["tenantRef"].validate(valid => {
     if (valid) {
       if (form.value.tenantId != undefined) {
         updateTenant(form.value).then(response => {
           proxy.$modal.msgSuccess("修改成功");
           open.value = false;
           getList();
         });
       } else {
         addTenant(form.value).then(response => {
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
   const tenantIds = row.tenantId || ids.value;
   proxy.$modal.confirm('是否确认删除租户编号为"' + tenantIds + '"的数据项？').then(function() {
     return delTenant(tenantIds);
   }).then(() => {
     getList();
     proxy.$modal.msgSuccess("删除成功");
   }).catch(() => {});
 }
 
 /** 导出按钮操作 */
 function handleExport() {
   proxy.download("system/tenant/export", {
     ...queryParams.value
   }, `tenant_${new Date().getTime()}.xlsx`);
 }
 
 getList();
 </script>
 