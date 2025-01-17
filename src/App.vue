<script setup>
import { ref, onMounted, computed, shallowRef, onUnmounted } from 'vue'
import { menuItemsList, authorWorksList, onlineWorksList } from '@/utlis/menuItems'
import { ElDialog, ElMessageBox } from 'element-plus'
import { Message, Timer } from '@element-plus/icons-vue'
import SokobanGame from './components/games/SokobanGame.vue'
import ImageEditor from './components/image/ImageEditor.vue'
import MusicPlayer from "./components/MusicPlayer.vue"
import DyForm from './views/DyForm.vue'
import AnalogClock from './components/AnalogClock.vue'
import FruitCatcher from './components/games/FruitCatcher.vue'

// 判断是否为生产环境
const isProd = import.meta.env.PROD

const menuItems = ref(menuItemsList)

// 从本地存储初始化激活的菜单项
const activeItem = ref(parseInt(localStorage.getItem('activeItem')) || 1)
const isSidebarOpen = ref(false)
const isDarkMode = ref(localStorage.getItem('theme') === 'dark')
const showThemeDropdown = ref(false)
const showAuthorDropdown = ref(false)
const showOnlineWorksDropdown = ref(false)
const authorWorks = ref(authorWorksList)
const onlineWorks = ref(onlineWorksList)

// 从本地存储初始化点赞信息
const likedItemsInfo = ref(JSON.parse(localStorage.getItem('likedItemsInfo') || '{}'))

const toggleLike = (itemId) => {
  // 找到当前工具所属的菜单
  const currentMenu = menuItems.value.find(menu => 
    menu.tools && menu.tools.some(tool => tool.id === itemId)
  )
  
  if (likedItemsInfo.value[itemId]) {
    delete likedItemsInfo.value[itemId]
  } else if (currentMenu) {
    likedItemsInfo.value[itemId] = {
      menuName: currentMenu.name,
      menuIcon: currentMenu.icon,
      timestamp: new Date().getTime()
    }
  }
  
  // 保存到本地存储
  localStorage.setItem('likedItemsInfo', JSON.stringify(likedItemsInfo.value))

  // 添加果冻动画效果
  const heart = document.querySelector(`.heart-icon-${itemId}`)
  heart.classList.add('jelly')
  setTimeout(() => {
    heart.classList.remove('jelly')
  }, 600)
}

// 检查是否已点赞
const isLiked = (itemId) => {
  return !!likedItemsInfo.value[itemId]
}


const selectItem = (itemId) => {
  activeItem.value = itemId
  // 保存选中状态到本地存储
  localStorage.setItem('activeItem', itemId.toString())
}

const getCurrentTools = () => {
  const item = menuItems.value.find(item => item.id === activeItem.value)
  return item ? item.tools : []
}

const openLink = (link) => {
  if (link) {
    window.open(link, '_blank')
  }
}

const toggleTheme = () => {
  isDarkMode.value = !isDarkMode.value
  localStorage.setItem('theme', isDarkMode.value ? 'dark' : 'light')
  document.documentElement.classList.toggle('dark', isDarkMode.value)
}

const contextMenu = ref({
  show: false,
  x: 0,
  y: 0,
  tool: null
})

const handleRightClick = (event, tool) => {
  event.preventDefault()
  contextMenu.value = {
    show: true,
    x: event.clientX,
    y: event.clientY,
    tool
  }
}

const closeContextMenu = () => {
  contextMenu.value.show = false
}

const copyLink = () => {
  if (contextMenu.value.tool) {
    navigator.clipboard.writeText(contextMenu.value.tool.link)
    closeContextMenu()
  }
}

const openInNewTab = () => {
  if (contextMenu.value.tool) {
    window.open(contextMenu.value.tool.link, '_blank')
    closeContextMenu()
  }
}

const toggleAuthorDropdown = () => {
  showAuthorDropdown.value = !showAuthorDropdown.value
  showThemeDropdown.value = false
  showOnlineWorksDropdown.value = false
}

