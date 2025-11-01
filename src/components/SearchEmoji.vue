<script setup lang="ts">
import { ref, computed } from 'vue'

type Emoji = {
  emoji: string
  name: string
  shortname: string
  unicode: string
  html: string
  category: string
  order: string
}

const props = defineProps<{
  emojis: Emoji[]
}>()

const searchQuery = ref('')
const selectedEmoji = ref<Emoji | null>(null)

// Search through emojis
const searchResults = computed(() => {
  if (!searchQuery.value.trim()) return []

  const query = searchQuery.value.toLowerCase().trim()

  return props.emojis.filter(emoji => {
    const name = emoji.name.toLowerCase()
    const shortname = emoji.shortname.slice(1, -1).toLowerCase()
    const category = emoji.category.toLowerCase()

    return (
      name.includes(query) ||
      shortname.includes(query) ||
      category.includes(query)
    )
  })
})

const selectEmoji = (emoji: Emoji) => {
  selectedEmoji.value = emoji
}

const clearSelection = () => {
  selectedEmoji.value = null
}
</script>

<template>
  <div class="space-y-6">
    <!-- Search input -->
    <div class="max-w-2xl mx-auto">
      <div class="relative">
        <input
          v-model="searchQuery"
          type="text"
          placeholder="Search emojis by name (e.g., heart, smile, face)..."
          class="w-full px-4 py-3 pr-12 border-2 border-gray-300 rounded-lg focus:border-purple-500 focus:outline-none text-lg"
        />
        <svg
          class="absolute right-4 top-1/2 transform -translate-y-1/2 w-6 h-6 text-gray-400"
          fill="none"
          stroke="currentColor"
          viewBox="0 0 24 24"
        >
          <path
            stroke-linecap="round"
            stroke-linejoin="round"
            stroke-width="2"
            d="M21 21l-6-6m2-5a7 7 0 11-14 0 7 7 0 0114 0z"
          />
        </svg>
      </div>
    </div>

    <!-- Search results -->
    <div v-if="searchQuery.trim()">
      <div class="mb-4 text-center">
        <p class="text-gray-600">
          Found <strong>{{ searchResults.length }}</strong> result{{ searchResults.length !== 1 ? 's' : '' }}
          for "<strong>{{ searchQuery }}</strong>"
        </p>
      </div>

      <!-- No results -->
      <div v-if="searchResults.length === 0" class="text-center py-8">
        <p class="text-gray-500 text-lg">No emojis found matching your search</p>
      </div>

      <!-- Results grid -->
      <div
        v-if="searchResults.length > 0"
        class="grid grid-cols-2 sm:grid-cols-3 md:grid-cols-4 lg:grid-cols-6 gap-4 max-h-96 overflow-y-auto p-4 bg-gray-50 rounded-lg"
      >
        <button
          v-for="emoji in searchResults"
          :key="emoji.unicode + '-' + emoji.shortname"
          @click="selectEmoji(emoji)"
          class="flex flex-col items-center p-4 bg-white rounded-lg hover:bg-purple-50 hover:shadow-md transition-all cursor-pointer group"
        >
          <div class="text-4xl mb-2">{{ emoji.emoji }}</div>
          <div class="text-xs text-gray-600 text-center line-clamp-2 group-hover:text-purple-600">
            {{ emoji.shortname.slice(1, -1).replace(/_/g, ' ') }}
          </div>
        </button>
      </div>
    </div>

    <!-- Default state -->
    <div v-else class="text-center py-12">
      <div class="text-6xl mb-4">🔍</div>
      <p class="text-gray-600 text-lg">Type to search for emojis</p>
      <p class="text-gray-500 mt-2">
        Search by name, shortname, or category
      </p>
    </div>

    <!-- Selected emoji details -->
    <div
      v-if="selectedEmoji"
      class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center p-4 z-50"
      @click="clearSelection"
    >
      <div
        class="bg-white rounded-lg p-8 max-w-md w-full"
        @click.stop
      >
        <button
          @click="clearSelection"
          class="float-right text-gray-500 hover:text-gray-700 text-2xl"
        >
          ×
        </button>

        <div class="text-center space-y-4">
          <div class="text-8xl">{{ selectedEmoji.emoji }}</div>

          <div class="space-y-2">
            <h3 class="text-xl font-bold text-gray-800">
              {{ selectedEmoji.name }}
            </h3>
            <p class="text-gray-600">
              <strong>Shortname:</strong> {{ selectedEmoji.shortname }}
            </p>
            <p class="text-gray-600">
              <strong>Category:</strong> {{ selectedEmoji.category }}
            </p>
            <p class="text-gray-600 text-sm">
              <strong>Unicode:</strong> {{ selectedEmoji.unicode }}
            </p>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
