<script setup lang="ts">
import {ref, onMounted, computed, reactive} from 'vue'
import CalHeatmap from 'cal-heatmap';
import 'cal-heatmap/cal-heatmap.css';
import {useToast} from 'vue-toastification'
import Tooltip from 'cal-heatmap/plugins/Tooltip';
import {useData} from "vitepress";

/**
 * Response Meta
 */
interface rssEntity {
  title: string,
  url: string,
  summary: string,
  timestamp: number,
}

interface hentaiAPI {
  'Resources': rssEntity[],
  'News': rssEntity[],
  'DLsite Game Ranking': rssEntity[],
  'DLsite Voice Ranking': rssEntity[],
  'DLsite Comic Ranking': rssEntity[],
}

// 定义字段映射配置
const FIELD_CONFIG = {
  'Resources': {
    price: 'FREE',
    type: 'tip',
    ranking: false,
    desc: "",
    rss: 'https://raw.githubusercontent.com/bGZo/hentai-daily/refs/heads/vitepress/api/feeds/resources.xml'
  },
  'News': {
    price: 'FREE',
    type: 'tip',
    ranking: false,
    desc: "",
    rss: 'https://raw.githubusercontent.com/bGZo/hentai-daily/refs/heads/vitepress/api/feeds/news.xml'
  },
  'DLsite Game Ranking': {
    price: 'PAID',
    type: 'danger',
    ranking: true,
    desc: "",
    rss: 'https://raw.githubusercontent.com/bGZo/hentai-daily/refs/heads/vitepress/api/feeds/dlsite-game-ranking.xml'
  },
  'DLsite Voice Ranking': {
    price: 'PAID',
    type: 'danger',
    ranking: true,
    desc: "",
    rss: 'https://raw.githubusercontent.com/bGZo/hentai-daily/refs/heads/vitepress/api/feeds/dlsite-voice-ranking.xml'
  },
  'DLsite Comic Ranking': {
    price: 'PAID',
    type: 'danger',
    ranking: true,
    desc: "",
    rss: 'https://raw.githubusercontent.com/bGZo/hentai-daily/refs/heads/vitepress/api/feeds/dlsite-comic-ranking.xml'
  },
} as const;

/**
 * Fields
 */

const data = ref<hentaiAPI>(null)
const loading = ref(false)
const error = ref(null)
// 日期
const currentDate = ref('')
// 目录限制
const tocCountLimit = 5
const showContent = ref(true)
// 目录配置
const isCollapsed = ref(true)
const showAllItems = reactive<Record<string, boolean>>({})
// 消息通知
const toast = useToast()
// 是否是黑暗模式
const { isDark } = useData()
// 构建 API URL - 使用相对路径，会被代理转发
const apiUrl = computed(() => {
  return `/api/archives/${currentDate.value}.json`
})

/**
 * 获取昨日凌晨的时间戳（本地时间）
 * 精确到秒（非毫秒）
 */
const getYesterdayMidnightTimestamp = (): number => {
  const yesterday = new Date(currentDate.value);
  yesterday.setDate(yesterday.getDate() - 1);
  yesterday.setHours(0, 0, 0, 0);
  return yesterday.getTime() / 1000;
};

// 格式化时间戳为可读字符串
const formatTimestamp = (timestamp: number): string => {
  const date = new Date(timestamp);
  return date.toLocaleString(); // 或者使用更具体的格式化方法
};

// 获取当前日期并格式化为 YYYY/MM/DD
// FIXME: 凌晨怎么办？？？
const getCurrentDate = (now: Date) => {
  const year = now.getFullYear()
  const month = String(now.getMonth() + 1).padStart(2, '0')
  const day = String(now.getDate()).padStart(2, '0')
  return `${year}/${month}/${day}`
}

