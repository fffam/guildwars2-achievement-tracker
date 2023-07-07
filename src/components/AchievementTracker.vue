<template>
<div class="achievement-tracker">
  <header>
    <div class="update-info" :style="{ background: updateProgressBackground }">
      <span class="last-update" v-if="loaded">Next update: {{updateTimeFriendly}}</span>
      <button class="turbo" v-if="loaded" :class="{ 'action-turbo--active': isTurbo }">
        <i aria-label="Turbo" class="button fa fa-forward action-turbo" title="Turbo" @click="toggleTurbo"></i>
      </button>
      <button @click="updateAchievementProgress" v-if="loaded">
        <i aria-label="Reload" class="button fa fa-refresh"></i>
      </button>
    </div>
    
    <div class="filter" v-if="loaded">
      <input class="filter" v-model="filterText" placeholder="Search">
    </div>

    <div class="category-assign">
      <select class="category-mass-assign" v-model="selectedCategory" v-if="categories.length">
        <option v-for="category in categories">{{category.name}}</option>
      </select>
      <button @click="addSelectedAchievementsToSelectedCategory()">Add selected</button>
    </div>
  </header>
  
  <div class="achievement-tracker--body">
    <div class="loading-message" v-if="!loaded">
      <h2>Loading</h2>
      <progress class="loading-progress-indicator" max="100" :value="loadingProgress">{{loadingProgress}}</progress>
    </div>

    <details v-if="loaded" open>    
    <summary><strong>Pinned</strong> <small :data-count="filteredPinnedAchievements.length">{{filteredPinnedAchievements.length}}</small></summary>
    <ul>
      <Achievement v-for="achievement in filteredPinnedAchievements" :key="achievement.id" :achievement="achievement" @unpinned="onAchievementUnpinned" @ignore="onAchievementIgnored"></Achievement>
    </ul>
    </details>

    <button v-if="loaded" class="add-category" @click="createNewCategoryAfter()"><i class="fa fa-plus" aria-hidden="true"></i> New category</button>

    <div v-for="category in categories" v-if="loaded">
    <details>    
    <summary><strong>{{category.name}}</strong> <small :data-count="category.achievements.length">{{category.achievements.length}}</small> <i class="fa fa-trash-o" aria-hidden="true" title="Delete category" @click="deleteCategory(category.name)"></i> </summary>
    <ul>
      <Achievement v-for="achievement in category.achievements" :key="achievement.id" :achievement="achievement" :userCategory="category.name" @unpinned="onAchievementUnpinned" @ignore="onAchievementIgnored" @categoryRemoved="onCategoryRemovedFromAchievement"></Achievement>
    </ul>
    </details>
    <button class="add-category" @click="createNewCategoryAfter(category.name)"><i class="fa fa-plus" aria-hidden="true"></i> New category</button>  
    </div>

    <details v-if="loaded">    
    <summary><strong>Incomplete</strong> <small :data-count="filteredIncompleteAchievements.length">{{filteredIncompleteAchievements.length}}</small></summary>
    <ul>
      <Achievement v-for="achievement in filteredIncompleteAchievements" :key="achievement.id" :achievement="achievement" @pinned="onAchievementPinned" @unpinned="onAchievementUnpinned" @ignore="onAchievementIgnored"></Achievement>
    </ul>
    </details>

    <details v-if="loaded">    
    <summary><strong>Complete</strong> <small :data-count="filteredCompleteAchievements.length">{{filteredCompleteAchievements.length}}</small></summary>
    <ul>
      <Achievement v-for="achievement in filteredCompleteAchievements" :key="achievement.id" :achievement="achievement" @pinned="onAchievementPinned" @unpinned="onAchievementUnpinned" @ignore="onAchievementIgnored"></Achievement>
    </ul>
    </details>

    <details v-if="loaded">    
    <summary><strong>Ignored</strong> <small :data-count="filteredIgnoredAchievements.length">{{filteredIgnoredAchievements.length}}</small></summary>
    <ul>
      <Achievement v-for="achievement in filteredIgnoredAchievements" :key="achievement.id" :achievement="achievement" @pinned="onAchievementPinned" @unpinned="onAchievementUnpinned" @unignore="onAchievementUnignored"></Achievement>
    </ul>
    </details>
  </div>
</div>
</template>

