<script setup lang="ts">
/**
 * MCQ demo page — integrates the close-ended activity player hosting a
 * multi-MCQSR item. The host page only uses the Activity interface;
 * it never calls item-player APIs directly.
 *
 * Side-effect imports register the required custom elements before mount.
 */

// Register custom elements on the client only — Lit uses browser-only DOM APIs.
if (import.meta.client) {
  await Promise.all([
    import('@comprodls/w-close-ended'), // activity shell
    import('@comprodls/w-multi-mcqsr'), // item player (side-effect only)
  ])
}

import { ref, onMounted, onBeforeUnmount } from 'vue'
import type { Activity } from '@comprodls/w-player-core'

// ---------------------------------------------------------------------------
// Activity JSON — MCQ item wrapped in a close-ended activity shell.
// ---------------------------------------------------------------------------
const LAUNCH_ID = 'starter-mcq-01'

const itemData = {
  itemId: 'item-solar',
  type: 'mcqsr',
  subType: 'multi',
  status: 'active',
  displayType: 'block',
  toolName: 'starter-app',
  toolVersion: '1.0.0',
  systemMeta: {},
  itemMeta: {},
  itemResources: {},
  itemBody: {
    preferences: {
      score: 3,
      numberingFormat: 'A',
      optionImageAspectRatio: 'automatic',
      layout: {
        orientation: 'top-bottom',
        positionAnswerOptions: 'vertically',
        alignAnswerOptions: 'left',
        positionRadio: 'outside-image',
        mediaPlacement: 'top-bottom',
      },
    },
    content: null,
    prompt: null,
    stimulus: {
      content: { type: 'item-text', data: 'Answer each question about the solar system.' },
    },
    instruction: {
      content: {
        type: 'item-xhtml',
        data: '<p>Select <strong>one</strong> answer for each question.</p>',
      },
    },
    questions: [
      {
        id: 'q1',
        data: {
          stimulus: { content: { type: 'item-text', data: 'Which planet is closest to the Sun?' } },
          media: null,
          options: [
            { id: 'q1-a', content: { type: 'item-text', data: 'Mercury' }, media: null },
            { id: 'q1-b', content: { type: 'item-text', data: 'Venus' }, media: null },
            { id: 'q1-c', content: { type: 'item-text', data: 'Earth' }, media: null },
            { id: 'q1-d', content: { type: 'item-text', data: 'Mars' }, media: null },
          ],
        },
      },
      {
        id: 'q2',
        data: {
          stimulus: { content: { type: 'item-text', data: 'Which planet has the most moons?' } },
          media: null,
          options: [
            { id: 'q2-a', content: { type: 'item-text', data: 'Jupiter' }, media: null },
            { id: 'q2-b', content: { type: 'item-text', data: 'Saturn' }, media: null },
            { id: 'q2-c', content: { type: 'item-text', data: 'Uranus' }, media: null },
            { id: 'q2-d', content: { type: 'item-text', data: 'Neptune' }, media: null },
          ],
        },
      },
    ],
    validations: [
      { validationId: 'val-q1', responseContainer: 'q1', operator: 'id_equals', correctResponse: 'q1-a' },
      { validationId: 'val-q2', responseContainer: 'q2', operator: 'id_equals', correctResponse: 'q2-b' },
    ],
    scoring: {
      rules: [{
        type: 'sum',
        score: 2,
        rules: [
          { type: 'compute', validationId: 'val-q1', score: 1 },
          { type: 'compute', validationId: 'val-q2', score: 1 },
        ],
      }],
    },
    viewLabels: { instructionHeading: 'Directions', selectOne: 'Select one answer per question.' },
    hints: [],
    feedback: null,
  },
}

const activityJson = {
  activityId: LAUNCH_ID,
  itemId: LAUNCH_ID,
  type: 'activity',
  subType: 'close-ended',
  itemMeta: {},
  itemBody: {
    questions: [{ id: 'item-solar', meta: { score: 3, defaultScore: 0 } }],
  },
  itemFragments: {
    'item-solar': { data: itemData },
  },
}

// ---------------------------------------------------------------------------
// Reactive state
// ---------------------------------------------------------------------------
const containerRef = ref<HTMLElement | null>(null)
let player: Activity | null = null
const eventLog = ref<string[]>([])
const response = ref<string | null>(null)

