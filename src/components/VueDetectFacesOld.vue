<template>
  <div>
    <table>
      <tr>
        <td><video id="myVideo" ref="video" :width="width" :height="height" :src="source" hidden></video></td>
      </tr>
      <tr>
        <td><canvas id="myCanvas" ref="canvas"></canvas></td>
      </tr>
    </table>
    
  </div>
</template>

<script>
export default {
  name: 'VueDetectFaces',
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
      default: '100%'
    },
    height: {
      type: [Number, String],
      default: 500
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
      let constraints = { video: true, audio: false, facingMode: this.facingMode }
      
      // if (this.resolution) {

        
        // constraints.video = {}
        // constraints.video.height = this.height
        // constraints.video.width = this.width
      // }

      this.stream = await navigator.mediaDevices.getUserMedia(constraints)

      if ('srcObject' in this.$refs.video) {
        // new browsers api
        this.$refs.video.srcObject = this.stream
      } else {
        // old broswers
        this.source = window.HTMLMediaElement.srcObject(this.stream)
      }

      await this.$refs.video.play()

      this.drawImage()

      // this.startTracker()
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
    drawImage () {
      
      this.$refs.canvas.width = this.$refs.video.videoWidth
      this.$refs.canvas.height = this.$refs.video.videoHeight

      const context2D = this.$refs.canvas.getContext('2d')
      context2D.drawImage(this.$refs.video, 0, 0)

      this.frameId = window.requestAnimationFrame(this.drawImage)

      this.startTracker()

    },
    setupTracker() {

      this.tracker = new tracking.ObjectTracker(['face'])
      
      let vm = this

      this.tracker.on('track', event => {

        // console.log(event)

        if (event.data.length === 0) {
          // No colors were detected in this frame.
        } else {

          event.data.filter(face => {
            console.log(face)
            // filter by face size
            // return !(face.width <= this.$refs.video.videoWidth / 5 || face.height <= this.$refs.video.videoHeight / 5)
            return true
          }).map(this.drawFaceBox)
        }

      })
    },
    startTracker() {

      const trackerTask = tracking.track('#myCanvas', this.tracker)
      // const trackerTask = tracking.track('#myVideo', this.tracker)
      

      // const width = this.$refs.canvas.width
      // const height = this.$refs.canvas.height
      // const context = this.$refs.canvas.getContext('2d')
      // const imageData = context.getImageData(0, 0, width, height)

      // this.tracker.track(imageData.data, width, height)

    },
    drawFaceBox(face, color) {

      const context2D = this.$refs.canvas.getContext('2d')

      console.log('face', face)

      const x = face.x
      const y = face.y
      const width = face.width
      const height = face.height

      this.captureFace(face)
      

      context2D.strokeStyle = color || 'yellow'
      context2D.strokeRect(x, y, width, height)
    },
    captureFace(face) {

      // let vm = this

      const context2D = this.$refs.canvas.getContext('2d')

      const imageData = context2D.getImageData(face.x, face.y, face.width, face.height)

      // console.log(imageData)

      const newCanvas = document.createElement("canvas")

      newCanvas.width = face.width;
      newCanvas.height = face.height;

      const newContext = newCanvas.getContext("2d")
      newContext.putImageData(imageData, 0, 0)

      const base64Img = newCanvas.toDataURL("image/jpeg")

      this.allFaces.push(base64Img)
      this.newFace = base64Img

      this.$emit('faceDetected', base64Img)
    }

  }

}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
h1, h2 {
  font-weight: normal;
}

ul {
  list-style-type: none;
  padding: 0;
}

li {
  display: inline-block;
  margin: 0 10px;
}

a {
  color: #42b983;
}
</style>
