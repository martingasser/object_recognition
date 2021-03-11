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
        <canvas ref="canvas"></canvas>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'App',
  components: {
  },
  data () {
    return {
      videoDevice: '',
      resultWidth: 0,
      resultHeight: 0,
      devices: []
    }
  },
  mounted () {
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