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

type GuessRow = {
  inputs: string[]
  letterStates: ('correct' | 'present' | 'absent' | null)[]
  isLocked: boolean
}

const props = defineProps<{
  emojis: Emoji[]
}>()

const currentEmoji = ref<Emoji | null>(null)
const guessRows = ref<GuessRow[]>([])
const currentRowIndex = ref(0)
const showResult = ref(false)
const isCorrect = ref(false)
const score = ref(0)
const totalGuesses = ref(0)

// Game filters
const selectedLetter = ref<string | null>(null)
const minLength = ref<number>(3)
const maxLength = ref<number>(15)
const selectedLevel1 = ref('all')
const selectedLevel2 = ref('all')
const showFilters = ref(true)

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

// Get available letters based on emoji names
const availableLetters = computed(() => {
  const letters = new Set<string>()
  props.emojis.forEach(emoji => {
    const firstChar = emoji.name.charAt(0).toLowerCase()
    if (/[a-z]/.test(firstChar)) {
      letters.add(firstChar)
    }
  })
  return Array.from(letters).sort()
})

// Filter emojis by selected criteria
const filteredEmojis = computed(() => {
  return props.emojis.filter(emoji => {
    const expectedName = emoji.shortname
      .slice(1, -1)
      .replace(/_/g, ' ')
    const length = expectedName.replace(/ /g, '').length

    const letterMatch = !selectedLetter.value ||
      emoji.name.charAt(0).toLowerCase() === selectedLetter.value

    const lengthMatch = length >= minLength.value && length <= maxLength.value

    // Category filter
    const parsed = parseCategory(emoji.category)
    let categoryMatch = true
    if (selectedLevel1.value !== 'all') {
      if (selectedLevel2.value === 'all') {
        categoryMatch = parsed.level1 === selectedLevel1.value
      } else {
        categoryMatch = parsed.level1 === selectedLevel1.value && parsed.level2 === selectedLevel2.value
      }
    }

    return letterMatch && lengthMatch && categoryMatch
  })
})

// Calculate input boxes based on the emoji name
const inputBoxes = computed(() => {
  if (!currentEmoji.value) return []

  const expectedName = currentEmoji.value.shortname
    .slice(1, -1)
    .replace(/_/g, ' ')

  const words = expectedName.split(' ')
  const boxes: { isSpace?: boolean }[] = []

  words.forEach((word, wordIndex) => {
    for (let i = 0; i < word.length; i++) {
      boxes.push({})
    }
    if (wordIndex < words.length - 1) {
      boxes.push({ isSpace: true })
    }
  })

  return boxes
})

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

const selectRandomEmoji = () => {
  const emojisToUse = filteredEmojis.value.length > 0 ? filteredEmojis.value : props.emojis

  if (emojisToUse.length === 0) {
    currentEmoji.value = null
    return
  }
  const randomIndex = Math.floor(Math.random() * emojisToUse.length)
  currentEmoji.value = emojisToUse[randomIndex] || null
  guessRows.value = []
  currentRowIndex.value = 0
  showResult.value = false
  isCorrect.value = false
  showFilters.value = false
  addNewRow()
}

const addNewRow = () => {
  const boxCount = inputBoxes.value.filter(box => !box.isSpace).length
  guessRows.value.push({
    inputs: new Array(boxCount).fill(''),
    letterStates: new Array(boxCount).fill(null),
    isLocked: false
  })
  currentRowIndex.value = guessRows.value.length - 1

  // Auto-focus first input
  setTimeout(() => {
    const firstBoxIndex = inputBoxes.value.findIndex(box => !box.isSpace)
    if (firstBoxIndex !== -1) {
      const firstInput = document.querySelector(`input[data-row="${currentRowIndex.value}"][data-box="${firstBoxIndex}"]`) as HTMLInputElement
      if (firstInput) firstInput.focus()
    }
  }, 100)
}

