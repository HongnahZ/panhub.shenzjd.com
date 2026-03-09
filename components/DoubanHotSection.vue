<template>
  <div v-if="!hasAnyData" class="hidden"></div>

  <div v-else class="douban-movie-section">
    <!-- 分类 Tabs -->
    <div class="category-tabs">
      <button
        v-for="cat in availableCategories"
        :key="cat.id"
        :class="['tab-btn', { active: selectedCategoryId === cat.id }]"
        @click="selectCategory(cat.id)"
      >
        {{ cat.label }} {{ cat.type || "榜单" }}
      </button>
    </div>

    <div v-if="loading && items.length === 0" class="loading-state">
      <div class="spinner"></div>
      <span>加载中…</span>
    </div>
    <div v-else>
      <div class="movie-grid">
        <button
          v-for="item in items"
          :key="(item.id ?? 0) + item.title"
          class="movie-card"
          :aria-label="`搜索 ${extractTerm(item.title)}`"
          @click="onItemClick(item.title)"
        >
          <div class="movie-cover">
            <img
              v-if="item.cover && !imgFailed.includes(item.id ?? 0)"
              :src="proxyCover(item.cover)"
              :alt="extractTerm(item.title)"
              loading="lazy"
              referrerpolicy="no-referrer"
              @error="onImgError(item.id ?? 0)"
            />
            <div v-else class="cover-placeholder">🎬</div>
          </div>
          <div class="movie-info">
            <span class="movie-title">{{ extractTerm(item.title) }}</span>
            <span v-if="item.desc" class="movie-desc">{{ item.desc }}</span>
          </div>
        </button>
      </div>

      <!-- 加载更多触发器 -->
      <div
        v-if="hasMore || loadingMore"
        ref="loadTriggerRef"
        class="load-more-trigger"
      >
        <div v-if="loadingMore" class="loading-more">
          <div class="spinner small"></div>
          <span>加载更多…</span>
        </div>
      </div>

      <!-- 没有更多数据提示 -->
      <div v-else-if="items.length > 0" class="no-more">
        没有更多了
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, onBeforeUnmount, nextTick } from "vue";

interface Props {
  onSearch: (term: string) => void;
}

interface DoubanHotItem {
  id?: number;
  title: string;
  url?: string;
  cover?: string;
  desc?: string;
  hot?: number;
}

const props = defineProps<Props>();

const loading = ref(false);
const loadingMore = ref(false);
const items = ref<DoubanHotItem[]>([]);
const hasMore = ref(true);
const imgFailed = ref<number[]>([]);
const selectedCategoryId = ref<string>("douban-movie");
const currentPage = ref(1);
const loadObserver = ref<IntersectionObserver | null>(null);
const loadTriggerRef = ref<HTMLElement | null>(null);

// 所有可用的分类配置
const availableCategories = computed(() => {
  return [
    { id: "douban-top250", label: "电影", type: "Top250" },
    { id: "douban-movie", label: "电影", type: "新片榜" },
    { id: "douban-weekly", label: "电影", type: "口碑榜" },
    { id: "douban-us-box", label: "电影", type: "北美票房" },
  ];
});

// 是否有任何数据
const hasAnyData = computed(() => {
  return items.value.length > 0;
});

function onImgError(id: number) {
  if (!imgFailed.value.includes(id)) {
    imgFailed.value = [...imgFailed.value, id];
  }
}