const fetchData = async () => {
  loading.value = true
  error.value = null

  try {
    console.log('请求 URL:', apiUrl.value)
    const response = await fetch(apiUrl.value, {
      method: 'GET',
      headers: {
        'Content-Type': 'application/json',
      }
    })

    if (!response.ok) {
      showContent.value = false
      data.value = {} as hentaiAPI
      // error route
      if (404 == response.status) {
        toast.info("It seems not exist for today. Please check other days")
      } else {
        toast.error(`Except resonse with: ${response.status}, please contact with admin`)
      }
      throw new Error(`HTTP error! status: ${response.status}`)
    } else {
      showContent.value = true
    }

    const result = await response.json()
    // 类型断言和验证
    if (isValidHentaiAPI(result)) {
      data.value = result as hentaiAPI
      // formatResponse(data.value) TODO 过滤
    } else {
      throw new Error('Invalid API response format')
    }

  } catch (err) {
    error.value = err.message
    console.error('API 请求失败:', err)
  } finally {
    loading.value = false
  }
}

const clickCopyLink = (url:string) => {
  navigator.clipboard.writeText(url).then(i => {
        toast.info('Copy link successful.')
  }).catch(e=>console.error(e));
}

const handleSubscribeClick = (index: string) => {
  window.open(FIELD_CONFIG[index].rss, '_blank')
}

const handleCardClick = (url: string) => {
  // NOTE: open in current tab
  // window.location.href = url
  // NOTE: open new tab
  window.open(url, '_blank')
}

const handleCardCss = (entity_index: number, index: string) => {
  if (!FIELD_CONFIG[index].ranking) {
    // no ranking no handle
    return 'card-style-common'
  } else {
    switch (entity_index) {
      case 0:
        return 'card-style-king'
      case 1:
        return 'card-style-silver'
      case 2:
        return 'card-style-bronze'
      case 3:
        return 'card-style-common'
      case 4:
        return 'card-style-common'
    }
  }
}

// 类型验证函数
function isValidHentaiAPI(obj: any): obj is hentaiAPI {
  if (!obj || typeof obj !== 'object') return false
  const requiredKeys: (keyof hentaiAPI)[] = [
    'Resources',
    'News',
    'DLsite Game Ranking',
    'DLsite Voice Ranking',
    'DLsite Comic Ranking'
  ]
  return requiredKeys.every(key => {
    const value = obj[key]
    return Array.isArray(value) && value.every(isValidRssEntity)
  })
}

function isValidRssEntity(obj: any): obj is rssEntity {
  return obj &&
      typeof obj === 'object' &&
      typeof obj.title === 'string' &&
      typeof obj.url === 'string' &&
      typeof obj.summary === 'string' &&
      typeof obj.timestamp === 'number'
}

/**
 * 过滤出戒指昨天的内容
 * @param list
 */
const filterToday = (list: rssEntity[]) => {
  return list.filter(i => i.timestamp > getYesterdayMidnightTimestamp())
}

const toggleItemsVisibility = (index: string) => {
  showAllItems[index] = !showAllItems[index]
}

const refreshToday = (timestamp?: number) => {
  if (timestamp) {
    currentDate.value = getCurrentDate(new Date(timestamp))
  } else {
    currentDate.value = getCurrentDate(new Date())
  }
  fetchData()
}

function watchDarkMode(callback) {
  if (typeof window === 'undefined') return
  const observer = new MutationObserver(() => {
    const isDark = document.documentElement.classList.contains('dark')
    callback(isDark)
  })
  observer.observe(document.documentElement, {
    attributes: true,
    attributeFilter: ['class']
  })
  // 初始触发一次
  // callback(document.documentElement.classList.contains('dark'))
}

function createCalHeatmap(){
  const cal = new CalHeatmap();
  cal.paint({
    itemSelector: '#cal-heatmap',
    domain: {
      type: 'month',
    },
    subDomain: {
      type: 'ghDay'
    },
    date: {
      start: new Date('2025-01-01'),
    },
    data: {
      source: '/api/count.json',
      x: 'date',
      y: 'value',
    },
    scale: {
      color: {
        range: ['#9be9a8', '#40c463', '#30a14e', '#216e39'],
        domain: [0, 30],
      }
    },
    theme: isDark.value ? 'dark': 'light',
  }, [[Tooltip, {
    text: t => `${new Date(t).toLocaleDateString()}`,
  }]]);

  cal.on('click', ((event: any, timestamp: number, value: any) => {
    console.log('click' + new Date(timestamp).toLocaleDateString());
    if (timestamp > new Date().getTime()) {
      toast.info("The future is yours. Check it in few days later.")
    } else {
      refreshToday(timestamp)
    }
  }) as any); // 关键：使用 as any 绕过类型检查
  return cal;
}

