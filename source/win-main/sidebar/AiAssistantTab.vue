<template>
  <div role="tabpanel" class="ai-assistant-tab">
    <h1>AI Assistant</h1>

    <!-- API Key setup -->
    <div v-if="!apiKey" class="api-key-section">
      <p>Enter your Mistral API key to get started. Get one at console.mistral.ai. It is stored locally on your machine only.</p>
      <input
        v-model="apiKeyInput"
        type="password"
        placeholder="your-mistral-api-key"
        class="api-key-input"
        @keyup.enter="saveApiKey"
      />
      <button @click="saveApiKey" :disabled="!apiKeyInput.trim()">Save API Key</button>
    </div>

    <template v-else>
      <!-- Model selector -->
      <div class="setting-row">
        <label>Model</label>
        <select v-model="selectedModel">
          <option value="mistral-small-latest">Mistral Small (fast)</option>
          <option value="mistral-medium-latest">Mistral Medium</option>
          <option value="mistral-large-latest">Mistral Large</option>
        </select>
      </div>

      <!-- Quick actions -->
      <div class="section-label">Quick actions (use selected text)</div>
      <div class="quick-actions">
        <button @click="runAction('continue')" :disabled="isLoading">Continue</button>
        <button @click="runAction('rewrite')" :disabled="isLoading">Rewrite</button>
        <button @click="runAction('summarize')" :disabled="isLoading">Summarize</button>
        <button @click="runAction('grammar')" :disabled="isLoading">Fix grammar</button>
        <button @click="runAction('humanize')" :disabled="isLoading">Humanize tone</button>
        <button @click="runAction('rephrase')" :disabled="isLoading">Rephrase (x3)</button>
      </div>

      <!-- Expand from notes -->
      <div class="section-label">Expand from notes</div>
      <div class="prompt-section">
        <textarea
          v-model="notesInput"
          placeholder="- Your bullet points&#10;- In your own words&#10;- AI expands into prose"
          rows="4"
        ></textarea>
        <button
          @click="runAction('expand')"
          :disabled="isLoading || !notesInput.trim()"
          class="action-btn"
        >Expand into prose</button>
      </div>

      <!-- Style matching -->
      <div class="section-label">Style matching</div>
      <div class="style-section">
        <textarea
          v-model="styleSample"
          placeholder="Paste a sample of YOUR writing here so AI can match your voice…"
          rows="3"
        ></textarea>
        <button
          @click="runAction('style')"
          :disabled="isLoading || !styleSample.trim()"
          class="action-btn"
        >Rewrite in my style</button>
        <p class="hint">Select the text you want rewritten first, then click the button.</p>
      </div>

      <!-- Custom prompt -->
      <div class="section-label">Custom instruction</div>
      <div class="prompt-section">
        <textarea
          v-model="customPrompt"
          placeholder="e.g. Make this more formal…"
          rows="3"
        ></textarea>
        <button
          @click="runAction('custom')"
          :disabled="isLoading || !customPrompt.trim()"
          class="action-btn"
        >Run</button>
      </div>

      <!-- Loading -->
      <div v-if="isLoading" class="loading">Thinking…</div>

      <!-- Error -->
      <div v-if="error" class="ai-error">{{ error }}</div>

      <!-- Result -->
      <div v-if="result" class="ai-result">
        <div class="result-header">
          <span>Result</span>
          <button @click="copyResult">Copy</button>
          <button @click="result = ''">Clear</button>
        </div>
        <div class="result-text">{{ result }}</div>
      </div>

      <!-- Footer -->
      <div class="footer">
        <button @click="resetApiKey" class="reset-key-btn">Change API key</button>
      </div>
    </template>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted } from 'vue'

const STORAGE_KEY = 'zettlr-mistral-api-key'

const apiKey = ref('')
const apiKeyInput = ref('')
const selectedModel = ref('mistral-small-latest')
const customPrompt = ref('')
const notesInput = ref('')
const styleSample = ref('')
const result = ref('')
const error = ref('')
const isLoading = ref(false)

onMounted(() => {
  apiKey.value = localStorage.getItem(STORAGE_KEY) ?? ''
})

