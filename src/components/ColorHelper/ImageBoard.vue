<script setup lang="ts">
import ColoredCheckbox from './ColoredCheckbox.vue';
import Point from './scriptlib/Point';
import { computed, onMounted, onUnmounted, ref, watch, nextTick } from 'vue';
import { colorSimilarity, pixOfImageDataArray, pixOfImageDataString, parseToImageData, throttle, isImageLight, type RegionRowData, type PositionRowData, adbHelper, fileToDataURL, type ScreenCapResult, asColorArray } from './tools';
import { executors as innerExecutors, type IExecutor } from './executor/Executor';
import { ElNotification, type UploadFile, type UploadFiles } from 'element-plus';
import emitter from './eventBus';
import GraphicHelper from './scriptlib/GraphicHelper';
import { InfoFilled, Loading, UploadFilled } from '@element-plus/icons-vue';
import 'element-plus/dist/index.css'
import 'element-plus/theme-chalk/dark/css-vars.css'
import { shortcutManager } from './shortcutManager';

const imageCanvasRef = ref<HTMLCanvasElement>();
const maskCanvasRef = ref<HTMLCanvasElement>();
const magnifierCanvasRef = ref<HTMLCanvasElement>();
const magnifierRef = ref<HTMLDivElement>();
const positionX = ref(-1);
const positionY = ref(-1);
const positionColor = ref('000000');
const positionData = ref<PositionRowData[]>([]);
const executors = computed(() => {
    return Array.from(innerExecutors);
});
const importPositionDataText = ref('');
const importPositionDataPopVisible = ref(false);
const exportPositionDataText = ref('');
const exportPositionDataPopVisible = ref(false);
const positionRegionTabModel = ref('色组');
const regionData = ref<RegionRowData[]>([]);
const positionUnreadNum = ref<number>(0);
const regionUnreadNum = ref<number>(0);
watch(positionRegionTabModel, (newVal, oldVal) => {
    if (newVal === '区域') {
        regionUnreadNum.value = 0;
    }
    else if (newVal === '色组') {
        positionUnreadNum.value = 0;
    }
});



let img: HTMLImageElement;
let imgLoaded = ref(false);

const magnifierShown = ref(false);

let imageCtx: CanvasRenderingContext2D;
let maskCtx: CanvasRenderingContext2D;
let magnifierCtx: CanvasRenderingContext2D;
let scale: number;
// 放大镜显示宽度(px)
let imgClipWidth = 15;

/**
 * 将显示坐标转换为原始图片坐标
 * @param displayX 显示坐标X
 * @param displayY 显示坐标Y
 * @returns 原始图片坐标
 */
const displayToImageCoord = (displayX: number, displayY: number): [number, number] => {
    if (!imgLoaded.value) return [displayX, displayY];
    const scaleFactor = zoomRatio.value / 100;
    return [
        Math.round(displayX / scaleFactor),
        Math.round(displayY / scaleFactor)
    ];
};

/**
 * 将原始图片坐标转换为显示坐标
 * @param imageX 原始图片坐标X
 * @param imageY 原始图片坐标Y
 * @returns 显示坐标
 */
const imageToDisplayCoord = (imageX: number, imageY: number): [number, number] => {
    if (!imgLoaded.value) return [imageX, imageY];
    const scaleFactor = zoomRatio.value / 100;
    return [
        Math.round(imageX * scaleFactor),
        Math.round(imageY * scaleFactor)
    ];
};
let magnifierScale = 15; // 放大倍数


const $props = defineProps({
    src: String,
    actived: {
        type: Boolean,
        default: false
    }
});



/**
 * 监听props.src变化，变化后读取图片
 */
watch(() => $props.src, (newVal, oldVal) => {
    loadImage(newVal);
});

/**
 * 监听actived属性变化，更新快捷键状态
 */
watch(() => $props.actived, (newVal, oldVal) => {
    if (newVal) {
        // 激活时注册快捷键
        registerShortcuts();
    } else {
        // 非激活时注销快捷键
        unregisterShortcuts();
    }
});

const resize = throttle(() => {
    // let w = canvasDOM.value.parentElement.offsetWidth;
    // let h = Math.floor(w * 9 / 16);
    // if (img) {
    //     h = Math.floor(w * img.height / img.width);
    // }
    // canvasDOM.value.style.width = w + 'px';
    // canvasDOM.value.style.height = h + 'px';
}, 200);


let currentExecutor: IExecutor = null;
let executorMode: string = null;
let currentSimilarity: number = null;
let zoom: number = null;
const loadThisSettings = (option?: any) => {
    // TODO 加载本地配置，本地无配置的从默认配置中加载
    // 先做成从默认配置中加载
    // TODO 从option中加载更新的配置

    // 执行器
    const executorName = localStorage.getItem('ColorHelper.Settings.default.executor');
    currentExecutor = executors.value.find(executor => executor.name === executorName);

    // 执行器模式
    executorMode = localStorage.getItem('ColorHelper.Settings.default.executorMode');

    // 相似度
    currentSimilarity = parseInt(localStorage.getItem('ColorHelper.Settings.default.similarity'));

    // 默认缩放
    zoom = parseInt(localStorage.getItem('ColorHelper.Settings.default.zoom'));
    console.log('ImageBoard完成配置更新');
}


let mouseStatus = 'NONE'; // DOWN/NONE
const mouseDownPosition: Point = new Point(-1, -1);
const regionDownPosition: Point = new Point(-1, -1);
let mouseMoved = false;
let centerPoint: Point = new Point(-1, -1);

/**
 * 鼠标综合行为（色组）：
 * 1. 放大镜不显示时点击，设置准心位置为鼠标位置并显示放大镜；
 * 2. 显示放大镜时双击，设置准心位置为鼠标位置；
 * 3. 跟随模式时，且焦点在图片上，准心位置跟随鼠标相对移动；
 * 4. 滑动模式时，点击鼠标拖动每2个像素，准心位置相对移动1个像素
 * 5. 滑动模式时，双击鼠标则设置准心位置为鼠标位置；
 * 6. 鼠标在色组表格上悬停，反向在图片上更新准心与放大镜的位置；
 * 7. 鼠标点击选点（同空格逻辑）
 * 
 * 鼠标综合行为（区域）：
 * 1. 点击并拖动鼠标可选择区域，松开时，将区域信息添加到列表中；
 * 2. 鼠标在区域列表悬停时，反向在图片上高亮区域位置；
 * 
 * 键盘综合行为：
 * 1. 焦点在图片上时，点击键盘的A、S、D、空格键时会将准心位置的坐标、颜色、锚点信息添加到列表中；
 * 2. 键盘左、右、上、下键，移动准心位置；
 */
const maskMouseDownEvent = (e: MouseEvent) => {
    if (!imgLoaded.value) return;
    const [imageX, imageY] = displayToImageCoord(e.offsetX, e.offsetY);
    if (document.activeElement === maskCanvasRef.value) {
        regionDownPosition.set(centerPoint.x, centerPoint.y);
        mouseStatus = 'DOWN';
    } else {
        regionDownPosition.set(imageX, imageY);
    }
    maskCanvasRef.value.focus();
    mouseMoved = false;
    mouseDownPosition.set(e.offsetX, e.offsetY);
    if (!magnifierShown.value) {
        centerPoint.set(imageX, imageY, [0, 0, img.width - 1, img.height - 1]);
        drawMask();
        magnifierRefresh();
    } else if (centerPoint.x !== -1) {
        magnifierRefresh();
    }
}


