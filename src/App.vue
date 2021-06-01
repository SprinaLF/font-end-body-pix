<template>
  <div id="app" class="root">
    <div class="content">
      <h2>人像分割demo</h2>
      <!--图片展示-->
      <img id='image' width="200" src='./images/single.jpeg' crossorigin="anonymous"/>
      <canvas id="picCanvas" width="500" height="500"></canvas>
      <!-- 拍照 -->
      <canvas ref="photoCanvas" width="320" height="240" style="margin-left: 20px"></canvas>
      <!--操作-->
      <div class="operation">
        <el-button type="primary" round @click="callCamera">开启摄像头</el-button>
        <el-button round @click="closeCamera">关闭摄像头</el-button>
        <el-radio-group v-model="radio" style="margin: 0px 40px;" @change="switchVideoMode">
          <el-radio :label="1">默认</el-radio>
          <el-radio :label="2">背景虚化</el-radio>
          <el-radio :label="3">背景替换</el-radio>
        </el-radio-group>
         <el-button type="primary" round @click="photograph">拍照</el-button>
        <el-button round @click="()=>clearCanvas(this.$refs['photoCanvas'])">删除照片</el-button>
      </div>

      <!-- 视频区 -->
      <div class="container">
        <div class="item">
          <div class="title">
            原视频
          </div>
          <div class="video">
            <video ref="video" width="640" height="480" autoplay></video>
          </div>
        </div>
        <div class="item">
          <div class="title">
            处理后视频
          </div>
          <div class="video">
            <canvas id="videoCanvas" ref="newVideo" width="640" height="480"></canvas>
          </div>
        </div>
      </div>
      <img id="maskImg" ref="maskBc" src="./images/bc.png" width="640" height="360" style="display: none;" />
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
      isOpen: false,    // 是否开启摄像头
      radio: 1,
      timer: null,   // 定时器
      videoCanvas: null,     // 处理后的视频帧的绘制区域
      net: null,

      opacity: 0.3,
      flipHorizontal: false,    // 是否水平翻转
      backgroundBlurAmount: 3,  // 背景虚化程度 0~20
      edgeBlurAmount: 3,    // 边缘模糊

      maskBlurAmount: 3    // this will darken the background and blur the darkened background's edge.
    }
  },
  mounted() {
    const img = document.getElementById('image');
    img.setAttribute('crossOrigin', 'anonymous');
    this.videoCanvas = document.getElementById('videoCanvas');
    this.loadAndPredict()
  },
  methods: {
     // 调用摄像头
     callCamera () {
      // H5调用电脑摄像头API
      navigator.mediaDevices.getUserMedia({
        video: true
      }).then(success => {
        // 摄像头开启成功
        this.$refs['video'].srcObject = success
        this.isOpen = true,
        // 实时拍照效果
        this.$refs['video'].play()
        // 判断处于哪种模式
        this.switchVideoMode()
      }).catch(error => {
        console.error(error||'摄像头开启失败')
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
      console.log(size)

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
      clearInterval(this.timer)
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

    // 2虚化背景
    async blurBackground () {
      const img = this.$refs['video']
      const segmentation = await this.net.segmentPerson(img);
      bodyPix.drawBokehEffect(
        this.videoCanvas, img, segmentation, 3,
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
        context.drawImage(this.$refs['maskBc'], 0, 0, 640, 480)
        context.globalCompositeOperation = 'destination-over' // 新图形只在不重合的区域绘制
        context.drawImage(img, 0, 0)
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

    async loadAndPredict() {
      const img = document.getElementById('image');
      // 加载模型
       this.net = await bodyPix.load(
       { 
            architecture: 'MobileNetV1',
            outputStride: 16,
            multiplier: 0.75,
            quantBytes: 2
        }
      );
   
      const canvas = document.getElementById('picCanvas');
      // 背景虚化
      bodyPix.drawBokehEffect(
      canvas, img, this.segmentation, 3,
      this.edgeBlurAmount, this.flipHorizontal);
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
</style>
