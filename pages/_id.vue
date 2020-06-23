<template>
  <div class="content">
    <video class="content__video-local" id="localVideo" autoplay muted />
    <video class="content__video-remote" id="remoteVideo" autoplay />
  </div>
</template>

<script>

export default {
  data () {
    return {
      pc: null,
      room: null,
      configuration: '',
      drone: null,
      observableId: null
    }
  },
  async mounted () {
    const openMediaDevices = async (constraints) => {
      // eslint-disable-next-line no-return-await
      return await navigator.mediaDevices.getUserMedia(constraints)
    }

    try {
      const stream = await openMediaDevices({ 'video': true, 'audio': true })

      console.log('openMediaDevices video and audio: ', stream)

      // eslint-disable-next-line no-undef
      this.drone = new ScaleDrone('XhdNuofanemp85RM')
      this.observableId = 'observable-' + this.$route.params.id
      // eslint-disable-next-line no-unused-vars
      this.configuration = {
        iceServers: [
          { url: 'stun:stun.l.google.com:19302' },
          {
            url: 'turn:coturn.myams.biz',
            credential: 'Q1w2E3r4T5y6',
            username: 'ams'
          }]
      }
      this.drone.on('open', (error) => {
        if (error) {
          console.error(error)
        }
        this.room = this.drone.subscribe(this.observableId)
        this.room.on('open', (error) => {
          if (error) {
            this.onError(error)
          }
        })
        // We're connected to the room and received an array of 'members'
        // connected to the room (including us). Signaling server is ready.
        this.room.on('members', (members) => {
          // console.log('MEMBERS', members)
          // If we are the second user to connect to the room we will be creating the offer
          const isOfferer = members.length === 2
          this.startWebRTC(isOfferer)
          console.log('members on start WebRTC connection: ', members)
        })
      })
    } catch (error) {
      console.error('Error accessing media devices.', error)
    }
  },
  methods: {
    sendMessage (message) {
      this.drone.publish({
        room: this.observableId,
        message
      })
    },
    async startWebRTC (isOfferer) {
      // eslint-disable-next-line no-undef,prefer-const
      this.pc = await new RTCPeerConnection(this.configuration)
      console.log('new RTCPeerConnection', this.pc)

      // 'onicecandidate' notifies us whenever an ICE agent needs to deliver a
      // message to the other peer through the signaling server
      this.pc.onicecandidate = (event) => {
        console.log('onicecandidate', event)
        if (event.candidate) {
          console.log('ICE Candidate: ' + event.candidate)
          this.sendMessage({ 'candidate': event.candidate })
        }
      }

      // If user is offerer let the 'negotiationneeded' event create the offer
      if (isOfferer) {
        this.pc.onnegotiationneeded = () => {
          this.pc.createOffer().then(this.localDescCreated).catch(this.onError)
        }
      }

      // When a remote stream arrives display it in the #remoteVideo element
      this.pc.ontrack = (event) => {
        console.log('ontrack')
        const remoteVideo = document.querySelector('video#remoteVideo')
        const stream = event.streams[0]
        if (!remoteVideo.srcObject || remoteVideo.srcObject.id !== stream.id) {
          remoteVideo.srcObject = stream
        }
      }
      navigator.mediaDevices.getUserMedia({
        audio: true,
        video: true
      }).then((stream) => {
        const localVideo = document.querySelector('video#localVideo')
        // Display your local video in #localVideo element
        localVideo.srcObject = stream
        // Add your stream to be sent to the conneting peer
        stream.getTracks().forEach(track => this.pc.addTrack(track, stream))
      }, this.onError)

      // Listen to signaling data from Scaledrone
      this.room.on('data', (message, client) => {
        // Message was sent by us
        if (client.id === this.drone.clientId) {
          return
        }

        if (message.sdp) {
          console.log('setRemoteDescription on sdp: ', message.sdp)
          // This is called after receiving an offer or answer from another peer
          this.pc.setRemoteDescription(new RTCSessionDescription(message.sdp), () => {
            // When receiving an offer lets answer it
            if (this.pc.remoteDescription.type === 'offer') {
              this.pc.createAnswer().then(this.localDescCreated).catch(this.onError)
            }
          }, this.onError)
        } else if (message.candidate) {
          // Add the new ICE candidate to our connections remote description
          this.pc.addIceCandidate(
            new RTCIceCandidate(message.candidate), this.onSuccess, this.onError
          )
          console.log('addIceCandidate on candidate: ', message.candidate)
        }
      })
    },
    onSuccess (result) {
      console.log('success', result)
    },
    onError (error) {
      console.error(error)
    },
    localDescCreated (desc) {
      this.pc.setLocalDescription(
        desc,
        () => this.sendMessage({ 'sdp': this.pc.localDescription }),
        this.onError
      )
      // eslint-disable-next-line no-console
      console.log('local desc: ', desc)
      console.log('sended sdp: ', this.pc.localDescription)
    }
  }
}
</script>

<style>
  .content {
    position: relative;
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    width: 100%;
    height: 100vh;
  }

  .content__video-local {
    width: 50%;
    height: 50%;
    bottom: 0;
    align-self: flex-end;
    position: absolute;
  }

  @media (max-width: 768px) {
    .content__video-local {
      align-self: center;
    }
  }

  .content__video-remote {
    width: 100%;
    height: 100%;
  }
</style>
