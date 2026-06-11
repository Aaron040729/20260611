<script setup>
import { ref, computed, reactive, watch, onMounted, onBeforeUnmount } from 'vue'

// 1. 模式切換狀態
const currentView = ref('home')

// 煉金特效（canvas confetti + 簡單音效），並記錄每頁煉化次數
const confettiCanvas = ref(null)
let ctx = null
let particles = []
let sparkles = []
let floatingTexts = []
let rafId = null

const triggerAlchemy = (ev) => {
  const canvas = confettiCanvas.value
  if (!canvas) return
  const rect = canvas.getBoundingClientRect()
  const origin = {
    x: ((ev && ev.clientX) ? (ev.clientX - rect.left) : rect.width / 2) * devicePixelRatio,
    y: ((ev && ev.clientY) ? (ev.clientY - rect.top) : rect.height / 2) * devicePixelRatio
  }
  createParticles(80, origin)
  createSparkles(40, origin)
  addFloatingText('煉化成功！', origin)
  playSfxBoost()
  if (!rafId) animate()
  // 儲存煉化次數
  try {
    const key = 'alchemy_mastery'
    const saved = JSON.parse(localStorage.getItem(key) || '{}')
    saved[currentView.value] = (saved[currentView.value] || 0) + 1
    localStorage.setItem(key, JSON.stringify(saved))
  } catch (e) {}
}

function rand(min, max) { return Math.random() * (max - min) + min }

function createParticles(n, origin) {
  const w = ctx.canvas.width
  const h = ctx.canvas.height
  for (let i = 0; i < n; i++) {
    const angle = rand(0, Math.PI * 2)
    const speed = rand(2, 9)
    const shape = ['rect','circle','triangle'][Math.floor(Math.random()*3)]
    particles.push({
      x: origin.x + rand(-20,20),
      y: origin.y + rand(-10,10),
      vx: Math.cos(angle) * speed,
      vy: Math.sin(angle) * speed * -1 + rand(-2,2),
      size: rand(4,12),
      color: `hsl(${Math.floor(rand(0,360))},75%,55%)`,
      life: 60 + Math.floor(rand(20,80)),
      rot: rand(0,Math.PI*2),
      drot: rand(-0.2,0.2),
      shape
    })
  }
}

function createSparkles(n, origin) {
  for (let i = 0; i < n; i++) {
    sparkles.push({
      x: origin.x + rand(-30,30),
      y: origin.y + rand(-20,20),
      vx: rand(-1.5,1.5),
      vy: rand(-4,-1),
      size: rand(1,3),
      life: 20 + Math.floor(rand(10,30)),
      alpha: 1
    })
  }
}

function addFloatingText(text, origin) {
  floatingTexts.push({ text, x: origin.x, y: origin.y, life: 60, alpha: 1, vy: -1.2 })
}

function animate() {
  const c = ctx
  if (!c) return
  c.clearRect(0, 0, c.canvas.width, c.canvas.height)
  updateParticles()
  updateSparkles()
  updateFloatingTexts()
  rafId = requestAnimationFrame(animate)
  if (particles.length === 0 && sparkles.length === 0 && floatingTexts.length === 0) {
    cancelAnimationFrame(rafId)
    rafId = null
  }
}

function updateParticles() {
  for (let i = particles.length - 1; i >= 0; i--) {
    const p = particles[i]
    p.vy += 0.18
    p.vx *= 0.995
    p.x += p.vx
    p.y += p.vy
    p.rot += p.drot
    p.life--
    ctx.save()
    ctx.translate(p.x, p.y)
    ctx.rotate(p.rot)
    ctx.fillStyle = p.color
    if (p.shape === 'rect') ctx.fillRect(-p.size/2, -p.size/2, p.size, p.size)
    else if (p.shape === 'circle') {
      ctx.beginPath(); ctx.arc(0,0,p.size/1.2,0,Math.PI*2); ctx.fill()
    } else {
      ctx.beginPath(); ctx.moveTo(0,-p.size); ctx.lineTo(p.size, p.size); ctx.lineTo(-p.size, p.size); ctx.closePath(); ctx.fill()
    }
    ctx.restore()
    if (p.life <= 0 || p.y > ctx.canvas.height + 50) particles.splice(i, 1)
  }
}