function extractTerm(title: string): string {
  return title.replace(/^【[\d.]+】/, "").replace(/^#\d+\s*/, "").trim() || title;
}

function proxyCover(url: string): string {
  if (!url) return "";
  return `/api/img?url=${encodeURIComponent(url)}`;
}

async function fetchCategoryData(categoryId: string, page: number, append = false) {
  if (page === 1) {
    loading.value = true;
  } else {
    loadingMore.value = true;
  }

  try {
    const response = await fetch(`/api/douban-hot?category=${categoryId}&page=${page}&limit=25`);
    const data = await response.json();

    if (data.code === 0 && data.data) {
      const newItems = data.data.items || [];
      if (append) {
        items.value = [...items.value, ...newItems];
      } else {
        items.value = newItems;
      }
      hasMore.value = data.data.hasMore !== undefined ? data.data.hasMore : newItems.length >= 25;
      currentPage.value = page;
    } else {
      if (!append) {
        items.value = [];
      }
      hasMore.value = false;
    }
  } catch {
    if (!append) {
      items.value = [];
    }
    hasMore.value = false;
  } finally {
    loading.value = false;
    loadingMore.value = false;
  }
}

async function selectCategory(categoryId: string) {
  if (categoryId === selectedCategoryId.value && items.value.length > 0) return;
  selectedCategoryId.value = categoryId;
  currentPage.value = 1;
  hasMore.value = true;
  await fetchCategoryData(categoryId, 1, false);
  await nextTick();
  setupLoadMoreObserver();
}

async function loadMore() {
  if (loadingMore.value || !hasMore.value) return;
  await fetchCategoryData(selectedCategoryId.value, currentPage.value + 1, true);
}

function setupLoadMoreObserver() {
  // 清理旧的 observer
  if (loadObserver.value) {
    loadObserver.value.disconnect();
  }

  // 等待 DOM 更新
  nextTick(() => {
    const target = loadTriggerRef.value;
    if (!target) return;

    loadObserver.value = new IntersectionObserver(
      (entries) => {
        if (entries[0].isIntersecting && hasMore.value && !loading.value && !loadingMore.value) {
          loadMore();
        }
      },
      {
        root: null,
        rootMargin: "100px",
        threshold: 0.1,
      }
    );

    loadObserver.value.observe(target);
  });
}

function onItemClick(title: string) {
  const term = extractTerm(title);
  if (term) props.onSearch(term);
}

async function init() {
  await fetchCategoryData(selectedCategoryId.value, 1, false);
  await nextTick();
  setupLoadMoreObserver();
}

async function refresh() {
  currentPage.value = 1;
  hasMore.value = true;
  await fetchCategoryData(selectedCategoryId.value, 1, false);
}

onBeforeUnmount(() => {
  if (loadObserver.value) {
    loadObserver.value.disconnect();
  }
});

defineExpose({ init, refresh });
</script>

<style scoped>
.douban-movie-section {
  width: 100%;
}

.section-head {
  display: flex;
  align-items: baseline;
  justify-content: space-between;
  gap: 10px;
  margin-bottom: 12px;
}

.section-title {
  font-size: 16px;
  font-weight: 800;
  color: var(--text-primary);
  margin: 0;
}

.section-subtitle {
  margin: 0;
  font-size: 12px;
  color: var(--text-tertiary);
}

.category-tabs {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  margin-bottom: 16px;
  padding-bottom: 12px;
  border-bottom: 1px solid var(--border-light);
}

.tab-btn {
  padding: 6px 12px;
  font-size: 13px;
  font-weight: 500;
  color: var(--text-secondary);
  background: rgba(255, 255, 255, 0.6);
  border: 1px solid var(--border-light);
  border-radius: 8px;
  cursor: pointer;
  transition: all 180ms ease;
  white-space: nowrap;
}

.tab-btn:hover {
  background: rgba(255, 255, 255, 0.85);
  color: var(--text-primary);
  border-color: var(--primary);
}

.tab-btn.active {
  color: var(--primary);
  background: rgba(15, 118, 110, 0.08);
  border-color: var(--primary);
  font-weight: 600;
}

.loading-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 12px;
  padding: 48px 20px;
  color: var(--text-secondary);
  background: rgba(255, 255, 255, 0.55);
  backdrop-filter: blur(8px);
  border: 1px solid var(--border-light);
  border-radius: 14px;
}

.spinner {
  width: 28px;
  height: 28px;
  border: 3px solid rgba(15, 118, 110, 0.2);
  border-top-color: var(--primary);
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

.movie-grid {
  display: grid;
  grid-template-columns: repeat(5, 1fr);
  gap: 12px;
}

.movie-card {
  display: flex;
  flex-direction: column;
  align-items: stretch;
  background: rgba(255, 255, 255, 0.7);
  backdrop-filter: blur(8px);
  border: 1px solid var(--border-light);
  border-radius: 12px;
  overflow: hidden;
  cursor: pointer;
  transition: transform 180ms ease, box-shadow 180ms ease;
  text-align: left;
  padding: 0;
}

.movie-card:hover {
  transform: translateY(-4px);
  box-shadow: 0 12px 24px rgba(15, 118, 110, 0.15);
}

.movie-cover {
  aspect-ratio: 2 / 3;
  background: linear-gradient(135deg, #e5e7eb 0%, #d1d5db 100%);
  overflow: hidden;
}

.movie-cover img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  display: block;
}

.cover-placeholder {
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 28px;
}

.movie-info {
  padding: 8px 10px;
  display: flex;
  flex-direction: column;
  gap: 2px;
  min-height: 0;
}

.movie-title {
  font-size: 12px;
  font-weight: 600;
  color: var(--text-primary);
  line-height: 1.3;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

.movie-desc {
  font-size: 10px;
  color: var(--text-tertiary);
  line-height: 1.2;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.hidden {
  display: none;
}

@media (max-width: 500px) {
  .movie-grid {
    grid-template-columns: repeat(4, 1fr);
    gap: 10px;
  }
}

@media (min-width: 640px) {
  .movie-grid {
    gap: 14px;
  }

  .movie-title {
    font-size: 13px;
  }
}

@media (prefers-color-scheme: dark) {
  .loading-state,
  .movie-card {
    background: rgba(17, 24, 39, 0.6);
    border-color: rgba(75, 85, 99, 0.4);
  }

  .movie-cover {
    background: linear-gradient(135deg, #374151 0%, #4b5563 100%);
  }

  .movie-card:hover {
    box-shadow: 0 12px 24px rgba(15, 118, 110, 0.25);
  }
}

@media (prefers-reduced-motion: reduce) {
  .spinner,
  .movie-card {
    animation: none;
    transition: none;
  }

  .movie-card:hover {
    transform: none;
  }
}

.load-more-trigger {
  padding: 20px 0;
  min-height: 60px;
  display: flex;
  justify-content: center;
  align-items: center;
}

.loading-more {
  display: flex;
  align-items: center;
  gap: 8px;
  color: var(--text-secondary);
  font-size: 14px;
}

.spinner.small {
  width: 16px;
  height: 16px;
  border-width: 2px;
}

.no-more {
  padding: 20px 0;
  text-align: center;
  color: var(--text-tertiary);
  font-size: 13px;
}
</style>
