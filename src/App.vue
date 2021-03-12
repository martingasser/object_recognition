<template>
  <div id="app">
    <div id="center-container">
      <select id="camera-select" v-model="videoDevice" @change="initWebcamStream()">
        <option v-for="device in devices" v-bind:key="device.deviceId" v-bind:value="device.deviceId">
          {{ device.label }}
        </option>
      </select>
      <div id="result-frame">
        <video ref="video" autoplay></video>
        <canvas ref="canvas" :width="resultWidth" :height="resultHeight"></canvas>
      </div>
      <div id="result-table">
        <div v-for="(prediction, index) in predictions" v-bind:key="index">
          {{prediction.name}}, {{prediction.score}} %
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import * as tf from '@tensorflow/tfjs'
import * as cocoSsd from '@tensorflow-models/coco-ssd'

export default {
  name: 'App',
  components: {
  },
  data () {
    return {
      videoDevice: '',
      resultWidth: 0,
      resultHeight: 0,
      devices: [],
      baseModel: 'mobilenet_v2',
      isModelReady: false,
      predictions: []
    }
  },
  mounted () {
    tf.setBackend('webgl')
    
    this.listVideoDevices()
    .then(videoDevices => {
      for (let device of videoDevices) {
        this.devices.push(device)
      }
      this.videoDevice = videoDevices[0].deviceId
    })
    .then(() => {
      return this.initWebcamStream()
    })
    .then(() => {
      return this.loadModel()
      .then(() => {
        this.detectObjects()
      })
    })    
  },
  methods: {
    listVideoDevices () {
      return navigator.mediaDevices.enumerateDevices()
        .then(devices => {
          return devices.filter(device => device.kind === 'videoinput')
      })
    },
    initWebcamStream () {
      this.isVideoStreamReady = false
      // if the browser supports mediaDevices.getUserMedia API
      if (navigator.mediaDevices && navigator.mediaDevices.getUserMedia) {
        return navigator.mediaDevices.getUserMedia({
          video: { deviceId: this.videoDevice }
        })
        .then(stream => {
          // set <video> source as the webcam input
          let video = this.$refs.video
          video.srcObject = stream
          return new Promise((resolve) => {
            // when video is loaded
            video.onloadedmetadata = () => {
              // calculate the video ratio
              this.videoRatio = video.videoHeight / video.videoWidth
              // add event listener on resize to reset the <video> and <canvas> sizes
              window.addEventListener('resize', this.setResultSize)
              // set the initial size
              this.setResultSize()
              this.isVideoStreamReady = true
              resolve()
            }
          })
        })
        // error handling
        .catch(error => {
          console.log('failed to initialize webcam stream', error)
        })
      }
    },
    
    setResultSize () {
      let clientWidth = document.documentElement.clientWidth
      this.resultWidth = Math.min(600, clientWidth)
      this.resultHeight = this.resultWidth * this.videoRatio
      let video = this.$refs.video
      video.width = this.resultWidth
      video.height = this.resultHeight
    },

    loadModel () {
        return cocoSsd.load(this.baseModel)
        .then(model => {
            this.model = model
            this.isModelReady = true
            console.log('model loaded')
        })
        .catch((error) => {
            console.log('failed to load the model', error)
            throw (error)
        })
    },

    detectObjects () {
        if (!this.isModelReady) return

        if (this.isVideoStreamReady) {
        this.model.detect(this.$refs.video)
            .then(predictions => {
                this.renderPredictions(predictions)
                requestAnimationFrame(() => {
                this.detectObjects()
                })
            })
        } else {
        requestAnimationFrame(() => {
            this.detectObjects()
        })
        }
    },

    renderPredictions (predictions) {
        this.predictions.splice(0)

        const ctx = this.$refs.canvas.getContext('2d')
        ctx.clearRect(0, 0, ctx.canvas.width, ctx.canvas.height)
        predictions.forEach(prediction => {
            ctx.beginPath()
            ctx.rect(...prediction.bbox)
            ctx.lineWidth = 3
            ctx.strokeStyle = 'red'
            ctx.fillStyle = 'red'
            ctx.stroke()
            ctx.shadowColor = 'white'
            ctx.shadowBlur = 10
            ctx.font = '24px Arial bold'
            ctx.fillText(
                `${(prediction.score * 100).toFixed(1)}% ${prediction.class}`,
                prediction.bbox[0],
                prediction.bbox[1] > 10 ? prediction.bbox[1] - 5 : 10
            )

            this.predictions.push({
              name: prediction.class,
              score: (prediction.score * 100).toFixed(1)
            })
        })
    }    
  }
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  color: #2c3e50;
  margin-top: 60px;
}

#result-frame {
  height: 500px;
}

video {
  position: absolute;
}

canvas {
  position: absolute;
}

#center-container {
  width: 600px;
  margin: 0 auto;
}

#camera-select {
  width: 300px;
  margin-bottom: 50px;
}
</style>