// Helper function to find the next non-space input box index
const findNextInputBoxIndex = (currentBoxIndex: number): number | null => {
  if (!inputBoxes.value) return null
  for (let i = currentBoxIndex + 1; i < inputBoxes.value.length; i++) {
    const box = inputBoxes.value[i]
    if (box && !box.isSpace) {
      return i
    }
  }
  return null
}

// Helper function to find the previous non-space input box index
const findPrevInputBoxIndex = (currentBoxIndex: number): number | null => {
  if (!inputBoxes.value) return null
  for (let i = currentBoxIndex - 1; i >= 0; i--) {
    const box = inputBoxes.value[i]
    if (box && !box.isSpace) {
      return i
    }
  }
  return null
}

// Helper function to get the input index (0-based in row.inputs) from box index
const getInputIndexFromBoxIndex = (boxIndex: number): number => {
  if (!inputBoxes.value) return 0
  let count = 0
  for (let i = 0; i < boxIndex; i++) {
    const box = inputBoxes.value[i]
    if (box && !box.isSpace) {
      count++
    }
  }
  return count
}

const handleKeydown = (rowIndex: number, boxIndex: number, event: KeyboardEvent) => {
  const row = guessRows.value[rowIndex]
  if (!row || row.isLocked) return

  const inputIndex = getInputIndexFromBoxIndex(boxIndex)

  if (event.key === 'Backspace') {
    event.preventDefault()
    if (row.inputs[inputIndex]) {
      row.inputs[inputIndex] = ''
    } else {
      const prevBoxIndex = findPrevInputBoxIndex(boxIndex)
      if (prevBoxIndex !== null) {
        const prevInputIndex = getInputIndexFromBoxIndex(prevBoxIndex)
        row.inputs[prevInputIndex] = ''
        const prevInput = document.querySelector(`input[data-row="${rowIndex}"][data-box="${prevBoxIndex}"]`) as HTMLInputElement
        if (prevInput) prevInput.focus()
      }
    }
  } else if (event.key === 'ArrowLeft') {
    event.preventDefault()
    const prevBoxIndex = findPrevInputBoxIndex(boxIndex)
    if (prevBoxIndex !== null) {
      const prevInput = document.querySelector(`input[data-row="${rowIndex}"][data-box="${prevBoxIndex}"]`) as HTMLInputElement
      if (prevInput) prevInput.focus()
    }
  } else if (event.key === 'ArrowRight') {
    event.preventDefault()
    const nextBoxIndex = findNextInputBoxIndex(boxIndex)
    if (nextBoxIndex !== null) {
      const nextInput = document.querySelector(`input[data-row="${rowIndex}"][data-box="${nextBoxIndex}"]`) as HTMLInputElement
      if (nextInput) nextInput.focus()
    }
  } else if (event.key === 'Enter') {
    event.preventDefault()
    // If this is the last input box, submit the guess
    const nextBoxIndex = findNextInputBoxIndex(boxIndex)
    if (nextBoxIndex === null) {
      submitGuess()
    }
  }
}

// Handle input changes
const handleInput = (rowIndex: number, boxIndex: number, event: Event) => {
  const row = guessRows.value[rowIndex]
  if (!row || row.isLocked) return

  const inputIndex = getInputIndexFromBoxIndex(boxIndex)
  const target = event.target as HTMLInputElement
  const value = target.value.slice(-1).toLowerCase()

  if (/[a-z]/.test(value)) {
    row.inputs[inputIndex] = value
    // Auto-advance to next input
    const nextBoxIndex = findNextInputBoxIndex(boxIndex)
    if (nextBoxIndex !== null) {
      const nextInput = document.querySelector(`input[data-row="${rowIndex}"][data-box="${nextBoxIndex}"]`) as HTMLInputElement
      if (nextInput) nextInput.focus()
    }
  } else if (value === '') {
    row.inputs[inputIndex] = ''
  }
}

