<template>
    <el-row>
        <el-col :span="10">
            <el-row>
                <el-col :span="8">
                    <el-switch v-model="auto" inline-prompt active-text="开始自动" inactive-text="关闭自动" />
                </el-col>
                <el-col :span="16">
                    <el-button type="primary" @click="handleScreenshot" :loading="loading">截图</el-button>
                </el-col>
            </el-row>
            <el-descriptions v-if="enhancedRecommendation" class="gear-info" title="装备信息" :column="1">
                <el-descriptions-item label="装备等级">
                    {{ level }}
                </el-descriptions-item>
                <el-descriptions-item label="装备级别">
                    {{ tier }} - {{ tierMapping[tier] }}
                </el-descriptions-item>
                <el-descriptions-item label="强化等级">
                    {{ enhancementLevel !== undefined ? '+' + enhancementLevel : '' }}
                </el-descriptions-item>
                <el-descriptions-item label="主属性">
                    {{ primaryAttribute[0] }} {{ primaryAttribute[1] }}
                </el-descriptions-item>
                <el-descriptions-item v-for="(item, index) in attribute" :key="index" :label="item[0]">
                    {{ item[1] }}
                </el-descriptions-item>
                <el-descriptions-item label="套装">{{ set }}</el-descriptions-item>
            </el-descriptions>
        </el-col>
        <el-col :span="14">
            <el-row v-if="enhancedRecommendation">
                <el-col>
                    <el-text class="mx-1" size="large">当前分数：</el-text>
                    <el-text class="mx-1" size="large">{{ score }}</el-text>
                </el-col>
                <el-col :span="12">
                    <el-text class="mx-1" size="large">重铸前：预估平均分：</el-text>
                    <el-text class="mx-1" size="large">{{ expectantScore }}</el-text>
                </el-col>
                <el-col :span="12">
                    <el-text class="mx-1" size="large">最高分数：</el-text>
                    <el-text class="mx-1" size="large">{{ expectantMaxScore }}</el-text>
                </el-col>
                <el-col :span="12">
                    <el-text class="mx-1" size="large">重铸后：预估平均分：</el-text>
                    <el-text class="mx-1" size="large">{{ (expectantScore + 12).toFixed(2) }}</el-text>
                </el-col>
                <el-col :span="12">
                    <el-text class="mx-1" size="large">最高分数：</el-text>
                    <el-text class="mx-1" size="large">{{ (expectantMaxScore + 12).toFixed(2) }}</el-text>
                </el-col>
                <el-col>
                    <el-text class="mx-1" size="large">强化建议：</el-text>
                    <el-text class="mx-1" size="large">{{ enhancedRecommendation }}</el-text>
                </el-col>
                <el-col style="margin-top: 5px;">
                    <el-tag style="margin-right: 5px;" size="large" effect="dark" :type="recommendAbandon ? 'danger' : 'info'">放弃</el-tag>
                    <el-tag style="margin-right: 5px;" size="large" effect="dark" :type="recommendHighScore ? 'success' : 'info'">高分</el-tag>
                    <el-tag style="margin-right: 5px;" size="large" effect="dark" :type="recommendHighSpeed ? 'success' : 'info'">高速</el-tag>
                    <el-tag style="margin-right: 5px;" size="large" effect="dark" :type="warningPrimaryAttributeFixed ? 'warning' : 'info'">固定</el-tag>
                    <el-tag style="margin-right: 5px;" size="large" effect="dark" :type="warningBothEffAndRes ? 'warning' : 'info'">双效</el-tag>
                </el-col>
            </el-row>
        </el-col>
    </el-row>
</template>

<script lang="ts" setup>
import { onMounted, onUnmounted, ref, nextTick } from 'vue'
import { exec, spawn } from 'child_process'
import { useAdbStore } from '../store/adb'
import path from 'path'
import fs from 'fs'
import { ElMessage } from 'element-plus'
import { ipcRenderer } from 'electron'

let configData = require(path.join(process.cwd(), 'tiezhu.config.json'))
let tiezhuConfig = ref(configData)

