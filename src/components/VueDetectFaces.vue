<template>
  <div>
    <div style="position: relative">
      <video :hidden="invisible" id="inputVideo" ref="video" :width="width" :height="height" :src="source"></video>
      <canvas id="overlay" ref="canvas" />
    </div>
  </div>
</template>

<script>
import * as faceapi from 'face-api.js'

export default {
  name: 'VueDetectFaces',
  data () {
    return {
      source: null,
      stream: null,
      allFaces: [],
      newFace: null,

    }
  },
  props: {
    width: {
      type: [Number, String],
      default: 640
    },
    height: {
      type: [Number, String],
      default: 480
    },
    start: {
      type: Boolean,
      default: true
    },
    facingMode: {
      type: String,
      default: 'user'
    },
    invisible: {
      type: Boolean,
      default: true      
    }
  },
  mounted () {
    this.setupCamera()
  },
  watch: {
    start: {
      handler: function (newValue, oldValue) {
        console.log(newValue)
        if (newValue) {
          this.startCamera()
        } else {
          this.stopCamera()
        }
      }
    }
  },
  methods: {

    setupCamera () {
      if (navigator.mediaDevices === undefined) {
        navigator.mediaDevices = {}
      }

      if (navigator.mediaDevices.getUserMedia === undefined) {
        navigator.mediaDevices.getUserMedia = this.legacyGetUserMediaSupport()
      }
    },
    legacyGetUserMediaSupport () {
      return constraints => {
        // First get ahold of the legacy getUserMedia, if present
        let getUserMedia =
          navigator.getUserMedia ||
          navigator.webkitGetUserMedia ||
          navigator.mozGetUserMedia ||
          navigator.msGetUserMedia ||
          navigator.oGetUserMedia

        // Some browsers just don't implement it - return a rejected promise with an error
        // to keep a consistent interface
        if (!getUserMedia) {
          return Promise.reject(
            new Error('getUserMedia is not implemented in this browser')
          )
        }
        // Otherwise, wrap the call to the old navigator.getUserMedia with a Promise
        return new Promise((resolve, reject) => {
          getUserMedia.call(navigator, constraints, resolve, reject)
        })
      }
    },
    async startCamera () {
      let constraints = { 
        video: {
          width: { ideal: this.width },
          height: { ideal: this.height },
          facingMode: this.facingMode
        }, 
        audio: false
      }
      
      this.stream = await navigator.mediaDevices.getUserMedia(constraints)

      if ('srcObject' in this.$refs.video) {
        // new browsers api
        this.$refs.video.srcObject = this.stream
      } else {
        // old broswers
        this.source = window.HTMLMediaElement.srcObject(this.stream)
      }

      await this.$refs.video.play()

      await this.setupDetector()

      await this.startDetector()
    },
    stopCamera () {
      if (this.$refs.video) {
        this.$refs.video.pause()
        this.$refs.video.srcObject = null
      }

      if (this.stream) {
        this.stream.getVideoTracks()[0].stop()
        this.stream = null
      }

      this.source = null
      this.stream = null

    },
    async setupDetector() {

      console.log(faceapi.nets)

      await faceapi.nets.tinyFaceDetector.load('/static/models')
    },
    async startDetector() {

      const result = await faceapi.detectAllFaces(this.$refs.video, new faceapi.TinyFaceDetectorOptions({
        inputSize: 128,
        scoreThreshold: 0.5
      }))

      // console.log('result', result)

      if (result.length === 0) {
        // console.log(`${result.length} faces tracked`)
        this.clearFaceBox()
        
      } else {

        if (result.length > 1) {
          console.log(`${result.length} faces tracked`)
        }

        result.filter(face => {
          // console.log(face)
          // filter by face size
          // return !(face.width <= this.$refs.video.videoWidth / 5 || face.height <= this.$refs.video.videoHeight / 5)
          return true
        }).map(this.drawFaceBox)
      }      
    
      setTimeout(() => {
        this.startDetector()
      })

    },
    clearFaceBox() {
      const context2D = this.$refs.canvas.getContext('2d')

      context2D.clearRect(0, 0, this.$refs.canvas.width, this.$refs.canvas.height)

    },
    drawFaceBox(result) {

      const dims = faceapi.matchDimensions(this.$refs.canvas, this.$refs.video, true)

      const resized = faceapi.resizeResults(result, dims)

      if (!this.invisible) {
        faceapi.draw.drawDetections(this.$refs.canvas, resized)
      }

      this.captureFace(resized)
    },
    captureFace(result) {


      const newCanvas = document.createElement("canvas")

      newCanvas.width = result.box.width
      newCanvas.height = result.box.height

      const context2D = newCanvas.getContext('2d')

      context2D.drawImage(this.$refs.video, 
        result.box.x, 
        result.box.y, 
        result.box.width, 
        result.box.height,
        0,0,result.box.width,result.box.height)

      const base64Img = newCanvas.toDataURL("image/jpeg")

      this.allFaces.push(base64Img)
      this.newFace = base64Img

      this.$emit('faceDetected', base64Img)
    }

  }

}
</script>

<style scoped>

#overlay, .overlay {
  position: absolute;
  top: 0;
  left: 0;
}

</style>