function updateSparkles() {
  for (let i = sparkles.length - 1; i >= 0; i--) {
    const s = sparkles[i]
    s.vy += 0.12
    s.x += s.vx
    s.y += s.vy
    s.life--
    s.alpha = s.life / 40
    ctx.fillStyle = `rgba(255,255,255,${Math.max(0, s.alpha)})`
    ctx.beginPath(); ctx.arc(s.x, s.y, s.size, 0, Math.PI*2); ctx.fill()
    if (s.life <= 0) sparkles.splice(i,1)
  }
}

function updateFloatingTexts() {
  ctx.font = '20px system-ui'
  ctx.textAlign = 'center'
  ctx.fillStyle = '#fff'
  for (let i = floatingTexts.length - 1; i >= 0; i--) {
    const t = floatingTexts[i]
    t.y += t.vy
    t.life--
    t.alpha = Math.max(0, t.life / 60)
    ctx.save()
    ctx.globalAlpha = t.alpha
    ctx.shadowColor = 'rgba(0,0,0,0.4)'
    ctx.shadowBlur = 8
    ctx.fillStyle = '#ffea9a'
    ctx.fillText(t.text, t.x, t.y)
    ctx.restore()
    if (t.life <= 0) floatingTexts.splice(i,1)
  }
}

function playSfxBoost() {
  try {
    const audioCtx = new (window.AudioContext || window.webkitAudioContext)()
    const now = audioCtx.currentTime
    const o1 = audioCtx.createOscillator(); const g1 = audioCtx.createGain();
    o1.type = 'triangle'; o1.frequency.value = 880; o1.connect(g1); g1.connect(audioCtx.destination)
    g1.gain.setValueAtTime(0.0001, now); g1.gain.exponentialRampToValueAtTime(0.2, now+0.01)
    o1.frequency.exponentialRampToValueAtTime(330, now+0.3)
    g1.gain.exponentialRampToValueAtTime(0.0001, now+0.6)
    o1.start(now); o1.stop(now+0.65)
    // short click
    const o2 = audioCtx.createOscillator(); const g2 = audioCtx.createGain();
    o2.type = 'square'; o2.frequency.value = 1400; o2.connect(g2); g2.connect(audioCtx.destination)
    g2.gain.setValueAtTime(0.0001, now); g2.gain.exponentialRampToValueAtTime(0.12, now+0.005)
    g2.gain.exponentialRampToValueAtTime(0.0001, now+0.09)
    o2.start(now); o2.stop(now+0.1)
  } catch(e) {}
}

onMounted(() => {
  const canvas = confettiCanvas.value
  if (canvas) {
    ctx = canvas.getContext('2d')
    const resize = () => {
      const rect = canvas.getBoundingClientRect()
      canvas.width = rect.width * devicePixelRatio
      canvas.height = rect.height * devicePixelRatio
      if (ctx) ctx.setTransform(devicePixelRatio, 0, 0, devicePixelRatio, 0, 0)
    }
    window.addEventListener('resize', resize)
    resize()
  }
})

onBeforeUnmount(() => {
  if (rafId) cancelAnimationFrame(rafId)
})

// 頁面配圖數據
const pageImages = {
  home: 'https://shop.8way.com.tw/images/ProSlide/P001/Pork-Dumpling.png',
  boiled: 'https://tokyo-kitchen.icook.network/uploads/recipe/cover/103372/a77590159184a59a.jpg',
  fried: 'https://img.ltn.com.tw/Upload/playing/page/2018/10/14/181014-16623-1-150.jpg',
  sauce: 'https://tokyo-kitchen.icook.network/uploads/recipe/cover/420956/72b63420a04c578c.jpg'
}

