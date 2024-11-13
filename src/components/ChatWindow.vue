<script setup lang="ts">
// 在文件开头添加环境变量类型声明
declare module '@vue/runtime-core' {
  interface ImportMetaEnv {
    VITE_OPENAI_API_URL: string
    VITE_OPENAI_API_KEY: string
  }
}

import { ref, onMounted, nextTick } from 'vue'
import { marked } from 'marked'
import hljs from 'highlight.js'
import 'highlight.js/styles/github.css'
import { handleApiError } from '../utils/errorHandler'

// 配置 marked
marked.setOptions({
  breaks: true,        // 启用换行符
  gfm: true,          // 启用 GitHub 风格的 Markdown
  headerIds: true,    // 为标题添加 id
  mangle: false,      // 不转义内联 HTML
  pedantic: false,    // 尽可能地兼容 markdown.pl
  sanitize: false,    // 不过滤 HTML 标签
  smartLists: true,   // 使用更智能的列表行为
  smartypants: false, // 不使用更智能的标点符号
  langPrefix: 'hljs language-', // 添加 highlight.js 的类名前缀
  highlight: function(code: string, lang: string) {
    if (lang && hljs.getLanguage(lang)) {
      try {
        return hljs.highlight(code, { language: lang }).value
      } catch (err) {
        console.error('代码高亮失败:', err)
      }
    }
    return hljs.highlightAuto(code).value
  }
})

// 修改 renderMarkdown 函数
function renderMarkdown(content: string) {
  try {
    const html = marked.parse(content)
    console.log('Rendered HTML:', html) // 添加调试日志
    return html
  } catch (error) {
    console.error('Markdown 渲染失败:', error)
    return content
  }
}

// 定义消息类型
interface ChatMessage {
  role: 'user' | 'assistant' | 'system'
  content: string
  isPlaying?: boolean
  audioUrl?: string
}

const messages = ref<ChatMessage[]>([])
const inputMessage = ref('')
const isLoading = ref(false)

// 添加消息容器的引用
const messageContainer = ref<HTMLElement | null>(null)

// 添加滚动到底部的函数
const scrollToBottom = async () => {
  await nextTick() // 等待 DOM 更新
  if (messageContainer.value) {
    messageContainer.value.scrollTop = messageContainer.value.scrollHeight
  }
}

// 系统提示词，用来定义 AI 助手的行为和记忆
const systemPrompt = `你是一个友好的AI助手。请记住以下关于用户的信息，并在对话中自然地运用这些信息：
- 你应该记住用户之前提到过的信息
- 在回答中要体现出你记得之前的对话内容
- 如果用户说了一些新的个人信息，你应该记住它们
请以自然、友好的方式与用户交谈。`

// 初始化函数
onMounted(async () => {
  // 从 localStorage 加载历史消息
  const savedMessages = localStorage.getItem('chatHistory')
  if (savedMessages) {
    messages.value = JSON.parse(savedMessages)
    await scrollToBottom()
  } else {
    // 如果没有历史消息，添加系统提示
    messages.value = [{
      role: 'system',
      content: systemPrompt
    }]
  }
})

// 保存消息到 localStorage
function saveMessages() {
  localStorage.setItem('chatHistory', JSON.stringify(messages.value))
}

// 清除历史记录
function clearHistory() {
  messages.value = [{
    role: 'system',
    content: systemPrompt
  }]
  localStorage.removeItem('chatHistory')
}

// 添加音频播放控制
const audioPlayer = ref<HTMLAudioElement | null>(null)
const isPlaying = ref(false)

