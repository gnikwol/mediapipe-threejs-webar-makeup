<script setup lang="ts">
import * as THREE from 'three';
import { computed, ref } from 'vue';
import Makeup from './components/Makeup.vue';

const lipstickMaterial = computed(() =>
  new THREE.MeshBasicMaterial({
    color: 0xe761a4,
    transparent: true,
    opacity: lip.value ? 0.1 : 0, // lip が OFF なら表示しない
  })
);

const lip = ref(true);
</script>

<template>
  <Makeup :lipstick-material="lipstickMaterial" :component-size="640" />
  <button @click="() => lip = !lip">
    リップ {{ !lip ? 'ON' : 'OFF' }}
  </button>
</template>
