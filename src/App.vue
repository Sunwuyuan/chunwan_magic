<template>
  <v-app>
    <v-app-bar color="primary" dark>
      <v-app-bar-title>魔术计算器</v-app-bar-title>
      <v-spacer></v-spacer>
      <v-btn icon @click="settingsDialog = true">
        <v-icon>mdi-cog</v-icon>
      </v-btn>
    </v-app-bar>

    <v-main>
      <v-container class="fill-height" fluid>
        <v-row justify="center" align="center">
          <v-col cols="12" sm="8" md="6" lg="4">
            <Calculator 
              :magic-mode="magicMode"
              :magic-numbers="magicNumbers"
              :magic-target="magicTarget"
              @complete="onMagicComplete"
            />
          </v-col>
        </v-row>
      </v-container>
    </v-main>

    <!-- Settings Dialog -->
    <v-dialog v-model="settingsDialog" max-width="500">
      <v-card>
        <v-card-title>
          <span class="text-h5">魔术设置</span>
        </v-card-title>
        
        <v-card-text>
          <v-switch
            v-model="magicMode"
            label="启用魔术模式"
            color="primary"
          ></v-switch>

          <v-divider class="my-4"></v-divider>

          <div v-if="magicMode">
            <v-text-field
              v-model="targetYear"
              label="年"
              type="number"
              variant="outlined"
              density="compact"
            ></v-text-field>

            <v-text-field
              v-model="targetMonth"
              label="月"
              type="number"
              min="1"
              max="12"
              variant="outlined"
              density="compact"
            ></v-text-field>

            <v-text-field
              v-model="targetDay"
              label="日"
              type="number"
              min="1"
              max="31"
              variant="outlined"
              density="compact"
            ></v-text-field>

            <v-text-field
              v-model="targetHour"
              label="时"
              type="number"
              min="0"
              max="23"
              variant="outlined"
              density="compact"
            ></v-text-field>

            <v-text-field
              v-model="targetMinute"
              label="分"
              type="number"
              min="0"
              max="59"
              variant="outlined"
              density="compact"
            ></v-text-field>

            <v-text-field
              v-model="targetSecond"
              label="秒"
              type="number"
              min="0"
              max="59"
              variant="outlined"
              density="compact"
            ></v-text-field>

            <v-switch
              v-model="padZeros"
              label="日期补零"
              color="primary"
            ></v-switch>

            <v-text-field
              v-model.number="operationCount"
              label="操作次数"
              type="number"
              min="2"
              max="10"
              variant="outlined"
              density="compact"
              hint="计算器将通过这么多次加法达到目标数字"
            ></v-text-field>

            <v-text-field
              v-model="calculatedTarget"
              label="目标数字"
              readonly
              variant="outlined"
              density="compact"
              class="mt-4"
            ></v-text-field>
          </div>
        </v-card-text>

        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn text @click="settingsDialog = false">取消</v-btn>
          <v-btn color="primary" @click="applySettings">确定</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </v-app>
</template>

<script>
import { ref, computed, watch } from 'vue'
import Calculator from './components/Calculator.vue'

export default {
  name: 'App',
  components: {
    Calculator
  },
  setup() {
    const settingsDialog = ref(false)
    const magicMode = ref(false)
    
    // Date/time settings
    const targetYear = ref(2026)
    const targetMonth = ref(2)
    const targetDay = ref(6)
    const targetHour = ref(22)
    const targetMinute = ref(33)
    const targetSecond = ref(33)
    const padZeros = ref(true)
    const operationCount = ref(5)
    
    // Magic mode state
    const magicNumbers = ref([])
    const magicTarget = ref(0)

    const calculatedTarget = computed(() => {
      const year = targetYear.value
      let month = targetMonth.value
      let day = targetDay.value
      let hour = targetHour.value
      let minute = targetMinute.value
      let second = targetSecond.value

      if (padZeros.value) {
        month = String(month).padStart(2, '0')
        day = String(day).padStart(2, '0')
        hour = String(hour).padStart(2, '0')
        minute = String(minute).padStart(2, '0')
        second = String(second).padStart(2, '0')
      }

      return `${year}${month}${day}${hour}${minute}${second}`
    })

    const generateMagicNumbers = (target, count) => {
      const MIN_VALUE_PERCENTAGE = 0.1 // Each number should be at least 10% of the average
      const numbers = []
      let remaining = parseInt(target)
      
      // Generate random numbers that sum to target
      for (let i = 0; i < count - 1; i++) {
        // Generate a random portion of the remaining value
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

    const applySettings = () => {
      if (magicMode.value) {
        const target = calculatedTarget.value
        magicTarget.value = parseInt(target)
        magicNumbers.value = generateMagicNumbers(target, operationCount.value)
      } else {
        magicNumbers.value = []
        magicTarget.value = 0
      }
      settingsDialog.value = false
    }

    const onMagicComplete = () => {
      // Reset magic mode after showing result
      setTimeout(() => {
        magicMode.value = false
        magicNumbers.value = []
        magicTarget.value = 0
      }, 3000)
    }

    return {
      settingsDialog,
      magicMode,
      targetYear,
      targetMonth,
      targetDay,
      targetHour,
      targetMinute,
      targetSecond,
      padZeros,
      operationCount,
      calculatedTarget,
      magicNumbers,
      magicTarget,
      applySettings,
      onMagicComplete
    }
  }
}
</script>

<style scoped>
.fill-height {
  min-height: calc(100vh - 64px);
}
</style>
