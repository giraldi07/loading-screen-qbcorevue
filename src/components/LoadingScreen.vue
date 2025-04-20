<template>
  <div class="loading-container">
    <!-- Background Elements (unchanged) -->
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
      </header>

      <!-- Left Popup: Tips -->
      <section class="popup popup-left">
        <h2 class="popup-title">💡 Tips</h2>
        <ul class="popup-list">
          <li><strong>/help</strong> – Lihat daftar perintah penting.</li>
          <li>Gunakan <strong>voice chat</strong> untuk interaksi yang lebih nyata.</li>
          <li>Pahami roleplay rules – <em>fairplay</em> itu keren!</li>
          <li>Jangan lupa <strong>/report</strong> kalau ada masalah.</li>
          <li>Aktif di <strong>Discord</strong> biar update terus!</li>
        </ul>
      </section>


      <!-- Right Popup: Developer List -->
      <section class="popup popup-right">
        <h2 class="popup-title">👨‍💻 Developers</h2>
        <ul class="popup-dev-list">
          <li class="dev-card">
            <img src="/man.png" alt="GeraldPra Avatar" class="dev-avatar" />
            <div class="dev-info">
              <p class="dev-name">GeraldPra</p>
              <p class="dev-role">Fullstack Architect</p>
            </div>
          </li>
          <li class="dev-card">
            <img src="/man.png" alt="Charles Christ Avatar" class="dev-avatar" />
            <div class="dev-info">
              <p class="dev-name">Charles Christ</p>
              <p class="dev-role">Gameplay Engineer</p>
            </div>
          </li>
          <li class="dev-card">
            <img src="/man.png" alt="Setyo Hp Avatar" class="dev-avatar" />
            <div class="dev-info">
              <p class="dev-name">Setyo Hp</p>
              <p class="dev-role">Infrastructure Specialist</p>
            </div>
          </li>
        </ul>
      </section>



      <!-- Main Content -->
      <main class="main-content">
        <div class="title-wrapper">
          <h1 class="main-title">
            Welcome to <span class="highlight">Blossom City</span>
          </h1>
          <p class="subtitle">ROLEPLAY SERVER</p>
        </div>

        <div class="loading-progress">
          <div class="progress-bar" :style="{ width: `${progress}%` }"></div>
          <span class="progress-text">Loading world data... {{ progress }}%</span>
        </div>

        <div class="tagline-scroller">
          <div class="tagline-track">
            <p class="tagline-line">Connecting to the city of dreams...</p>
            <p class="tagline-line">Where your next story begins.</p>
            <p class="tagline-line">Live your story. Shape your fate.</p>
            <p class="tagline-line">Only in Blossom City.</p>

            <!-- Duplicate for seamless looping -->
            <p class="tagline-line">Connecting to the city of dreams...</p>
            <p class="tagline-line">Where your next story begins.</p>
            <p class="tagline-line">Live your story. Shape your fate.</p>
            <p class="tagline-line">Only in Blossom City.</p>
          </div>
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

    <!-- Audio element - Using one that works as default -->
    <audio ref="audioPlayer" preload="auto"></audio>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted, watch } from 'vue'
import { SkipBack as LucideSkipBack, SkipForward as LucideSkipForward } from 'lucide-vue-next'

// Updated tracks with working paths - using the ones that aren't blocked
const tracks = [
  { name: 'Neon Skyline', url: '/music/jump-around.mp3' },
  { name: 'Cruising Waves', url: '/music/my-own-summer.mp3' },
  { name: 'Midnight Drift', url: '/music/hold-your-tongue.mp3' }, // Moved to end since it might be blocked
]

const currentTrackIndex = ref(0)
const currentTrack = ref(tracks[currentTrackIndex.value])
const audioPlayer = ref<HTMLAudioElement | null>(null)
const progress = ref(0)
const playerCount = ref(128)
const spectrumContainer = ref<HTMLElement | null>(null)
const spectrumBars = ref<Array<{ height: number; color: string }>>([])
const audioContext = ref<AudioContext | null>(null)
const analyser = ref<AnalyserNode | null>(null)
const dataArray = ref<Uint8Array | null>(null)
const animationFrameId = ref<number | null>(null)
const audioInitialized = ref(false)

// Helper for smooth transition
const lerp = (a: number, b: number, t: number) => a + (b - a) * t

