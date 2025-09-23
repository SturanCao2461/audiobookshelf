<template>
  <div class="page relative" :class="streamLibraryItem ? 'streaming' : ''">
    <app-book-shelf-toolbar page="narrators" is-home />
    <div id="bookshelf" class="w-full h-full px-1 py-4 md:p-8 relative overflow-y-auto">
      <table class="tracksTable max-w-2xl mx-auto">
        <!-- Header Row -->
        <tr>
          <!-- First Name -->
          <th class="text-left">
            <button class="px-2 py-1 flex items-center gap-1" :class="sortKey === 'first' ? 'text-white' : 'text-gray-400 hover:text-gray-200'" @click="setSort('first')">
              First Name
              <span v-if="sortKey === 'first'" class="material-symbols text-sm">
                {{ sortDesc ? 'arrow_drop_down' : 'arrow_drop_up' }}
              </span>
            </button>
          </th>

          <!-- Last Name -->
          <th class="text-left">
            <button class="px-2 py-1 flex items-center gap-1" :class="sortKey === 'last' ? 'text-white' : 'text-gray-400 hover:text-gray-200'" @click="setSort('last')">
              Last Name
              <span v-if="sortKey === 'last'" class="material-symbols text-sm">
                {{ sortDesc ? 'arrow_drop_down' : 'arrow_drop_up' }}
              </span>
            </button>
          </th>

          <!-- Books -->
          <th class="text-center w-24">
            <button class="px-2 py-1 flex items-center gap-1 mx-auto" :class="sortKey === 'books' ? 'text-white' : 'text-gray-400 hover:text-gray-200'" @click="setSort('books')">
              {{ $strings.LabelBooks }}
              <span v-if="sortKey === 'books'" class="material-symbols text-sm">
                {{ sortDesc ? 'arrow_drop_down' : 'arrow_drop_up' }}
              </span>
            </button>
          </th>

          <th v-if="userCanUpdate" class="w-40"></th>
        </tr>

        <tr v-for="narrator in sortedNarrators" :key="narrator.id">
          <!-- First Name -->
          <td>
            <nuxt-link v-if="selectedNarrator?.id !== narrator.id" :to="`/library/${currentLibraryId}/bookshelf?filter=narrators.${narrator.id}`" class="text-sm md:text-base text-gray-100 hover:underline">
              {{ getFirst(narrator) }}
            </nuxt-link>
            <form v-else @submit.prevent="saveClick">
              <ui-text-input v-model="newNarratorName" />
            </form>
          </td>

          <!-- Last Name -->
          <td class="truncate">
            {{ getLast(narrator) }}
          </td>

          <!-- Books -->
          <td class="text-center w-24">
            <nuxt-link :to="`/library/${currentLibraryId}/bookshelf?filter=narrators.${narrator.id}`" class="hover:underline">
              {{ getBooksCount(narrator) }}
            </nuxt-link>
          </td>

          <td v-if="userCanUpdate" class="w-40">
            <div class="flex justify-end items-center h-10">
              <template v-if="selectedNarrator?.id !== narrator.id">
                <ui-icon-btn icon="edit" borderless :size="8" icon-font-size="1.1rem" class="mx-1" @click="editClick(narrator)" />
                <ui-icon-btn icon="delete" borderless :size="8" icon-font-size="1.1rem" @click="removeClick(narrator)" />
              </template>
              <template v-else>
                <ui-btn color="bg-success" small class="mr-2" @click.stop="saveClick">{{ $strings.ButtonSave }}</ui-btn>
                <ui-btn small @click.stop="cancelEditClick">{{ $strings.ButtonCancel }}</ui-btn>
              </template>
            </div>
          </td>
        </tr>
      </table>
    </div>

    <div v-if="loading" class="absolute top-0 left-0 w-full h-[calc(100%-40px)] mt-10 flex items-center justify-center bg-black/25">
      <ui-loading-indicator />
    </div>
  </div>
</template>

