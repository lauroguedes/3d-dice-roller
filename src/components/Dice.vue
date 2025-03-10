<script setup lang="ts">
import { ref, onMounted } from 'vue';
import gsap from 'gsap';

const props = defineProps<{
  value: number;
  index?: number; // Add index prop to know which dice this is
  total?: number; // Add total prop to know how many dice there are
}>();

const emit = defineEmits(['roll-requested']);

const diceRef = ref<HTMLElement | null>(null);
const isRolling = ref(false);
const activeFace = ref<HTMLElement | null>(null);
const currentValue = ref(props.value);

// Computed property to determine if badge should be shown
const showBadge = () => {
  // Only show badge if this is part of a multi-dice setup (total > 1)
  return props.total && props.total > 1;
};

// Map dice values to specific rotations
// These rotations are carefully calibrated to match the CSS transforms and template
type Rotation = { x: number; y: number; z: number };
type FaceRotations = { [key: number]: Rotation } & {
  1: Rotation;
  2: Rotation;
  3: Rotation;
  4: Rotation;
  5: Rotation;
  6: Rotation;
};

const faceRotations: FaceRotations = {
  1: { x: 0, y: 0, z: 0 },         // front
  2: { x: 0, y: -90, z: 0 },       // left (value-2 is on the left face)
  3: { x: -90, y: 0, z: 0 },       // top
  4: { x: 90, y: 0, z: 0 },        // bottom
  5: { x: 0, y: 90, z: 0 },        // right (value-5 is on the right face)
  6: { x: 0, y: 180, z: 0 }        // back
};

// Function to handle dice click
const handleDiceClick = () => {
  if (!isRolling.value) {
    emit('roll-requested');
  }
};

// Function to highlight the active face after rolling
const highlightActiveFace = () => {
  // Find the face with the current value
  const faces = diceRef.value?.querySelectorAll('.face');
  faces?.forEach(face => {
    if (face.classList.contains(`value-${currentValue.value}`)) {
      activeFace.value = face as HTMLElement;
    }
  });

  if (activeFace.value) {
    // Get all dots on the active face
    const dots = activeFace.value.querySelectorAll('.dot');
    
    // Create a timeline for the dots
    const tl = gsap.timeline();
    
    // Pulse the dots
    tl.to(dots, {
      duration: 0.2,
      backgroundColor: '#ffeb3b',
      scale: 1.3,
      ease: "power1.inOut"
    })
    .to(dots, {
      duration: 0.2,
      backgroundColor: '#333',
      scale: 1,
      ease: "power1.inOut"
    })
    .to(dots, {
      duration: 0.2,
      backgroundColor: '#ffeb3b',
      scale: 1.3,
      ease: "power1.inOut"
    })
    .to(dots, {
      duration: 0.2,
      backgroundColor: '#333',
      scale: 1,
      ease: "power1.inOut"
    });
  }
};

// Function to roll the dice with a new value and callback when complete
const rollDice = (newValue?: number, onComplete?: () => void) => {
  if (!diceRef.value || isRolling.value) return;
  
  // If a new value is provided, use it
  if (newValue !== undefined) {
    currentValue.value = newValue;
  }
  
  isRolling.value = true;
  
  // Kill any existing animations
  gsap.killTweensOf(diceRef.value);
  
  // Get the final rotation for the current value
  const value = currentValue.value;
  // Ensure value is a valid dice value (1-6)
  const diceValue = value >= 1 && value <= 6 ? value as 1|2|3|4|5|6 : 1;
  const finalRotation = faceRotations[diceValue];
  
  // Add more pronounced random tilt for realism
  // Increased tilt range from ±5 to ±15 degrees
  const tiltX = Math.random() * 30 - 15;
  const tiltY = Math.random() * 30 - 15;
  const tiltZ = Math.random() * 30 - 15;
  
  // Ensure at least one axis has significant tilt
  const ensureMinimumTilt = () => {
    const minTilt = 12; // Increased minimum tilt from 5 to 12 degrees
    if (Math.abs(tiltX) < minTilt && Math.abs(tiltY) < minTilt && Math.abs(tiltZ) < minTilt) {
      // If all tilts are too small, increase one randomly
      const axis = Math.floor(Math.random() * 3);
      if (axis === 0) return { x: tiltX + (tiltX >= 0 ? minTilt : -minTilt), y: tiltY, z: tiltZ };
      if (axis === 1) return { x: tiltX, y: tiltY + (tiltY >= 0 ? minTilt : -minTilt), z: tiltZ };
      return { x: tiltX, y: tiltY, z: tiltZ + (tiltZ >= 0 ? minTilt : -minTilt) };
    }
    return { x: tiltX, y: tiltY, z: tiltZ };
  };
  
  const tilt = ensureMinimumTilt();
  
  const realisticTilt = {
    x: finalRotation.x + tilt.x,
    y: finalRotation.y + tilt.y,
    z: finalRotation.z + tilt.z
  };
  
  // Reset to a neutral position first to ensure consistent animation
  gsap.set(diceRef.value, {
    rotationX: 0,
    rotationY: 0,
    rotationZ: 0
  });
  
  // Create a single animation timeline with more controlled randomness
  gsap.timeline({
    onComplete: () => {
      isRolling.value = false;
      highlightActiveFace();
      
      // Call the onComplete callback if provided
      if (onComplete) {
        onComplete();
      }
    }
  })
  .to(diceRef.value, {
    duration: 0.1,
    scale: 0.9,
    ease: "power1.in"
  })
  .to(diceRef.value, {
    duration: 1.2,
    rotationX: 720 + Math.floor(Math.random() * 2) * 180,
    rotationY: 720 + Math.floor(Math.random() * 2) * 180,
    rotationZ: Math.floor(Math.random() * 2) * 180,
    scale: 1,
    ease: "power2.out"
  })
  .to(diceRef.value, {
    duration: 0.5,
    rotationX: realisticTilt.x,
    rotationY: realisticTilt.y,
    rotationZ: realisticTilt.z,
    ease: "power1.inOut",
    onComplete: () => {
      // Add a small settling movement to enhance realism
      gsap.to(diceRef.value, {
        duration: 0.15,
        rotationX: realisticTilt.x - (tilt.x * 0.1),
        rotationY: realisticTilt.y - (tilt.y * 0.1),
        rotationZ: realisticTilt.z - (tilt.z * 0.1),
        ease: "power1.out"
      });
    }
  });
};

