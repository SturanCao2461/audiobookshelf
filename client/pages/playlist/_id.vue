<template>
  <div id="page-wrapper" class="bg-bg page overflow-hidden" :class="streamLibraryItem ? 'streaming' : ''">
    <div class="w-full h-full overflow-y-auto px-2 py-6 md:p-8">
      <div class="flex flex-col sm:flex-row max-w-6xl mx-auto">
        <div class="w-full flex justify-center md:block sm:w-32 md:w-52" style="min-width: 200px">
          <div class="relative" style="height: fit-content">
            <covers-playlist-cover :items="playlistItems" :width="200" :height="200" />
          </div>
        </div>
        <div class="grow px-2 py-6 md:py-0 md:px-10">
          <div class="flex items-end flex-row flex-wrap md:flex-nowrap">
            <h1 class="text-2xl md:text-3xl font-sans w-full md:w-fit mb-4 md:mb-0">
              {{ playlistName }}
            </h1>
            <div class="grow" />

            <ui-btn v-if="showPlayButton" :disabled="streaming" color="bg-success" :padding-x="4" small class="flex items-center h-9 mr-2" @click="clickPlay">
              <span v-show="!streaming" class="material-symbols fill text-2xl -ml-2 pr-1 text-white">play_arrow</span>
              {{ streaming ? $strings.ButtonPlaying : $strings.ButtonPlayAll }}
            </ui-btn>

            <!-- Sort (dark-grey themed custom select) -->
            <div ref="sortWrap" class="relative h-9 ml-2">
              <button
                ref="sortBtn"
                type="button"
                class="h-9 w-36 sm:w-44 md:w-48 pl-3 pr-9 rounded-md border border-white/10 bg-[#2b2b2b] text-gray-100 hover:bg-[#363636] focus:outline-none focus:ring-2 focus:ring-[#6aa0ff80] focus:border-white/20 transition-colors flex items-center justify-between"
                @click="isSortOpen = !isSortOpen"
                @keydown.enter.prevent="isSortOpen = !isSortOpen"
                @keydown.space.prevent="isSortOpen = !isSortOpen"
                @keydown.esc.prevent="isSortOpen = false"
                aria-haspopup="listbox"
                :aria-expanded="isSortOpen ? 'true' : 'false'"
              >
                <span class="truncate text-sm">{{ sortLabel }}</span>
                <span class="material-symbols text-base text-gray-300">expand_more</span>
              </button>

              <!-- dragdown -->
              <div v-show="isSortOpen" class="absolute z-20 mt-1 w-36 sm:w-44 md:w-48 bg-[#2b2b2b] text-gray-100 border border-white/10 rounded-md shadow-lg ring-1 ring-black/5 max-h-80 overflow-auto" role="listbox">
                <div v-for="it in sortItems" :key="it.value" role="option" :aria-selected="it.value === sortOption ? 'true' : 'false'" @click="setSort(it.value)" class="px-3 py-2 text-sm cursor-pointer select-none flex items-center justify-between" :class="it.value === sortOption ? 'bg-[#363636] text-yellow-400' : 'text-gray-200 hover:bg-[#3a3a3a] hover:text-white'">
                  <span class="truncate">{{ it.text }}</span>
                  <span v-if="it.value === sortOption" class="material-symbols text-yellow-400">check</span>
                </div>
              </div>
            </div>

            <ui-icon-btn icon="edit" class="mx-0.5" @click="editClick" />

            <ui-icon-btn icon="delete" class="mx-0.5" @click="removeClick" />
          </div>

          <div class="my-8 max-w-2xl">
            <p class="text-base text-gray-100">{{ description }}</p>
          </div>

          <tables-playlist-items-table :items="sortedItems" :playlist-id="playlistId" />
        </div>
      </div>
    </div>
    <div v-show="processingRemove" class="absolute top-0 left-0 w-full h-full z-10 bg-black/40 flex items-center justify-center">
      <ui-loading-indicator />
    </div>
  </div>
</template>

