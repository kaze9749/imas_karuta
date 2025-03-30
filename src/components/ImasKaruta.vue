<template>
  <div>
    <h1>アイドル再生アプリ</h1>
    <!-- フィルタ入力 -->
    <div>
      <label for="brand">Brand:</label>
      <select v-model="selectedBrand" id="brand">
        <option value="">全て</option>
        <option value="CinderellaGirls">CinderellaGirls</option>
        <option value="ShinyColors">ShinyColors</option>
        <option value="MillionLive!">MillionLive!</option>
        <option value="Campus">Campus</option>
        <option value="Others">Others</option>
      </select>
    </div>
    <div>
      <label for="ageInput">Age:</label>
      <input type="number" v-model.number="ageInput" id="ageInput" placeholder="数字を入力" />
      <select v-model="ageFilterType">
        <option value="above">以上</option>
        <option value="below">以下</option>
      </select>
    </div>
    <button @click="filterIdols">フィルタ実行</button>

    <!-- 再生モードの選択 -->
    <div style="margin-top:1em;">
      <label>再生モード:</label>
      <label>
        <input type="radio" value="auto" v-model="playbackMode" /> 自動再生
      </label>
      <label>
        <input type="radio" value="manual" v-model="playbackMode" /> 手動再生
      </label>
    </div>

    <!-- 自動再生モード設定 -->
    <div v-if="playbackMode === 'auto'" style="margin-top:1em;">
      <label for="intervalSec">再生間隔（秒）:</label>
      <input type="number" v-model.number="intervalSec" id="intervalSec" min="0" step="1" />
      <button @click="startAutoPlayback" :disabled="randomIdols.length === 0">自動再生開始</button>
    </div>

    <!-- 手動再生モード設定 -->
    <div v-if="playbackMode === 'manual'" style="margin-top:1em;">
      <button @click="startManualPlayback" :disabled="randomIdols.length === 0">手動再生開始</button>
      <div v-if="isManualPlaying" style="margin-top:1em;">
        <p>現在再生中: {{ currentIdolName }}</p>
        <button @click="playPrevious">前の曲</button>
        <button @click="replayCurrent">再生</button>
        <button @click="playNext">次の曲</button>
      </div>
    </div>

    <!-- audio 要素 -->
    <audio ref="audioPlayer"></audio>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'

// JSON データ
const idols = ref([])
// フィルタ後のアイドル（オブジェクト配列）
const filteredIdols = ref([])
// ランダムに並び替えたアイドルリスト
const randomIdols = ref([])

// フィルタ条件
const selectedBrand = ref("")
const ageInput = ref(null)
const ageFilterType = ref("above")

// 再生モード: "auto"（自動） or "manual"（手動）
const playbackMode = ref("auto")

// 自動再生用の間隔（秒）
const intervalSec = ref(3)

// 現在の再生インデックスと再生状態
const currentIndex = ref(0)
const isAutoPlaying = ref(false)
const isManualPlaying = ref(false)

// 現在再生中のアイドル名（手動モード表示用）
const currentIdolName = computed(() => {
  if (randomIdols.value.length > 0 && currentIndex.value < randomIdols.value.length) {
    return randomIdols.value[currentIndex.value].name
  }
  return ""
})

// audio 要素への参照
const audioPlayer = ref(null)

// JSON を読み込み（public/data/idols.json に配置）
async function fetchIdols() {
  const response = await fetch(`${process.env.BASE_URL}data/idols.json`)
  idols.value = await response.json()
}
fetchIdols()

// フィルタ実行：条件に合致するアイドルを抽出し、ランダムに並び替える
function filterIdols() {
  filteredIdols.value = idols.value.filter((idol) => {
    let matchBrand = true
    let matchAge = true
    if (selectedBrand.value !== "") {
      matchBrand = idol.brand === selectedBrand.value
    }
    if (ageInput.value !== null && ageInput.value !== "") {
      if (ageFilterType.value === "above") {
        matchAge = idol.age >= ageInput.value
      } else {
        matchAge = idol.age <= ageInput.value
      }
    }
    return matchBrand && matchAge
  })
  // ランダムに並び替え
  randomIdols.value = [...filteredIdols.value].sort(() => Math.random() - 0.5)
}

// 自動再生モード
function startAutoPlayback() {
  if (randomIdols.value.length === 0) return
  isAutoPlaying.value = true
  currentIndex.value = 0
  playCurrentAuto()
}

function playCurrentAuto() {
  if (currentIndex.value < randomIdols.value.length) {
    const idol = randomIdols.value[currentIndex.value]
    // 音声ファイルは /audio/{brand}/{code}_short.wav の形式
    const audioPath = `${process.env.BASE_URL}audio/${idol.brand}/short/${idol.code}_short.wav`
    audioPlayer.value.src = audioPath
    audioPlayer.value.play().catch(err => console.error(err))
  } else {
    // 全曲再生終了
    isAutoPlaying.value = false
  }
}

function onAudioEndedAuto() {
  if (!isAutoPlaying.value) return
  // 指定秒数待って次のファイルを再生
  setTimeout(() => {
    currentIndex.value++
    if (currentIndex.value < randomIdols.value.length) {
      playCurrentAuto()
    } else {
      isAutoPlaying.value = false
    }
  }, intervalSec.value * 1000)
}

// 手動再生モード
function startManualPlayback() {
  if (randomIdols.value.length === 0) return
  isManualPlaying.value = true
  currentIndex.value = 0
  playCurrentManual()
}

function playCurrentManual() {
  if (currentIndex.value < randomIdols.value.length) {
    const idol = randomIdols.value[currentIndex.value]
    const audioPath = `${process.env.BASE_URL}audio/${idol.brand}/short/${idol.code}_short.wav`
    audioPlayer.value.src = audioPath
    audioPlayer.value.src = audioPath
    audioPlayer.value.play().catch(err => console.error(err))
  }
}

function playPrevious() {
  if (currentIndex.value > 0) {
    currentIndex.value--
    playCurrentManual()
  }
}

function replayCurrent() {
  playCurrentManual()
}

function playNext() {
  if (currentIndex.value < randomIdols.value.length - 1) {
    currentIndex.value++
    playCurrentManual()
  }
}

// audio の ended イベントを利用して自動再生時に次へ
onMounted(() => {
  audioPlayer.value.addEventListener('ended', () => {
    if (playbackMode.value === "auto") {
      onAudioEndedAuto()
    }
  })
})
</script>