let timer = ref(0)
const auto = ref(false)
const loading = ref(false)

const adbStore = useAdbStore()
const selectedDeviceId = adbStore.device
const Sharp = require('sharp')

const gearInfos = ref<[string][]>([])

const level = ref(0) // 装备等级，85、88、90
const enhancementLevel = ref(0) // 强化等级
const tier = ref('') // 装备级别，英雄（紫装）、传说（红装）等等
const part = ref('') // 装备部位，武器、头盔、铠甲、项链、戒指、鞋子
const set = ref('') // 套装
const primaryAttribute = ref<string[]>(['', '']) // 主属性
const attribute = ref<[string, string][]>([["", ""], ["", ""], ["", ""], ["", ""]]) // 副属性
const score = ref(0) // 当前分数
const enhancedRecommendation = ref('') // 强化建议文字
const expectantScore = ref(0) // 预估平均分
const expectantMaxScore = ref(0) // 预估最高分
const recommendAbandon = ref(false) // 推荐放弃
const recommendHighScore = ref(false) // 推荐高分
const recommendHighSpeed = ref(false) // 推荐高速
const warningPrimaryAttributeFixed = ref(false) // 警告主属性固定
const warningBothEffAndRes = ref(false) // 警告同时具有双效

const tierMapping: { [key: string]: string } = {
    "稀有": "蓝装",
    "英雄": "紫装",
    "传说": "红装"
}
const setMapping: { [key: string]: string } = {
    "破灭": "set_cri_dmg",
    "愤怒": "set_rage",
    "命中": "set_acc",
    "穿透": "set_penetrate",
    "攻击": "set_att",
    "抵抗": "set_res",
    "憎恨": "set_revenge",
    "防御": "set_def",
    "吸血": "set_vampire",
    "伤口": "set_scar",
    "生命": "set_max_hp",
    "反击": "set_counter",
    "守护": "set_shield",
    "速度": "set_speed",
    "夹攻": "set_coop",
    "激流": "set_torrent",
    "暴击": "set_cri",
    "免疫": "set_immune"
}
type StatsMapping = {
    [key in string]: string
}
const statsMapping: StatsMapping = {
    "攻击力": "att",
    "防御力": "def",
    "生命值": "max_hp",
    "效果命中": "acc",
    "效果抗性": "res",
    "速度": "speed",
    "暴击伤害": "cri_dmg",
    "暴击率": "cri"
}

onMounted(() => {
    ElMessage('始化中……')
    //重新加载配置文件
    delete require.cache[require.resolve(path.join(process.cwd(), 'tiezhu.config.json'))]
    configData = require(path.join(process.cwd(), 'tiezhu.config.json'))
    tiezhuConfig.value = configData

    // 开启自动执行的定时器
    timer.value = window.setInterval(autoExec, 1500)
})

onUnmounted(() => {
    // 移除自动执行的定时器
    window.clearInterval(timer.value)
    
    //停止监听
    // ipcRenderer.removeListener('query-result', queryResultListener)

    // 杀死子进程（如果有的话）
    if (child) {
        child.kill()
    }

    // 删除temp文件夹下的所有图片文件
    const tempFolderPath = path.join(process.cwd(), 'temp')
    if (fs.existsSync(tempFolderPath)) {
        fs.readdirSync(tempFolderPath).forEach((file) => {
            const filePath = path.join(tempFolderPath, file)
            if (fs.statSync(filePath).isFile() && /\.(png|jpg|jpeg|gif|bmp)$/i.test(file)) {
                fs.unlinkSync(filePath) // 删除图片文件
            }
        })
    }
})

const child = spawn(path.join(process.cwd(), 'PaddleOCR-json', 'PaddleOCR-json.exe'), {
    cwd: path.join(process.cwd(), 'PaddleOCR-json'),
    stdio: ['pipe', 'pipe', 'pipe']
})


const autoExec = () => {
    if (auto.value == false) {
        return
    }
    if (loading.value == true) {
        return
    }
    if (adbStore.status === 0) {
        return
    }
    handleScreenshot()
}

