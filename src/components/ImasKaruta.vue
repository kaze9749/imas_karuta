<template>
  <div class="container">
    <h1>アイマスかるた</h1>

    <!-- フィルタ条件 -->
    <div class="filter-group">
      <label for="ageInput">Age:</label>
      <input
        type="number"
        v-model.number="ageInput"
        id="ageInput"
        placeholder="数字を入力"
      />
      <select v-model="ageFilterType">
        <option value="above">以上</option>
        <option value="below">以下</option>
      </select>
      <div class="filter-row">
        <label>Blood Type:</label>
        <select v-model="bloodTypeFilter">
          <option value="">全て</option>
          <option value="A">A</option>
          <option value="B">B</option>
          <option value="O">O</option>
          <option value="AB">AB</option>
        </select>
      </div>
      <div class="filter-row">
        <label for="heightInput">Height:</label>
        <input
          type="number"
          v-model.number="heightInput"
          id="heightInput"
          placeholder="数字を入力"
        />
        <select v-model="heightFilterType">
          <option value="above">以上</option>
          <option value="below">以下</option>
        </select>
      </div>
      <div class="filter-row">
        <label for="birthMonthFilter">Birth Month:</label>
        <select v-model.number="birthMonthFilter" id="birthMonthFilter">
          <option value="">全て</option>
          <option v-for="month in 12" :key="month" :value="month">
            {{ month }}
          </option>
        </select>
      </div>
      <div class="filter-row">
        <label for="regionFilter">Region:</label>
        <input
          type="text"
          v-model="regionFilter"
          id="regionFilter"
          placeholder="例: 関東"
        />
      </div>
    </div>

    <!-- ブランド選択オプション -->
    <div class="filter-group">
      <h2>ブランド選択</h2>
      <div
        v-for="(option, brand) in brandOptions"
        :key="brand"
        class="filter-row"
      >
        <label>
          <input type="checkbox" v-model="brandOptions[brand].enabled" />
          {{ brand }}
        </label>
        <span v-if="brandOptions[brand].enabled">
          選択数:
          <input
            type="number"
            v-model.number="brandOptions[brand].count"
            min="1"
            :placeholder="getMaxCountPlaceholder(brand)"
          />
          （デフォルトは全件）
        </span>
      </div>
    </div>

    <!-- ソート(シャッフル)進捗バー -->
    <div v-if="isShuffling" class="progress-bar">
      <progress
        :value="shuffleProgress"
        max="100"
        style="width: 100%"
      ></progress>
      <span>{{ shuffleProgress }}%</span>
    </div>

    <button class="action-btn" @click="filterIdols" :disabled="isFiltering">
      フィルタ実行
    </button>

    <!-- 音声タイプ選択 -->
    <div class="filter-group">
      <h2>音声タイプ選択</h2>
      <label>
        <input type="radio" value="short" v-model="selectedAudioType" /> Short
      </label>
      <label>
        <input type="radio" value="long" v-model="selectedAudioType" /> Long
      </label>
    </div>

    <!-- 現在再生中の表示切替 -->
    <div class="filter-group">
      <label>
        <input type="checkbox" v-model="showCurrentIdolName" />
        現在再生中のアイドルを表示
      </label>
    </div>

    <!-- 追加：現在の順番と最大数 -->
    <p class="playback-count">
      {{ currentIndex + 1 }} / {{ randomIdols.length }}
    </p>

    <!-- 再生コントロール -->
    <div v-if="randomIdols.length > 0" class="playback-group">
      <!-- 外部からインデックス指定 -->

      <label class="index-input">
        再生インデックス:
        <input
          type="number"
          v-model="indexInput"
          :min="0"
          :max="randomIdols.length - 1"
          @blur="validateIndex"
        />
      </label>
      <br />

      <p v-if="showCurrentIdolName">現在再生中: {{ currentIdolName }}</p>

      <button
        class="control-btn"
        @click="playPrevious"
        :disabled="isFiltering || currentIndex === 0"
      >
        前のアイドル
      </button>
      <button
        class="control-btn"
        @click="replayCurrent"
        :disabled="isFiltering"
      >
        再生
      </button>
      <button
        class="control-btn"
        @click="playNext"
        :disabled="isFiltering || currentIndex === randomIdols.length - 1"
      >
        次のアイドル
      </button>
      <br />
      <br />
      <label>
        <input type="checkbox" v-model="isRepeatMode" /> 繰り返し再生モード
      </label>
      <label v-if="isRepeatMode">
        繰り返し間隔（秒）:
        <input type="number" v-model.number="repeatIntervalSec" min="1" />
      </label>
    </div>

    <!-- 定期再生モード -->
    <div v-if="randomIdols.length > 0" class="playback-group">
      <h2>定期再生モード</h2>
      <label>
        再生間隔（秒）:
        <input type="number" v-model.number="periodicIntervalSec" min="1" />
      </label>
      <button
        class="action-btn"
        @click="startPeriodicPlayback"
        :disabled="isPeriodicPlaying"
      >
        定期再生開始
      </button>
      <button
        class="action-btn"
        @click="stopPeriodicPlayback"
        :disabled="!isPeriodicPlaying"
      >
        定期再生停止
      </button>
    </div>

    <!-- audio 要素 -->
    <audio ref="audioPlayer"></audio>
  </div>
