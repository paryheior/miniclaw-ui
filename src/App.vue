<template>
  <div class="app-shell">
    <aside class="sidebar">
      <div class="panel brand-panel">
        <div>
          <div class="eyebrow">MiniClaw</div>
          <h1>Chat 调试框架</h1>
          <p>用于调试聊天、流式输出、参数、工具调用和请求载荷。</p>
        </div>
        <span class="badge" :class="connectionStatus">{{ connectionText }}</span>
      </div>

      <div class="panel">
        <div class="panel-title">模型参数</div>

        <label class="field">
          <span>模型</span>
          <select v-model="settings.model">
            <option value="qwen-max">qwen-max</option>
            <option value="qwen-plus">qwen-plus</option>
            <option value="qwen-long">qwen-long</option>
            <option value="deepseek-chat">deepseek-chat</option>
            <option value="gpt-4.1">gpt-4.1</option>
          </select>
        </label>

        <label class="field">
          <span>Temperature</span>
          <input v-model.number="settings.temperature" type="range" min="0" max="2" step="0.1" />
          <small>{{ settings.temperature.toFixed(1) }}</small>
        </label>

        <label class="field">
          <span>Max Tokens</span>
          <input v-model.number="settings.maxTokens" type="number" min="64" max="8192" step="64" />
        </label>

        <label class="switch-row">
          <span>流式输出</span>
          <input v-model="settings.stream" type="checkbox" />
        </label>

        <label class="switch-row">
          <span>显示思考轨迹占位</span>
          <input v-model="settings.showReasoning" type="checkbox" />
        </label>
      </div>

      <div class="panel">
        <div class="panel-title">系统提示词</div>
        <textarea v-model="settings.systemPrompt" rows="6" placeholder="输入 system prompt..." />
      </div>

      <div class="panel">
        <div class="panel-title">调试操作</div>
        <div class="action-grid">
          <button class="secondary" @click="insertSystemMessage">插入 System</button>
          <button class="secondary" @click="insertToolMessage">插入 Tool</button>
          <button class="secondary" @click="copyPayload">复制 Payload</button>
          <button class="danger" @click="clearChat">清空对话</button>
        </div>
      </div>

      <div class="panel">
        <div class="panel-title">请求预览</div>
        <pre class="json-box">{{ requestPayload }}</pre>
      </div>
    </aside>

    <main class="main">
      <header class="topbar panel">
        <div>
          <div class="eyebrow">Workspace</div>
          <h2>Chat Playground</h2>
        </div>
        <div class="topbar-actions">
          <button class="secondary" @click="seedConversation">生成示例</button>
          <button class="primary" @click="fakeReconnect">重连 SSE</button>
        </div>
      </header>

      <section ref="messageListRef" class="chat panel">
        <div v-if="messages.length === 0" class="empty-state">
          <div class="empty-icon">✦</div>
          <h3>开始调试你的聊天应用</h3>
          <p>支持消息记录、流式回复、请求体预览。</p>
        </div>

        <div
            v-for="message in messages"
            :key="message.id"
            class="message-row"
            :class="message.role"
        >
          <div class="avatar">{{ roleAvatar(message.role) }}</div>

          <div class="bubble-wrap">
            <div class="meta">
              <strong>{{ roleLabel(message.role) }}</strong>
              <span>{{ formatTime(message.createdAt) }}</span>
              <span v-if="message.streaming" class="streaming-dot">● 正在输出</span>
            </div>

            <div class="bubble">
              <template v-if="message.role === 'tool'">
                <div class="tool-title">工具调用结果</div>
                <pre>{{ message.content }}</pre>
              </template>

              <template v-else>
                <div class="markdown-body" v-html="parseMarkdown(message.content)"></div>
              </template>
            </div>
          </div>
        </div>
      </section>

      <section class="composer panel">
        <div class="quick-actions">
          <button class="chip" @click="usePreset('帮我总结一下当前会话')">总结会话</button>
          <button class="chip" @click="usePreset('请用 Markdown 格式给我一段 JavaScript 代码示例')">生成代码测试</button>
          <button class="chip" @click="usePreset('请输出本次请求的 JSON')">输出 JSON</button>
        </div>

        <textarea
            v-model="draft"
            rows="3"
            placeholder="输入你的问题，Enter 发送，Shift + Enter 换行"
            @keydown="handleKeydown"
        />

        <div class="composer-footer">
          <span class="hint">当前消息数：{{ messages.length }}</span>

          <div class="topbar-actions">
            <button class="secondary" :disabled="!isSending" @click="stopStreaming">停止</button>
            <button class="primary" :disabled="!draft.trim() || isSending" @click="sendMessage">
              {{ isSending ? '发送中...' : '发送' }}
            </button>
          </div>
        </div>
      </section>
    </main>

    <aside class="debugger">
      <div class="panel debugger-panel">
        <div class="panel-title">事件日志</div>

        <div class="log-list">
          <div v-for="log in logs" :key="log.id" class="log-item">
            <div class="log-time">{{ formatTime(log.time) }}</div>
            <div class="log-text">{{ log.text }}</div>
          </div>
        </div>
      </div>
    </aside>
  </div>
