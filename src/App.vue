<template>
  <div id="app" class="root">
    <h2>人像分割demo</h2>
    <!--图片展示-->
    <div class="content">
      <img id='image' width="200" src='./images/single.jpeg' crossorigin="anonymous"/>
      <!-- <img id='image' width="200" src='./images/double.jpeg' crossorigin="anonymous"/> -->
      <canvas id="canvas" width="500" height="500"></canvas>
      <!-- 拍照 -->
      <canvas ref="photoCanvas" width="320" height="240" style="margin-left: 20px"></canvas>
      <!--操作-->
      <div class="operation">
        <el-button type="primary" round @click="callCamera">开启摄像头</el-button>
        <el-button type="primary" round @click="photograph">拍照</el-button>
        <el-button round @click="closeCamera">关闭摄像头</el-button>
        <el-radio-group v-model="radio" style="margin-left: 40px;" @change="switchVideoMode">
          <el-radio :label="1">默认</el-radio>
          <el-radio :label="2">背景虚化</el-radio>
          <el-radio :label="3">替换背景</el-radio>
        </el-radio-group>
      </div>

      <!-- 视频区 -->
      <div class="container">
        <div class="item">
          <div class="title">
            原始视频
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
      <img id="maskImg" src="./images/bc.png" width="640" height="360" style="display: none;" />
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
      radio: 1,
      timer: null,   // 定时器
      videoCanvas: null,     // 处理后的视频帧的绘制区域

      segmentation: null,     // 分割类型
      flipHorizontal: false,    // 是否水平翻转
      backgroundBlurAmount: 3,  // 背景虚化程度 0~20
      edgeBlurAmount: 3    // 边缘模糊
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
        // 实时拍照效果
        this.$refs['video'].play()
      }).catch(error => {
        console.error('摄像头开启失败，请检查摄像头是否可用！')
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

 　　  // 上传拍照信息  调用接口上传图片 .........

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
      if(this.radio===1) {
        clearInterval(this.timer)
        this.clearCanvas(this.videoCanvas)
        return
      }
      if(this.radio===2) {
        // 虚化背景
        this.blurBackground()
      }
      // 替换背景

    },

    // 虚化背景
    blurBackground () {
      this.timer = setInterval(() => {
        // 把当前视频帧内容渲染到canvas上
        // let ctx = document.getElementById('newVideo');
        let ctx = this.$refs['newVideo'].getContext('2d')
        // ctx.drawImage(this.$refs['video'], 0, 0, 640, 480)

        const img = this.$refs['video']
        // const img = document.getElementById('image');

        bodyPix.drawBokehEffect(
        this.videoCanvas, img, this.segmentation, this.backgroundBlurAmount,
        this.edgeBlurAmount, this.flipHorizontal);
      }, 60)
    },

    async loadAndPredict() {
      const img = document.getElementById('image');
      
      // 加载模型
      const net = await bodyPix.load(
       { 
            architecture: 'MobileNetV1',
            outputStride: 16,
            multiplier: 0.5,
            quantBytes: 2
        }
      );
      this.segmentation = await net.segmentPerson(img);
      const coloredPartImage = bodyPix.toMask(this.segmentation);
      const opacity = 0.3;
   
      const canvas = document.getElementById('canvas');

      // 背景虚化
      bodyPix.drawBokehEffect(
      canvas, img, this.segmentation, this.backgroundBlurAmount,
      this.edgeBlurAmount, this.flipHorizontal);
    }
  }
}
   
</script>

<style lang="scss">
  .root {
    margin: 40px 50px;
  }
  .operation {
    display: flex;
    align-items: center;
    margin-top: 20px;
  }
  .container {
    .video{
      padding: 20px;
    }
    margin-top: 20px;
    display: flex;
  }
</style>
