<script setup lang="ts">
import { ref, onMounted, watch, computed, onBeforeMount } from 'vue';
import Dice from './components/Dice.vue';

const isDarkMode = ref(false);

const toggleTheme = () => {
  isDarkMode.value = !isDarkMode.value;
  document.body.classList.toggle('dark-theme', isDarkMode.value);
};

const diceCount = ref(1);
const diceValues = ref<number[]>([]);
const diceRefs = ref<any[]>([]);
const history = ref<{ values: number[], total: number, timestamp: string }[]>([]);
const isRolling = ref(false);
const maxDiceCount = 3;
const hasRolled = ref(false);

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
          hasRolled.value = true;
          
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
  // Initial random values without animation
  const initialValues = Array(diceCount.value).fill(0).map(() => 
    Math.floor(Math.random() * 6) + 1
  );
  diceValues.value = initialValues;
  
  // No history entry for initial state
});
</script>

<template>
  <div class="app-wrapper">
    <div class="container">
      <!-- Header -->
      <header>
        <h1><i class="ri-dice-line"></i> 3D Dice Roller</h1>
        <button class="theme-toggle" @click="toggleTheme">
          <i v-if="isDarkMode" class="ri-sun-line"></i>
          <i v-else class="ri-moon-line"></i>
        </button>
      </header>

      <!-- Main content -->
      <main>
        <div class="layout-grid">
          <!-- Dice Card -->
          <div class="card dice-card">
            <!-- Roll Instructions in top left -->
            <div class="roll-instructions">
              <i class="ri-cursor-fill"></i> Click dice to roll
            </div>
            
            <!-- Dice Count Controls in top right -->
            <div class="dice-count-controls">
              <button 
                class="counter-btn" 
                @click="decrementDiceCount"
                :disabled="isRolling || diceCount <= 1"
                title="Decrease dice count"
              ><i class="ri-subtract-line"></i></button>
              <span class="counter-label">{{ diceCount }}</span>
              <button 
                class="counter-btn" 
                @click="incrementDiceCount"
                :disabled="isRolling || diceCount >= maxDiceCount"
                title="Increase dice count"
              ><i class="ri-add-line"></i></button>
            </div>
            
            <!-- Dice Container -->
            <div class="dice-container">
              <Dice 
                v-for="(value, index) in diceValues" 
                :key="index"
                :value="value"
                :index="index"
                :total="diceCount"
                ref="diceRefs"
                @roll-requested="rollDice"
              />
            </div>

            <!-- Total Result -->
            <div style="display: flex; justify-content: center;">
              <transition name="fade">
                <div 
                  v-if="!isRolling && hasRolled && diceCount > 1" 
                  class="total-result"
                  :style="{ backgroundColor: showRollCompleteAlert ? 'var(--secondary-color)' : 'var(--primary-color)' }"
                >
                  <i v-if="showRollCompleteAlert" class="ri-check-line"></i>
                  <i v-else class="ri-equalizer-line"></i>
                  Total: {{ totalValue }}
                </div>
              </transition>
            </div>
          </div>

          <!-- History Section -->
          <div class="history-section card">
            <div class="history-header">
              <h2><i class="ri-history-line"></i> Roll History</h2>
              <button 
                class="clear-btn" 
                @click="clearHistory"
                :disabled="history.length === 0"
                title="Clear history"
              >
                <i class="ri-delete-bin-line"></i> Clear
              </button>
            </div>

            <div v-if="history.length > 0" class="history-list">
              <div v-for="(roll, index) in history" :key="index" class="history-item">
                <span class="history-values"><i class="ri-dice-multiple-line"></i> {{ roll.values.join(', ') }}</span>
                <span class="history-time"><i class="ri-time-line"></i> {{ roll.timestamp }}</span>
                <span class="history-total">Total: {{ roll.total }}</span>
              </div>
            </div>

            <div v-else class="empty-history">
              <i class="ri-inbox-line empty-icon"></i>
              <p>No roll history yet</p>
            </div>
          </div>
        </div>
      </main>
    </div>

    <!-- Footer -->
    <footer>
      <p>
        <i class="ri-copyright-line"></i> {{ new Date().getFullYear() }} - 3D Dice Roller - 
        Created by <a href="https://lauroguedes.dev" target="_blank" rel="noopener noreferrer">Lauro Guedes</a>
      </p>
      <div class="social-links">
        <a href="https://linkedin.com/in/lauroguedes" target="_blank" rel="noopener noreferrer" title="LinkedIn">
          <i class="ri-linkedin-fill"></i>
        </a>
        <a href="https://github.com/lauroguedes/3d-dice-roller" target="_blank" rel="noopener noreferrer" title="GitHub">
          <i class="ri-github-fill"></i>
        </a>
      </div>
    </footer>
  </div>
