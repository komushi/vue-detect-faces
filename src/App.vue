<template>
  <div id="app">
    <table>
      <tr>
        <td>
          <button v-on:click="openCloseCamera">start camera</button>
          <button v-on:click="openCloseDetector">start detecting</button>
          <div>
            <label>faceDetectionSize</label>
            <input type="range" step="1" min="4" max="20" v-model.number="faceDetectionSize">
            <input type="number" v-model.number="faceDetectionSize"/>
          </div>
          <div>
            <label>faceDetectionConfidence</label>
            <input type="range" step="0.05" min="0" max="0.95" v-model.number="faceDetectionConfidence">
            <input type="number" v-model.number="faceDetectionConfidence"/>
          </div>
          <div>
            <label>faceMatchDistance</label>
            <input type="range" step="0.05" min="0" max="0.95" v-model.number="faceMatchDistance">
            <input type="number" v-model.number="faceMatchDistance"/>
          </div>
        </td>
      </tr>
      <tr>
        <td>
          <vue-detect-faces 
            width="320"
            height="240"
            :invisible="invisible" 
            :openCamera="openCamera"
            :openDetector="openDetector"
            @faceDetected="onFaceDetected"
            :faceDetectionConfidence="faceDetectionConfidence"
            :faceMatchDistance="faceMatchDistance"
            :faceDetectionSize="faceDetectionSize">
            </vue-detect-faces>
        </td>
      </tr>
<!--       <tr>
        <td>
          <button v-on:click="stopStartTracking">start tracking</button>
        </td>
        <td>
          <vue-track-faces :start="startTracking" @faceDetected="onFaceDetected"></vue-track-faces>
        </td>
      </tr> -->
    </table>
    <div class="amplify-album">
      <div 
        class="amplify-album-container"
      >
        <div
          class="amplify-image-container"
          v-for="item in allFaces"
        >

          <img class="amplify-image" :src="item"/>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import VueTrackFaces from './components/VueTrackFaces'
import VueDetectFaces from './components/VueDetectFaces'

export default {
  name: 'App',
  data () {
    return {
      openCamera: false,
      openDetector: false,
      startDetecting: false,
      allFaces: [],
      invisible: false,
      faceDetectionConfidence: 0.65,
      faceMatchDistance: 0.65,
      faceDetectionSize: 8
    }
  },
  components: {
    VueDetectFaces,
    VueTrackFaces
  },
  methods: {
    stopStartTracking () {
      this.startTracking = !this.startTracking
    },
    openCloseCamera () {
      this.openCamera = !this.openCamera
    },
    openCloseDetector () {
      this.openDetector = !this.openDetector
    },
    onFaceDetected (val) {
      // console.log(val)
      this.allFaces.push(val)
    }
  }
}
</script>