let currentRegion: RegionRowData = null;
const maskMouseMoveEvent = (e: MouseEvent) => {
    if (!focused) return;
    requestAnimationFrame(() => {
        let moveScale = 1;
        const [imageX, imageY] = displayToImageCoord(e.offsetX, e.offsetY);
        centerPoint.set(imageX, imageY);
        drawMask();
        magnifierRefresh();
        mouseMoved = true;
        if (mouseStatus === 'DOWN') {
            if (!currentRegion) {
                currentRegion = {
                    checked: true,
                    anchor: img.width > img.height ? 'C' : 'M',
                    x0: regionDownPosition.x,
                    y0: regionDownPosition.y,
                    x1: imageX,
                    y1: imageY
                }
            } else {
                currentRegion.x1 = imageX;
                currentRegion.y1 = imageY;
            }
        }
        mouseDownPosition.set(e.offsetX, e.offsetY);
    })
};


const maskMouseUpEvent = (e: MouseEvent) => {
    if (!imgLoaded.value) return;

    if (!mouseMoved && mouseStatus === 'DOWN') {
        addCenterToPostionData('Space');
    } else if (Math.abs(currentRegion.x1 - currentRegion.x0) <= 3 && Math.abs(currentRegion.y1 - currentRegion.y0) <= 3) {
        addCenterToPostionData('Space');
        currentRegion = null;
    } else if (mouseMoved) {
        addCurrentRegionToPositionData(true);
        magnifierRefresh();
        drawMask();
    }
    mouseStatus = 'NONE';
}

const maskMouseLeaveEvent = (e: MouseEvent) => {
    mouseStatus = 'NONE';
}


let focused = false;
const maskKeyEvent = (e: KeyboardEvent) => {
    if (!imgLoaded.value) return;
    if (!focused) return;
    if (e.code === 'ArrowLeft') {
        centerPoint.offset(-1, 0);
        drawMask();
        magnifierRefresh();
    } else if (e.code === 'ArrowRight') {
        centerPoint.offset(1, 0);
        drawMask();
        magnifierRefresh();
    } else if (e.code === 'ArrowUp') {
        centerPoint.offset(0, -1);
        drawMask();
        magnifierRefresh();
    } else if (e.code === 'ArrowDown') {
        centerPoint.offset(0, 1);
        drawMask();
        magnifierRefresh();
    } else if (e.code === 'Space' ||
        e.code === 'KeyA' || e.code === 'KeyS' || e.code === 'KeyD' ||
        e.code === 'KeyW' || e.code === 'KeyX'
    ) {
        mouseMoved = false;
        addCenterToPostionData(e.code);
    }
}

const maskFocusEvent = () => {
    if (!imgLoaded.value) return;
    focused = true;

    magnifierRefresh();
}

const maskBlurEvent = () => {
    magnifierShown.value = false;
    focused = false;
}

const addCenterToPostionData = (code: string) => {
    let anchor: 'N' | 'L' | 'C' | 'R' | 'T' | 'M' | 'B' = 'N';
    if (code === 'KeyA') anchor = 'L';
    else if (code === 'KeyS') {
        if (img.width > img.height) {
            anchor = 'C';
        } else {
            anchor = 'M';
        }
    } else if (code === 'KeyD') anchor = 'R';
    else if (code === 'KeyW') anchor = 'T';
    else if (code === 'KeyX') anchor = 'B';
    else if (code === 'Space') {
        //  按1/4 1/2 1/4比例来，分别为LCR
        const anchorRatios = [.25, .5, .25].map(ratio => ratio * img.width);
        if (img.width > img.height) {
            if (centerPoint.x <= anchorRatios[0]) anchor = 'L';
            else if (centerPoint.x >= anchorRatios[1]) anchor = 'R';
            else anchor = 'C'; // 最后一个不用判断，取剩余空间
        } else {
            if (centerPoint.y <= anchorRatios[0]) anchor = 'T';
            else if (centerPoint.y >= anchorRatios[1]) anchor = 'B';
            else anchor = 'M'; // 最后一个不用判断，取剩余空间
        }
    }
    let data: PositionRowData;
    if (!mouseMoved) {
        data = {
            checked: true,
            anchor: anchor, // C-Center/L-Left/R-Right/N-NONE
            coordinate: `${centerPoint.x},${centerPoint.y}`,
            color: `0x${pixOfImageDataString(imageCtx.getImageData(centerPoint.x, centerPoint.y, 1, 1), 0, 0)}`,
            similarity: 100,
        };
    }
    else {
        data = {
            checked: true,
            anchor: anchor, // C-Center/L-Left/R-Right/N-NONE
            coordinate: `${currentRegion.x0},${currentRegion.y0}`,
            color: `0x${pixOfImageDataString(imageCtx.getImageData(currentRegion.x0, currentRegion.y0, 1, 1), 0, 0)}`,
            similarity: 100,
        };
        mouseMoved = false;
    }
    let flag = true;
    for (let i = 0; i < positionData.value.length; i++) {
        if (positionData.value[i].coordinate === data.coordinate) {
            positionData.value[i] = data;
            flag = false;
            break;
        }
    }
    if (flag) {
        positionData.value.push(data);
        if (positionRegionTabModel.value !== '色组') {
            positionUnreadNum.value++;
        }
    }
}

const addCurrentRegionToPositionData = (flag: boolean): void => {
    if (!currentRegion) return;
    regionData.value.push({
        ...currentRegion, ...{
            x0: Math.min(currentRegion.x0, currentRegion.x1),
            y0: Math.min(currentRegion.y0, currentRegion.y1),
            x1: Math.max(currentRegion.x0, currentRegion.x1),
            y1: Math.max(currentRegion.y0, currentRegion.y1)
        }
    });
    if (positionRegionTabModel.value !== '区域') {
        regionUnreadNum.value++;
    }
    if (flag) {
        currentRegion = null;
    }
}


const positionDataRowContextMenuEvent = (row: any, column: any, e: Event) => {
    e.preventDefault();
    console.log(row, column, e);
}

const clearBtnClickEvent = () => {
    if (positionRegionTabModel.value === '色组') {
        positionData.value = [];
    } else if (positionRegionTabModel.value === '区域') {
        regionData.value = [];
        drawMask();
    }
}

const exportPositionDataEvent = () => {
    try {
        importPositionDataPopVisible.value = false;
        const imageData = imageCtx.getImageData(0, 0, imageCanvasRef.value.width, imageCanvasRef.value.height);
        exportPositionDataText.value = currentExecutor.format(positionData.value, imageData, {
            executorName: currentExecutor.name,
            executorMode,
            regions: regionData.value,
            similarity: currentSimilarity
        });
        exportPositionDataText.value = exportPositionDataText.value.trimEnd() + ',';
        exportPositionDataPopVisible.value = true;
    } catch (e: any) {
        console.error(e);
        ElNotification({
            message: `导出失败：${e.message}。`,
            type: 'error',
        });
    }
}

