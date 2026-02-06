<template>
  <div class="card">
    <h1>HTML 链接下载工具</h1>
    <p class="description">输入多个链接（每行一个），下载这些链接的 HTML 内容到一个 ZIP 文件中。</p>
    
    <div class="form-group">
      <label for="proxy">代理服务（解决跨域问题）：</label>
      <select id="proxy" v-model="selectedProxy">
        <option v-for="service in proxyServices" :key="service.name" :value="service.url">
          {{ service.name }}
        </option>
      </select>
      <p class="hint">如果下载失败，请尝试切换不同的代理服务。</p>
    </div>

    <div class="form-group">
      <label for="urls">链接列表：</label>
      <textarea 
        id="urls"
        v-model="urlsText" 
        placeholder="每行输入一个链接，例如：&#10;https://example.com&#10;https://github.com&#10;https://www.google.com"
      ></textarea>
    </div>

    <div class="actions">
      <button @click="generateZip" :disabled="isLoading || !urlsText.trim()">
        <span v-if="!isLoading">生成 ZIP</span>
        <span v-else>处理中... ({{ progress.current }}/{{ progress.total }})</span>
      </button>

      <a 
        v-if="downloadUrl" 
        :href="downloadUrl" 
        :download="downloadFilename" 
        class="download-btn"
        @click="addLog('✓ 已开始下载文件', 'success')"
      >
        点击下载 {{ downloadFilename }}
      </a>
    </div>

    <div v-if="isLoading && progress.total > 0" class="progress-container">
      <div 
        class="progress-bar" 
        :style="{ width: `${(progress.current / progress.total) * 100}%` }"
      ></div>
    </div>

    <div v-if="isLoading" class="file-progress">
      <div v-if="currentFileProgress.loaded > 0" class="file-progress-text">
        <span v-if="currentFileProgress.total > 0">
          正在下载当前文件: {{ currentFileProgress.percent }}% ({{ formatBytes(currentFileProgress.loaded) }} / {{ formatBytes(currentFileProgress.total) }})
        </span>
        <span v-else>
          正在下载当前文件: {{ formatBytes(currentFileProgress.loaded) }}
        </span>
      </div>
    </div>

    <div v-if="statusMessage" class="status" :class="statusType">
      {{ statusMessage }}
    </div>

    <div v-if="logs.length > 0" class="logs">
      <h3>处理日志：</h3>
      <div v-for="(log, index) in logs" :key="index" class="log-item" :class="log.type">
        {{ log.message }}
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import JSZip from 'jszip'

const urlsText = ref('')
const isLoading = ref(false)
const statusMessage = ref('')
const statusType = ref('')
const logs = ref([])
const progress = ref({ current: 0, total: 0 })
const currentFileProgress = ref({ loaded: 0, total: 0, percent: 0 })
const downloadUrl = ref(null)
const downloadFilename = ref('')

const proxyServices = [
  { name: 'CORS Proxy.io (推荐)', url: 'https://corsproxy.io/?' },
  { name: 'AllOrigins', url: 'https://api.allorigins.win/raw?url=' },
  { name: 'ThingProxy', url: 'https://thingproxy.freeboard.io/fetch/' },
  { name: '不使用代理 (直连)', url: '' }
]
const selectedProxy = ref(proxyServices[0].url)

const formatBytes = (bytes) => {
  if (bytes === 0) return '0 B'
  const k = 1024
  const sizes = ['B', 'KB', 'MB', 'GB']
  const i = Math.floor(Math.log(bytes) / Math.log(k))
  return parseFloat((bytes / Math.pow(k, i)).toFixed(2)) + ' ' + sizes[i]
}

const addLog = (message, type = 'info') => {
  logs.value.push({ message, type })
}

const setStatus = (message, type = 'info') => {
  statusMessage.value = message
  statusType.value = type
}