</template>

<script setup>
import { computed, nextTick, onMounted, ref, watch } from 'vue'
import { marked } from 'marked'
import DOMPurify from 'dompurify'

const API_BASE_URL = 'http://localhost:8080'

// 如果你的后端还没改 Agent 接口，可以临时把这里改回 /api/chat 和 /api/chat/stream
const API = {
  normal: `${API_BASE_URL}/api/send`,
  stream: `${API_BASE_URL}/api/send/stream`
}

const messageListRef = ref(null)
const draft = ref('')
const isSending = ref(false)
const connectionStatus = ref('online')
const abortController = ref(null)

const settings = ref({
  model: 'qwen-plus',
  temperature: 0.7,
  maxTokens: 2048,
  stream: true,
  showReasoning: false,
  systemPrompt: '你是一个适合调试聊天应用的 AI 助手。请优先输出结构化、可复现的建议。'
})

const messages = ref([])

const logs = ref([
  { id: crypto.randomUUID(), time: Date.now(), text: '应用启动完成。' },
  { id: crypto.randomUUID(), time: Date.now(), text: `当前服务：${API_BASE_URL}` },
  { id: crypto.randomUUID(), time: Date.now(), text: `非流式接口：${API.normal}` },
  { id: crypto.randomUUID(), time: Date.now(), text: `流式接口：${API.stream}` }
])

const connectionText = computed(() => {
  if (connectionStatus.value === 'online') return '在线'
  if (connectionStatus.value === 'reconnecting') return '重连中'
  return '离线'
})

const requestPayload = computed(() => {
  return JSON.stringify(buildPayload(), null, 2)
})

function buildPayload() {
  return {
    model: settings.value.model,
    stream: settings.value.stream,
    temperature: settings.value.temperature,
    maxTokens: settings.value.maxTokens,
    systemPrompt: settings.value.systemPrompt,
    messages: collectRequestMessages()
  }
}

function collectRequestMessages() {
  return messages.value
      .filter((m) => ['user', 'assistant', 'system'].includes(m.role))
      .filter((m) => typeof m.content === 'string' && m.content.trim() !== '')
      .map(({ role, content }) => ({ role, content }))
}

function parseMarkdown(text) {
  if (!text) return ''
  const html = marked.parse(text)
  return DOMPurify.sanitize(html)
}

function roleAvatar(role) {
  return {
    user: '你',
    assistant: 'AI',
    system: 'SYS',
    tool: 'TOOL'
  }[role] || 'MSG'
}

function roleLabel(role) {
  return {
    user: '用户',
    assistant: '助手',
    system: '系统',
    tool: '工具'
  }[role] || '消息'
}

function formatTime(time) {
  return new Date(time).toLocaleTimeString([], {
    hour: '2-digit',
    minute: '2-digit',
    second: '2-digit'
  })
}

function addLog(text) {
  logs.value.unshift({
    id: crypto.randomUUID(),
    time: Date.now(),
    text
  })
}

function pushMessage(role, content, extra = {}) {
  messages.value.push({
    id: crypto.randomUUID(),
    role,
    content,
    createdAt: Date.now(),
    streaming: false,
    ...extra
  })
}

function insertSystemMessage() {
  pushMessage('system', '你现在处于调试模式，请以 JSON 格式返回关键字段。')
  addLog('插入了一条 system 消息。')
}

