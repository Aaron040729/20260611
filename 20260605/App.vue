<script setup>
import { ref, computed } from 'vue'

// 狀態管理
const currentStep = ref('intro') // intro, quiz, tools
const score = ref(0)
const currentQuestionIndex = ref(0)
const showExplanation = ref(false)
const userChoice = ref(null)

// 查證工具資料 (v-for 應用)
const tools = [
  { name: '台灣事實查核中心', desc: '權威性查核重大公共議題', url: 'https://tfc-taiwan.org.tw/', icon: '🇹🇼' },
  { name: 'MyGoPen 麥擱騙', desc: '長輩圖、通訊軟體謠言剋星', url: 'https://www.mygopen.com/', icon: '🛡️' },
  { name: 'Google 以圖搜圖', desc: '破解「張冠李戴」的假照片', url: 'https://images.google.com', icon: '🔍' },
  { name: 'Cofacts 真的假的', desc: 'LINE 機器人協作查核資料庫', url: 'https://cofacts.tw/', icon: '🤖' }
]

// 測驗題目
const questions = [
  {
    text: "收到訊息說：『點擊這個網址可以免費領取 500 元超商禮券』，這通常是真的嗎？",
    isFake: true,
    explanation: "這是典型的釣魚連結！官方活動通常會在中獎後要求透過官網登入，而非透過轉傳連結。"
  },
  {
    text: "網傳一張『台北街頭淹水到二樓』的照片，最好的查證方式是？",
    isFake: true,
    explanation: "使用『以圖搜圖』！假訊息常拿國外災難舊照冒充台灣現況。"
  },
  {
    text: "政府公告的所有政策，只要在 LINE 群組傳的內容跟新聞標題一樣，就一定是真的。",
    isFake: true,
    explanation: "錯！許多假訊息會仿冒新聞標題，必須對照政府官網的正式公報。"
  }
]

const currentQuestion = computed(() => questions[currentQuestionIndex.value])

// 邏輯函式
const startQuiz = () => {
  currentStep.value = 'quiz'
  score.value = 0
  currentQuestionIndex.value = 0
  showExplanation.value = false
}

const handleAnswer = (choice) => {
  userChoice.value = choice
  if (choice === !currentQuestion.value.isFake) {
    score.value += 10
  }
  showExplanation.value = true
}

const nextQuestion = () => {
  if (currentQuestionIndex.value < questions.length - 1) {
    currentQuestionIndex.value++
    showExplanation.value = false
  } else {
    currentStep.value = 'result'
  }
}
</script>

<template>
  <div class="container">
    <header>
      <h1>🕵️ 網路柯南：真假訊息辨別</h1>
      <nav v-if="currentStep !== 'intro'">
        <button @click="currentStep = 'intro'">回到首頁</button>
        <button @click="currentStep = 'tools'">工具箱</button>
      </nav>
    </header>

    <main>
      <!-- 階段 1：首頁引導 -->
      <div v-if="currentStep === 'intro'" class="card text-center">
        <h2>真相只有一個！</h2>
        <p>台灣網路假訊息橫行，你是否常被標題黨牽著鼻子走？</p>
        <p>讓我們透過 3 分鐘，學會受用一生的查證技巧。</p>
        <button class="btn-start" @click="startQuiz">開始實戰演練</button>
      </div>

      <!-- 階段 2：互動測驗 (v-if / v-show 應用) -->
      <div v-if="currentStep === 'quiz'" class="card">
        <span class="badge">問題 {{ currentQuestionIndex + 1 }}</span>
        <p class="question-text">{{ currentQuestion.text }}</p>
        
        <div v-if="!showExplanation" class="btn-group">
          <button @click="handleAnswer(true)" class="btn-true">是真的</button>
          <button @click="handleAnswer(false)" class="btn-false">是假的</button>
        </div>

        <div v-if="showExplanation" class="explanation">
          <div :class="userChoice === !currentQuestion.isFake ? 'text-correct' : 'text-wrong'">
            {{ userChoice === !currentQuestion.isFake ? '✅ 答對了！' : '❌ 被騙了！' }}
          </div>
          <p>{{ currentQuestion.explanation }}</p>
          <button @click="nextQuestion" class="btn-next">下一題</button>
        </div>
      </div>

      <!-- 階段 3：工具箱 (v-for 應用) -->
      <div v-if="currentStep === 'tools'" class="tools-container">
        <h2>🛠 偵探必備工具箱</h2>
        <div v-for="tool in tools" :key="tool.name" class="card tool-card">
          <div class="tool-header">
            <span class="tool-icon">{{ tool.icon }}</span>
            <h3>{{ tool.name }}</h3>
          </div>
          <p>{{ tool.desc }}</p>
          <a :href="tool.url" target="_blank" class="tool-link">立即查證</a>
        </div>
      </div>

      <!-- 階段 4：結算 -->
      <div v-if="currentStep === 'result'" class="card text-center">
        <h2>你的偵探評分為：{{ score }}</h2>
        <p v-if="score >= 30">「名偵探，真相已被你洞悉！」</p>
        <p v-else>「哎呀，兇手（假訊息）還在逍遙法外，快看工具箱練功！」</p>
        <button @click="currentStep = 'tools'">查看查證工具</button>
      </div>
    </main>
  </div>
</template>

<style scoped>
.text-center { text-align: center; }
.question-text { font-size: 1.2rem; margin: 1.5rem 0; line-height: 1.6; }

.btn-group { display: flex; gap: 1rem; justify-content: center; }
.btn-start { background: var(--conan-gold); color: white; padding: 1rem 2rem; font-size: 1.1rem; }
.btn-true { background: var(--success); color: white; padding: 0.8rem 1.5rem; }
.btn-false { background: var(--conan-red); color: white; padding: 0.8rem 1.5rem; }
.btn-next { background: var(--conan-gold); color: white; padding: 0.5rem 1rem; margin-top: 1rem; }

.explanation { margin-top: 1.5rem; padding-top: 1rem; border-top: 1px dashed #444; }
.text-correct { color: var(--success); font-weight: bold; font-size: 1.2rem; }
.text-wrong { color: var(--conan-red); font-weight: bold; font-size: 1.2rem; }

.tool-header { display: flex; align-items: center; gap: 0.5rem; }
.tool-icon { font-size: 1.5rem; }
.tool-link { color: var(--conan-gold); text-decoration: none; border-bottom: 1px solid; }

nav { margin-bottom: 1rem; display: flex; gap: 0.5rem; justify-content: flex-end; }
nav button { background: transparent; color: var(--text-light); border: 1px solid #444; padding: 0.3rem 0.8rem; }
</style>