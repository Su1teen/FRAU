<script setup lang="ts">
import { computed, reactive, ref } from 'vue'
import * as XLSX from 'xlsx'

type StepKey = 'layout' | 'milling' | 'countertop' | 'colors' | 'handles'
type Submission = {
  timestamp: string
  layoutType: string
  roomHeight: string
  roomLength: string
  roomWidth: string
  milling: string
  countertop: string
  color: string
  handle: string
  lighting: string
}

const stepOrder: StepKey[] = ['layout', 'milling', 'countertop', 'colors', 'handles']

const steps: Record<StepKey, { title: string; caption: string }> = {
  layout: {
    title: 'Формат кухни',
    caption: 'Выберите геометрию и укажите габариты помещения',
  },
  milling: {
    title: 'Фрезеровка фасадов',
    caption: 'Один стиль — чистый, аккуратный, без визуального шума',
  },
  countertop: {
    title: 'Столешница',
    caption: 'Материал и настроение рабочей поверхности',
  },
  colors: {
    title: 'Палитра',
    caption: '32 оттенка, собранные под современную кухню',
  },
  handles: {
    title: 'Фурнитура',
    caption: 'Ручки и подсветка — финальные акценты',
  },
}

const state = reactive({
  layoutType: '' as 'straight' | 'corner' | '',
  roomHeight: '',
  roomLength: '',
  roomWidth: '',
  milling: '',
  countertop: '',
  color: '',
  handle: '',
  lighting: null as null | boolean,
})

const STORAGE_KEY = 'frau-submissions'

const millingOptions = [
  {
    id: 'vybor',
    title: 'Выборка',
    note: 'Мягкие канты, акцент на объём',
    accent: '#8ec5fc',
    pattern: 'linear-gradient(135deg, rgba(255,255,255,.24), transparent)',
  },
  {
    id: 'quadro',
    title: 'Квадро',
    note: 'Геометрия с тонкой рамкой',
    accent: '#ffb6c1',
    pattern: 'repeating-linear-gradient(90deg, rgba(255,255,255,.35) 0 8px, transparent 8px 16px)',
  },
  {
    id: 'line',
    title: 'Линейная',
    note: 'Чистые полосы, минимум декора',
    accent: '#b1f5d2',
    pattern: 'repeating-linear-gradient(0deg, rgba(255,255,255,.35) 0 6px, transparent 6px 12px)',
  },
  {
    id: 'panel',
    title: 'Панель',
    note: 'Глубокая выборка под классику',
    accent: '#d9c2ff',
    pattern: 'radial-gradient(circle at 30% 30%, rgba(255,255,255,.45), transparent 40%)',
  },
]

const countertopOptions = [
  {
    id: 'stone-sand',
    title: 'Искусственный камень',
    variant: 'Песочница',
    accent: '#f1d3a8',
    texture: 'radial-gradient(circle, rgba(0,0,0,.08) 1px, transparent 1px)',
  },
  {
    id: 'stone-marble',
    title: 'Искусственный камень',
    variant: 'Под мрамор',
    accent: '#f5f5f5',
    texture: 'linear-gradient(135deg, rgba(0,0,0,.08) 0 2px, transparent 2px 8px)',
  },
  {
    id: 'ldsp-wood',
    title: 'ЛДСП',
    variant: 'Под дерево',
    accent: '#d7b48a',
    texture: 'repeating-linear-gradient(90deg, rgba(0,0,0,.08) 0 3px, transparent 3px 12px)',
  },
  {
    id: 'ldsp-marble',
    title: 'ЛДСП',
    variant: 'Под мрамор',
    accent: '#e8e8ef',
    texture: 'radial-gradient(circle at 70% 30%, rgba(0,0,0,.05), transparent 45%)',
  },
  {
    id: 'ldsp-sand',
    title: 'ЛДСП',
    variant: 'Песочница',
    accent: '#e4c9a6',
    texture: 'radial-gradient(circle, rgba(0,0,0,.05) 1px, transparent 3px)',
  },
]

const colorPalette = [
  '#0f0f11', '#1c1f24', '#262a30', '#2f3540', '#3b4150', '#45516a', '#506081', '#5b7099',
  '#6780b2', '#7693c7', '#88a7d5', '#9bb9df', '#b3cbe6', '#d0deec', '#f3f5f8', '#f2e6da',
  '#e5d4c3', '#d4c2ae', '#c3af99', '#b19d85', '#a18d76', '#918068', '#81735b', '#72664f',
  '#635843', '#564c3a', '#f2c7b5', '#f2a999', '#e78d7c', '#d56f60', '#c55349', '#b03638',
]

const handles = [
  { id: 'overhead', title: 'Накладные', note: 'Архитектурная линия по фасаду' },
  { id: 'edge', title: 'Торцевые', note: 'Хватаетесь за тонкий кант' },
  { id: 'gola', title: 'Gola-профиль', note: 'Скрытая выемка, без ручек' },
]

const currentStepIndex = ref(0)