<script>
export default {
  async asyncData({ store, params, redirect }) {
    const libraryId = params.library
    const libraryData = await store.dispatch('libraries/fetch', libraryId)
    if (!libraryData) {
      return redirect('/oops?message=Library not found')
    }

    const library = libraryData.library
    if (library.mediaType === 'podcast') {
      return redirect(`/library/${libraryId}`)
    }

    return { libraryId }
  },
  data() {
    return {
      loading: true,
      narrators: [],
      selectedNarrator: null,
      newNarratorName: null,
      sortKey: 'first',
      sortDesc: false
    }
  },
  computed: {
    streamLibraryItem() {
      return this.$store.state.streamLibraryItem
    },
    currentLibraryId() {
      return this.$store.state.libraries.currentLibraryId
    },
    userCanUpdate() {
      return this.$store.getters['user/getUserCanUpdate']
    },

    sortedNarrators() {
      const rows = this.narrators.map((it, idx) => ({ it, idx }))

      const first = (n) => (n.name || '').trim().split(/\s+/)[0] || ''
      const last = (n) => {
        const parts = (n.name || '').trim().split(/\s+/)
        return parts.length > 1 ? parts[parts.length - 1] : ''
      }
      const books = (n) => Number(n.numBooks || n.bookCount || 0)

      const cmpStr = (a, b) => a.localeCompare(b, undefined, { sensitivity: 'base' })
      const cmpNum = (a, b) => a - b
      const byIdx = (A, B) => A.idx - B.idx

      let cmp
      if (this.sortKey === 'first') cmp = (A, B) => cmpStr(first(A.it), first(B.it))
      else if (this.sortKey === 'last') cmp = (A, B) => cmpStr(last(A.it), last(B.it))
      else cmp = (A, B) => cmpNum(books(A.it), books(B.it))

      rows.sort((A, B) => {
        const r = cmp(A, B)
        const s = this.sortDesc ? -r : r
        return s || byIdx(A, B)
      })

      return rows.map((x) => x.it)
    }
  },
  methods: {
    removeClick(narrator) {
      const payload = {
        message: this.$getString('MessageConfirmRemoveNarrator', [narrator.name]),
        callback: (confirmed) => {
          if (confirmed) this.removeNarrator(narrator.id)
        },
        type: 'yesNo'
      }
      this.$store.commit('globals/setConfirmPrompt', payload)
    },
    editClick(narrator) {
      this.selectedNarrator = narrator
      this.newNarratorName = narrator.name
    },
    cancelEditClick() {
      this.selectedNarrator = null
      this.newNarratorName = null
    },
    saveClick() {
      if (!this.selectedNarrator) return
      this.newNarratorName = this.newNarratorName?.trim() || ''
      if (!this.newNarratorName || this.newNarratorName === this.selectedNarrator.name) {
        this.cancelEditClick()
        return
      }

      this.loading = true
      this.$axios
        .$patch(`/api/libraries/${this.currentLibraryId}/narrators/${this.selectedNarrator.id}`, { name: this.newNarratorName })
        .then((data) => {
          if (data.updated) {
            this.$toast.success(this.$getString('MessageItemsUpdated', [data.updated]))
          } else {
            this.$toast.info(this.$strings.MessageNoUpdatesWereNecessary)
          }
          this.cancelEditClick()
          this.init()
        })
        .catch((error) => {
          console.error('Failed to updated narrator', error)
          this.$toast.error(this.$strings.ToastFailedToUpdate)
          this.loading = false
        })
    },
    removeNarrator(id) {
      this.loading = true
      this.$axios
        .$delete(`/api/libraries/${this.currentLibraryId}/narrators/${id}`)
        .then((data) => {
          if (data.updated) {
            this.$toast.success(this.$getString('MessageItemsUpdated', [data.updated]))
          } else {
            this.$toast.info(this.$strings.MessageNoUpdatesWereNecessary)
          }
          this.init()
        })
        .catch((error) => {
          console.error('Failed to remove narrator', error)
          this.$toast.error(this.$strings.ToastRemoveFailed)
          this.loading = false
        })
    },
    async init() {
      this.narrators = await this.$axios
        .$get(`/api/libraries/${this.currentLibraryId}/narrators`)
        .then((response) => response.narrators)
        .catch((error) => {
          console.error('Failed to load narrators', error)
          return []
        })
      this.loading = false
    },

    setSort(key) {
      if (this.sortKey === key) this.sortDesc = !this.sortDesc
      else {
        this.sortKey = key
        this.sortDesc = false
      }
    },
    getFirst(n) {
      return (n.name || '').trim().split(/\s+/)[0] || ''
    },
    getLast(n) {
      const parts = (n.name || '').trim().split(/\s+/)
      return parts.length > 1 ? parts[parts.length - 1] : ''
    },
    getBooksCount(n) {
      return Number(n.numBooks || n.bookCount || 0)
    }
  },
  mounted() {
    this.init()
  },
  beforeDestroy() {}
}
</script>