const toggleThemeDropdown = () => {
  showThemeDropdown.value = !showThemeDropdown.value
  showAuthorDropdown.value = false
  showOnlineWorksDropdown.value = false
}

const toggleOnlineWorksDropdown = () => {
  showOnlineWorksDropdown.value = !showOnlineWorksDropdown.value
  showThemeDropdown.value = false
  showAuthorDropdown.value = false
}

const searchQuery = ref('')
const clearSearch = () => {
  searchQuery.value = ''
}
const filteredTools = computed(() => {
  if (!searchQuery.value) return getCurrentTools()
  const query = searchQuery.value.toLowerCase()
  return getCurrentTools().filter(tool => 
    tool.name.toLowerCase().includes(query) || 
    tool.desc.toLowerCase().includes(query)
  )
})

// 获取已点赞的工具列表（包含菜单信息）
const likedToolsList = computed(() => {
  const allTools = menuItems.value.reduce((acc, item) => {
    return acc.concat(item.tools || [])
  }, [])
  return allTools
    .filter(tool => likedItemsInfo.value[tool.id])
    .map(tool => ({
      ...tool,
      menuInfo: likedItemsInfo.value[tool.id]
    }))
})

// 打开历史记录
const openLikeHistory = () => {
  showLikeHistory.value = true
}

// 关闭历史记录
const closeLikeHistory = () => {
  showLikeHistory.value = false
}

// 清空所有点赞
const clearAllLikes = () => {
  ElMessageBox.confirm(
    `确定要清空历史爱心记录<span style="color: #ff4757; font-weight: bold; font-size: 16px; margin: 0 4px; font-weight: 800; font-stretch: expanded;">${likedToolsList.value.length}</span>条点赞记录吗？`,
    '提示',
    {
      confirmButtonText: '确定',
      cancelButtonText: '取消',
      type: 'warning',
      dangerouslyUseHTMLString: true,
      customClass: 'custom-message-box'
    }
  )
    .then(() => {
      likedItemsInfo.value = {}
      localStorage.setItem('likedItemsInfo', '{}')
    })
    .catch(() => {})
}

const showGameDialog = ref(false)
const currentGame = shallowRef(null)
const gameTitle = ref('')

const handleCloseDialog = (done) => {
  const currentItem = menuItems.value.find(item => item.id === activeItem.value)
  let message = '确定要退出吗？'
  
  // 根据type显示不同的提示信息
  if (currentItem.type === 'game') {
    message = '确定要退出游戏吗？'
  } else if (currentItem.type === 'image') {
    message = '确定要退出图片操作吗？'
  }

  ElMessageBox.confirm(message, '提示', {
    confirmButtonText: '确定',
    cancelButtonText: '取消',
    type: 'warning',
  })
    .then(() => {
      done()
    })
    .catch(() => {})
}

const openGame = (work) => {
  showGameDialog.value = true
  gameTitle.value = work.name
  
  switch(work.type) {
    case 'game':
      currentGame.value = SokobanGame
      break
    case 'image':
      currentGame.value = ImageEditor
      break
    case 'video':
      currentGame.value = MusicPlayer
      break
    case 'dyform':
      currentGame.value = DyForm
      break
    case 'fruitgame':
      currentGame.value = FruitCatcher
      break
    default:
      currentGame.value = null
  }
}

// 添加历史记录弹窗状态
const showLikeHistory = ref(false)

const currentTime = ref('')
let timer = null