// 添加文字转语音的函数
async function textToSpeech(text: string, messageIndex: number) {
  try {
    // 如果该消息正在播放，则停止
    if (messages.value[messageIndex].isPlaying) {
      stopPlaying(messageIndex)
      return
    }

    // 如果已经有音频 URL，直接播放
    if (messages.value[messageIndex].audioUrl) {
      playAudio(messages.value[messageIndex].audioUrl, messageIndex)
      return
    }

    const response = await fetch('https://api.gptsapi.net/v1/audio/speech', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${import.meta.env.VITE_OPENAI_API_KEY}`
      },
      body: JSON.stringify({
        model: 'tts-1',
        input: text,
        voice: 'alloy',
        speed: 1.0
      })
    })

    if (!response.ok) {
      throw new Error('语音生成失败')
    }

    const audioBlob = await response.blob()
    const audioUrl = URL.createObjectURL(audioBlob)
    
    // 保存音频 URL
    messages.value[messageIndex].audioUrl = audioUrl
    playAudio(audioUrl, messageIndex)
  } catch (error) {
    console.error('语音生成失败:', error)
    alert('语音生成失败，请稍后重试')
  }
}

// 添加播放音频的函数
function playAudio(audioUrl: string, messageIndex: number) {
  // 先停止所有正在播放的音频
  messages.value.forEach((msg, index) => {
    if (msg.isPlaying && index !== messageIndex) {
      stopPlaying(index)
    }
  })

  if (audioPlayer.value) {
    audioPlayer.value.src = audioUrl
    audioPlayer.value.onended = () => {
      messages.value[messageIndex].isPlaying = false
    }
    audioPlayer.value.play()
    messages.value[messageIndex].isPlaying = true
  }
}

// 修改停止播放函数，合并两个版本的功能
function stopPlaying(messageIndex?: number) {
  if (audioPlayer.value) {
    audioPlayer.value.pause()
    audioPlayer.value.currentTime = 0
    
    // 如果提供了消息索引，更新特定消息的播放状态
    if (typeof messageIndex === 'number') {
      messages.value[messageIndex].isPlaying = false
    }
    
    // 更新全局播放状态
    isPlaying.value = false
  }
}

// 修改处理流式响应的函数
async function handleStream(reader: ReadableStreamDefaultReader<Uint8Array>) {
  const decoder = new TextDecoder()
  let currentMessage = ''
  
  try {
    while (true) {
      const { done, value } = await reader.read()
      if (done) {
        // 消息接收完成，获取最后一条消息的索引
        const lastMessageIndex = messages.value.length - 1
        // 自动播放最后一条消息
        await textToSpeech(currentMessage, lastMessageIndex)
        break
      }
      
      const chunk = decoder.decode(value)
      const lines = chunk.split('\n')
      
      for (const line of lines) {
        if (line.startsWith('data: ')) {
          const data = line.slice(6)
          if (data === '[DONE]') continue
          
          try {
            const parsed = JSON.parse(data)
            const content = parsed.choices[0].delta.content
            if (content) {
              currentMessage += content
              messages.value[messages.value.length - 1].content = currentMessage
              saveMessages()
              // 每次更新消息后滚动到底部
              await scrollToBottom()
            }
          } catch (e) {
            console.error('解析响应数据失败:', e)
          }
        }
      }
    }
  } catch (error) {
    console.error('读取流数据失败:', error)
  }
}

// 修改音频播放结束的处理函数
function handleAudioEnded() {
  stopPlaying() // 使用统一的 stopPlaying 函数
}

// 发送消息的方法
async function sendMessage() {
  if (!inputMessage.value.trim()) return
  
  // 添加用户消息
  messages.value.push({
    role: 'user',
    content: inputMessage.value
  })
  
  // 保存用户消息并清空输入框
  const userMessage = inputMessage.value
  inputMessage.value = ''
  
  // 滚动到底部
  await scrollToBottom()
  
  try {
    isLoading.value = true
    
    // 检查环境变量
    if (!import.meta.env.VITE_OPENAI_API_URL || !import.meta.env.VITE_OPENAI_API_KEY) {
      throw new Error('API 配置缺失，请检查环境变量')
    }
    
    const response = await fetch(import.meta.env.VITE_OPENAI_API_URL, {
      method: 'POST',
      headers: new Headers({
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${import.meta.env.VITE_OPENAI_API_KEY}`
      }),
      body: JSON.stringify({
        model: "gpt-3.5-turbo",
        messages: messages.value.map(msg => ({
          role: msg.role,
          content: msg.content
        })),
        temperature: 0.7,
        stream: true
      })
    })
    
    if (!response.ok) {
      const errorText = await response.text()
      console.error('API 响应错误:', errorText)
      throw new Error(`API 请求失败: ${response.status} ${response.statusText}`)
    }
    
    if (!response.body) {
      throw new Error('响应中没有数据')
    }

    // 开始接收流式响应时才添加助手消息
    messages.value.push({
      role: 'assistant',
      content: ''
    })

    const reader = response.body.getReader()
    await handleStream(reader)
    
  } catch (error: any) {
    console.error('发送消息失败:', error)
    // 如果已经添加了助手消息，则移除它
    if (messages.value[messages.value.length - 1].role === 'assistant') {
      messages.value.pop()
    }
    const errorMessage = handleApiError(error)
    alert(`发送消息失败: ${errorMessage}\n请检查控制台以获取更多信息`)
  } finally {
    isLoading.value = false
  }
}

