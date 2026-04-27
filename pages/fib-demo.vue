<script setup lang="ts">
/**
 * FIB demo page — integrates the close-ended activity player hosting a
 * fill-in-the-blanks item. The host page only uses the Activity interface;
 * it never calls item-player APIs directly.
 *
 * Side-effect imports register the required custom elements before mount.
 */

// Register custom elements on the client only — Lit uses browser-only DOM APIs.
if (import.meta.client) {
  await Promise.all([
    import('@comprodls/w-close-ended'), // activity shell
    import('@comprodls/w-fib'),         // item player (side-effect only)
  ])
}

import { ref, onMounted, onBeforeUnmount } from 'vue'
import type { Activity } from '@comprodls/w-player-core'

// ---------------------------------------------------------------------------
// Activity JSON — FIB item wrapped in a close-ended activity shell.
// ---------------------------------------------------------------------------
const LAUNCH_ID = 'starter-fib-01'

const itemData = {
  itemId: 'item-capitals',
  type: 'fib',
  subType: 'text',
  status: 'active',
  displayType: 'block',
  toolName: 'starter-app',
  toolVersion: '1.0.0',
  systemMeta: {},
  itemMeta: {},
  itemResources: {},
  itemBody: {
    preferences: {
      layout: {
        orientation: 'top-bottom',
        responseAreaAlignment: 'left',
        responseInputType: 'word',
        responseTextAlignment: 'center',
      },
      ariaLabels: {
        questionLabel: 'Fill in the Blanks Question',
        directionLabel: 'Title',
        instructionLabel: 'Instructions',
        promptLabel: 'Prompt',
        questionItemLabel: 'Question Area',
      },
    },
    stimulus: {
      content: { type: 'item-text', data: 'World Capitals' },
    },
    instruction: {
      content: {
        type: 'item-xhtml',
        data: '<p>Type the capital city for each country.</p>',
      },
    },
    viewLabels: { instructionHeading: 'Directions' },
    responseContainer: [
      { id: 'cap-1', type: 'text' },
      { id: 'cap-2', type: 'text' },
      { id: 'cap-3', type: 'text' },
    ],
    responseTemplate: {
      content: {
        type: 'item-xhtml-editor-player',
        data: {
          editor: {
            content: {
              type: 'item-xhtml',
              data: 'The capital of France is <blank id="cap-1"/>. The capital of Japan is <blank id="cap-2"/>. The capital of Brazil is <blank id="cap-3"/>.',
            },
          },
          player: {
            content: {
              type: 'item-xhtml',
              data: 'The capital of France is <blank id="cap-1"/>. The capital of Japan is <blank id="cap-2"/>. The capital of Brazil is <blank id="cap-3"/>.',
            },
          },
        },
      },
    },
    validations: [
      { validationId: 'v1', responseContainer: 'cap-1', correctResponse: 'Paris',    correctDisplay: 'Paris',    operator: 'text_equals' },
      { validationId: 'v2', responseContainer: 'cap-2', correctResponse: 'Tokyo',    correctDisplay: 'Tokyo',    operator: 'text_equals' },
      { validationId: 'v3', responseContainer: 'cap-3', correctResponse: 'Brasilia', correctDisplay: 'Brasilia', operator: 'text_equals' },
    ],
    scoring: {
      rules: [{
        type: 'sum',
        score: 3,
        rules: [
          { type: 'compute', validationId: 'v1', score: 1 },
          { type: 'compute', validationId: 'v2', score: 1 },
          { type: 'compute', validationId: 'v3', score: 1 },
        ],
      }],
    },
    hints: [],
    feedback: null,
    prompt: null,
  },
}

const activityJson = {
  activityId: LAUNCH_ID,
  itemId: LAUNCH_ID,
  type: 'activity',
  subType: 'close-ended',
  itemMeta: {},
  itemBody: {
    questions: [{ id: 'item-capitals', meta: { score: 3, defaultScore: 0 } }],
  },
  itemFragments: {
    'item-capitals': { data: itemData },
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
      <h1>FIB Activity Demo</h1>
      <p>Shell: <code>@comprodls/w-close-ended</code> · Item: <code>@comprodls/w-fib</code></p>
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
