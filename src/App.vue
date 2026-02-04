<script setup>
import { ref, computed, onMounted } from 'vue'

// 検索サイト定義（フィルタ付き）
const SEARCH_SITES = [
  { id: 'google', name: 'Google', icon: '🔍', color: '#8b8fc7',
    baseUrl: (q, trusted) => trusted
      ? `https://www.google.com/search?q=${q}+site:ac.jp+OR+site:ed.jp+OR+site:go.jp+OR+site:lg.jp`
      : `https://www.google.com/search?q=${q}`,
    filters: [
      { id: 'default', label: '通常', suffix: '' },
      { id: '1y', label: '1年以内', suffix: '&tbs=qdr:y' },
      { id: '1m', label: '1ヶ月以内', suffix: '&tbs=qdr:m' },
      { id: '1w', label: '1週間以内', suffix: '&tbs=qdr:w' },
    ],
  },
  { id: 'youtube', name: 'YouTube', icon: '▶️', color: '#b07070',
    baseUrl: (q) => `https://www.youtube.com/results?search_query=${q}`,
    filters: [
      { id: 'default', label: '関連度順', suffix: '' },
      { id: 'date', label: '新しい順', suffix: '&sp=CAI%3D' },
      { id: 'long', label: '長い動画', suffix: '&sp=EgIYAQ%3D%3D' },
      { id: 'short', label: '短い動画', suffix: '&sp=EgIYAg%3D%3D' },
    ],
  },
  { id: 'amazon', name: 'Amazon (Kindle)', icon: '🛒', color: '#b8a06a',
    baseUrl: (q) => `https://www.amazon.co.jp/s?k=${q}&i=digital-text`,
    filters: [
      { id: 'review', label: '高評価順', suffix: '&s=review-rank' },
      { id: 'date', label: '新着順', suffix: '&s=date-desc-rank' },
    ],
  },
  { id: 'dmm', name: 'DMM', icon: '📚', color: '#7aab7a',
    baseUrl: (q) => `https://book.dmm.com/search/?searchstr=${q}`,
    filters: [
      { id: 'default', label: '通常', suffix: '' },
      { id: 'review', label: 'レビュー順', suffix: '&sort=review' },
      { id: 'popular', label: '人気順', suffix: '&sort=popular' },
      { id: 'date', label: '新着順', suffix: '&sort=date' },
    ],
  },
  { id: 'mercari', name: 'メルカリ', icon: '🔄', color: '#b07080',
    baseUrl: (q) => `https://jp.mercari.com/search?keyword=${q}`,
    filters: [
      { id: 'default', label: '通常', suffix: '' },
      { id: 'onsale', label: '販売中のみ', suffix: '&status=on_sale' },
      { id: 'new', label: '新着順', suffix: '&sort=created_time&order=desc' },
    ],
  },
  { id: 'wikipedia', name: 'Wikipedia', icon: '📖', color: '#8a95a5',
    baseUrl: (q) => `https://ja.wikipedia.org/w/index.php?search=${q}`,
  },
  { id: 'scholar', name: 'Google Scholar', icon: '🎓', color: '#7a9bb5',
    baseUrl: (q) => `https://scholar.google.co.jp/scholar?q=${q}`,
  },
  { id: 'x', name: 'X', icon: '𝕏', color: '#9a9ab0',
    baseUrl: (q) => `https://x.com/search?q=${q}&src=typed_query`,
    filters: [
      { id: 'default', label: '話題', suffix: '' },
      { id: 'live', label: '最新', suffix: '&f=live' },
      { id: 'image', label: '画像付き', suffix: '&f=image' },
    ],
  },
  { id: 'bluesky', name: 'Bluesky', icon: '🦋', color: '#6a9ec7',
    baseUrl: (q) => `https://bsky.app/search?q=${q}`,
  },
  { id: 'niconico', name: 'ニコニコ動画', icon: '📺', color: '#9a9a7a',
    baseUrl: (q) => `https://www.nicovideo.jp/search/${q}`,
  },
  { id: 'aozora', name: '青空文庫', icon: '📜', color: '#8aaa8a',
    baseUrl: (q) => `https://www.google.com/search?q=${q}+site:aozora.gr.jp`,
  },
  { id: 'ndl', name: '国会図書館DC', icon: '🏛️', color: '#a08a7a',
    baseUrl: (q) => `https://dl.ndl.go.jp/search?searchWord=${q}`,
  },
  { id: 'cinii', name: 'CiNii', icon: '📑', color: '#7a8ab0',
    baseUrl: (q) => `https://cir.nii.ac.jp/all?q=${q}`,
  },
]