const importPositionDataEvent = () => {
    if (importPositionDataText.value.trim() === '') {
        // 尝试从剪贴板获取数据
        navigator.clipboard.readText().then(text => {
            importPositionDataText.value = text;
            importPositionDataEvent();
        }).catch(err => {
            console.error('无法读取剪贴板数据：', err);
            ElNotification({
                message: '导入失败：无法读取剪贴板数据。',
                type: 'error',
            });
        });
        return;
    }
    try {
        const imageData = imageCtx.getImageData(0, 0, imageCanvasRef.value.width, imageCanvasRef.value.height);
        const result = currentExecutor.parse(importPositionDataText.value, imageData, {
            executorName: currentExecutor.name,
            executorMode,
            regions: regionData.value,
            similarity: currentSimilarity
        });
        if (result.positionData) {
            positionData.value = result.positionData;
            if (positionRegionTabModel.value !== '色组') {
                positionUnreadNum.value = result.positionData.length;
            }
        }
        if (result.regionData) {
            regionData.value = result.regionData;
            if (positionRegionTabModel.value !== '区域') {
                regionUnreadNum.value = result.regionData.length;
            }
        }
        drawMask();
        importPositionDataText.value = '';
        importPositionDataPopVisible.value = false;
        ElNotification({
            message: '导入成功',
            type: 'success',
        });
    } catch (e) {
        console.error(e);
        ElNotification({
            message: '导入失败：无法解析。',
            type: 'error',
        });
    }
};
const closeImportPositionData = () => {
    importPositionDataPopVisible.value = false
    importPositionDataText.value = ''
}
const positionDataDeleteRow = (scope: any, e: PointerEvent) => {
    positionData.value.splice(scope.$index, 1);
}

const renewColor = (scope: any, e: PointerEvent) => {
    const [x, y] = scope.row.coordinate.split(',').map(Number);
    positionData.value[scope.$index].color = `0x${pixOfImageDataString(imageCtx.getImageData(x, y, 1, 1), 0, 0)}`;
    positionData.value[scope.$index].similarity = 100;
}

const regionDataDeleteRow = (scope: any, e: PointerEvent) => {
    regionData.value.splice(scope.$index, 1);
    drawMask();
}

const testBtnClickEvent = () => {
    const result = currentExecutor.execute(positionData.value, imageCtx.getImageData(0, 0, imageCanvasRef.value.width, imageCanvasRef.value.height), {
        executorName: currentExecutor.name,
        executorMode,
        regions: regionData.value,
        similarity: currentSimilarity
    });
    if (result.success) {
        ElNotification({
            message: result.message,
            type: 'success',
        });
    } else {
        // TODO 返回的数据中有data，则根据data在mask上标注
        ElNotification({
            message: result.message,
            type: 'error',
        });
    }
}

const drawMask = () => {
    // 判断准心是否在视图区域内，如果不在视图区域内，则滚动到试图区域内
    const maskRect = maskCanvasRef.value.getBoundingClientRect();
    const containerRect = maskCanvasRef.value.parentElement.getBoundingClientRect();
    let flag = false;
    const scrollTarget = [
        maskCanvasRef.value.parentElement.scrollLeft,
        maskCanvasRef.value.parentElement.scrollTop
    ];

    // 将原始图片坐标转换为显示坐标
    const [displayX, displayY] = imageToDisplayCoord(centerPoint.x, centerPoint.y);

    // 超左边界
    if (displayX + maskRect.left < containerRect.left + 30) {
        flag = true;
        scrollTarget[0] = Math.max(maskCanvasRef.value.parentElement.scrollLeft - ((containerRect.left + 30) - (displayX + maskRect.left)), 0);
    }
    // 超右边界
    if (displayX + maskRect.left > containerRect.right - 30) {
        flag = true;
        scrollTarget[0] = maskCanvasRef.value.parentElement.scrollLeft + ((displayX + maskRect.left) - (containerRect.right - 30));
    }
    // 超上边界
    if (displayY + maskRect.top < containerRect.top + 30) {
        flag = true;
        scrollTarget[1] = Math.max(maskCanvasRef.value.parentElement.scrollTop - ((containerRect.top + 30) - (displayY + maskRect.top)), 0);
    }
    // 超下边界
    if (displayY + maskRect.top > containerRect.bottom - 30) {
        flag = true;
        scrollTarget[1] = maskCanvasRef.value.parentElement.scrollTop + ((displayY + maskRect.top) - (containerRect.bottom - 30));
    }

    if (flag) {
        maskCanvasRef.value.parentElement.scrollTo(scrollTarget[0], scrollTarget[1]);
    }

    maskCtx.clearRect(0, 0, maskCanvasRef.value.width, maskCanvasRef.value.height);
    drawCrossHair();
    drawRegions();
}

/**
 * 画准心时同步获取放大镜的位图数据，并使用放大镜位图数据判断是用亮色准心还是暗色准心
 */
const drawCrossHair = () => {
    // 计算放需要放大图片的显示位置
    let mLeft = centerPoint.x - (imgClipWidth - 1) / 2;
    let mTop = centerPoint.y - (imgClipWidth - 1) / 2;

    magBmpData = imageCtx.getImageData(mLeft, mTop, imgClipWidth, imgClipWidth);
    if (isImageLight(magBmpData)) {
        maskCtx.fillStyle = '#00008B';
    } else {
        maskCtx.fillStyle = '#FFFF00';
    }

    // 如果没有画过准心，在该位置画一个准心
    const regions: number[][] = [ // 相对中心位置
        [-1, -8, 2, 6], // 上 x, y, w, h
        [-1, 2, 2, 6], // 下
        [-8, -1, 6, 2], // 左
        [2, -1, 6, 2], // 右
    ];
    for (let region of regions) {
        maskCtx.fillRect(centerPoint.x + region[0], centerPoint.y + region[1], region[2], region[3]);
    }
}

const drawRegions = () => {
    // 画列表
    for (let i = 0; i < regionData.value.length; i++) {
        if (regionData.value[i].checked) {
            drawRegionInner(regionData.value[i], '#FFFFFF');
        }
    }

    // 画当前
    if (currentRegion) {
        drawRegionInner(currentRegion, '#FFFF00');
    }

    function drawRegionInner(region: RegionRowData, color: string): void {
        maskCtx.lineWidth = 2;
        maskCtx.strokeStyle = color;
        maskCtx.beginPath();
        maskCtx.setLineDash([10, 10]);
        maskCtx.moveTo(region.x0, region.y0);
        maskCtx.lineTo(region.x1, region.y0);
        maskCtx.lineTo(region.x1, region.y1);
        maskCtx.lineTo(region.x0, region.y1);
        maskCtx.lineTo(region.x0, region.y0);
        maskCtx.stroke();
    };
}

let magBmpData: ImageData = null;