const updateTime = () => {
  const now = new Date()
  const hours = String(now.getHours()).padStart(2, '0')
  const minutes = String(now.getMinutes()).padStart(2, '0')
  const seconds = String(now.getSeconds()).padStart(2, '0')
  currentTime.value = `${hours}:${minutes}:${seconds}`
}
const onClickWork=item=>{
  window.open(item.link)
}
onMounted(() => {
  const theme = localStorage.getItem('theme')
  if (theme) {
    isDarkMode.value = theme === 'dark'
    document.documentElement.classList.toggle('dark', isDarkMode.value)
  }

  // 添加点击外部关闭下拉菜单
  document.addEventListener('click', (e) => {
    const themeDropdownEl = document.querySelector('.dropdown')
    const authorDropdownEl = document.querySelectorAll('.dropdown')[1]
    const onlineWorksDropdownEl = document.querySelectorAll('.dropdown')[2]

    if (!themeDropdownEl?.contains(e.target)) {
      showThemeDropdown.value = false
    }
    if (!authorDropdownEl?.contains(e.target)) {
      showAuthorDropdown.value = false
    }
    if (!onlineWorksDropdownEl?.contains(e.target)) {
      showOnlineWorksDropdown.value = false
    }
    closeContextMenu()
  })

  // 添加全局右键事件监听
  document.addEventListener('contextmenu', (event) => {
    const toolCard = event.target.closest('.tool-card')
    if (!toolCard && isProd) {
      event.preventDefault()
      window.open('about:blank', '_blank')
    }
  })

  updateTime()
  timer = setInterval(updateTime, 1000)
})

onUnmounted(() => {
  if (timer) {
    clearInterval(timer)
  }
})
</script>