const giveUp = () => {
  if (!currentEmoji.value) return

  isCorrect.value = false
  showResult.value = true

  const expectedName = currentEmoji.value.shortname
    .slice(1, -1)
    .replace(/_/g, ' ')
    .toLowerCase()

  const currentRow = guessRows.value[currentRowIndex.value]
  if (currentRow) {
    currentRow.isLocked = true
  }
}

const submitGuess = () => {
  if (!currentEmoji.value || showResult.value) return

  const currentRow = guessRows.value[currentRowIndex.value]
  if (!currentRow) return

  // Check if all inputs are filled
  const hasEmptyInput = currentRow.inputs.some(input => !input || input.trim() === '')
  if (hasEmptyInput) return

  totalGuesses.value++

  // Extract the expected name
  const expectedName = currentEmoji.value.shortname
    .slice(1, -1)
    .replace(/_/g, ' ')
    .toLowerCase()

  // Build guess from inputs
  const guess = currentRow.inputs.join('').toLowerCase()

  // Check if correct (remove spaces from expectedName for comparison)
  isCorrect.value = guess === expectedName.replace(/ /g, '')

  if (isCorrect.value) {
    score.value++
    // Mark all letters as correct
    for (let i = 0; i < currentRow.inputs.length; i++) {
      currentRow.letterStates[i] = 'correct'
    }
    currentRow.isLocked = true
    showResult.value = true
  } else {
    // Wordle-style feedback with correct duplicate handling
    const expected = expectedName.replace(/ /g, '').split('')
    const guessLetters = currentRow.inputs
    const states: ('correct' | 'present' | 'absent')[] = new Array(guessLetters.length)

    // First pass: mark correct positions and count remaining letters in expected
    const expectedLetterCounts: { [key: string]: number } = {}
    for (let i = 0; i < expected.length; i++) {
      const letter = expected[i]!
      if (guessLetters[i] === letter) {
        states[i] = 'correct'
      } else {
        expectedLetterCounts[letter] = (expectedLetterCounts[letter] || 0) + 1
      }
    }

    // Second pass: mark present/absent based on remaining letter counts
    for (let i = 0; i < guessLetters.length; i++) {
      if (states[i] === 'correct') continue

      const letter = guessLetters[i]!
      const count = expectedLetterCounts[letter]
      if (count !== undefined && count > 0) {
        states[i] = 'present'
        expectedLetterCounts[letter] = count - 1
      } else {
        states[i] = 'absent'
      }
    }

    // Update letter states
    for (let i = 0; i < states.length; i++) {
      if (states[i]) {
        currentRow.letterStates[i] = states[i]!
      }
    }

    // Lock current row and add new row
    currentRow.isLocked = true
    addNewRow()
  }
}

const nextEmoji = () => {
  selectRandomEmoji()
}

const resetGame = () => {
  showFilters.value = true
  selectedLetter.value = null
  selectedLevel1.value = 'all'
  selectedLevel2.value = 'all'
  currentEmoji.value = null
  guessRows.value = []
  showResult.value = false
  isCorrect.value = false
}
</script>

