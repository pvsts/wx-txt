<template>
  <div class="fixed-viewport">
    <div class="wx-main">
      <header class="header">
        <div class="title">软件功能记忆库</div>
        <div class="search-bar">
          <input v-model="searchQuery" type="text" placeholder="🔍 搜索软件名或功能..." />
        </div>
      </header>

      <main ref="chatContainer" class="chat-content" @click="closeMenu">
        <div v-for="(item, index) in filteredList" :key="item.id" class="msg-group">
          <div v-if="shouldShowTime(item, filteredList[index - 1])" class="time-tag">
            {{ formatDate(item.created_at) }}
          </div>
          
          <div class="msg-row right">
            <div class="bubble-container">
              <div v-if="menuVisible && activeItem?.id === item.id" class="pop-menu" :style="{ top: '-45px', right: '0' }">
                <div @click.stop="toEdit">修改内容</div>
                <div class="del" @click.stop="toDelete">删除记录</div>
              </div>

              <div class="bubble" 
                   @contextmenu.prevent="openMenu($event, item)" 
                   @touchstart="handleTouchStart($event, item)"
                   @touchend="handleTouchEnd"
                   @touchmove="handleTouchEnd">
                <div class="app-tag">
                  # 
                  <a v-if="item.url" :href="item.url" target="_blank" @click.stop class="app-link">{{ item.name }}</a>
                  <span v-else>{{ item.name }}</span>
                </div>
                <div class="feature-text">{{ item.feature }}</div>
              </div>
            </div>
            <div class="my-avatar">Me</div>
          </div>
        </div>
        <div v-if="searchQuery && filteredList.length === 0" class="empty-tip">未找到相关记录</div>
      </main>

      <footer class="footer">
        <div class="input-row">
          <textarea 
            ref="inputBox"
            v-model="inputText" 
            @input="autoHeight"
            @keydown.enter.prevent="handleEnter"
            placeholder="软件名: 功能内容"
          ></textarea>
          <button @click="sendData" :class="{ 'active': inputText.trim() }">发送</button>
        </div>
      </footer>

      <div v-if="showModal" class="modal-mask">
        <div class="modal-box">
          <h4>{{ modalType === 'edit' ? '编辑记录' : '管理验证' }}</h4>
          
          <div v-if="modalType === 'edit'" class="edit-fields">
            <input v-model="form.name" placeholder="软件名" class="modal-input" />
            <input v-model="form.url" placeholder="跳转链接 (需 http://)" class="modal-input" />
            <textarea v-model="form.feature" placeholder="描述内容" class="modal-input-area"></textarea>
          </div>

          <div class="auth-section">
            <input v-model="pwd" type="password" placeholder="请输入身份验证密码" class="pwd-input" autocomplete="new-password" />
            <label class="remember-pwd">
              <input type="checkbox" v-model="rememberPwd" /> 记住密码
            </label>
          </div>

          <div class="btns">
            <button @click="showModal = false" class="btn-cancel">取消</button>
            <button @click="doSubmit" class="btn-confirm">确定</button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, computed, nextTick } from 'vue'

// 【关键】替换为你自己的 Cloudflare Worker 域名
const API_URL = 'https://sql.xxsxxs.top' 

const list = ref([])
const searchQuery = ref('')
const inputText = ref('')
const chatContainer = ref(null)
const inputBox = ref(null)

const menuVisible = ref(false)
const activeItem = ref(null)
const showModal = ref(false)
const modalType = ref('')
const pwd = ref('')
const rememberPwd = ref(localStorage.getItem('wx_remember') === 'true')
const form = ref({ name: '', feature: '', url: '' })

let tTimer = null

const filteredList = computed(() => {
  const q = searchQuery.value.toLowerCase()
  return list.value.filter(i => i.name.toLowerCase().includes(q) || i.feature.toLowerCase().includes(q))
})

// --- API 请求逻辑 ---