<script>
export default {
  async asyncData({ store, params, app, redirect, route }) {
    if (!store.state.user.user) {
      return redirect(`/login?redirect=${route.path}`)
    }
    var playlist = await app.$axios.$get(`/api/playlists/${params.id}`).catch((error) => {
      console.error('Failed', error)
      return false
    })
    if (!playlist) {
      return redirect('/')
    }

    // If playlist is a different library then set library as current
    if (playlist.libraryId !== store.state.libraries.currentLibraryId) {
      await store.dispatch('libraries/fetch', playlist.libraryId)
    }

    store.commit('libraries/addUpdateUserPlaylist', playlist)
    return {
      playlistId: playlist.id
    }
  },
  data() {
    return {
      processingRemove: false,
      sortOption: 'added_desc',
      isSortOpen: false,
      sortItems: [
        { text: 'Added ↓', value: 'added_desc' },
        { text: 'Added ↑', value: 'added_asc' },
        { text: 'Title A→Z', value: 'title_asc' },
        { text: 'Title Z→A', value: 'title_desc' },
        { text: 'Author A→Z', value: 'author_asc' },
        { text: 'Author Z→A', value: 'author_desc' },
        { text: 'Read First', value: 'read_first' },
        { text: 'Unread First', value: 'unread_first' }
      ]
    }
  },
  computed: {
    streamLibraryItem() {
      return this.$store.state.streamLibraryItem
    },
    playlistItems() {
      return this.playlist.items || []
    },
    playlistName() {
      return this.playlist.name || ''
    },
    description() {
      return this.playlist.description || ''
    },
    playlist() {
      return this.$store.getters['libraries/getPlaylist'](this.playlistId) || {}
    },
    playableItems() {
      return this.playlistItems.filter((item) => {
        const libraryItem = item.libraryItem
        if (libraryItem.isMissing || libraryItem.isInvalid) return false
        if (item.episode) return item.episode.audioFile
        return libraryItem.media.tracks.length
      })
    },
    streaming() {
      return !!this.playableItems.find((i) => this.$store.getters['getIsMediaStreaming'](i.libraryItemId, i.episodeId))
    },
    showPlayButton() {
      return this.playableItems.length
    },
    userCanUpdate() {
      return this.$store.getters['user/getUserCanUpdate']
    },
    userCanDelete() {
      return this.$store.getters['user/getUserCanDelete']
    },
    baseItems() {
      if (this.filteredItems) return this.filteredItems
      return this.playlist && this.playlist.items ? this.playlist.items : []
    },
    sortLabel() {
      const found = this.sortItems.find((s) => s.value === this.sortOption)
      return found ? found.text : ''
    },

    // 2) ✅ 排序后的最终列表：绑定给子表格组件
    sortedItems() {
      // 稳定排序：把原始 index 带入作为兜底
      const withIdx = this.baseItems.map((it, idx) => ({ it, idx }))

      const getTitle = (it) => (it.episode ? it.episode.title || '' : it.libraryItem?.media?.metadata?.title || '')

      const getAuthorKey = (it) => {
        if (it.episode) return '' // 剧集没有作者就返回空串
        const authors = it.libraryItem?.media?.metadata?.authors || []
        // authors 是对象数组 { id, name }，用 name 拼接做排序键
        return authors.map((a) => a?.name || '').join(', ')
      }

      const getIsFinished = (it) => {
        const libraryItemId = it.libraryItem?.id
        const episodeId = it.episode ? it.episode.id : null
        const prog = this.$store.getters['user/getUserMediaProgress'](libraryItemId, episodeId)
        return !!(prog && prog.isFinished)
      }

      const getAdded = (it, idx) => {
        // 若后端有时间字段（addedAt/createdAt），优先用；否则用原始顺序 idx 当“添加序”
        return it.addedAt || it.createdAt || idx
      }

      const cmpStr = (a, b) => a.localeCompare(b, undefined, { sensitivity: 'base' })
      const cmpBool = (a, b) => (a === b ? 0 : a ? -1 : 1) // true 在前
      const byIdx = (a, b) => a.idx - b.idx // 稳定性兜底

      switch (this.sortOption) {
        case 'title_asc':
          withIdx.sort((A, B) => cmpStr(getTitle(A.it), getTitle(B.it)) || byIdx(A, B))
          break
        case 'title_desc':
          withIdx.sort((A, B) => cmpStr(getTitle(B.it), getTitle(A.it)) || byIdx(A, B))
          break
        case 'author_asc':
          withIdx.sort((A, B) => cmpStr(getAuthorKey(A.it), getAuthorKey(B.it)) || byIdx(A, B))
          break
        case 'author_desc':
          withIdx.sort((A, B) => cmpStr(getAuthorKey(B.it), getAuthorKey(A.it)) || byIdx(A, B))
          break
        case 'added_asc':
          withIdx.sort((A, B) => {
            const a = getAdded(A.it, A.idx)
            const b = getAdded(B.it, B.idx)
            // 日期字符串就转 Date，比数字就直接比
            const va = isNaN(+a) ? +new Date(a) : +a
            const vb = isNaN(+b) ? +new Date(b) : +b
            return va - vb || byIdx(A, B)
          })
          break
        case 'added_desc':
          withIdx.sort((A, B) => {
            const a = getAdded(A.it, A.idx)
            const b = getAdded(B.it, B.idx)
            const va = isNaN(+a) ? +new Date(a) : +a
            const vb = isNaN(+b) ? +new Date(b) : +b
            return vb - va || byIdx(A, B)
          })
          break
        case 'read_first':
          withIdx.sort((A, B) => {
            const da = getIsFinished(A.it)
            const db = getIsFinished(B.it)
            return cmpBool(da, db) || byIdx(A, B)
          })
          break
        case 'unread_first':
          withIdx.sort((A, B) => {
            const da = getIsFinished(A.it)
            const db = getIsFinished(B.it)
            // 未读优先：把上面的结果取反
            return cmpBool(db, da) || byIdx(A, B)
          })
          break
        default:
          // fallback：不排序，保持原顺序
          break
      }

      return withIdx.map((x) => x.it)
    }
  },
  methods: {
    editClick() {
      this.$store.commit('globals/setEditPlaylist', this.playlist)
    },
    removeClick() {
      const payload = {
        message: this.$getString('MessageConfirmRemovePlaylist', [this.playlistName]),
        callback: (confirmed) => {
          if (confirmed) {
            this.removePlaylist()
          }
        },
        type: 'yesNo'
      }
      this.$store.commit('globals/setConfirmPrompt', payload)
    },
    removePlaylist() {
      this.processingRemove = true
      this.$axios
        .$delete(`/api/playlists/${this.playlist.id}`)
        .then(() => {
          this.$toast.success(this.$strings.ToastPlaylistRemoveSuccess)
        })
        .catch((error) => {
          console.error('Failed to remove playlist', error)
          this.$toast.error(this.$strings.ToastRemoveFailed)
        })
        .finally(() => {
          this.processingRemove = false
        })
    },
    setSort(value) {
      this.sortOption = value
      this.isSortOpen = false
      this.$nextTick(() => this.$refs.sortBtn && this.$refs.sortBtn.focus())
    },
    onDocClick(e) {
      const el = this.$refs.sortWrap
      if (this.isSortOpen && el && !el.contains(e.target)) {
        this.isSortOpen = false
      }
    },
    clickPlay() {
      const queueItems = []

      // Playlist queue will start at the first unfinished item
      //   if all items are finished then entire playlist is queued
      const itemsWithProgress = this.playableItems.map((item) => {
        return {
          ...item,
          progress: this.$store.getters['user/getUserMediaProgress'](item.libraryItemId, item.episodeId)
        }
      })

      const hasUnfinishedItems = itemsWithProgress.some((i) => !i.progress || !i.progress.isFinished)
      if (!hasUnfinishedItems) {
        console.warn('All items in playlist are finished - starting at first item')
      }

      for (let i = 0; i < itemsWithProgress.length; i++) {
        const playlistItem = itemsWithProgress[i]
        if (!hasUnfinishedItems || !playlistItem.progress || !playlistItem.progress.isFinished) {
          const libraryItem = playlistItem.libraryItem
          if (playlistItem.episode) {
            queueItems.push({
              libraryItemId: libraryItem.id,
              libraryId: libraryItem.libraryId,
              episodeId: playlistItem.episode.id,
              title: playlistItem.episode.title,
              subtitle: libraryItem.media.metadata.title,
              caption: '',
              duration: playlistItem.episode.duration || null,
              coverPath: libraryItem.media.coverPath || null
            })
          } else {
            queueItems.push({
              libraryItemId: libraryItem.id,
              libraryId: libraryItem.libraryId,
              episodeId: null,
              title: libraryItem.media.metadata.title,
              subtitle: libraryItem.media.metadata.authors.map((au) => au.name).join(', '),
              caption: '',
              duration: libraryItem.media.duration || null,
              coverPath: libraryItem.media.coverPath || null
            })
          }
        }
      }

      if (queueItems.length >= 0) {
        this.$eventBus.$emit('play-item', {
          libraryItemId: queueItems[0].libraryItemId,
          episodeId: queueItems[0].episodeId,
          queueItems
        })
      }
    }
  },
  mounted() {
    document.addEventListener('click', this.onDocClick, { capture: true })
  },
  beforeDestroy() {
    document.removeEventListener('click', this.onDocClick, { capture: true })
  }
}
</script>
