<template>
    <div class="app-container">
        <el-form :model="queryParams" ref="queryRef" v-show="showSearch" :inline="true">
            <el-form-item label="渠道名称" prop="channelName">
                <el-input
                    v-model="queryParams.searchValue"
                    placeholder="请输入渠道名称"
                    clearable
                    style="width: 240px"
                    @keyup.enter="handleQuery"
                />
            </el-form-item>
            <el-form-item label="状态" prop="status">
                <el-select
                    v-model="queryParams.status"
                    placeholder="渠道状态"
                    clearable
                    style="width: 240px"
                >
                    <el-option
                        v-for="dict in channel_status"
                        :key="dict.value"
                        :label="dict.label"
                        :value="dict.value"
                    />
                </el-select>
            </el-form-item>
            <el-form-item label="类型" prop="status">
                <el-select
                    v-model="queryParams.type"
                    placeholder="渠道类型"
                    clearable
                    style="width: 240px"
                >
                    <el-option
                        v-for="dict in channel_type"
                        :key="dict.value"
                        :label="dict.label"
                        :value="dict.value"
                    />
                </el-select>
            </el-form-item>
            <el-form-item label="创建时间" style="width: 308px">
                <el-date-picker
                    v-model="dateRange"
                    value-format="YYYY-MM-DD"
                    type="daterange"
                    range-separator="-"
                    start-placeholder="开始日期"
                    end-placeholder="结束日期"
                ></el-date-picker>
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
                    v-hasPermi="['system:role:add']"
                >新增
                </el-button>
            </el-col>
            <el-col :span="1.5">
                <el-button
                    type="success"
                    plain
                    icon="Edit"
                    :disabled="single"
                    @click="handleUpdate"
                    v-hasPermi="['system:role:edit']"
                >修改
                </el-button>
            </el-col>
            <el-col :span="1.5">
                <el-button
                    type="danger"
                    plain
                    icon="Delete"
                    :disabled="multiple"
                    @click="handleDelete"
                    v-hasPermi="['system:role:remove']"
                >删除
                </el-button>
            </el-col>
            <el-col :span="1.5">
                <el-button
                    type="warning"
                    plain
                    icon="Download"
                    @click="handleExport"
                    v-hasPermi="['system:role:export']"
                >导出
                </el-button>
            </el-col>
            <right-toolbar v-model:showSearch="showSearch" @queryTable="getList"></right-toolbar>
        </el-row>

        <!-- 表格数据 -->
        <el-table v-loading="loading" :data="activityList" @selection-change="handleSelectionChange">
            <el-table-column type="selection" width="55" align="center"/>
            <el-table-column label="活动Id" prop="activityId" width="120"/>
            <el-table-column label="活动名称" prop="name" width="120"/>
            <el-table-column label="状态" prop="status" :show-overflow-tooltip="true" width="150">
                <template #default="scope">
                    <dict-tag :options="activity_status" :value="scope.row.status" />
                </template>
            </el-table-column>
            <el-table-column label="类型" prop="type" :show-overflow-tooltip="true" width="150">
                <template #default="scope">
                    <dict-tag :options="channel_type" :value="scope.row.type" />
                </template>
            </el-table-column>
            <el-table-column label="活动简介" prop="info" :show-overflow-tooltip="true" width="150"/>
            <el-table-column label="开始时间" align="center" prop="beginTime">
                <template #default="scope">
                    <span>{{ parseTime(scope.row.beginTime) }}</span>
                </template>
            </el-table-column>
            <el-table-column label="结束时间" align="center" prop="endTime">
                <template #default="scope">
                    <span>{{ parseTime(scope.row.endTime) }}</span>
                </template>
            </el-table-column>
            <el-table-column label="操作" align="center" class-name="small-padding fixed-width">
                <template #default="scope">
                    <el-tooltip content="修改" placement="top" v-if="scope.row.activityId !== 0">
                        <el-button
                            type="text"
                            icon="Edit"
                            @click="handleUpdate(scope.row)"
                            v-hasPermi="['system:role:edit']"
                        ></el-button>
                    </el-tooltip>
                    <el-tooltip content="删除" placement="top" v-if="scope.row.activityId !== 0">
                        <el-button
                            type="text"
                            icon="Delete"
                            @click="handleDelete(scope.row)"
                            v-hasPermi="['system:role:remove']"
                        ></el-button>
                    </el-tooltip>
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

        <!-- 添加或修改活动对话框 -->
        <el-dialog :title="title" v-model="open" width="500px" append-to-body>
            <el-form ref="activityRef" :model="form" :rules="rules" label-width="100px">
                <el-form-item label="活动名称" >
                    <el-input v-model="form.name" placeholder="请输入活动名称"/>
                </el-form-item>
                <el-form-item label="活动类型">
                    <el-radio-group v-model="form.type">
                        <el-radio
                            v-for="dict in activity_type"
                            :key="dict.value"
                            :label="dict.value"
                        >{{ dict.label }}</el-radio>
                    </el-radio-group>
                </el-form-item>
                <el-form-item label="折扣券" >
                    <el-input-number v-model="form.discount" placeholder="折扣券"/>
                </el-form-item>
                <el-form-item label="活动简介">
                    <el-input v-model="form.info" type="textarea" placeholder="请输入活动简介"></el-input>
                </el-form-item>
            </el-form>
            <template #footer>
                <div class="dialog-footer">
                    <el-button type="primary" @click="submitForm">确 定</el-button>
                    <el-button @click="cancel">取 消</el-button>
                </div>
            </template>
        </el-dialog>

        <!-- 分配角色数据权限对话框 -->
        <el-dialog :title="title" v-model="openDataScope" width="500px" append-to-body>
            <el-form :model="form" label-width="80px">
                <el-form-item label="角色名称">
                    <el-input v-model="form.roleName" :disabled="true"/>
                </el-form-item>
                <el-form-item label="权限字符">
                    <el-input v-model="form.roleKey" :disabled="true"/>
                </el-form-item>
                <el-form-item label="权限范围">
                    <el-select v-model="form.dataScope" @change="dataScopeSelectChange">
                        <el-option
                            v-for="item in dataScopeOptions"
                            :key="item.value"
                            :label="item.label"
                            :value="item.value"
                        ></el-option>
                    </el-select>
                </el-form-item>
                <el-form-item label="数据权限" v-show="form.dataScope == 2">
                    <el-checkbox v-model="deptExpand" @change="handleCheckedTreeExpand($event, 'dept')">展开/折叠
                    </el-checkbox>
                    <el-checkbox v-model="deptNodeAll" @change="handleCheckedTreeNodeAll($event, 'dept')">全选/全不选
                    </el-checkbox>
                    <el-checkbox v-model="form.deptCheckStrictly" @change="handleCheckedTreeConnect($event, 'dept')">
                        父子联动
                    </el-checkbox>
                    <el-tree
                        class="tree-border"
                        :data="deptOptions"
                        show-checkbox
                        default-expand-all
                        ref="deptRef"
                        node-key="id"
                        :check-strictly="!form.deptCheckStrictly"
                        empty-text="加载中，请稍候"
                        :props="{ label: 'label', children: 'children' }"
                    ></el-tree>
                </el-form-item>
            </el-form>
            <template #footer>
                <div class="dialog-footer">
                    <el-button type="primary" @click="submitDataScope">确 定</el-button>
                    <el-button @click="cancelDataScope">取 消</el-button>
                </div>
            </template>
        </el-dialog>
    </div>