function insertToolMessage() {
  pushMessage(
      'tool',
      JSON.stringify(
          {
            tool: 'search_docs',
            success: true,
            latency_ms: 143
          },
          null,
          2
      )
  )
  addLog('插入了一条 tool 消息。')
}

function createAssistantMessage(streaming) {
  const message = {
    id: crypto.randomUUID(),
    role: 'assistant',
    content: '',
    createdAt: Date.now(),
    streaming
  }

  messages.value.push(message)

  return message.id
}

function getMessageIndexById(messageId) {
  return messages.value.findIndex((m) => m.id === messageId)
}

function updateAssistantMessage(messageId, content) {
  const index = getMessageIndexById(messageId)
  if (index === -1) return

  messages.value[index] = {
    ...messages.value[index],
    content
  }
}

function appendAssistantMessage(messageId, chunk) {
  if (!chunk) return

  const index = getMessageIndexById(messageId)
  if (index === -1) return

  messages.value[index] = {
    ...messages.value[index],
    content: messages.value[index].content + chunk
  }
}

function setAssistantStreaming(messageId, streaming) {
  const index = getMessageIndexById(messageId)
  if (index === -1) return

  messages.value[index] = {
    ...messages.value[index],
    streaming
  }
}

async function sendMessage() {
  const text = draft.value.trim()
  if (!text || isSending.value) return

  pushMessage('user', text)
  addLog(`发送：${text.slice(0, 20)}...`)

  draft.value = ''
  isSending.value = true
  connectionStatus.value = 'online'

  const assistantMessageId = createAssistantMessage(settings.value.stream)
  const payload = buildPayload()

  abortController.value = new AbortController()

  try {
    if (settings.value.stream) {
      await sendStreamRequest(payload, assistantMessageId)
    } else {
      await sendNormalRequest(payload, assistantMessageId)
    }
  } catch (error) {
    setAssistantStreaming(assistantMessageId, false)

    if (error.name === 'AbortError') {
      addLog('请求已中断。')
      return
    }

    const index = getMessageIndexById(assistantMessageId)
    const currentContent = index === -1 ? '' : messages.value[index].content

    if (!currentContent) {
      updateAssistantMessage(assistantMessageId, `请求失败：${error.message}`)
    }

    connectionStatus.value = 'offline'
    addLog(`请求失败：${error.message}`)
  } finally {
    setAssistantStreaming(assistantMessageId, false)
    isSending.value = false
    abortController.value = null
    await scrollToBottom()
  }
}

async function sendNormalRequest(payload, assistantMessageId) {
  addLog('请求非流式接口...')

  const response = await fetch(API.normal, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(payload),
    signal: abortController.value.signal
  })

  if (!response.ok) {
    const errorText = await safeReadText(response)
    throw new Error(errorText || `HTTP ${response.status}`)
  }

  const data = await response.json()

  if (data.success === false) {
    throw new Error(data.errorMessage || 'Agent 执行失败')
  }

  updateAssistantMessage(assistantMessageId, data.output || data.content || '')
  setAssistantStreaming(assistantMessageId, false)

  addLog('收到非流式响应。')
  await scrollToBottom()
}

async function sendStreamRequest(payload, assistantMessageId) {
  addLog('请求流式接口...')

  const response = await fetch(API.stream, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Accept: 'text/event-stream'
    },
    body: JSON.stringify(payload),
    signal: abortController.value.signal
  })

  if (!response.ok || !response.body) {
    const errorText = await safeReadText(response)
    throw new Error(errorText || `HTTP ${response.status}`)
  }

  connectionStatus.value = 'online'

  await consumeSseStream(response.body, assistantMessageId)

  setAssistantStreaming(assistantMessageId, false)
  addLog('流式输出结束。')
}

async function consumeSseStream(body, assistantMessageId) {
  const reader = body.getReader()
  const decoder = new TextDecoder('utf-8')
  let buffer = ''

  while (true) {
    const { value, done } = await reader.read()

    if (done) {
      buffer += decoder.decode()

      if (buffer.trim()) {
        handleSseEvent(buffer, assistantMessageId)
      }

      break
    }

    buffer += decoder.decode(value, { stream: true })

    const events = buffer.split(/\r?\n\r?\n/)
    buffer = events.pop() || ''

    for (const rawEvent of events) {
      const shouldStop = handleSseEvent(rawEvent, assistantMessageId)
      await scrollToBottom()

      if (shouldStop) {
        return
      }
    }
  }
}