const isFirst = computed(() => currentStepIndex.value === 0)
const isLast = computed(() => currentStepIndex.value === stepOrder.length - 1)
const currentStepKey = computed<StepKey>(() => stepOrder[currentStepIndex.value] ?? 'layout')
const currentStep = computed(() => steps[currentStepKey.value])

const nextStep = () => {
  if (isLast.value) return
  currentStepIndex.value += 1
}

const prevStep = () => {
  if (isFirst.value) return
  currentStepIndex.value -= 1
}

const goToStep = (index: number) => {
  if (index < 0 || index >= stepOrder.length) return
  currentStepIndex.value = index
}

const select = (field: keyof typeof state, value: string | boolean) => {
  ;(state as any)[field] = value
}

const isSelected = (field: keyof typeof state, value: string | boolean) =>
  (state as any)[field] === value

const canExport = computed(() => {
  const hasLayout = state.layoutType !== ''
  const hasGeometry = state.roomHeight && state.roomLength && (state.layoutType === 'corner' ? state.roomWidth : true)
  const hasChoices = state.milling && state.countertop && state.color && state.handle
  const hasLighting = state.lighting !== null
  return Boolean(hasLayout && hasGeometry && hasChoices && hasLighting)
})

const buildSubmission = (): Submission => ({
  timestamp: new Date().toISOString(),
  layoutType: state.layoutType || '-\n',
  roomHeight: state.roomHeight || '-',
  roomLength: state.roomLength || '-',
  roomWidth: state.layoutType === 'corner' ? state.roomWidth || '-' : '-',
  milling: state.milling || '-',
  countertop: state.countertop || '-',
  color: state.color || '-',
  handle: state.handle || '-',
  lighting: state.lighting === null ? '-' : state.lighting ? 'Да' : 'Нет',
})

const loadSubmissions = (): Submission[] => {
  try {
    const raw = localStorage.getItem(STORAGE_KEY)
    if (!raw) return []
    return JSON.parse(raw) as Submission[]
  } catch (e) {
    console.warn('Не удалось прочитать локальные заявки', e)
    return []
  }
}

const persistSubmissions = (items: Submission[]) => {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(items))
}

const exportToExcel = () => {
  const submission = buildSubmission()
  const items = loadSubmissions()
  items.push(submission)
  persistSubmissions(items)

  const worksheet = XLSX.utils.json_to_sheet(items)
  const workbook = XLSX.utils.book_new()
  XLSX.utils.book_append_sheet(workbook, worksheet, 'Заявки')
  XLSX.writeFile(workbook, 'frau-configurations.xlsx')
}

const summaryChips = computed(() => {
  const chips = [] as string[]
  if (state.layoutType === 'corner') chips.push('Угловая')
  if (state.layoutType === 'straight') chips.push('Прямая')
  if (state.roomHeight) chips.push(`Высота ${state.roomHeight} м`)
  if (state.roomLength) chips.push(`Длина ${state.roomLength} м`)
  if (state.roomWidth && state.layoutType === 'corner') chips.push(`Ширина ${state.roomWidth} м`)
  if (state.milling) chips.push(`Фрезеровка: ${state.milling}`)
  if (state.countertop) chips.push(`Столешница: ${state.countertop}`)
  if (state.color) chips.push('Цвет выбран')
  if (state.handle) chips.push(`Ручки: ${state.handle}`)
  if (state.lighting !== null) chips.push(state.lighting ? 'С подсветкой' : 'Без подсветки')
  return chips
})
</script>

