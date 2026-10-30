<script setup>
import { ref, computed, onMounted, watch } from 'vue'

/* =====================================================
   DATA
===================================================== */

const STORAGE_KEY = 'taskify-tasks'

const tasks = ref([])
const newTask = ref('')

/* =====================================================
   LOAD DATA
===================================================== */

onMounted(() => {
  const saved = localStorage.getItem(STORAGE_KEY)

  if (saved) {
    try {
      const parsed = JSON.parse(saved)
      tasks.value = Array.isArray(parsed)
        ? parsed.map(task => ({ ...task, completed: Boolean(task.completed), completedAt: task.completedAt || null }))
        : []
    } catch {
      tasks.value = []
    }
  }
})

/* =====================================================
   SAVE DATA
===================================================== */

watch(
  tasks,
  (value) => {
    localStorage.setItem(
      STORAGE_KEY,
      JSON.stringify(value)
    )
  },
  {
    deep: true
  }
)

/* =====================================================
   ADD TASK
===================================================== */

function addTask() {
  const title = newTask.value.trim()

  if (!title) return

  tasks.value.unshift({
    id:
      Date.now() +
      Math.random(),

    title,

    completed: false,
    createdAt: new Date().toISOString(),
    completedAt: null
  })

  newTask.value = ''
}

/* =====================================================
   TASK COMPLETION AND DELETE
===================================================== */

function updateCompletionTime(task) {
  task.completedAt = task.completed ? new Date().toISOString() : null
}

function deleteTask(id) {
  const task = tasks.value.find(item => item.id === id)
  if (!task) return
  if (!window.confirm(`Yakin ingin menghapus tugas "${task.title}"?`)) return
  tasks.value = tasks.value.filter(item => item.id !== id)
}

/* =====================================================
   CLEAR ALL
===================================================== */

function clearAll() {
  if (!tasks.value.length) return

  const confirmDelete =
    window.confirm(
      'Yakin Ingin Menghapus Tugas?'
    )

  if (confirmDelete) {
    tasks.value = []
  }
}

/* =====================================================
   STATISTICS
===================================================== */

const totalTasks = computed(() => {
  return tasks.value.length
})

const completedTasks = computed(() =>
  tasks.value.reduce((count, task) => count + (task.completed === true ? 1 : 0), 0)
)

const pendingTasks = computed(() => {
  return (
    totalTasks.value -
    completedTasks.value
  )
})

const progress = computed(() => {
  if (!totalTasks.value) {
    return 0
  }

  return Math.round(
    (
      completedTasks.value /
      totalTasks.value
    ) * 100
  )
})

/* =====================================================
   PROGRESS CIRCLE
===================================================== */

const circleRadius = 43

const circleLength =
  2 * Math.PI * circleRadius

const progressOffset = computed(() => {
  return (
    circleLength -
    (
      progress.value /
      100
    ) *
      circleLength
  )
})

/* =====================================================
   WEEKLY CHART
===================================================== */

const dayNames = ['Min', 'Sen', 'Sel', 'Rab', 'Kam', 'Jum', 'Sab']

const weeklyData = computed(() => {
  const today = new Date()
  const start = new Date(today)
  start.setDate(today.getDate() - 6)
  start.setHours(0, 0, 0, 0)

  return Array.from({ length: 7 }, (_, index) => {
    const date = new Date(start)
    date.setDate(start.getDate() + index)
    const nextDate = new Date(date)
    nextDate.setDate(date.getDate() + 1)
    const completedCount = tasks.value.filter(task => {
      if (!task.completedAt) return false
      const completedDate = new Date(task.completedAt)
      return completedDate >= date && completedDate < nextDate
    }).length
    return { day: dayNames[date.getDay()], value: totalTasks.value ? Math.min(100, Math.round(completedCount / totalTasks.value * 100)) : 0 }
  })
})

/* =====================================================
   CHART
===================================================== */

const chartWidth = 520
const chartHeight = 170

