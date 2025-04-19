<template>
  <div class="loading-container">
    <!-- Background Elements -->
    <video autoplay muted loop class="bg-video">
      <source src="/videoclip.mp4" type="video/mp4" />
    </video>
    <div class="gradient-overlay"></div>

    <!-- Content Grid Layout -->
    <div class="content-grid">
      <!-- Header Section -->
      <header class="header">
        <div class="logo-wrapper">
          <img src="/blossombiz.svg" alt="Server Logo" class="logo" />
          <div class="server-info">
            <span class="server-version">v5.0.2</span>
            <span class="server-status online">ONLINE</span>
          </div>
        </div>
      </header>

      <!-- Main Content -->
      <main class="main-content">
        <div class="title-wrapper">
          <h1 class="main-title">
            Welcome to <span class="highlight">BB City</span>
          </h1>
          <p class="subtitle">ROLEPLAY SERVER</p>
        </div>

        <div class="audio-spectrum" ref="spectrumContainer">
          <div 
            v-for="(bar, index) in spectrumBars" 
            :key="index" 
            class="spectrum-bar" 
            :style="{ 
              height: `${bar.height}px`, 
              background: bar.color,
              opacity: bar.height / 100
            }"
          ></div>
        </div>

        <div class="loading-progress">
          <div class="progress-bar" :style="{ width: `${progress}%` }"></div>
          <span class="progress-text">Loading world data... {{ progress }}%</span>
        </div>

        <div class="tagline-wrapper">
          <p class="tagline">
            <span class="tagline-text">Connecting to the city of dreams...</span>
            <span class="tagline-text">Where your next story begins.</span>
          </p>
        </div>
      </main>

      <!-- Footer Section -->
      <footer class="footer">
        <div class="player-count">
          <span class="count">{{ playerCount }}</span>
          <span>PLAYERS ONLINE</span>
        </div>

        <div class="music-player">
          <button @click="prevSong" class="player-control">
            <LucideSkipBack size="18" />
          </button>
          <div class="track-info">
            <p class="track-name">{{ currentTrack.name }}</p>
            <p class="track-artist">BB City Radio</p>
          </div>
          <button @click="nextSong" class="player-control">
            <LucideSkipForward size="18" />
          </button>
        </div>

        <div class="quick-info">
          <span class="info-item">FIVEM</span>
          <span class="info-item">RP v2.3</span>
          <span class="info-item">BB-CITY.COM</span>
        </div>
      </footer>
    </div>

    <!-- Audio element (now properly integrated) -->
    <audio ref="audioPlayer" :src="currentTrack.url" loop></audio>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'
import { SkipBack as LucideSkipBack, SkipForward as LucideSkipForward } from 'lucide-vue-next'

const tracks = [
  { name: 'Midnight Drift', url: '/music/hold-your-tongue.mp3' },
  { name: 'Neon Skyline', url: '/music/jump-around.mp3' },
  { name: 'Cruising Waves', url: '/music/my-own-summer.mp3' },
]

const currentTrackIndex = ref(0)
const currentTrack = ref(tracks[currentTrackIndex.value])
const audioPlayer = ref<HTMLAudioElement | null>(null)
const progress = ref(0)
const spectrumContainer = ref<HTMLElement | null>(null)
const spectrumBars = ref<Array<{ height: number; color: string }>>([])
const audioContext = ref<AudioContext | null>(null)
const analyser = ref<AnalyserNode | null>(null)
const dataArray = ref<Uint8Array | null>(null)
const animationFrameId = ref<number | null>(null)

// Initialize audio spectrum
const initAudioSpectrum = () => {
  if (!audioPlayer.value) return

  // Create audio context
  const AudioContext = window.AudioContext || (window as any).webkitAudioContext
  audioContext.value = new AudioContext()
  const source = audioContext.value.createMediaElementSource(audioPlayer.value)
  
  // Create analyser
  analyser.value = audioContext.value.createAnalyser()
  analyser.value.fftSize = 256
  source.connect(analyser.value)
  analyser.value.connect(audioContext.value.destination)
  
  // Initialize data array
  const bufferLength = analyser.value.frequencyBinCount
  dataArray.value = new Uint8Array(bufferLength)
  
  // Initialize spectrum bars
  const barCount = 50
  spectrumBars.value = Array(barCount).fill(0).map((_, i) => ({
    height: 2,
    color: `hsl(${(i * 360) / barCount}, 100%, 50%)`
  }))
}

// Update spectrum visualization
const updateSpectrum = () => {
  if (!analyser.value || !dataArray.value) return

  analyser.value.getByteFrequencyData(dataArray.value)
  
  // Normalize and update bar heights
  const barCount = spectrumBars.value.length
  const step = Math.floor(dataArray.value.length / barCount)
  
  spectrumBars.value.forEach((bar, i) => {
    const value = dataArray.value![i * step] || 0
    bar.height = 2 + (value / 255) * 100 // Scale height from 2px to 102px
  })
  
  animationFrameId.value = requestAnimationFrame(updateSpectrum)
}

// Simulate loading progress
const startLoadingSimulation = () => {
  const interval = setInterval(() => {
    if (progress.value < 100) {
      progress.value += Math.floor(Math.random() * 5) + 1
      if (progress.value > 100) progress.value = 100
    } else {
      clearInterval(interval)
    }
  }, 300)
}

const nextSong = () => {
  currentTrackIndex.value = (currentTrackIndex.value + 1) % tracks.length
  currentTrack.value = tracks[currentTrackIndex.value]
  play()
}

const prevSong = () => {
  currentTrackIndex.value = (currentTrackIndex.value - 1 + tracks.length) % tracks.length
  currentTrack.value = tracks[currentTrackIndex.value]
  play()
}

const play = () => {
  if (audioPlayer.value) {
    // Resume audio context if suspended
    if (audioContext.value?.state === 'suspended') {
      audioContext.value.resume()
    }
    
    audioPlayer.value.load()
    audioPlayer.value.play().catch(() => {})
  }
}

onMounted(() => {
  initAudioSpectrum()
  play()
  startLoadingSimulation()
  
  // Start spectrum animation when audio plays
  audioPlayer.value?.addEventListener('play', () => {
    if (audioContext.value?.state === 'suspended') {
      audioContext.value.resume()
    }
    updateSpectrum()
  })
})

onUnmounted(() => {
  if (animationFrameId.value) {
    cancelAnimationFrame(animationFrameId.value)
  }
  if (audioContext.value) {
    audioContext.value.close()
  }
})
</script>