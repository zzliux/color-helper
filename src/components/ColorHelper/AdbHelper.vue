<script setup lang="ts">
import { onMounted, ref } from 'vue';
import { adbHelper } from './tools';
import { onUnmounted } from 'vue';
import { ElNotification } from 'element-plus';
import { Plus, Loading, } from '@element-plus/icons-vue';
import { watch } from 'vue';

const deviceIdMode = ref<string | null>(null);
const deviceIdOptions = ref<{ value: string, disabled?: boolean, label: string, type: 'mumu' | 'adb', connected: boolean }[]>([]);
const deviceId = ref<string>(null);
const shown = ref<boolean>(false);
const loadingScreenCap = ref<boolean>(false);
const loadingDevices = ref<boolean>(false);
const loadingConnect = ref<boolean>(false);
const remoteDeviceId = ref<string>(null);
let callbackId: number = null;

const $props = defineProps({
    onScreencap: {
        type: Function,
        required: false,
    }
});

watch(deviceId, (newVal, oldVal) => {
    adbHelper.setCurrentDeviceId(newVal);
    // 如果没有连接，手动异步连接
    if (deviceIdOptions.value.find(item => item.value === newVal && !item.connected)) {
        connect(newVal);
    }
});

const screencap = async () => {
    if (!deviceId.value) {
        await refreshDevices(false);
        if (deviceIdOptions.value.length > 0) {
            deviceId.value = deviceIdOptions.value[0].value;
        }
    }
    if ($props.onScreencap) {
        loadingScreenCap.value = true;
        try {
            const dataUrl = await adbHelper.screencap(deviceId.value);
            $props.onScreencap(dataUrl);
        } catch (e) {
            console.error(e);
        }
        loadingScreenCap.value = false;
    }
}

const connect = async (deviceId?: string) => {
    if (!deviceId) {
        deviceId = remoteDeviceId.value;
    }
    loadingConnect.value = true;
    try {
        const msg = await adbHelper.connect(deviceId);
        ElNotification({
            message: msg,
            type: 'info'
        })
    } catch (e) {
        console.log(e);
    }
    loadingConnect.value = false;
}

const refreshDevices = async (visible: boolean) => {
    if (!visible) return;

    loadingDevices.value = true;
    try {
        const devices: any[] = (await adbHelper.devices(deviceIdMode.value)) || [];
        deviceIdOptions.value = devices.map(item => ({
            value: item.value,
            label: item.label || item.value,
            disabled: /offline$/i.test(item.label),
            type: item.type,
            connected: item.connected
        }));
        deviceId.value = deviceIdOptions.value.length > 0 ? deviceIdOptions.value[0].value : '';
    } catch (e) {
        console.log(e);
    }
    loadingDevices.value = false;
}
const localBridge = async () => {
    window.location.href = "colorhelperbridge://open?param=123";
}
onMounted(async () => {
    callbackId = adbHelper.setCallback(function () {
        shown.value = true;
        // refreshDevices();
    }, function () {
        shown.value = false;
    });
});

onUnmounted(() => {
    adbHelper.removeCallback(callbackId);
});

</script>

<template>
    <div v-if="shown">
        <el-form :inline="true">
            <el-form>
                <el-button @click="screencap" :disabled="loadingScreenCap || !deviceId" type="primary" style="margin-left: 20px;">
                    <el-icon v-if="loadingScreenCap" class="is-loading">
                        <Loading />
                    </el-icon>
                    <template v-if="!loadingScreenCap">截图</template>
                </el-button>
                <el-select v-model="deviceIdMode" placeholder="选择连接模式" style="width:130px; margin-left: 10px" @change="refreshDevices">
                    <el-option label="MuMu模拟器" value="mumu" />
                    <el-option label="ADB连接" value="adb" />
                </el-select>
                <el-select v-if="deviceIdMode" v-model="deviceId" :loading="loadingDevices" placeholder="选择设备"
                    style="width: 200px; margin-left: 10px; margin-right: 10px;" @visible-change="refreshDevices">
                    <el-option v-for="item in deviceIdOptions" :key="item.value" :label="item.label" :value="item.value"
                        :disabled="item.disabled" />
                </el-select>
                <el-popover v-if="deviceIdMode === 'adb'" placement="bottom" trigger="click" :width="300">
                    <template #reference>
                        <el-button>
                            <el-icon>
                                <Plus />
                            </el-icon>
                        </el-button>
                    </template>
                    <div>
                        <el-input v-model="remoteDeviceId" placeholder="IP[:PORT]"
                            style="width:200px; margin-right: 10px;" :readonly="loadingConnect">
                        </el-input>
                        <el-button @click="connect" type="primary" style="width: 60px">
                            <el-icon v-if="loadingConnect" class="is-loading">
                                <Loading />
                            </el-icon>
                            <template v-if="!loadingConnect">连接</template>
                        </el-button>
                    </div>
                </el-popover>
            </el-form>
        </el-form>
    </div>
    <div v-if="!shown">
        <el-button @click="localBridge" type="success" style="margin-left: 20px;">
            <el-icon v-if="loadingScreenCap" class="is-loading">
                <Loading />
            </el-icon>
            <template v-if="!loadingScreenCap">启用 ADB 连接（需本地桥应用）</template>
        </el-button>
    </div>
</template>
<style></style>