<template>
  <div class="container">
    <h1>PeerJS Call & Screen Recording</h1>

    <div class="video-container">
      <video ref="localVideo" autoplay muted></video>
      <video ref="remoteVideo" autoplay></video>
    </div>

    <div class="controls">
      <button @click="startScreenShare">Start Screen Share</button>
      <button @click="callServerPeer">Call Server</button>
      <button @click="stopRecording" :disabled="!isRecording">Stop Recording</button>
      <p>Your Peer ID: {{ peerId }}</p>
    </div>
  </div>
</template>

<script>
import Peer from 'peerjs';

export default {
  data() {
    return {
      peer: null,
      localStream: null,
      mediaRecorder: null,
      recordedChunks: [],
      isRecording: false,
      peerId: '',
      call: null,
      serverPeerId: 'server-peer-id', // Replace with your actual server's Peer ID
    };
  },
  mounted() {
    // Initialize PeerJS with the configured secure server
    this.peer = new Peer({
      host: 'localhost',
      port: 8000,
      path: '/peerjs',
      secure: false,
    });
    console.log(this.peer)
    // Display the peer ID once connected
    this.peer.on('open', (id) => {
      console.log(`My Peer ID is: ${id}`);
      this.peerId = id;
    });

    // Handle incoming streams from the server
    this.peer.on('call', (incomingCall) => {
      console.log(`Receiving call from ${incomingCall.peer}`);
      incomingCall.answer(this.localStream);
      this.setupCallEvents(incomingCall);
    });
  },
  methods: {
    async startScreenShare() {
      try {
        // Get the screen stream
        this.localStream = await navigator.mediaDevices.getDisplayMedia({
          video: true,
          audio: true,
        });

        this.$refs.localVideo.srcObject = this.localStream;

        // Start recording the screen stream
        this.startRecording(this.localStream);
      } catch (err) {
        console.error('Failed to get screen media:', err);
      }
    },

    callServerPeer() {
      // Initiate a call to the server with the local screen stream
      this.call = this.peer.call(this.serverPeerId, this.localStream);
      this.setupCallEvents(this.call);
    },

    setupCallEvents(call) {
      call.on('stream', (remoteStream) => {
        console.log('Received remote stream');
        this.$refs.remoteVideo.srcObject = remoteStream;
      });

      call.on('close', () => {
        console.log('Call ended');
        alert('Call ended');
      });
    },

    startRecording(stream) {
      this.recordedChunks = [];
      this.mediaRecorder = new MediaRecorder(stream, {
        mimeType: 'video/webm; codecs=vp9',
      });

      this.mediaRecorder.ondataavailable = (event) => {
        if (event.data.size > 0) {
          this.recordedChunks.push(event.data);
        }
      };

      this.mediaRecorder.onstop = this.saveRecording;
      this.mediaRecorder.start();
      this.isRecording = true;
      console.log('Recording started');
    },

    stopRecording() {
      if (this.mediaRecorder) {
        this.mediaRecorder.stop();
        this.isRecording = false;
      }
    },

    saveRecording() {
      const blob = new Blob(this.recordedChunks, { type: 'video/webm' });
      const url = URL.createObjectURL(blob);

      const a = document.createElement('a');
      a.style.display = 'none';
      a.href = url;
      a.download = 'screen-recording.webm';
      document.body.appendChild(a);
      a.click();
      window.URL.revokeObjectURL(url);
      console.log('Recording saved');
    },
  },
};
</script>

<style scoped>
.container {
  max-width: 800px;
  margin: 0 auto;
  text-align: center;
}
.video-container {
  display: flex;
  justify-content: space-around;
  margin-top: 20px;
}
video {
  width: 45%;
  border: 1px solid #ddd;
  border-radius: 5px;
}
.controls {
  margin-top: 20px;
}
button {
  margin-right: 10px;
}
</style>