const HISTORY_KEY = 'slg-search-history'
const THEME_KEY = 'slg-theme'
const FILTERS_KEY = 'slg-site-filters'
const MAX_HISTORY = 20

const DEFAULT_TERMS = '坂本龍馬, 高杉晋作, 桂小五郎, 久坂玄瑞, 中岡慎太郎, 山縣有朋, 伊藤博文, 杉文, 西郷隆盛, 大久保利通, 篤姫, 黒田清隆, 西郷ツン, 吉田松陰, 楠本イネ, 福沢諭吉, 中沢琴, 榎本武揚, 嘉納治五郎, 島田虎之助, 武田物外, 男谷信友, 須藤由蔵, 千葉周作, 楢崎将作, 楢崎龍, 千葉佐那, 大石種次, 岡田以蔵, 中村半次郎, 小笠原清務, 松平容保, 飯塚伊賀七, 宮部鼎蔵, 伊東甲子太郎, 河上彦斎, 伊庭八郎, 田中新兵衛, 相楽総三, 山田浅右衛門, タウンゼント・ハリス, こんぴら狗, 村山たか, 井伊直弼, 薄雲大夫, 勝海舟, 徳川慶喜, 渋沢栄一, 間部詮勝, 高橋泥舟, 清河八郎, マシュー・ペリー, ラザフォード・オールコック, ジュール・ブリュネ, アーネスト・サトウ, フェリーチェ・ベアト, ロバート・フォーチュン, マーカス・サミュエル, 近藤勇, 土方歳三, 斎藤一, 沖田総司, 永倉新八, 藤堂平助, 山岡鉄舟, 芹沢鴨, 鈴木三樹三郎, 伊牟田尚平, 井上林太郎, 工藤恒郷, 松田重助, 三遊亭圓朝'

// リアクティブ状態
const searchInput = ref(DEFAULT_TERMS)
const shuffleEnabled = ref(false)
const trustedSitesOnly = ref(false)
const searchHistory = ref([])
const showHistory = ref(false)
const DEFAULT_ENABLED = ['google', 'youtube', 'amazon', 'dmm', 'wikipedia', 'x', 'bluesky', 'niconico']
const enabledSites = ref([...DEFAULT_ENABLED])

// サイト別フィルタ状態
function buildDefaultFilters() {
  const obj = {}
  SEARCH_SITES.forEach(site => {
    if (site.filters && site.filters.length > 0) {
      obj[site.id] = site.filters[0].id
    }
  })
  return obj
}
const siteFilters = ref(buildDefaultFilters())

function loadFilters() {
  try {
    const data = localStorage.getItem(FILTERS_KEY)
    if (data) {
      const saved = JSON.parse(data)
      Object.assign(siteFilters.value, saved)
    }
  } catch { /* デフォルトのまま */ }
}

function saveFilters() {
  localStorage.setItem(FILTERS_KEY, JSON.stringify(siteFilters.value))
}

function setFilter(siteId, filterId) {
  siteFilters.value[siteId] = filterId
  saveFilters()
}

// ONかつフィルタが2つ以上あるサイト
const sitesWithActiveFilters = computed(() => {
  return SEARCH_SITES.filter(s =>
    enabledSites.value.includes(s.id) && s.filters && s.filters.length > 1
  )
})

// テーマ
const isDark = ref(true)

function initTheme() {
  const saved = localStorage.getItem(THEME_KEY)
  if (saved) {
    isDark.value = saved === 'dark'
  } else {
    isDark.value = window.matchMedia('(prefers-color-scheme: dark)').matches
  }
  applyTheme()
}

function applyTheme() {
  document.documentElement.setAttribute('data-theme', isDark.value ? 'dark' : 'light')
}

