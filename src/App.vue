<template>
  <div class="card">
    <h1>HTML 链接下载工具</h1>
    <p class="description">输入多个链接（每行一个），下载这些链接的 HTML 内容到一个 ZIP 文件中。</p>
    
    <div class="form-group">
      <label for="urls">链接列表：</label>
      <textarea 
        id="urls"
        v-model="urlsText" 
        placeholder="每行输入一个链接，例如：&#10;https://example.com&#10;https://github.com&#10;https://www.google.com"
      ></textarea>
    </div>

    <div class="actions">
      <button @click="downloadZip" :disabled="isLoading || !urlsText.trim()">
        <span v-if="!isLoading">下载 ZIP</span>
        <span v-else>处理中... ({{ progress.current }}/{{ progress.total }})</span>
      </button>
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

const addLog = (message, type = 'info') => {
  logs.value.push({ message, type })
}

const setStatus = (message, type = 'info') => {
  statusMessage.value = message
  statusType.value = type
}

const fetchHtmlContent = async (url) => {
  return new Promise((resolve, reject) => {
    const callbackName = `jsonp_callback_${Date.now()}_${Math.floor(Math.random() * 100000)}`
    
    window[callbackName] = (data) => {
      delete window[callbackName]
      document.body.removeChild(script)
      if (data && data.contents) {
        resolve(data.contents)
      } else {
        reject(new Error('No content received'))
      }
    }

    const script = document.createElement('script')
    script.src = `https://api.allorigins.win/get?url=${encodeURIComponent(url)}&callback=${callbackName}`
    script.onerror = () => {
      delete window[callbackName]
      document.body.removeChild(script)
      reject(new Error('Cross-domain request failed via script tag'))
    }
    
    document.body.appendChild(script)
  })
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

const downloadZip = async () => {
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
      const html = await fetchHtmlContent(url)
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
      
      // Create download link
      const link = document.createElement('a')
      link.href = URL.createObjectURL(content)
      const dateStr = new Date().toISOString().replace(/[:.]/g, '-').slice(0, 19)
      link.download = `html_links_${dateStr}.zip`
      link.click()
      
      setStatus(`成功！已下载 ${successCount} 个页面${failCount > 0 ? `，${failCount} 个失败` : ''}`, 'success')
      addLog('✓ ZIP 文件已生成并开始下载', 'success')
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

.actions {
  margin-bottom: 20px;
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
}
</style>
