<template>
  <v-card elevation="8" class="calculator-card">
    <v-card-text>
      <!-- Display -->
      <div class="display-container mb-4">
        <div class="display">
          {{ displayValue }}
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
    magicEnabled: {
      type: Boolean,
      default: false
    },
    triggerNumber: {
      type: String,
      default: ''
    },
    magicTarget: {
      type: Number,
      default: 0
    },
    operationCount: {
      type: Number,
      default: 5
    }
  },
  emits: ['complete'],
  setup(props, { emit }) {
    const currentValue = ref('0')
    const previousValue = ref(null)
    const operation = ref(null)
    const shouldResetDisplay = ref(false)
    
    // Magic mode state
    const magicActive = ref(false)
    const magicInitialValue = ref(0)
    const magicRunningTotal = ref(0)
    const magicOperationsCompleted = ref(0)
    const magicNumbers = ref([])
    const magicCurrentIndex = ref(0)

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

    const generateMagicNumbers = () => {
      const MIN_VALUE_PERCENTAGE = 0.1
      const numbers = []
      const target = props.magicTarget - magicInitialValue.value
      let remaining = target
      const count = props.operationCount
      
      // Generate random numbers that sum to (target - initial value)
      for (let i = 0; i < count - 1; i++) {
        const maxValue = Math.floor(remaining / (count - i))
        const minValue = Math.floor(maxValue * MIN_VALUE_PERCENTAGE)
        const randomNum = Math.floor(Math.random() * (maxValue - minValue + 1)) + minValue
        numbers.push(randomNum)
        remaining -= randomNum
      }
      
      // Last number is whatever remains
      numbers.push(remaining)
      
      // Shuffle the array
      for (let i = numbers.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1));
        [numbers[i], numbers[j]] = [numbers[j], numbers[i]]
      }
      
      return numbers
    }

    const activateMagicMode = () => {
      magicActive.value = true
      magicInitialValue.value = parseFloat(currentValue.value)
      magicRunningTotal.value = magicInitialValue.value
      magicOperationsCompleted.value = 0
      magicCurrentIndex.value = 0
      magicNumbers.value = generateMagicNumbers()
      shouldResetDisplay.value = true
    }

    const checkTriggerNumber = () => {
      if (props.magicEnabled && !magicActive.value && props.triggerNumber) {
        if (currentValue.value === props.triggerNumber) {
          activateMagicMode()
          return true
        }
      }
      return false
    }

    const handleButtonClick = (button) => {
      if (button.value === 'empty' || button.value === 'empty2') {
        return
      }

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
        handleOperation(value)
        return
      }

      if (value === 'equals') {
        handleEquals()
      }
    }

    const inputNumber = (num) => {
      if (magicActive.value) {
        // In magic mode, check if we've reached target or completed operations
        if (magicRunningTotal.value >= props.magicTarget || 
            magicOperationsCompleted.value >= props.operationCount) {
          return // Don't allow more input
        }
        
        // Show the pre-generated magic number and increment index
        if (magicCurrentIndex.value < magicNumbers.value.length) {
          currentValue.value = String(magicNumbers.value[magicCurrentIndex.value])
          magicCurrentIndex.value++
          shouldResetDisplay.value = false
        }
        return
      }

      // Normal calculator input
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

      // Check if trigger number was entered
      checkTriggerNumber()
    }

    const handleOperation = (op) => {
      if (magicActive.value) {
        // In magic mode, only allow addition
        if (op !== 'add') {
          return
        }

        // Check if we've reached target or completed operations
        if (magicRunningTotal.value >= props.magicTarget || 
            magicOperationsCompleted.value >= props.operationCount) {
          return
        }

        // Skip adding if no valid current number to add (first + after trigger)
        if (shouldResetDisplay.value) {
          shouldResetDisplay.value = false
          return
        }

        // Calculate the addition
        const currentNum = parseFloat(currentValue.value)
        magicRunningTotal.value += currentNum
        
        // Ensure we don't exceed target
        if (magicRunningTotal.value > props.magicTarget) {
          magicRunningTotal.value = props.magicTarget
        }
        
        currentValue.value = String(magicRunningTotal.value)
        magicOperationsCompleted.value++
        shouldResetDisplay.value = true

        // Check if we've reached the target
        if (magicRunningTotal.value >= props.magicTarget || 
            magicOperationsCompleted.value >= props.operationCount) {
          currentValue.value = String(props.magicTarget)
          emit('complete')
          setTimeout(() => {
            resetMagicMode()
          }, 2000)
        }
        return
      }

      // Normal calculator operation
      if (previousValue.value !== null && operation.value && !shouldResetDisplay.value) {
        calculate()
      }
      previousValue.value = parseFloat(currentValue.value)
      operation.value = op
      shouldResetDisplay.value = true
    }

    const handleEquals = () => {
      if (magicActive.value) {
        // In magic mode, equals shows the final target
        currentValue.value = String(props.magicTarget)
        emit('complete')
        setTimeout(() => {
          resetMagicMode()
        }, 2000)
        return
      }

      // Normal calculator equals
      if (previousValue.value !== null && operation.value) {
        calculate()
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
      resetMagicMode()
    }

    const backspace = () => {
      if (magicActive.value) {
        return // Don't allow backspace in magic mode
      }
      
      if (currentValue.value.length > 1) {
        currentValue.value = currentValue.value.slice(0, -1)
      } else {
        currentValue.value = '0'
      }
    }

    const resetMagicMode = () => {
      magicActive.value = false
      magicInitialValue.value = 0
      magicRunningTotal.value = 0
      magicOperationsCompleted.value = 0
      magicCurrentIndex.value = 0
      magicNumbers.value = []
    }

    // Watch for magic enabled changes
    watch(() => props.magicEnabled, (newVal) => {
      if (!newVal) {
        resetMagicMode()
      }
    })

    return {
      displayValue,
      buttons,
      handleButtonClick
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