function toggleTheme() {
  isDark.value = !isDark.value
  localStorage.setItem(THEME_KEY, isDark.value ? 'dark' : 'light')
  applyTheme()
}

// 検索ワードパース
const searchTerms = computed(() => {
  return searchInput.value
    .split(',')
    .map(t => t.trim())
    .filter(t => t.length > 0)
})

// URL生成
function buildUrl(site, encodedQuery) {
  const base = site.baseUrl(encodedQuery, trustedSitesOnly.value)
  if (!site.filters) return base
  const filterId = siteFilters.value[site.id]
  const filter = site.filters.find(f => f.id === filterId)
  return filter ? base + filter.suffix : base
}

const generatedLinks = computed(() => {
  return searchTerms.value.map(term => {
    const encoded = encodeURIComponent(term)
    const sites = SEARCH_SITES
      .filter(s => enabledSites.value.includes(s.id))
      .map(site => ({
        ...site,
        href: buildUrl(site, encoded),
      }))
    return { term, sites }
  })
})

// シャッフル（配列をランダムに並び替え）
const shuffledIndices = ref([])

function reshuffle() {
  const arr = Array.from({ length: generatedLinks.value.length }, (_, i) => i)
  for (let i = arr.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [arr[i], arr[j]] = [arr[j], arr[i]]
  }
  shuffledIndices.value = arr
}

const displayedLinks = computed(() => {
  if (!shuffleEnabled.value) return generatedLinks.value
  return shuffledIndices.value.map(i => generatedLinks.value[i]).filter(Boolean)
})

function toggleShuffle() {
  shuffleEnabled.value = !shuffleEnabled.value
  if (shuffleEnabled.value) reshuffle()
}

// 履歴
function loadHistory() {
  try {
    const data = localStorage.getItem(HISTORY_KEY)
    searchHistory.value = data ? JSON.parse(data) : []
  } catch (error) {
    console.warn('履歴の読み込みに失敗:', error)
    searchHistory.value = []
  }
}

function saveToHistory() {
  const input = searchInput.value.trim()
  if (!input) return
  const filtered = searchHistory.value.filter(h => h !== input)
  filtered.unshift(input)
  searchHistory.value = filtered.slice(0, MAX_HISTORY)
  localStorage.setItem(HISTORY_KEY, JSON.stringify(searchHistory.value))
}

function loadFromHistory(entry) {
  searchInput.value = entry
  showHistory.value = false
}

function removeFromHistory(index) {
  searchHistory.value.splice(index, 1)
  localStorage.setItem(HISTORY_KEY, JSON.stringify(searchHistory.value))
}

function clearHistory() {
  searchHistory.value = []
  localStorage.removeItem(HISTORY_KEY)
}

function handleGenerate() {
  saveToHistory()
  if (shuffleEnabled.value) reshuffle()
}

function isLastEnabled(siteId) {
  return enabledSites.value.length === 1 && enabledSites.value.includes(siteId)
}

function toggleSite(siteId) {
  const idx = enabledSites.value.indexOf(siteId)
  if (idx >= 0) {
    if (enabledSites.value.length > 1) {
      enabledSites.value.splice(idx, 1)
    }
  } else {
    enabledSites.value.push(siteId)
  }
}

function siteChipStyle(site) {
  if (!enabledSites.value.includes(site.id)) return {}
  const opacity = isDark.value ? '25' : '18'
  return { borderColor: site.color, color: site.color, backgroundColor: site.color + opacity }
}

function siteLinkBg(site) {
  const opacity = isDark.value ? '20' : '15'
  return { '--site-color': site.color, backgroundColor: site.color + opacity }
}

function getActiveFilterLabel(siteId) {
  const site = SEARCH_SITES.find(s => s.id === siteId)
  if (!site || !site.filters) return ''
  const filterId = siteFilters.value[siteId]
  const filter = site.filters.find(f => f.id === filterId)
  if (!filter || filter.id === 'default') return ''
  return filter.label
}

onMounted(() => {
  initTheme()
  loadHistory()
  loadFilters()
})
</script>

