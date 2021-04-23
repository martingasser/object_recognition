<template>
  <div class="app">
    <div class="center-container">
      <select class="margin-10 device-select" v-model="audioDevice">
        <option v-for="device, id in audioDevices" :key="id" :value="device.deviceId">
          {{ device.label }}
        </option>
      </select>
      <!-- <input class="margin-10 text-input" type="text" v-model="username" placeholder="Enter username"> -->
      <table class="margin-10">
        <tr class="top-row" v-if="topPrediction != null">
          <td>{{ topPrediction.name }}</td>
          <td>{{ topPrediction.score }} %</td>
        </tr>
        <tr v-else>
          <td>Not sure...</td>
        </tr>
        <tr class="other-rows" v-for="(prediction, index) in filteredSortedPredictions" v-bind:key="index">
          <td>{{prediction.name}}</td>
          <td>{{prediction.score}} %</td>
        </tr>
      </table>
    </div>
  </div>
</template>

<script>
import * as tf from '@tensorflow/tfjs'
import * as speechCommands from '@tensorflow-models/speech-commands'

export default {
  name: 'App',
  components: {
  },
  data () {
    return {
      isModelReady: false,
      predictions: [],
      lastPrediction: null,
      audioTrackConstraints: {},
      audioDevices: [],
      audioDevice: '',
      // username: ''
    }
  },
  watch: {
    audioDevice: function (deviceId) {
      if (this.recognizer) {
        this.setupRecognizer(deviceId)
      }
    }
  },
  computed: {
    sortedPredictions () {
      const predictions = [...this.predictions]
      predictions.sort((a, b) => {
        return (b.score - a.score)
      })
      return predictions
    },
    topPrediction () {
      if (this.sortedPredictions.length > 0) {
        return this.sortedPredictions[0]
      }
      else {
        return null
      }
    },
    filteredSortedPredictions () {
      const t = this.topPrediction
      return this.sortedPredictions.filter(p => p != t)
    }
  },
  async mounted () {
    tf.setBackend('webgl')

    const enumeratorPromise = navigator.mediaDevices.enumerateDevices()

    enumeratorPromise.then(async devices => {
      devices.forEach(device => {
        if (device.kind === 'audioinput') {
          this.audioDevices.push(device)
        }
      })
      this.audioDevice = this.audioDevices[0].deviceId

      this.recognizer = await this.createModel()
      this.classLabels = this.recognizer.wordLabels() // get class labels
      
      this.setupRecognizer(this.audioDevice)
    })


    // this.webSocket = new WebSocket('ws://localhost:8081')
    this.webSocket = new WebSocket('wss://5ad55184c310.ngrok.io')

  },
  methods: {
    async setupRecognizer (deviceId) {
      if (this.recognizer.isListening()) {
        await this.recognizer.stopListening()
      }
      // listen() takes two arguments:
      // 1. A callback function that is invoked anytime a word is recognized.
      // 2. A configuration object with adjustable fields
      await this.recognizer.listen(result => {
        this.handlePredictions(result.scores)
      }, {
          includeSpectrogram: false, // in case listen should return result.spectrogram
          probabilityThreshold: 0.75,
          invokeCallbackOnNoiseAndUnknown: true,
          overlapFactor: 0.5, // probably want between 0.5 and 0.75. More info in README
          audioTrackConstraints: {
            deviceId: deviceId
          }
      })
    },
    async createModel() {
        // const recognizer = speechCommands.create('BROWSER_FFT', 'directional4w')
        // (1) try replacing the '18w' function argument with 'directional4w'.
        // - What happens? Why?
        // - check https://github.com/tensorflow/tfjs-models/tree/master/speech-commands

        // (2) replace the model trained on Speech Commands with a model trained on your own data
        const checkpointURL = `${location.protocol}//${location.host}/model.json` // model topology
        const metadataURL = `${location.protocol}//${location.host}/metadata.json` // model metadata

        const recognizer = speechCommands.create(
            'BROWSER_FFT', // fourier transform type, not useful to change
            undefined, // speech commands vocabulary feature, not useful for your models
            checkpointURL,
            metadataURL)

        // check that model and metadata are loaded via HTTPS requests.
        await recognizer.ensureModelLoaded()
        return recognizer
    },

    handlePredictions (predictions) {
        this.predictions.length = 0
        
        predictions.forEach((prediction, index) => {
            this.predictions.push({
              name: this.classLabels[index],
              score: (prediction * 100).toFixed(1)
            })
        })

        if (this.lastPrediction == null) {
          this.lastPrediction = this.topPrediction
        }
        else if (this.lastPrediction.name != this.topPrediction.name) {
          const message = {
            detectionType: 'speech',
            // username: this.username,
            className: this.topPrediction.name,
            score: this.topPrediction.score
          }

          this.webSocket.send(JSON.stringify(message))
          this.lastPrediction = this.topPrediction
        }
    }
  }
}
</script>

<style>
.app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  color: #2c3e50;
  margin-top: 60px;
}

table {
  width: 100%;
  border-spacing: 5px;
}

.margin-10 {
  margin: 10px;
}

.top-row {
  background-color: rgba(0, 200, 0, 0.1);
  height: 40px;
  font-weight: 900;
}

.other-rows {
  background-color: rgba(200, 0, 0, 0.1);
  height: 30px;
  font-weight: 400;
}

td {
  padding: 5px;
}

td:nth-child(1) {
  width: 70%;
    text-align: left;
}

td:nth-child(2) {
  width: 30%;
  text-align: center;
}

.center-container {
  width: 600px;
  margin: 0 auto;
}

.device-select {
  width: 100%;
}

.text-input {
  width: 50%;
}
</style>