function chartX(index) {
  return (
    index *
    (chartWidth / 6)
  )
}

function chartY(value) {
  return (
    chartHeight -
    (
      value /
      100
    ) *
      chartHeight
  )
}

const chartPoints = computed(() => {
  return weeklyData.value
    .map(
      (item, index) =>
        `${chartX(index)},${chartY(item.value)}`
    )
    .join(' ')
})

const chartArea = computed(() => {
  const points =
    weeklyData.value
      .map(
        (item, index) =>
          `${chartX(index)},${chartY(item.value)}`
      )
      .join(' ')

  return `
    0,${chartHeight}
    ${points}
    ${chartWidth},${chartHeight}
  `
})
</script>

<template>

  <div class="app">

    <!-- =========================================
         BACKGROUND
    ========================================== -->

    <div class="background">

      <div class="glow glow-one"></div>

      <div class="glow glow-two"></div>

      <div class="glow glow-three"></div>

      <div class="stars"></div>

    </div>

    <!-- =========================================
         MAIN CONTAINER
    ========================================== -->

    <main class="container">

      <!-- =======================================
           HEADER
      ======================================== -->

      <header class="header">

        <div class="logo">

          <svg
            viewBox="0 0 24 24"
            fill="none"
            stroke="currentColor"
            stroke-width="2"
            stroke-linecap="round"
            stroke-linejoin="round"
          >

            <rect
              x="3"
              y="5"
              width="18"
              height="16"
              rx="3"
            />

            <path
              d="M9 5V3h6v2"
            />

            <path
              d="M8 13l3 3 5-6"
            />

          </svg>

        </div>

        <h1>
          TASKIFY VUE
        </h1>

        <p>
          Kelola Tugasmu, Capai Tujuanmu ♡
        </p>

      </header>

      <!-- =======================================
           ADD TASK
      ======================================== -->

      <form
        class="add-task"
        @submit.prevent="addTask"
      >

        <div class="input-wrapper">

          <span class="input-icon">
            ♡
          </span>

          <input
            v-model="newTask"
            type="text"
            placeholder="Masukkan Tugas Baru"
            autocomplete="off"
          />

        </div>

        <button
          type="submit"
          class="add-button"
        >

          <span>
            Tambah
          </span>

          <strong>
            +
          </strong>

        </button>

      </form>

      <!-- =======================================
           TASK LIST
      ======================================== -->

      <TransitionGroup
        name="task"
        tag="section"
        class="task-list"
      >

        <article
          v-for="task in tasks"
          :key="task.id"
          class="task-item"
        >

          <label class="task-content">

            <input
              v-model="task.completed"
              @change="updateCompletionTime(task)"
              type="checkbox"
            />

            <span class="custom-checkbox">

              <span>
                ✓
              </span>

            </span>

            <span
              class="task-title"
              :class="{
                completed:
                  task.completed
              }"
            >
              {{ task.title }}
            </span>

          </label>

          <button
            class="delete-button"
            type="button"
            :aria-label="`Hapus tugas ${task.title}`"
            @click="deleteTask(task.id)"
          >

            <span class="trash">
              🗑
            </span>

            <span>
              Hapus
            </span>

          </button>

        </article>

      </TransitionGroup>

      <!-- =======================================
           EMPTY STATE
      ======================================== -->

      <Transition name="fade">

        <div
          v-if="tasks.length === 0"
          class="empty"
        >

          <div class="empty-icon">
            ♡
          </div>

          <h3>
            Belum Ada Tugas
          </h3>

          <p>
            Yuk, Tambahkan Tugas Pertamamu!
          </p>

        </div>

      </Transition>

      <!-- =======================================
           STATISTICS
      ======================================== -->

      <section class="statistics">

        <!-- STATISTICS HEADER -->

        <div class="statistics-header">

          <div class="statistics-heading">

            <div class="statistics-icon">

              <span></span>
              <span></span>
              <span></span>

            </div>

            <div>

              <h2>
                Statistik
              </h2>

              <p>
                Ringkasan Progres Tugasmu Hari Ini
              </p>

            </div>

          </div>

          <button
            v-if="tasks.length"
            class="clear-button"
            type="button"
            @click="clearAll"
          >
            Bersihkan
          </button>

        </div>

        <!-- =====================================
             STATISTICS CONTENT
        ====================================== -->

        <div class="statistics-content">

          <!-- ===================================
               STAT CARDS
          ==================================== -->

          <div class="stat-grid">

            <!-- TOTAL -->

            <div
              class="stat-card stat-total"
            >

              <div class="stat-card-top">

                <div class="stat-symbol">
                  ▣
                </div>

                <span>
                  Total Tugas
                </span>

              </div>

              <strong class="stat-number">
                {{ totalTasks }}
              </strong>

              <div class="stat-line"></div>

            </div>

            <!-- COMPLETED -->

            <div
              class="stat-card stat-completed"
            >

              <div class="stat-card-top">

                <div class="stat-symbol">
                  ✓
                </div>

                <span>
                  Selesai
                </span>

              </div>

              <strong class="stat-number">
                {{ completedTasks }}
              </strong>

              <div class="stat-line"></div>

            </div>

            <!-- PENDING -->

            <div
              class="stat-card stat-pending"
            >

              <div class="stat-card-top">

                <div class="stat-symbol">
                  ◷
                </div>

                <span>
                  Belum Selesai
                </span>

              </div>

              <strong class="stat-number">
                {{ pendingTasks }}
              </strong>

              <div class="stat-line"></div>

            </div>

            <!-- PROGRESS -->

            <div
              class="stat-card stat-progress"
            >

              <div class="stat-card-top">

                <div class="stat-symbol">
                  %
                </div>

                <span>
                  Progress
                </span>

              </div>

              <strong class="stat-number">
                {{ progress }}%
              </strong>

              <div class="stat-line"></div>

            </div>

          </div>

          <!-- ===================================
               CHART
          ==================================== -->

          <div class="chart-card">

            <div class="chart-header">

              <div class="chart-icon">
                ↗
              </div>

              <div>

                <h3>
                  Tren Penyelesaian Tugas
                </h3>

                <p>
                  7 Hari Terakhir
                </p>

              </div>

            </div>

            <div class="chart-container">

              <div class="chart-y-axis">

                <span>
                  100%
                </span>

                <span>
                  75%
                </span>

                <span>
                  50%
                </span>

                <span>
                  25%
                </span>

                <span>
                  0%
                </span>

              </div>

              <div class="chart-area">

                <div class="chart-grid">

                  <span></span>
                  <span></span>
                  <span></span>
                  <span></span>
                  <span></span>

                </div>

                <svg
                  class="chart-svg"
                  :key="chartPoints"
                  viewBox="0 0 520 170"
                  preserveAspectRatio="none"
                >

                  <defs>

                    <linearGradient
                      id="chartGradient"
                      x1="0"
                      y1="0"
                      x2="0"
                      y2="1"
                    >

                      <stop
                        offset="0%"
                        stop-color="#bd68ff"
                        stop-opacity=".4"
                      />

                      <stop
                        offset="100%"
                        stop-color="#bd68ff"
                        stop-opacity="0"
                      />

                    </linearGradient>

                  </defs>

                  <polygon
                    :points="chartArea"
                    class="chart-fill"
                  />

                  <polyline
                    :points="chartPoints"
                    class="chart-line"
                  />

                  <circle
                    v-for="(item, index) in weeklyData"
                    :key="index"
                    class="chart-dot"
                    :cx="chartX(index)"
                    :cy="chartY(item.value)"
                    r="5"
                  />

                </svg>

                <div class="chart-days">

                  <span
                    v-for="item in weeklyData"
                    :key="item.day"
                  >
                    {{ item.day }}
                  </span>

                </div>

              </div>

            </div>

          </div>

        </div>

        <!-- =====================================
             BIG PROGRESS
        ====================================== -->

        <div class="progress-summary">

          <div class="progress-summary-text">

            <span>
              Progress keseluruhan ({{ completedTasks }} dari {{ totalTasks }} tugas selesai)
            </span>

            <strong>
              {{ progress }}%
            </strong>

          </div>

          <div class="progress-bar">

            <div
              class="progress-value"
              :style="{ width: `${progress}%` }"
              role="progressbar"
              :aria-valuenow="progress"
              aria-valuemin="0"
              aria-valuemax="100"
              :aria-label="`Progress keseluruhan: ${completedTasks} dari ${totalTasks} tugas selesai`"
            ></div>

          </div>

        </div>

      </section>

    </main>

  </div>