const magnifierRefresh = () => {
    if (!imgLoaded.value) return;
    if (!magBmpData) return;

    setTimeout(() => {
        const offsetpx = 10;
        // const { pageX, pageY } = e;
        const canvasRect = imageCanvasRef.value.getBoundingClientRect();
        // 将原始图片坐标转换为显示坐标
        const [displayX, displayY] = imageToDisplayCoord(centerPoint.x, centerPoint.y);
        const pageX = displayX + canvasRect.left;
        const pageY = displayY + canvasRect.top;
        let { width: clientWidth, height: clientHeight } = magnifierCanvasRef.value;
        clientHeight += magnifierCanvasRef.value.nextElementSibling.clientHeight || 25;
        // 计算放大镜的位置
        let offsetX = Math.max(pageX + offsetpx, offsetpx);
        let offsetY = Math.max(pageY + offsetpx, offsetpx);
        if (offsetX + clientWidth + offsetpx > window.innerWidth) {
            offsetX = Math.min(pageX - clientWidth - offsetpx, window.innerWidth - clientWidth - offsetpx);
        }
        if (offsetY + clientHeight + offsetpx > window.innerHeight) {
            offsetY = Math.min(pageY - clientHeight - offsetpx, window.innerHeight - clientHeight - offsetpx);
        }

        setTimeout(() => {
            magnifierRef.value.style.left = offsetX + 'px';
            magnifierRef.value.style.top = offsetY + 'px';
        }, 0)


        magnifierCtx.clearRect(0, 0, magnifierCanvasRef.value.width, magnifierCanvasRef.value.height);
        // 画放大的图片，为避免模糊需像素放大画正方形
        for (let y = 0; y < imgClipWidth; y++) {
            for (let x = 0; x < imgClipWidth; x++) {
                const color = pixOfImageDataString(magBmpData, x, y, true);
                magnifierCtx.fillStyle = `#${color}`;
                magnifierCtx.fillRect(x * magnifierScale, y * magnifierScale, magnifierScale, magnifierScale);
            }
        }

        // 给中心区域画框，判断颜色是否和周围颜色相近画红、橙、绿的框
        const magCenterX = (imgClipWidth - 1) / 2;
        const magCenterY = (imgClipWidth - 1) / 2;
        const magCenterColor = pixOfImageDataArray(magBmpData, magCenterX, magCenterY, true);
        const colorSimilarMap = [
            [2, 2, 2, 2, 2],
            [2, 1, 1, 1, 2],
            [2, 1, 0, 1, 2],
            [2, 1, 1, 1, 2],
            [2, 2, 2, 2, 2],
        ];
        let isGreen = true, isOrange = true, isRed = true;
        if (magCenterColor[3] < 255) { // 如果有透明的话就直接给红色
            isGreen = false;
            isOrange = false;
        } else {
            for (let i = -2; i <= 2; i++) {
                for (let j = -2; j <= 2; j++) {
                    const curColor = pixOfImageDataArray(magBmpData, magCenterX + j, magCenterY + i, true);
                    const similar = colorSimilarity(magCenterColor, curColor);
                    // 附近有透明色或，或附近有不相似的颜色，不能给绿色或橙色
                    if (colorSimilarMap[i + 2][j + 2] === 2 && (similar < .95 || curColor[3] < 255)) {
                        isGreen = false;
                    } else if (colorSimilarMap[i + 2][j + 2] === 1 && (similar < .95 || curColor[3] < 255)) {
                        isGreen = false;
                        isOrange = false;
                    }
                }
            }
        }
        if (isGreen) magnifierCtx.strokeStyle = '#00CC00';
        else if (isOrange) magnifierCtx.strokeStyle = '#FF9900';
        else magnifierCtx.strokeStyle = '#CC0000';
        magnifierCtx.strokeRect(magCenterX * magnifierScale, magCenterY * magnifierScale, magnifierScale, magnifierScale);

        // 更新界面
        positionX.value = centerPoint.x;
        positionY.value = centerPoint.y;
        positionColor.value = pixOfImageDataString(magBmpData, magCenterX, magCenterY);

        // 没有显示的话再显示一下
        if (!magnifierShown.value) magnifierShown.value = true;
    }, 0);
}

let centerPointTemp: Point = null;
let magnifierShownTemp: boolean;
const positionTableCellMouseEnterEvent = (row: PositionRowData, column: any, cell: HTMLTableCellElement, event: Event) => {
    if (!centerPointTemp) {
        centerPointTemp = new Point(centerPoint.x, centerPoint.y, centerPoint.region);
    }
    if (typeof magnifierShownTemp === 'undefined') {
        magnifierShownTemp = magnifierShown.value;
    }
    const [x, y] = row.coordinate.split(',').map(Number);
    centerPoint.set(x, y);
    drawMask();
    magnifierRefresh();
};

const positionTableCellMouseLeaveEvent = (row: any, column: any, cell: HTMLTableCellElement, event: Event) => {
    centerPoint = centerPointTemp;
    centerPointTemp = null;
    drawMask();
    if (magnifierShownTemp) {
        magnifierRefresh();
    }
    magnifierShown.value = magnifierShownTemp;
    magnifierShownTemp = undefined;
};

const regionTableCellMouseEnterEvent = (row: RegionRowData, column: any, cell: HTMLTableCellElement, event: Event) => {
    currentRegion = row;
    drawMask();
};

const regionTableCellMouseLeaveEvent = (row: any, column: any, cell: HTMLTableCellElement, event: Event) => {
    currentRegion = null;
    drawMask();
};

let superpositionImageStack = ref<ImageData[]>([]);
const superpositionImageStackMaxSize: number = 10; // 记录上限，以免无限叠加导致内存爆掉
let superpositionImageStackCurrentIndex = ref<number>(0);

const superpositionFileChange = async (file: UploadFile, files: UploadFiles) => {
    const t1 = Date.now();
    console.log(`${file.name}: 开始叠加`);
    // const dataUrl = await fileToDataURL(file.raw);a
    const imageData = await parseToImageData(file.raw);
    console.log(`${file.name}: 加载耗时: ${Date.now() - t1}ms`);
    superpositionImageData(file.name, imageData);
}