<template>
  <div class="page">
    <div class="bg-shell">
      <div class="orb orb-a" />
      <div class="orb orb-b" />
      <div class="orb orb-c" />
    </div>

    <main class="shell">
      <header class="hero">
        <div>
          <p class="eyebrow">FRAU / интерьерное бюро</p>
          <h1>Конфигуратор кухни с умной навигацией</h1>
          <p class="lede">
            Пять шагов без перегруза: выбирайте геометрию, фрезеровку, столешницу, цвет и фурнитуру.
            Данные готовы к отправке в ИИ, когда появится бэкенд.
          </p>
          <div class="chips" v-if="summaryChips.length">
            <span v-for="chip in summaryChips" :key="chip" class="chip">{{ chip }}</span>
          </div>
        </div>
        <div class="media-card">
          <div class="media-tag">public-media-mebel.jpg</div>
          <div class="media" aria-hidden="true" />
          <div class="media-frost">Замените изображение в /public/public-media-mebel.jpg</div>
        </div>
      </header>

      <section class="panel">
        <div class="panel-head">
          <div>
            <p class="eyebrow">Шаг {{ currentStepIndex + 1 }}/{{ stepOrder.length }}</p>
            <h2>{{ currentStep.title }}</h2>
            <p class="muted">{{ currentStep.caption }}</p>
          </div>
          <div class="step-dots" role="tablist" aria-label="Навигация по шагам">
            <button
              v-for="(key, idx) in stepOrder"
              :key="key"
              :class="['dot', { active: idx === currentStepIndex }]"
              :aria-label="`Шаг ${idx + 1}`"
              :aria-selected="idx === currentStepIndex"
              @click="goToStep(idx)"
            >
              {{ idx + 1 }}
            </button>
          </div>
        </div>

        <div class="step-grid" v-if="currentStepKey === 'layout'">
          <div class="card focus" :class="{ selected: isSelected('layoutType', 'straight') }" @click="select('layoutType', 'straight')">
            <div class="pill">Прямая</div>
            <h3>Линейная кухня</h3>
            <p class="muted">Минимум швов, чистый фасад, всё по одной стене.</p>
            <div class="layout-visual linear" />
          </div>
          <div class="card focus" :class="{ selected: isSelected('layoutType', 'corner') }" @click="select('layoutType', 'corner')">
            <div class="pill">Угловая</div>
            <h3>Г-образная кухня</h3>
            <p class="muted">Рабочий треугольник и больше места для хранения.</p>
            <div class="layout-visual corner" />
          </div>
        </div>

        <div class="form-grid" v-if="currentStepKey === 'layout'">
          <label class="field">
            <span>Высота комнаты (м)</span>
            <textarea rows="2" v-model="state.roomHeight" placeholder="Например, 2.8" />
          </label>
          <label class="field">
            <span>Длина стены (м)</span>
            <textarea rows="2" v-model="state.roomLength" placeholder="Например, 4.2" />
          </label>
          <label v-if="state.layoutType === 'corner'" class="field">
            <span>Ширина второй стены (м)</span>
            <textarea rows="2" v-model="state.roomWidth" placeholder="Например, 2.3" />
          </label>
        </div>

        <div class="card-grid" v-else-if="currentStepKey === 'milling'">
          <button
            v-for="item in millingOptions"
            :key="item.id"
            class="card focus milling"
            :class="{ selected: isSelected('milling', item.title) }"
            @click="select('milling', item.title)"
          >
            <div class="card-top" :style="{ background: item.accent }">
              <div class="pattern" :style="{ backgroundImage: item.pattern }" />
            </div>
            <div class="card-body">
              <div class="pill">{{ item.id }}</div>
              <h3>{{ item.title }}</h3>
              <p class="muted">{{ item.note }}</p>
            </div>
          </button>
        </div>

        <div class="card-grid" v-else-if="currentStepKey === 'countertop'">
          <button
            v-for="item in countertopOptions"
            :key="item.id"
            class="card focus countertop"
            :class="{ selected: isSelected('countertop', `${item.title} — ${item.variant}`) }"
            @click="select('countertop', `${item.title} — ${item.variant}`)"
          >
            <div class="card-top" :style="{ background: item.accent }">
              <div class="texture" :style="{ backgroundImage: item.texture }" />
            </div>
            <div class="card-body">
              <div class="pill">{{ item.title }}</div>
              <h3>{{ item.variant }}</h3>
              <p class="muted">Выбор под ваш образ кухни</p>
            </div>
          </button>
        </div>

        <div v-else-if="currentStepKey === 'colors'">
          <div class="palette">
            <button
              v-for="color in colorPalette"
              :key="color"
              class="swatch"
              :style="{ background: color }"
              :class="{ selected: isSelected('color', color) }"
              @click="select('color', color)"
              :aria-label="`Цвет ${color}`"
            />
          </div>
          <p class="muted">32 оттенка: глубокие, пастельные и тёплые деревяшки.</p>
        </div>

        <div v-else-if="currentStepKey === 'handles'" class="handles">
          <div class="card-grid">
            <button
              v-for="item in handles"
              :key="item.id"
              class="card focus"
              :class="{ selected: isSelected('handle', item.title) }"
              @click="select('handle', item.title)"
            >
              <div class="pill">{{ item.title }}</div>
              <h3>{{ item.note }}</h3>
              <p class="muted">Минималистичный профиль и эргономика</p>
            </button>
          </div>
          <div class="lighting">
            <p>Подсветка цоколя и рабочей зоны</p>
            <div class="toggle-row">
              <button
                class="ghost"
                :class="{ active: state.lighting === true }"
                @click="select('lighting', true)"
              >
                Да, нужна атмосфера
              </button>
              <button
                class="ghost"
                :class="{ active: state.lighting === false }"
                @click="select('lighting', false)"
              >
                Нет, оставим базу
              </button>
            </div>
          </div>
        </div>

        <footer class="footer-nav">
          <button class="ghost" :disabled="isFirst" @click="prevStep">Назад</button>
          <div class="step-dots" role="tablist" aria-label="Навигация по шагам">
            <button
              v-for="(key, idx) in stepOrder"
              :key="key + '-footer'"
              :class="['dot', { active: idx === currentStepIndex }]"
              :aria-label="`Шаг ${idx + 1}`"
              :aria-selected="idx === currentStepIndex"
              @click="goToStep(idx)"
            >
              {{ idx + 1 }}
            </button>
          </div>
          <button
            v-if="!isLast"
            class="primary"
            :disabled="isLast"
            @click="nextStep"
          >
            Далее
          </button>
          <button
            v-else
            class="primary"
            :disabled="!canExport"
            @click="exportToExcel"
          >
            Сохранить в Excel
          </button>
        </footer>
      </section>
    </main>
  </div>
</template>