</template>


<script setup name="Channel">
import {addRole, changeRoleStatus, dataScope, delRole, getRole, listRole, updateRole} from "@/api/system/role";
import {roleMenuTreeselect, treeselect as menuTreeselect} from "@/api/system/menu";
import {treeselect as deptTreeselect, roleDeptTreeselect} from "@/api/system/dept";
import {listActivity,getActivity,addActivity,updateActivity,delActivity} from "@/api/tienchin/activity";

const router = useRouter();
const {proxy} = getCurrentInstance();
const {sys_normal_disable,activity_type} = proxy.useDict("sys_normal_disable","activity_type");

const activityList = ref([]);
const open = ref(false);
const loading = ref(true);
const showSearch = ref(true);
const ids = ref([]);
const single = ref(true);
const multiple = ref(true);
const total = ref(0);
const title = ref("");
const dateRange = ref([]);
const menuOptions = ref([]);
const menuExpand = ref(false);
const menuNodeAll = ref(false);
const deptExpand = ref(true);
const deptNodeAll = ref(false);
const deptOptions = ref([]);
const openDataScope = ref(false);
const menuRef = ref(null);
const deptRef = ref(null);

/** 数据范围选项*/
const dataScopeOptions = ref([
    {value: "1", label: "全部数据权限"},
    {value: "2", label: "自定数据权限"},
    {value: "3", label: "本部门数据权限"},
    {value: "4", label: "本部门及以下数据权限"},
    {value: "5", label: "仅本人数据权限"}
]);