const superpositionImageData = (fileName: string, imageData: ImageData) => {
    const currentImageData = imageCtx.getImageData(0, 0, imageCanvasRef.value.width, imageCanvasRef.value.height);
    if (currentImageData.width !== imageData.width || currentImageData.height !== imageData.height) {
        ElNotification({
            message: '叠加层图像尺寸与原始图像尺寸不一致，无法叠加',
            type: 'error',
        });
        return;
    }
    // 原始图像与叠加层图像逐像素对比，若相似度小于相似度配置，则将图像置为透明？黑色？
    const agHelper = new GraphicHelper(); // 先使用该工具的相似度，后续可能提供配置使用什么相似度算法
    imageCtx.fillStyle = '#000000';
    const t2 = Date.now();
    for (let y = 0; y < currentImageData.height; y++) {
        for (let x = 0; x < currentImageData.width; x++) {
            const currentColor = pixOfImageDataArray(currentImageData, x, y, true);
            if (currentColor[3] < 255) continue; // 如果当前位置已经叠加过且相似度小于相似度配置，则没必要继续计算
            const superpositionColor = pixOfImageDataArray(imageData, x, y);
            const similarity = agHelper.Similarity(currentColor, superpositionColor);
            if (similarity < currentSimilarity) {
                // imageCtx.fillRect(x, y, 1, 1);
                imageCtx.clearRect(x, y, 1, 1);
            }
        }
    }
    console.log(`${fileName}: 叠加耗时: ${Date.now() - t2}ms`);

    // 栈顶
    if (superpositionImageStackCurrentIndex.value === superpositionImageStack.value.length) {
        superpositionImageStack.value.push(imageCtx.getImageData(0, 0, imageCanvasRef.value.width, imageCanvasRef.value.height));
    } else if (superpositionImageStackCurrentIndex.value < superpositionImageStack.value.length) { // 撤销过
        superpositionImageStack.value.splice(superpositionImageStackCurrentIndex.value);
        superpositionImageStack.value.push(imageCtx.getImageData(0, 0, imageCanvasRef.value.width, imageCanvasRef.value.height));
    }
    if (superpositionImageStack.value.length > superpositionImageStackMaxSize) {
        superpositionImageStack.value.shift();
    }
    superpositionImageStackCurrentIndex.value = superpositionImageStack.value.length;
}


const randomId = Math.random().toString(36).substring(2, 8);

// 快捷键ID
const shortcutIds = {
    esc: `imageboard-esc-${randomId}`,
    ctrlWheelZoom: `imageboard-ctrl-wheel-zoom-${randomId}`,
    cmdWheelZoom: `imageboard-cmd-wheel-zoom-${randomId}`
};

// 注册快捷键
const registerShortcuts = () => {
    // ESC键：关闭放大镜
    shortcutManager.register({
        shortcut: {
            id: shortcutIds.esc,
            description: '关闭放大镜',
            key: 'Escape',
            enabled: true,
            preventDefault: true,
            stopPropagation: true,
            handler: (event: KeyboardEvent | WheelEvent) => {
                if (magnifierShown.value) {
                    magnifierShown.value = false;
                }
            }
        }
    });

    // Ctrl+鼠标滚轮：缩放图片
    shortcutManager.register({
        shortcut: {
            id: shortcutIds.ctrlWheelZoom,
            description: 'Ctrl+鼠标滚轮缩放图片',
            modifier: 'ctrl',
            enabled: true,
            preventDefault: true,
            stopPropagation: true,
            handler: (event: WheelEvent) => {
                const delta = event.deltaY > 0 ? -1 : 1;
                zoomRatio.value += delta * zoomStep.value;
                zoomRatio.value = Math.min(Math.max(zoomRatio.value, 1), 2000);
                handleZoom();
            }
        }
    });

    // Cmd+鼠标滚轮 (Mac)：缩放图片
    shortcutManager.register({
        shortcut: {
            id: shortcutIds.cmdWheelZoom,
            description: 'Cmd+鼠标滚轮缩放图片 (Mac)',
            modifier: 'meta',
            enabled: true,
            preventDefault: true,
            stopPropagation: true,
            handler: (event: WheelEvent) => {
                const delta = event.deltaY > 0 ? -1 : 1;
                zoomRatio.value += delta * zoomStep.value;
                zoomRatio.value = Math.min(Math.max(zoomRatio.value, 1), 2000);
                handleZoom();
            }
        }
    });
};

// 注销快捷键
const unregisterShortcuts = () => {
    shortcutManager.unregister(shortcutIds.esc);
    shortcutManager.unregister(shortcutIds.ctrlWheelZoom);
    shortcutManager.unregister(shortcutIds.cmdWheelZoom);
};

onMounted(() => {
    emitter.on('Event.ColorHelper.Settings.change', (option) => {
        console.log('Event.ColorHelper.Settings.change', option);
        loadThisSettings(option);
    });
    loadThisSettings();

    emitter.on('Event.ColorHelper.AdbHelper.canScreencapStatusChange', (status: boolean) => {
        console.log('Event.ColorHelper.AdbHelper.canScreencapStatusChange', status);
        if (status) {
            canAdbScreencap.value = true;
        } else {
            canAdbScreencap.value = false;
        }
    });

    // 根据初始的actived状态注册快捷键
    if ($props.actived) {
        registerShortcuts();
    }

    imageCtx = imageCanvasRef.value.getContext('2d', { willReadFrequently: true });

    imageCtx.imageSmoothingEnabled = false;
    maskCtx = maskCanvasRef.value.getContext('2d');
    maskCtx.imageSmoothingEnabled = false;
    magnifierCtx = magnifierCanvasRef.value.getContext('2d');
    magnifierCtx.imageSmoothingEnabled = false;

    imageCanvasRef.value.style.imageRendering = 'pixelated';
    maskCanvasRef.value.style.imageRendering = 'pixelated';
    magnifierCanvasRef.value.style.imageRendering = 'pixelated';

    resize();
    window.addEventListener('resize', resize);

    magnifierCanvasRef.value.width = imgClipWidth * magnifierScale;
    magnifierCanvasRef.value.height = imgClipWidth * magnifierScale;
    // 调试用，关闭图片平滑显示后，使用小width+height配合大style的width+height仍然会模糊
    // magnifierCanvasRef.value.style.width = (15 * 15) + 'px';
    // magnifierCanvasRef.value.style.height = (15 * 15) + 'px';
    // magnifierCanvasRef.value.style.transform = 'scale(15)';

    if ($props.src) {
        loadImage($props.src);
    }
});

onUnmounted(() => {
    window.removeEventListener('resize', resize);
    emitter.off('Event.ColorHelper.Settings.change');
    emitter.off('Event.ColorHelper.AdbHelper.canScreencapStatusChange');

    // 注销ImageBoard的快捷键
    unregisterShortcuts();
});

/**
 * 给画板加载图片
 * @param src 图片地址或dataURL
 * @param timeout  超时时间，默认5000ms
 */
const loadImage = async (src: string | HTMLImageElement, timeout: number = 5000) => {
    return new Promise((resolve, reject) => {
        if (src instanceof HTMLImageElement) {
            imageCanvasRef.value.style.width = img.width + 'px';
            imageCanvasRef.value.style.height = img.height + 'px';
            imageCanvasRef.value.width = img.width;
            imageCanvasRef.value.height = img.height;

            maskCanvasRef.value.width = img.width;
            maskCanvasRef.value.height = img.height;
            imageCtx.clearRect(0, 0, imageCanvasRef.value.width, imageCanvasRef.value.height);
            imageCtx.drawImage(img, 0, 0, img.width, img.height);
            centerPoint.setRegion([0, 0, img.width - 1, img.height - 1]);
            imgLoaded.value = true;
            handleZoom();
            resolve(true);
        } else if (typeof src === 'string') {
            const timmer = setTimeout(() => {
                ElNotification({
                    message: '图片加载超时',
                    type: 'error',
                });
                reject(`load image timeout: ${timeout}`);
            }, timeout);
            img = new Image();
            img.onload = () => {
                clearTimeout(timmer);
                imageCanvasRef.value.style.width = img.width + 'px';
                imageCanvasRef.value.style.height = img.height + 'px';
                imageCanvasRef.value.width = img.width;
                imageCanvasRef.value.height = img.height;

                maskCanvasRef.value.width = img.width;
                maskCanvasRef.value.height = img.height;
                imageCtx.clearRect(0, 0, imageCanvasRef.value.width, imageCanvasRef.value.height);
                imageCtx.drawImage(img, 0, 0, img.width, img.height);
                centerPoint.setRegion([0, 0, img.width - 1, img.height - 1]);
                imgLoaded.value = true;
                handleZoom();
                resolve(true);
            }
            img.onerror = () => {
                ElNotification({
                    message: '图片加载失败',
                    type: 'error',
                });
                clearTimeout(timmer);
                reject(`Image failed to load.`);
            }
            img.src = src;
        }
    });
}

