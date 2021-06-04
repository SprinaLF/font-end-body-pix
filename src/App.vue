<template>
  <div id="app" class="root">
    <div class="content">
      <h2>demo</h2>
      <!-- 拍照 -->
      <!--操作-->
      <div class="operation">
        <el-button type="primary" round @click="callCamera">开启摄像头</el-button>
        <el-button round @click="closeCamera">关闭摄像头</el-button>
        <el-button type="primary" round @click="photograph">拍照</el-button>
        <el-button round @click="()=>clearCanvas(this.$refs['photoCanvas'])">删除照片</el-button>
      </div>

      <div class="operation">
        镜像：
        <el-switch
          v-model="isFlipHorizontal"
        ></el-switch>
        <el-radio-group v-model="radio" style="margin: 0px 40px;" @change="switchVideoMode">
          <el-radio :label="1">默认</el-radio>
          <el-radio :label="2">背景虚化</el-radio>
          <el-radio :label="3">背景替换</el-radio>
        </el-radio-group>
         <el-radio-group v-model="backgroundBlurAmount" style="margin: 0 20px;" v-if="radio===2">
          <el-radio :label="20">高</el-radio>
          <el-radio :label="10">中</el-radio>
          <el-radio :label="3">低</el-radio>
        </el-radio-group>
      </div>

      <!-- 背景图片区 -->
      <div style="display: flex; margin-top: 20px" v-if="radio===3">
        <el-radio-group
          v-model="bcRadio"
          style="display: flex; align-items: center"
          @change="switchBackground"
        >
          <template v-for="(item, i) in imgUrlArray">
            <el-radio-button :label="i" :key="item">
              <img class="bc-img" :ref="'img' + i" :src="item" />
            </el-radio-button>
          </template>
        </el-radio-group>
      </div>

      <!-- 视频区 -->
      <div class="container">
        <div>
          原始视频
          <div class="video">
            <video v-bind:class="{flipHorizontal: isFlipHorizontal}" ref="video" width="400" height="300" autoplay></video>
          </div>
        </div>
        <div>
          结果视频
          <div class="video">
            <canvas v-bind:class="{flipHorizontal: isFlipHorizontal}" id="videoCanvas" width="400" height="300"></canvas>
          </div>
        </div>
      </div>

      <!-- 照片 -->
      <canvas ref="photoCanvas" width="320" height="240" style="margin-left: 20px"></canvas>
    </div>
  </div>
</template>