<template>
  <div id="app" class="app-container" :class="{ 'dark': isDarkMode }">
    <!-- 左侧导航栏 -->
    <nav class="sidebar">
      <div class="logo">HooksVue</div>
      <ul class="nav-list">
        <li v-for="item in menuItems" :key="item.id"
            :class="{ 'active': activeItem === item.id }"
            @click="selectItem(item.id)">
          <span class="nav-icon">{{ item.icon }}</span>
          <span>{{ item.name }}</span>
        </li>
      </ul>
    </nav>

    <!-- 主内容区域 -->
    <main class="main-content">
      <div class="header-actions">
        <div class="header-icons">
          <AnalogClock class="clock-component" />
          <a href="mailto:869710179@qq.com" class="email-icon" title="联系我">
            <el-icon><Message /></el-icon>
          </a>
        </div>
        <div class="dropdown" ref="themeDropdown">
          <button class="dropdown-trigger" @click="toggleThemeDropdown">
            {{ !isDarkMode ? '☀️' : '🌙' }} 主题
            <span class="arrow">▼</span>
          </button>
          <div class="dropdown-menu" v-show="showThemeDropdown">
            <div class="dropdown-item" @click="() => { isDarkMode = false; toggleTheme() }">
              🌙 深色模式
            </div>
            <div class="dropdown-item" @click="() => { isDarkMode = true; toggleTheme() }">

              ☀️ 浅色模式
            </div>
          </div>
        </div>

        <div class="dropdown" ref="authorDropdown">
          <button class="dropdown-trigger" @click="toggleAuthorDropdown">
            👨‍💻 作者作品集
            <span class="arrow">▼</span>
          </button>
          <div v-if="showAuthorDropdown" class="dropdown-menu">
            <div v-for="work in authorWorks" :key="work.name" class="dropdown-item"
            @click="onClickWork(work)"
            >
              <div class="dropdown-item-left-01">
                <div class="word-name">{{ work.name }}</div>
                <div class="work-desc">{{ work.desc }}</div>
              </div>
            </div>
          </div>
        </div>

        <div class="dropdown" ref="onlineWorksDropdown">
          <button class="dropdown-trigger" @click="toggleOnlineWorksDropdown">
            🎨 在线作品查看
            <span class="arrow">▼</span>
          </button>
          <div v-if="showOnlineWorksDropdown" class="dropdown-menu">
            <a v-for="work in onlineWorks"
               :key="work.name"
               :href="work.component === 'dialog' ? '#' : work.link"
               @click.prevent="work.component === 'dialog' && openGame(work)"
               target="_blank"
               class="dropdown-item">
              <div class="dropdown-item-left-01">
                <div class="word-name">{{ work.name }}</div>
                <div class="work-desc">{{ work.desc }}</div>
              </div>
            </a>
          </div>
        </div>

        <button class="dropdown-trigger like-history-btn" @click="openLikeHistory">
          ❤️ 历史爱心记录
        </button>
      </div>
      <div class="search-wrapper">
          <input
            type="text"
            v-model="searchQuery"
            placeholder="搜索工具..."
            class="search-input"
          >
          <button
            v-show="searchQuery"
            @click="clearSearch"
            class="clear-button"
            title="清除搜索"
          >
            ✕
          </button>
        </div>
      <div class="tools-grid">
        <!-- 搜索框 -->


        <!-- 工具卡片列表 -->
        <template v-if="filteredTools.length > 0">
          <div v-for="(tool, index) in filteredTools" :key="tool.id" class="tool-wrapper">
            <div class="tool-card"
                :title="`${tool.name} - ${tool.desc}`"

                @contextmenu="(event) => handleRightClick(event, tool)">
              <div class="tool-header" @click="openLink(tool.link)">
                <span class="tool-icon">{{ tool.icon }}</span>
                <h3 class="tool-name">{{ tool.name }}</h3>
                <!-- 爱心图标 -->

              </div>
              <div
                  :class="['heart-icon', `heart-icon-${tool.id}`, { 'liked': isLiked(tool.id) }]"
                  @click.stop="toggleLike(tool.id)"
                >
                  <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                    <path d="M20.84 4.61a5.5 5.5 0 0 0-7.78 0L12 5.67l-1.06-1.06a5.5 5.5 0 0 0-7.78 7.78l1.06 1.06L12 21.23l7.78-7.78 1.06-1.06a5.5 5.5 0 0 0 0-7.78z"></path>
                  </svg>
                </div>
              <div class="tool-info">
                <p>{{ tool.desc }}</p>
                <div v-if="tool.needVPN" class="vpn-tag">需要VPN</div>
              </div>
              <div class="tool-link" :title="'点击跳转: ' + tool.link" @click="openLink(tool.link)">
                <span class="link-icon">🔗</span>
              </div>
            </div>
          </div>
        </template>
        <template v-else>
          <div class="no-results">
            <span>🔍</span>
            <p>暂无搜索结果</p>
            <p>试试其他关键词吧</p>
          </div>
        </template>
      </div>
    </main>
    <!-- 自定义右键菜单 -->
    <div v-if="contextMenu.show"
         class="context-menu"
         :style="{ top: contextMenu.y + 'px', left: contextMenu.x + 'px' }">
      <div class="context-menu-item" @click="openInNewTab">
        <span class="context-menu-icon">🔗</span>
        新标签页打开
      </div>
      <div class="context-menu-item" @click="copyLink">
        <span class="context-menu-icon">📋</span>
        复制链接
      </div>
    </div>

    <!-- 游戏对话框 -->
    <el-dialog
      v-model="showGameDialog"
      :title="gameTitle"
      width="80%"
      :close-on-click-modal="false"
      :close-on-press-escape="false"
      :before-close="handleCloseDialog"
      destroy-on-close
      class="game-dialog"
    >
      <component :is="currentGame" v-if="currentGame" @close="showGameDialog = false" />
    </el-dialog>
    <!-- 邮箱图标 -->
    <a href="mailto:869710179@qq.com" class="email-icon" title="联系我">
      <i class="el-icon-message"></i>
      📧
    </a>
    <!-- 历史爱心记录弹窗 -->
    <el-dialog
      v-model="showLikeHistory"
      :title="`历史爱心记录(${likedToolsList.length})`"
      width="60%"
      destroy-on-close
      class="like-history-dialog"
    >

      <div class="like-history-header">
        <div class="like-history-title">历史记录</div>
        <button
          v-if="likedToolsList.length > 0"
          class="clear-all-btn"
          @click="clearAllLikes"
          title="清空所有点赞"
        >
          🗑️ 清空记录
        </button>
      </div>
      <div class="liked-tools-list" :class="{ 'scrollable': likedToolsList.length > 10 }">
        <div v-if="likedToolsList.length === 0" class="no-likes">
          <p>还没有点赞过任何工具哦~ 💝</p>
        </div>
        <div v-else v-for="tool in likedToolsList" :key="tool.id" class="liked-tool-item">
          <div class="liked-tool-info" @click="openLink(tool.link)">
            <span class="tool-icon">{{ tool.icon }}</span>
            <div class="tool-details">
              <h4>{{ tool.name }}</h4>
              <p>
                <span class="menu-info">{{ tool.menuInfo.menuIcon }} {{ tool.menuInfo.menuName }}</span>
                {{ tool.desc }}
              </p>
            </div>
          </div>
          <div class="liked-tool-actions">

            <button class="link-btn" @click="openLink(tool.link)" :title="`访问链接${tool.link}`">
              🔗
            </button>
            <button
              class="unlike-btn"
              @click="toggleLike(tool.id)"
              title="取消点赞"
            >
              ❤️
            </button>
            <span class="menu-info menu-menuName">来自{{ tool.menuInfo?.menuName}}</span>
          </div>
        </div>
      </div>
    </el-dialog>
  </div>