const data = reactive({
    form: {},
    queryParams: {
        pageNum: 1,
        pageSize: 10,
        searchValue: undefined,
        roleKey: undefined,
        type: undefined,
        status: undefined
    },
    rules: {
        name: [{required: true, message: "活动名称不能为空", trigger: "blur"}],
        type: [{required: true, message: "活动类型不能为空", trigger: "blur"}]
    },
});

const {queryParams, form, rules} = toRefs(data);

/** 查询角色列表 */
function getList() {
    loading.value = true;
    listActivity(proxy.addDateRange(queryParams.value, dateRange.value)).then(response => {
        activityList.value = response.rows;
        total.value = response.total;
        loading.value = false;
    });
}

/** 搜索按钮操作 */
function handleQuery() {
    queryParams.value.pageNum = 1;
    getList();
}

/** 重置按钮操作 */
function resetQuery() {
    dateRange.value = [];
    proxy.resetForm("queryRef");
    handleQuery();
}

/** 删除按钮操作 */
function handleDelete(row) {
    const activityIds = row.activityId || ids.value;
    proxy.$modal.confirm('是否确认删除活动编号为"' + activityIds + '"的数据项?').then(function () {
        return delActivity(activityIds);
    }).then(() => {
        getList();
        proxy.$modal.msgSuccess("删除成功");
    }).catch(() => {
    });
}

/** 导出按钮操作 */
function handleExport() {
    proxy.download("system/role/export", {
        ...queryParams.value,
    }, `role_${new Date().getTime()}.xlsx`);
}

/** 多选框选中数据 */
function handleSelectionChange(selection) {
    ids.value = selection.map(item => item.activityId);
    single.value = selection.length != 1;
    multiple.value = !selection.length;
}

/** 角色状态修改 */
function handleStatusChange(row) {
    let text = row.status === "0" ? "启用" : "停用";
    proxy.$modal.confirm('确认要"' + text + '""' + row.roleName + '"角色吗?').then(function () {
        return changeRoleStatus(row.roleId, row.status);
    }).then(() => {
        proxy.$modal.msgSuccess(text + "成功");
    }).catch(function () {
        row.status = row.status === "0" ? "1" : "0";
    });
}

/** 更多操作 */
function handleCommand(command, row) {
    switch (command) {
        case "handleDataScope":
            handleDataScope(row);
            break;
        case "handleAuthUser":
            handleAuthUser(row);
            break;
        default:
            break;
    }
}

/** 分配用户 */
function handleAuthUser(row) {
    router.push("/system/role-auth/user/" + row.roleId);
}

/** 查询菜单树结构 */
function getMenuTreeselect() {
    menuTreeselect().then(response => {
        menuOptions.value = response.data;
    });
}

/** 所有部门节点数据 */
function getDeptAllCheckedKeys() {
    // 目前被选中的部门节点
    let checkedKeys = deptRef.value.getCheckedKeys();
    // 半选中的部门节点
    let halfCheckedKeys = deptRef.value.getHalfCheckedKeys();
    checkedKeys.unshift.apply(checkedKeys, halfCheckedKeys);
    return checkedKeys;
}

/** 重置新增的表单以及其他数据  */
function reset() {
    form.value = {
        activityId:undefined,
        name: undefined,
        type: undefined,
        status: undefined,
        info: undefined,
        discount: undefined,
        voucher: undefined,
        beginTime: undefined,
        endTime: undefined
    };
    // 校验
    proxy.resetForm("activityRef");
}

/** 添加角色 */
function handleAdd() {
    reset();
    getMenuTreeselect();
    open.value = true;
    title.value = "添加角色";
}

