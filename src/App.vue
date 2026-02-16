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
          <v-col cols="12" sm="10" md="10" lg="8">
            <Calculator 
              :magic-enabled="magicMode"
              :trigger-number="triggerNumber"
              :magic-target="magicTarget"
              :operation-count="operationCount"
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

            <v-btn
              variant="tonal"
              color="primary"
              class="mb-4"
              @click="setOneMinuteLater"
            >
              设为一分钟后
            </v-btn>

            <v-switch
              v-model="padZeros"
              label="日期补零"
              color="primary"
            ></v-switch>

            <v-text-field
              v-model="triggerNumber"
              label="触发数字"
              type="text"
              variant="outlined"
              density="compact"
              hint="输入此数字后激活魔术模式"
            ></v-text-field>

            <v-text-field
              v-model.number="operationCount"
              label="期望操作次数"
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
import { ref, computed } from 'vue'
import Calculator from './components/Calculator.vue'

export default {
  name: 'App',
  components: {
    Calculator
  },
  setup() {
    const settingsDialog = ref(false)
    const magicMode = ref(true)
    
    // Date/time settings
    const targetYear = ref(26)
    const targetMonth = ref(2)
    const targetDay = ref(6)
    const targetHour = ref(22)
    const targetMinute = ref(33)
    const targetSecond = ref(33)
    const padZeros = ref(false)
    const operationCount = ref(5)
    const triggerNumber = ref('888')
    
    const calculatedTarget = computed(() => {
      const year = String(targetYear.value)
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

    // Magic mode state
    const magicTarget = ref(parseInt(calculatedTarget.value))

    const setOneMinuteLater = () => {
      const future = new Date(Date.now() + 60 * 1000)
      targetYear.value = future.getFullYear() % 100
      targetMonth.value = future.getMonth() + 1
      targetDay.value = future.getDate()
      targetHour.value = future.getHours()
      targetMinute.value = future.getMinutes()
      targetSecond.value = future.getSeconds()
    }

    const applySettings = () => {
      if (magicMode.value) {
        const target = calculatedTarget.value
        magicTarget.value = parseInt(target)
      } else {
        magicTarget.value = 0
      }
      settingsDialog.value = false
    }

    const onMagicComplete = () => {
      if (magicMode.value) {
        magicTarget.value = parseInt(calculatedTarget.value)
      }
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
      triggerNumber,
      calculatedTarget,
      magicTarget,
      setOneMinuteLater,
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