// 2. 測驗相關狀態
const currentQuestionIndex = ref(0)
const score = ref(0)
const showResult = ref(false)
const selectedAnswer = ref(null)
const answered = ref(false)

const quizQuestions = [
  {
    q: "完美的冰花液中，麵粉與水的黃金比例是多少？",
    options: ["1 : 5", "1 : 10", "1 : 20"],
    correct: 1,
    desc: "正確答案是 1:10！這是脆皮成形的最佳平衡點，水份過多會太軟，太少則會變成整片麵糊。"
  },
  {
    q: "水煮餃子時，「點水」的主要目的是什麼？",
    options: ["讓水質保持乾淨", "讓內餡熟透且保持皮的彈性", "單純降溫防止燙傷"],
    correct: 1,
    desc: "點水能讓外皮降溫收縮增加彈性，同時利用熱能傳導讓較難熟的肉餡徹底煮熟。"
  },
  {
    q: "煎餃法陣中，哪一個元素是決定「脆度」的關鍵？",
    options: ["麵粉水的比例", "鍋蓋有沒有蓋緊", "餃子的數量"],
    correct: 0,
    desc: "麵粉水的比例最為關鍵！掌握了煉金比例，才能煎出蟬翼般的冰花。"
  }
]

const currentQuestion = computed(() => quizQuestions[currentQuestionIndex.value])

// 3. 沾醬煉金數據
const sauceIngredients = [
  { name: '鎮江香醋', ratio: '3' },
  { name: '薄鹽生抽', ratio: '2' },
  { name: '特製辣油', ratio: '1' },
  { name: '壓碎蒜泥', ratio: '適量' }
]

// 4. 食材數據
const boiledIngredients = [
  { name: '靈魂載體', amount: '冷凍水餃 12-20 顆' },
  { name: '純淨之水', amount: '2000ml (至少淹過餃子)' },
  { name: '結界之鹽', amount: '1 小匙 (增加面皮韌性)' },
  { name: '低溫藥水', amount: '冷水 3 碗 (用於點水)' }
]

// 4. 冰花比例計算機
const dumplingCount = ref(10)
const alchemyResult = computed(() => {
  const count = dumplingCount.value || 0
  return {
    flour: (count * 1.5).toFixed(1),      // 麵粉量 (克)
    water: (count * 15).toFixed(0),      // 水量 (毫升)
    oil: (count * 0.8).toFixed(1),       // 油量 (克)
    ratio: '1 : 10'                      // 黃金粉水比
  }
})

// 5. 教學數據
const boiledSteps = [
  { title: '鼎中佈陣', desc: '取深鍋注入大量清水，加入一小匙鹽。鹽分能改變水的沸點並收緊面皮，這就是「防破結界」。' },
  { title: '珠落玉盤', desc: '水大開後，用湯勺背面順時針攪動水流形成漩渦，再輕放餃子。漩渦能防止餃子直接沉底黏鍋。' },
  { title: '初次點水', desc: '當餃子首度沸騰浮起，倒入一碗冷水。此舉可令麵皮急速收縮增加Ｑ度，並將熱能逼入肉餡。' },
  { title: '二度降溫', desc: '再次水開後，倒入第二碗冷水。持續觀察麵皮，它應呈現半透明的飽滿感。' },
  { title: '末次覺醒', desc: '第三次開水後倒入最後一碗冷水。此時餃子應像充滿氣的小船，不斷翻滾。' },
  { title: '試煉終結', desc: '撈起後不可重疊，需立即瀝乾水分，利用餘溫散發濕氣，方能煉成「爆汁神丸」。' }
]

