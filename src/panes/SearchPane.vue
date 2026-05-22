<script setup lang="ts">
import { computed, ref } from 'vue'
import { commands, SearchSort } from '../bindings.ts'
import { useMessage } from 'naive-ui'
import ComicCard from '../components/ComicCard.vue'
import FloatLabelInput from '../components/FloatLabelInput.vue'
import { PhMagnifyingGlass } from '@phosphor-icons/vue'
import { SelectProps } from 'naive-ui'
import { useStore } from '../store.ts'

const store = useStore()

const message = useMessage()

const PAGE_SIZE = 80

const sortOptions: SelectProps['options'] = [
  { label: '最新', value: 'Latest' },
  { label: '最多点击', value: 'View' },
  { label: '最多图片', value: 'Picture' },
  { label: '最多爱心', value: 'Like' },
]

const searchInput = ref<string>('')
const searching = ref<boolean>(false)
const downloadingAllSearchResults = ref<boolean>(false)
const sortSelected = ref<SearchSort>('Latest')
const searchPage = ref<number>(1)

const searchPageCount = computed(() => {
  if (store.searchResult === undefined) {
    return 0
  }
  const total = store.searchResult.total
  return Math.ceil(total / PAGE_SIZE)
})

async function search(keyword: string, page: number, sort: SearchSort) {
  if (searching.value) {
    message.warning('有搜索正在进行，请稍后再试')
    return
  }

  searching.value = true
  console.log(keyword, page, sort)
  searchPage.value = page

  const result = await commands.search(keyword, page, sort)
  if (result.status === 'error') {
    console.error(result.error)
    searching.value = false
    return
  }
  const searchResultVariant = result.data
  if ('SearchResult' in searchResultVariant) {
    const respData = searchResultVariant.SearchResult
    if (respData.content.length === 0) {
      message.warning('什么都没有搜到，请尝试其他关键词')
      searching.value = false
      return
    }
    store.searchResult = respData
    console.log(respData)
  } else if ('Comic' in searchResultVariant) {
    const comic = searchResultVariant.Comic
    store.pickedComic = comic
    console.log(comic)
    store.currentTabName = 'chapter'
  }

  searching.value = false
}

function sleep(ms: number): Promise<void> {
  return new Promise((resolve) => window.setTimeout(resolve, ms))
}

async function collectSearchResultIds(keyword: string, sort: SearchSort): Promise<number[]> {
  const ids = new Set<number>()
  const pageCount = searchPageCount.value

  for (let page = 1; page <= pageCount; page += 1) {
    let content = store.searchResult?.content
    if (page !== searchPage.value) {
      const result = await commands.search(keyword, page, sort)
      if (result.status === 'error') {
        console.error(result.error)
        continue
      }

      const searchResultVariant = result.data
      if (!('SearchResult' in searchResultVariant)) {
        continue
      }

      content = searchResultVariant.SearchResult.content
    }

    content?.forEach((comic) => {
      if (!comic.isDownloaded) {
        ids.add(comic.id)
      }
    })

    await sleep(150)
  }

  return Array.from(ids)
}

async function downloadAllSearchResults() {
  if (downloadingAllSearchResults.value) {
    message.warning('正在批量加入下载队列，请稍后再试')
    return
  }

  if (searching.value) {
    message.warning('有搜索正在进行，请稍后再试')
    return
  }

  if (store.searchResult === undefined || searchPageCount.value <= 0) {
    message.warning('没有可下载的搜索结果')
    return
  }

  const keyword = searchInput.value.trim()
  if (keyword.length === 0) {
    message.warning('请先输入关键词并完成搜索')
    return
  }

  downloadingAllSearchResults.value = true

  let queuedCount = 0
  let failedCount = 0

  try {
    const ids = await collectSearchResultIds(keyword, sortSelected.value)
    if (ids.length === 0) {
      message.warning('没有新的搜索结果需要下载')
      return
    }

    for (const id of ids) {
      const result = await commands.downloadComic(id)
      if (result.status === 'error') {
        console.error(result.error)
        failedCount += 1
      } else {
        queuedCount += 1
      }
      await sleep(150)
    }

    message.success(`已加入队列：${queuedCount}，失败：${failedCount}，跳过已下载：${Math.max(0, store.searchResult.total - ids.length)}`)
  } finally {
    downloadingAllSearchResults.value = false
  }
}
</script>

<template>
  <div class="h-full flex flex-col gap-2">
    <n-input-group class="box-border px-2 pt-2">
      <FloatLabelInput
        label="关键词(jm号也可以)"
        size="small"
        v-model:value="searchInput"
        clearable
        @keydown.enter="search(searchInput.trim(), 1, sortSelected)" />
      <n-select
        class="w-45%"
        v-model:value="sortSelected"
        :options="sortOptions"
        :show-checkmark="false"
        size="small"
        @update-value="search(searchInput.trim(), 1, $event)" />
      <n-button
        :loading="searching"
        type="primary"
        size="small"
        class="w-15%"
        @click="search(searchInput.trim(), 1, sortSelected)">
        <template #icon>
          <n-icon size="22">
            <PhMagnifyingGlass />
          </n-icon>
        </template>
      </n-button>
    </n-input-group>

    <n-button
      v-if="store.searchResult !== undefined"
      class="mx-2"
      type="primary"
      secondary
      size="small"
      :loading="downloadingAllSearchResults"
      :disabled="searchPageCount <= 0"
      @click="downloadAllSearchResults">
      一键下载全部搜索结果（{{ store.searchResult.total }}）
    </n-button>

    <div v-if="store.searchResult !== undefined" class="flex flex-col gap-row-2 overflow-auto box-border px-2">
      <ComicCard
        v-for="comicInSearch in store.searchResult.content"
        :key="comicInSearch.id"
        :comic-id="comicInSearch.id"
        :comic-title="comicInSearch.name"
        :comic-author="comicInSearch.author"
        :comic-category="comicInSearch.category"
        :comic-category-sub="comicInSearch.categorySub"
        :comic-downloaded="comicInSearch.isDownloaded"
        :comic-download-dir="comicInSearch.comicDownloadDir" />
    </div>

    <n-pagination
      v-if="searchPageCount > 0"
      class="box-border p-2 pt-0 mt-auto"
      :page-count="searchPageCount"
      :page="searchPage"
      @update:page="search(searchInput.trim(), $event, sortSelected)" />
  </div>
</template>
