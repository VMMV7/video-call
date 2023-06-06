<template>
  <q-page class="flex flex-center">
    <div class="absolute-full bg-blue-1">
      <div class="q-pa-lg" v-if="loading">
        <q-spinner size="md"/>
      </div>
      <!-- for Managing meeting status -->
      <div id="join-screen" class="q-pa-lg" v-if="!loading && !isJoined">
        <div v-if="msg" class="q-mb-sm">{{ msg }}</div>
        <div class="row items-center q-gutter-x-md">
          <!-- Create new Meeting Button -->
          <span><q-input dense outlined type="text" v-model="meetingId" placeholder="Enter Meeting id"/></span>
          <q-btn @click="createNewMeeting">New Meeting</q-btn>
          <div> - OR -</div>
          <!-- Join existing Meeting -->
          <div class="col-auto">
            <q-btn @click="joinMeeting">Join Meeting</q-btn>
          </div>
          <q-space/>
        </div>
      </div>


      <div id="grid-screen" class="full-width full-height q-pa-lg" v-show="isJoined">
        <!-- To Display MeetingId -->
        <h3>Meeting Id: {{ meetingId }}</h3>

        <!-- Controllers -->
        <q-btn color="grey" dense outlined @click="leaveMeeting">Leave</q-btn>
        <q-btn flat dense @click="toggleMic">Toggle Mic</q-btn>
        <q-btn flat dense @click="toggleWebcam">Toggle WebCam</q-btn>

        <!-- render Video -->
        <div id="videoContainer"></div>
      </div>
    </div>
  </q-page>
</template>

<script setup>
import {VideoSDK} from "@videosdk.live/js-sdk";
import {nextTick, onMounted, ref} from "vue";
import {Notify} from "quasar";

const loading = ref(false)
const isJoined = ref(false)
const joinNotify = ref(null)
const msg = ref('')
// const apiKey = 'd482eebc-c609-4001-a1a7-6ba931627407'
// const secretKey = '72341031899e0f85ebbc5166371d726fc05b77bfa6923ed2d7d7a30a4c5ce1f8'
const token = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhcGlrZXkiOiJkNDgyZWViYy1jNjA5LTQwMDEtYTFhNy02YmE5MzE2Mjc0MDciLCJwZXJtaXNzaW9ucyI6WyJhbGxvd19qb2luIl0sImlhdCI6MTY4NjA4ODU0NCwiZXhwIjoxNjg2NjkzMzQ0fQ.7KVBdWjyNqPzOu4TP8vwE8ySLzmaFSoyueAbiGDmcm0'
const meetingId = ref('')
const meeting = ref(null)
const localParticipant = ref(null)
const videoContainer = ref(null)

// options
const isMicOn = ref(false)
const isWebCamOn = ref(false)

// messages
const IS_JOINING = 'Joining the meeting...' // 'Please wait, we are joining the meeting'

async function joinMeeting() {
  msg.value = IS_JOINING
  loading.value = true
  const joinNotify = Notify.create({
    type: 'ongoing',
    message: IS_JOINING
  })

  await initializeMeeting()

  loading.value = false
  joinNotify()
}

async function createNewMeeting() {
  msg.value = IS_JOINING
  showLoading()

  // API call to create meeting
  const url = `https://api.videosdk.live/v2/rooms`;
  const options = {
    method: "POST",
    headers: {Authorization: token, "Content-Type": "application/json"},
  };

  try {
    const response = await fetch(url, options)
    const {roomId} = await response.json()
    meetingId.value = roomId;

    await initializeMeeting()
  } catch (error) {
    Notify.create({
      type: 'warning',
      message: error.message
    })
    msg.value = error.message
    hideLoading()
  }
}

function showLoading() {
  loading.value = true
  if (joinNotify.value) return
  joinNotify.value = Notify.create({
    type: 'ongoing',
    message: IS_JOINING
  })
}

function hideLoading() {
  loading.value = false
  if (joinNotify.value) joinNotify.value()
}