// Initialize audio spectrum
const initAudioSpectrum = () => {
  if (!audioPlayer.value || audioInitialized.value) return

  try {
    // Audio context setup
    const AudioContext = window.AudioContext || (window as any).webkitAudioContext
    audioContext.value = new AudioContext()
    const source = audioContext.value.createMediaElementSource(audioPlayer.value)

    // Analyser config
    analyser.value = audioContext.value.createAnalyser()
    analyser.value.fftSize = 1024
    analyser.value.smoothingTimeConstant = 0.85 // smooth decay

    source.connect(analyser.value)
    analyser.value.connect(audioContext.value.destination)

    // Frequency data array
    const bufferLength = analyser.value.frequencyBinCount
    dataArray.value = new Uint8Array(bufferLength)

    // Visual bars
    const barCount = 40
    spectrumBars.value = Array.from({ length: barCount }, () => ({
      height: 1,
      color: '#ff0000'
    }))

    audioInitialized.value = true
  } catch (error) {
    console.error("Error initializing audio context:", error)
  }
}

// Update visualization frame
const updateSpectrum = () => {
  if (!analyser.value || !dataArray.value) return

  analyser.value.getByteFrequencyData(dataArray.value)

  const barCount = spectrumBars.value.length
  const totalBins = dataArray.value.length

  for (let i = 0; i < barCount; i++) {
    // Logarithmic frequency bin mapping
    const logIndex = Math.pow(i / barCount, 2) * (totalBins - 1)
    const binStart = Math.floor(logIndex)
    const binEnd = Math.min(totalBins - 1, binStart + 2)

    // Average frequency value in the bin range
    let sum = 0
    for (let j = binStart; j <= binEnd; j++) {
      sum += dataArray.value[j]
    }
    const avg = sum / (binEnd - binStart + 1)

    // Scaled height for visual bars
    const scaledHeight = 1 + Math.pow(avg / 255, 2.5) * 40

    // Smooth height transition
    spectrumBars.value[i].height = lerp(spectrumBars.value[i].height, scaledHeight, 0.2)

    // Color via HSL gradient
    spectrumBars.value[i].color = `hsl(${(i / barCount) * 360}, 100%, 60%)`
  }

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
  if (!audioPlayer.value) return
  
  try {
    // Set the source
    audioPlayer.value.src = currentTrack.value.url
    
    // Resume audio context if suspended
    if (audioContext.value?.state === 'suspended') {
      audioContext.value.resume()
    }
    
    // Play audio with user interaction requirement handling
    const playPromise = audioPlayer.value.play()
    
    if (playPromise !== undefined) {
      playPromise.catch(error => {
        console.error("Audio playback error:", error)
        // Auto-fallback to next song if current one fails
        if (error.name === 'NotAllowedError' || error.name === 'AbortError') {
          console.log("Trying next song due to playback issue")
          nextSong()
        }
      })
    }
  } catch (error) {
    console.error("Play error:", error)
  }
}

// Watch for changes to the audio player reference
watch(audioPlayer, (newPlayer) => {
  if (newPlayer && !audioInitialized.value) {
    // Add event listeners for audio element
    newPlayer.addEventListener('canplaythrough', () => {
      console.log("Audio can play through")
    })
    
    newPlayer.addEventListener('error', (e) => {
      console.error("Audio error:", e)
      // Try next song on error
      nextSong()
    })
  }
})

onMounted(() => {
  // Initialize after a small delay to ensure DOM is fully loaded
  setTimeout(() => {
    // Set up click handler for the entire document to initialize audio
    document.addEventListener('click', () => {
      if (!audioInitialized.value && audioPlayer.value) {
        initAudioSpectrum()
        // Set initial source
        audioPlayer.value.src = currentTrack.value.url
        play()
      }
    }, { once: true })
    
    // Still try to initialize
    initAudioSpectrum()
    
    // Set initial source
    if (audioPlayer.value) {
      audioPlayer.value.src = currentTrack.value.url
    }
    
    startLoadingSimulation()
    
    // Try to play (might be blocked without user interaction)
    play()
    
    // Start spectrum animation when audio plays
    audioPlayer.value?.addEventListener('play', () => {
      if (audioContext.value?.state === 'suspended') {
        audioContext.value.resume()
      }
      updateSpectrum()
    })
  }, 100)
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