<template>
  <div class="space-y-6">
    <!-- Filters (only show when not playing) -->
    <div v-if="showFilters" class="space-y-4">
      <h3 class="text-lg font-medium text-gray-700">Choose your challenge:</h3>

      <!-- Letter selection -->
      <div class="space-y-2">
        <label class="text-sm font-medium text-gray-600">Starting Letter (optional):</label>
        <div class="flex flex-wrap gap-2 justify-center">
          <button @click="selectedLetter = null" :class="[
            'px-3 py-1 rounded-lg font-medium text-sm transition-all',
            selectedLetter === null
              ? 'bg-purple-500 text-white shadow-lg'
              : 'bg-gray-200 text-gray-700 hover:bg-purple-200'
          ]">
            Any
          </button>
          <button v-for="letter in availableLetters" :key="letter" @click="selectedLetter = letter" :class="[
            'w-10 h-10 rounded-lg font-bold text-lg transition-all',
            selectedLetter === letter
              ? 'bg-purple-500 text-white shadow-lg'
              : 'bg-gray-200 text-gray-700 hover:bg-purple-200 hover:scale-105'
          ]">
            {{ letter.toUpperCase() }}
          </button>
        </div>
      </div>

      <!-- Length range selection -->
      <div class="space-y-2">
        <label class="text-sm font-medium text-gray-600">Word Length Range:</label>
        <div class="flex items-center gap-4 justify-center">
          <div class="flex items-center gap-2">
            <span class="text-sm text-gray-600">Min:</span>
            <input v-model.number="minLength" type="number" min="1" max="20"
              class="w-16 px-2 py-1 border border-gray-300 rounded text-center" />
          </div>
          <span class="text-gray-400">-</span>
          <div class="flex items-center gap-2">
            <span class="text-sm text-gray-600">Max:</span>
            <input v-model.number="maxLength" type="number" :min="minLength" max="20"
              class="w-16 px-2 py-1 border border-gray-300 rounded text-center" />
          </div>
        </div>
      </div>

      <!-- Category selection -->
      <div class="space-y-2">
        <label class="text-sm font-medium text-gray-600">Category (optional):</label>
        <div class="flex flex-wrap gap-2">
          <!-- All button -->
          <button @click="selectLevel1('all')" :class="[
            'px-3 py-1 rounded-lg font-medium text-sm transition-colors',
            selectedLevel1 === 'all'
              ? 'bg-purple-500 text-white'
              : 'bg-gray-200 text-gray-700 hover:bg-gray-300'
          ]">
            All
          </button>

          <!-- Level 1 categories -->
          <button v-for="level1 in level1Categories" :key="level1" @click="selectLevel1(level1)" :class="[
            'px-3 py-1 rounded-lg font-medium text-sm transition-colors',
            selectedLevel1 === level1
              ? 'bg-purple-500 text-white'
              : 'bg-gray-200 text-gray-700 hover:bg-gray-300'
          ]">
            {{ level1 }}
          </button>
        </div>

        <!-- Level 2 categories (shown when a level1 is selected) -->
        <div v-if="selectedLevel1 !== 'all'" class="ml-4 space-y-2">
          <div class="text-sm font-medium text-gray-600">
            Subcategories:
          </div>
          <div class="flex flex-wrap gap-2">
            <button @click="selectLevel2('all')" :class="[
              'px-2 py-1 rounded text-xs font-medium transition-colors',
              selectedLevel2 === 'all'
                ? 'bg-purple-400 text-white'
                : 'bg-gray-100 text-gray-600 hover:bg-gray-200'
            ]">
              All
            </button>
            <button v-for="level2 in getLevel2Categories(selectedLevel1)" :key="level2" @click="selectLevel2(level2)"
              :class="[
                'px-2 py-1 rounded text-xs font-medium transition-colors',
                selectedLevel2 === level2
                  ? 'bg-purple-400 text-white'
                  : 'bg-gray-100 text-gray-600 hover:bg-gray-200'
              ]">
              {{ level2 }}
            </button>
          </div>
        </div>
      </div>

      <!-- Filter summary -->
      <div class="text-center text-sm text-gray-600">
        <span v-if="selectedLetter">
          Starting with "<strong>{{ selectedLetter.toUpperCase() }}</strong>" •
        </span>
        <span>Length: <strong>{{ minLength }}-{{ maxLength }}</strong> letters</span>
        <span v-if="selectedLevel1 !== 'all'">
          • Category: <strong>{{ selectedLevel1 }}</strong>
          <span v-if="selectedLevel2 !== 'all'"> > {{ selectedLevel2 }}</span>
        </span>
        <span v-if="filteredEmojis.length > 0">
          • <strong>{{ filteredEmojis.length }}</strong> emojis match
        </span>
      </div>

      <!-- Start game button -->
      <div class="text-center">
        <button @click="selectRandomEmoji" :disabled="filteredEmojis.length === 0"
          class="px-8 py-3 bg-purple-500 text-white rounded-lg hover:bg-purple-600 disabled:bg-gray-400 font-medium transition-colors text-lg">
          🎯 Start Game
        </button>
      </div>
    </div>

    <!-- Game area (only show when playing) -->
    <div v-if="!showFilters && currentEmoji" class="text-center">
      <div class="text-8xl mb-4">{{ currentEmoji?.emoji }}</div>
      <p class="text-gray-600">Type the emoji name in the boxes below:</p>
    </div>

    <div v-if="!showFilters && currentEmoji" class="space-y-4">
      <!-- Guess rows -->
      <div class="space-y-3">
        <div v-for="(row, rowIndex) in guessRows" :key="rowIndex" class="flex gap-2 justify-center flex-wrap px-4">
          <template v-for="(box, boxIndex) in inputBoxes" :key="boxIndex">
            <input v-if="!box.isSpace" :data-row="rowIndex" :data-box="boxIndex"
              :value="row.inputs[getInputIndexFromBoxIndex(boxIndex)] || ''"
              @keydown="handleKeydown(rowIndex, boxIndex, $event)" @input="handleInput(rowIndex, boxIndex, $event)"
              :disabled="row.isLocked || showResult" maxlength="1" :class="[
                'w-12 h-12 border-2 rounded-lg text-center text-xl font-bold focus:outline-none uppercase transition-colors',
                row.letterStates[getInputIndexFromBoxIndex(boxIndex)] === 'correct' ? 'bg-green-500 border-green-500 text-white' : '',
                row.letterStates[getInputIndexFromBoxIndex(boxIndex)] === 'present' ? 'bg-yellow-500 border-yellow-500 text-white' : '',
                row.letterStates[getInputIndexFromBoxIndex(boxIndex)] === 'absent' ? 'bg-gray-400 border-gray-400 text-white' : '',
                !row.letterStates[getInputIndexFromBoxIndex(boxIndex)] && !row.isLocked ? 'border-gray-300 focus:border-purple-500' : '',
                row.isLocked ? 'opacity-100' : ''
              ]" />
            <div v-else class="w-12 h-12 flex items-center justify-center text-gray-400 text-xl font-bold">␣</div>
          </template>
        </div>
      </div>

      <!-- Control buttons -->
      <div class="text-center flex gap-4 justify-center">
        <button @click="submitGuess" :disabled="showResult"
          class="px-6 py-3 bg-purple-500 text-white rounded-lg hover:bg-purple-600 disabled:bg-gray-400 font-medium transition-colors">
          Submit Guess
        </button>
        <button @click="giveUp" :disabled="showResult"
          class="px-6 py-3 bg-red-500 text-white rounded-lg hover:bg-red-600 disabled:bg-gray-400 font-medium transition-colors">
          Give Up
        </button>
      </div>

      <!-- Result message -->
      <div v-if="showResult" class="text-center space-y-4">
        <div :class="[
          'text-2xl font-bold',
          isCorrect ? 'text-green-500' : 'text-orange-500'
        ]">
          {{ isCorrect ? '✅ Correct!' : '💡 Answer Revealed!' }}
        </div>
        <div class="text-gray-700">
          <p v-if="!isCorrect">
            The answer was: <strong>{{ currentEmoji.shortname.slice(1, -1).replace(/_/g, ' ') }}</strong>
          </p>
        </div>
        <button @click="nextEmoji"
          class="px-6 py-3 bg-blue-500 text-white rounded-lg hover:bg-blue-600 font-medium transition-colors">
          Next Emoji →
        </button>
      </div>

      <!-- Back to filters button -->
      <div class="text-center">
        <button @click="resetGame" class="text-sm text-gray-600 hover:text-purple-600 transition-colors">
          ← Back to Filters
        </button>
      </div>

      <!-- Score -->
      <div class="text-center text-gray-600 mt-6">
        Score: {{ score }} / {{ totalGuesses }}
        <span v-if="totalGuesses > 0">
          ({{ Math.round((score / totalGuesses) * 100) }}%)
        </span>
      </div>
    </div>
  </div>
</template>
