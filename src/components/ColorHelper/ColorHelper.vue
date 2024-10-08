<script setup lang="ts">
import { ref } from 'vue';
import ImageBoard from './ImageBoard.vue';
import SettingsPopover from './SettingsPopover.vue';
import { ElNotification, type TabPaneName } from 'element-plus';
import { fileToDataURL } from './tools';
import AdbHelper from './AdbHelper.vue';

let tabIndex = 1
const editableTabsValue = ref('1')
const editableTabs = ref([
    // {
    //     title: 'Tab 1',
    //     name: '1',
    //     src: undefined,
    // }
]);
const handleTabsEdit = (targetName: TabPaneName | undefined, action: 'remove' | 'add') => {
    if (action === 'add') {
        const newTabName = `${++tabIndex}`
        editableTabs.value.push({
            title: 'New Tab',
            name: newTabName,
            src: undefined,
        });
        editableTabsValue.value = newTabName;
    } else if (action === 'remove') {
        const tabs = editableTabs.value
        let activeName = editableTabsValue.value
        if (activeName === targetName) {
            tabs.forEach((tab, index) => {
                if (tab.name === targetName) {
                    const nextTab = tabs[index + 1] || tabs[index - 1]
                    if (nextTab) {
                        activeName = nextTab.name
                    }
                }
            })
        }

        editableTabsValue.value = activeName
        editableTabs.value = tabs.filter((tab) => tab.name !== targetName)
    }
}

const handleFileChange = async (file: any) => {
    // TODO 暂时不允许重复加载图片，后续再考重复加载的图片更新命名后新增页签加载
    if (editableTabs.value.find(tab => tab.name === file.name)) {
        ElNotification({
            message: `已切换至${file.name}`,
            type: 'info',
        });
        editableTabsValue.value = file.name;
        return;
    }
    const newTabName = file.name;
    const dataUrl = await fileToDataURL(file.raw);
    // console.log(dataUrl)
    editableTabs.value.push({
        title: newTabName,
        name: newTabName,
        src: dataUrl
    })
    editableTabsValue.value = newTabName;
}

const screenCap = async (data: { fileName: string, dataUrl: string }) => {
    // console.log(dataUrl);
    editableTabs.value.push({
        title: data.fileName,
        name: data.fileName,
        src: data.dataUrl
    });
    editableTabsValue.value = data.fileName;
}

const settingsPopoverShown = ref(false);

</script>

<template>
    <div class="color-helper-main-toolbar">
        <div style="display: flex;">
            <el-upload multiple :on-change="handleFileChange" accept="image/*" :auto-upload="false" :show-file-list="false">
                <el-button type="primary">加载图片</el-button>
            </el-upload>
            <SettingsPopover :visible="settingsPopoverShown">
                <template #reference>
                    <el-button type="default" @click="settingsPopoverShown = true" style="margin-left: 10px">设置</el-button>
                </template>
                <template #footer>
                    <el-button @click="settingsPopoverShown = false" size="small" type="primary" style="margin-top: 10px;"
                        link>关闭</el-button>
                </template>
            </SettingsPopover>
        </div>
        <div style="display: flex;">
            <AdbHelper :on-screencap="screenCap">
            </AdbHelper>
        </div>
    </div>
    <div class="color-helper-main-container">
        <el-tabs v-model="editableTabsValue" type="border-card" editable @edit="handleTabsEdit" class="color-helper-tabs">
            <el-tab-pane v-for="item in editableTabs" :key="item.name" :label="item.title" :name="item.name">
                <ImageBoard :src="item.src"></ImageBoard>
            </el-tab-pane>
        </el-tabs>
    </div>
</template>
<style scoped>
.color-helper-main-toolbar {
    display: flex;
    height: 42px;
    padding-bottom: 10px;
    width: 100%;
    justify-content: space-between;
}

.color-helper-main-container {
    height: calc(100% - 42px);
}
</style>
<style>
/* 隐藏+按钮，tabs通过上传图片新增 */
/* 没有考虑好如何通过子组件加载的图片的图片的文件名反馈到tabs的标签名上 */
.color-helper-tabs .el-tabs__new-tab {
    display: none;
    margin-right: 10px;
}

.color-helper-tabs {
    height: 100%;
    overflow: auto;
}

.color-helper-tabs .el-tab-pane {
    height: 100%;
}
</style>
