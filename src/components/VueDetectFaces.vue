<template>
  <div>
    <div style="position: relative">
      <video  
        id="inputVideo" 
        ref="video" 
        :width="width" 
        :height="height" 
        :src="source"
        :hidden="invisible"
        autoplay
        playsinline
        >
      </video>
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
      faceDescriptions: [],
      faceMatcher: null
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
    openCamera: {
      type: Boolean,
      default: true
    },
    openDetector: {
      type: Boolean,
      default: true
    },
    facingMode: {
      type: String,
      default: 'user'
    },
    invisible: {
      type: Boolean,
      default: false      
    },
    faceDetectionConfidence: {
      type: Number,
      default: 0.65
    },
    faceMatchDistance: {
      type: Number,
      default: 0.65      
    },
    faceDetectionSize: {
      type: Number,
      default: 8    
    },
    
  },
  mounted () {
    this.setupCamera()

    this.setupDetector()
  },
  watch: {
    openCamera: {
      handler: async function (newValue, oldValue) {
        console.log(newValue)
        if (newValue) {
          await this.startCamera()
        } else {
          this.stopCamera()
        }
      }
    },
    openDetector: {
      handler: async function (newValue, oldValue) {
        console.log(newValue)
        if (newValue) {
          await this.startDetector()
        }
      }
    },
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

      // await faceapi.nets.ssdMobilenetv1.load('/static/models')
      await faceapi.nets.tinyFaceDetector.load('/static/models')
      await faceapi.nets.faceLandmark68Net.load('/static/models')
      await faceapi.nets.faceRecognitionNet.load('/static/models')
    },
    async startDetector() {

      // const result = await faceapi
      //   .detectAllFaces(this.$refs.video)
      //   .withFaceLandmarks()
      //   .withFaceDescriptors()

      if (!this.openDetector) {
        return
      }

      const result = await faceapi
        .detectAllFaces(this.$refs.video, new faceapi.TinyFaceDetectorOptions({
          inputSize: this.faceDetectionSize * 32,
          scoreThreshold: this.faceDetectionConfidence
        }))
        .withFaceLandmarks()
        .withFaceDescriptors()


      if (result.length === 0) {
        // console.log(`${result.length} faces tracked`)
        this.clearFaceBox()
        
      } else {

        if (result.length > 1) {
          console.log(`${result.length} faces tracked`)
          // console.log('result', JSON.stringify(result))
        }

        if (this.faceDescriptions.length === 0) {
          this.faceMatcher = new faceapi.FaceMatcher(result)
        } else {
          this.faceMatcher = new faceapi.FaceMatcher(this.faceDescriptions)
        }

        const dims = faceapi.matchDimensions(this.$refs.canvas, this.$refs.video, true)

        const resized = faceapi.resizeResults(result, dims)

        resized
          .map(this.matchFace)
          .map(this.drawFaceBox)
          .map(this.captureFace)
          
      }      
    
      // setTimeout(async () => {
      //   await this.startDetector()
      // })

      await new Promise(resolve => setTimeout(resolve, 0))

      await this.startDetector()

    },
    matchFace(faceDescription) {

      const findMatch = this.faceMatcher.findBestMatch(faceDescription.descriptor)
      const label = findMatch.toString()

      if (this.faceDescriptions.length === 0) {
        this.faceDescriptions.push(faceDescription)

        return {faceDescription, foundNew: true}
      } else {
        if (findMatch.distance > this.faceMatchDistance) {
          this.faceDescriptions.push(faceDescription)
          console.log('findMatch new face', findMatch)
          console.log(this.faceDescriptions.length)
          return {faceDescription, foundNew: true}
        } else {
          // console.log('findMatch', findMatch)
        }
      }


      return {faceDescription, foundNew: false}

    },
    clearFaceBox() {
      const context2D = this.$refs.canvas.getContext('2d')

      context2D.clearRect(0, 0, this.$refs.canvas.width, this.$refs.canvas.height)

    },
    drawFaceBox({faceDescription, foundNew}) {
      if (!this.invisible) {
        faceapi.draw.drawDetections(this.$refs.canvas, faceDescription)
      }

      return {faceDescription, foundNew}
    },
    captureFace({faceDescription, foundNew}) {
      if (!foundNew) {
        return
      }

      const newCanvas = document.createElement("canvas")

      newCanvas.width = faceDescription.detection.box.width
      newCanvas.height = faceDescription.detection.box.height

      const context2D = newCanvas.getContext('2d')

      context2D.drawImage(this.$refs.video, 
        faceDescription.detection.box.x, 
        faceDescription.detection.box.y, 
        faceDescription.detection.box.width, 
        faceDescription.detection.box.height,
        0,0,faceDescription.detection.box.width,faceDescription.detection.box.height)

      const base64Img = newCanvas.toDataURL("image/jpeg")

      // this.allFaces.push(base64Img)
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