const uploadFileChangeEvent = async (file: any) => {
    loadImage(await fileToDataURL(file.raw));
}

const resetImageBtnEvent = async (e: MouseEvent) => {
    loadImage(img);
    superpositionImageStack.value = [];
    superpositionImageStackCurrentIndex.value = 0;
}

const canAdbScreencap = ref<boolean>(adbHelper.canScreencap());
const loadingScreenCap = ref<boolean>(false);
const superpositionAdbScreencap = async (e: MouseEvent) => {
    loadingScreenCap.value = true;
    try {
        const t1 = Date.now();
        const screenReulst = await adbHelper.screencap();
        const t2 = Date.now();
        console.log(`adbHelper.screencap: ${t2 - t1}ms`);
        const imageData = await parseToImageData(screenReulst.dataUrl);
        superpositionImageData(screenReulst.fileName, imageData);
    } catch (e) {
        console.error(e);
    }
    loadingScreenCap.value = false;
}

const superpositionUndo = async (e: MouseEvent) => {
    --superpositionImageStackCurrentIndex.value;
    if (superpositionImageStackCurrentIndex.value === 0) {
        imageCtx.drawImage(img, 0, 0, imageCanvasRef.value.width, imageCanvasRef.value.height);
    } else {
        const currentImageData = superpositionImageStack.value[superpositionImageStackCurrentIndex.value - 1];
        imageCtx.putImageData(currentImageData, 0, 0);
    }
}

const superpositionRedo = async (e: MouseEvent) => {
    const currentImageData = superpositionImageStack.value[superpositionImageStackCurrentIndex.value++];
    imageCtx.putImageData(currentImageData, 0, 0);
}

const zoomStep = ref(25);
const zoomRatio = ref(parseInt(localStorage.getItem('ColorHelper.Settings.default.zoom')));
/**
 * 处理图片缩放
 * @param delta 缩放方向：1为放大，-1为缩小
 */
const handleZoom = () => {
    if (!imgLoaded.value) return;

    imageCanvasRef.value.style.width = (img.width * zoomRatio.value / 100) + 'px';
    imageCanvasRef.value.style.height = (img.height * zoomRatio.value / 100) + 'px';
    maskCanvasRef.value.style.width = (img.width * zoomRatio.value / 100) + 'px';
    maskCanvasRef.value.style.height = (img.height * zoomRatio.value / 100) + 'px';


    // 将原始图片坐标转换为显示坐标
    const [displayX, displayY] = imageToDisplayCoord(centerPoint.x, centerPoint.y);

    // 计算容器中心位置
    const container = maskCanvasRef.value.parentElement;
    const containerWidth = container.clientWidth;
    const containerHeight = container.clientHeight;

    // 计算滚动位置，使centerPoint位于容器中心
    const targetScrollLeft = Math.max(0, displayX - containerWidth / 2);
    const targetScrollTop = Math.max(0, displayY - containerHeight / 2);

    // 设置滚动位置
    container.scrollTo(targetScrollLeft, targetScrollTop);

    // // 缩放后重新绘制mask，确保centerPoint在可视区域内
    // drawMask();
    // // 同时更新放大镜位置
    // magnifierRefresh();
};


const handleDescCoordinateChange = (scope: any) => {
    console.log(scope);
    const row: PositionRowData = scope.row;

    // TODO 验证放到rules里面去实现，暂时不做
    const r = row.coordinate.match(/^(\d+),\s*(\d+)$/)
    if (!r) {
        row.similarity = 0;
        return;
    } else {
        const x = parseInt(r[1]);
        const y = parseInt(r[2]);
        if (x < 0 || x >= imageCanvasRef.value.width || y < 0 || y >= imageCanvasRef.value.height) {
            row.similarity = 0;
            return;
        }
        row.similarity = currentExecutor.colorSimilarity(asColorArray(row.color), pixOfImageDataArray(imageCtx.getImageData(0, 0, imageCanvasRef.value.width, imageCanvasRef.value.height), x, y));
    }
}

const prefersDark = window.matchMedia('(prefers-color-scheme: dark)')

const updateDarkClass = (e?: MediaQueryListEvent) => {
    if ((e ? e.matches : prefersDark.matches)) {
        document.documentElement.classList.add('dark')
    } else {
        document.documentElement.classList.remove('dark')
    }
}

// 页面初次执行
updateDarkClass()

// 监听用户切换浏览器暗色模式
prefersDark.addEventListener('change', updateDarkClass)
const copyExportText = () => {
    if (!exportPositionDataText.value) return
    navigator.clipboard.writeText(exportPositionDataText.value)
        .then(() => {
            // 成功提示
            ElNotification({
                title: '成功',
                message: '已复制到剪贴板',
                type: 'success',
            })
            exportPositionDataPopVisible.value = false
        })
        .catch(() => {
            ElNotification({
                title: '失败',
                message: '复制失败，请手动复制',
                type: 'error',
            })
        })
}

</script>

