<template>
  <div class="container mx-auto p-4">
    <h1 class="text-2xl font-bold mb-4">網頁卡拉OK麥克風</h1>
    <div class="space-y-4">
      <button
        class="bg-blue-500 text-white px-4 py-2 rounded"
        :class="{ 'bg-red-500': isActive }"
        @click="toggleMicrophone"
      >
        {{ isActive ? '停止' : '開始' }}
      </button>
      <div class="space-y-2">
        <label class="block">
           迴響強度:
          <input
            v-model="reverbAmount"
            type="range"
            min="0"
            max="1"
            step="0.1"
            class="w-full"
          >
        </label>
        <label class="block">
          迴響延遲:
          <input
            v-model="delayTime"
            type="range"
            min="0"
            max="1"
            step="0.01"
            class="w-full"
          >
        </label>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
const isActive = ref(false)
const reverbAmount = ref(0.5)
const delayTime = ref(0.01)
let audioContext: AudioContext | null = null
let stream: MediaStream | null = null
let source: MediaStreamAudioSourceNode | null = null
let reverb: ConvolverNode | null = null
let delay: DelayNode | null = null

const initAudio = async () => {
  audioContext = new (window.AudioContext || window.webkitAudioContext)()

  reverb = audioContext.createConvolver()
  const impulseResponse = await createImpulseResponse(audioContext)
  reverb.buffer = impulseResponse

  delay = audioContext.createDelay(1.0)
  delay.delayTime.value = delayTime.value

  delay.connect(reverb)
  reverb.connect(audioContext.destination)
}

const createImpulseResponse = async (context: AudioContext) => {
  const sampleRate = context.sampleRate
  const length = sampleRate / 2
  const impulseResponse = context.createBuffer(2, length, sampleRate)

  for (let channel = 0; channel < 2; channel++) {
    const channelData = impulseResponse.getChannelData(channel)
    for (let i = 0; i < length; i++) {
      channelData[i] = (Math.random() * 2 - 1) * Math.exp(-i / (sampleRate * 0.1))
    }
  }
  return impulseResponse
}

const toggleMicrophone = async () => {
  if (!isActive.value) {
    try {
      if (!audioContext) await initAudio()
      if (!audioContext) throw new Error('無法初始化音訊環境')

      stream = await navigator.mediaDevices.getUserMedia({ audio: true })
      source = audioContext.createMediaStreamSource(stream)

      const gainNode = audioContext.createGain()
      gainNode.gain.value = reverbAmount.value

      source.connect(gainNode)
      if (!delay) throw new Error('無法初始化延遲效果')
      gainNode.connect(delay)

      isActive.value = true
    } catch (error) {
      console.error('無法存取麥克風:', error)
    }
  } else {
    if (stream) {
      stream.getTracks().forEach(track => track.stop())
      if (source) source.disconnect()
      isActive.value = false
    }
  }
}

watch(reverbAmount, (newValue) => {
  if (audioContext && reverb) {
    const gainNode = audioContext.createGain()
    gainNode.gain.value = newValue
    if (source) {
      source.disconnect()
      source.connect(gainNode)
      gainNode.connect(reverb)
    }
  }
})

watch(delayTime, (newValue) => {
  if (delay) {
    delay.delayTime.value = newValue
  }
})
</script>