const fetchData = async () => {
  try {
    const res = await fetch(API_URL)
    const data = await res.json()
    list.value = data
    await scrollDown()
  } catch (e) {
    console.error('获取失败:', e)
  }
}

const sendData = async () => {
  const t = inputText.value.trim(); if (!t) return
  let n = '未分类', f = t
  if (t.includes(':')) [n, f] = t.split(':')
  else if (t.includes('：')) [n, f] = t.split('：')
  
  try {
    const res = await fetch(API_URL, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ name: n.trim(), feature: f.trim() })
    })
    if (res.ok) {
      inputText.value = ''; 
      await nextTick(); 
      autoHeight(); 
      fetchData(); 
    }
  } catch (e) {
    alert('发送失败')
  }
}

const doSubmit = async () => {
  if (pwd.value !== 'admin@123') return alert('身份验证失败')
  if (rememberPwd.value) {
    localStorage.setItem('wx_pwd', pwd.value); localStorage.setItem('wx_remember', 'true')
  }

  const id = activeItem.value.id
  try {
    if (modalType.value === 'del') {
      await fetch(`${API_URL}?id=${id}`, { method: 'DELETE' })
    } else {
      await fetch(`${API_URL}?id=${id}`, {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ 
          name: form.value.name, 
          feature: form.value.feature, 
          url: form.value.url 
        })
      })
    }
    showModal.value = false
    fetchData()
  } catch (e) {
    alert('操作失败')
  }
}

// --- UI 交互逻辑 ---

const autoHeight = () => {
  const el = inputBox.value
  if (!el) return
  el.style.height = '34px'
  if (inputText.value.trim()) el.style.height = Math.min(el.scrollHeight, 150) + 'px'
}

const shouldShowTime = (current, previous) => {
  if (!previous) return true
  const currTime = new Date(current.created_at).getTime()
  const prevTime = new Date(previous.created_at).getTime()
  return (currTime - prevTime) > 5 * 60 * 1000
}

const formatDate = (s) => { 
  const d = new Date(s); const now = new Date();
  const isToday = d.toDateString() === now.toDateString();
  const timeStr = `${d.getHours()}:${String(d.getMinutes()).padStart(2, '0')}`;
  return isToday ? timeStr : `${d.getMonth()+1}月${d.getDate()}日 ${timeStr}`;
}

const handleEnter = (e) => { if (!e.shiftKey) sendData() }
const scrollDown = async () => { await nextTick(); if (chatContainer.value) chatContainer.value.scrollTop = chatContainer.value.scrollHeight }
const openMenu = (e, item) => { activeItem.value = item; menuVisible.value = true }
const handleTouchStart = (e, item) => { tTimer = setTimeout(() => { openMenu(null, item) }, 800) }
const handleTouchEnd = () => { clearTimeout(tTimer) }
const closeMenu = () => { menuVisible.value = false }
const toDelete = () => { modalType.value = 'del'; openAuth(); menuVisible.value = false }
const toEdit = () => { 
  modalType.value = 'edit';
  form.value = { name: activeItem.value.name, feature: activeItem.value.feature, url: activeItem.value.url || '' };
  openAuth(); menuVisible.value = false 
}
const openAuth = () => {
  pwd.value = rememberPwd.value ? (localStorage.getItem('wx_pwd') || '') : '';
  showModal.value = true;
}

onMounted(() => { fetchData(); nextTick(() => autoHeight()); })
</script>