function handleSseEvent(rawEvent, assistantMessageId) {
  if (!rawEvent || !rawEvent.trim()) return false

  let eventName = 'message'
  const dataLines = []

  const lines = rawEvent.split(/\r?\n/)

  for (const line of lines) {
    if (line.startsWith(':')) {
      continue
    }

    if (line.startsWith('event:')) {
      eventName = line.slice('event:'.length).trim()
      continue
    }

    if (line.startsWith('data:')) {
      let data = line.slice('data:'.length)

      // 只去掉 SSE 协议自动加的一个空格，不要 trim()
      if (data.startsWith(' ')) {
        data = data.slice(1)
      }

      dataLines.push(data)
    }
  }

  const data = dataLines.join('\n')

  if (data === '[DONE]') {
    return true
  }

  if (eventName === 'done') {
    return true
  }

  if (eventName === 'error') {
    appendAssistantMessage(assistantMessageId, data ? `\n\n请求错误：${data}` : '\n\n请求错误')
    return true
  }

  if (dataLines.length > 0) {
    appendAssistantMessage(assistantMessageId, data)
  }

  return false
}

async function safeReadText(response) {
  try {
    return await response.text()
  } catch {
    return ''
  }
}

function stopStreaming() {
  if (abortController.value) {
    abortController.value.abort()
    abortController.value = null
  }
}

function clearChat() {
  stopStreaming()
  messages.value = []
  addLog('已清空记录。')
}

function fakeReconnect() {
  connectionStatus.value = 'reconnecting'
  addLog('模拟重连...')

  setTimeout(() => {
    connectionStatus.value = 'online'
    addLog('已恢复。')
  }, 1200)
}

function seedConversation() {
  clearChat()

  pushMessage('system', '你是一个前端联调助手。')
  pushMessage(
      'user',
      '测试 **Markdown** 渲染和代码块：\n```javascript\nconsole.log("Hello MiniClaw!")\n```'
  )
  pushMessage(
      'assistant',
      '一切正常，**字号已变大**，且支持 `行内代码` 和列表！'
  )

  addLog('生成示例对话。')
}

function usePreset(text) {
  draft.value = text
}

async function copyPayload() {
  await navigator.clipboard.writeText(requestPayload.value)
  addLog('已复制请求体。')
}

function handleKeydown(event) {
  if (event.key === 'Enter' && !event.shiftKey) {
    event.preventDefault()
    sendMessage()
  }
}

async function scrollToBottom() {
  await nextTick()

  const el = messageListRef.value

  if (el) {
    el.scrollTop = el.scrollHeight
  }
}

watch(
    () => messages.value.length,
    async () => {
      await scrollToBottom()
    }
)

onMounted(() => {
  seedConversation()
})
</script>

<style scoped>
:global(body) {
  margin: 0;
  padding: 0;
  overflow: hidden;
  font-family: Inter, ui-sans-serif, system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
  background: #0b1020;
  color: #e5e7eb;
}

:global(*) {
  box-sizing: border-box;
}

::-webkit-scrollbar {
  width: 6px;
  height: 6px;
}

::-webkit-scrollbar-track {
  background: transparent;
}

::-webkit-scrollbar-thumb {
  background: rgba(148, 163, 184, 0.2);
  border-radius: 10px;
}

::-webkit-scrollbar-thumb:hover {
  background: rgba(148, 163, 184, 0.4);
}

.app-shell {
  position: fixed;
  inset: 0;
  display: flex;
  gap: 16px;
  padding: 16px;
  background:
      radial-gradient(circle at top left, rgba(59, 130, 246, 0.18), transparent 24%),
      radial-gradient(circle at bottom right, rgba(16, 185, 129, 0.12), transparent 22%),
      #0b1020;
}

.sidebar {
  width: 330px;
  flex-shrink: 0;
  display: flex;
  flex-direction: column;
  gap: 16px;
  height: 100%;
  overflow-y: auto;
  padding-right: 4px;
}