function saveApiKey (): void {
  const key = apiKeyInput.value.trim()
  if (!key) return
  apiKey.value = key
  localStorage.setItem(STORAGE_KEY, key)
  apiKeyInput.value = ''
}

function resetApiKey (): void {
  apiKey.value = ''
  localStorage.removeItem(STORAGE_KEY)
}

function getSelectedText (): string {
  return window.getSelection()?.toString() ?? ''
}

const SYSTEM_PROMPTS: Record<string, string> = {
  continue: 'You are a writing assistant. Continue the text the user provides, naturally matching its style and tone. Return only the continuation, not the original text.',
  rewrite: 'You are a writing assistant. Rewrite the text to improve clarity, flow, and style. Return only the rewritten version.',
  summarize: 'You are a writing assistant. Write a concise summary of the text. Return only the summary.',
  grammar: 'You are a writing assistant. Fix all grammar, spelling, and punctuation errors. Return only the corrected text.',
  humanize: 'You are a writing assistant. Rewrite the text so it sounds natural and human: use varied sentence lengths, everyday vocabulary, contractions where appropriate, and an authentic conversational tone. Avoid stiff or overly formal phrasing. Return only the rewritten version.',
  rephrase: 'You are a writing assistant. Provide exactly 3 distinct rephrased versions of the text, each on its own numbered line (1. 2. 3.). Vary the structure and wording meaningfully between versions. Return only the 3 versions.',
}

async function runAction (action: string): Promise<void> {
  error.value = ''
  result.value = ''

  const selection = getSelectedText()
  let messages: Array<{ role: string; content: string }>

  if (action === 'custom') {
    const content = selection
      ? `${customPrompt.value}\n\nText:\n${selection}`
      : customPrompt.value
    messages = [{ role: 'user', content }]

  } else if (action === 'expand') {
    messages = [
      { role: 'system', content: 'You are a writing assistant. The user provides rough bullet-point notes. Expand them into well-written, natural prose paragraphs. Keep the user\'s meaning and voice. Return only the expanded prose.' },
      { role: 'user', content: notesInput.value }
    ]

  } else if (action === 'style') {
    if (!selection) {
      error.value = 'Select the text you want rewritten first.'
      return
    }
    messages = [
      { role: 'system', content: 'You are a writing assistant that matches writing styles. The user will give you a style sample showing how they write, then text to rewrite. Rewrite the target text to closely match the voice, tone, sentence length, vocabulary, and rhythm of the style sample. Return only the rewritten text.' },
      { role: 'user', content: `Style sample (this is how I write):\n${styleSample.value}\n\nText to rewrite in my style:\n${selection}` }
    ]

  } else {
    messages = [
      { role: 'system', content: SYSTEM_PROMPTS[action] },
      { role: 'user', content: selection || '(No text selected.)' }
    ]
  }

  isLoading.value = true
  try {
    const response = await fetch('https://api.mistral.ai/v1/chat/completions', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        Authorization: `Bearer ${apiKey.value}`
      },
      body: JSON.stringify({
        model: selectedModel.value,
        messages,
        max_tokens: 1024
      })
    })

    if (!response.ok) {
      const errBody = await response.json().catch(() => ({}))
      throw new Error((errBody as any).error?.message ?? `API error ${response.status}`)
    }

    const data = await response.json() as any
    result.value = data.choices?.[0]?.message?.content ?? ''
  } catch (err: any) {
    error.value = String(err?.message ?? 'Unknown error')
  } finally {
    isLoading.value = false
  }
}

function copyResult (): void {
  navigator.clipboard.writeText(result.value).catch(() => {})
}
</script>

