<template>
  <li class="achievement" :class="{'achievement--details-open': detailsPanelIsOpen, 'achievement--is-complete': isComplete}" :style="{ background: backgroundStyle }">
    
    <div class="achievement--details-toggle" @click="toggleDetailsPanel">
      <i class="fa fa-caret-down" aria-hidden="true" v-if="detailsPanelIsOpen"></i>
      <i class="fa fa-caret-right" aria-hidden="true" v-if="!detailsPanelIsOpen"></i>
    </div>

    <div class="achievement--category-icon">
      <img v-if="achievement.category" :src="achievement.category.icon" :alt="achievement.category.name" />
      <span class="achievement--progress-count"><strong>{{achievement.progress}}<wbr><span v-if="achievement.max">/{{achievement.max}}</span></strong></span>
    </div>
    
    <strong class="achievement--title">
      <i aria-label="Hidden" class="fa fa-eye hidden" v-if="isHidden" title="Hidden"></i>
      {{achievement.name}} <small>{{achievement.id}}</small>
    </strong>
    <span class="achievement--grouping" v-if="achievement.category">
      <span class="group">{{achievement.category.group.name}}</span> <i class="fa fa-chevron-right" aria-hidden="true"></i>
      <span class="category">{{achievement.category.name}}</span>
    </span>
    <div class="achievement--info">
      <p v-if="achievement.locked_text" class="achievement--locked-text">{{achievement.locked_text}}</p>
      <p v-if="achievement.description" class="achievement--description">{{achievement.description}}</p>
      <p v-if="achievement.requirement || achievement.progress" class="achievement--requirement">{{achievement.requirement}} <span v-if="achievement.progress && !achievement.completed && achievement.max" class="achievement--progress-count">(<strong>{{achievement.progress}}<span v-if="achievement.max">/{{achievement.max}}</span></strong>)</span></p>
    </div>

    <div class="achievement--details">

      <caption v-if="achievement.bits">Requirements</caption>
      <table class="achievement-bits" v-if="achievement.bits">
        <tr v-for="bit of bitsTable" class="achievement--bit" :class="{'achievement--bit-is-completed':bit.completed}">
          <td class="achievement--bit-completed">
            <i v-if="bit.completed" class="fa fa-check" aria-hidden="true"></i>
            <i v-if="!bit.completed" class="fa fa-circle" aria-hidden="true"></i>
          </td>
          <td class="achievement--bit--text" v-if="bit.type=='Text'">{{bit.text}}</td>
          <td class="achievement--bit--item" v-if="bit.type=='Item' && bit.item"><img class="gameicon" :class="'gameicon--rarity-'+bit.item.rarity" :src="bit.item.icon"> <a :href="'https://wiki.guildwars2.com/wiki/'+bit.item.name" target="_blank">{{bit.item.name}}</a></td>
          <td class="achievement--bit--skin" v-if="bit.type=='Skin' && bit.skin"><img class="gameicon" :src="bit.skin.icon"> <a :href="'https://wiki.guildwars2.com/wiki/'+bit.skin.name" target="_blank">{{bit.skin.name}}</a></td>
          <td class="achievement--bit--minipet" v-if="bit.type=='Minipet' && bit.minipet"><img class="gameicon" :class="'gameicon--rarity-'+bit.minipet.rarity" :src="bit.minipet.icon"> <a :href="'https://wiki.guildwars2.com/wiki/'+bit.minipet.name" target="_blank">{{bit.minipet.name}}</a></td>
        </tr>
      </table>
      
      <caption v-if="achievement.tiers">Tiers</caption>
      <table class="achievement-tiers" v-if="achievement.tiers">
        <tr v-for="tier of tiersTable" class="achievement--tier" :class="{'achievement--tier-is-completed':tier.completed}">
          <td class="achievement--tier-completed">
            <i v-if="tier.completed" class="fa fa-check" aria-hidden="true"></i>
            <i v-if="!tier.completed" class="fa fa-circle" aria-hidden="true"></i>
          </td>
          <th class="achievement--tier-tier">{{tier.tier}}</th>
          <td class="achievement--tier-count">{{tier.count}}</td>
          <td class="achievement--tier-points"><span class="ap" v-if="tier.points"><img src="https://cdn.glitch.global/b91f7cf5-ad74-42de-8335-8724d4534fdb/AP.png?v=1645545374258" class="ap"> {{tier.points}}</span></td>
          <td class="achievement--tier-reward" v-if="tier.reward"><span class="quantity" v-if="tier.reward.amount > 1">{{tier.reward.amount}}× </span><a :href="'https://wiki.guildwars2.com/wiki/'+tier.reward.name" target="_blank">{{tier.reward.name}}</a></td>
        </tr>
      </table>
  
      <caption v-if="achievement.rewards">Completion rewards</caption>
      <table class="achievement-rewards" v-if="achievement.rewards">
        <tr v-for="reward of achievement.rewards" class="achievement--tier" :class="{'achievement--tier-is-completed':isComplete}">
          <td class="achievement--tier-completed">
            <i v-if="isComplete" class="fa fa-check" aria-hidden="true"></i>
            <i v-if="!isComplete" class="fa fa-circle" aria-hidden="true"></i>
          </td>
          <td class="achievement--tier-reward" :class="'achievement--tier-reward--type-' + reward.type">
            
            <img v-if="reward.type=='Title'" class="title" src="https://cdn.glitch.global/b91f7cf5-ad74-42de-8335-8724d4534fdb/Title.png?v=1645562865344">
            <span>{{reward.title}}</span>

            <span class="quantity" v-if="!reward.type=='Coins' && reward.count > 1">{{reward.amount}}× </span>
            <img v-if="reward.item" class="gameicon" :class="'gameicon--rarity-'+reward.item.rarity" :src="reward.item.icon">
            <a v-if="reward.item" :href="'https://wiki.guildwars2.com/wiki/'+reward.item.name" target="_blank">{{reward.item.name}}</a>

            <strong v-if="reward.type=='Coins'" class="coins">{{reward.count}} coins</strong>

            <MasteryIcon v-if="reward.type == 'Mastery'" :type="reward.region"></MasteryIcon>
          </td>    
        </tr>
      </table>
    </div>
    
    <div class="achievement--buttons">
      <a :href="'https://wiki.guildwars2.com/wiki/'+achievement.name" target="_blank"><i aria-label="Wiki" class="button fa fa-wikipedia-w" title="Open on GW2 Wiki"></i></a>
      <i aria-label="Pin" class="button fa fa-thumb-tack action-pin" v-if="!isPinned" title="Pin" @click="pin"></i>
      <i aria-label="Unpin" class="button fa fa-thumb-tack action-unpin" v-if="isPinned" title="Unpin" @click="unpin"></i>

      <i aria-label="Ignore" class="button fa fa-eye-slash action-ignore" v-if="!isIgnored" title="Ignore" @click="ignore"></i>
      <i aria-label="Remove from ignored" class="button fa fa-eye-slash action-unignore" v-if="isIgnored" title="Remove from ignored" @click="unignore"></i>

      <input v-if="!userCategory" type="checkbox" v-model="achievement.isChecked">
      <i aria-label="Remove from category" class="button fa fa-minus-circle action-remove-from-category" v-if="userCategory" title="Remove from category" @click="removeFromCategory"></i>
    </div>
  </li>
