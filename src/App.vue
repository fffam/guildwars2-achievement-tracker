<template>
  <div class="wrapper">
    <div class="content">
      <div class="api-key-holder">
        <input v-model="apiKey" placeholder="API KEY">
      </div>
      <AchievementTracker :api-key="apiKey" />      
    </div>
  </div>
</template>

<script setup>
  import AchievementTracker from './components/AchievementTracker.vue'
  
  import { ref, defineProps, watch, computed, onMounted } from 'vue';
  import apiClient from 'gw2api-client';
  import { set as setInKeyStore, get as getFromKeyStore, createStore } from 'idb-keyval';
  
  const apiKey = ref("");

  const keyStore = createStore('gw2tracker', 'gw2tracker-apikey');
  
  const loadApiKey = async () => {
    console.log('Loading API key from cache');
    let cachedKey = await getFromKeyStore('apiKey',keyStore);
    console.log('Cached key:', cachedKey)
    if( cachedKey ) {
      apiKey.value = cachedKey;
    }
  }
  
  onMounted(async () => {

    await loadApiKey();
  });
  
  watch(apiKey, async (newKey, oldKey) => {
    if (newKey.length == 72 ) {
      console.log('Caching key:', newKey);
      let cacheResult = await setInKeyStore( 'apiKey', newKey, keyStore );
    }
  });
  
  
  
</script>

<style>  
  .content .api-key-holder {
    background: #444;
    padding: 0.25em;
  }
  .content .api-key-holder input {
    width: calc(100% - 1em);
    border: none;
    padding: 0.5em 0.5em;
    background: #555;
    color: #FFF;
  }
</style>