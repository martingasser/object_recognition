# Object recognition example

Built with tensorflow.js & Vue.js

# Create app

Open a terminal window.

```
vue create object_recognition
```

Select preset *Default ([Vue 2] babel, eslint)*

```
cd object_recognition
```

## Open a code editor

If you are using Visual Studio Code:

```
code .
```

## Run the app

```
npm run serve
```

And open a browser window: `http://localhost:8080/`

## Edit the source code

- Remove everything inside `<div id="app">...</div>`
- Remove the `HelloWorld` component

You can also delete 
- `src/components/HelloWorld.vue` 
- `src/assets/logo.png`

## Prepare a Github repository

Go back to the terminal window.

Exit the server with *Ctrl+C*.

Commit all changes:

```
git commit -am 'prepared barebone app'
```

Now go to Github and create a new repository (`object_recognition`).

After creating the repository, you should see a 'Quick setup' page.

In the second paragraph (`…or push an existing repository from the command line`) type the second command starting with `git remote add origin...` into your terminal window.

After doing that, type

```
git push -u origin main
```

Now your code should be on Github. Check!

## Add some HTML

Put this code into the section `<div id="app">...</div>`.

```html
<div id="app">
  <div id="center-container">
    <select id="camera-select">
      <option>
      </option>
    </select>
    <div id="result-frame">
      <video ref="video" autoplay></video>
      <canvas ref="canvas"></canvas>
    </div>
  </div>
</div>
```

### Explanation

The `video` element will be used to display the live video from the webcam, the `canvas` will be used to draw additional information on top of the video.

## Add a data section

The app needs to keep track of some information (current video device, video width and height, all available video devices), so let's add a `data` section:

```js
data () {
  return {
    videoDevice: '',
    resultWidth: 0,
    resultHeight: 0,
    devices: []
  }
},
```

## Add some functionality

We have to add functions for camera initialization.

Add a `methods` section to the app.

```js
methods: {

}
```

Put these three functions into the methods section:

```js
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

```

Finally, we have to initialize the video devices when the application is `mounted`.

Put this before the `methods` section:

```js
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
```

We also have to extend the HTML slightly:


```html
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
  </div>
</div>
```

## Add some styling

```css
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
```

Now you should be able to select a video camera (if you have multiple cameras attached to your computer)

# Add object recognition

## Install tensorflow.js and the required libraries

Go to the terminal, exit the web server, and type in those commands:

```
npm install @tensorflow-models/coco-ssd
npm install @tensorflow/tfjs
npm install @tensorflow/tfjs-converter
npm install @tensorflow/tfjs-core
```

Restart the web server

```
npm run serve
```

# Import tensorflow libraries into your app

At the top of the `script` section, add those imports:

```js
import * as tf from '@tensorflow/tfjs'
import * as cocoSsd from '@tensorflow-models/coco-ssd'
```

We have to set the Tensorflow backend. In the `mounted` lifecycle hook, add the line

```js
tf.setBackend('webgl')
```

at the top to use the webgl backend. By using this backend, we are using the graphics card of the computer, which makes the calculations much faster.

We have to add some additional data items to the data function:

```js
data () {
    return {
        videoDevice: '',
        resultWidth: 0,
        resultHeight: 0,
        devices: [],
        baseModel: 'mobilenet_v2',
        isModelReady: false
    }
},
```

Set the size of the canvas to the correct values (`resultWidth`, `resultHeight`).

```
...
<canvas ref="canvas" :width="resultWidth" :height="resultHeight"></canvas>
...
```

Then we need a few additional functions.

```js
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
    })
},
```

Finally, update the `mounted` lifecycle hook to load the machine learning model and start the inference loop.

Add the following code to the promise in the hook function:

```js
.then(() => {
  return this.loadModel()
  .then(() => {
    this.detectObjects()
  })
})
```

Now you should see detected objects drawn as overlays over the video image.
