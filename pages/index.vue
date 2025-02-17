<template>
  <div class="container mx-auto p-4">
    <h1 class="text-2xl font-bold mb-4">網頁卡拉OK麥克風</h1>
    <div class="space-y-4">
      <button
        class="bg-blue-500 text-white px-4 py-2 rounded" :class="{ 'bg-red-500': isActive }"
        @click="toggleMicrophone">
        {{ isActive ? '停止' : '開始' }}
      </button>
      <div class="space-y-2">
        <label class="block">
          音量增益:
          <input v-model="gainAmount" type="range" min="0" max="2" step="0.1" class="w-full">
        </label>
        <label class="block">
          迴響強度:
          <input v-model="reverbAmount" type="range" min="0" max="1" step="0.1" class="w-full">
        </label>
        <label class="block">
          迴響延遲:
          <input v-model="delayTime" type="range" min="0" max="1" step="0.01" class="w-full">
        </label>
        <label class="block">
          迴響長度:
          <input v-model="reverbLength" type="range" min="0.1" max="5" step="0.1" class="w-full">
        </label>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
const isActive = ref(false)
const reverbAmount = ref(0.3)
const delayTime = ref(0.005)
const gainAmount = ref(1.2)
const reverbLength = ref(2.0)
let audioContext: AudioContext | null = null
let stream: MediaStream | null = null
let source: MediaStreamAudioSourceNode | null = null
let reverb: ConvolverNode | null = null
let delay: DelayNode | null = null
let mainGainNode: GainNode | null = null

const initAudio = async () => {
  audioContext = new (window.AudioContext || window.webkitAudioContext)()

  mainGainNode = audioContext.createGain()
  mainGainNode.gain.value = gainAmount.value

  reverb = audioContext.createConvolver()
  const impulseResponse = await createImpulseResponse(audioContext, reverbLength.value)
  reverb.buffer = impulseResponse

  delay = audioContext.createDelay(1.0)
  delay.delayTime.value = delayTime.value

  // 更新音訊處理鏈
  mainGainNode.connect(delay)
  delay.connect(reverb)
  reverb.connect(audioContext.destination)
}

const createImpulseResponse = async (context: AudioContext, length: number) => {
  const sampleRate = context.sampleRate
  const bufferLength = Math.floor(sampleRate * length)
  const impulseResponse = context.createBuffer(2, bufferLength, sampleRate)

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

      const reverbGainNode = audioContext.createGain()
      reverbGainNode.gain.value = reverbAmount.value

      // 更新連接鏈
      source.connect(reverbGainNode)
      if (!mainGainNode || !delay) throw new Error('無法初始化音訊效果')
      reverbGainNode.connect(mainGainNode)

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

// 添加音量增益監聽
watch(gainAmount, (newValue) => {
  if (mainGainNode) {
    mainGainNode.gain.value = newValue
  }
})

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

watch(reverbLength, async (newValue) => {
  if (audioContext && reverb && source) {
    const newImpulseResponse = await createImpulseResponse(audioContext, newValue)
    reverb.buffer = newImpulseResponse
  }
})
</script>