<template>
    <div style="width: 100%; height: 100%;">
        <div class="container-datatable" style="padding: 0px;width: 315px;">
            <div class="positionData-table-toolbar">
                <el-row>
                    <el-button-group>
                        <el-upload multiple :on-change="superpositionFileChange" accept="image/*" :auto-upload="false"
                            :show-file-list="false">
                            <el-button style="width: 105.8px">图片叠加&nbsp;
                                <el-tooltip placement="bottom" effect="light">
                                    <template #content>
                                        <div style="width: 200px">
                                            <el-text size="default">
                                                选择一个或多个图片，与当前图片叠加，新图与原图逐像素对比，颜色相似（配置的相似度）则保留原来图片的颜色，否则清除该处颜色（使其透明）
                                            </el-text>
                                        </div>
                                    </template>
                                    <el-icon>
                                        <InfoFilled />
                                    </el-icon>
                                </el-tooltip>
                            </el-button>
                        </el-upload>
                        <el-button @click="superpositionAdbScreencap" size="default" v-if="canAdbScreencap"
                            style="width: 48px" :disabled="loadingScreenCap">
                            <el-icon v-if="loadingScreenCap" class="is-loading">
                                <Loading />
                            </el-icon>
                            <template v-if="!loadingScreenCap">截图</template>
                        </el-button>
                        <el-button size="default" @click="superpositionUndo"
                            :disabled="!(superpositionImageStackCurrentIndex >= 1)"><span class="iconfont icon-chexiao"
                                style="font-size: 12px;"></span></el-button>
                        <el-button size="default" @click="superpositionRedo"
                            :disabled="!(superpositionImageStackCurrentIndex !== superpositionImageStack.length)"><span
                                class="iconfont icon-zhongzuo" style="font-size: 12px;"></span></el-button>
                        <el-button style="width: 64.6px" @click="resetImageBtnEvent">重置</el-button>
                    </el-button-group>
                </el-row>
                <el-row>
                    <div>
                        <el-button-group>
                            <el-button style="width: 64.6px" @click="clearBtnClickEvent">清空</el-button>
                            <el-popover placement="bottom" :visible="exportPositionDataPopVisible" :width="400">
                                <template #reference>
                                    <el-button style="width: 64.6px" @click="exportPositionDataEvent">导出</el-button>
                                </template>
                                <div>
                                    <span
                                        style="margin-left: 8px; margin-bottom: 5px; display: inline-block; font-size: 12px; font-weight: bold;">导出</span>
                                    <el-input v-model="exportPositionDataText" style="width: 100%" type="textarea"
                                        :rows="8" />
                                    <div style="text-align: right;">
                                        <el-button type="primary" style="margin-top: 10px;" @click="copyExportText">
                                            复制文本
                                        </el-button>
                                        <el-button @click="exportPositionDataPopVisible = false" size="default"
                                            type="primary" style="margin-top: 10px;" link>关闭</el-button>
                                    </div>
                                </div>
                            </el-popover>
                            <el-popover placement="bottom" :visible="importPositionDataPopVisible" :width="400">
                                <template #reference>
                                    <el-button style="width: 64.6px"
                                        @click="importPositionDataPopVisible = true; exportPositionDataPopVisible = false">导入</el-button>
                                </template>
                                <div>
                                    <span
                                        style="margin-left: 8px; margin-bottom: 5px; display: inline-block; font-size: 12px; font-weight: bold;">导入</span>
                                    <el-input v-model="importPositionDataText" style="width: 100%" type="textarea"
                                        :rows="8" placeholder="请输入数据后点击确定 或 点击确定自动识别粘贴板" />
                                    <div style="text-align: right;">
                                        <el-button @click=closeImportPositionData size="default" type="primary"
                                            style="margin-top: 10px;" link>关闭</el-button>
                                        <el-button @click="importPositionDataEvent" size="default" type="primary"
                                            style="margin-top: 10px;">确定</el-button>
                                    </div>
                                </div>
                            </el-popover>
                            <el-button style="width: 64.6px" @click="testBtnClickEvent">测试</el-button>
                        </el-button-group>
                    </div>
                </el-row>
                <el-row>
                    <div>
                        <el-button-group>
                            <el-input-number v-model="zoomRatio" :min="50" :max="2000" :step="zoomStep"
                                style="width: 192px" @change="handleZoom">
                                <template #prefix>缩放</template>
                                <template #suffix>%</template>
                            </el-input-number>
                            <el-button style="width: 64.6px" @click="zoomRatio = zoom; handleZoom()">重置</el-button>
                        </el-button-group>
                    </div>
                </el-row>
            </div>
            <el-tabs v-model="positionRegionTabModel" class="position-region-tabs">
                <el-tab-pane name="色组">
                    <template #label>
                        色组
                        <el-badge :show-zero="false" :value="positionUnreadNum" size="mini"></el-badge>
                    </template>
                    <el-table :data="positionData" class="positionData-table" cell-class-name="positionData-table-cell"
                        header-cell-class-name="positionData-table-cell" row-class-name="positionData-table-row"
                        header-row-class-name="positionData-table-row"
                        @cell-mouse-enter="positionTableCellMouseEnterEvent"
                        @cell-mouse-leave="positionTableCellMouseLeaveEvent">
                        <el-table-column label="" width="20px">
                            <template #default="scope">
                                <div style="display: flex; justify-content: space-around;">
                                    <ColoredCheckbox v-model="scope.row.checked"
                                        :background-color="`#${scope.row.color.slice(2)}`" />
                                </div>
                            </template>
                        </el-table-column>
                        <el-table-column label="锚点" width="35px">
                            <template #default="scope">
                                <!-- <el-input v-model="scope.row.anchor" size="small" /> -->
                                <el-select v-model="scope.row.anchor" size="small" style="width: 43px"
                                    popper-class="anchor-select">
                                    <el-option label="N" value="N" />
                                    <el-option label="L" value="L" />
                                    <el-option label="C" value="C" />
                                    <el-option label="R" value="R" />
                                    <el-option label="T" value="T" />
                                    <el-option label="M" value="M" />
                                    <el-option label="B" value="B" />
                                </el-select>
                            </template>
                        </el-table-column>
                        <el-table-column label="坐标" width="80px">
                            <template #default="scope">
                                <el-input v-model="scope.row.coordinate" size="small"
                                    @change="handleDescCoordinateChange(scope)" />
                            </template>
                        </el-table-column>
                        <el-table-column label="颜色" width="70px" prop="color">
                            <template #default="scope">
                                <el-input v-model="scope.row.color" size="small" readonly />
                            </template>
                        </el-table-column>
                        <el-table-column label="相似度" width="50px" prop="color">
                            <template #default="scope">
                                <el-input v-model="scope.row.similarity" size="small" readonly />
                            </template>
                        </el-table-column>
                        <el-table-column label="操作" width="40px">
                            <template #default="scope">
                                <span style="padding-bottom: 2px; display: inline-block;">
                                    <el-popover placement="bottom" popper-class="positionData-operator-popper"
                                        :width="80" trigger="hover">
                                        <template #reference>
                                            <el-button link size="small">操作</el-button>
                                        </template>
                                        <div>
                                            <div>
                                                <el-button @click="positionDataDeleteRow(scope, $event)" link
                                                    size="small" type="danger">删除</el-button>
                                            </div>
                                            <div>
                                                <el-button @click="renewColor(scope, $event)" link size="small"
                                                    type="default">刷新颜色值</el-button>
                                            </div>
                                            <!-- <div><el-button link size="small">复制</el-button></div> -->
                                        </div>
                                    </el-popover>
                                </span>
                            </template>
                        </el-table-column>
                    </el-table>
                </el-tab-pane>
                <el-tab-pane name="区域">
                    <template #label>
                        区域
                        <el-badge :show-zero="false" :value="regionUnreadNum" size="mini"></el-badge>
                    </template>
                    <el-table :data="regionData" class="positionData-table" cell-class-name="positionData-table-cell"
                        header-cell-class-name="positionData-table-cell" row-class-name="positionData-table-row"
                        header-row-class-name="positionData-table-row"
                        @cell-mouse-enter="regionTableCellMouseEnterEvent"
                        @cell-mouse-leave="regionTableCellMouseLeaveEvent">
                        <el-table-column label="" width="20px">
                            <template #default="scope">
                                <div style="display: flex; justify-content: space-around;">
                                    <!-- <ColoredCheckbox v-model="scope.row.checked"
                                        :background-color="`#${scope.row.color.slice(2)}`" /> -->
                                    <el-checkbox v-model="scope.row.checked" style="height: 14px;" />
                                </div>
                            </template>
                        </el-table-column>
                        <el-table-column label="锚点" width="35px">
                            <template #default="scope">
                                <!-- <el-input v-model="scope.row.anchor" size="small" /> -->
                                <el-select v-model="scope.row.anchor" size="small" style="width: 43px"
                                    popper-class="anchor-select">
                                    <el-option label="N" value="N" />
                                    <el-option label="L" value="L" />
                                    <el-option label="C" value="C" />
                                    <el-option label="R" value="R" />
                                </el-select>
                            </template>
                        </el-table-column>
                        <el-table-column label="区域坐标" width="150px">
                            <template #default="scope">
                                <el-input :value="`${scope.row.x0},${scope.row.y0},${scope.row.x1},${scope.row.y1}`"
                                    size="small" readonly />
                            </template>
                        </el-table-column>
                        <el-table-column label="操作" width="40px">
                            <template #default="scope">
                                <span style="padding-bottom: 2px; display: inline-block;">
                                    <el-popover placement="bottom" popper-class="positionData-operator-popper"
                                        :width="20" trigger="hover">
                                        <template #reference>
                                            <el-button link size="small">操作</el-button>
                                        </template>
                                        <div>
                                            <div>
                                                <el-button @click="regionDataDeleteRow(scope, $event)" link size="small"
                                                    type="danger">删除</el-button>
                                            </div>
                                            <!-- <div><el-button link size="small">复制</el-button></div> -->
                                        </div>
                                    </el-popover>
                                </span>
                            </template>
                        </el-table-column>
                    </el-table>
                </el-tab-pane>
            </el-tabs>
        </div>
        <div class="container-board" ref="containerBoardRef" @scroll="magnifierRefresh">
            <!-- <el-scrollbar ref="imageCanvasScrollContainerRef"> -->
            <canvas class="image-canvas" ref="imageCanvasRef"></canvas> <!-- 仅画原图 -->
            <canvas class="mask-canvas" ref="maskCanvasRef" @mousemove="maskMouseMoveEvent"
                @mousedown.prevent="maskMouseDownEvent" @mouseup="maskMouseUpEvent" @mouseleave="maskMouseLeaveEvent"
                @keydown.prevent="maskKeyEvent" tabindex="0" @focus="maskFocusEvent" @blur="maskBlurEvent"></canvas>
            <!-- </el-scrollbar> -->
            <el-upload v-if="!imgLoaded" accept="image/*" class="mask-upload" drag :limit="2" :auto-upload="false"
                :show-file-list="false" :on-change="uploadFileChangeEvent">
                <el-icon class="el-icon--upload">
                    <UploadFilled />
                </el-icon>
                <div class="el-upload__text">
                    将图片拖放至此 或 <span style="font-style: italic; color:#007bff;">点击上传图片</span>
                </div>
            </el-upload>
            <!-- 在上面画命中点，准心之类的 -->
            <div ref="magnifierRef" class="magnifier" :style="'display: ' + (magnifierShown ? 'initial' : 'none')">
                <!-- <div ref="magnifierRef" class="magnifier" v-if="magnifierShown"> -->
                <span class="magnifier-close" @click.prevent="magnifierShown = false">×</span>
                <canvas ref="magnifierCanvasRef" width="150" height="150"></canvas>
                <div class="positionInfo">
                    <input type="text" class="positionInfo-position" :value="`${positionX},${positionY}`" readonly>
                    <input type="text" class="positionInfo-color" :value="`0x${positionColor}`" readonly>
                </div>
            </div>
        </div>
    </div>