// Initialize the dice with the correct rotation on mount
onMounted(() => {
  if (diceRef.value) {
    const value = currentValue.value;
    // Ensure value is a valid dice value (1-6)
    const diceValue = value >= 1 && value <= 6 ? value as 1|2|3|4|5|6 : 1;
    const initialRotation = faceRotations[diceValue];
    
    // Add more pronounced tilt for initial position
    const tiltX = Math.random() * 24 - 12;
    const tiltY = Math.random() * 24 - 12;
    const tiltZ = Math.random() * 24 - 12;
    
    // Ensure at least one axis has significant tilt
    const minTilt = 10; // Increased from 4 to 10
    if (Math.abs(tiltX) < minTilt && Math.abs(tiltY) < minTilt && Math.abs(tiltZ) < minTilt) {
      const axis = Math.floor(Math.random() * 3);
      if (axis === 0) {
        gsap.set(diceRef.value, {
          rotationX: initialRotation.x + tiltX + (tiltX >= 0 ? minTilt : -minTilt),
          rotationY: initialRotation.y + tiltY,
          rotationZ: initialRotation.z + tiltZ
        });
      } else if (axis === 1) {
        gsap.set(diceRef.value, {
          rotationX: initialRotation.x + tiltX,
          rotationY: initialRotation.y + tiltY + (tiltY >= 0 ? minTilt : -minTilt),
          rotationZ: initialRotation.z + tiltZ
        });
      } else {
        gsap.set(diceRef.value, {
          rotationX: initialRotation.x + tiltX,
          rotationY: initialRotation.y + tiltY,
          rotationZ: initialRotation.z + tiltZ + (tiltZ >= 0 ? minTilt : -minTilt)
        });
      }
    } else {
      gsap.set(diceRef.value, {
        rotationX: initialRotation.x + tiltX,
        rotationY: initialRotation.y + tiltY,
        rotationZ: initialRotation.z + tiltZ
      });
    }
  }
});

// Test function to verify face rotations
const testAllFaces = () => {
  if (!diceRef.value) return;
  
  let currentFace = 1;
  const interval = setInterval(() => {
    if (currentFace > 6) {
      clearInterval(interval);
      return;
    }
    
    currentValue.value = currentFace;
    // Ensure value is a valid dice value (1-6)
    const diceValue = currentFace >= 1 && currentFace <= 6 ? currentFace as 1|2|3|4|5|6 : 1;
    const rotation = faceRotations[diceValue];
    
    // Add more pronounced tilt for test function
    const tiltX = Math.random() * 24 - 12;
    const tiltY = Math.random() * 24 - 12;
    const tiltZ = Math.random() * 24 - 12;
    
    // Ensure at least one axis has significant tilt
    const minTilt = 12; // Increased from 5 to 12
    let finalX = rotation.x + tiltX;
    let finalY = rotation.y + tiltY;
    let finalZ = rotation.z + tiltZ;
    
    if (Math.abs(tiltX) < minTilt && Math.abs(tiltY) < minTilt && Math.abs(tiltZ) < minTilt) {
      const axis = Math.floor(Math.random() * 3);
      if (axis === 0) finalX += (tiltX >= 0 ? minTilt : -minTilt);
      else if (axis === 1) finalY += (tiltY >= 0 ? minTilt : -minTilt);
      else finalZ += (tiltZ >= 0 ? minTilt : -minTilt);
    }
    
    gsap.to(diceRef.value, {
      duration: 0.5,
      rotationX: finalX,
      rotationY: finalY,
      rotationZ: finalZ,
      onComplete: highlightActiveFace
    });
    
    currentFace++;
  }, 2000);
};

