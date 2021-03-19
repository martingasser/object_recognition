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
      <ul>
        <li v-for="(prediction, index) in predictions" v-bind:key="index">
          {{prediction.name}}, {{prediction.score}} %
        </li>
      </ul>
    </div>
  </div>
</template>

<script>
import * as tf from '@tensorflow/tfjs'
import * as tmImage from '@teachablemachine/image'

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
      predictions: [],
      synth: window.speechSynthesis,
      lastPrediction: ''
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
      return tmImage.load('/model.json', '/metadata.json')
      .then(model => {
        this.model = model
        this.isModelReady = true
      })
      .catch(error => {
        console.log('failed to load the mode', error)
        throw(error)
      })
    },

    detectObjects () {
        if (!this.isModelReady) return

        if (this.isVideoStreamReady) {
          this.model.predict(this.$refs.video)
          .then(predictions => {
            this.handlePredictions(predictions)
            
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

    handlePredictions (predictions) {
        this.predictions.splice(0)
        
        let maxPrediction
        let maxProb = 0

        predictions.forEach(prediction => {
            this.predictions.push({
              name: prediction.className,
              score: (prediction.probability * 100).toFixed(1)
            })

            if (prediction.probability > maxProb) {
              maxProb = prediction.probability
              maxPrediction = prediction
            }
        })

        if (! this.synth.speaking && this.lastPrediction != maxPrediction.className) {
          this.speak(maxPrediction.className)
          this.lastPrediction = maxPrediction.className
        }
    },

    speak (prediction) {
      const utterThis = new SpeechSynthesisUtterance(prediction);
      this.synth.speak(utterThis)
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