</template>

<script setup>
import { ref, computed, nextTick, watch, onMounted } from "vue";
/* global defineProps, defineEmits */

// 外部インデックス入力用の computed setter/getter
const indexInput = computed({
  get() {
    return currentIndex.value;
  },
  set(val) {
    if (val === "" || val == null) {
      // 空入力時は 0 にフォールバック
      currentIndex.value = 0;
    } else {
      const num = parseInt(val, 10);
      if (!isNaN(num)) {
        const max = randomIdols.value.length - 1;
        currentIndex.value = Math.min(Math.max(num, 0), max);
      }
    }
  },
});

// 入力フォーカスアウト時の追加バリデーション
function validateIndex() {
  const max = randomIdols.value.length - 1;
  if (
    currentIndex.value < 0 ||
    currentIndex.value > max ||
    isNaN(currentIndex.value)
  ) {
    currentIndex.value = 0;
  }
}

// 外部からインデックスを指定
const props = defineProps({ index: { type: Number, default: 0 } });
const emit = defineEmits(["update:index"]);
// フィルタ処理中フラグ
const isFiltering = ref(false);
// 繰り返し再生モード
const isRepeatMode = ref(false);
// 繰り返し再生間隔（秒）
const repeatIntervalSec = ref(3);
// 外部インデックスと同期
const currentIndex = ref(props.index);
watch(
  () => props.index,
  (val) => (currentIndex.value = val)
);
watch(currentIndex, (val) => emit("update:index", val));
onMounted(() => {
  audioPlayer.value?.addEventListener("ended", () => {
    if (isRepeatMode.value) {
      setTimeout(() => playCurrentManual(), repeatIntervalSec.value * 1000);
    }
  });
});

// idols.json のデータ（public/data/idols.json に配置）
const idols = ref([]);

// フィルタ条件
const ageInput = ref(null);
const ageFilterType = ref("above");

const bloodTypeFilter = ref("");
const heightInput = ref(null);
const heightFilterType = ref("above");
const birthMonthFilter = ref("");
const regionFilter = ref("");

// ソート(シャッフル)進捗表示用
const isShuffling = ref(false);
const shuffleProgress = ref(0);

// ブランド選択オプション
const brandOptions = ref({
  CinderellaGirls: { enabled: true, count: null },
  ShinyColors: { enabled: true, count: null },
  "MillionLive!": { enabled: true, count: null },
  Others: { enabled: true, count: null },
  Campus: { enabled: true, count: null },
});

// フィルタ実行後のアイドルリスト（ランダム順）
const randomIdols = ref([]);

// 定期再生モードの状態
const periodicIntervalSec = ref(3);
const isPeriodicPlaying = ref(false);
let periodicTimer = null;

// 音声タイプの選択（"short" または "long"）
const selectedAudioType = ref("short");

// 現在再生中のアイドル名表示のオン／オフ
const showCurrentIdolName = ref(true);

// audio 要素への参照
const audioPlayer = ref(null);

// 現在再生中のアイドル名（computed）
const currentIdolName = computed(() => {
  if (
    randomIdols.value.length > 0 &&
    currentIndex.value < randomIdols.value.length
  ) {
    return randomIdols.value[currentIndex.value].name;
  }
  return "";
});

// idols.json の読み込み
async function fetchIdols() {
  const response = await fetch(`${process.env.BASE_URL}data/idols.json`);
  idols.value = await response.json();
}
fetchIdols();

// フィルタ条件による絞り込み
function filteredByOtherCriteria() {
  return idols.value.filter((idol) => {
    let matchAge = true;
    if (ageInput.value !== null && ageInput.value !== "") {
      if (ageFilterType.value === "above") {
        matchAge = idol.age >= ageInput.value;
      } else {
        matchAge = idol.age <= ageInput.value;
      }
    }
    let matchBlood = true;
    if (bloodTypeFilter.value !== "") {
      matchBlood = idol.bloodType === bloodTypeFilter.value;
    }
    let matchHeight = true;
    if (heightInput.value !== null && heightInput.value !== "") {
      if (heightFilterType.value === "above") {
        matchHeight = idol.height >= heightInput.value;
      } else {
        matchHeight = idol.height <= heightInput.value;
      }
    }
    let matchBirthMonth = true;
    if (birthMonthFilter.value !== "" && birthMonthFilter.value !== null) {
      matchBirthMonth = idol.birthMonth === Number(birthMonthFilter.value);
    }
    let matchRegion = true;
    if (regionFilter.value !== "") {
      matchRegion = idol.region
        .toLowerCase()
        .includes(regionFilter.value.toLowerCase());
    }
    return (
      matchAge && matchBlood && matchHeight && matchBirthMonth && matchRegion
    );
  });
}

// 各ブランドごとの最大件数（プレースホルダー用）
function getMaxCountPlaceholder(brand) {
  const count = filteredByOtherCriteria().filter(
    (idol) => idol.brand === brand
  ).length;
  return count ? count : 0;
}