const friedSteps = [
  { title: '法陣配置', desc: '在不沾平底鍋中心抹上一層薄薄的食用油，由中心向外放射狀整齊排列冷凍餃子（切勿退冰）。' },
  { title: '靈力預熱', desc: '開啟中火乾煎約 1 分鐘，直到餃子底部呈現微微的琥珀色金黃，鎖定香氣基礎。' },
  { title: '煉金藥水', desc: '將計算機得出的「麵粉、水、油」充分搖勻，繞圈倒入鍋中。水位應到餃子的 1/3 高度。' },
  { title: '封印時間', desc: '蓋上鍋蓋鎖住蒸氣。以中火悶煎 7-8 分鐘，利用對流蒸熟肉餡。' },
  { title: '破殼而出', desc: '當水聲從「咕嘟」轉為「吱吱」脆響，表示水份已乾。開蓋轉大火，將剩餘油份逼出，直到冰花形成。' },
  { title: '神蹟顯現', desc: '用盤子覆蓋鍋面，華麗翻轉！如蟬翼般的冰花應在燈光下閃爍著煉金師的光芒。' }
]

// 6. 測驗邏輯函式
const checkAnswer = (index) => {
  if (answered.value) return
  selectedAnswer.value = index
  answered.value = true
  if (index === currentQuestion.value.correct) {
    score.value += 33
  }
}

const nextQuestion = () => {
  if (currentQuestionIndex.value < quizQuestions.length - 1) {
    currentQuestionIndex.value++
    selectedAnswer.value = null
    answered.value = false
  } else {
    showResult.value = true
  }
}

const resetQuiz = () => {
  currentQuestionIndex.value = 0
  score.value = 0
  showResult.value = false
  answered.value = false
  selectedAnswer.value = null
  currentView.value = 'home'
}
</script>