//截图
const handleScreenshot = async () => {
    if (adbStore.status === 0) {
        ElMessage({
            message: '尚未连接',
            type: 'error',
        })
        return
    }
    loading.value = true
    await nextTick();
    const adbPath = path.join(process.cwd(), 'platform-tools', 'adb.exe')
    const tempFolderPath = path.join(process.cwd(), 'temp') // 指定temp文件夹的路径
    const screenshotFilePath = path.join(tempFolderPath, 'screenshot.png') // 指定截图文件的路径
    const adbCommand = adbPath + ' -s ' + selectedDeviceId + ' shell screencap -p /sdcard/screenshot.png && ' + adbPath + ' -s ' + selectedDeviceId + ' pull /sdcard/screenshot.png ' + screenshotFilePath

    // 检查temp文件夹是否存在，不存在则创建它
    if (!fs.existsSync(tempFolderPath)) {
        fs.mkdirSync(tempFolderPath)
    }

    gearInfos.value = [];

    exec(adbCommand, async (error, stdout, stderr) => {
        if (error) {
            console.error('截图错误:', error)
            return
        }
        if (stderr) {
            console.error('截图错误:', stderr)
            return
        }
        // 检查 stdout 是否包含 "file pulled" 字符串
        if (stdout.includes("file pulled")) {
            await screenshotAndOCR()
        } else {
            console.error("截图失败，未能成功拉取文件")
        }
    })
    function getGearInfoWithRetry(times: number) {
        if ( times > 15 ) {
            loading.value = false
            return
        }
        if (gearInfos.value.length > 0) {
            getGearInfo(gearInfos.value[0])
            loading.value = false
        } else {
            setTimeout(function () {
                getGearInfoWithRetry(times + 1);
            }, 300)
        }
    }
    setTimeout(function () {
        getGearInfoWithRetry(1)
    }, 1000)

    const activeElement = document.activeElement as HTMLElement
    if (activeElement) {
        activeElement.blur()
    }
}

//获取装备信息
const screenshotAndOCR = async () => {
    const processedImagePath1 = path.join('temp', 'gear_info_1.png') // 使用 path.join 拼接路径
    const cropOptions1 = { left: 35, top: 102, width: 435, height: 500 }
    const blackOverlay1 = Buffer.from(
        `<svg width="435" height="500">
            <rect x="55" y="32" width="80" height="20" fill="black" />
            <rect x="0" y="380" width="435" height="60" fill="black" />
            <rect x="0" y="50" width="435" height="110" fill="black" />
            <rect x="0" y="160" width="45" height="60" fill="black" />
            <rect x="0" y="210" width="435" height="30" fill="black" />
            <rect x="0" y="440" width="60" height="60" fill="black" />
        </svg>`
    )
    // <rect x="0" y="0" width="85" height="60" fill="black" />
    await Sharp(path.join('temp', 'screenshot.png')) // 使用 path.join 拼接路径
        .resize(1600, 900)
        .extract(cropOptions1)
        .composite([{ input: blackOverlay1, top: 0, left: 0 }])
        .toFile(processedImagePath1)
    await textOcr(processedImagePath1)

    const processedImagePath2 = path.join('temp', 'gear_info_2.png') 
    const cropOptions2 = { left: 415, top: 137, width: 370, height: 550 }
    const blackOverlay2 = Buffer.from(
        `<svg width="370" height="550">
            <rect x="55" y="32" width="80" height="20" fill="black" />
            <rect x="0" y="460" width="370" height="40" fill="black" />
            <rect x="0" y="50" width="370" height="200" fill="black" />
            <rect x="0" y="250" width="45" height="45" fill="black" />
            <rect x="0" y="490" width="60" height="60" fill="black" />
        </svg>`
    )
    // <rect x="0" y="0" width="85" height="60" fill="black" />
    await Sharp(path.join('temp', 'screenshot.png')) // 使用 path.join 拼接路径
        .resize(1600, 900)
        .extract(cropOptions2)
        .composite([{ input: blackOverlay2, top: 0, left: 0 }])
        .toFile(processedImagePath2)
    await textOcr(processedImagePath2)
}