// 添加默认导出
defineExpose({
  sendMessage,
  clearHistory
})
</script>

<template>
  <div class="chat-container">
    <!-- 添加音频播放器 -->
    <audio 
      ref="audioPlayer"
      @ended="handleAudioEnded"
      style="display: none;"
    ></audio>

    <div class="chat-header">
      <h1>AI 助手</h1>
      <div class="header-buttons">
        <button 
          v-if="isPlaying" 
          class="stop-button"
          @click="stopPlaying"
        >
          停止播放
        </button>
        <button class="clear-button" @click="clearHistory">
          清除历史
        </button>
      </div>
    </div>

    <div class="messages" ref="messageContainer">
      <div v-for="(message, index) in messages" 
           :key="index" 
           :class="['message', message.role]">
        <!-- 不显示系统消息 -->
        <template v-if="message.role !== 'system'">
          <div class="avatar">
            {{ message.role === 'user' ? '👤' : '🤖' }}
          </div>
          <div class="message-content">
            <div v-if="message.role === 'user'">
              {{ message.content }}
            </div>
            <div v-else class="assistant-message">
              <div class="markdown-body"
                   v-html="renderMarkdown(message.content)">
              </div>
              <button 
                v-if="message.role === 'assistant'"
                class="play-button"
                :data-playing="message.isPlaying"
                @click="textToSpeech(message.content, index)"
              >
                {{ message.isPlaying ? '⏹️' : '▶️' }}
              </button>
            </div>
          </div>
        </template>
      </div>
      <div v-if="isLoading" class="loading">
        <div class="typing-indicator">
          <span></span>
          <span></span>
          <span></span>
        </div>
      </div>
    </div>
    
    <div class="input-container">
      <div class="input-area">
        <input 
          v-model="inputMessage"
          @keyup.enter="sendMessage"
          placeholder="输入你的问题..."
          type="text"
        />
        <button @click="sendMessage" :disabled="isLoading">
          <span v-if="!isLoading">发送</span>
          <span v-else>发送中...</span>
        </button>
      </div>
    </div>
  </div>
</template>

<style scoped>
.chat-container {
  max-width: 900px;
  margin: 0 auto;
  height: 90vh;
  display: flex;
  flex-direction: column;
  background: #ffffff;
  border-radius: 16px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.08);
}

.chat-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 20px;
  border-bottom: 1px solid #eee;
}

.chat-header h1 {
  margin: 0;
  font-size: 1.5rem;
  color: #333;
  font-weight: 500;
}

.clear-button {
  padding: 8px 16px;
  background: #ff4757;
  color: white;
  border: none;
  border-radius: 6px;
  font-size: 14px;
  cursor: pointer;
  transition: all 0.3s ease;
}

.clear-button:hover {
  background: #ff6b81;
}

.messages {
  flex-grow: 1;
  overflow-y: auto;
  padding: 20px;
  background: #f8f9fa;
  scroll-behavior: smooth;
}

.message {
  display: flex;
  align-items: flex-start;
  margin-bottom: 20px;
  animation: fadeIn 0.3s ease;
}