const fetchHtmlContent = async (url, onProgress) => {
  // 使用选定的代理服务
  // 如果选择了空字符串，则尝试直连（可能会跨域失败）
  let targetUrl = url
  if (selectedProxy.value) {
    // 处理 corsproxy.io 的特殊情况，它直接拼接 url，而其他通常是 ?url= 参数
    // 但 corsproxy.io/?url 也是支持的，所以统一处理
    // 注意：corsproxy.io 需要对 url 进行编码吗？
    // 官方示例是 https://corsproxy.io/?https%3A%2F%2Fgoogle.com
    // 所以统一使用 encodeURIComponent 是安全的
    targetUrl = `${selectedProxy.value}${encodeURIComponent(url)}`
    
    // 特殊修正：corsproxy.io 实际上支持不编码直接拼接，但也支持编码
    // 如果是 corsproxy.io，我们尝试使用 encodeURIComponent
  }
  
  try {
    const response = await fetch(targetUrl)
    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`)
    }

    if (!response.body) {
      return await response.text()
    }

    const reader = response.body.getReader()
    const contentLength = +response.headers.get('Content-Length')
    let receivedLength = 0
    let chunks = []

    while(true) {
      const {done, value} = await reader.read()
      if (done) break
      
      chunks.push(value)
      receivedLength += value.length
      
      if (onProgress) {
        onProgress(receivedLength, contentLength)
      }
    }

    const chunksAll = new Uint8Array(receivedLength)
    let position = 0
    for(let chunk of chunks) {
      chunksAll.set(chunk, position)
      position += chunk.length
    }

    return new TextDecoder("utf-8").decode(chunksAll)
  } catch (error) {
    throw new Error(`代理请求失败: ${error.message}`)
  }
}

const getFilenameFromUrl = (url) => {
  try {
    const urlObj = new URL(url)
    let filename = urlObj.hostname + urlObj.pathname
    filename = filename.replace(/\//g, '_')
    if (!filename.endsWith('.html')) {
      filename += '.html'
    }
    return filename
  } catch {
    return `page_${Date.now()}.html`
  }
}

const generateZip = async () => {
  if (downloadUrl.value) {
    URL.revokeObjectURL(downloadUrl.value)
    downloadUrl.value = null
    downloadFilename.value = ''
  }

  const urls = urlsText.value
    .split('\n')
    .map(url => url.trim())
    .filter(url => url && (url.startsWith('http://') || url.startsWith('https://')))

  if (urls.length === 0) {
    setStatus('请输入至少一个有效的链接（需以 http:// 或 https:// 开头）', 'error')
    return
  }

  isLoading.value = true
  logs.value = []
  progress.value = { current: 0, total: urls.length }
  setStatus('', '')

  const zip = new JSZip()
  let successCount = 0
  let failCount = 0

  for (let i = 0; i < urls.length; i++) {
    const url = urls[i]
    progress.value.current = i + 1
    
    try {
      addLog(`正在获取: ${url}`, 'info')
      currentFileProgress.value = { loaded: 0, total: 0, percent: 0 }
      
      const html = await fetchHtmlContent(url, (loaded, total) => {
        let percent = 0
        if (total > 0) {
          percent = Math.floor((loaded / total) * 100)
        }
        currentFileProgress.value = { loaded, total, percent }
      })
      
      const filename = getFilenameFromUrl(url)
      zip.file(filename, html)
      successCount++
      addLog(`✓ 成功: ${url} -> ${filename}`, 'success')
    } catch (error) {
      failCount++
      addLog(`✗ 失败: ${url} - ${error.message}`, 'error')
    }
  }

  if (successCount > 0) {
    try {
      addLog('正在生成 ZIP 文件...', 'info')
      const content = await zip.generateAsync({ type: 'blob' })
      
      const dateStr = new Date().toISOString().replace(/[:.]/g, '-').slice(0, 19)
      downloadFilename.value = `html_links_${dateStr}.zip`
      downloadUrl.value = URL.createObjectURL(content)
      
      setStatus(`生成成功！已准备好下载链接（${successCount} 个页面${failCount > 0 ? `，${failCount} 个失败` : ''}）`, 'success')
      addLog('✓ ZIP 文件生成成功，请点击下载按钮', 'success')
    } catch (error) {
      setStatus('生成 ZIP 文件失败: ' + error.message, 'error')
      addLog(`✗ ZIP 生成失败: ${error.message}`, 'error')
    }
  } else {
    setStatus('所有链接都获取失败，请检查链接', 'error')
  }

  isLoading.value = false
}
</script>

<style scoped>
h1 {
  color: #333;
  margin-bottom: 10px;
  font-size: 2em;
}

.description {
  color: #666;
  margin-bottom: 25px;
  font-size: 1.1em;
}

.form-group {
  margin-bottom: 20px;
}

label {
  display: block;
  margin-bottom: 8px;
  font-weight: 500;
  color: #555;
}

.options {
  margin-bottom: 20px;
}

.options label {
  display: flex;
  align-items: center;
  gap: 8px;
  font-weight: normal;
  cursor: pointer;
}

.options input[type="checkbox"] {
  cursor: pointer;
  width: 18px;
  height: 18px;
}

select {
  width: 100%;
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 1em;
  background-color: white;
  margin-bottom: 5px;
}

.hint {
  font-size: 0.85em;
  color: #888;
  margin: 0;
}

.actions {
  margin-bottom: 20px;
}

.download-btn {
  display: inline-block;
  padding: 8px 16px;
  background-color: #28a745;
  color: white;
  text-decoration: none;
  border-radius: 4px;
  margin-left: 10px;
  font-size: 0.9em;
  transition: background-color 0.2s;
}

.download-btn:hover {
  background-color: #218838;
}

.file-progress {
  margin-top: -15px;
  margin-bottom: 20px;
  text-align: center;
  font-size: 0.9em;
  color: #666;
}

.file-progress-text {
  background-color: #f8f9fa;
  padding: 4px 8px;
  border-radius: 4px;
  display: inline-block;
}

.status {
  padding: 12px;
  border-radius: 8px;
  margin-bottom: 20px;
  font-weight: 500;
}

.status.success {
  background-color: #d4edda;
  color: #155724;
  border: 1px solid #c3e6cb;
}

.status.error {
  background-color: #f8d7da;
  color: #721c24;
  border: 1px solid #f5c6cb;
}

.status.info {
  background-color: #d1ecf1;
  color: #0c5460;
  border: 1px solid #bee5eb;
}

.logs {
  margin-top: 20px;
  padding-top: 20px;
  border-top: 1px solid #ddd;
}

.logs h3 {
  margin-bottom: 10px;
  font-size: 1.2em;
  color: #555;
}

.log-item {
  padding: 8px 12px;
  margin-bottom: 6px;
  border-radius: 4px;
  font-family: 'Consolas', 'Monaco', monospace;
  font-size: 0.9em;
}

.log-item.info {
  background-color: #e7f3ff;
  color: #004085;
}

.log-item.success {
  background-color: #d4edda;
  color: #155724;
}

.log-item.error {
  background-color: #f8d7da;
  color: #721c24;
}

.progress-container {
  width: 100%;
  height: 10px;
  background-color: #e9ecef;
  border-radius: 5px;
  margin-bottom: 20px;
  overflow: hidden;
}

.progress-bar {
  height: 100%;
  background-color: #28a745;
  transition: width 0.3s ease;
}

@media (prefers-color-scheme: dark) {
  h1 {
    color: #fff;
  }
  
  .description,
  label,
  .logs h3 {
    color: #ccc;
  }
  
  .status.success {
    background-color: #1e4620;
    color: #9fdf9f;
    border-color: #2d5930;
  }
  
  .status.error {
    background-color: #4a1c1c;
    color: #f19ca0;
    border-color: #6b2c2c;
  }
  
  .status.info {
    background-color: #1c3d4a;
    color: #9fccdf;
    border-color: #2c5369;
  }
  
  .log-item.info {
    background-color: #1c3d4a;
    color: #9fccdf;
  }
  
  .log-item.success {
    background-color: #1e4620;
    color: #9fdf9f;
  }
  
  .log-item.error {
    background-color: #4a1c1c;
    color: #f19ca0;
  }
  
  .logs {
    border-top-color: #444;
  }

  .progress-container {
    background-color: #444;
  }
  
  .progress-bar {
    background-color: #2ea043;
  }
  
  select {
    background-color: #333;
    color: #eee;
    border-color: #555;
  }
}
</style>
