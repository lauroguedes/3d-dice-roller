<script setup lang="ts">
import { ref, onMounted, watch, computed, onBeforeMount } from 'vue';
import Dice from './components/Dice.vue';

const isDarkMode = ref(false);

const toggleTheme = () => {
  isDarkMode.value = !isDarkMode.value;
  document.body.classList.toggle('dark-theme', isDarkMode.value);
};

const diceCount = ref(1);
const diceValues = ref<number[]>([1]);
const diceRefs = ref<any[]>([]);
const history = ref<{ values: number[], total: number, timestamp: string }[]>([]);
const isRolling = ref(false);
const maxDiceCount = 3;

const totalValue = computed(() => {
  return diceValues.value.reduce((sum, val) => sum + val, 0);
});

// Watch for changes in dice count to update the dice values array
watch(diceCount, (newCount, oldCount) => {
  // Prevent changes while rolling
  if (isRolling.value) return;
  
  // Update diceValues array based on the new count
  if (newCount > oldCount) {
    // Add new dice with random values
    const newDice = Array(newCount - oldCount).fill(0).map(() => 
      Math.floor(Math.random() * 6) + 1
    );
    diceValues.value = [...diceValues.value, ...newDice];
  } else if (newCount < oldCount) {
    // Remove excess dice
    diceValues.value = diceValues.value.slice(0, newCount);
  }
});

const showRollCompleteAlert = ref(false);

const rollDice = () => {
  // Prevent rolling while animation is in progress
  if (isRolling.value) return;
  
  isRolling.value = true;
  showRollCompleteAlert.value = false;
  
  // Generate new random values for all dice
  const newValues = Array(diceCount.value).fill(0).map(() => 
    Math.floor(Math.random() * 6) + 1
  );
  
  // Set the values for the dice components
  diceValues.value = newValues;
  
  // Count completed animations
  let completedAnimations = 0;
  
  // Start the dice rolling animations and pass the callback
  diceRefs.value.forEach((dice, index) => {
    if (dice && index < newValues.length) {
      dice.rollDice(newValues[index], () => {
        completedAnimations++;
        
        // When all dice have finished rolling, update history
        if (completedAnimations === Math.min(diceRefs.value.length, newValues.length)) {
          const total = newValues.reduce((a, b) => a + b, 0);
          
          // Add to history
          history.value.unshift({
            values: [...newValues],
            total,
            timestamp: new Date().toLocaleTimeString()
          });
          
          isRolling.value = false;
          
          // Show roll complete alert briefly
          showRollCompleteAlert.value = true;
          setTimeout(() => {
            showRollCompleteAlert.value = false;
          }, 2000);
        }
      });
    }
  });
};

const incrementDiceCount = () => {
  if (diceCount.value < maxDiceCount && !isRolling.value) {
    diceCount.value++;
  }
};

const decrementDiceCount = () => {
  if (diceCount.value > 1 && !isRolling.value) {
    diceCount.value--;
  }
};

const clearHistory = () => {
  history.value = [];
};

onBeforeMount(() => {
  // Set the theme based on system preference
  if (window.matchMedia && window.matchMedia('(prefers-color-scheme: dark)').matches) {
    isDarkMode.value = true;
    document.body.classList.add('dark-theme');
  }
});

onMounted(() => {
  // Initial roll without animation
  const initialValues = Array(diceCount.value).fill(0).map(() => 
    Math.floor(Math.random() * 6) + 1
  );
  diceValues.value = initialValues;
  
  // Add to history
  history.value.unshift({
    values: [...initialValues],
    total: initialValues.reduce((a, b) => a + b, 0),
    timestamp: new Date().toLocaleTimeString()
  });
});
</script>

<template>
  <div class="container">
    <!-- Header -->
    <header>
      <h1>3D Dice Roller</h1>
      <button class="theme-toggle" @click="toggleTheme">
        <span v-if="isDarkMode">☀️</span>
        <span v-else>🌙</span>
      </button>
    </header>

    <!-- Main content -->
    <main>
      <div class="layout-grid">
        <!-- Dice Card -->
        <div class="card">
          <!-- Dice Container -->
          <div class="dice-container">
            <Dice 
              v-for="(value, index) in diceValues" 
              :key="index"
              :value="value"
              ref="diceRefs"
            />
          </div>

          <!-- Total Result -->
          <div style="display: flex; justify-content: center;">
            <transition name="fade">
              <div 
                v-if="!isRolling" 
                class="total-result"
                :style="{ backgroundColor: showRollCompleteAlert ? 'var(--secondary-color)' : 'var(--primary-color)' }"
              >
                <span v-if="showRollCompleteAlert" style="margin-right: 8px;">✓</span>
                Total: {{ totalValue }}
              </div>
            </transition>
          </div>

          <div class="divider"></div>
          
          <!-- Controls -->
          <div class="controls">
            <!-- Dice Count Controls -->
            <div class="dice-count">
              <button 
                class="counter-btn" 
                @click="decrementDiceCount"
                :disabled="isRolling || diceCount <= 1"
              >-</button>
              <span class="counter-label">{{ diceCount }} {{ diceCount === 1 ? 'die' : 'dice' }}</span>
              <button 
                class="counter-btn" 
                @click="incrementDiceCount"
                :disabled="isRolling || diceCount >= maxDiceCount"
              >+</button>
            </div>

            <!-- Roll Button -->
            <button 
              class="roll-btn" 
              @click="rollDice"
              :disabled="isRolling"
            >
              <span v-if="isRolling" class="loading"></span>
              {{ isRolling ? 'Rolling...' : 'Roll Dice' }}
            </button>
          </div>
        </div>

        <!-- History Section -->
        <div class="history-section card">
          <div class="history-header">
            <h2>Roll History</h2>
            <button 
              class="clear-btn" 
              @click="clearHistory"
              :disabled="history.length === 0"
            >
              Clear
            </button>
          </div>

          <div v-if="history.length > 0" class="history-list">
            <div v-for="(roll, index) in history" :key="index" class="history-item">
              <span class="history-values">{{ roll.values.join(', ') }}</span>
              <span class="history-time">{{ roll.timestamp }}</span>
              <span class="history-total">Total: {{ roll.total }}</span>
            </div>
          </div>

          <div v-else class="empty-history">
            No roll history yet
          </div>
        </div>
      </div>
    </main>

    <!-- Footer -->
    <footer>
      <p>&copy; {{ new Date().getFullYear() }} - 3D Dice Roller</p>
    </footer>
  </div>
</template>

<style scoped>
footer {
  margin-top: 2rem;
  padding: 1rem;
  text-align: center;
  border-top: 1px solid var(--border-color);
  font-size: 0.9rem;
  color: #888;
}
</style>