<style scoped>
/* 样式部分保持不变，确保 UI 体验一致 */
.fixed-viewport { position: fixed; inset: 0; background: #1a1a1a; display: flex; justify-content: center; overflow: hidden; }
.wx-main { width: 375px; height: 100%; background: #f3f3f3; display: flex; flex-direction: column; overflow: hidden; position: relative; }
.header { background: #ededed; padding: 10px 15px; border-bottom: 1px solid #ddd; flex-shrink: 0; }
.title { text-align: center; font-weight: bold; margin-bottom: 8px; font-size: 15px; }
.search-bar input { width: 100%; border: none; padding: 7px 10px; border-radius: 6px; outline: none; font-size: 13px; box-sizing: border-box; }
.chat-content { flex: 1; overflow-y: auto; padding: 10px 15px; background: #f3f3f3; }
.msg-group { margin-bottom: 8px; }
.time-tag { text-align: center; color: #aaa; font-size: 11px; margin: 15px 0; }
.msg-row { display: flex; justify-content: flex-end; align-items: flex-start; position: relative; }
.bubble-container { position: relative; max-width: 260px; }
.my-avatar { width: 38px; height: 38px; background: #07c160; color: white; border-radius: 4px; display: flex; align-items: center; justify-content: center; font-size: 12px; margin-left: 10px; flex-shrink: 0; }
.bubble { background: #95ec69; padding: 10px 12px; border-radius: 6px; position: relative; font-size: 15px; border: 1px solid #83d45a; word-break: break-all; -webkit-tap-highlight-color: transparent; }
.bubble::after { content: ""; position: absolute; right: -6px; top: 12px; border: 6px solid transparent; border-left-color: #95ec69; }
.app-tag { font-weight: bold; font-size: 13px; color: #2c3e50; border-bottom: 1px dashed rgba(0,0,0,0.1); margin-bottom: 6px; padding-bottom: 4px; }
.app-link { color: #0055ff; text-decoration: underline; cursor: pointer; }
.feature-text { line-height: 1.5; color: #000; white-space: pre-wrap; }
.pop-menu { position: absolute; background: #4c4c4c; color: white; border-radius: 6px; z-index: 999; display: flex; flex-direction: column; box-shadow: 0 4px 10px rgba(0,0,0,0.3); width: 100px; }
.pop-menu div { padding: 12px; font-size: 13px; text-align: center; border-bottom: 1px solid #5a5a5a; cursor: pointer; }
.pop-menu .del { color: #ff5b5b; border-bottom: none; }
.footer { background: #f7f7f7; padding: 10px 15px 35px; border-top: 1px solid #ddd; flex-shrink: 0; }
.input-row { display: flex; align-items: flex-end; background: white; border-radius: 4px; padding: 6px 10px; border: 1px solid #e0e0e0; }
.input-row textarea { flex: 1; border: none; outline: none; resize: none; font-size: 16px; line-height: 24px; min-height: 24px; height: 34px; }
.input-row button { background: #e1e1e1; color: #b2b2b2; border: none; padding: 6px 16px; border-radius: 4px; margin-left: 10px; font-size: 14px; cursor: pointer; }
.input-row .active { background: #07c160; color: white; }
.modal-mask { position: fixed; inset: 0; background: rgba(0,0,0,0.7); display: flex; align-items: center; justify-content: center; z-index: 2000; }
.modal-box { background: white; width: 310px; border-radius: 12px; padding: 22px; }
.modal-box h4 { text-align: center; margin-bottom: 20px; }
.modal-input, .modal-input-area, .pwd-input { width: 100%; margin-bottom: 12px; padding: 10px; border: 1px solid #ddd; border-radius: 6px; box-sizing: border-box; font-size: 14px; outline: none; }
.modal-input-area { height: 90px; resize: none; }
.remember-pwd { display: flex; align-items: center; font-size: 12px; color: #666; gap: 5px; margin-bottom: 15px; cursor: pointer; }
.btns { display: flex; gap: 12px; }
.btns button { flex: 1; padding: 11px; border-radius: 6px; border: none; font-size: 14px; font-weight: bold; cursor: pointer; }
.btn-cancel { background: #f0f0f0; color: #666; }
.btn-confirm { background: #07c160; color: white; }
.empty-tip { text-align: center; color: #999; margin-top: 50px; font-size: 14px; }
</style>