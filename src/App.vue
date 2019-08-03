<template>
  <div id="app">
    <table>
      <tr>
        <td>
          <button v-on:click="stopStart">start</button>
        </td>
        <td>
          <vue-detect-faces :invisible="invisible" :start="startCamera" @faceDetected="onFaceDetected"></vue-detect-faces>
        </td>
      </tr>
    </table>
<!--     <div>
      <div
        v-for="item in allFaces"
      >
        <img
          :src="item"
        />
      </div>

    </div> -->
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
// import VueDetectFaces from './components/VueDetectFaces'
import VueDetectFaces from './components/VueDetectFaces'

export default {
  name: 'App',
  data () {
    return {
      startCamera: false,
      allFaces: [],
      invisible: true
    }
  },
  components: {
    VueDetectFaces
  },
  methods: {
    stopStart () {
      this.startCamera = !this.startCamera
    },
    onFaceDetected (val) {
      // console.log(val)
      this.allFaces.push(val)
    }
  }
}
</script>

