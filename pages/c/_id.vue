<template>
  <v-layout
    column
    justify-center
    align-center
  >
    <v-snackbar v-model="snackbar.show" :timeout="snackbar.timeout">
      {{ snackbar.message }}
    </v-snackbar>

    <v-container class="d-flex justify-center">
      <div id="videos">
        <video id="localVideo" autoplay muted></video>
      </div>
    </v-container>
  </v-layout>
</template>

<script>

  export default {
    data () {
      return {
        text: 'Text',
        pc: null,
        room: null,
        configuration: '',
        drone: null,
        observableId: null,
        snackbar: {
          show: false,
          message: '',
          timeout: 1000
        },
        localStream: null,
        remoteStream: null,
        connectedUsers: []
      }
    },
    async mounted () {
      try {

        await this.openUserMedia()

        console.log('openMediaDevices: ', this.localStream)

        // eslint-disable-next-line no-undef
        this.drone = new ScaleDrone('XhdNuofanemp85RM')
        this.observableId = 'observable-' + this.$route.params.id
        // eslint-disable-next-line no-unused-vars
        this.configuration = {
          iceServers: [
            { url: 'stun:stun.l.google.com:19302' },
          ],
          iceCandidatePoolSize: 10
        }
        this.drone.on('open',async (error) => {
          if (error) {
            console.error(error)
          }
          this.room = this.drone.subscribe(this.observableId)
          await this.room.on('open', async (error) => {

            if (error) {
              this.snackbar = {
                show: true,
                message: 'error on open websocket room',
                timeout: 1000
              }
              this.onError(error)
            }
          })
          this.room.on('members', async (members) => {
            console.log('members: ', members)
            for (const {id} of members) {
              if (this.$route.query.type !== 'creator'){
                await this.registerNewConnections(id)
              }else {
                await this.listenNewConnections(id)
              }
            }
          })

          this.room.on('member_join', async ({id}) => {
            await this.listenNewConnections(id)
          })

          this.room.on('member_leave', ({id}) => {
            document.querySelector(`#remoteVideo-${id}`).remove()
          })

        })
      } catch (error) {
        this.snackbar = {
          show: true,
          message: 'Error accessing media devices. ' + error,
          timeout: 1000
        }
        console.error('Error accessing media devices.', error)
      }
    },
    methods: {

      registerPeerConnectionListeners(peerConnection, id) {
        peerConnection.addEventListener('icegatheringstatechange', () => {
          console.log(
            `ICE gathering state changed: ${peerConnection.iceGatheringState}`);
        });

        peerConnection.addEventListener('connectionstatechange', () => {
          console.log(`Connection state change: ${peerConnection.connectionState}`);
          if (peerConnection.connectionState === "disconnected") {
            document.querySelector(`#remoteVideo-${id}`).remove()
          }
        });

        peerConnection.addEventListener('signalingstatechange', () => {
          console.log(`Signaling state change: ${peerConnection.signalingState}`);
        });

        peerConnection.addEventListener('iceconnectionstatechange ', () => {
          console.log(
            `ICE connection state change: ${peerConnection.iceConnectionState}`);
        });
      },

      async openUserMedia(e) {
        let stream;
        try {
          stream = await navigator.mediaDevices.getUserMedia(
            {
              video: {
                width: 360,
                height: 240,
                aspectRatio: 1.5,
                frameRate: 25
              },
              audio: {
                sampleSize: 16,
                channelCount: 2,
                echoCancellation: true
              }
            });

        } catch (e) {
          console.log(e)
          alert(`Permission denied. Refresh to try again.`)
          throw new Error("Something went badly wrong!");
        }

        document.querySelector('#localVideo').srcObject = stream;
        this.localStream = stream;

        console.log('Stream:', document.querySelector('#localVideo').srcObject);
      },
      sendMessage (message) {
        this.drone.publish({
          room: this.observableId,
          message
        })

        this.snackbar = {
          show: true,
          message: 'message sended on websocket room: ' + message,
          timeout: 1000
        }
      },
      async registerNewConnections (id) {
        const remoteStream = new MediaStream();
        const videoElement = document.createElement('video');
        videoElement.id = `remoteVideo-${id}`;
        videoElement.autoplay = true;
        document.querySelector('#videos').appendChild(videoElement);
        document.querySelector(`#remoteVideo-${id}`).srcObject = remoteStream

        this.connectedUsers[id] = {};
        this.connectedUsers[id].remoteTrack = remoteStream

        this.pc = new RTCPeerConnection(this.configuration)
        console.log('new RTCPeerConnection', this.pc)

        this.registerPeerConnectionListeners(this.pc, id);

        // Add your stream to be sent to the conneting peer
        this.localStream.getTracks().forEach(track => this.pc.addTrack(track, this.localStream))

        this.pc.onicecandidate = (event) => {
          console.log('onicecandidate', event)
          if (event.candidate) {
            console.log('ICE Candidate: ' + event.candidate)
            this.sendMessage({ 'candidate': event.candidate })
          }else {
            return;
          }
        }

        const offer = await this.pc.createOffer();
        await this.localDescCreated(offer);
        console.log("Created offer:", offer);

        this.pc.ontrack = (event) => {
          event.streams[0].getTracks().forEach((track) => {
            console.log("Add a track to the remoteStream:", track);
            this.connectedUsers[id].remoteTrack.addTrack(track)
          });
        }

        // Listen to signaling data from Scaledrone
        this.room.on('data', async (message, client) => {
          // Message was sent by us
          if (client.id === this.drone.clientId) {
            return
          }

          if (message.sdp) {
            console.log('message.answer', message.sdp)
            if (!this.pc.currentRemoteDescription && message.sdp) {
              console.log('Got remote description: ', message.sdp);
              const rtcSessionDescription = new RTCSessionDescription(message.sdp);
              await this.pc.setRemoteDescription(rtcSessionDescription);
            }
            console.log('on answer: ', message.sdp)
          } else if (message.candidate) {
            // Add the new ICE candidate to our connections remote description
            await this.pc.addIceCandidate(
              new RTCIceCandidate(message.candidate), this.onSuccess, this.onError
            )
            console.log('on candidate: ', message.candidate)
          }
        })
      },

      async listenNewConnections(id) {
        const remoteStream = new MediaStream();
        const videoElement = document.createElement('video');
        videoElement.id = `remoteVideo-${id}`;
        videoElement.autoplay = true;
        document.querySelector('#videos').appendChild(videoElement);
        document.querySelector(`video#remoteVideo-${id}`).srcObject = remoteStream

        this.connectedUsers[id] = {};
        this.connectedUsers[id].remoteTrack = remoteStream

        // eslint-disable-next-line no-undef,prefer-const
        this.pc = await new RTCPeerConnection(this.configuration)
        console.log('new RTCPeerConnection', this.pc)

        this.registerPeerConnectionListeners(this.pc, id);

        // Add your stream to be sent to the conneting peer
        this.localStream.getTracks().forEach(track => this.pc.addTrack(track, this.localStream))

        this.pc.ontrack = (event) => {
          event.streams[0].getTracks().forEach((track) => {
            console.log("Add a track to the remoteStream:", track);
            this.connectedUsers[id].remoteTrack.addTrack(track)
          });
        }

        this.pc.onicecandidate = (event) => {
          console.log('onicecandidate', event)
          if (event.candidate) {
            console.log('ICE Candidate: ' + event.candidate)
            this.sendMessage({ 'candidate': event.candidate })
          }
        }

        // Listen to signaling data from Scaledrone
        this.room.on('data', async (message, client) => {
          // Message was sent by us
          if (client.id === this.drone.clientId) {
            return
          }

          console.log('message: ', message)

          if (message.sdp) {
            // This is called after receiving an offer or answer from another peer

            console.log('Got remote description: ', message.sdp);
            await this.pc.setRemoteDescription(
              new RTCSessionDescription(message.sdp)
            );

            const answer = await this.pc.createAnswer()
            console.log("Created answer:", answer);
            this.localDescCreated(answer)

          } else if (message.candidate) {
            // Add the new ICE candidate to our connections remote description
            await this.pc.addIceCandidate(
              new RTCIceCandidate(message.candidate), this.onSuccess, this.onError
            )
            console.log('on candidate: ', message.candidate)
          }
        })
        this.connectedUsers[id].peerConnection = this.pc
      }
      ,
      onSuccess (result) {
        console.log('success', result)
        this.snackbar = {
          show: true,
          message: 'success' + result,
          timeout: 1000
        }
      },
      onError (error) {
        console.error(error)
        this.snackbar = {
          show: true,
          message: 'error: ' + error,
          timeout: 1000
        }
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
        this.snackbar = {
          show: true,
          message: 'sended sdp: ' + this.pc.localDescription,
          timeout: 1000
        }
      }
    }
  }
</script>