</template>

<style scoped>
*{box-sizing:border-box;}
:global(html){min-height:100%;}
:global(body){margin:0;min-height:100vh;font-family:Inter,Poppins,"Segoe UI",sans-serif;background:#f7f2fb;}
:global(button),:global(input){font-family:inherit;}
.app{position:relative;min-height:100vh;padding:45px 25px;overflow:hidden;background:radial-gradient( circle at 10% 90%,rgba(107,56,255,.32),transparent 30% ),radial-gradient( circle at 90% 10%,rgba(148,72,255,.3),transparent 32% ),linear-gradient( 135deg,#f8f5fc 0%,#eee6f7 45%,#e4d7f1 100% );}
.background{position:fixed;inset:0;pointer-events:none;overflow:hidden;}
.glow{position:absolute;border-radius:50%;filter:blur(100px);animation:glowAnimation 8s ease-in-out infinite;}
.glow-one{width:400px;height:400px;top:-200px;right:-100px;background:rgba(156,78,255,.3);}
.glow-two{width:500px;height:500px;left:-250px;bottom:-250px;background:rgba(89,67,255,.25);animation-delay:-4s;}
.glow-three{width:280px;height:280px;top:40%;left:45%;background:rgba(182,76,255,.08);animation-delay:-2s;}
@keyframes glowAnimation{0%,100%{transform:scale(1) translate(0,0);}
50%{transform:scale(1.15) translate( 15px,-15px );}
}
.stars{position:absolute;inset:0;opacity:.3;background-image:radial-gradient( white 1px,transparent 1px );background-size:80px 80px;animation:starsMove 20s linear infinite;}
@keyframes starsMove{from{background-position:0 0;}
to{background-position:80px 80px;}
}
.container{position:relative;z-index:2;width:100%;max-width:1280px;margin:0 auto;padding:42px;border-radius:35px;background:rgba(28,27,70,.7);border:1px solid rgba(192,155,255,.28);backdrop-filter:blur(25px);-webkit-backdrop-filter:blur(25px);box-shadow:0 35px 100px rgba(0,0,0,.5),inset 0 0 40px rgba(205,170,255,.04);animation:containerEnter .9s cubic-bezier( .16,1,.3,1 );}
@keyframes containerEnter{from{opacity:0;transform:translateY(35px) scale(.96);}
to{opacity:1;transform:translateY(0) scale(1);}
}
.header{display:flex;flex-direction:column;align-items:center;text-align:center;margin-bottom:35px;}
.logo{width:82px;height:82px;display:flex;align-items:center;justify-content:center;margin-bottom:15px;border-radius:24px;color:white;background:linear-gradient( 135deg,#a98bc7,#c7a9d9 );box-shadow:0 0 30px rgba(148,84,255,.55);animation:logoAnimation 3s ease-in-out infinite;}
.logo svg{width:48px;height:48px;}
@keyframes logoAnimation{0%,100%{transform:translateY(0) rotate(0);}
50%{transform:translateY(-7px) rotate(2deg);}
}
.header h1{margin:0;color:#49395c;font-size:clamp( 32px,4vw,46px );font-weight:850;letter-spacing:.5px;text-shadow:0 0 20px rgba(193,145,255,.3);}
.header p{margin:7px 0 0;color:#786987;font-size:16px;}
.add-task{display:flex;gap:15px;margin-bottom:25px;}
.input-wrapper{flex:1;display:flex;align-items:center;min-width:0;padding:0 20px;border-radius:18px;border:1px solid rgba(177,148,255,.3);background:rgba(59,62,126,.25);transition:.3s ease;}
.input-wrapper:focus-within{border-color:#a889ff;box-shadow:0 0 25px rgba(155,112,255,.18);transform:translateY(-2px);}
.input-icon{margin-right:13px;color:#9a82b0;font-size:24px;}
.input-wrapper input{width:100%;padding:19px 0;border:none;outline:none;color:white;background:transparent;font-size:16px;}
.input-wrapper input::placeholder{color:#9586a3;}
.add-button{position:relative;overflow:hidden;min-width:145px;padding:0 25px;border:1px solid rgba(255,255,255,.65);border-radius:18px;color:white;background:linear-gradient( 135deg,#a98bc7,#c7a9d9 );font-size:16px;font-weight:700;cursor:pointer;box-shadow:0 10px 30px rgba(119,69,255,.25);transition:.3s ease;}
.add-button::after{content:'';position:absolute;width:70px;height:160%;top:-30%;left:-120px;background:rgba(255,255,255,.25);transform:rotate(25deg);transition:.6s ease;}
.add-button:hover::after{left:130%;}
.add-button:hover{transform:translateY(-4px);box-shadow:0 15px 40px rgba(146,78,255,.45);}
.add-button strong{margin-left:5px;font-size:22px;}
.task-list{display:flex;flex-direction:column;gap:12px;}
.task-item{display:flex;align-items:center;justify-content:space-between;gap:15px;padding:17px 20px;border-radius:17px;border:1px solid rgba(176,145,255,.17);background:linear-gradient( 110deg,rgba(62,70,139,.35),rgba(89,48,141,.35) );transition:.3s ease;}
.task-item:hover{transform:translateY(-3px);border-color:rgba(190,160,255,.4);box-shadow:0 10px 30px rgba(62,40,120,.25);}
.task-content{display:flex;align-items:center;gap:14px;min-width:0;cursor:pointer;}
.task-content input{position:absolute;opacity:0;}
.custom-checkbox{width:23px;height:23px;flex-shrink:0;display:flex;align-items:center;justify-content:center;border:2px solid rgba(192,164,255,.65);border-radius:7px;transition:.25s ease;}
.custom-checkbox span{opacity:0;color:white;font-size:14px;font-weight:900;transform:scale(.5);transition:.2s ease;}
.task-content input:checked + .custom-checkbox{border-color:#b99bd2;background:linear-gradient( 135deg,#6755ff,#c051ed );box-shadow:0 0 15px rgba(185,82,255,.5);}
.task-content input:checked + .custom-checkbox span{opacity:1;transform:scale(1);}
.task-title{color:#49395c;font-size:16px;overflow-wrap:anywhere;transition:.25s ease;}
.task-title.completed{color:#9a8da6;text-decoration:line-through;}
.delete-button{display:flex;align-items:center;gap:6px;padding:8px 11px;border:none;border-radius:10px;color:#ff80aa;background:transparent;cursor:pointer;transition:.25s ease;}
.delete-button:hover{background:rgba(255,77,132,.1);transform:scale(1.05);}
.task-enter-active,.task-leave-active{transition:.45s cubic-bezier( .16,1,.3,1 );}
.task-enter-from{opacity:0;transform:translateX(-30px) scale(.96);}
.task-leave-to{opacity:0;transform:translateX(30px) scale(.96);}
.task-move{transition:transform .45s ease;}
.empty{margin-top:12px;animation:emptyEnter .45s cubic-bezier(.16,1,.3,1) both;padding:35px 20px;text-align:center;border:1px dashed rgba(187,157,255,.25);border-radius:18px;background:rgba(95,70,155,.08);color:#786987;}
.empty-icon{margin-bottom:5px;color:#a98bc7;font-size:30px;}
.empty h3{margin:0 0 5px;color:#49395c;font-size:clamp(24px,3vw,30px);font-weight:750;}
.empty p{margin:0;font-size:16px;}
@keyframes emptyEnter{from{opacity:0;transform:translateY(12px) scale(.98);}
to{opacity:1;transform:translateY(0) scale(1);}
}
.fade-enter-active,.fade-leave-active{transition:.35s ease;}
.fade-enter-from,.fade-leave-to{opacity:0;transform:translateY(10px);}
.statistics{margin-top:30px;padding:25px;border-radius:24px;border:1px solid rgba(178,148,255,.23);background:rgba(24,24,66,.5);box-shadow:inset 0 0 30px rgba(173,135,255,.03);animation:statisticsEnter .8s ease .3s both;}
@keyframes statisticsEnter{from{opacity:0;transform:translateY(25px);}
to{opacity:1;transform:translateY(0);}
}
.statistics-header{display:flex;align-items:center;justify-content:space-between;gap:20px;margin-bottom:22px;}
.statistics-heading{display:flex;align-items:center;gap:15px;}
.statistics-icon{display:flex;align-items:flex-end;gap:4px;height:35px;}
.statistics-icon span{display:block;width:7px;border-radius:5px;background:#ae7cff;box-shadow:0 0 10px rgba(174,124,255,.6);}
.statistics-icon span:nth-child(1){height:16px;}
.statistics-icon span:nth-child(2){height:25px;}
.statistics-icon span:nth-child(3){height:35px;}
.statistics-heading h2{margin:0;color:#49395c;font-size:28px;font-weight:800;}
.statistics-heading p{margin:3px 0 0;color:#877991;font-size:13px;}
.clear-button{padding:8px 13px;border:1px solid rgba(255,104,151,.25);border-radius:10px;color:#ff91b5;background:rgba(255,80,130,.07);cursor:pointer;transition:.25s ease;}
.clear-button:hover{transform:translateY(-2px);background:rgba(255,80,130,.15);}
.statistics-content{display:grid;grid-template-columns:minmax(0,1.1fr) minmax(420px,1fr);gap:20px;}
.stat-grid{display:grid;grid-template-columns:repeat(4,1fr);gap:12px;}
.stat-card{position:relative;min-height:185px;overflow:hidden;display:flex;flex-direction:column;justify-content:space-between;padding:20px;border:1px solid rgba(191,159,255,.18);border-radius:18px;background:rgba(49,46,105,.5);opacity:0;animation:statEnter .6s cubic-bezier( .16,1,.3,1 ) forwards;transition:.3s ease;}
.stat-card:nth-child(1){animation-delay:.15s;}
.stat-card:nth-child(2){animation-delay:.25s;}
.stat-card:nth-child(3){animation-delay:.35s;}
.stat-card:nth-child(4){animation-delay:.45s;}
@keyframes statEnter{from{opacity:0;transform:translateY(25px) scale(.92);}
to{opacity:1;transform:translateY(0) scale(1);}
}
.stat-card::before{content:'';position:absolute;width:120px;height:120px;right:-60px;top:-60px;border-radius:50%;background:rgba(168,85,247,.15);filter:blur(15px);transition:.5s ease;}
.stat-card:hover{transform:translateY(-7px) scale(1.02);border-color:rgba(204,175,255,.45);box-shadow:0 15px 35px rgba(71,40,130,.3);}
.stat-card:hover::before{transform:scale(1.8);}
.stat-card-top{position:relative;z-index:1;display:flex;flex-direction:column;gap:12px;}
.stat-card-top span{color:#746481;font-size:14px;font-weight:600;}
.stat-symbol{width:48px;height:48px;display:flex;align-items:center;justify-content:center;border-radius:50%;color:#8e73a8;background:rgba(168,85,247,.18);font-size:24px;font-weight:800;}
.stat-number{position:relative;z-index:1;color:#f8f2ff;font-size:clamp( 38px,4vw,48px );line-height:1;font-weight:850;letter-spacing:-1.5px;text-shadow:0 0 18px rgba(206,178,255,.2);}
.stat-line{position:relative;z-index:1;width:55px;height:5px;border-radius:20px;background:#b99bd2;box-shadow:0 0 12px rgba(168,85,247,.65);}
.stat-completed .stat-symbol{color:#5cf0ad;background:rgba(70,235,157,.15);}
.stat-completed .stat-line{background:#4df0a5;}
.stat-pending .stat-symbol{color:#ffd06b;background:rgba(255,190,70,.15);}
.stat-pending .stat-line{background:#ffc04e;}
.stat-progress .stat-symbol{color:#b98bd0;background:rgba(212,83,255,.15);}
.stat-progress .stat-line{background:#d85cff;}
.chart-card{min-width:0;padding:20px;border-radius:19px;border:1px solid rgba(183,151,255,.2);background:rgba(33,31,78,.5);}
.chart-header{display:flex;align-items:center;gap:12px;margin-bottom:15px;}
.chart-icon{color:#c175ff;font-size:31px;text-shadow:0 0 12px rgba(193,117,255,.7);}
.chart-header h3{margin:0;color:#49395c;font-size:17px;}
.chart-header p{margin:2px 0 0;color:#887b95;font-size:12px;}
.chart-container{display:flex;height:190px;gap:8px;}
.chart-y-axis{width:38px;display:flex;flex-direction:column;justify-content:space-between;padding-bottom:25px;color:#897c96;font-size:10px;text-align:right;}
.chart-area{position:relative;flex:1;min-width:0;}
.chart-grid{position:absolute;inset:0 0 25px 0;display:flex;flex-direction:column;justify-content:space-between;}
.chart-grid span{width:100%;border-top:1px dashed rgba(177,155,237,.15);}
.chart-svg{position:absolute;top:0;left:0;width:100%;height:165px;overflow:visible;}
.chart-fill{fill:url(#chartGradient);opacity:0;animation:fillAppear 1s ease 1s forwards;}
@keyframes fillAppear{to{opacity:1;}
}
.chart-line{fill:none;stroke:#b99bd2;stroke-width:3;stroke-linecap:round;stroke-linejoin:round;stroke-dasharray:1000;stroke-dashoffset:1000;filter:drop-shadow( 0 0 6px rgba(193,108,255,.8) );animation:drawChart 1.7s ease .5s forwards;}
@keyframes drawChart{to{stroke-dashoffset:0;}
}
.chart-dot{fill:#dda1ff;stroke:white;stroke-width:1.5;filter:drop-shadow( 0 0 7px rgba(221,161,255,.8) );opacity:0;animation:dotAnimation .3s ease 1.5s forwards;}
@keyframes dotAnimation{from{opacity:0;transform:scale(0);}
to{opacity:1;transform:scale(1);}
}
.chart-days{position:absolute;left:0;right:0;bottom:0;display:flex;justify-content:space-between;color:#897c96;font-size:10px;}
.progress-summary{margin-top:20px;padding-top:20px;border-top:1px solid rgba(180,150,255,.12);}
.progress-summary-text{display:flex;justify-content:space-between;align-items:center;margin-bottom:9px;color:#746481;font-size:13px;}
.progress-summary-text strong{color:#49395c;font-size:18px;}
.progress-bar{width:100%;height:10px;overflow:hidden;border-radius:100px;background:rgba(120, 95, 145, .12);}
.progress-value{height:100%;border-radius:inherit;background:linear-gradient( 90deg,#aa8bc8,#c7a6dc );box-shadow:0 0 15px rgba(194,83,255,.65);transition:width .8s cubic-bezier( .16,1,.3,1 );}
@media (max-width:1100px){.statistics-content{grid-template-columns:1fr;}
.stat-grid{grid-template-columns:repeat(4,1fr);}
}
@media (max-width:760px){.app{padding:20px 12px;}
.container{padding:27px 16px;border-radius:25px;}
.header{margin-bottom:25px;}
.logo{width:70px;height:70px;border-radius:20px;}
.logo svg{width:42px;height:42px;}
.header h1{font-size:30px;}
.header p{font-size:13px;}
.add-task{flex-direction:column;}
.add-button{min-height:55px;}
.task-item{padding:14px;}
.delete-button span:not(.trash){display:none;}
.statistics{padding:17px;}
.statistics-header{align-items:flex-start;}
.statistics-heading h2{font-size:23px;}
.statistics-heading p{font-size:11px;}
.statistics-content{grid-template-columns:1fr;}
.stat-grid{grid-template-columns:repeat(2,1fr);}
.stat-card{min-height:160px;padding:17px;}
.stat-number{font-size:36px;}
.chart-container{height:175px;}
}
@media (max-width:430px){.statistics-header{gap:8px;}
.clear-button{font-size:10px;padding:7px 9px;}
.stat-grid{gap:9px;}
.stat-card{min-height:145px;padding:14px;}
.stat-card-top span{font-size:12px;}
.stat-symbol{width:42px;height:42px;font-size:20px;}
.stat-number{font-size:31px;}
.chart-card{padding:15px;}
}
@media (prefers-reduced-motion:reduce){*,*::before,*::after{animation:none !important;transition:none !important;}
}

/* Soft lilac palette matched to the reference */
:global(body){background:#bdb0d0;}
.app{background:#bdb0d0;}
.background .glow-one{background:rgba(255,255,255,.28);}
.background .glow-two{background:rgba(153,130,174,.18);}
.background .glow-three{background:rgba(255,255,255,.18);}
.stars{opacity:.12;}
.container{background:rgba(249,246,251,.88);border-color:rgba(111,88,130,.22);box-shadow:0 24px 70px rgba(72,54,88,.2),inset 0 0 30px rgba(255,255,255,.45);}
.logo,.add-button{background:linear-gradient(135deg,#927da5,#bba7c9);}
.input-wrapper{background:rgba(255,255,255,.72);border-color:rgba(133,108,153,.3);}
.input-wrapper input,.task-title,.header h1,.statistics-heading h2,.chart-header h3,.stat-number,.progress-summary-text strong{color:#493b55;}
.input-wrapper input::placeholder,.header p,.statistics-heading p,.chart-header p,.chart-y-axis,.chart-days,.stat-card-top span,.progress-summary-text{color:#796b83;}
.task-item{background:linear-gradient(110deg,rgba(227,218,235,.9),rgba(245,240,248,.95));border-color:rgba(133,108,153,.22);}
.task-item:hover{border-color:rgba(133,108,153,.5);}
.custom-checkbox{border-color:#a18bad;}
.task-content input:checked + .custom-checkbox{border-color:#9b83ad;background:linear-gradient(135deg,#927da5,#bba7c9);}
.delete-button{color:#9b6687;}
.empty{background:rgba(255,255,255,.32);border-color:rgba(111,88,130,.28);color:#74657c;}
.empty h3,.empty-icon{color:#655174;}
.statistics,.chart-card{background:rgba(255,255,255,.58);border-color:rgba(111,88,130,.2);}
.stat-card{background:rgba(249,246,251,.82);border-color:rgba(111,88,130,.18);}
.stat-symbol{color:#78628a;background:rgba(189,176,208,.3);}
.stat-line,.stat-total .stat-line,.stat-completed .stat-line,.stat-pending .stat-line,.stat-progress .stat-line{background:#aa94ba;box-shadow:0 0 10px rgba(170,148,186,.35);}
.clear-button{color:#805e86;border-color:rgba(133,108,153,.3);background:rgba(189,176,208,.18);}
.chart-line{stroke:#927da5;}.chart-dot{fill:#b39fc1;}
.progress-bar{background:rgba(116,96,132,.14);}
.progress-value{background:linear-gradient(90deg,#927da5,#c5b4d2);box-shadow:0 0 12px rgba(170,148,186,.38);}
</style>

