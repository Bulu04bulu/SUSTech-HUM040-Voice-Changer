<script setup>
</script>

<script>
import Recorder from "recorder-core";
import 'recorder-core/src/engine/wav'
import 'recorder-core/src/extensions/waveview'
import { ElNotification } from "element-plus";
import { ref } from "vue";
import AxiosFunctions from '../src/utils/api.js'

import { analysisApi, sovitsApi } from "../src/utils/api.js";

const bitRate = 16
const sampleRate = 24000

export default {
  data() {
    return {
      isRecording: false,
      soundLength: 0,
      wave: [],
      isPlaying: false,
      playedLength: 0,
      playedPercentage: ref(0),
      analysisApi: analysisApi,
      sovitsApi: sovitsApi    
    }
  },

  methods: {
    recOpen() {
      this.rec = Recorder({
        type: "wav",
        sampleRate: sampleRate,
        bitRate: bitRate,
        onProcess: (buffers, powerLevel, bufferDuration, bufferSampleRate, newBufferIdx, asyncEnd) => {
          if (this.wave) this.wave.input(buffers[buffers.length - 1], powerLevel, bufferSampleRate)
        }
      });
      this.rec.open(() => {
        if (this.$refs.recwave) {
          this.wave = Recorder.WaveView({ elem: this.$refs.recwave })
        }
      }, (msg, isUserNotAllow) => {
        if (isUserNotAllow) {
          ElNotification({
            title: "Failed",
            message: "请同意浏览器录音！",
            type: "error"
          })
        }
      });
    },

    recStart() {
      if (this.rec && !this.isRecording) {
        this.isRecording = true
        this.rec.start()
      }
    },

    recStop() {
      if (!this.rec) {
        return
      }
      this.isRecording = false
      this.rec.stop((blob, duration) => {
        this.recBlob = blob
        this.soundLength = duration
      }, (err) => {
      });
    },

    upload(url) {
      let blob = this.recBlob;
      if (!blob) {
        ElNotification({
          title: "Failed",
          message: "还没有录音！",
          type: "error"
        });
        return;
      }
      let fileName = "voice.wav";
      let wavFile = new File([blob], fileName);
      console.log("Uploading to URL:", url); // Add this line for debugging
      AxiosFunctions.methods.update(wavFile, url, sampleRate)
        .then((response) => {
          console.log("Response received:", response);
          if (url === this.analysis_api) {
            this.result_analysis = response.data;
          } else {
            this.result_audio = response.data;
          }
          ElNotification({
            title: "Success",
            message: "完成",
            type: "success"
          });
        })
        .catch((error) => {
          ElNotification({
            title: "Failed",
            message: "网络错误",
            type: "error"
          });
          console.log("Error occurred:", error);
        });
    },


    play(sound) {
      this.isPlaying = true
      this.playedLength = 0
      this.playedPercentage = 0
      let localUrl = URL.createObjectURL(sound)
      let audio = document.createElement("audio")
      audio.controls = true
      // document.body.appendChild(audio)  // show it on the web page
      audio.src = localUrl
      audio.play()
      setTimeout(function () {
        URL.revokeObjectURL(audio.src)
      }, 5000)
    },

    recPlay() {
      if (this.recBlob) {
        this.play(this.recBlob)
      } else {
        ElNotification({
          title: "Failed",
          message: "还没有录音！",
          type: "error"
        })
      }
    },

    // show the analysis result (text)
    getAnalysis() {
      if (this.result_analysis) {
        ElNotification({
          title: "Analysis Result",
          message: this.result_analysis,
          type: "success"
        })
      } else {
        ElNotification({
          title: "Failed",
          message: "还没有情感分析结果！",
          type: "error"
        })
      }
    },

    resultPlay() {
      if (this.result_audio) {
        let blob = new Blob([this.result_audio], { type: 'audio/wav' })
        console.log(blob)
        this.play(blob)

        // 将结果下载到本地
        // let fileName = 'result.wav'
        // let url = window.URL.createObjectURL(blob)
        // let link = document.createElement('a')
        // link.style.display = 'none'
        // link.href = url
        // link.setAttribute('download', fileName)
        // document.body.appendChild(link)
        // link.click()
        // window.URL.revokeObjectURL(link.href);
        // document.body.removeChild(link)
      } else {
        ElNotification({
          title: "Failed",
          message: "还没有变音结果！",
          type: "error"
        })
      }
    }
  },

  mounted() {
    this.$nextTick(() => {
      this.recOpen()
      const tickLengthMs = 5
      setInterval(() => {
        if (this.isPlaying) {
          this.playedLength += tickLengthMs
          this.playedPercentage = Math.round(this.playedLength / this.soundLength * 100)
          if (this.playedPercentage > 100) {
            this.playedPercentage = 0
            this.playedLength = 0
            this.isPlaying = false
          }
        }
      }, tickLengthMs)
    })
  },

  unmounted() {
    if (this.rec) {
      this.rec.close()
    }
  }
}
</script>

<template>
  <div class="page">
    <div class="container">
      <div class="items">
        <h1> sovits 音色转换器 </h1>
      </div>
      <div class="items">
        <h4> 《中文信息处理》 课程小组 </h4>
      </div>
      <div class="items">
        <el-space>
          <el-button @click="recStart" id="beginButton" :disabled="this.isRecording" type="primary"> 开始录音 </el-button>
          <el-button @click="recStop" :disabled="!this.isRecording" type="danger"> 结束录音 </el-button>
        </el-space>
      </div>
      <div class="items">
        <div style="height:200px; width:600px;" ref="recwave"></div>
      </div>
      <div class="items">
        <el-space>
          <el-button @click="recPlay" type="info"> 本地试听 </el-button>
          <el-button-group>
            <el-button @click="upload(analysisApi)" type="primary"> 情感分析 </el-button>
            <el-button @click="upload(sovitsApi)" type="primary"> 开始变声 </el-button>
          </el-button-group>
          <el-button @click="getAnalysis" type="success"> 情感分析结果 </el-button>
          <el-button @click="resultPlay" type="success"> 变声结果 </el-button>
        </el-space>
      </div>
      <div class="items">
        <el-text> {{ this.soundLength ? '录音时长：' + this.soundLength / 1000 + ' s' : '' }} </el-text>
      </div>
      <div class="items">
        <progress :value="this.playedPercentage" :max="100" />
      </div>
    </div>
  </div>
</template>

<style scoped>
.page {
  position: relative;
  height: 100vh;
  width: 100vw;
}

.container {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  text-align: center;
}

.items {
  padding-bottom: 35px;
}
</style>