.main {
  flex: 1;
  min-width: 380px;
  display: flex;
  flex-direction: column;
  gap: 16px;
  height: 100%;
}

.debugger {
  width: 300px;
  flex-shrink: 0;
  display: flex;
  flex-direction: column;
  gap: 16px;
  height: 100%;
  overflow-y: auto;
  padding-right: 4px;
}

.topbar {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
}

.topbar-actions {
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
}

.chat {
  flex: 1;
  min-height: 0;
  overflow-y: auto;
  width: 100%;
}

.composer {
  flex-shrink: 0;
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.panel {
  background: rgba(15, 23, 42, 0.82);
  border: 1px solid rgba(148, 163, 184, 0.16);
  border-radius: 24px;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.25);
  backdrop-filter: blur(10px);
}

.brand-panel,
.topbar,
.composer,
.debugger-panel,
.chat,
.sidebar .panel {
  padding: 18px;
}

.eyebrow {
  font-size: 13px;
  text-transform: uppercase;
  letter-spacing: 0.12em;
  color: #93c5fd;
  margin-bottom: 6px;
}

h1,
h2,
h3,
p {
  margin: 0;
}

h1,
h2 {
  font-size: 22px;
}

p {
  color: #cbd5e1;
  line-height: 1.6;
  font-size: 15px;
}

.badge {
  display: inline-block;
  margin-top: 12px;
  padding: 6px 12px;
  border-radius: 999px;
  font-size: 13px;
  border: 1px solid rgba(255, 255, 255, 0.1);
}

.badge.online {
  background: rgba(16, 185, 129, 0.14);
  color: #86efac;
}

.badge.reconnecting {
  background: rgba(250, 204, 21, 0.14);
  color: #fde68a;
}

.badge.offline {
  background: rgba(239, 68, 68, 0.14);
  color: #fca5a5;
}

.panel-title {
  font-weight: 700;
  font-size: 16px;
  margin-bottom: 14px;
  color: #f1f5f9;
}

.field,
.switch-row {
  display: flex;
  flex-direction: column;
  gap: 8px;
  margin-bottom: 14px;
  font-size: 15px;
}

.switch-row {
  flex-direction: row;
  justify-content: space-between;
  align-items: center;
}

input,
select,
textarea,
button {
  font: inherit;
}

input[type='number'],
select,
textarea {
  width: 100%;
  background: rgba(30, 41, 59, 0.92);
  color: #e2e8f0;
  border: 1px solid rgba(148, 163, 184, 0.18);
  border-radius: 14px;
  padding: 12px 14px;
  outline: none;
  font-size: 16px;
}

textarea {
  resize: vertical;
  line-height: 1.5;
}

button {
  border: 0;
  border-radius: 12px;
  padding: 12px 16px;
  cursor: pointer;
  transition: 0.15s;
  font-size: 15px;
  font-weight: 500;
}

button:hover {
  transform: translateY(-1px);
}

button:disabled {
  cursor: not-allowed;
  opacity: 0.6;
  transform: none;
}

.primary {
  background: linear-gradient(135deg, #2563eb, #4f46e5);
  color: white;
}

.secondary {
  background: rgba(51, 65, 85, 0.85);
  color: #e2e8f0;
}

.danger {
  background: rgba(127, 29, 29, 0.82);
  color: #fecaca;
}

.action-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 10px;
}

.json-box {
  margin: 0;
  white-space: pre-wrap;
  word-break: break-word;
  background: rgba(2, 6, 23, 0.8);
  border-radius: 14px;
  padding: 12px;
  font-size: 13px;
  color: #bfdbfe;
  max-height: 200px;
  overflow-y: auto;
}

.empty-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 100%;
  text-align: center;
  gap: 12px;
  color: #94a3b8;
}

.empty-icon {
  width: 64px;
  height: 64px;
  display: grid;
  place-items: center;
  border-radius: 18px;
  font-size: 28px;
  background: rgba(37, 99, 235, 0.14);
}

.message-row {
  display: flex;
  gap: 16px;
  margin-bottom: 24px;
}

.avatar {
  width: 42px;
  height: 42px;
  border-radius: 14px;
  display: grid;
  place-items: center;
  background: rgba(51, 65, 85, 0.9);
  color: #e2e8f0;
  font-size: 13px;
  font-weight: 600;
  flex-shrink: 0;
}