<template>
  <div class="container">
    <header class="text-center" style="margin-bottom: 30px;">
      <h1 class="main-title">🥟 餃子煉金術</h1>
      <p class="subtitle">爆汁水餃與完美冰花的秘密</p>
      
      <!-- 網站常駐導覽按鈕 -->
      <div class="mode-switcher">
        <button @click="currentView = 'home'" :class="['btn-alchemy', { active: currentView === 'home' }]">🏠 首頁</button>
        <button @click="currentView = 'boiled'" :class="['btn-alchemy', { active: currentView === 'boiled' }]">🔥 爆汁水餃</button>
        <button @click="currentView = 'fried'" :class="['btn-alchemy', { active: currentView === 'fried' }]">❄️ 冰花煎餃</button>
        <button @click="currentView = 'sauce'" :class="['btn-alchemy', { active: currentView === 'sauce' }]">🥣 秘製沾醬</button>
        <button @click="currentView = 'quiz'" :class="['btn-alchemy', { active: currentView === 'quiz' }]">🎓 煉金考核</button>
      </div>
    
        <!-- 煉金觸發器（按下會觸發 confetti 與音效） -->
        <div class="alchemy-panel">
          <button class="btn-alchemy btn-trigger" @click="triggerAlchemy($event)">🔮 煉化!</button>
          <canvas ref="confettiCanvas" class="confetti-canvas" aria-hidden="true"></canvas>
        </div>
    </header>

    <main>
      <!-- 1. 首頁 -->
      <div v-if="currentView === 'home'" class="alchemy-card text-center animate-in">
        <img :src="pageImages.home" class="page-img" alt="餃子煉金術">
        <h2 class="card-title">歡迎來到煉金工坊</h2>
        <p>在這裡，我們將平凡的冷凍水餃轉化為究極的美食。</p>
        <div style="margin-top: 20px;">
          <p>請選擇上方的煉金術法開始你的烹飪旅程：</p>
          <ul style="list-style: none; padding: 0; margin-top: 10px;">
            <li>🌊 <b>爆汁水餃</b>：傳承三度點水法，皮Ｑ餡多汁。</li>
            <li>❄️ <b>冰花煎餃</b>：精確比例計算，打造如蟬翼般的脆皮。</li>
            <li>⚖️ <b>考核挑戰</b>：證明你已掌握餃子煉金的奧義。</li>
          </ul>
        </div>
      </div>

      <!-- 2. 冰花煎餃模式 -->
      <div v-if="currentView === 'fried'" class="animate-in">
        <div class="alchemy-card">
        <img :src="pageImages.fried" class="page-img" alt="冰花煎餃">
        <h2 class="card-title">⚖️ 冰花液比例計算機</h2>
        <div class="calculator-box">
          <div class="input-group">
            <label>預計煎幾顆水餃？</label>
            <input type="number" v-model.number="dumplingCount" min="1" max="50">
          </div>
          
          <div class="result-grid">
            <div class="result-item">
              <span class="label">低筋麵粉</span>
              <span class="value">{{ alchemyResult.flour }} g</span>
            </div>
            <div class="result-item">
              <span class="label">冷水</span>
              <span class="value">{{ alchemyResult.water }} ml</span>
            </div>
            <div class="result-item">
              <span class="label">食用油</span>
              <span class="value">{{ alchemyResult.oil }} g</span>
            </div>
          </div>
          <p class="hint">* 秘訣：粉水比維持 <b>{{ alchemyResult.ratio }}</b> 是脆皮成形的關鍵！</p>
        </div>
        </div>

        <!-- 煎餃食材 -->
        <div class="alchemy-card">
          <h2 class="card-title">📦 煉金材料</h2>
          <ul class="ingredient-list">
            <li><span class="ing-name">冷凍水餃</span><span class="ing-amount">{{ dumplingCount }} 顆</span></li>
            <li><span class="ing-name">低筋麵粉</span><span class="ing-amount">{{ alchemyResult.flour }} g</span></li>
            <li><span class="ing-name">冷水</span><span class="ing-amount">{{ alchemyResult.water }} ml</span></li>
            <li><span class="ing-name">食用油</span><span class="ing-amount">{{ alchemyResult.oil }} g</span></li>
          </ul>
        </div>

        <div class="alchemy-card">
          <h2 class="card-title">🍳 煎餃法陣配置</h2>
          <ul class="step-list">
            <li v-for="(step, index) in friedSteps" :key="index">
              <div class="step-num">{{ index + 1 }}</div>
              <div class="step-content">
                <h3>{{ step.title }}</h3>
                <p>{{ step.desc }}</p>
              </div>
            </li>
          </ul>
        </div>
      </div>

      <!-- 3. 爆汁水餃模式 -->
      <div v-if="currentView === 'boiled'" class="animate-in">
        <div class="alchemy-card">
          <img :src="pageImages.boiled" class="page-img" alt="爆汁水餃">
          <h2 class="card-title">📦 所需食材</h2>
          <ul class="ingredient-list">
            <li v-for="ing in boiledIngredients" :key="ing.name">
              <span class="ing-name">{{ ing.name }}</span>
              <span class="ing-amount">{{ ing.amount }}</span>
            </li>
          </ul>
        </div>

        <div class="alchemy-card">
        <h2 class="card-title">✨ 水煮煉金指引</h2>
        <ul class="step-list">
          <li v-for="(step, index) in boiledSteps" :key="index">
            <div class="step-num">{{ index + 1 }}</div>
            <div class="step-content">
              <h3>{{ step.title }}</h3>
              <p>{{ step.desc }}</p>
            </div>
          </li>
        </ul>
        </div>
      </div>

      <!-- 4. 沾醬頁面 -->
      <div v-if="currentView === 'sauce'" class="alchemy-card sauce-section animate-in">
        <img :src="pageImages.sauce" class="page-img" alt="秘製沾醬">
        <h2 class="card-title">酱 沾醬煉金大成</h2>
        <p style="font-size: 0.9rem; margin-bottom: 15px;">完美的餃子需要完美的靈魂伴侶：</p>
        <div class="sauce-grid">
          <div v-for="item in sauceIngredients" :key="item.name" class="sauce-item">
            <span class="sauce-name">{{ item.name }}</span>
            <span class="sauce-ratio">{{ item.ratio }}</span>
          </div>
        </div>
      </div>
      
      <!-- 5. 測驗模式介面 -->
      <div v-if="currentView === 'quiz'" class="quiz-container animate-in">
        <div class="alchemy-card">
        <div v-if="!showResult">
          <div class="quiz-header">
            <span>煉金術考核 - 第 {{ currentQuestionIndex + 1 }} 題</span>
            <button @click="resetQuiz" class="btn-close">✕</button>
          </div>
          
          <h2 class="question-text">{{ currentQuestion.q }}</h2>
          
          <div class="options-list">
            <button 
              v-for="(opt, idx) in currentQuestion.options" 
              :key="idx"
              @click="checkAnswer(idx)"
              :class="['opt-btn', { 
                correct: answered && idx === currentQuestion.correct,
                wrong: answered && selectedAnswer === idx && idx !== currentQuestion.correct
              }]"
            >
              {{ opt }}
            </button>
          </div>

          <div v-if="answered" class="explanation-box animate-in">
            <p class="status">{{ selectedAnswer === currentQuestion.correct ? '✅ 賢者之智慧！' : '❌ 煉金失敗...' }}</p>
            <p class="desc">{{ currentQuestion.desc }}</p>
            <button @click="nextQuestion" class="btn-alchemy">下一題</button>
          </div>
        </div>

        <div v-else class="text-center result-box">
          <h1 class="result-title">考核結束</h1>
          <div class="score-display">
            <span class="score-num">{{ score > 90 ? 100 : score }}</span>
            <span class="score-unit">分</span>
          </div>
          <p class="rank-text">
            {{ score > 90 ? '🏆 首席煉金大師' : score > 60 ? '📜 資深煉金士' : '🌱 初級學徒' }}
          </p>
          <button @click="resetQuiz" class="btn-alchemy" style="margin-top: 20px;">回主頁</button>
        </div>
      </div>
      </div>
    </main>

    <footer class="text-center footer-text" v-if="currentView !== 'quiz'">
      <p>掌握比例，你也能將市售水餃變身餐廳名菜。</p>
    </footer>
  </div>