//Ocr识别
const textOcr = async (imagePath: string): Promise<any> => {
    // 准备待发送的指令
    const imgObj = { "image_path": path.join(process.cwd(), imagePath) }
    const imgStr = JSON.stringify(imgObj) + "\n"
    child.stdin.write(imgStr)
}

// 从子进程接收数据
child.stdout.on('data', (data: Buffer) => {
    let strOut: string = data.toString()
    // 检测启动成功
    if (strOut.includes('OCR init completed.')) {
        console.log('初始化完成！')
        ElMessage({
            message: '初始化完成！',
            type: 'success',
        })
    } else if (strOut.includes('PaddleOCR-json v1.3.0')) {
        console.log(strOut)
    } else {
        try {
            // console.log(strOut)
            let jsonOutput = JSON.parse(strOut)
            if (jsonOutput.code === 100) {
                let gearInfo = jsonOutput.data
                    .filter((item: { score: number }) => item.score >= 0.5)
                    .map((item: { text: string }) => item.text)
                if (gearInfo.length <= 10) {
                    return
                }
                if (gearInfo[0] == "背包") {
                    return
                }
                gearInfos.value.push(gearInfo);
            } else {
                if (jsonOutput.code === 101) {
                    ElMessage({
                        message: '没有获取到数据，请确认图片内容',
                        type: 'error',
                    })
                }
                console.log('Code is not 100, original output:', strOut)
            }
        } catch (e) {
            console.error('Error parsing JSON:', e)
        }
    }
})

const getGearInfo = (gearInfo: string[]) => {
    // console.info("gearInfo="+gearInfo)
    if (gearInfo[0].startsWith("+")) {
        //获取强化等级
        const matchResult = gearInfo[0].match(/\d+/)
        if (matchResult) {
            enhancementLevel.value = parseInt(matchResult[0])
            gearInfo.shift() // 移除数组的第一个元素
        } else {
            enhancementLevel.value = 0
        }

        //获取装备等级
        const matchResult2 = gearInfo[0].match(/\d+/)
        if (matchResult2) {
            level.value = parseInt(matchResult2[0])
        } else {
            level.value = 0
        }
        gearInfo.shift()
    } else if (/\d{2}/.test(gearInfo[0])) {
        //获取装备等级
        const matchResult2 = gearInfo[0].match(/\d+/)
        if (matchResult2) {
            level.value = parseInt(matchResult2[0])
        } else {
            level.value = 0
        }
        gearInfo.shift()

        if (gearInfo[0].startsWith("+")) {
            //获取强化等级
            const matchResult = gearInfo[0].match(/\d+/)
            if (matchResult) {
                enhancementLevel.value = parseInt(matchResult[0])
                gearInfo.shift() // 移除数组的第一个元素
            } else {
                enhancementLevel.value = 0
            }
        } else {
            enhancementLevel.value = 0
        }
    } else {
        level.value = 0
        enhancementLevel.value = 0
    }
    // 获取装备级别和装备部位
    tier.value = gearInfo[0].slice(0, 2)
    part.value = gearInfo[0].slice(2)
    gearInfo.shift()
    //是否自动切换
    const isPart = ["项链", "戒指", "鞋子", "武器", "头盔", "铠甲"].includes(part.value)
    if (!isPart) {
        return
    }
    //获取主属性
    primaryAttribute.value = gearInfo.slice(0, 2)
    gearInfo.splice(0, 2)
    //获取套装
    let lastItem = gearInfo[gearInfo.length - 1]
    const indexOfSuit = lastItem.indexOf("套")
    if (indexOfSuit !== -1) {
        set.value = lastItem.slice(0, indexOfSuit)
    }
    gearInfo.pop()
    //获取副属性
    const enhancementIndex = gearInfo.indexOf("强化+12时获得")
    if (enhancementIndex !== -1) {
        gearInfo = gearInfo.slice(0, enhancementIndex)
    }
    type AttributePair = [string, string]
    let pairedAttributes: AttributePair[] = []
    for (let i = 0; i < gearInfo.length; i += 2) {
        // 使用正则表达式匹配并保留中文字符 防止识别到重铸标志
        const matchedChinese = gearInfo[i].match(/[\u4e00-\u9fa5]/g)
        const chineseOnly = matchedChinese ? matchedChinese.join('') : ''
        pairedAttributes.push([chineseOnly, gearInfo[i + 1]])
    }
    attribute.value = pairedAttributes
    //计算分数
    score.value = calculateScore(attribute.value)
    //装备推荐
    calculateAnalysis()
    expectantScore.value = parseFloat((expectant() + score.value).toFixed(2))
    expectantMaxScore.value = parseFloat((calculateMaxScore() + score.value).toFixed(2))

    ipcRenderer.send('query-database', translateSetName(set.value))
}