@keyframes fadeIn {
  from { opacity: 0; transform: translateY(10px); }
  to { opacity: 1; transform: translateY(0); }
}

.avatar {
  width: 35px;
  height: 35px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 20px;
  margin-right: 12px;
  background: #fff;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.message-content {
  padding: 12px 16px;
  border-radius: 12px;
  width: 70%;
  line-height: 1.5;
  word-wrap: break-word;
  overflow-wrap: break-word;
}

.user {
  flex-direction: row-reverse;
  justify-content: flex-start;
}

.user .avatar {
  margin-right: 0;
  margin-left: 12px;
}

.user .message-content {
  background: #007AFF;
  color: white;
  border-radius: 12px 12px 0 12px;
}

.assistant .message-content {
  background: white;
  color: #333;
  border-radius: 12px 12px 12px 0;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  justify-content: flex-start;
}

.input-container {
  padding: 20px;
  background: #ffffff;
  border-top: 1px solid #eee;
}

.input-area {
  display: flex;
  gap: 10px;
  max-width: 900px;
  margin: 0 auto;
}

input {
  flex-grow: 1;
  padding: 12px 16px;
  border: 1px solid #e0e0e0;
  border-radius: 8px;
  font-size: 14px;
  transition: all 0.3s ease;
}

input:focus {
  outline: none;
  border-color: #007AFF;
  box-shadow: 0 0 0 2px rgba(0, 122, 255, 0.1);
}

button {
  padding: 12px 24px;
  background: #007AFF;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 14px;
  cursor: pointer;
  transition: all 0.3s ease;
}

button:hover {
  background: #0056b3;
}

button:disabled {
  background: #cccccc;
  cursor: not-allowed;
}

.loading {
  display: flex;
  justify-content: center;
  margin: 10px 0;
}

.typing-indicator {
  display: flex;
  gap: 4px;
}

.typing-indicator span {
  width: 8px;
  height: 8px;
  background: #007AFF;
  border-radius: 50%;
  animation: bounce 1.4s infinite ease-in-out;
}

.typing-indicator span:nth-child(1) { animation-delay: -0.32s; }
.typing-indicator span:nth-child(2) { animation-delay: -0.16s; }

@keyframes bounce {
  0%, 80%, 100% { transform: scale(0); }
  40% { transform: scale(1); }
}

/* 滚动条样式 */
.messages::-webkit-scrollbar {
  width: 6px;
}

.messages::-webkit-scrollbar-track {
  background: #f1f1f1;
}

.messages::-webkit-scrollbar-thumb {
  background: #888;
  border-radius: 3px;
}

.messages::-webkit-scrollbar-thumb:hover {
  background: #555;
}

@media (max-width: 768px) {
  .chat-container {
    height: 100vh;
    border-radius: 0;
  }

  .message-content {
    width: 85%;
  }

  .input-area {
    padding: 10px;
  }
}

/* 修改 Markdown 样式 */
.markdown-body {
  font-size: 14px;
  line-height: 1.6;
}

.markdown-body :deep(pre) {
  background-color: #f6f8fa;
  border-radius: 8px;
  padding: 16px;
  overflow: auto;
  margin: 8px 0;
}

.markdown-body :deep(code) {
  font-family: ui-monospace, SFMono-Regular, SF Mono, Menlo, Consolas, Liberation Mono, monospace;
  font-size: 13px;
  padding: 0.2em 0.4em;
  margin: 0;
  border-radius: 6px;
}

.markdown-body :deep(pre code) {
  background-color: transparent;
  padding: 0;
  margin: 0;
  font-size: 13px;
  white-space: pre;
  word-break: normal;
  overflow-wrap: normal;
}

.markdown-body :deep(p) {
  margin: 8px 0;
  line-height: 1.6;
}

.markdown-body :deep(ul), 
.markdown-body :deep(ol) {
  padding-left: 2em;
  margin: 8px 0;
}

.markdown-body :deep(li) {
  margin: 4px 0;
}

