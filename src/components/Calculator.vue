<template>
  <v-card class="mx-auto rounded-xl" elevation="0" border max-width="1200">
    <v-card-text class="pa-0">
      <v-row no-gutters>
        <!-- Calculator Area -->
        <v-col cols="12" md="7" lg="8" class="pa-6">
          <!-- Display -->
          <v-sheet
            class="d-flex align-end justify-end mb-6 px-4 py-2 rounded-lg bg-surface-variant"
            height="120"
            elevation="0"
          >
            <div class="text-h3 font-weight-medium text-truncate w-100 text-right">
              {{ displayValue }}
            </div>
          </v-sheet>

          <!-- Keypad -->
          <v-row dense>
            <v-col cols="3" v-for="button in buttons" :key="button.value">
              <v-btn
                :color="button.color"
                :variant="button.variant || 'text'"
                height="72"
                block
                class="text-h5 font-weight-regular rounded-lg"
                @click="handleButtonClick(button)"
              >
                {{ button.label }}
              </v-btn>
            </v-col>
          </v-row>
        </v-col>

        <!-- Divider for mobile/desktop -->
        <v-divider vertical class="hidden-sm-and-down"></v-divider>
        <v-divider class="hidden-md-and-up"></v-divider>

        <!-- History Area -->
        <v-col cols="12" md="5" lg="4" class="bg-grey-lighten-5">
          <div class="d-flex flex-column h-100 pa-4">
            <div class="text-overline text-medium-emphasis mb-2">History</div>
            
            <v-sheet 
              class="flex-grow-1 bg-transparent overflow-y-auto" 
              style="max-height: 400px; min-height: 200px;"
            >
              <div v-if="historyItems.length === 0" class="text-body-2 text-medium-emphasis pa-2">
                No calculations yet
              </div>
              
              <v-list v-else bg-color="transparent" density="compact" class="pa-0">
                <v-list-item
                  v-for="(item, index) in historyItems"
                  :key="`${index}-${item}`"
                  class="px-2 mb-1 rounded"
                >
                  <template v-slot:title>
                    <div class="text-right text-body-1 font-weight-regular">{{ item }}</div>
                  </template>
                </v-list-item>
              </v-list>
            </v-sheet>
            
            <div class="mt-auto pt-2 text-center" v-if="historyItems.length > 0">
              <v-btn
                variant="text"
                size="small"
                color="medium-emphasis"
                @click="clearHistoryOnly"
              >
                Clear History
              </v-btn>
            </div>
          </div>
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
    const pendingValue = ref(null)
    const pendingOperation = ref(null)
    const shouldResetDisplay = ref(false)
    const historyItems = ref([])
    const calculationFinalized = ref(false)
    
    // Magic mode state
    const magicActive = ref(false)
    const magicNumbers = ref([])
    const magicRandomIndex = ref(0)
    const magicPlannedInput = ref('')
    const magicCurrentDigitIndex = ref(0)
    const magicInputBlocked = ref(false)

    const displayValue = computed(() => {
      return currentValue.value === '' ? '' : currentValue.value
    })

    const operatorLabelMap = {
      add: '+',
      subtract: '−',
      multiply: '×',
      divide: '÷'
    }

    // Modern color palette
    // Numbers: Default text color (usually black/dark grey), variant 'text'
    // Actions (C, BS): Error color, variant 'text'
    // Operators: Primary color, variant 'tonal'
    // Equals: Primary color, variant 'flat' (filled)
    
    const buttons = [
      { label: 'C', value: 'clear', color: 'error', variant: 'text' },
      { label: '÷', value: 'divide', color: 'primary', variant: 'tonal' },
      { label: '×', value: 'multiply', color: 'primary', variant: 'tonal' },
      { label: '⌫', value: 'backspace', color: 'medium-emphasis', variant: 'text' },
      
      { label: '7', value: '7' },
      { label: '8', value: '8' },
      { label: '9', value: '9' },
      { label: '−', value: 'subtract', color: 'primary', variant: 'tonal' },
      
      { label: '4', value: '4' },
      { label: '5', value: '5' },
      { label: '6', value: '6' },
      { label: '+', value: 'add', color: 'primary', variant: 'tonal' },
      
      { label: '1', value: '1' },
      { label: '2', value: '2' },
      { label: '3', value: '3' },
      { label: '=', value: 'equals', color: 'primary', variant: 'flat' },
      
      { label: '0', value: '0' },
      { label: '.', value: '.' },
      { label: '', value: 'empty', disabled: true },
      { label: '', value: 'empty2', disabled: true }
    ]

    const getMagicBaseValue = () => {
      if (pendingValue.value !== null && (currentValue.value === '' || shouldResetDisplay.value)) {
        return pendingValue.value
      }

      if (currentValue.value === '' || currentValue.value === 'error') {
        return pendingValue.value ?? 0
      }

      const parsed = parseFloat(currentValue.value)
      return Number.isNaN(parsed) ? 0 : parsed
    }

    const generateMagicNumbers = (total) => {
      const safeTotal = Math.max(0, Math.floor(Number(total) || 0))
      const count = Math.max(1, Math.floor(Number(props.operationCount) || 1))
      const numbers = []
      let remaining = safeTotal

      for (let i = 0; i < count - 1; i++) {
        const randomNum = remaining > 0 ? Math.floor(Math.random() * (remaining + 1)) : 0
        numbers.push(randomNum)
        remaining -= randomNum
      }

      numbers.push(Math.max(0, remaining))

      for (let i = numbers.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1))
        ;[numbers[i], numbers[j]] = [numbers[j], numbers[i]]
      }

      return numbers
    }

    const checkActivateByFunctionKey = () => {
      if (!props.magicEnabled || magicActive.value || !props.triggerNumber) {
        return false
      }
      return currentValue.value === props.triggerNumber
    }

    const refreshMagicPlannedInput = () => {
      if (!magicActive.value) {
        return
      }

      const target = Math.max(0, Number(props.magicTarget) || 0)
      const base = getMagicBaseValue()

      if (base > target) {
        currentValue.value = 'error'
        magicNumbers.value = []
        magicRandomIndex.value = 0
        magicPlannedInput.value = ''
        magicCurrentDigitIndex.value = 0
        magicInputBlocked.value = true
        return
      }

      const nextValue = magicRandomIndex.value < magicNumbers.value.length
        ? magicNumbers.value[magicRandomIndex.value]
        : (target - base)

      magicPlannedInput.value = String(Math.max(0, Math.floor(Number(nextValue) || 0)))
      magicCurrentDigitIndex.value = 0
      magicInputBlocked.value = false
      currentValue.value = ''
      shouldResetDisplay.value = false
    }

    const activateMagicMode = () => {
      magicActive.value = true
      const target = Math.max(0, Number(props.magicTarget) || 0)
      const base = getMagicBaseValue()

      if (base > target) {
        currentValue.value = 'error'
        magicNumbers.value = []
        magicRandomIndex.value = 0
        magicPlannedInput.value = ''
        magicCurrentDigitIndex.value = 0
        magicInputBlocked.value = true
        return
      }

      magicNumbers.value = generateMagicNumbers(target - base)
      magicRandomIndex.value = 0
      magicCurrentDigitIndex.value = 0
      magicPlannedInput.value = ''
      magicInputBlocked.value = false
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

    const calculateBinary = (left, right, op) => {
      switch (op) {
        case 'add':
          return left + right
        case 'subtract':
          return left - right
        case 'multiply':
          return left * right
        case 'divide':
          return right !== 0 ? left / right : 0
        default:
          return right
      }
    }

    const replaceLastHistoryOperator = (op) => {
      if (historyItems.value.length === 0) {
        return
      }
      const lastIndex = historyItems.value.length - 1
      const operatorRegex = /\s[+\-×÷]$/
      if (operatorRegex.test(historyItems.value[lastIndex])) {
        historyItems.value[lastIndex] = historyItems.value[lastIndex].replace(operatorRegex, ` ${operatorLabelMap[op]}`)
      }
    }

    const clearHistoryForNextSession = () => {
      if (!calculationFinalized.value) {
        return
      }
      historyItems.value = []
      calculationFinalized.value = false
    }

    const clearHistoryOnly = () => {
      historyItems.value = []
    }

    const onMagicInputCommitted = () => {
      if (!magicActive.value) {
        return
      }
      if (magicPlannedInput.value !== '' && magicRandomIndex.value < magicNumbers.value.length) {
        magicRandomIndex.value++
      }
      magicCurrentDigitIndex.value = 0
      magicPlannedInput.value = ''
      magicInputBlocked.value = false
    }

    const inputNumber = (num) => {
      clearHistoryForNextSession()

      if (magicActive.value) {
        if (magicInputBlocked.value) {
          return
        }

        if (!magicPlannedInput.value) {
          refreshMagicPlannedInput()
          if (magicInputBlocked.value || !magicPlannedInput.value) {
            return
          }
        }

        if (magicCurrentDigitIndex.value < magicPlannedInput.value.length) {
          const digit = magicPlannedInput.value[magicCurrentDigitIndex.value]
          
          if (currentValue.value === '0' || currentValue.value === '') {
            currentValue.value = digit
          } else {
            currentValue.value += digit
          }
          
          shouldResetDisplay.value = false
          magicCurrentDigitIndex.value++

          if (magicCurrentDigitIndex.value >= magicPlannedInput.value.length) {
            magicInputBlocked.value = true
          }
        }
        
        return
      }

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

    const handleOperation = (op) => {
      clearHistoryForNextSession()
      const activateAfterOperation = checkActivateByFunctionKey()

      if (pendingValue.value !== null && (currentValue.value === '' || shouldResetDisplay.value)) {
        pendingOperation.value = op
        replaceLastHistoryOperator(op)
        if (magicActive.value) {
          refreshMagicPlannedInput()
        }
        if (activateAfterOperation) {
          activateMagicMode()
        }
        return
      }

      const enteredValue = currentValue.value === '' ? '0' : currentValue.value
      const currentNum = parseFloat(enteredValue)

      if (pendingValue.value === null) {
        pendingValue.value = currentNum
      } else if (pendingOperation.value !== null) {
        pendingValue.value = calculateBinary(pendingValue.value, currentNum, pendingOperation.value)
      }

      historyItems.value.push(`${enteredValue} ${operatorLabelMap[op]}`)
      onMagicInputCommitted()

      pendingOperation.value = op
      currentValue.value = ''
      shouldResetDisplay.value = true

      if (magicActive.value) {
        refreshMagicPlannedInput()
      }

      if (activateAfterOperation) {
        activateMagicMode()
      }
    }

    const handleEquals = () => {
      const activateAfterEquals = checkActivateByFunctionKey()

      if (pendingValue.value === null || pendingOperation.value === null) {
        if (activateAfterEquals) {
          activateMagicMode()
        }
        return
      }

      if (currentValue.value === '' || shouldResetDisplay.value) {
        if (activateAfterEquals) {
          activateMagicMode()
        }
        return
      }

      const enteredValue = currentValue.value
      const currentNum = parseFloat(enteredValue)
      const result = calculateBinary(pendingValue.value, currentNum, pendingOperation.value)

      currentValue.value = String(result)
      historyItems.value.push(`${enteredValue} = ${result}`)
      onMagicInputCommitted()
      pendingValue.value = null
      pendingOperation.value = null
      shouldResetDisplay.value = true
      calculationFinalized.value = true

      if (magicActive.value && result === (Number(props.magicTarget) || 0)) {
        emit('complete')
      }

      if (activateAfterEquals) {
        activateMagicMode()
      }
    }

    const clear = () => {
      currentValue.value = '0'
      pendingValue.value = null
      pendingOperation.value = null
      shouldResetDisplay.value = false
      historyItems.value = []
      calculationFinalized.value = false
      resetMagicMode()
    }

    const backspace = () => {
      clearHistoryForNextSession()

      if (currentValue.value === 'error') {
        currentValue.value = '0'
        return
      }

      if (currentValue.value.length > 1) {
        currentValue.value = currentValue.value.slice(0, -1)
      } else {
        currentValue.value = '0'
      }
    }

    const resetMagicMode = () => {
      magicActive.value = false
      magicNumbers.value = []
      magicRandomIndex.value = 0
      magicPlannedInput.value = ''
      magicCurrentDigitIndex.value = 0
      magicInputBlocked.value = false
    }

    // Watch for magic enabled changes
    watch(() => props.magicEnabled, (newVal) => {
      if (!newVal) {
        resetMagicMode()
      }
    })

    return {
      displayValue,
      historyItems,
      buttons,
      handleButtonClick,
      clearHistoryOnly
    }
  }
}
</script>

<style scoped>
/* Optional: slightly denser button text if needed, but text-h5 is usually good */
</style>
