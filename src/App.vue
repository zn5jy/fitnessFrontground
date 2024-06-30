<template>
    <div>
        <el-container>
            <el-header>
                <el-tag type="primary" size="large" effect="dark">运动健身教练</el-tag>
                <el-select v-model="value" placeholder="请选择动作" @change="change">
                    <el-option v-for="item in options" :key="item.value" :label="item.label" :value="item.value" :disabled="item.disabled"/>
                </el-select>
                <el-button id="startButton" type="success" :disabled="startStatus" @click="start">开始检测</el-button>
                <el-button id="stopButton" type="danger" :disabled="stopStatus" @click="stop">停止检测</el-button>
            </el-header>
            <el-main class="container">
                <div class="videos">
                    <el-empty class="empty" description="Camera" v-show="!isLoadedCamera"/>
                    <video id="videoElement" autoplay v-show="isLoadedCamera"></video>
                    <el-empty class="empty" description="Result" v-show="!isLoadedImage"/>
                    <img :src="currentFrame" alt="Video Frame" v-show="isLoadedImage">
                </div>
            </el-main>
            <el-footer>
                <el-text class="mx-1" type="primary" v-show="results[0].isShow || results[1].isShow">
                    conclusion:
                </el-text>
                <el-text class="mx-1" type="danger" v-show="results[0].isShow">
                    {{ results[0].actionName }} : {{ results[0].evaluation }}, count : {{ results[0].count }}
                </el-text>
                <br>
                <el-text class="mx-1" type="danger" v-show="results[1].isShow">
                    {{ results[1].actionName }} : {{ results[1].evaluation }}, count : {{ results[1].count }}
                </el-text>
            </el-footer>
        </el-container>
    </div>
    <canvas id="canvasElement" style="display: none;"></canvas>
</template>

<script>
export default {
    name: 'App',
}
</script>

<script setup>
import {ref} from "vue";
const currentFrame = ref('');
const value = ref('')
const options = [
    {
        value: '0',
        label: '侧平举',
    }
]
const isLoadedCamera = ref(false);
const isLoadedImage = ref(false);
const startStatus = ref(true);
const stopStatus = ref(true);
let action;
let socket;
let intervalId;
const results = ref(
    [
        {
            isShow: false,
            actionName:'',
            evaluation:'',
            count:0,
        },
        {
            isShow: false,
            actionName:'',
            evaluation:'',
            count:0,
        }
    ]
);

function start() {
    startStatus.value = true;
    stopStatus.value = false;
    const videoElement = document.getElementById('videoElement');
    const canvasElement = document.getElementById('canvasElement');
    canvasElement.width = 640;
    canvasElement.height = 480;
    const canvasContext = canvasElement.getContext('2d');
    socket = new WebSocket('ws://localhost:8080/ws/camera');

    navigator.mediaDevices.getUserMedia({ video: true })
        .then(stream => {
            videoElement.srcObject = stream;
        })
        .catch(error => {
            console.error('Error accessing camera:', error);
        });
    isLoadedCamera.value = true;

    intervalId = setInterval(() => {
        canvasContext.drawImage(videoElement, 0, 0, canvasElement.width, canvasElement.height);
        const imageData = canvasElement.toDataURL('image/jpeg', 1);
        if(imageData.length < 10){
            return;
        }
        socket.send(imageData);
    }, 200);

    socket.onopen = () => {
        console.log('WebSocket connected');
    };

    socket.onmessage = (event) => {
        isLoadedImage.value = true;
        let message = JSON.parse(event.data);
        currentFrame.value = message.frame;
        isLoadedImage.value = true;
        for (let i = 0; i < message.results.length; ++i) {
            if(message.results[i].actionNumber < results.value.length && message.results[i].ResultType === 'FAILURE'){
                results.value[message.results[i].actionNumber].isShow = true;
                results.value[message.results[i].actionNumber].actionName = message.results[i].actionName;
                results.value[message.results[i].actionNumber].evaluation = message.results[i].evaluation;
                results.value[message.results[i].actionNumber].count++;
            }
        }
    }

    socket.onclose = () => {
        console.log('WebSocket closed');
    };

}

function stop() {
    startStatus.value = false;
    stopStatus.value = true;
    socket.close();
    clearInterval(intervalId);
    isLoadedCamera.value = false;
    isLoadedImage.value = false;
}

function change(event) {
    startStatus.value = false;
    action = event;
}

</script>

<style scoped>
.container {
    display: flex;
    flex-direction: column;
    align-items: center;
    width: 100%;
}
.videos {
    display: flex;
    align-items: center;
    width: 100%;
}
video {
    flex: 1;
    max-width: 50%;
}

img{
    flex: 1;
    margin: 20px;
    max-width: 50%;
}

slot {
    display: flex;
    width: 100%;
}

textarea {
    width: 100%;
    height: 100px;
}

el-tag {
    width: 100%;
}

/deep/ .el-empty {
    width: 50%;
}


/deep/ .el-tag {
    width: 15%;
    height: 100%;
    font-size: 20px;
}

/deep/ .el-select {
    margin-left: 5%;
    width: 15%;
}

/deep/ .el-button {
    margin-left: 5%;
    height: 100%;
    font-size: 20px;
}

/deep/ .el-text {
    font-size: 20px;
}
</style>