.markdown-body :deep(h1),
.markdown-body :deep(h2),
.markdown-body :deep(h3),
.markdown-body :deep(h4),
.markdown-body :deep(h5),
.markdown-body :deep(h6) {
  margin: 16px 0 8px;
  font-weight: 600;
  line-height: 1.25;
}

.markdown-body :deep(h1) { font-size: 1.5em; }
.markdown-body :deep(h2) { font-size: 1.3em; }
.markdown-body :deep(h3) { font-size: 1.1em; }

.markdown-body :deep(table) {
  border-collapse: collapse;
  width: 100%;
  margin: 8px 0;
  display: block;
  overflow-x: auto;
}

.markdown-body :deep(th),
.markdown-body :deep(td) {
  padding: 6px 13px;
  border: 1px solid #d0d7de;
}

.markdown-body :deep(tr:nth-child(2n)) {
  background-color: #f6f8fa;
}

.markdown-body :deep(blockquote) {
  margin: 8px 0;
  padding: 0 1em;
  color: #656d76;
  border-left: 0.25em solid #d0d7de;
}

.markdown-body :deep(hr) {
  height: 0.25em;
  padding: 0;
  margin: 16px 0;
  background-color: #d0d7de;
  border: 0;
}

/* 代码高亮主题样式 */
.markdown-body :deep(.hljs) {
  display: block;
  overflow-x: auto;
  padding: 1em;
  color: #24292e;
  background: #f6f8fa;
}

.markdown-body :deep(.hljs-comment),
.markdown-body :deep(.hljs-quote) {
  color: #6a737d;
  font-style: italic;
}

.markdown-body :deep(.hljs-keyword),
.markdown-body :deep(.hljs-selector-tag) {
  color: #d73a49;
}

.markdown-body :deep(.hljs-string),
.markdown-body :deep(.hljs-attr) {
  color: #032f62;
}

.markdown-body :deep(.hljs-number),
.markdown-body :deep(.hljs-literal) {
  color: #005cc5;
}

.markdown-body :deep(.hljs-title),
.markdown-body :deep(.hljs-class) {
  color: #6f42c1;
}

/* 调整消息内容样式 */
.assistant .message-content {
  padding: 16px;
  background: white;
}

.user .message-content {
  padding: 16px;
}

/* 添加新的样式 */
.header-buttons {
  display: flex;
  gap: 10px;
}

.stop-button {
  padding: 8px 16px;
  background: #666;
  color: white;
  border: none;
  border-radius: 6px;
  font-size: 14px;
  cursor: pointer;
  transition: all 0.3s ease;
}

.stop-button:hover {
  background: #555;
}

.assistant-message {
  position: relative;
  padding-right: 32px; /* 减小右侧padding */
}

.play-button {
  position: absolute;
  right: -32px;
  top: 50%;
  transform: translateY(-50%) scale(0.9); /* 默认稍微缩小 */
  padding: 4px 8px;  /* 减小内边距 */
  background: #007AFF;
  color: white;
  border: none;
  border-radius: 4px;
  font-size: 11px;  /* 减小字体大小 */
  cursor: pointer;
  opacity: 0;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1); /* 添加平滑过渡 */
  transform-origin: center;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.message:hover .play-button {
  opacity: 1;
  transform: translateY(-50%) scale(1); /* 悬停时恢复大小 */
}

.play-button:hover {
  background: #0056b3;
  transform: translateY(-50%) scale(1.05); /* 悬停时稍微放大 */
  box-shadow: 0 3px 6px rgba(0, 0, 0, 0.15);
}

.play-button:active {
  transform: translateY(-50%) scale(0.95); /* 点击时缩小 */
  background: #004494;
}

/* 添加播放状态的动画 */
@keyframes pulse {
  0% { box-shadow: 0 0 0 0 rgba(0, 122, 255, 0.4); }
  70% { box-shadow: 0 0 0 6px rgba(0, 122, 255, 0); }
  100% { box-shadow: 0 0 0 0 rgba(0, 122, 255, 0); }
}

.play-button[data-playing="true"] {
  animation: pulse 2s infinite;
  background: #ff4757; /* 播放时改变颜色 */
}
</style> 