</template>

<script setup>
import { defineEmit, ref, defineProps, toRefs, computed } from 'vue';
import MasteryIcon from './MasteryIcon.vue';
  
const props = defineProps({
  achievement: Object,
  userCategory: String
});
  
const emit = defineEmit(['pinned', 'unpinned', 'ignore', 'unignore', 'categoryRemoved']);

const { achievement } = toRefs(props);
const detailsPanel = ref(null);
const detailsPanelIsOpen = ref(false);
  
const toggleDetailsPanel = () => {
  detailsPanelIsOpen.value = !detailsPanelIsOpen.value; 
}

const pin = () => {
  achievement.value.pinned = true;
  emit('pinned',achievement.value.id);
}

const unpin = () => {
  achievement.value.pinned = false;
  emit('unpinned',achievement.value.id);
}

const isPinned = computed(() => {
  return !!(achievement.value.pinned);
});
  
const removeFromCategory = () => {
  achievement.value.categories = achievement.value.categories.filter( a => a != props.userCategory );
  emit('categoryRemoved', achievement.value.id, props.userCategory );
}

const ignore = () => {
  achievement.value.ignored = true;
  achievement.value.pinned = false;
  emit('ignore',achievement.value.id);
}

const unignore = () => {
  achievement.value.ignored = false;
  emit('unignore',achievement.value.id);
}
const isIgnored = computed(() => {
  return !!(achievement.value.ignored);
});
  
