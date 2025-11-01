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

const selectedLetter = ref<string | null>(null)
const currentEmoji = ref<Emoji | null>(null)
const guessRows = ref<GuessRow[]>([])
const currentRowIndex = ref(0)
const showResult = ref(false)
const isCorrect = ref(false)
const score = ref(0)
const totalGuesses = ref(0)

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

// Filter emojis by selected letter
const filteredEmojis = computed(() => {
  if (!selectedLetter.value) return []
  return props.emojis.filter(emoji => {
    return emoji.name.charAt(0).toLowerCase() === selectedLetter.value
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

// Select a letter
const selectLetter = (letter: string) => {
  selectedLetter.value = letter
  selectRandomEmoji()
}

// Select random emoji from filtered list
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
}

const handleKeydown = (rowIndex: number, boxIndex: number, event: KeyboardEvent) => {
  const row = guessRows.value[rowIndex]
  if (!row || row.isLocked) return

  const target = event.target as HTMLInputElement

  if (event.key === 'Backspace') {
    event.preventDefault()
    if (row.inputs[boxIndex]) {
      row.inputs[boxIndex] = ''
    } else if (boxIndex > 0) {
      row.inputs[boxIndex - 1] = ''
      const prevInput = document.querySelector(`input[data-row="${rowIndex}"][data-box="${boxIndex - 1}"]`) as HTMLInputElement
      if (prevInput) prevInput.focus()
    }
  } else if (event.key === 'ArrowLeft') {
    event.preventDefault()
    if (boxIndex > 0) {
      const prevInput = document.querySelector(`input[data-row="${rowIndex}"][data-box="${boxIndex - 1}"]`) as HTMLInputElement
      if (prevInput) prevInput.focus()
    }
  } else if (event.key === 'ArrowRight') {
    event.preventDefault()
    if (boxIndex < row.inputs.length - 1) {
      const nextInput = document.querySelector(`input[data-row="${rowIndex}"][data-box="${boxIndex + 1}"]`) as HTMLInputElement
      if (nextInput) nextInput.focus()
    }
  } else if (event.key.length === 1 && /[a-zA-Z]/.test(event.key)) {
    event.preventDefault()
    const letter = event.key.toLowerCase()
    row.inputs[boxIndex] = letter

    if (boxIndex < row.inputs.length - 1) {
      const nextInput = document.querySelector(`input[data-row="${rowIndex}"][data-box="${boxIndex + 1}"]`) as HTMLInputElement
      if (nextInput) nextInput.focus()
    }
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
    // Only set the current row to show the answer
    // Do not modify the letterStates of previous rows to preserve their colors
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

  // Check if correct
  isCorrect.value = guess === expectedName

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
    const expected = expectedName.split('')
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
</script>

<template>
  <div class="space-y-6">
    <!-- Letter selection -->
    <div class="space-y-4">
      <h3 class="text-lg font-medium text-gray-700">Choose a letter to start:</h3>
      <div class="flex flex-wrap gap-2 justify-center">
        <button
          v-for="letter in availableLetters"
          :key="letter"
          @click="selectLetter(letter)"
          :class="[
            'w-12 h-12 rounded-lg font-bold text-xl transition-all',
            selectedLetter === letter
              ? 'bg-purple-500 text-white shadow-lg'
              : 'bg-gray-200 text-gray-700 hover:bg-purple-200 hover:scale-105'
          ]"
        >
          {{ letter.toUpperCase() }}
        </button>
      </div>
      <div v-if="selectedLetter" class="text-center text-sm text-gray-600">
        Showing emojis starting with "<strong>{{ selectedLetter.toUpperCase() }}</strong>" ({{ filteredEmojis.length }} emojis)
      </div>
    </div>

    <div v-if="currentEmoji" class="text-center">
      <div class="text-8xl mb-4">{{ currentEmoji?.emoji }}</div>
      <p class="text-gray-600">Type the emoji name in the boxes below:</p>
    </div>

    <div v-if="currentEmoji" class="space-y-4">
      <!-- Guess rows -->
      <div class="space-y-3">
        <div
          v-for="(row, rowIndex) in guessRows"
          :key="rowIndex"
          class="flex gap-2 justify-center flex-wrap px-4"
        >
          <template v-for="(box, boxIndex) in inputBoxes" :key="boxIndex">
            <input
              v-if="!box.isSpace"
              :data-row="rowIndex"
              :data-box="boxIndex"
              :value="row.inputs[boxIndex] || ''"
              @keydown="handleKeydown(rowIndex, boxIndex, $event)"
              :disabled="row.isLocked || showResult"
              maxlength="1"
              :class="[
                'w-12 h-12 border-2 rounded-lg text-center text-xl font-bold focus:outline-none uppercase transition-colors',
                row.letterStates[boxIndex] === 'correct' ? 'bg-green-500 border-green-500 text-white' : '',
                row.letterStates[boxIndex] === 'present' ? 'bg-yellow-500 border-yellow-500 text-white' : '',
                row.letterStates[boxIndex] === 'absent' ? 'bg-gray-400 border-gray-400 text-white' : '',
                !row.letterStates[boxIndex] && !row.isLocked ? 'border-gray-300 focus:border-purple-500' : '',
                row.isLocked ? 'opacity-100' : ''
              ]"
            />
            <div v-else class="w-4"></div>
          </template>
        </div>
      </div>

      <!-- Control buttons -->
      <div class="text-center flex gap-4 justify-center">
        <button
          @click="submitGuess"
          :disabled="showResult"
          class="px-6 py-3 bg-purple-500 text-white rounded-lg hover:bg-purple-600 disabled:bg-gray-400 font-medium transition-colors"
        >
          Submit Guess
        </button>
        <button
          @click="giveUp"
          :disabled="showResult"
          class="px-6 py-3 bg-red-500 text-white rounded-lg hover:bg-red-600 disabled:bg-gray-400 font-medium transition-colors"
        >
          Give Up
        </button>
      </div>

      <!-- Result message -->
      <div v-if="showResult" class="text-center space-y-4">
        <div
          :class="[
            'text-2xl font-bold',
            isCorrect ? 'text-green-500' : 'text-orange-500'
          ]"
        >
          {{ isCorrect ? '✅ Correct!' : '💡 Answer Revealed!' }}
        </div>
        <div class="text-gray-700">
          <p v-if="!isCorrect">
            The answer was: <strong>{{ currentEmoji.shortname.slice(1, -1).replace(/_/g, ' ') }}</strong>
          </p>
        </div>
        <button
          @click="nextEmoji"
          class="px-6 py-3 bg-blue-500 text-white rounded-lg hover:bg-blue-600 font-medium transition-colors"
        >
          Next Emoji →
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