</template>

<style scoped>
.app-wrapper {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
}

.container {
  flex: 1;
}

footer {
  margin-top: 2rem;
  padding: 1rem;
  text-align: center;
  border-top: 1px solid var(--border-color);
  font-size: 0.9rem;
  color: #888;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.5rem;
  width: 100%;
}

.social-links {
  display: flex;
  gap: 1rem;
}

.social-links a {
  color: var(--primary-color);
  font-size: 1.5rem;
  transition: color 0.2s, transform 0.2s;
}

.social-links a:hover {
  color: var(--secondary-color);
  transform: scale(1.1);
}

.social-links i {
  margin-right: 0;
}

footer a {
  color: var(--primary-color);
  text-decoration: none;
  transition: color 0.2s;
}

footer a:hover {
  color: var(--secondary-color);
}

i {
  vertical-align: middle;
  margin-right: 4px;
}

h1 i {
  font-size: 1.8rem;
  color: var(--primary-color);
  margin-right: 8px;
}

.theme-toggle i {
  font-size: 1.5rem;
  margin-right: 0;
}

.roll-btn i {
  font-size: 1.1rem;
}

.counter-btn i {
  margin-right: 0;
}

.empty-icon {
  font-size: 2.5rem;
  margin-bottom: 0.5rem;
  opacity: 0.5;
  display: block;
}

.empty-history {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 2rem 0;
}

.empty-history p {
  margin: 0;
}

/* Position the dice count controls in the top right */
.card {
  position: relative;
  padding: 1.5rem;
}

.dice-card {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}

.roll-instructions {
  position: absolute;
  top: 15px;
  left: 15px;
  font-size: 0.8rem;
  color: var(--text-color-secondary, #666);
  opacity: 0.8;
  display: flex;
  align-items: center;
  gap: 4px;
}

.roll-instructions i {
  font-size: 0.7rem;
}

.dice-count-controls {
  position: absolute;
  top: 15px;
  right: 15px;
  display: flex;
  align-items: center;
  gap: 0.5rem;
  z-index: 10;
}

.counter-btn {
  background-color: var(--bg-color-secondary);
  color: var(--text-color);
  border: 1px solid var(--border-color);
  border-radius: 50%;
  width: 28px;
  height: 28px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: all 0.2s ease;
  padding: 0;
  font-size: 0.9rem;
}

.counter-btn:hover:not(:disabled) {
  background-color: var(--primary-color);
  color: white;
}

.counter-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.counter-label {
  font-size: 0.9rem;
  font-weight: 500;
  color: var(--text-color);
  min-width: 16px;
  text-align: center;
}

/* Center the dice in the container */
.dice-container {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  align-items: center;
  gap: 1.5rem;
  min-height: 140px;
  padding: 1rem 0;
  margin: 0 auto;
  width: 100%;
}

/* Remove the divider and controls section */
.divider {
  display: none;
}

/* Improve the total result display */
.total-result {
  margin-top: 0.5rem;
  padding: 0.5rem 1rem;
  border-radius: 20px;
  color: white;
  font-weight: 500;
  display: inline-flex;
  align-items: center;
  gap: 0.5rem;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

/* Add scrolling to history list */
.history-list {
  max-height: 300px;
  overflow-y: auto;
  scrollbar-width: thin;
  scrollbar-color: var(--border-color) transparent;
}

.history-list::-webkit-scrollbar {
  width: 6px;
}

.history-list::-webkit-scrollbar-track {
  background: transparent;
}

.history-list::-webkit-scrollbar-thumb {
  background-color: var(--border-color);
  border-radius: 6px;
}

/* Fade transition */
.fade-enter-active, .fade-leave-active {
  transition: opacity 0.3s;
}

.fade-enter-from, .fade-leave-to {
  opacity: 0;
}
</style>