const calculateScore = (attribute: [string, string][]): number => {
    let score = 0
    for (let [name, value] of attribute) {
        switch (name) {
            case "攻击力":
                if (value.includes("%")) {
                    score += parseFloat(value.replace('%', '')) * 1
                } else {
                    score += parseFloat(value) * 3.46 / 39
                }
                break
            case "防御力":
                if (value.includes("%")) {
                    score += parseFloat(value.replace('%', '')) * 1
                } else {
                    score += parseFloat(value) * 4.99 / 31
                }
                break
            case "生命值":
                if (value.includes("%")) {
                    score += parseFloat(value.replace('%', '')) * 1
                } else {
                    score += parseFloat(value) * 3.09 / 174
                }
                break
            case "效果命中":
            case "效果抗性":
                score += parseFloat(value.replace('%', '')) * 1
                break
            case "速度":
                score += parseFloat(value) * 2
                break
            case "暴击伤害":
                score += parseFloat(value.replace('%', '')) * 1.143
                break
            case "暴击率":
                score += parseFloat(value.replace('%', '')) * 1.6
                break
        }
    }
    const roundedScore = parseFloat(score.toFixed(2))
    return roundedScore
}

const calculateAnalysis = () => {
    enhancedRecommendation.value = '' 
    recommendHighScore.value = false
    recommendHighSpeed.value = false
    recommendAbandon.value = false
    warningPrimaryAttributeFixed.value = false
    warningBothEffAndRes.value = false

    if (["项链", "戒指", "鞋子"].includes(part.value)
        && ["攻击力", "防御力", "生命值"].includes(primaryAttribute.value[0]) 
        && !primaryAttribute.value[1].includes('%')) {
        enhancedRecommendation.value = "固定值主属性，建议放弃"
        warningPrimaryAttributeFixed.value = true
        recommendAbandon.value = true
        return
    }

    let speed = 0
    let hasEff = false // 判断是否包含效果命中
    let hasRes = false // 判断是否包含效果抗性
    for (let [name, value] of attribute.value) {
        if (name == "速度") {
            speed = parseInt(value)
        } else if (name == "效果命中") {
            hasEff = true
        } else if (name == "效果抗性") {
            hasRes = true
        }
    }

    if(hasEff && hasRes) warningBothEffAndRes.value = true

    if ("鞋子" != part.value && tier.value == "英雄") {
        if (enhancementLevel.value >= 0 && enhancementLevel.value <= 2) {
            recommendHighSpeed.value = speed >= 3
        } else if (enhancementLevel.value >= 3 && enhancementLevel.value <= 5) {
            recommendHighSpeed.value = speed >= 6
        } else if (enhancementLevel.value >= 6 && enhancementLevel.value <= 8) {
            recommendHighSpeed.value = speed >= 9
        } else if (enhancementLevel.value >= 9 && enhancementLevel.value <= 14) {
            recommendHighSpeed.value = speed >= 12
        } else if (enhancementLevel.value == 15 ) {
            recommendHighSpeed.value = speed >= 16
        } 
    } else if ("鞋子" != part.value && tier.value == "传说") {
        if (enhancementLevel.value >= 0 && enhancementLevel.value <= 2) {
            recommendHighSpeed.value = speed >= 3
        } else if (enhancementLevel.value >= 3 && enhancementLevel.value <= 5) {
            recommendHighSpeed.value = speed >= 6
        } else if (enhancementLevel.value >= 6 && enhancementLevel.value <= 8) {
            recommendHighSpeed.value = speed >= 9
        } else if (enhancementLevel.value >= 9 && enhancementLevel.value <= 11) {
            recommendHighSpeed.value = speed >= 10
        } else if (enhancementLevel.value >= 12 && enhancementLevel.value <= 14) {
            recommendHighSpeed.value = speed >= 12
        } else if (enhancementLevel.value == 15 ) {
            recommendHighSpeed.value = speed >= 16
        } 
    }
    
    let initScore = tiezhuConfig.value.scoreThreshold.left;
    if (["项链", "戒指", "鞋子"].includes(part.value)) {
        initScore = tiezhuConfig.value.scoreThreshold.right;
    }
    if (enhancementLevel.value < 3 && score.value >= initScore) recommendHighScore.value = true 
    else if (enhancementLevel.value < 6 && score.value >= initScore + 6) recommendHighScore.value = true // 28
    else if (enhancementLevel.value < 9 && score.value >= initScore + 13) recommendHighScore.value = true // 35
    else if (enhancementLevel.value < 12 && score.value >= initScore + 20) recommendHighScore.value = true // 42
    else if (enhancementLevel.value < 15 && score.value >= initScore + 27) recommendHighScore.value = true // 49
    else if (enhancementLevel.value == 15 && score.value >= initScore + 34) recommendHighScore.value = true // 56

    if (recommendHighScore.value == true) {
        if (enhancementLevel.value < 15) enhancedRecommendation.value = "继续强化"
        else enhancedRecommendation.value = "继续重铸"
    } else if (recommendHighSpeed.value == true) {
        if (enhancementLevel.value < 15) enhancedRecommendation.value = "继续赌速"
        else enhancedRecommendation.value = "继续重铸"
    } else {
        recommendAbandon.value = true
        enhancedRecommendation.value = "建议放弃"
    }
}