<template>
  <div class="app-container">
    <section class="control-panel glass" aria-labelledby="panel-heading">
      <div class="panel-header">
        <span id="panel-heading" class="panel-title">🔗 Search Link Generator</span>
        <button class="theme-toggle" @click="toggleTheme"
          :aria-label="isDark ? 'ライトモードに切替' : 'ダークモードに切替'">
          {{ isDark ? '☀️' : '🌙' }}
        </button>
      </div>

      <div class="input-group">
        <label for="search-input" class="input-label">キーワード（カンマ区切り） ※初期値は「Rise of the Ronin」の登場キャラ</label>
        <textarea id="search-input" v-model="searchInput" rows="3"
          placeholder="坂本龍馬, 高杉晋作, 桂小五郎 ..." class="search-textarea"
          @keydown.enter.ctrl="handleGenerate" />
      </div>

      <div class="actions">
        <button class="btn btn-primary" @click="handleGenerate">
          <span aria-hidden="true">🔍</span> リンク生成
        </button>
        <button class="btn btn-ghost" :class="{ active: shuffleEnabled }"
          @click="toggleShuffle" :aria-pressed="shuffleEnabled">
          <span aria-hidden="true">🔀</span> シャッフル
        </button>
        <button class="btn btn-ghost" :class="{ active: showHistory }"
          @click="showHistory = !showHistory" :aria-expanded="showHistory">
          <span aria-hidden="true">📋</span> 履歴
        </button>
      </div>

      <div class="options">
        <label class="toggle-label">
          <input type="checkbox" v-model="trustedSitesOnly" class="toggle-input" />
          <span class="toggle-switch"></span>
          <span class="toggle-text">Google検索を信頼性高に</span>
          <span class="toggle-detail">ac.jp / ed.jp / go.jp / lg.jp のみに絞り込み（大学・教育・政府・自治体）</span>
        </label>
      </div>

      <div class="site-toggles" role="group" aria-label="検索サイトの選択">
        <button v-for="site in SEARCH_SITES" :key="site.id"
          class="site-chip"
          :class="{ active: enabledSites.includes(site.id), 'last-one': isLastEnabled(site.id) }"
          :style="siteChipStyle(site)"
          :disabled="isLastEnabled(site.id)"
          :aria-pressed="enabledSites.includes(site.id)"
          @click="toggleSite(site.id)">
          <span aria-hidden="true">{{ site.icon }}</span> {{ site.name }}
          <span v-if="enabledSites.includes(site.id)" class="chip-check" aria-hidden="true">✓</span>
        </button>
      </div>

      <!-- サイト別フィルタ -->
      <div v-if="sitesWithActiveFilters.length > 0" class="filter-section">
        <div class="filter-section-label">絞り込みオプション</div>
        <div v-for="site in sitesWithActiveFilters" :key="'f-' + site.id" class="filter-row">
          <span class="filter-site-label" :style="{ color: site.color }">
            {{ site.icon }} {{ site.name }}
          </span>
          <div class="filter-chips">
            <button v-for="filter in site.filters" :key="filter.id"
              class="filter-chip"
              :class="{ active: siteFilters[site.id] === filter.id }"
              :style="siteFilters[site.id] === filter.id
                ? { borderColor: site.color, color: site.color, backgroundColor: site.color + (isDark ? '20' : '15') }
                : {}"
              @click="setFilter(site.id, filter.id)">
              {{ filter.label }}
            </button>
          </div>
        </div>
      </div>
    </section>

    <!-- 履歴 -->
    <Transition name="slide">
      <section v-if="showHistory" class="history-panel glass" aria-label="検索履歴">
        <div class="history-header">
          <h3>🕐 検索履歴 <span class="history-count">（最大{{ MAX_HISTORY }}件）</span></h3>
          <button v-if="searchHistory.length" class="btn btn-danger-sm" @click="clearHistory">すべて削除</button>
        </div>
        <div v-if="searchHistory.length === 0" class="history-empty">履歴はありません</div>
        <div v-else class="history-list">
          <div v-for="(entry, idx) in searchHistory" :key="idx" class="history-item">
            <button class="history-text" @click="loadFromHistory(entry)">
              {{ entry.length > 80 ? entry.slice(0, 80) + '...' : entry }}
            </button>
            <button class="history-remove" @click="removeFromHistory(idx)" aria-label="この履歴を削除">✕</button>
          </div>
        </div>
      </section>
    </Transition>

    <div class="result-count" v-if="displayedLinks.length > 0" aria-live="polite">
      <span class="count-badge">{{ displayedLinks.length }}</span> 件
    </div>

    <!-- カード -->
    <section class="card-grid" aria-label="検索リンク一覧">
      <TransitionGroup name="card">
        <div v-for="link in displayedLinks" :key="link.term" class="search-card glass">
          <h3 class="card-title">{{ link.term }}</h3>
          <div class="card-links">
            <a v-for="site in link.sites" :key="site.id"
              :href="site.href" target="_blank" rel="noopener noreferrer"
              class="site-link" :style="siteLinkBg(site)">
              <span class="site-icon" aria-hidden="true">{{ site.icon }}</span>
              <span class="site-name">{{ site.name }}</span>
              <span v-if="getActiveFilterLabel(site.id)" class="filter-badge" :style="{ color: site.color }">
                {{ getActiveFilterLabel(site.id) }}
              </span>
              <span class="external-icon" aria-hidden="true">↗</span>
            </a>
          </div>
        </div>
      </TransitionGroup>
    </section>

    <footer class="footer">
      <p>iPhoneでアプリが開いてしまう場合 →
        <a href="https://www.google.com/search?q=iPhone+ブラウザで開きたい+アプリに飛ばないようにする" target="_blank" rel="noopener noreferrer">対処法を検索</a>
      </p>
      <p><a href="https://mayholiholi.github.io/" target="_blank" rel="noopener noreferrer">← 他のツール</a></p>
    </footer>
  </div>
