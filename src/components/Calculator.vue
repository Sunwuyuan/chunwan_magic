<template>
  <v-card elevation="8" class="calculator-card">
    <v-card-text>
      <!-- Display -->
      <div class="display-container mb-4">
        <div class="display">
          {{ displayValue }}
        </div>
        <div v-if="magicMode" class="magic-indicator">
          <v-chip color="purple" size="small" class="mt-2">
            魔术模式 ({{ magicCounter }}/{{ magicNumbers.length }})
          </v-chip>
        </div>
      </div>

      <!-- Calculator Buttons -->
      <v-row dense>
        <v-col cols="3" v-for="button in buttons" :key="button.value">
          <v-btn
            :color="button.color || 'grey-lighten-1'"
            :variant="button.variant || 'elevated'"
            size="x-large"
            block
            @click="handleButtonClick(button)"
            class="calculator-button"
          >
            {{ button.label }}
          </v-btn>
        </v-col>
      </v-row>
    </v-card-text>
  </v-card>
</template>

<script>
import { ref, computed, watch } from 'vue'

export default {
  name: 'Calculator',
  props: {
    magicMode: {
      type: Boolean,
      default: false
    },
    magicNumbers: {
      type: Array,
      default: () => []
    },
    magicTarget: {
      type: Number,
      default: 0
    }
  },
  emits: ['complete'],
  setup(props, { emit }) {
    const currentValue = ref('0')
    const previousValue = ref(null)
    const operation = ref(null)
    const shouldResetDisplay = ref(false)
    const magicCounter = ref(0)
    const magicOperationCount = ref(0)

    const displayValue = computed(() => {
      return currentValue.value
    })

    const buttons = [
      { label: 'C', value: 'clear', color: 'red-lighten-1' },
      { label: '÷', value: 'divide', color: 'orange' },
      { label: '×', value: 'multiply', color: 'orange' },
      { label: '⌫', value: 'backspace', color: 'grey' },
      
      { label: '7', value: '7' },
      { label: '8', value: '8' },
      { label: '9', value: '9' },
      { label: '-', value: 'subtract', color: 'orange' },
      
      { label: '4', value: '4' },
      { label: '5', value: '5' },
      { label: '6', value: '6' },
      { label: '+', value: 'add', color: 'orange' },
      
      { label: '1', value: '1' },
      { label: '2', value: '2' },
      { label: '3', value: '3' },
      { label: '=', value: 'equals', color: 'green', variant: 'elevated' },
      
      { label: '0', value: '0' },
      { label: '.', value: '.' },
      { label: '', value: 'empty' },
      { label: '', value: 'empty2' }
    ]

    const handleButtonClick = (button) => {
      if (button.value === 'empty' || button.value === 'empty2') {
        return
      }

      // In magic mode, intercept number inputs
      if (props.magicMode && magicCounter.value < props.magicNumbers.length) {
        handleMagicMode(button)
      } else {
        handleNormalMode(button)
      }
    }

    const handleMagicMode = (button) => {
      const value = button.value

      // Allow clear and backspace in magic mode
      if (value === 'clear') {
        clear()
        return
      }

      if (value === 'backspace') {
        backspace()
        return
      }

      // For number inputs, use the pre-generated magic number
      if (!isNaN(value) || value === '.') {
        // Always use the pre-generated magic number in magic mode
        currentValue.value = String(props.magicNumbers[magicCounter.value])
        shouldResetDisplay.value = false
        magicCounter.value++
        return
      }

      // For operations
      if (value === 'add' || value === 'subtract' || value === 'multiply' || value === 'divide') {
        if (previousValue.value !== null && operation.value && !shouldResetDisplay.value) {
          calculate()
        }
        previousValue.value = parseFloat(currentValue.value)
        operation.value = value
        shouldResetDisplay.value = true
        magicOperationCount.value++
        return
      }

      // For equals
      if (value === 'equals') {
        if (previousValue.value !== null && operation.value) {
          calculate()
          // Check if we've completed all magic operations
          if (magicCounter.value >= props.magicNumbers.length) {
            // Show the target number
            currentValue.value = String(props.magicTarget)
            emit('complete')
            // Reset state
            setTimeout(() => {
              resetMagicMode()
            }, 2000)
          }
        }
      }
    }

    const handleNormalMode = (button) => {
      const value = button.value

      if (value === 'clear') {
        clear()
        return
      }

      if (value === 'backspace') {
        backspace()
        return
      }

      if (!isNaN(value) || value === '.') {
        inputNumber(value)
        return
      }

      if (value === 'add' || value === 'subtract' || value === 'multiply' || value === 'divide') {
        if (previousValue.value !== null && operation.value && !shouldResetDisplay.value) {
          calculate()
        }
        previousValue.value = parseFloat(currentValue.value)
        operation.value = value
        shouldResetDisplay.value = true
        return
      }

      if (value === 'equals') {
        if (previousValue.value !== null && operation.value) {
          calculate()
        }
      }
    }

    const inputNumber = (num) => {
      if (shouldResetDisplay.value) {
        currentValue.value = num === '.' ? '0.' : num
        shouldResetDisplay.value = false
      } else {
        if (num === '.' && currentValue.value.includes('.')) {
          return
        }
        currentValue.value = currentValue.value === '0' && num !== '.' 
          ? num 
          : currentValue.value + num
      }
    }

    const calculate = () => {
      const prev = previousValue.value
      const current = parseFloat(currentValue.value)
      let result = 0

      switch (operation.value) {
        case 'add':
          result = prev + current
          break
        case 'subtract':
          result = prev - current
          break
        case 'multiply':
          result = prev * current
          break
        case 'divide':
          result = current !== 0 ? prev / current : 0
          break
      }

      currentValue.value = String(result)
      previousValue.value = null
      operation.value = null
      shouldResetDisplay.value = true
    }

    const clear = () => {
      currentValue.value = '0'
      previousValue.value = null
      operation.value = null
      shouldResetDisplay.value = false
      if (props.magicMode) {
        resetMagicMode()
      }
    }

    const backspace = () => {
      if (currentValue.value.length > 1) {
        currentValue.value = currentValue.value.slice(0, -1)
      } else {
        currentValue.value = '0'
      }
    }

    const resetMagicMode = () => {
      magicCounter.value = 0
      magicOperationCount.value = 0
    }

    // Watch for magic mode changes
    watch(() => props.magicMode, (newVal) => {
      if (newVal) {
        clear()
      } else {
        resetMagicMode()
      }
    })

    watch(() => props.magicNumbers, () => {
      resetMagicMode()
    })

    return {
      displayValue,
      buttons,
      handleButtonClick,
      magicCounter
    }
  }
}
</script>

<style scoped>
.calculator-card {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  padding: 16px;
}

.display-container {
  background-color: #1a1a1a;
  border-radius: 8px;
  padding: 16px;
  min-height: 80px;
  display: flex;
  flex-direction: column;
  justify-content: center;
}

.display {
  color: #00ff00;
  font-size: 2.5rem;
  font-family: 'Courier New', monospace;
  text-align: right;
  word-break: break-all;
  font-weight: bold;
}

.magic-indicator {
  text-align: right;
}

.calculator-button {
  height: 70px;
  font-size: 1.5rem;
  font-weight: bold;
}

@media (max-width: 600px) {
  .display {
    font-size: 1.8rem;
  }
  
  .calculator-button {
    height: 60px;
    font-size: 1.2rem;
  }
}
</style>