const expectant = (): number => {
    let expectant = 0
    for (let [name, value] of attribute.value) {
        switch (name) {
            case "攻击力":
                if (value.includes("%")) {
                    expectant += (4 + 8) / 2
                } else {
                    expectant += (33 + 46) / 2 / 39
                }
                break
            case "防御力":
                if (value.includes("%")) {
                    expectant += (4 + 8) / 2
                } else {
                    expectant += (28 + 35) / 2 / 31
                }
                break
            case "生命值":
                if (value.includes("%")) {
                    expectant += (4 + 8) / 2
                } else {
                    expectant += (157 + 202) / 2 / 174
                }
                break
            case "效果命中":
            case "效果抗性":
                expectant += (4 + 8) / 2
                break
            case "速度":
                expectant += (2 + 4) / 2 * 2
                break
            case "暴击伤害":
                if (value.includes("%")) {
                    expectant += (4 + 7) / 2 * 1.143
                }
                break
            case "暴击率":
                if (value.includes("%")) {
                    expectant += (3 + 5) / 2 * 1.6
                }
                break
        }
    }
    return expectant / 4 * Math.floor((17 - enhancementLevel.value) / 3)
}

const calculateMaxScore = (): number => {
    return 8 * Math.floor((17 - enhancementLevel.value) / 3)
}

// 监听退出事件
child.on('close', () => {
    console.log('子进程退出')
})

const translateSetName = (cnName: string) => {
    return setMapping[cnName]
}
const translateStatName = (cnStatName: string): string => {
    return statsMapping[cnStatName]
}
</script>

<style lang="scss">
.gear-info {
    margin-top: 20px;
    padding-right: 20px;
}

.gear-info .el-descriptions__label {
    width: 100px;
    display: inline-block;
}

.gear-info .el-descriptions__cell {
    padding-bottom: 5px !important;
}

.hero-info .el-descriptions__label {
    width: 48px;
    display: inline-block;
}

.hero-info .el-descriptions__cell {
    padding-bottom: 0 !important;
    line-height: 18px !important;
}

.hero-info {
    margin-bottom: 10px;
}
</style>
  