</template>

<style scoped>
:deep(.el-tabs__content) {
    background-color: #ffffff;
    min-height: 300px;
}

@media (prefers-color-scheme: dark) {
    :deep(.el-tabs__content) {
        background-color: #1d1e1f !important;
    }
}

.container-board {
    position: relative;
    display: inline-block;
    /* float: left; */
    max-width: calc(100% - 335px);
    height: 100%;
    overflow: auto;
}

.container-datatable {
    width: 335px;
    display: inline-block;
    /* float: left; */
    height: 100%;
    overflow: auto;
    padding: 0px 10px;
}

.position-region-tabs {
    height: calc(100% - 100px);
}

.position-region-tabs .el-tab-pane {
    height: 100%;
    overflow: auto;
}

.container-board canvas {
    outline: none;
    user-select: none;
    display: block;
}

.mask-canvas {
    position: absolute;
    top: 0;
    left: 0;
}

.mask-upload {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
}

.magnifier {
    background-color: #eee;
    position: fixed;
    pointer-events: none;
    z-index: 14;
}

.magnifier-close {
    position: absolute;
    top: 0;
    right: 0;
    cursor: pointer;
    pointer-events: initial;
    color: #666;
    width: 18px;
    height: 18px;
    font-size: 18px;
    font-weight: bold;

}

.positionInfo {
    background-color: #666;
    pointer-events: initial;
}

.positionInfo-position {
    width: 80px;
}

.positionInfo-color {
    width: 80px;
}

.dinamic-color-checkbox {
    cursor: pointer;

}
</style>

<style>
/* .positionData-table {
    height: calc(100% - 24px)
} */

.positionData-table-cell .cell {
    padding: 0px 2px;
    text-align: center;
}

.positionData-table-row .el-table__cell {
    border-bottom: none !important;
    padding: 2px 0;
}

.positionData-table-cell input {
    font-family: Consolas, monaco, monospace;
}

.positionData-table-cell .el-select__wrapper.is-hovering,
.positionData-table-cell .el-select__wrapper,
.positionData-table-cell .el-input__wrapper {
    box-shadow: none;
    border-radius: 0;
    background: transparent;
}

.positionData-table-cell .el-select__wrapper.is-focus,
.positionData-table-cell .el-select__wrapper.is-hovering,
.positionData-table-cell .el-input__wrapper:hover,
.positionData-table-cell .el-input__wrapper.is-focus {
    border-bottom: 1px solid #cacaca;
}


.positionData-table-cell .el-select__wrapper.is-hovering>.el-select__suffix {
    display: flex;
}

.positionData-table-cell .el-select__wrapper>.el-select__suffix {
    display: none;
}

.anchor-select .el-select-dropdown__item {
    height: 18px;
    line-height: 18px;
    padding: 0 20px 0 20px;
    font-size: 12px;
}

.anchor-select .el-select-dropdown__list {
    padding: 1px 0;
}

.positionData-operator-popper {
    min-width: 50px !important;
    padding: 5px 5px 7px 5px !important;
    text-align: center;
}

.positionData-table-toolbar {
    height: 100px;
}

.mask-upload .el-upload-dragger {
    padding: 20px 10px;
}

/* 处理上传组件内的按钮不受btn-group样式影响 */
.el-button-group>div:not(:last-child) {
    margin-right: -1px;
}

.el-button-group>div:first-child button {
    border-bottom-right-radius: 0;
    border-top-right-radius: 0;
}

.el-button-group>div:last-child {
    border-bottom-left-radius: 0;
    border-top-left-radius: 0;
}

.el-button-group>div {
    float: left;
    position: relative;
    display: flex;
}
</style>