// Initialize meeting
async function initializeMeeting() {
  showLoading()
  VideoSDK.config(token);
  isJoined.value = false
  msg.value = ''

  meeting.value = VideoSDK.initMeeting({
    meetingId: meetingId.value, // required
    name: "Thomas Edison", // required
    micEnabled: true, // optional, default: true
    webcamEnabled: true, // optional, default: true
  });

  meeting.value.join();

  // Creating local participant
  createLocalParticipant();

  // Setting local participant stream
  meeting.value.localParticipant.on("stream-enabled", (stream) => {
    setTrack(stream, null, meeting.value.localParticipant, true);
  });

  // meeting joined event
  meeting.value.on("meeting-joined", () => {
    isJoined.value = true
    Notify.create({
      type: 'positive',
      message: 'Join Success.'
    })
    hideLoading()
  });


  //  participant joined
  meeting.value.on("participant-joined", (participant) => {
    let videoElement = createVideoElement(
      participant.id,
      participant.displayName
    );
    let audioElement = createAudioElement(participant.id);
    // stream-enabled
    participant.on("stream-enabled", (stream) => {
      setTrack(stream, audioElement, participant, false);
    });
    videoContainer.value.appendChild(videoElement);
    videoContainer.value.appendChild(audioElement);
    Notify.create({
      type: 'positive',
      message: 'Participant Join Success.'
    })
  });

  // participants left
  meeting.value.on("participant-left", (participant) => {
    let vElement = document.getElementById(`f-${participant.id}`);
    vElement.remove();

    let aElement = document.getElementById(`a-${participant.id}`);
    aElement.remove();
    Notify.create({
      type: 'positive',
      message: 'Participant Leave meeting.'
    })
  });
}

function setTrack(stream, audioElement, participant, isLocal) {
  if (stream.kind === "video") {
    isWebCamOn.value = true;
    const mediaStream = new MediaStream();
    mediaStream.addTrack(stream.track);
    let videoElm = document.getElementById(`v-${participant.id}`);
    videoElm.srcObject = mediaStream;
    videoElm
      .play()
      .catch((error) =>
        console.error("videoElem.current.play() failed", error)
      );
  } else if (stream.kind === "audio") {
    if (isLocal) {
      isMicOn.value = true;
    } else {
      const mediaStream = new MediaStream();
      mediaStream.addTrack(stream.track);
      audioElement.srcObject = mediaStream;
      audioElement
        .play()
        .catch((error) => console.error("audioElem.play() failed", error));
    }
  }
}

// creating local participant
function createLocalParticipant() {
  localParticipant.value = createVideoElement(
    meeting.value.localParticipant.id,
    meeting.value.localParticipant.displayName
  );
  videoContainer.value.appendChild(localParticipant.value);
}

async function leaveMeeting() {
  await meeting.value.leave()
}

function toggleMic() {

}

async function toggleWebcam() {
  isWebCamOn.value = !isWebCamOn.value
  if (isWebCamOn.value)
    await meeting.value.disableWebcam()
  else
    meeting.value.enableWebcam()
}

//#region create media tags
function createVideoElement(pId, name) {
  let videoFrame = document.createElement("div");
  videoFrame.setAttribute("id", `f-${pId}`);

  //create video
  let videoElement = document.createElement("video");
  videoElement.classList.add("video-frame");
  videoElement.setAttribute("id", `v-${pId}`);
  videoElement.setAttribute("playsinline", true);
  videoElement.setAttribute("width", "300");
  videoFrame.appendChild(videoElement);

  let displayName = document.createElement("div");
  displayName.innerHTML = `Name : ${name}`;

  videoFrame.appendChild(displayName);
  return videoFrame;
}

// creating audio element
function createAudioElement(pId) {
  let audioElement = document.createElement("audio");
  audioElement.setAttribute("autoPlay", "false");
  audioElement.setAttribute("playsInline", "true");
  audioElement.setAttribute("controls", "false");
  audioElement.setAttribute("id", `a-${pId}`);
  audioElement.style.display = "none";
  return audioElement;
}

//#endregion

onMounted(() => {
  nextTick().then(() => {
    videoContainer.value = document.getElementById('videoContainer')
  })
})
</script>
