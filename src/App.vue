<template>
  <div id="app">
    <table>
      <tr>
        <td>
          <button v-on:click="stopStartTracking">start tracking</button>
        </td>
        <td>
          <vue-track-faces :start="startTracking" @faceDetected="onFaceDetected"></vue-track-faces>
        </td>
      </tr>
      <tr>
        <td>
          <button v-on:click="stopStartDetecting">start detecting</button>
        </td>
        <td>
          <vue-detect-faces :invisible="invisible" :start="startDetecting" @faceDetected="onFaceDetected"></vue-detect-faces>
        </td>
      </tr>
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
      startTracking: false,
      startDetecting: false,
      allFaces: [],
      invisible: false
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
    stopStartDetecting () {
      this.startDetecting = !this.startDetecting
    },
    onFaceDetected (val) {
      // console.log(val)
      // this.allFaces.push(val)
    }
  }
}
</script>