const backgroundStyle = computed(() => {
  let pct = Math.round(achievement.value.progressPct);
  return "linear-gradient(90deg, var(--achievement-complete-background-color) 0%, var(--achievement-complete-background-color) "+pct+"%, white "+pct+"%, white 100%)";
});
  
const isComplete = computed(() => {
  //return achievement.value.name == "Glory to the Blood Legion";
  return achievement.value.completed;
});

const isHidden = computed(() => {
  return achievement.value.flags && achievement.value.flags.includes('Hidden')
});

const requiresUnlock = computed(() => {
  return achievement.value.flags && achievement.value.flags.indexOf('RequiresUnlock') > -1
});
  
const isDaily = computed(() => {
  return achievement.value.flags && achievement.value.flags.indexOf('Daily') > -1
});

const isWeekly = computed(() => {
  return achievement.value.flags && achievement.value.flags.indexOf('Daily') > -1
});

const isMonthly = computed(() => {
  return achievement.value.flags && achievement.value.flags.indexOf('Daily') > -1
});
  
const hasMasteryPointReward = computed(() => {
  if( achievement.rewards && achievement.rewards.some( r => r.type == 'Mastery') ) return true;
  if( achievement.tierrewards && achievement.tierrewards.some( r => (r.reward && r.reward.name == 'Mastery Point'))) return true;
  return false;
});
  
const tiersTable = computed(() => {
  let tiers = [];
  let rewards = achievement.value.tierrewards;
  if( achievement.value.tiers ) {
    for( let tierIdx in achievement.value.tiers ) {
      let tier = achievement.value.tiers[tierIdx];
      tiers[tierIdx] = {
        tier: tierIdx,
        count: tier.count,
        points: tier.points,
        reward: (rewards && rewards[tierIdx]) ? rewards[tierIdx] : null,
        completed: achievement.value.progress >= tier.count
      }
    }
  }
  return tiers;
});
  
const bitsTable = computed(() => {
  return achievement.value.bits;    
});
  
</script>