<script setup>
  import tierRewardsData from './../data/tierrewards.json';
  import Achievement from './Achievement.vue';
  import { ref, defineProps, watch, watchEffect, computed, onMounted } from 'vue';
  import useDebouncedRef from './../composables/useDebouncedRef.js';
  import apiClient from 'gw2api-client';
  import cacheBrowserStorage from 'gw2api-client/src/cache/browser';
  import { set as setInKeyStore, get as getFromKeyStore, createStore } from 'idb-keyval';
  const userDataKeyStore = createStore('gw2tracker-userdata', 'gw2tracker-userdata');
  
  const sleep = async (ms) => {
    return new Promise(resolve => setTimeout(resolve, ms));
  }

  // Props
  const props = defineProps({
    apiKey: String
  });

  // Refs
  const achievements = ref([]);
  const loadingProgress = ref(0);
  const loaded = ref(false);
  const filterText = ref('');
  const userData = ref({});
  const selectedCategory = ref('');
  const isTurboMode = ref(false);
  const lastUpdateTime = ref(new Date());
  const secondsBetweenUpdates = ref(299);
  const secondsBetweenTurboUpdates = ref(60);
  const msElapsedSinceLastUpdate = ref(0);
  
  let tierRewardsMap = {};
  for( let tierRewardsGroup of tierRewardsData ) {
    for( let tierRewardAchievementId in tierRewardsGroup ) {
      tierRewardsMap[tierRewardAchievementId] = tierRewardsGroup[tierRewardAchievementId].tiers;
    }
  }

  let api = apiClient();
  const options = {
    gcTick: 3 * 60 * 60 * 1000 // 3 hours
  }
  const cache = cacheBrowserStorage(options);
  api.cacheStorage(cache);
  api.language('en');

  const loadAchievementsWithGroups = async () => {

    await api.flushCacheIfGameUpdated();
    loadingProgress.value = 0;
    
    // Load all the groups
    let groups = await api.achievements().groups().all();
    loadingProgress.value = 5;
    
    // For each group, load the categories
    let achievementToCategoryMap = {};
    for( let groupIdx in groups ) {
      let group = groups[groupIdx];
      if( group.categories ) {
        let categories = await api.achievements().categories().many(group.categories);
        for( let category of categories ) {
          category.group = group;
          for( let achievementId of category.achievements ) {
            achievementToCategoryMap[achievementId] = category; 
          }
        }
      }
      loadingProgress.value = 5 + (25*(groupIdx/groups.length));
    }

    // Load the actual list of achievements
    let data = await api.achievements().all();
    let achievementData = {};
    
    for( let achieve of data ) {
      // Find the category
      achieve.category = achievementToCategoryMap[achieve.id];
      
      // Add in the tier rewards data
      if( tierRewardsMap[achieve.id] ) {
        achieve.tierrewards = tierRewardsMap[achieve.id];
        achieve.additionalTextForSearch = achieve.tierrewards.map(a=>a.name).join('|');
      }
      
      if( !achieve.max ) {
        achieve.max = 1;
      }
      achievementData[achieve.id] = achieve;
    }
    loadingProgress.value = 60;
    let returnVal = Object.values(achievementData).filter( a => {
      return true;
      //return a.name.includes('Glory to the') || a.name.includes('Morale') || a.name.includes('Stretch') || a.name.includes('Stormcaller') || a.name.includes('Gold Fractal Master') || a.name.includes('Coliseum') || a.name.includes('Style Guide') || a.name.includes('Set III')
    });
    //returnVal = Object.values(achievementData);
    return returnVal;
  }
  
  watch(() => props.apiKey, async(newKey, oldKey) => {
    if( newKey.length == 72 && newKey != oldKey ) {
      try {
        loaded.value = false;
        api.authenticate(newKey);
        achievements.value = await loadAchievementsWithGroups();
        await loadUserData();
        await loadAllBitAndRewardInfo();
        await updateAchievementProgress();
        loaded.value = true;
      } catch(e) {
        alert(e);
        console.log(e);
      }
    }
  });
  
  const loadAllBitAndRewardInfo = async () => {
    
    // Get all item from rewards
    // Also get all item IDs, minipet & skin IDs from bits (achievement requirements)
    let allItemIds = [];
    let allSkinIds = [];
    let allMinipetIds = [];
    achievements.value.forEach( (a) => {
      if( a.rewards ) {
        let achieveItemRewards = a.rewards.filter(r=>r.type=='Item').map(r=>r.id);
        if( achieveItemRewards.length ) {
          allItemIds = [...allItemIds,...achieveItemRewards];
        }
      }
      if( a.bits ) {
        let achieveBitItems = a.bits.filter(r=>r.type=='Item').map(r=>r.id);
        if( achieveBitItems.length ) {
          allItemIds = [...allItemIds,...achieveBitItems];
        }
        let achieveBitSkins = a.bits.filter(r=>r.type=='Skin').map(r=>r.id);
        if( achieveBitSkins.length ) {
          allSkinIds = [...allSkinIds,...achieveBitSkins];
        }
        let achieveMinipetSkins = a.bits.filter(r=>r.type=='Minipet').map(r=>r.id);
        if( achieveMinipetSkins.length ) {
          allMinipetIds = [...allMinipetIds,...achieveMinipetSkins];
        }
      }
    });

    const chunkArray = (arr, size) => {
      return Array.from({ length: Math.ceil(arr.length / size) }, (v, i) =>
        arr.slice(i * size, i * size + size)
      );
    }
    
    // Load the items in batches
    // Any IDs missing? Lets not try to load them next time to speed things up
    let missingItemIds = await cache.get('missing-item-ids') || [];
    let itemIdChunks = chunkArray(allItemIds.filter(x=>!missingItemIds.includes(x)).sort(), 40);
    let itemMap = {};
    for( let itemIds of itemIdChunks ) {
      let items = await api.items().many(itemIds);
      for( let item of items ) {
        itemMap[item.id] = item;
      }
      await sleep(50);
    }
    cache.set('missing-item-ids', allItemIds.filter(x => !itemMap[x]), 24 * 60 * 60 * 1000);
    
    // Load the skins in batches
    let missingSkinIds = await cache.get('missing-skin-ids') || [];
    let skinIdChunks = chunkArray(allSkinIds.filter(x=>!missingSkinIds.includes(x)).sort(), 40);
    let skinMap = {};
    for( let skinIds of skinIdChunks ) {
      let skins = await api.skins().many(skinIds);
      for( let skin of skins ) {
        skinMap[skin.id] = skin;
      }
      await sleep(50);
    }
    cache.set('missing-skin-ids', allSkinIds.filter(x => !skinMap[x]), 24 * 60 * 60 * 1000);
    
    // Load the minipets in batches
    let minipetIdChunks = chunkArray(allMinipetIds.sort(), 50);
    let minipetMap = {};
    for( let minipetIds of minipetIdChunks ) {
      let minis = await api.minis().many(minipetIds);
      for( let mini of minis ) {
        minipetMap[mini.id] = mini;
      }
      await sleep(50);
    }
    
    let titlesData = await api.titles().all();
    let titlesMap = {};
    for( let title of titlesData ) {
      titlesMap[title.id] = title;
    }
    
    // Assign items/titles to achievement rewards
    // Also add some of it to searchText for filtering
    for( let a of achievements.value) {
      let searchText = '';
      if( a.rewards ) {
        for( let r of a.rewards ) {
          if( r.type == 'Item' && itemMap[r.id] ) {
            r.item = itemMap[r.id];
            searchText += itemMap[r.id].name + '|';
          } else if( r.type == 'Title' && titlesMap[r.id] ) {
            r.title = titlesMap[r.id].name;
            searchText += r.title + '|';
          }
        }
      }
      if( searchText ) {
        if( a.additionalTextForSearch ) {
          a.additionalTextForSearch = a.additionalTextForSearch + '|' + searchText;  
        } else {
          a.additionalTextForSearch = searchText;
        }        
      }
      if( a.bits ) {
        for( let r of a.bits ) {
          if( r.type == 'Item' ) {
            r.item = itemMap[r.id];
          } else if( r.type == 'Skin' ) {
            r.skin = skinMap[r.id];
          } else if( r.type == 'Minipet' ) {
            r.minipet = minipetMap[r.id];
          }
        }
      }
    }

  };
  
  const updateAchievementProgress = async (a) => {
    
    msElapsedSinceLastUpdate.value = 0;
    
    let accountAchievementsEndpoint = api.account().achievements();
    accountAchievementsEndpoint.cacheTime = 0;
    let allAccountAchievements = await accountAchievementsEndpoint.all();
    let allAccountAchievementsMap = {};
    for( let ach of allAccountAchievements ) {
      allAccountAchievementsMap[ach.id] = ach;
    }
    for( let achievement of achievements.value ) {
      let accountAchievementData = allAccountAchievementsMap[achievement.id];
      
      if( !accountAchievementData ) {
        achievement.progress = 0;
        achievement.progressPct = 0;
        achievement.completed = false;
        achievement.repeated = 0;
        achievement.unlocked = false;
        continue;
      }
      
      achievement.progress = accountAchievementData.current || 0;
      achievement.progressPct = Math.round(100 * (accountAchievementData.current / (accountAchievementData.max||1)));
      achievement.max = accountAchievementData.max || 1;
      achievement.completed = accountAchievementData.done;
      achievement.repeated = accountAchievementData.repeated || 0;
      achievement.unlocked = accountAchievementData.unlocked || false;
      if( achievement.bits ) {
        for( let bitIdx in achievement.bits ) {
          let bit = achievement.bits[bitIdx];
          if( accountAchievementData.done && (!accountAchievementData.bits || !accountAchievementData.bits.length) ) {
            bit.completed = true;
          } else if(accountAchievementData.bits && accountAchievementData.bits.indexOf(1*bitIdx) > -1) {
            bit.completed = true;
          } else {
            bit.completed = false;
          }      
        }
      }
      
       
    }    
  }

  const calculateSearchScoreForAchievement = (a,search) => {
    let result = 0;
    search = search.toLowerCase();
    if( a.name.toLowerCase() == search ) result += 20;
    if( a.name.toLowerCase().indexOf(search) === 0 ) result += 5;
    if( a.name.toLowerCase().includes(search) ) result += 10;
    if( a.locked_text && a.locked_text.toLowerCase().includes(search) ) result += 2;
    if( a.requirement && a.requirement.toLowerCase().includes(search) ) result += 2;
    if( a.description && a.description.toLowerCase().includes(search) ) result += 1;
    if( a.category && a.category.name.toLowerCase().includes(search) ) result += 2;
    if( a.category && a.category.group && a.category.group.name.toLowerCase().includes(search) ) result += 1;
    if( a.additionalTextForSearch && a.additionalTextForSearch.toLowerCase().includes(search) ) result += 1;
    return result;
  };

  const filterAchievementList = ( additionalFilter ) => {
    if( !achievements.value ) {
      console.log('No value');
      return [];
    }
    
    // If no filter provided, return all achievements
    if( !additionalFilter ) {
      additionalFilter = (a) => true
    }
    
    let defaultSort = (a,b) => {
      return a.name - b.name;
    }

    let textFilterValue = filterText.value;
    if( !textFilterValue ) {
      return achievements.value.filter(additionalFilter).sort(defaultSort);
      
    // Short query string? Don't sort (to improve performance)
    } else if( textFilterValue && textFilterValue.length < 4 ) {
      let fullList = achievements.value;
      return fullList.filter(additionalFilter).filter( a => {
        return calculateSearchScoreForAchievement(a,textFilterValue) > 0
      });
    } else {
      let fullList = achievements.value;
      let result = fullList.filter(additionalFilter).filter( a => {
        return calculateSearchScoreForAchievement(a,textFilterValue) > 0
      }).sort( (a,b) => {
        let ascore = calculateSearchScoreForAchievement(a,textFilterValue);
        let bscore = calculateSearchScoreForAchievement(b,textFilterValue);
        return bscore-ascore;
      });
      return result;
    }
  }

  const filteredPinnedAchievements = computed(() => {
    return filterAchievementList( a => a.pinned && !a.ignored );
  });
  
  const filteredIncompleteAchievements = computed(() => {
    return filterAchievementList( a => !a.completed && !a.ignored );
  });
  
  const filteredCompleteAchievements = computed(() => {
    return filterAchievementList( a => a.completed && !a.ignored );
  });

  const filteredIgnoredAchievements = computed(() => {
    return filterAchievementList( a => a.ignored );
  });

  const filteredAchievements = computed(() => {
    return filterAchievementList();
  });
  
  const categories = computed(() => {
    if( userData.value.categorynames ) {
      return userData.value.categorynames.map( cat => {
        return {name:cat, achievements: filterAchievementList( a => a.categories && a.categories.includes(cat) ) };
      });
    } else {
      return [];
    }
  });


  
  
  const onAchievementPinned = async (achievementId) => {
    userData.value.pins = [...userData.value.pins,achievementId];
    userData.value.ignores = userData.value.ignores.filter( id => achievementId != id );
    await cacheUserData();
  }
  const onAchievementUnpinned = async (achievementId) => {
    userData.value.pins = userData.value.pins.filter( id => achievementId != id );
    await cacheUserData();
  }
  const onAchievementIgnored = async (achievementId) => {
    userData.value.ignores = [...userData.value.ignores,achievementId];
    await cacheUserData();
  }
  const onAchievementUnignored = async (achievementId) => {
    userData.value.ignores = userData.value.ignores.filter( id => achievementId != id );
    await cacheUserData();
  }
  
  const loadUserData = async () => {
    userData.value = await getFromKeyStore('userData',userDataKeyStore) || {};
    
    if( userData.value.pins ) {
      for( let achievement of achievements.value ) {
        if( userData.value.pins.includes(achievement.id) ) {
          achievement.pinned = true;
        }
      }
    } else {
      userData.value.pins = [];
    }
    
    if( userData.value.achievementcategories ) {
      for( let achievement of achievements.value ) {
        if( userData.value.achievementcategories[achievement.id] ) {
          achievement.categories = userData.value.achievementcategories[achievement.id];
        }
      }
      selectedCategory.value = userData.value.categorynames.length ? userData.value.categorynames[0] : '';
    } else {
      userData.value.achievementcategories = {};
      userData.value.categorynames = [];
      
    }
    
    if( userData.value.ignores ) {
      for( let achievement of achievements.value ) {
        if( userData.value.ignores.includes(achievement.id) ) {
          achievement.ignored = true;
        }
      }
    } else {
      userData.value.ignores = [];
    }

    // Setup checkbox for moving to category
    for( let achievement of achievements.value ) {
      achievement.isChecked = false;
    }
  }
  
  const cacheUserData = async () => {
    let cacheResult = await setInKeyStore( 'userData', JSON.parse(JSON.stringify(userData.value)), userDataKeyStore );
  }
  
  const createNewCategoryAfter = ( prevCategory ) => {
    
    let idx = 0;
    // No category specified? then create new category at top (under Pinned)
    if( prevCategory )  {
      idx = userData.value.categorynames ? (userData.value.categorynames.indexOf(prevCategory)+1) : 0;
    }
      
    let newName = prompt('Enter the new category name', 'Create category');
    if( newName ) {
      if( userData.value.categorynames.includes(newName) ) {
        alert('Category cannot have the same name as an existing category');
      }
      userData.value.categorynames.splice(idx,0,newName);
      selectedCategory.value = newName;
    }
    cacheUserData();
  }
  
  const deleteCategory = (categoryName) => {
    if( confirm("Delete category: "+categoryName+"?") ) {
      userData.value.categorynames = userData.value.categorynames.filter( a => a != categoryName );
    }
    if( selectedCategory.value == categoryName ) {
      if( userData.value.categorynames.length ) {
        selectedCategory.value = userData.value.categorynames[0];
      }
    }
    cacheUserData();
  }
  
  const addSelectedAchievementsToSelectedCategory = () => {
    if(!selectedCategory) return;
    achievements.value.filter( a => a.isChecked ).forEach( (achievement) => {
      if( !achievement.categories ) {
        achievement.categories = [];
      }
      if( !achievement.categories.includes(selectedCategory.value) ) {
        achievement.categories.push(selectedCategory.value);
      }
      userData.value.achievementcategories[achievement.id] = achievement.categories;
      achievement.isChecked = false;
    });
    cacheUserData();    
  }
  
  const onCategoryRemovedFromAchievement = (achievementId,categoryName) => {
    if( userData.value.achievementcategories[achievementId] ) {
      userData.value.achievementcategories[achievementId] = userData.value.achievementcategories[achievementId].filter( cat => cat != categoryName );
    }
    cacheUserData();
  }

  const toggleTurbo = () => {
    isTurboMode.value = !isTurboMode.value;
  }

  const isTurbo = computed(() => {
    return isTurboMode.value;
  });

  const updateTick = () => {
    msElapsedSinceLastUpdate.value += 200;
    let msBetweenUpdate = isTurboMode.value ? (secondsBetweenTurboUpdates.value*1000) : (secondsBetweenUpdates.value*1000);
    if( msElapsedSinceLastUpdate.value > msBetweenUpdate ) {
      updateAchievementProgress();      
    }
  }
  
  onMounted( () => {
    setInterval( updateTick, 200 );  
  });
  
  const updateTimeFriendly = computed(() => {
    let msBetweenUpdate = isTurboMode.value ? (secondsBetweenTurboUpdates.value*1000) : (secondsBetweenUpdates.value*1000);    
    let secondsLeft = Math.floor((msBetweenUpdate - msElapsedSinceLastUpdate.value)/1000);
    if( secondsLeft == 0 ) {
      return 'Updating...';
    }
    if( secondsLeft < 60 ) {
      return secondsLeft + 's';
    } else {
      return Math.floor(secondsLeft/60) + 'm ' + (secondsLeft%60) + 's';
    }
  });

  const updateProgressBackground = computed(() => {
    let msBetweenUpdate = isTurboMode.value ? (secondsBetweenTurboUpdates.value*1000) : (secondsBetweenUpdates.value*1000);
    let pct = Math.min(1, msElapsedSinceLastUpdate.value / msBetweenUpdate ) * 100;
    return "linear-gradient(90deg, var(--update-progress-color) 0%, var(--update-progress-color) "+pct+"%, var(--update-progress-background-color) "+pct+"%, var(--update-progress-background-color) 100%)";
  });
  
  