</template>

<style scoped>
@import url('@/style/style.css');

.main-content {
  flex: 1;
  padding: 20px;
  overflow-y: auto;
  height: calc(100vh - 60px);
  background: var(--el-bg-color);
}

.heart-icon {
  position: absolute;
  right: 10px;
  top: 10px;
  cursor: pointer;
  transition: transform 0.3s ease, color 0.3s ease;
  color: #999;
  opacity: 0.6;
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 99;
}

.heart-icon.liked {
  color: #ff4757;
  opacity: 1;
}

.heart-icon.liked svg {
  fill: currentColor;
}

.heart-icon:hover {
  transform: scale(1.1);
  opacity: 1;
}

.tool-header {
  position: relative;
  display: flex;
  align-items: center;
  padding: 10px;
}

.heart-icon {
  position: absolute;
  right: 10px;
  font-size: 12px;
  cursor: pointer;
  transition: transform 0.3s ease, color 0.3s ease;
  color: #999;
  opacity: 0.6;
}

.heart-icon.liked {
  color: #ff4757;
  opacity: 1;
}

.heart-icon:hover {
  transform: scale(1.1);
  opacity: 1;
}

@keyframes jelly {
  0% { transform: scale(1); }
  25% { transform: scale(1.2); }
  50% { transform: scale(0.95); }
  75% { transform: scale(1.1); }
  100% { transform: scale(1); }
}

.jelly {
  animation: jelly 0.6s ease;
}

.like-history-btn {
  margin-left: 10px;
}

.liked-tools-list {
  padding: 10px;
  max-height: none;
  overflow-y: hidden;
}

.liked-tools-list.scrollable {
  max-height: 70vh;
  overflow-y: auto;
  scrollbar-width: thin;
  scrollbar-color: #888 #f1f1f1;
}

.liked-tools-list.scrollable::-webkit-scrollbar {
  width: 6px;
}

.liked-tools-list.scrollable::-webkit-scrollbar-track {
  background: #f1f1f1;
  border-radius: 3px;
}

.liked-tools-list.scrollable::-webkit-scrollbar-thumb {
  background: #888;
  border-radius: 3px;
}

.liked-tools-list.scrollable::-webkit-scrollbar-thumb:hover {
  background: #555;
}

/* 暗色模式下的滚动条样式 */
.dark .liked-tools-list.scrollable {
  scrollbar-color: #666 #2c2c2c;
}

.dark .liked-tools-list.scrollable::-webkit-scrollbar-track {
  background: #2c2c2c;
}

.dark .liked-tools-list.scrollable::-webkit-scrollbar-thumb {
  background: #666;
}

.dark .liked-tools-list.scrollable::-webkit-scrollbar-thumb:hover {
  background: #888;
}

.like-history-dialog {
  max-height: 90vh;
  display: flex;
  flex-direction: column;
}