</template>

<style scoped>
.app-container { display: flex; flex-direction: column; gap: 1rem; }

.panel-header { margin-bottom: 0.1rem; display: flex; justify-content: space-between; align-items: center; }
.panel-title { font-size: 0.7rem; font-weight: 600; color: var(--text-muted); letter-spacing: 0.03em; }

.theme-toggle {
  background: var(--btn-ghost-bg); border: 1px solid var(--glass-border);
  border-radius: 8px; padding: 0.35rem 0.5rem; font-size: 0.9rem;
  cursor: pointer; transition: all 0.2s; line-height: 1;
}
.theme-toggle:active { background: var(--btn-ghost-active-bg); }

.control-panel { padding: 1rem; display: flex; flex-direction: column; gap: 0.8rem; }
.input-label { font-size: 0.65rem; color: var(--text-muted); margin-bottom: 0.2rem; display: block; }

.search-textarea {
  width: 100%; background: var(--input-bg); border: 1px solid var(--glass-border);
  border-radius: 8px; padding: 0.6rem 0.8rem; color: var(--text-primary);
  font-family: inherit; font-size: 0.82rem; resize: vertical;
  transition: border-color 0.2s, background 0.3s;
}
.search-textarea:focus { outline: none; border-color: var(--accent-purple); box-shadow: 0 0 0 2px rgba(123, 114, 176, 0.12); }
.search-textarea::placeholder { color: var(--text-muted); }

.actions { display: flex; gap: 0.4rem; flex-wrap: wrap; }

.btn {
  display: inline-flex; align-items: center; gap: 0.3rem;
  padding: 0.6rem 0.9rem; border-radius: 8px; font-size: 0.78rem;
  font-weight: 600; font-family: inherit; cursor: pointer;
  transition: all 0.2s; border: 1px solid transparent;
  flex: 1; justify-content: center; min-height: 40px;
}
.btn-primary { background: rgba(123, 114, 176, 0.2); color: var(--accent-purple); border: 1px solid rgba(123, 114, 176, 0.25); }
.btn-primary:active { background: rgba(123, 114, 176, 0.3); }
.btn-ghost { background: var(--btn-ghost-bg); color: var(--text-secondary); border: 1px solid var(--glass-border); }
.btn-ghost:active { background: var(--btn-ghost-active-bg); }
.btn-ghost.active { background: rgba(123, 114, 176, 0.12); color: var(--accent-purple); border-color: rgba(123, 114, 176, 0.2); }