</script>

<style scoped>
  ul {
    margin: 0;
    padding: 0;
  }
  
  .achievement-tracker--body {
    padding: 0.25em 0.5em;
  }
  
  .achievement-tracker > header {
    background: #F0F0F0;
        display: grid;
    grid-template-columns: minmax(0, 1fr) minmax(200px, 1fr);
    grid-template-rows: min-content min-content;
    grid-template-areas: "updateprogress updateprogress"
                         "filter categories";
  }
  
  .achievement-tracker > header .update-info {
    grid-area: updateprogress;
    font-size: 0.8rem;
    padding: 0.25em 1em;
    position: relative;
  }
  .last-update {
    position: relative;
    top: 2px;
  }
  .achievement-tracker > header .update-info button {
    padding: 0;
    margin-left: 0.5em;
  }
  .achievement-tracker > header .category-assign {
    grid-area: categories;
    text-align: right;
    padding: 0.25em;
  }
  .achievement-tracker > header .filter {
    grid-area: filter;
  }
  .achievement-tracker > header .filter input {
    width: calc(100% - 1em);
    margin: 0.25em;
    font-size: 1.125em;
    padding: 0.25em;
    border: none;
  }  
  .achievement-tracker > header .update-info button {
    border: none;
    background: none;
  }
  button.action-turbo--active i {
    border: none;
    color: #FFF;
    background-color: var(--color-primary);
    animation-name: color;
    animation-duration: 5s;
    animation-iteration-count: infinite;
  }
   
  
  .achievement-tracker details > summary {
    cursor: pointer;
    font-size: 1.4em;
    font-family: var(--font-serif);
    font-weight: 200;
    margin-bottom: 0em;
    margin-top: 0;
  }
  
  .achievement-tracker details[open=""] > summary {
    margin-bottom: 0.25em;
  }
  
  .achievement-tracker details > summary > small {
    font-size: 0.6em;
    color: #FFFFFF;
    background: #7eb0d6;
    font-family: var(--font-sans);
    font-weight: bold;
    padding: 0px 0.5em 1px 0.52em;
    display: inline-block;
    border-radius: 2em;
    position: relative;
    top: -2px;
  }
  .achievement-tracker details > summary > small[data-count='0'] {
    background-color: #DDD;
  }
  .achievement-tracker details > summary i {
    font-size: 0.9rem;
    opacity: 0.4;
    cursor: pointer;
  }
  .achievement-tracker details > summary i:hover {
    opacity: 1;
  }
  .add-category {
    background: none;
    border: none;
    color: #999;
    margin: 0;
    padding: 0;
    display: block;
    text-align: center;
    width: 100%;
    cursor: pointer;
  }
  .add-category:hover {
     color: #222; 
  }
  
</style>