<script>
import * as bodyPix from '@tensorflow-models/body-pix';
import * as tfjs from '@tensorflow/tfjs';
export default {
  name: 'App',
  data() {
    return {
      model: {
        architecture: 'MobileNetV1',
        outputStride: 16,
        multiplier: 0.75,
        quantBytes: 2
      },
      isFlipHorizontal: false,   // 是否水平翻转(镜像)

      radio: 1,
      bcRadio: 0,
      videoCanvas: null,     // 处理后的视频帧的绘制区域
      net: null,
      backgroundImg: null,

      flipHorizontal: false,    // 是否水平翻转
      backgroundBlurAmount: 10,  // 背景虚化程度 0~20
      edgeBlurAmount: 3,    // 边缘模糊

      imgUrlArray: ['https://static.yximgs.com/udata/pkg/y-tech-open-platform/common/2ed5edc1a56e353bf8dabdabb640480e.jpeg',
    'https://static.yximgs.com/udata/pkg/y-tech-open-platform/common/d842f4673127224d42a84b9a9684f07f.jpeg',
    'https://static.yximgs.com/udata/pkg/y-tech-open-platform/common/dea36ed8b7756727351a94f197b0daad.jpeg',
    'https://static.yximgs.com/udata/pkg/y-tech-open-platform/common/ed83312458fa92498baacff647537f90.jpeg']
    }
  },
  mounted() {
    // 加载模型
    this.loadAndPredict(this.model)
    this.videoCanvas = document.getElementById('videoCanvas');

    // 加载背景图
    this.backgroundImg = new Image()
    this.backgroundImg.src = this.imgUrlArray[this.bcRadio]
    this.backgroundImg.setAttribute('crossOrigin', 'Anonymous')
  },
  methods: {
     // 开启摄像头
     callCamera () {
      return new Promise(async (resolve) => {
        const success = await navigator.mediaDevices.getUserMedia({
          video: true
        })
        this.isOpen = true
        this.$nextTick(() => {
          this.$refs['video'].onloadeddata = () => {
            resolve()
            this.switchVideoMode()   // 判断处于哪种模式
          }
          // 摄像头开启成功
          this.$refs['video'].srcObject = success
        })
      })
     },
    // 拍照
    photograph () {
      let ctx = this.$refs['photoCanvas'].getContext('2d')
      ctx.drawImage(this.$refs['video'], 0, 0, 320, 240)
      // 转base64格式、图片格式转换、图片质量压缩
      let imgBase64 = this.$refs['photoCanvas'].toDataURL('image/jpeg', 0.7)
　　　 // 由字节转换为KB 判断大小
      let str = imgBase64.replace('data:image/jpeg;base64,', '')
      let strLength = str.length
      let fileLength = parseInt(strLength - (strLength / 8) * 2)
　　　 // 图片尺寸  用于判断
      let size = (fileLength / 1024).toFixed(2)
      // 保存到本地
      this.dialogCamera = false
      let ADOM = document.createElement('a')
      ADOM.href = this.headImgSrc
      ADOM.download = new Date().getTime() + '.jpeg'
      ADOM.click()
    },

    // 清空canvas
    clearCanvas(obj) {
      obj.getContext("2d").clearRect(0,0,obj.width,obj.height)
    },
    // 关闭摄像头
    closeCamera () {
      if (!this.$refs['video'].srcObject) {
        this.dialogCamera = false
        return
      }
      let stream = this.$refs['video'].srcObject
      let tracks = stream.getTracks()
      tracks.forEach(track => {
        track.stop()
      })
      this.$refs['video'].srcObject = null
      this.clearCanvas(this.videoCanvas)
    },

    // 切换模式
    switchVideoMode () {
      switch(this.radio) {
        case 1: 
          this.clearCanvas(this.videoCanvas)
          break
        case 2: 
          this.blurBackground()  // 虚化背景
          break
        case 3:
          this.replaceBackground()    // 背景替换
      }
    },

    // 切换背景图
    switchBackground() {
      this.backgroundImg.src = this.imgUrlArray[this.bcRadio]
    },

    // 2虚化背景
    async blurBackground () {
      if(!this.isOpen) return
      const img = this.$refs['video']   // 获取视频帧
      const segmentation = await this.net.segmentPerson(img);
      bodyPix.drawBokehEffect(
        this.videoCanvas, img, segmentation, this.backgroundBlurAmount,
        this.edgeBlurAmount, this.flipHorizontal);
      if(this.radio===2) {
        requestAnimationFrame(
          this.blurBackground
        )
      } else {
          this.clearCanvas(this.videoCanvas)
      }
      
      // this.timer = setInterval(async() => {
      //   this.segmentation = await this.net.segmentPerson(img);
      //   bodyPix.drawBokehEffect(
      //     this.videoCanvas, img, this.segmentation, 3,
      //     this.edgeBlurAmount, this.flipHorizontal);
      // }, 60)
    },

    // 3背景替换
    async replaceBackground() {
      if(!this.isOpen) return
      const img = this.$refs['video']
      const segmentation = await this.net.segmentPerson(img);

      const foregroundColor = { r: 0, g: 0, b: 0, a: 0 }
      const backgroundColor = { r: 0, g: 0, b: 0, a: 255 }
      let backgroundDarkeningMask = bodyPix.toMask(
        segmentation,
        foregroundColor,
        backgroundColor
      )
      if (backgroundDarkeningMask) {
        let context = this.videoCanvas.getContext('2d')
        // 合成
        context.putImageData(backgroundDarkeningMask, 0, 0)
        context.globalCompositeOperation = 'source-in' // 新图形只在重合区域绘制
        context.drawImage(this.backgroundImg, 0, 0, this.videoCanvas.width, this.videoCanvas.height)
        context.globalCompositeOperation = 'destination-over' // 新图形在画布内容后面绘制
        context.drawImage(img, 0, 0, this.videoCanvas.width, this.videoCanvas.height)
        context.globalCompositeOperation = 'source-over' // 恢复
      }
      if(this.radio===3) {
        requestAnimationFrame(
          this.replaceBackground
        )
      } else {
        this.clearCanvas(this.videoCanvas)
      }
    },

    async loadAndPredict(model) {
      const img = document.getElementById('image');
      // 加载模型
      this.net = await bodyPix.load(model);
    }
  }
}
</script>

<style>
  .root {
    margin: 40px 50px;
  }
  .operation {
    display: flex;
    align-items: center;
    margin-top: 20px;
  }
  .container {
    margin-top: 20px;
    display: flex;
  }
  .video{
    padding: 20px;
  }
  .bc-img {
    width: 144px;
    height: 108px;
  }
  .flipHorizontal {
    transform: rotateY(180deg);
  }
</style>