defineExpose({ rollDice, testAllFaces });
</script>

<template>
  <div class="dice-container">
    <div style="position: relative;">
      <!-- Only show badge if this is part of a multi-dice setup -->
      <div class="dice-badge" v-if="!isRolling && showBadge()">
        {{ props.value }}
      </div>
      <div class="dice-wrapper">
        <div 
          ref="diceRef" 
          class="dice" 
          :class="{ rolling: isRolling, clickable: !isRolling }" 
          @click="handleDiceClick"
        >
          <div class="face front value-1">
            <div class="dot center"></div>
          </div>
          <div class="face back value-6">
            <div class="dot top-left"></div>
            <div class="dot top-right"></div>
            <div class="dot middle-left"></div>
            <div class="dot middle-right"></div>
            <div class="dot bottom-left"></div>
            <div class="dot bottom-right"></div>
          </div>
          <div class="face right value-2">
            <div class="dot top-right"></div>
            <div class="dot bottom-left"></div>
          </div>
          <div class="face left value-5">
            <div class="dot top-left"></div>
            <div class="dot top-right"></div>
            <div class="dot center"></div>
            <div class="dot bottom-left"></div>
            <div class="dot bottom-right"></div>
          </div>
          <div class="face top value-3">
            <div class="dot top-right"></div>
            <div class="dot center"></div>
            <div class="dot bottom-left"></div>
          </div>
          <div class="face bottom value-4">
            <div class="dot top-left"></div>
            <div class="dot top-right"></div>
            <div class="dot bottom-left"></div>
            <div class="dot bottom-right"></div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.dice-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  position: relative;
}

.dice-wrapper {
  width: 100px;
  height: 100px;
  perspective: 1000px;
  margin: 10px;
  position: relative;
}

.dice {
  width: 100%;
  height: 100%;
  position: relative;
  transform-style: preserve-3d;
  transition: transform 0.1s;
}

.clickable {
  cursor: pointer;
}

.clickable:hover {
  transform: scale(1.05);
  transition: transform 0.2s ease;
}

.face {
  position: absolute;
  width: 100%;
  height: 100%;
  background: white;
  border: 2px solid #ccc;
  box-shadow: inset 0 0 15px rgba(0, 0, 0, 0.1);
  border-radius: 10px;
  backface-visibility: visible;
  transform-style: preserve-3d;
  transform-origin: center center;
  padding: 10px;
}

.face.active {
  z-index: 1;
}

.dot {
  position: absolute;
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background-color: #333;
  box-shadow: inset 0 0 5px rgba(0, 0, 0, 0.5);
}

/* Dot positions */
.center {
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}

.top-left {
  top: 15%;
  left: 15%;
}

.top-right {
  top: 15%;
  right: 15%;
}

.bottom-left {
  bottom: 15%;
  left: 15%;
}

.bottom-right {
  bottom: 15%;
  right: 15%;
}

.middle-left {
  top: 50%;
  left: 15%;
  transform: translateY(-50%);
}

.middle-right {
  top: 50%;
  right: 15%;
  transform: translateY(-50%);
}

.front { transform: translateZ(50px); }
.back { transform: rotateY(180deg) translateZ(50px); }
.right { transform: rotateY(90deg) translateZ(50px); }
.left { transform: rotateY(-90deg) translateZ(50px); }
.top { transform: rotateX(90deg) translateZ(50px); }
.bottom { transform: rotateX(-90deg) translateZ(50px); }

.rolling {
  transition: none;
  cursor: default;
}

.fade-enter-active, .fade-leave-active {
  transition: opacity 0.3s;
}

.fade-enter-from, .fade-leave-to {
  opacity: 0;
}

.dice-badge {
  position: absolute;
  top: -10px;
  right: -10px;
  background-color: var(--primary-color, #4caf50);
  color: white;
  border-radius: 50%;
  width: 24px;
  height: 24px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 0.8rem;
  font-weight: bold;
  z-index: 10;
}
</style>