<script setup lang="ts">
import { computed, ref } from 'vue';
import type { EntityConfig } from '../../types/floorplan';
import { useFloorplanStore } from '../../stores/floorplan';

const props = defineProps<{
  entity: EntityConfig
}>();

const store = useFloorplanStore();
const isSelected = computed(() => store.selectedEntityId === props.entity.id);

// Drag logic
const isDragging = ref(false);
const dragStart = ref({ x: 0, y: 0 });

// We need to calculate % movement based on parent size. 
// Since we don't have easy access to parent ref here without inject/props, 
// we'll rely on event.target.offsetParent to get dimensions.

function onMouseDown(event: MouseEvent) {
  event.stopPropagation();
  store.selectedEntityId = props.entity.id;
  
  isDragging.value = true;
  dragStart.value = { x: event.clientX, y: event.clientY };
  
  window.addEventListener('mousemove', onMouseMove);
  window.addEventListener('mouseup', onMouseUp);
}

function onMouseMove(event: MouseEvent) {
  if (!isDragging.value) return;
  
  const dx = event.clientX - dragStart.value.x;
  const dy = event.clientY - dragStart.value.y;
  
  // Calculate percentage change
  // We need the container dimensions. 
  // We can get them from the store or DOM. 
  // Let's assume the entity's parent is the .image-wrapper
  const container = document.querySelector('.image-wrapper'); 
  if (!container) return;
  
  const rect = container.getBoundingClientRect();
  const width = rect.width;
  const height = rect.height;
  
  const dxPercent = (dx / width) * 100;
  const dyPercent = (dy / height) * 100;
  
  store.updateEntity(props.entity.id, {
    x: props.entity.x + dxPercent,
    y: props.entity.y + dyPercent
  });
  
  dragStart.value = { x: event.clientX, y: event.clientY };
}

function onMouseUp() {
  isDragging.value = false;
  window.removeEventListener('mousemove', onMouseMove);
  window.removeEventListener('mouseup', onMouseUp);
}


// Style computation
const styleObject = computed(() => {
  const { shape, style, x, y } = props.entity;
  
  const baseStyle: Record<string, any> = {
    left: `${x}%`,
    top: `${y}%`,
    width: `${style.width}%`,
    height: `${style.height}%`,
    backgroundColor: style.offColor,
    opacity: style.offOpacity,
    transform: `translate(-50%, -50%) rotate(${style.rotation}deg)`,
    position: 'absolute' as const,
    border: isSelected.value ? '2px solid var(--color-primary)' : '2px solid transparent',
    borderRadius: shape === 'circle' ? '50%' : '4px', // default for square/rect
    cursor: 'move',
    zIndex: isSelected.value ? 10 : 1
  };
  
  if (shape === 'square') {
      // For square we might want aspect ratio lock, but here w/h are % of parent.
      // If we want a true square, it's tricky with % unless parent is square.
      // We will stick to w/h percentages for now.
  }
  
  return baseStyle;
});

</script>

<template>
  <div 
    class="entity-overlay" 
    :style="styleObject"
    @mousedown="onMouseDown"
    @click.stop
  >
    <!-- Optional: Label inside or tooltip? -->
    <div v-if="entity.labelConfig.show" class="entity-label">
        {{ entity.label }}
    </div>
  </div>
</template>

<style scoped>
.entity-overlay {
  box-sizing: border-box;
  transition: border-color 0.2s;
}

.entity-label {
    position: absolute;
    top: 100%;
    left: 50%;
    transform: translateX(-50%);
    background: rgba(0,0,0,0.7);
    color: white;
    padding: 2px 4px;
    font-size: 0.75rem;
    border-radius: 4px;
    white-space: nowrap;
    white-space: nowrap;
    pointer-events: auto;
    margin-top: 4px;
}
</style>