.btn-danger-sm {
  background: rgba(180, 90, 90, 0.1); color: #b07070; border: 1px solid rgba(180, 90, 90, 0.15);
  padding: 0.3rem 0.8rem; font-size: 0.7rem; border-radius: 6px;
  cursor: pointer; font-family: inherit; transition: all 0.2s;
}
.btn-danger-sm:active { background: rgba(180, 90, 90, 0.2); }

.options { display: flex; flex-wrap: wrap; gap: 0.6rem; }
.toggle-label { display: flex; align-items: center; gap: 0.5rem; cursor: pointer; user-select: none; flex-wrap: wrap; }
.toggle-input { display: none; }

.toggle-switch {
  width: 36px; height: 20px; background: var(--toggle-bg); border-radius: 10px;
  position: relative; transition: background 0.2s; flex-shrink: 0;
}
.toggle-switch::after {
  content: ''; position: absolute; top: 3px; left: 3px; width: 14px; height: 14px;
  background: var(--toggle-knob); border-radius: 50%; transition: transform 0.2s;
}
.toggle-input:checked + .toggle-switch { background: var(--accent-purple); }
.toggle-input:checked + .toggle-switch::after { transform: translateX(16px); background: #fff; }
.toggle-text { font-size: 0.78rem; color: var(--text-secondary); }
.toggle-detail { font-size: 0.65rem; color: var(--text-muted); width: 100%; margin-top: -0.2rem; padding-left: calc(36px + 0.5rem); }

/* サイトチップ */
.site-toggles { display: flex; flex-wrap: wrap; gap: 0.35rem; }
.site-chip {
  padding: 0.45rem 0.7rem; border-radius: 16px; font-size: 0.7rem; font-family: inherit;
  cursor: pointer; transition: all 0.15s; background: var(--chip-bg);
  border: 1px solid var(--glass-border); color: var(--text-muted);
  min-height: 32px; display: inline-flex; align-items: center; gap: 0.2rem;
}
.site-chip.active { background: var(--chip-active-bg); font-weight: 600; }
.site-chip:active { transform: scale(0.96); }
.site-chip.last-one { opacity: 0.5; cursor: not-allowed; }
.chip-check { font-size: 0.6rem; margin-left: 0.1rem; opacity: 0.7; }

/* フィルタセクション */
.filter-section {
  border-top: 1px solid var(--glass-border); padding-top: 0.6rem;
  display: flex; flex-direction: column; gap: 0.5rem;
}
.filter-section-label { font-size: 0.6rem; color: var(--text-muted); font-weight: 600; letter-spacing: 0.03em; }
.filter-row { display: flex; align-items: flex-start; gap: 0.4rem; flex-wrap: wrap; }
.filter-site-label { font-size: 0.65rem; font-weight: 600; min-width: 5rem; padding-top: 0.2rem; flex-shrink: 0; }
.filter-chips { display: flex; flex-wrap: wrap; gap: 0.25rem; flex: 1; }
.filter-chip {
  padding: 0.25rem 0.55rem; border-radius: 12px; font-size: 0.62rem;
  font-family: inherit; cursor: pointer; transition: all 0.15s;
  background: var(--chip-bg); border: 1px solid var(--glass-border);
  color: var(--text-muted); white-space: nowrap;
}
.filter-chip.active { font-weight: 600; }
.filter-chip:active { transform: scale(0.95); }

/* カード内フィルタバッジ */
.filter-badge { font-size: 0.55rem; font-weight: 600; opacity: 0.8; margin-left: auto; flex-shrink: 0; }

/* 履歴 */
.history-panel { padding: 0.8rem 1rem; }
.history-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 0.6rem; }
.history-header h3 { font-size: 0.85rem; font-weight: 600; }
.history-count { font-size: 0.65rem; font-weight: 400; color: var(--text-muted); }
.history-empty { color: var(--text-muted); font-size: 0.78rem; text-align: center; padding: 0.8rem; }
.history-list { display: flex; flex-direction: column; gap: 0.2rem; max-height: 240px; overflow-y: auto; padding-right: 0.3rem; }
.history-item { display: flex; align-items: center; gap: 0.4rem; padding: 0.4rem 0.5rem; border-radius: 6px; transition: background 0.15s; }
.history-item:active { background: var(--history-hover-bg); }
.history-text { flex: 1; text-align: left; background: none; border: none; color: var(--text-secondary); font-size: 0.75rem; font-family: inherit; cursor: pointer; padding: 0; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.history-remove { background: none; border: none; color: var(--text-muted); cursor: pointer; font-size: 0.75rem; padding: 6px 10px; border-radius: 4px; transition: all 0.15s; flex-shrink: 0; }

/* 結果 */
.result-count { font-size: 0.75rem; color: var(--text-muted); display: flex; align-items: center; gap: 0.3rem; }
.count-badge { background: rgba(123, 114, 176, 0.2); color: var(--accent-purple); font-weight: 700; font-size: 0.72rem; padding: 0.1rem 0.45rem; border-radius: 8px; }

/* カード */
.card-grid { display: grid; grid-template-columns: repeat(2, 1fr); gap: 0.5rem; }
.search-card { padding: 0.7rem; transition: transform 0.15s, box-shadow 0.15s; }
.search-card:active { transform: scale(0.98); }
.card-title { font-size: 0.82rem; font-weight: 600; margin-bottom: 0.5rem; text-align: center; color: var(--text-primary); line-height: 1.3; min-height: 2.2em; display: flex; align-items: center; justify-content: center; }
.card-links { display: flex; flex-direction: column; gap: 0.25rem; }
.site-link { display: flex; align-items: center; gap: 0.4rem; padding: 0.45rem 0.5rem; border-radius: 6px; text-decoration: none; color: var(--text-secondary); font-size: 0.72rem; font-weight: 500; border: 1px solid transparent; transition: all 0.15s; min-height: 34px; }
.site-link:active { border-color: var(--site-color); }
.site-icon { font-size: 0.85rem; flex-shrink: 0; }
.site-name { flex: 1; }
.external-icon { font-size: 0.6rem; opacity: 0.4; flex-shrink: 0; }

/* フッター */
.footer { text-align: center; padding: 1.5rem 0; font-size: 0.75rem; color: var(--text-muted); display: flex; flex-direction: column; gap: 0.4rem; }
.footer a { color: var(--accent-purple); text-decoration: underline; text-decoration-color: rgba(123, 114, 176, 0.3); text-underline-offset: 2px; }

/* アニメーション */
.slide-enter-active, .slide-leave-active { transition: all 0.25s ease; }
.slide-enter-from, .slide-leave-to { opacity: 0; transform: translateY(-8px); }
.card-enter-active { transition: all 0.2s ease; }
.card-enter-from { opacity: 0; transform: scale(0.95); }
.card-move { transition: transform 0.2s ease; }

/* PC */
@media (min-width: 768px) {
  .app-container { gap: 1.5rem; }
  .control-panel { padding: 1.5rem; gap: 1rem; }
  .search-textarea { font-size: 0.9rem; padding: 0.8rem 1rem; }
  .btn { flex: none; padding: 0.6rem 1.2rem; font-size: 0.85rem; }
  .card-grid { grid-template-columns: repeat(auto-fill, minmax(220px, 1fr)); gap: 1rem; }
  .search-card { padding: 1.2rem; }
  .card-title { font-size: 1rem; }
  .site-link { font-size: 0.8rem; padding: 0.5rem 0.7rem; }
  .filter-site-label { min-width: 7rem; }
}

/* hover */
@media (hover: hover) {
  .theme-toggle:hover { background: var(--btn-ghost-active-bg); }
  .search-card:hover { transform: translateY(-3px); box-shadow: var(--card-hover-shadow); }
  .site-link:hover { background: var(--link-hover-bg); border-color: var(--site-color); color: var(--text-primary); transform: translateX(3px); }
  .history-item:hover { background: var(--history-hover-bg); }
  .site-chip:hover:not(.last-one) { transform: scale(1.03); }
  .filter-chip:hover { transform: scale(1.05); }
  .btn:hover { filter: brightness(1.1); }
}

@media (prefers-reduced-motion: reduce) {
  .glass { transition: none; }
  .slide-enter-active, .slide-leave-active, .card-enter-active, .card-move { transition: none; }
}
</style>