<style lang="less">
.ai-assistant-tab {
  padding: 5px 2px;

  h1 { font-size: 16px; margin: 10px 0 12px; }
  p { font-size: 12px; line-height: 1.4; margin-bottom: 8px; }

  .hint {
    font-size: 10px;
    color: #999;
    margin-top: 4px;
    margin-bottom: 0;
  }

  .section-label {
    font-size: 10px;
    text-transform: uppercase;
    letter-spacing: 0.05em;
    color: #888;
    margin: 12px 0 5px;
  }

  /* ── API Key ── */
  .api-key-section {
    .api-key-input {
      width: 100%;
      box-sizing: border-box;
      padding: 6px 8px;
      border-radius: 5px;
      border: 1px solid #ccc;
      margin-bottom: 6px;
      font-size: 12px;
      font-family: monospace;
    }
    button {
      width: 100%;
      padding: 7px;
      border-radius: 5px;
      border: none;
      background: var(--system-accent-color, #4a6da7);
      color: white;
      font-size: 12px;
      cursor: pointer;
      &:disabled { opacity: 0.4; cursor: default; }
    }
  }

  /* ── Model selector ── */
  .setting-row {
    display: flex;
    align-items: center;
    gap: 8px;
    label { font-size: 12px; white-space: nowrap; }
    select {
      flex: 1;
      font-size: 12px;
      padding: 4px 6px;
      border-radius: 5px;
      border: 1px solid #ccc;
    }
  }

  /* ── Quick actions ── */
  .quick-actions {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 5px;

    button {
      padding: 6px 4px;
      border-radius: 5px;
      border: 1px solid #bbb;
      background: rgba(0,0,0,0.05);
      cursor: pointer;
      font-size: 11px;
      text-align: center;
      &:hover:not(:disabled) { background: rgba(0,0,0,0.1); }
      &:disabled { opacity: 0.4; cursor: default; }
    }
  }

  /* ── Text areas + action buttons ── */
  .prompt-section, .style-section {
    textarea {
      width: 100%;
      box-sizing: border-box;
      padding: 6px 8px;
      border-radius: 5px;
      border: 1px solid #ccc;
      font-size: 12px;
      resize: vertical;
      margin-bottom: 5px;
    }
  }

  .action-btn {
    width: 100%;
    padding: 7px;
    border-radius: 5px;
    border: none;
    background: var(--system-accent-color, #4a6da7);
    color: white;
    font-size: 12px;
    cursor: pointer;
    &:disabled { opacity: 0.4; cursor: default; }
    &:hover:not(:disabled) { opacity: 0.88; }
  }

  /* ── Status ── */
  .loading {
    font-size: 12px;
    color: #777;
    text-align: center;
    padding: 12px 0;
  }

  .ai-error {
    margin-top: 8px;
    font-size: 11px;
    color: #c00;
    background: rgba(200,0,0,0.06);
    padding: 8px;
    border-radius: 5px;
  }

  /* ── Result ── */
  .ai-result {
    margin-top: 10px;
    border-radius: 6px;
    background: rgba(0,0,0,0.04);
    padding: 8px;

    .result-header {
      display: flex;
      align-items: center;
      margin-bottom: 6px;
      span { font-size: 11px; font-weight: 600; flex: 1; }
      button {
        font-size: 10px;
        padding: 2px 7px;
        margin-left: 4px;
        border-radius: 3px;
        border: 1px solid #bbb;
        background: transparent;
        cursor: pointer;
        &:hover { background: rgba(0,0,0,0.07); }
      }
    }

    .result-text {
      font-size: 12px;
      white-space: pre-wrap;
      line-height: 1.6;
    }
  }

  /* ── Footer ── */
  .footer {
    margin-top: 14px;
    padding-top: 8px;
    border-top: 1px solid rgba(0,0,0,0.08);
    .reset-key-btn {
      background: none;
      border: none;
      font-size: 10px;
      color: #888;
      cursor: pointer;
      text-decoration: underline;
    }
  }
}

/* Dark mode */
body.dark .ai-assistant-tab {
  .api-key-input, textarea, select {
    background: rgba(255,255,255,0.08);
    border-color: rgba(255,255,255,0.18);
    color: inherit;
  }
  .quick-actions button {
    background: rgba(255,255,255,0.07);
    border-color: rgba(255,255,255,0.15);
    color: inherit;
    &:hover:not(:disabled) { background: rgba(255,255,255,0.13); }
  }
  .ai-result { background: rgba(255,255,255,0.05); }
  .footer { border-top-color: rgba(255,255,255,0.08); }
}
</style>
