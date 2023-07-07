<template>
  <img class="mastery" :src="masteryIconUrl" alt="Mastery Point">
</template>

<script setup>
import { defineProps, computed } from 'vue';

const props = defineProps({
  type: String
})

const masteryTypes = {
  'Tyria': 'https://cdn.glitch.global/b91f7cf5-ad74-42de-8335-8724d4534fdb/MP-Tyria.png?v=1645544635296',
  'Maguuma': 'https://cdn.glitch.global/b91f7cf5-ad74-42de-8335-8724d4534fdb/MP-HoT.png?v=1645544635296',
  'Desert': 'https://cdn.glitch.global/b91f7cf5-ad74-42de-8335-8724d4534fdb/MP-PoF.png?v=1645544635296',
  'Tundra': 'https://cdn.glitch.global/b91f7cf5-ad74-42de-8335-8724d4534fdb/MP-IBS.png?v=1645544635296',
  'Cantha': 'https://cdn.glitch.global/b91f7cf5-ad74-42de-8335-8724d4534fdb/MP-EoD.png?v=1645544635296'
}
  
const masteryIconUrl = computed(() => {
  return masteryTypes[props.type] || masteryTypes['Cantha'];
});
  
</script>

<style scoped></style>