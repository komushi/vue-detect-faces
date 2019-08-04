<template>
  <div>

    <div style="position: relative">
      <video id="myVideo" ref="video" :width="width" :height="height" :src="source"></video>
      <canvas id="overlay" ref="canvas" :width="width" :height="height" />
    </div>    
  </div>
</template>

<script>
export default {
  name: 'VueTrackFaces',
  data () {
    return {
      source: null,
      stream: null,
      frameId: null,
      tracker: null,
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
    }
  },
  mounted () {
    this.setupCamera()

    this.setupTracker()
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

      this.startTracker()
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

      window.cancelAnimationFrame(this.frameId)
    },
    setupTracker() {

      this.tracker = new tracking.ObjectTracker('face')

      this.tracker.setInitialScale(4)
      this.tracker.setStepSize(2)
      this.tracker.setEdgesDensity(0.1)

      let vm = this

      this.tracker.on('track', event => {

        // console.log(event)
        this.clearFaceBox()

        if (event.data.length === 0) {
          // No colors were detected in this frame.
        } else {

          if (event.data.length > 1) {
            console.log(`${event.data.length} faces tracked`)
          }

          event.data.filter(face => {
            // console.log(face)
            // filter by face size
            // return !(face.width <= this.$refs.video.videoWidth / 5 || face.height <= this.$refs.video.videoHeight / 5)
            return true
          }).map(this.drawFaceBox)
        }

      })
    },
    startTracker() {
      const trackerTask = tracking.track('#myVideo', this.tracker)
      
    },
    clearFaceBox() {
      const context2D = this.$refs.canvas.getContext('2d')

      context2D.clearRect(0, 0, this.$refs.canvas.width, this.$refs.canvas.height)

    },
    drawFaceBox(face, color) {

      // this.$refs.canvas.width = this.$refs.video.videoWidth
      // this.$refs.canvas.height = this.$refs.video.videoHeight

      const context2D = this.$refs.canvas.getContext('2d')

      // console.log('drawFaceBox face', face)

      const x = face.x
      const y = face.y
      const width = face.width
      const height = face.height

      context2D.strokeStyle = color || 'yellow'
      context2D.strokeRect(x, y, width, height)
      context2D.lineWidth = 3

      this.captureFace(face)
    },
    captureFace(face) {

      const newCanvas = document.createElement("canvas")

      newCanvas.width = face.width;
      newCanvas.height = face.height;

      const context2D = newCanvas.getContext('2d')

      context2D.drawImage(this.$refs.video, 
        face.x, 
        face.y, 
        face.width, 
        face.height,
        0,0,face.width,face.height)

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
