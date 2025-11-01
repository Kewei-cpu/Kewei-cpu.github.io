<script setup lang="ts">
import { ref } from 'vue'
import EmojiGuess from './components/EmojiGuess.vue'
import SearchEmoji from './components/SearchEmoji.vue'
import AllEmojis from './components/AllEmojis.vue'

type Emoji = {
  emoji: string
  name: string
  shortname: string
  unicode: string
  html: string
  category: string
  order: string
}

const activeTab = ref('guess')
const emojis = ref<Emoji[]>([])
const loading = ref(true)

// Load emojis data
const loadEmojis = async () => {
  try {
    const response = await fetch('/emojis.json')
    const data = await response.json()

    // Filter out skin tone variants and complex emojis
    // Keep only base emojis without skin tone modifiers
    const filtered = data.emojis.filter((e: Emoji) => {
      // Remove emojis with skin tone modifiers (1F3FB-1F3FF)
      const hasSkinTone = /1F3FB|1F3FC|1F3FD|1F3FE|1F3FF/.test(e.unicode)
      // Remove complex family emojis with zero-width joiners
      const hasZWJ = /200D/.test(e.unicode)

      // Keep simple emojis without skin tones and complex families
      return !hasSkinTone && !hasZWJ
    })

    // Sort by name for better organization
    emojis.value = filtered.sort((a: Emoji, b: Emoji) => {
      return a.name.localeCompare(b.name)
    })
  } catch (error) {
    console.error('Failed to load emojis:', error)
  } finally {
    loading.value = false
  }
}

loadEmojis()

const tabs = [
  { id: 'guess', label: 'Emoji Guess' },
  { id: 'search', label: 'Search' },
  { id: 'all', label: 'All Emojis' }
]
</script>

<template>
  <div class="min-h-screen bg-gradient-to-br from-purple-500 to-pink-500">
    <div class="container mx-auto px-4 py-8">
      <h1 class="text-4xl font-bold text-white text-center mb-8">
        🎯 Emoji Guess
      </h1>

      <div class="bg-white rounded-lg shadow-xl p-6">
        <!-- Tab Navigation -->
        <div class="flex flex-wrap gap-2 mb-6 border-b">
          <button v-for="tab in tabs" :key="tab.id" @click="activeTab = tab.id" :class="[
            'px-4 py-2 rounded-t-lg font-medium transition-colors',
            activeTab === tab.id
              ? 'bg-purple-500 text-white'
              : 'text-gray-600 hover:bg-gray-100'
          ]">
            {{ tab.label }}
          </button>
        </div>

        <!-- Loading State -->
        <div v-if="loading" class="text-center py-8">
          <div class="animate-spin rounded-full h-12 w-12 border-b-2 border-purple-500 mx-auto"></div>
          <p class="mt-4 text-gray-600">Loading emojis...</p>
        </div>

        <!-- Tab Content -->
        <div v-else>
          <!-- Emoji Guess Tab -->
          <div v-if="activeTab === 'guess'" class="space-y-4">
            <EmojiGuess :emojis="emojis" />
          </div>

          <!-- Search Tab -->
          <div v-if="activeTab === 'search'" class="space-y-4">
            <SearchEmoji :emojis="emojis" />
          </div>

          <!-- All Emojis Tab -->
          <div v-if="activeTab === 'all'" class="space-y-4">
            <AllEmojis :emojis="emojis" />
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
