<template>
    <v-flex>
      <v-list class="pa-4" color="yellow" subheader>
        <v-subheader class="mb-4">
          <h1 style="font-family: 'Source Code Pro', monospace;">Projects</h1>
        </v-subheader>
        <v-list-item class="pa-8" style="background-color: greenyellow; display: inline-block">
          <v-list-item-title>
            <h2 style="font-family: 'Source Code Pro', monospace; font-size: 22px">Sample---WebRTC</h2>
          </v-list-item-title>
          <v-list-item-content>
            <p>
              With WebRTC, you can add real-time communication capabilities to your application that works on top of an open standard. It supports video, voice, and generic data to be sent between peers, allowing developers to build powerful voice- and video-communication solutions. The technology is available on all modern browsers as well as on native clients for all major platforms. The technologies behind WebRTC are implemented as an open web standard and available as regular JavaScript APIs in all major browsers. For native clients, like Android and iOS applications, a library is available that provides the same functionality. The WebRTC project is open-source and supported by Apple, Google, Microsoft and Mozilla, amongst others. This page is maintained by the Google WebRTC team.
            </p>
          </v-list-item-content>
          <v-list-item-action-text>
            <nuxt-link to="/projects/sample---webrtc"  style="font-family: 'Source Code Pro', monospace;"><h2>Go -></h2></nuxt-link>
          </v-list-item-action-text>
        </v-list-item>
      </v-list>
    </v-flex>
</template>

<script>
  export default {
    name: 'index'
  }
</script>

<style scoped>

</style>