.no-likes {
  text-align: center;
  padding: 20px;
  color: #666;
}

.liked-tool-item {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 15px;
  border-bottom: 1px solid #eee;
  transition: background-color 0.3s;
  cursor: pointer;
}

.liked-tool-item:hover {
  background-color: #f5f5f5;
}

.dark .liked-tool-item:hover {
  background-color: #2c2c2c;
}

.liked-tool-info {
  display: flex;
  align-items: center;
  flex: 1;
}

.tool-details {
  margin-left: 15px;
}

.tool-details h4 {
  margin: 0 0 5px 0;
}

.tool-details p {
  margin: 0;
  font-size: 0.9em;
  color: #666;
}

.dark .tool-details p {
  color: #999;
}

.liked-tool-actions {
  display: flex;
  gap: 10px;
}

.liked-tool-actions button {
  background: none;
  border: none;
  cursor: pointer;
  padding: 5px;
  border-radius: 4px;
  transition: transform 0.2s;
}

.liked-tool-actions button:hover {
  transform: scale(1.1);
}

.unlike-btn {
  color: #ff4757;
}

.link-btn {
  color: #2196f3;
}

.dark .link-btn {
  color: #64b5f6;
}

.like-history-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0 20px 10px;
  border-bottom: 1px solid #eee;
}

.dark .like-history-header {
  border-bottom-color: #333;
}

.like-history-title {
  font-size: 16px;
  font-weight: 500;
}

.clear-all-btn {
  background: none;
  border: 1px solid #ddd;
  padding: 6px 12px;
  border-radius: 4px;
  cursor: pointer;
  display: flex;
  align-items: center;
  gap: 4px;
  transition: all 0.3s ease;
  color: #666;
}

.clear-all-btn:hover {
  background-color: #f5f5f5;
  border-color: #ccc;
  color: #333;
}

.dark .clear-all-btn {
  border-color: #444;
  color: #999;
}

.dark .clear-all-btn:hover {
  background-color: #2c2c2c;
  border-color: #555;
  color: #fff;
}

/* 自定义消息框样式 */
:deep(.custom-message-box) {
  .el-message-box__message {
    p {
      line-height: 1.8;
      font-size: 14px;
    }
  }
}

.menu-info {
  color: #666;
  font-size: 0.85em;
  margin-right: 8px;
  padding: 2px 6px;
  background-color: #f5f5f5;
  border-radius: 4px;
  display: inline-flex;
  align-items: center;
  gap: 4px;
}
.menu-menuName{
  padding: 3px;
  font-size: 12px;
}
.dark .menu-info {
  color: #999;
  background-color: #2c2c2c;
}

.app-container {
  height: 100vh;
  display: flex;
  flex-direction: column;
}

.app-header {
  padding: 10px 20px;
  background-color: #fff;
  border-bottom: 1px solid #dcdfe6;
  display: flex;
  justify-content: flex-end;
}

.header-icons {
  display: flex;
  align-items: center;
  gap: 15px;
}

.clock-container {
  display: flex;
  align-items: center;
  gap: 5px;
  color: #909399;
}

.time-icon {
  font-size: 16px;
}

.time-text {
  font-size: 14px;
}


.mail-icon {
  font-size: 20px;
  color: #909399;
  cursor: pointer;
}

.clock-component {
  margin-right: 5px;
}

.email-icon {
  color: #909399;
  text-decoration: none;
  display: flex;
  align-items: center;
}

.email-icon .el-icon {
  font-size: 20px;
}

.el-dialog {
  display: flex;
  flex-direction: column;
}

.el-dialog__body {
  flex: 1;
  padding: 0;
  overflow: hidden;
}

/* 游戏对话框特殊样式 */
:deep(.el-dialog.game-dialog) {
  min-height: 600px;
}

:deep(.el-dialog.game-dialog .el-dialog__body) {
  height: calc(100% - 54px);
  padding: 0;
}
</style>