// 组件挂载时设置当前日期
onMounted(() => {
  console.log('1绘制为 ', isDark)
  refreshToday()

  var cal = createCalHeatmap()
  // 监听黑暗模式变化，重新渲染图表
  watchDarkMode((isDark) => {
    if (cal) {
      cal.destroy()
    }
    cal = createCalHeatmap()
    console.log('绘制为 ', isDark)
  })
})

</script>

<template>
  <div class="today-title">
    <span class="hero">Hentai Daily</span>
    <span class="date">{{ new Date(currentDate).toLocaleDateString() }}</span>
  </div>
  <!-------------------------HeatMap--------------------------------->
  <div class="heatmap-scroll">
    <div id="cal-heatmap"></div>
  </div>
  <!-------------------------TOC--------------------------------->
  <div v-show="showContent" class="toc-container">
    <div class="toc-header" @click="isCollapsed = !isCollapsed">
      <h2>Table of Contents</h2>
      <span class="collapse-icon">
        {{ isCollapsed ? '▶' : '▼' }}
      </span>
    </div>
    <div class="toc-content"
         :class="{ collapsed: isCollapsed }">
      <ul class="toc-list">
        <li v-for="(today, index) in data"
            :key="index"
            class="toc-section">
          <a :href="`#section-${index}`"
             v-if="filterToday(today).length !== 0"
             class="section-link">
            {{ index }} ({{ filterToday(today).length }})
          </a>
          <ul class="toc-items">
            <li v-for="(entity, entity_index) in today.slice(0, showAllItems[index] ? undefined : tocCountLimit)"
                :key="entity_index"
                class="toc-item">
              <a :href="`#item-${index}-${entity_index}`"
                 v-if="entity.timestamp > getYesterdayMidnightTimestamp()"
                 class="item-link"
                 :title="entity.title">
                {{ entity.title }}
              </a>
            </li>
            <!-- 显示更多按钮 -->
            <li v-if="filterToday(today).length > tocCountLimit" class="show-more">
              <button
                  @click="toggleItemsVisibility(index)"
                  class="show-more-btn">
                {{ showAllItems[index] ? 'Show less' : `Show more(${filterToday(today).length - tocCountLimit})` }}
              </button>
            </li>
          </ul>
        </li>
      </ul>
    </div>
  </div>
  <!--------------------------Content-------------------------------->
  <div v-for="(today, index) in data" :key="`today-${index}-${filterToday(today).length}`">
    <h2 class="content-title" v-if="filterToday(today).length > 0" :id="`section-${index}`">
      {{ index }}  <Badge type="warning" text="subscribe" @click="handleSubscribeClick(index)"/>
    </h2>
    {{ FIELD_CONFIG[index].desc }}
    
    <div v-for="(entity, entity_index) in today" :key="entity_index">
      <div v-if="entity.timestamp > getYesterdayMidnightTimestamp()">
        <div class="card">
          <div :class="`card-content ${handleCardCss(entity_index, index)} card-style`">
            <span class="card-header">
                <h3 :id="`item-${index}-${entity_index}`" class='card-title' @click="clickCopyLink(entity.url)">
                {{ entity.title === '' ? 'Untitled' : entity.title }}
                  <Badge :type="FIELD_CONFIG[index].type" :text="FIELD_CONFIG[index].price"/>
                </h3>
                <span class="card-datetime">{{ formatTimestamp(entity.timestamp * 1000) }}</span>
            </span>
            <div class="message" v-html="entity.summary" @click="handleCardClick(entity.url)"/>
            <div class="message" v-html="entity.translate" @click="handleCardClick(entity.url)"/>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>

.heatmap-scroll {
  overflow-x: auto;
  padding-bottom: 20px; /* 为滚动条留出空间 */
}

#cal-heatmap {
  min-width: 600px; /* 热力图的最小宽度 */
}

.today-title {
  width: 100%;
  margin: 10px 0 10px 0;

  .hero {
    color: var(--vp-home-hero-name-color);
    font-size: 2em;
    font-weight: bold;
  }

  .date {
    color: var(--vp-c-text-1);
    font-size: 1em;
    float: right;
  }
}

#cal-heatmap {
  margin: 10px 0 20px 0;
}

.content-title {
  text-align: center;
}