</template>

<style scoped>
.text-center { text-align: center; }
.main-title { color: var(--alchemy-brown); font-size: 2.5rem; margin-bottom: 0.5rem; }
.subtitle { color: #888; font-style: italic; }

.mode-switcher {
  display: flex;
  justify-content: center;
  gap: 1rem;
  margin-top: 20px;
}

.card-title {
  border-bottom: 2px solid var(--alchemy-cream);
  padding-bottom: 10px;
  margin-bottom: 20px;
}

.calculator-box {
  background: #fff9f0;
  padding: 20px;
  border-radius: 10px;
}

.input-group { margin-bottom: 1.5rem; }
.input-group input {
  padding: 8px;
  font-size: 1.1rem;
  border: 2px solid var(--alchemy-gold);
  border-radius: 5px;
  width: 80px;
  text-align: center;
}

.result-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 10px;
}

.result-item {
  display: flex;
  flex-direction: column;
  align-items: center;
  background: white;
  padding: 10px;
  border-radius: 8px;
  box-shadow: 0 2px 5px rgba(0,0,0,0.05);
}

.result-item .label { font-size: 0.85rem; color: #888; }
.result-item .value { font-weight: bold; color: var(--alchemy-red); font-size: 1.2rem; }
.hint { margin-top: 15px; font-size: 0.9rem; color: #666; }

.step-list { list-style: none; padding: 0; }
.step-list li { display: flex; gap: 15px; margin-bottom: 20px; align-items: flex-start; }
.step-num {
  background: var(--alchemy-gold);
  color: white;
  width: 28px;
  height: 28px;
  border-radius: 50%;
  display: flex;
  justify-content: center;
  align-items: center;
  flex-shrink: 0;
  font-weight: bold;
}
.step-content h3 { margin: 0; font-size: 1.1rem; color: var(--alchemy-brown); }
.step-content p { margin: 5px 0 0; color: #555; font-size: 0.95rem; }
.footer-text { margin-top: 40px; color: #bbb; font-size: 0.8rem; }

/* 測驗樣式 */
.quiz-header { display: flex; justify-content: space-between; color: #888; font-size: 0.8rem; margin-bottom: 20px; }
.btn-close { background: none; border: none; font-size: 1.2rem; cursor: pointer; color: #ccc; }
.question-text { font-size: 1.4rem; color: var(--alchemy-brown); margin-bottom: 30px; }
.options-list { display: flex; flex-direction: column; gap: 10px; }
.opt-btn { 
  padding: 15px; border: 1px solid #ddd; border-radius: 8px; background: white; 
  cursor: pointer; text-align: left; transition: 0.2s; font-size: 1rem;
}
.opt-btn:hover { background: #fdfaf0; border-color: var(--alchemy-gold); }
.opt-btn.correct { background: #d4edda; border-color: #28a745; color: #155724; font-weight: bold; }
.opt-btn.wrong { background: #f8d7da; border-color: #dc3545; color: #721c24; }

.explanation-box { margin-top: 30px; padding: 20px; border-radius: 8px; background: #f8f9fa; border-left: 4px solid var(--alchemy-gold); }
.explanation-box .status { font-weight: bold; margin-bottom: 5px; }
.explanation-box .desc { font-size: 0.9rem; color: #666; margin-bottom: 15px; }

.score-display { margin: 30px 0; }
.score-num { font-size: 5rem; font-weight: bold; color: var(--alchemy-red); }
.rank-text { font-size: 1.5rem; font-weight: bold; color: var(--alchemy-gold); }

.quiz-trigger { border: 2px solid var(--alchemy-red); }
.sauce-grid { display: grid; grid-template-columns: repeat(2, 1fr); gap: 15px; }
.sauce-item { 
  background: var(--alchemy-cream); padding: 10px 15px; border-radius: 6px;
  display: flex; justify-content: space-between; align-items: center;
}
.sauce-name { font-weight: bold; color: var(--alchemy-brown); }

/* 喜歡按鈕樣式 */
.alchemy-panel { display: flex; justify-content: center; gap: 12px; align-items: center; margin-top: 12px; position: relative; }
.btn-trigger { padding: 10px 16px; border-radius: 24px; font-weight: bold; box-shadow: 0 6px 18px rgba(0,0,0,0.08); transition: transform 0.12s ease; }
.btn-trigger:active { transform: scale(0.96); }
.confetti-canvas { position: absolute; left: 0; top: 0; width: 100%; height: 140px; pointer-events: none; }
.sauce-ratio { background: var(--alchemy-red); color: white; padding: 2px 8px; border-radius: 20px; font-size: 0.8rem; }

.animate-in { animation: fadeIn 0.4s ease-out; }
@keyframes fadeIn { from { opacity: 0; transform: translateY(10px); } to { opacity: 1; transform: translateY(0); } }

/* 食材清單樣式 */
.ingredient-list { list-style: none; padding: 0; }
.ingredient-list li { 
  display: flex; justify-content: space-between; padding: 8px 0;
  border-bottom: 1px dashed var(--alchemy-cream);
}
.ing-name { font-weight: bold; }
.ing-amount { color: var(--alchemy-red); }

/* 圖片樣式 */
.page-img {
  width: 100%;
  height: 200px;
  object-fit: cover;
  border-radius: 10px;
  margin-bottom: 20px;
  border: 1px solid var(--alchemy-gold);
}
</style>
