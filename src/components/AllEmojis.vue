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

const selectedEmoji = ref<Emoji | null>(null)
const selectedLevel1 = ref('all')
const selectedLevel2 = ref('all')

// Parse category and extract level1 and level2
const parseCategory = (category: string) => {
  const match = category.match(/^(.+?)\s*\((.+)\)$/)
  if (match && match[1] && match[2]) {
    return {
      level1: match[1].trim(),
      level2: match[2].trim()
    }
  }
  return {
    level1: category,
    level2: null
  }
}

// Get unique level1 categories
const level1Categories = computed(() => {
  const level1Set = new Set<string>()
  props.emojis.forEach(e => {
    const parsed = parseCategory(e.category)
    level1Set.add(parsed.level1)
  })
  return Array.from(level1Set).sort()
})

// Get level2 categories for a given level1
const getLevel2Categories = (level1: string) => {
  const level2Set = new Set<string>()
  props.emojis.forEach(e => {
    const parsed = parseCategory(e.category)
    if (parsed.level1 === level1 && parsed.level2) {
      level2Set.add(parsed.level2)
    }
  })
  return Array.from(level2Set).sort()
}

// Filter emojis by selected categories
const filteredEmojis = computed(() => {
  if (selectedLevel1.value === 'all') {
    return props.emojis
  }

  return props.emojis.filter(emoji => {
    const parsed = parseCategory(emoji.category)

    if (selectedLevel2.value === 'all') {
      return parsed.level1 === selectedLevel1.value
    }

    return parsed.level1 === selectedLevel1.value && parsed.level2 === selectedLevel2.value
  })
})

const selectEmoji = (emoji: Emoji) => {
  selectedEmoji.value = emoji
}

const clearSelection = () => {
  selectedEmoji.value = null
}

// Handle level1 selection
const selectLevel1 = (level1: string) => {
  selectedLevel1.value = level1
  selectedLevel2.value = 'all' // Reset level2 when level1 changes
}

// Handle level2 selection
const selectLevel2 = (level2: string) => {
  selectedLevel2.value = level2
}

// Get emoji count for a specific category
const getEmojiCount = (level1: string, level2?: string) => {
  if (level1 === 'all') {
    return props.emojis.length
  }
  return props.emojis.filter(emoji => {
    const parsed = parseCategory(emoji.category)
    if (level2) {
      return parsed.level1 === level1 && parsed.level2 === level2
    }
    return parsed.level1 === level1
  }).length
}
</script>

<template>
  <div class="space-y-6">
    <!-- Category filter -->
    <div class="space-y-4">
      <label class="block text-lg font-medium text-gray-700">
        Filter by category:
      </label>
      <div class="flex flex-wrap gap-2">
        <!-- All button -->
        <button @click="selectLevel1('all')" :class="[
          'px-4 py-2 rounded-lg font-medium transition-colors',
          selectedLevel1 === 'all'
            ? 'bg-teal-500 text-white'
            : 'bg-gray-200 text-gray-700 hover:bg-gray-300'
        ]">
          All ({{ emojis.length }})
        </button>

        <!-- Level 1 categories -->
        <button v-for="level1 in level1Categories" :key="level1" @click="selectLevel1(level1)" :class="[
          'px-4 py-2 rounded-lg font-medium transition-colors',
          selectedLevel1 === level1
            ? 'bg-teal-500 text-white'
            : 'bg-gray-200 text-gray-700 hover:bg-gray-300'
        ]">
          {{ level1 }} ({{ getEmojiCount(level1) }})
        </button>
      </div>

      <!-- Level 2 categories (shown when a level1 is selected) -->
      <div v-if="selectedLevel1 !== 'all'" class="ml-6 space-y-2">
        <div class="text-sm font-medium text-gray-600 mb-2">
          Subcategories:
        </div>
        <div class="flex flex-wrap gap-2">
          <button @click="selectLevel2('all')" :class="[
            'px-3 py-1.5 rounded-lg text-sm font-medium transition-colors',
            selectedLevel2 === 'all'
              ? 'bg-teal-400 text-white'
              : 'bg-gray-100 text-gray-600 hover:bg-gray-200'
          ]">
            All ({{ getEmojiCount(selectedLevel1) }})
          </button>
          <button v-for="level2 in getLevel2Categories(selectedLevel1)" :key="level2" @click="selectLevel2(level2)"
            :class="[
              'px-3 py-1.5 rounded-lg text-sm font-medium transition-colors',
              selectedLevel2 === level2
                ? 'bg-teal-400 text-white'
                : 'bg-gray-100 text-gray-600 hover:bg-gray-200'
            ]">
            {{ level2 }} ({{ getEmojiCount(selectedLevel1, level2) }})
          </button>
        </div>
      </div>
    </div>

    <!-- Stats -->
    <div class="text-center text-gray-600 bg-gray-100 rounded-lg p-4">
      <p>
        Showing <strong>{{ filteredEmojis.length }}</strong> emoji{{ filteredEmojis.length !== 1 ? 's' : '' }}
        <span v-if="selectedLevel1 !== 'all'">
          in "{{ selectedLevel1 }}"
          <span v-if="selectedLevel2 !== 'all'"> > "{{ selectedLevel2 }}"</span>
        </span>
      </p>
    </div>

    <!-- Emojis grid -->
    <div
      class="grid grid-cols-2 sm:grid-cols-3 md:grid-cols-4 lg:grid-cols-6 xl:grid-cols-8 gap-4 max-h-[600px] overflow-y-auto p-4 bg-gray-50 rounded-lg">
      <button v-for="emoji in filteredEmojis" :key="emoji.unicode + '-' + emoji.shortname" @click="selectEmoji(emoji)"
        class="flex flex-col items-center p-4 bg-white rounded-lg hover:bg-teal-50 hover:shadow-md transition-all cursor-pointer group">
        <div class="text-3xl mb-2">{{ emoji.emoji }}</div>
        <div class="text-xs text-gray-600 text-center line-clamp-2 group-hover:text-teal-600">
          {{ emoji.shortname.slice(1, -1).replace(/_/g, ' ') }}
        </div>
      </button>
    </div>

    <!-- Selected emoji details modal -->
    <div v-if="selectedEmoji" class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center p-4 z-50"
      @click="clearSelection">
      <div class="bg-white rounded-lg p-8 max-w-md w-full" @click.stop>
        <button @click="clearSelection" class="float-right text-gray-500 hover:text-gray-700 text-2xl">
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