function log(msg: string) {
  eventLog.value.unshift(`[${new Date().toLocaleTimeString()}] ${msg}`)
  if (eventLog.value.length > 20) eventLog.value.pop()
}

onMounted(async () => {
  if (!containerRef.value) return

  player = document.createElement('w-close-ended-activity-player') as unknown as Activity

  await player.init(LAUNCH_ID, activityJson, {
    events: {
      launched:    () => log('launched'),
      ready:       () => log('ready'),
      change:      (d: unknown) => log(`change: ${JSON.stringify(d)}`),
      checkAnswer: (d: unknown) => log(`checkAnswer: ${JSON.stringify(d)}`),
      submit:      (d: unknown) => log(`submit: ${JSON.stringify(d)}`),
      error:       (d: unknown) => log(`⚠ error: ${JSON.stringify(d)}`),
    },
    playerUIConfig: { dimensions: 'content-space' },
  })

  if (!containerRef.value) return
  player.mount(LAUNCH_ID, containerRef.value)
})

onBeforeUnmount(() => {
  player?.destroy()
  player = null
})

// ---------------------------------------------------------------------------
// Controls — Activity interface only, no item-player API
// ---------------------------------------------------------------------------
function checkAnswer() {
  player?.checkAnswer(() => {}, {})
}

function submit() {
  player?.submit(() => {}, {})
}

function getResponse() {
  const r = player?.getResponse()
  response.value = r !== undefined ? JSON.stringify(r) : 'no response'
  log(`getResponse → ${response.value}`)
}

function reset() {
  player?.reset()
  response.value = null
  log('reset')
}
</script>

<template>
  <div class="demo-layout">
    <header class="demo-header">
      <NuxtLink to="/" class="demo-back">← Home</NuxtLink>
      <h1>MCQ Activity Demo</h1>
      <p>Shell: <code>@comprodls/w-close-ended</code> · Item: <code>@comprodls/w-multi-mcqsr</code></p>
    </header>

    <div class="demo-body">
      <!-- ── Activity player frame ── -->
      <section class="demo-player-frame">
        <div ref="containerRef" style="width:100%;height:100%;" />
      </section>

      <!-- ── Controls + event log ── -->
      <aside class="demo-sidebar">
        <div class="demo-controls">
          <button @click="checkAnswer">Check Answer</button>
          <button @click="submit">Submit</button>
          <button @click="getResponse">Get Response</button>
          <button @click="reset">Reset</button>
        </div>

        <div v-if="response" class="demo-score">
          Response: <code>{{ response }}</code>
        </div>

        <div class="demo-log">
          <p class="demo-log-title">Event log</p>
          <ul>
            <li v-for="(entry, i) in eventLog" :key="i">{{ entry }}</li>
          </ul>
        </div>
      </aside>
    </div>
  </div>
</template>

<style scoped>
.demo-layout {
  font-family: system-ui, sans-serif;
  max-width: 1100px;
  margin: 0 auto;
  padding: 1rem;
}
.demo-back { font-size: 0.875rem; color: #555; text-decoration: none; }
.demo-back:hover { text-decoration: underline; }
.demo-header h1 { margin-bottom: 0.25rem; }
.demo-header p  { color: #666; margin-top: 0; }

.demo-body {
  display: flex;
  gap: 1.5rem;
  margin-top: 1.5rem;
}

.demo-player-frame {
  flex: 1;
  min-height: 400px;
  border: 1px solid #ddd;
  border-radius: 8px;
  padding: 1rem;
}

.demo-sidebar {
  width: 260px;
  flex-shrink: 0;
}

.demo-controls {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
  margin-bottom: 1rem;
}
.demo-controls button {
  padding: 0.4rem 0.8rem;
  cursor: pointer;
  border: 1px solid #aaa;
  border-radius: 4px;
  background: #f5f5f5;
}
.demo-controls button:hover { background: #e0e0e0; }

.demo-score {
  margin-bottom: 0.75rem;
  padding: 0.4rem 0.6rem;
  background: #eef9ee;
  border-radius: 4px;
}

.demo-log-title { font-weight: 600; margin-bottom: 0.25rem; }
.demo-log ul {
  list-style: none;
  padding: 0;
  margin: 0;
  font-size: 0.78rem;
  font-family: monospace;
}
.demo-log li {
  padding: 0.15rem 0;
  border-bottom: 1px solid #eee;
  word-break: break-all;
}
</style>