/** 修改角色 */
function handleUpdate(row) {
    reset();
    const activityId = row.activityId || ids.value;
    getActivity(activityId).then(response => {
        form.value = response.data;
        // form.value.roleSort = Number(form.value.roleSort);
        open.value = true;
        // nextTick(() => {
        //     roleMenu.then((res) => {
        //         let checkedKeys = res.checkedKeys;
        //         checkedKeys.forEach((v) => {
        //             nextTick(() => {
        //                 menuRef.value.setChecked(v, true, false);
        //             });
        //         });
        //     });
        // });
        title.value = "修改活动";
    });
}

/** 根据角色ID查询菜单树结构 */
function getRoleMenuTreeselect(roleId) {
    return roleMenuTreeselect(roleId).then(response => {
        menuOptions.value = response.menus;
        return response;
    });
}

/** 根据角色ID查询部门树结构 */
function getRoleDeptTreeselect(roleId) {
    return roleDeptTreeselect(roleId).then(response => {
        deptOptions.value = response.depts;
        return response;
    });
}

/** 树权限（展开/折叠）*/
function handleCheckedTreeExpand(value, type) {
    if (type == "menu") {
        let treeList = menuOptions.value;
        for (let i = 0; i < treeList.length; i++) {
            menuRef.value.store.nodesMap[treeList[i].id].expanded = value;
        }
    } else if (type == "dept") {
        let treeList = deptOptions.value;
        for (let i = 0; i < treeList.length; i++) {
            deptRef.value.store.nodesMap[treeList[i].id].expanded = value;
        }
    }
}

/** 树权限（全选/全不选） */
function handleCheckedTreeNodeAll(value, type) {
    if (type == "menu") {
        menuRef.value.setCheckedNodes(value ? menuOptions.value : []);
    } else if (type == "dept") {
        deptRef.value.setCheckedNodes(value ? deptOptions.value : []);
    }
}

/** 树权限（父子联动） */
// function handleCheckedTreeConnect(value, type) {
//     if (type == "menu") {
//         form.value.menuCheckStrictly = value ? true : false;
//     } else if (type == "dept") {
//         form.value.deptCheckStrictly = value ? true : false;
//     }
// }
/** 所有菜单节点数据 */
// function getMenuAllCheckedKeys() {
//     // 目前被选中的菜单节点
//     let checkedKeys = menuRef.value.getCheckedKeys();
//     // 半选中的菜单节点
//     let halfCheckedKeys = menuRef.value.getHalfCheckedKeys();
//     checkedKeys.unshift.apply(checkedKeys, halfCheckedKeys);
//     return checkedKeys;
// }
/** 提交按钮 */
function submitForm() {
    proxy.$refs["activityRef"].validate(valid => {
        if (valid) {
            if (form.value.activityId != undefined) {
                updateActivity(form.value).then(response => {
                    proxy.$modal.msgSuccess("修改成功");
                    open.value = false;
                    getList();
                });
            } else {
                // form.value.menuIds = getMenuAllCheckedKeys();
                addActivity(form.value).then(response => {
                    proxy.$modal.msgSuccess("新增成功");
                    open.value = false;
                    getList();
                });
            }
        }
    });
}

/** 取消按钮 */
function cancel() {
    open.value = false;
    reset();
}

/** 选择角色权限范围触发 */
function dataScopeSelectChange(value) {
    if (value !== "2") {
        deptRef.value.setCheckedKeys([]);
    }
}

/** 分配数据权限操作 */
function handleDataScope(row) {
    reset();
    const roleDeptTreeselect = getRoleDeptTreeselect(row.roleId);
    getRole(row.roleId).then(response => {
        form.value = response.data;
        openDataScope.value = true;
        nextTick(() => {
            roleDeptTreeselect.then(res => {
                nextTick(() => {
                    if (deptRef.value) {
                        deptRef.value.setCheckedKeys(res.checkedKeys);
                    }
                });
            });
        });
        title.value = "分配数据权限";
    });
}

/** 提交按钮（数据权限） */
function submitDataScope() {
    if (form.value.roleId != undefined) {
        form.value.deptIds = getDeptAllCheckedKeys();
        dataScope(form.value).then(response => {
            proxy.$modal.msgSuccess("修改成功");
            openDataScope.value = false;
            getList();
        });
    }
}

/** 取消按钮（数据权限）*/
function cancelDataScope() {
    openDataScope.value = false;
    reset();
}

getList();
</script>

<style scoped>

</style>