.bubble-wrap {
  flex: 1;
  min-width: 0;
  display: flex;
  flex-direction: column;
}

.meta {
  display: flex;
  gap: 10px;
  align-items: center;
  font-size: 13px;
  color: #94a3b8;
  margin-bottom: 8px;
  flex-wrap: wrap;
}

.streaming-dot {
  color: #86efac;
}

.bubble {
  display: inline-block;
  width: fit-content;
  max-width: 100%;
  background: #0f172a;
  border: 1px solid #334155;
  border-radius: 16px;
  padding: 14px 18px;
  line-height: 1.7;
  font-size: 16px;
}

.message-row.user .bubble {
  background: rgba(30, 41, 59, 0.96);
}

.message-row.assistant .bubble {
  background: rgba(17, 24, 39, 0.98);
}

.message-row.system .bubble {
  border-color: rgba(96, 165, 250, 0.35);
}

.message-row.tool .bubble {
  border-color: rgba(16, 185, 129, 0.35);
}

.tool-title {
  font-size: 13px;
  color: #86efac;
  margin-bottom: 8px;
}

:deep(.markdown-body) {
  color: #e2e8f0;
}

:deep(.markdown-body p) {
  margin-top: 0;
  margin-bottom: 12px;
  font-size: 16px;
  color: inherit;
}

:deep(.markdown-body p:last-child) {
  margin-bottom: 0;
}

:deep(.markdown-body a) {
  color: #60a5fa;
  text-decoration: none;
}

:deep(.markdown-body a:hover) {
  text-decoration: underline;
}

:deep(.markdown-body code) {
  background: rgba(0, 0, 0, 0.3);
  padding: 3px 6px;
  border-radius: 6px;
  font-family: ui-monospace, SFMono-Regular, Consolas, "Liberation Mono", Menlo, monospace;
  font-size: 14px;
  color: #93c5fd;
}

:deep(.markdown-body pre) {
  background: rgba(2, 6, 23, 0.8);
  padding: 16px;
  border-radius: 12px;
  overflow-x: auto;
  margin: 12px 0;
  border: 1px solid rgba(148, 163, 184, 0.15);
}

:deep(.markdown-body pre code) {
  background: transparent;
  padding: 0;
  color: #e2e8f0;
  font-size: 14px;
}

:deep(.markdown-body ul),
:deep(.markdown-body ol) {
  margin-top: 0;
  margin-bottom: 12px;
  padding-left: 24px;
}

:deep(.markdown-body li) {
  margin-bottom: 6px;
}

:deep(.markdown-body blockquote) {
  border-left: 4px solid #3b82f6;
  margin: 0 0 12px 0;
  padding-left: 14px;
  color: #94a3b8;
  background: rgba(59, 130, 246, 0.05);
  padding-top: 8px;
  padding-bottom: 8px;
  border-radius: 0 8px 8px 0;
}

.quick-actions {
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
}

.chip {
  background: rgba(37, 99, 235, 0.12);
  color: #bfdbfe;
  border: 1px solid rgba(96, 165, 250, 0.16);
  padding: 8px 14px;
  border-radius: 999px;
  font-size: 13px;
}

.composer-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 12px;
  flex-wrap: wrap;
}

.hint {
  color: #94a3b8;
  font-size: 13px;
}

.log-list {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.log-item {
  border-radius: 14px;
  padding: 14px;
  background: rgba(2, 6, 23, 0.7);
  border: 1px solid rgba(148, 163, 184, 0.12);
}

.log-time {
  font-size: 12px;
  color: #93c5fd;
  margin-bottom: 6px;
}

.log-text {
  font-size: 14px;
  color: #e2e8f0;
  line-height: 1.6;
}

@media (max-width: 1200px) {
  .debugger {
    display: none;
  }
}

@media (max-width: 768px) {
  :global(body) {
    overflow: auto;
  }

  .app-shell {
    position: static;
    flex-direction: column;
    height: auto;
  }

  .sidebar,
  .main {
    width: 100%;
    min-width: 0;
  }

  .chat {
    min-height: 60vh;
  }
}
</style>