<style scoped>
  li.achievement {
    margin: 0 0 0.5rem;
    padding: 0;
    box-shadow:
      0 0px 1px rgba(0, 0, 0, 0.3),
      0 3px 3px rgba(0, 0, 0, 0.03);

    display: grid;
    grid-template-columns: 12px min-content minmax(0, 1fr) min-content;
    grid-template-rows: min-content min-content minmax(0, 1fr) min-content;
    grid-template-areas: "detailstoggle icon category buttons"
                         "detailstoggle icon title buttons"
                         "detailstoggle icon info buttons"
                         "detailstoggle icon details buttons";
  }
  .gameicon {
    height: 20px;
    vertical-align: top;
  }
  li.achievement.achievement--is-complete {
    border: 1px solid var(--achievement-complete-color);
  }
  li.achievement:hover {
    box-shadow:
      0 0px 1px rgba(0, 0, 0, 0.7),
      0 3px 3px rgba(0, 0, 0, 0.07);
  }
  small {
    opacity: 0;
  }
  li.achievement:hover small {
    opacity: 0.3;
  }
  .achievement--details-toggle {
    grid-area: detailstoggle;
    background: rgba(0,0,0,0.03);
    padding: 0.25em;
    vertical-align: middle;
    display: grid;
    justify-content: center;
    align-content: center;
    color: rgba(0,0,0,0.2);
    cursor: pointer;
  }
  .achievement--details-toggle:hover {
    background: rgba(0,0,0,0.07);
    color: rgba(0,0,0,0.4)
  }
  .achievement--category-icon {
    grid-area: icon;
    width: 40px;
    margin: 0.25em 0.5em;
  }
  .achievement--category-icon img {
    width: 40px;
  }
  .achievement--category-icon > span {
    font-size: 0.7em;
    display: block;
    text-align: center;
    margin-top: -5px;
    opacity: 0.7;
  }
  .achievement--title {
    grid-area: title;
    font-family: var(--font-serif);
    font-weight: 200;
    margin-bottom: 0em;
  }
  li.achievement.achievement--is-complete .achievement--title {
    color: var(--achievement-complete-color);
  }
  .achievement--title .hidden {
    opacity:0.5;
  }
  .achievement--grouping {
    grid-area: category;
    opacity: 0.5;
    font-size: 0.75rem;
    line-height: 1;
    margin-top: 0.5rem;
  }
  .achievement--grouping i {
    font-size: 0.7em;
    margin: 0 4px 0 2px;
    display: inline-block;
    opacity: 0.3;
    position: relative;
    top: -1px;
  }
  .achievement--info {
    grid-area: info;
    max-height: 3.6em;
    mask-image: linear-gradient(to bottom, black 58%, transparent 100%);
  }
  .achievement--info p {
    margin: 0;
    padding: 0;
  }
  .achievement--info p:last-child {
    margin: 0 0 .75em;
  }
  .achievement--details-open .achievement--info {
    max-height: none;
    mask-image: none;
  }
  .achievement--description,
  .achievement--requirement, 
  .achievement--locked-text {
    font-size: 0.7rem;
    color: var(--color-text);
  }
  .achievement--description {
    font-style: italic;
  }
  .achievement--details {
    grid-area: details;
    height: 0;
    overflow: hidden;
    font-size: 0.8rem;
  }
  .achievement--details-open .achievement--details {
    display: block;
    height: auto;
    overflow: auto;
  }
  .achievement--details caption {
    display: block;
    text-align: left;
    margin-bottom: 0.5em;
    color: var(--color-text-light);
  }
  .achievement--details li {
    margin: 0;
    padding: 0;
  }
  .achievement--progress-count {
    margin: 0;
    
  }
  .achievement-tiers,.achievement-rewards {
    margin-bottom: 1em;
  }
  .achievement-tiers tr, .achievement-rewards tr {
    opacity: 1;
  }
  .achievement-tiers tr.achievement--tier-is-completed, .achievement-rewards tr.achievement--tier-is-completed {
    opacity: 0.7;
  }
  .achievement--tier-completed, .achievement--bit-is-completed {
    color: #498727
  }
  .fa-circle {
    color: rgba(0,0,0,0.05);
  }
  .achievement--tier-tier {
    font-weight: normal;
    color: var(--color-text-light);
    text-align: right;
  }
  .achievement--tier-count {
    font-weight: bold;
    padding: 0 1em; 
    text-align: right;
    opacity: 1;
  }
  .achievement-tiers tr.achievement--tier-is-completed .achievement--tier-count {
    opacity: 0.7;
  }
  .achievement--tier-reward {
    padding-left: 1em;
    vertical-align: middle;
    line-height: 18px;
  }
  .achievement--tier-reward .gameicon {
    width: 20px;
    margin-right: 0.5em;
  }
  .achievement--tier-reward strong {
    vertical-align: middle;
    line-height: 18px;
  }
  .achievement--tier-reward img {
    vertical-align: top;
  }
  .achievement--bit img {
    filter: saturate(0);
  }
  .achievement--bit.achievement--bit-is-completed img {
    filter: saturate(1);
  }
  .achievement--buttons {
    grid-area: buttons;
    border-left: 1px solid rgba(0,0,0,0.1);
    padding: 0.125em 0.25em 0.125em 0.25em;
    margin-left: 0.5em;
    opacity: 1;
    text-align: center;
    font-size: 0.75em;
  }
  li:hover .achievement--buttons {
    opacity: 1;
  }
  .achievement--buttons .button {

  }
  .achievement--buttons input {
    margin: 6px 0 7px;
    width: 14px;
  }
  .action-pin {
    color: var(--color-text-light);
  }
  .action-unpin {
    color: var(--color-primary);
  }
  .action-turbo {
    color: var(--color-text-light);
  }
  .action-turbo.action-turbo--active {
    color: var(--color-primary);
  }
  li + li {
    margin-top: 0.5rem;
  }

</style>