async function shuffleArrayWithProgress(array) {
  const arr = array.slice();
  const n = arr.length;
  isShuffling.value = true;
  shuffleProgress.value = 0;
  const step = Math.max(1, Math.floor(n / 100));

  // Fisher–Yates シャッフル＋進捗更新
  for (let i = 0; i < n - 1; i++) {
    const j = i + Math.floor(Math.random() * (n - i));
    [arr[i], arr[j]] = [arr[j], arr[i]];
    if (i % step === 0) {
      shuffleProgress.value = Math.min(100, Math.floor((i / (n - 1)) * 100));
      await nextTick();
    }
  }

  // 最後に 100% 更新
  shuffleProgress.value = 100;
  await nextTick();

  // ★ここで2秒待機してからバーを隠す★
  await new Promise((resolve) => setTimeout(resolve, 2000));
  isShuffling.value = false;

  return arr;
}

// フィルタ実行
async function filterIdols() {
  isFiltering.value = true;
  const temp = filteredByOtherCriteria();
  let result = [];
  for (const brand in brandOptions.value) {
    if (brandOptions.value[brand].enabled) {
      let group = temp.filter((idol) => idol.brand === brand);
      group.sort(() => Math.random() - 0.5);
      const count = brandOptions.value[brand].count;
      if (count && count < group.length) {
        group = group.slice(0, count);
      }
      result = result.concat(group);
    }
  }
  randomIdols.value = await shuffleArrayWithProgress(result);
  isFiltering.value = false;
}

function playCurrentManual() {
  if (currentIndex.value < randomIdols.value.length) {
    const idol = randomIdols.value[currentIndex.value];
    const audioPath = `${process.env.BASE_URL}audio/${idol.brand}/${selectedAudioType.value}/${idol.code}_${selectedAudioType.value}.wav`;
    audioPlayer.value.src = audioPath;
    audioPlayer.value.play().catch((err) => console.error(err));
  }
}

function playPrevious() {
  if (currentIndex.value > 0) {
    currentIndex.value--;
    playCurrentManual();
  }
}

function replayCurrent() {
  playCurrentManual();
}

function playNext() {
  if (currentIndex.value < randomIdols.value.length - 1) {
    currentIndex.value++;
    playCurrentManual();
  }
}

// 定期再生モード
function startPeriodicPlayback() {
  if (randomIdols.value.length === 0) return;
  isPeriodicPlaying.value = true;

  playCurrentManual();
  periodicTimer = setInterval(() => {
    if (currentIndex.value < randomIdols.value.length - 1) {
      currentIndex.value++;
      playCurrentManual();
    } else {
      stopPeriodicPlayback();
    }
  }, periodicIntervalSec.value * 1000);
}

function stopPeriodicPlayback() {
  if (periodicTimer) {
    clearInterval(periodicTimer);
    periodicTimer = null;
  }
  isPeriodicPlaying.value = false;
}
</script>

<style scoped>
:root {
  --primary-color: #4caf50;
  --secondary-color: #f9f9f9;
  --accent-color: #ffc107;
  --font-family: "Helvetica Neue", Arial, sans-serif;
}

.container {
  max-width: 800px;
  margin: 20px auto;
  padding: 20px;
  background-color: var(--secondary-color);
  border-radius: 8px;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.1);
}

h1 {
  font-family: var(--font-family);
  color: var(--primary-color);
  text-align: center;
  margin-bottom: 20px;
}

h2 {
  font-family: var(--font-family);
  color: var(--primary-color);
  margin-bottom: 10px;
}

.filter-group {
  margin-bottom: 20px;
  padding: 10px;
  border: 1px solid #ddd;
  border-radius: 4px;
  background: #fff;
}

.filter-row {
  margin-bottom: 10px;
  display: flex;
  align-items: center;
}

.filter-row label {
  width: 120px;
  font-weight: bold;
}

input[type="number"],
select,
input[type="text"] {
  padding: 5px;
  border: 1px solid #ccc;
  border-radius: 4px;
  margin-right: 10px;
  flex-grow: 1;
}

.action-btn {
  background-color: #8dbbff;
  color: #fff;
  border: none;
  padding: 8px 16px;
  border-radius: 4px;
  cursor: pointer;
  font-family: var(--font-family);
  transition: background-color 0.3s;
  margin-right: 10px;
}

.action-btn:hover {
  background-color: #45a049;
}

.control-group {
  margin-top: 10px;
}

.control-btn {
  background-color: #8dbbff;
  color: #fff;
  border: none;
  padding: 6px 12px;
  border-radius: 4px;
  cursor: pointer;
  font-family: var(--font-family);
  transition: background-color 0.3s;
  margin-right: 8px;
}

.control-btn:hover {
  background-color: #e0a800;
}

.playback-group {
  margin-top: 20px;
  padding: 10px;
  border: 1px solid #ddd;
  border-radius: 4px;
  background: #fff;
}

audio {
  display: block;
  margin-top: 20px;
  width: 100%;
}

.playback-count {
  text-align: center;
  margin: 8px 0;
  font-weight: bold;
}

.progress-bar {
  margin: 10px 0;
}
</style>