.card {
  border-radius: 20px;
  overflow: hidden;

  margin: 40px 0 40px 0;
  width: 100%;
  position: relative;
  cursor: pointer;
  transition: all 0.4s cubic-bezier(0.175, 0.885, 0.32, 1.275);
}

.card:hover {
  transform: scale(1.05) rotate(1deg);
  box-shadow: 0 12px 24px -8px gray;
  /**
  var(--vp-c-brand-1)
   */
}

.card-content {
  padding: 2rem 1.5rem;
  position: relative;
  height: fit-content;
}

.card-header {
  width: 100%;
  display: inline-block;
  margin-bottom: 1rem;
  color: var(--vp-c-text-1);
}

.message {
  font-size: 1.0rem;
  font-weight: 700;
  margin-bottom: 0.5rem;
  line-height: 1.4;
  color: var(--vp-c-text-1);
}

.card-title {
  margin: 10px 0 10px 0;
  color: var(--vp-c-text-1);
}

.card-datetime {
  float: right;
}

.particle {
  position: absolute;
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: rgba(255, 255, 255, 0.5);
  animation: float 4s infinite;
}

@keyframes float {
  0%, 100% {
    transform: translateY(0) translateX(0);
    opacity: 0;
  }
  50% {
    transform: translateY(-20px) translateX(10px);
    opacity: 1;
  }
}

@keyframes pop {
  0% {
    transform: scale(0.8);
    opacity: 0;
  }
  50% {
    transform: scale(1.1);
  }
  100% {
    transform: scale(1);
    opacity: 1;
  }
}

.card.animate {
  animation: pop 0.5s cubic-bezier(0.175, 0.885, 0.32, 1.275);
}


.controls button {
  background: var(--vp-c-brand-3);
  color: white;
  border: none;
  padding: 8px 16px;
  border-radius: 4px;
  cursor: pointer;
}

.error {
  background: #ffebee;
  color: #c62828;
  padding: 10px;
  border-radius: 4px;
  margin: 10px 0;
}


/* 目录样式 */
.toc-container {
  border: 1px solid var(--vp-c-bg-soft);
  border-radius: 20px;
  height: 100%;
  background-color: var(--vp-c-bg-soft);
  transition: border-color 0.25s, background-color 0.25s;
}

.toc-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px 16px;
  cursor: pointer;
  user-select: none;
  transition: background-color 0.2s;
}

.toc-header:hover {
  /**
  background: #f3f4f6;
   */
}

.toc-header h2 {
  margin: 0;
  padding: 10px 0 10px 0;
  border-top: 0;
  font-weight: 600;
  color: var(--vp-c-text-1);
}

.collapse-icon {
  font-size: 14px;
  color: #6b7280;
  transition: transform 0.2s;
}

.toc-content {
  max-height: 400px;
  overflow-y: auto;
  transition: all 0.3s ease;
  padding: 16px;
}

.toc-content.collapsed {
  max-height: 0;
  padding: 0 16px;
  overflow: hidden;
}

.toc-list {
  list-style: none;
  padding: 0;
  margin: 0;
}

.toc-section {
  margin-bottom: 16px;
  padding-left: 8px;
  border-left: 3px solid #e5e7eb;
}

.section-link {
  display: block;
  font-weight: 600;
  color: var(--vp-c-brand-1);
  text-decoration: none;
  padding: 4px 0;
  transition: color 0.2s;
}

.section-link:hover {
  color: var(--vp-c-brand-3);
}

.toc-items {
  list-style: none;
  padding: 0;
  margin: 8px 0 0 0;
}

.toc-item {
  margin-bottom: 4px;
}

.item-link {
  display: block;
  color: var(--vp-c-text-2);
  text-decoration: none;
  font-size: 0.9rem;
  padding: 2px 0;
  line-height: 1.4;
  transition: color 0.2s;

  /* 文本截断 */
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.item-link:hover {
  color: #374151;
}

.show-more {
  margin-top: 8px;
}

.show-more-btn {
  background: none;
  border: none;
  color: var(--vp-c-brand-1);
  font-size: 0.8rem;
  cursor: pointer;
  padding: 4px 0;
  text-decoration: underline;
  transition: color 0.2s;
}

.show-more-btn:hover {
  color: var(--vp-c-brand-3);
}

</style>