<script setup lang="ts">
import { computed } from 'vue'
import { storeToRefs } from 'pinia'
import { useGameStore } from '../stores/gameStore'
import gameConfig from '../config/gameConfig'
import { getRarityColor, getRarityLabel } from '../utils/gameUtils'

const gameStore = useGameStore()

const {
  showEventModal,
  currentEvent,
  currentResponse,
  eventResult,
  showResponse,
  showResult,
  isEventFlowActive,
  eventFlowPhase
} = storeToRefs(gameStore)

const characterConfig = computed(() => {
  if (!currentEvent.value?.characterId) return null
  return gameConfig.characters.find(c => c.id === currentEvent.value!.characterId)
})

const responseCharacterConfig = computed(() => {
  if (!currentResponse.value?.characterId) return null
  return gameConfig.characters.find(c => c.id === currentResponse.value!.characterId)
})

const sceneBackground = computed(() => {
  const style = currentEvent.value?.narrationStyle
  const scene = currentEvent.value?.backgroundScene
  const backgrounds: Record<string, string> = {
    library: 'linear-gradient(135deg, #fef3c7 0%, #fde68a 100%)',
    cafe: 'linear-gradient(135deg, #fed7aa 0%, #fdba74 100%)',
    street_rain: 'linear-gradient(135deg, #bfdbfe 0%, #93c5fd 100%)',
    cafe_night: 'linear-gradient(135deg, #1e3a5f 0%, #0f172a 100%)',
    park_night: 'linear-gradient(135deg, #1e1b4b 0%, #312e81 100%)',
    library_birthday: 'linear-gradient(135deg, #fce7f3 0%, #fbcfe8 100%)',
    school: 'linear-gradient(135deg, #e0e7ff 0%, #c7d2fe 100%)'
  }
  return backgrounds[scene || ''] || backgrounds[style === 'romantic' ? 'library' : 'cafe']
})

const emotionIcon = computed(() => {
  const emotion = currentResponse.value?.emotion
  const icons: Record<string, string> = {
    happy: '😊',
    sad: '😢',
    angry: '😠',
    shy: '😳',
    neutral: '😐',
    surprised: '😮'
  }
  return icons[emotion || 'neutral']
})

const emotionLabel = computed(() => {
  const emotion = currentResponse.value?.emotion
  const labels: Record<string, string> = {
    happy: '开心',
    sad: '难过',
    angry: '生气',
    shy: '害羞',
    neutral: '平静',
    surprised: '惊讶'
  }
  return labels[emotion || 'neutral']
})

const styleClass = computed(() => {
  const style = currentEvent.value?.narrationStyle || 'normal'
  return `narration-${style}`
})

function handleChoice(choiceId: string) {
  if (eventFlowPhase.value !== 'choosing') return
  const choice = currentEvent.value?.choices.find(c => c.id === choiceId)
  if (choice) {
    gameStore.handleEventChoice(choice)
  }
}

function formatEffect(effect: any): string {
  let result = ''
  if (effect.affinityChange !== undefined) {
    const char = gameConfig.characters.find(c => c.id === effect.characterId)
    const name = char?.name || effect.characterId
    const sign = effect.affinityChange > 0 ? '+' : ''
    result += `${name} 好感 ${sign}${effect.affinityChange}`
  }
  if (effect.moodChange !== undefined) {
    if (result) result += '，'
    const sign = effect.moodChange > 0 ? '+' : ''
    result += `心情 ${sign}${effect.moodChange}`
  }
  return result
}

const showChoosing = computed(() => !showResponse.value && !showResult.value)
</script>

<template>
  <Teleport to="body">
    <div v-if="isEventFlowActive" class="modal-overlay">
      <div class="modal-content event-modal" :class="styleClass">
        <div class="scene-background" :style="{ background: sceneBackground }"></div>
        
        <div class="event-content">
          <Transition name="fade" mode="out-in">
            <div v-if="showChoosing && currentEvent" key="choosing" class="event-main">
              <div class="event-header">
                <div v-if="characterConfig" class="event-character">
                  <div class="char-avatar-large">
                    <span class="char-avatar-img">{{ characterConfig.avatar }}</span>
                    <div class="char-avatar-glow"></div>
                  </div>
                  <div class="char-info">
                    <span class="char-name-large">{{ characterConfig.name }}</span>
                    <span class="char-personality">{{ characterConfig.personality }}</span>
                  </div>
                </div>
                <span class="event-tag" :class="styleClass">{{ currentEvent.narrationStyle === 'romantic' ? '浪漫' : currentEvent.narrationStyle === 'dramatic' ? '剧情' : currentEvent.narrationStyle === 'mysterious' ? '神秘' : '事件' }}</span>
              </div>

              <h2 class="event-title">{{ currentEvent.title }}</h2>
              
              <div class="event-description-box">
                <p class="event-description">{{ currentEvent.description }}</p>
              </div>

              <div class="event-choices">
                <button
                  v-for="choice in currentEvent.choices"
                  :key="choice.id"
                  class="choice-btn"
                  @click="handleChoice(choice.id)"
                >
                  <span class="choice-text">{{ choice.text }}</span>
                  <div class="choice-effects">
                    <span 
                      v-for="(effect, idx) in choice.effects" 
                      :key="idx"
                      class="effect-tag"
                      :class="{ positive: effect.affinityChange > 0 || effect.moodChange > 0, negative: effect.affinityChange < 0 || effect.moodChange < 0 }"
                    >
                      {{ formatEffect(effect) }}
                    </span>
                    <span v-if="choice.resourceChange" class="effect-tag" :class="{ positive: choice.resourceChange > 0, negative: choice.resourceChange < 0 }">
                      代币 {{ choice.resourceChange > 0 ? '+' : '' }}{{ choice.resourceChange }}
                    </span>
                    <span v-if="choice.unlockCharacterId" class="effect-tag special">
                      ✨ 解锁角色
                    </span>
                    <span v-if="choice.addCardId" class="effect-tag special">
                      🎴 获得卡牌
                    </span>
                  </div>
                </button>
              </div>
            </div>

            <div v-else-if="showResponse && currentResponse" key="response" class="response-section">
              <div class="response-character">
                <div class="response-avatar" :class="currentResponse.emotion">
                  <span class="response-avatar-img">{{ responseCharacterConfig?.avatar }}</span>
                  <div class="response-emotion-badge">{{ emotionIcon }}</div>
                </div>
                <div class="response-info">
                  <span class="response-name">{{ responseCharacterConfig?.name }}</span>
                  <span class="response-emotion-label">{{ emotionLabel }}</span>
                </div>
              </div>
              <div class="response-bubble">
                <p class="response-text">{{ currentResponse.text }}</p>
                <div class="bubble-tail"></div>
              </div>
              <div class="progress-hint">
                <div class="progress-dots">
                  <span></span><span></span><span></span>
                </div>
              </div>
            </div>

            <div v-else-if="showResult && eventResult" key="result" class="result-section">
              <h3 class="result-title">✨ 结果</h3>
              
              <div v-if="eventResult.unlockedCard" class="result-card-unlock">
                <div class="card-reveal">
                  <div class="card-inner" :style="{ borderColor: getRarityColor(eventResult.unlockedCard.rarity) }">
                    <span class="card-image">{{ eventResult.unlockedCard.image }}</span>
                    <span class="card-rarity" :style="{ color: getRarityColor(eventResult.unlockedCard.rarity) }">{{ getRarityLabel(eventResult.unlockedCard.rarity) }}</span>
                    <span class="card-name">{{ eventResult.unlockedCard.name }}</span>
                  </div>
                </div>
                <p class="card-unlock-text">🎴 获得新卡牌！</p>
              </div>

              <div v-if="eventResult.unlockedCharacter" class="result-character-unlock">
                <p class="unlock-text">✨ 解锁新角色：{{ eventResult.unlockedCharacter }}！</p>
              </div>

              <div class="result-changes">
                <div v-if="eventResult.affinityChanges?.length" class="changes-group">
                  <div 
                    v-for="(change, idx) in eventResult.affinityChanges" 
                    :key="'aff-' + idx"
                    class="change-item affinity"
                    :class="{ positive: change.change > 0, negative: change.change < 0 }"
                    :style="{ animationDelay: idx * 100 + 'ms' }"
                  >
                    <span class="change-icon">💕</span>
                    <span class="change-name">{{ change.name }}</span>
                    <span class="change-value">{{ change.change > 0 ? '+' : '' }}{{ change.change }}</span>
                    <span class="change-label">好感</span>
                  </div>
                </div>

                <div v-if="eventResult.moodChanges?.length" class="changes-group">
                  <div 
                    v-for="(change, idx) in eventResult.moodChanges" 
                    :key="'mood-' + idx"
                    class="change-item mood"
                    :class="{ positive: change.change > 0, negative: change.change < 0 }"
                    :style="{ animationDelay: (eventResult.affinityChanges?.length || 0) * 100 + idx * 100 + 'ms' }"
                  >
                    <span class="change-icon">😊</span>
                    <span class="change-name">{{ change.name }}</span>
                    <span class="change-value">{{ change.change > 0 ? '+' : '' }}{{ change.change }}</span>
                    <span class="change-label">心情</span>
                  </div>
                </div>

                <div 
                  v-if="eventResult.resourceChange !== undefined" 
                  class="changes-group"
                >
                  <div 
                    class="change-item resource"
                    :class="{ positive: eventResult.resourceChange > 0, negative: eventResult.resourceChange < 0 }"
                    :style="{ animationDelay: ((eventResult.affinityChanges?.length || 0) + (eventResult.moodChanges?.length || 0)) * 100 + 'ms' }"
                  >
                    <span class="change-icon">💰</span>
                    <span class="change-name">代币</span>
                    <span class="change-value">{{ eventResult.resourceChange > 0 ? '+' : '' }}{{ eventResult.resourceChange }}</span>
                  </div>
                </div>
              </div>
            </div>
          </Transition>
        </div>
      </div>
    </div>
  </Teleport>
</template>

<style scoped>
.event-modal {
  padding: 0;
  overflow: hidden;
  position: relative;
  background: var(--bg-primary);
}

.scene-background {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  height: 200px;
  opacity: 0.3;
  z-index: 0;
}

.event-content {
  position: relative;
  z-index: 1;
  padding: 32px;
  min-height: 300px;
  display: flex;
  flex-direction: column;
}

.event-main {
  display: flex;
  flex-direction: column;
}

.fade-enter-active,
.fade-leave-active {
  transition: all 0.35s cubic-bezier(0.4, 0, 0.2, 1);
}

.fade-enter-from {
  opacity: 0;
  transform: translateY(15px);
}

.fade-leave-to {
  opacity: 0;
  transform: translateY(-15px);
}

.event-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 24px;
}

.event-character {
  display: flex;
  align-items: center;
  gap: 16px;
}

.char-avatar-large {
  position: relative;
  width: 80px;
  height: 80px;
}

.char-avatar-img {
  width: 80px;
  height: 80px;
  background: var(--bg-tertiary);
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 40px;
  position: relative;
  z-index: 2;
  border: 3px solid var(--accent-primary);
  animation: avatarPulse 2s ease-in-out infinite;
}

@keyframes avatarPulse {
  0%, 100% { transform: scale(1); }
  50% { transform: scale(1.05); }
}

.char-avatar-glow {
  position: absolute;
  top: -5px;
  left: -5px;
  right: -5px;
  bottom: -5px;
  background: linear-gradient(45deg, var(--accent-primary), #60a5fa);
  border-radius: 50%;
  opacity: 0.5;
  filter: blur(10px);
  z-index: 1;
  animation: glowPulse 3s ease-in-out infinite;
}

@keyframes glowPulse {
  0%, 100% { opacity: 0.3; transform: scale(1); }
  50% { opacity: 0.6; transform: scale(1.1); }
}

.char-info {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.char-name-large {
  font-weight: 700;
  font-size: 20px;
  color: var(--text-primary);
}

.char-personality {
  font-size: 13px;
  color: var(--text-secondary);
}

.event-tag {
  font-size: 12px;
  padding: 6px 16px;
  background: var(--accent-primary);
  color: white;
  border-radius: 9999px;
  font-weight: 600;
}

.event-tag.narration-romantic {
  background: linear-gradient(135deg, #ec4899, #f472b6);
}

.event-tag.narration-dramatic {
  background: linear-gradient(135deg, #8b5cf6, #a78bfa);
}

.event-tag.narration-mysterious {
  background: linear-gradient(135deg, #6366f1, #818cf8);
}

.event-title {
  font-size: 28px;
  font-weight: 800;
  margin-bottom: 20px;
  color: var(--text-primary);
  text-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.event-description-box {
  background: var(--bg-tertiary);
  border-radius: var(--radius-lg);
  padding: 24px;
  margin-bottom: 28px;
  position: relative;
  border-left: 4px solid var(--accent-primary);
}

.narration-romantic .event-description-box {
  border-left-color: #ec4899;
  background: linear-gradient(135deg, rgba(236, 72, 153, 0.1), var(--bg-tertiary));
}

.narration-dramatic .event-description-box {
  border-left-color: #8b5cf6;
  background: linear-gradient(135deg, rgba(139, 92, 246, 0.1), var(--bg-tertiary));
}

.narration-mysterious .event-description-box {
  border-left-color: #6366f1;
  background: linear-gradient(135deg, rgba(99, 102, 241, 0.1), var(--bg-tertiary));
}

.event-description {
  font-size: 16px;
  line-height: 1.9;
  color: var(--text-secondary);
  margin: 0;
}

.event-choices {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.choice-btn {
  width: 100%;
  padding: 20px 24px;
  text-align: left;
  background: var(--bg-tertiary);
  border: 2px solid transparent;
  border-radius: var(--radius-lg);
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  cursor: pointer;
  position: relative;
  overflow: hidden;
}

.choice-btn::before {
  content: '';
  position: absolute;
  top: 0;
  left: -100%;
  width: 100%;
  height: 100%;
  background: linear-gradient(90deg, transparent, rgba(255, 255, 255, 0.1), transparent);
  transition: left 0.5s;
}

.choice-btn:hover::before {
  left: 100%;
}

.choice-btn:hover {
  border-color: var(--accent-primary);
  background: var(--accent-light);
  transform: translateX(8px);
  box-shadow: 0 8px 25px rgba(0, 0, 0, 0.15);
}

.choice-text {
  display: block;
  font-size: 16px;
  font-weight: 600;
  margin-bottom: 10px;
  color: var(--text-primary);
}

.choice-effects {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}

.effect-tag {
  font-size: 12px;
  padding: 4px 10px;
  border-radius: 6px;
  background: var(--bg-secondary);
  color: var(--text-secondary);
  font-weight: 500;
}

.effect-tag.positive {
  background: #dcfce7;
  color: #166534;
}

[data-theme='dark'] .effect-tag.positive {
  background: #14532d;
  color: #86efac;
}

.effect-tag.negative {
  background: #fee2e2;
  color: #991b1b;
}

[data-theme='dark'] .effect-tag.negative {
  background: #7f1d1d;
  color: #fca5a5;
}

.effect-tag.special {
  background: linear-gradient(135deg, #fef3c7, #fde68a);
  color: #92400e;
}

[data-theme='dark'] .effect-tag.special {
  background: linear-gradient(135deg, #78350f, #92400e);
  color: #fef3c7;
}

.response-section {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 24px;
  padding: 10px 0;
}

.response-character {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 12px;
}

.response-avatar {
  position: relative;
  width: 120px;
  height: 120px;
  animation: bounce 1s ease-in-out infinite;
}

.response-avatar.happy { animation: bounceHappy 0.6s ease-in-out infinite; }
.response-avatar.sad { animation: shakeSad 0.8s ease-in-out infinite; }
.response-avatar.angry { animation: shakeAngry 0.5s ease-in-out infinite; }
.response-avatar.shy { animation: shySway 1.2s ease-in-out infinite; }
.response-avatar.surprised { animation: surprisedAnim 0.5s ease-in-out infinite; }

@keyframes bounce {
  0%, 100% { transform: translateY(0); }
  50% { transform: translateY(-8px); }
}

@keyframes bounceHappy {
  0%, 100% { transform: translateY(0) rotate(-3deg); }
  25% { transform: translateY(-12px) rotate(3deg); }
  50% { transform: translateY(-5px) rotate(-3deg); }
  75% { transform: translateY(-12px) rotate(3deg); }
}

@keyframes shakeSad {
  0%, 100% { transform: translateY(3px); }
  25% { transform: translateX(-3px) rotate(-2deg); }
  75% { transform: translateX(3px) rotate(2deg); }
}

@keyframes shakeAngry {
  0%, 100% { transform: translateX(0); }
  20% { transform: translateX(-5px) rotate(-2deg); }
  40% { transform: translateX(5px) rotate(2deg); }
  60% { transform: translateX(-4px) rotate(-1deg); }
  80% { transform: translateX(4px) rotate(1deg); }
}

@keyframes shySway {
  0%, 100% { transform: translateX(0) rotate(0); }
  50% { transform: translateX(-5px) rotate(-3deg); }
}

@keyframes surprisedAnim {
  0%, 100% { transform: scale(1); }
  25% { transform: scale(1.1); }
  75% { transform: scale(0.95); }
}

.response-avatar-img {
  width: 120px;
  height: 120px;
  background: var(--bg-tertiary);
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 60px;
  border: 4px solid var(--accent-primary);
  position: relative;
  z-index: 2;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.15);
}

.response-emotion-badge {
  position: absolute;
  bottom: -5px;
  right: -5px;
  width: 48px;
  height: 48px;
  background: white;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 24px;
  z-index: 3;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.2);
  animation: badgePop 0.5s ease-out;
}

@keyframes badgePop {
  0% { transform: scale(0); }
  70% { transform: scale(1.3); }
  100% { transform: scale(1); }
}

.response-info {
  text-align: center;
}

.response-name {
  display: block;
  font-weight: 700;
  font-size: 22px;
  color: var(--text-primary);
  margin-bottom: 4px;
}

.response-emotion-label {
  font-size: 14px;
  padding: 4px 12px;
  background: var(--accent-light);
  color: var(--accent-primary);
  border-radius: 9999px;
  font-weight: 600;
}

.response-bubble {
  position: relative;
  background: var(--bg-tertiary);
  padding: 24px 32px;
  border-radius: 24px;
  max-width: 100%;
  border: 2px solid var(--accent-primary);
}

.bubble-tail {
  position: absolute;
  top: -12px;
  left: 50%;
  width: 24px;
  height: 24px;
  background: var(--bg-tertiary);
  border-left: 2px solid var(--accent-primary);
  border-top: 2px solid var(--accent-primary);
  transform: translateX(-50%) rotate(45deg);
}

.response-text {
  font-size: 17px;
  line-height: 1.8;
  color: var(--text-primary);
  margin: 0;
  text-align: center;
}

.progress-hint {
  display: flex;
  align-items: center;
  justify-content: center;
  padding-top: 8px;
}

.progress-dots {
  display: flex;
  gap: 6px;
}

.progress-dots span {
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background: var(--accent-primary);
  opacity: 0.3;
  animation: dotBounce 1.4s ease-in-out infinite;
}

.progress-dots span:nth-child(2) {
  animation-delay: 0.2s;
}

.progress-dots span:nth-child(3) {
  animation-delay: 0.4s;
}

@keyframes dotBounce {
  0%, 80%, 100% { transform: scale(0.6); opacity: 0.3; }
  40% { transform: scale(1); opacity: 1; }
}

.result-section {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 24px;
  padding: 10px 0;
  width: 100%;
}

.result-title {
  font-size: 24px;
  font-weight: 800;
  color: var(--text-primary);
  margin: 0;
}

.result-card-unlock {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 16px;
  animation: cardReveal 1s ease-out;
}

@keyframes cardReveal {
  0% { opacity: 0; transform: perspective(1000px) rotateY(-90deg); }
  100% { opacity: 1; transform: perspective(1000px) rotateY(0deg); }
}

.card-reveal {
  animation: cardFloat 3s ease-in-out infinite;
}

@keyframes cardFloat {
  0%, 100% { transform: translateY(0); }
  50% { transform: translateY(-10px); }
}

.card-inner {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 32px;
  background: var(--bg-tertiary);
  border-radius: 16px;
  border: 3px solid;
  min-width: 200px;
  box-shadow: 0 10px 40px rgba(0, 0, 0, 0.2);
}

.card-image {
  font-size: 80px;
  margin-bottom: 12px;
  animation: sparkle 2s ease-in-out infinite;
}

@keyframes sparkle {
  0%, 100% { filter: brightness(1); }
  50% { filter: brightness(1.3) drop-shadow(0 0 20px currentColor); }
}

.card-rarity {
  font-size: 12px;
  font-weight: 700;
  margin-bottom: 8px;
  text-transform: uppercase;
  letter-spacing: 2px;
}

.card-name {
  font-size: 18px;
  font-weight: 700;
  color: var(--text-primary);
}

.card-unlock-text {
  font-size: 18px;
  font-weight: 700;
  color: var(--accent-primary);
  margin: 0;
  animation: pulse 1.5s ease-in-out infinite;
}

@keyframes pulse {
  0%, 100% { transform: scale(1); }
  50% { transform: scale(1.05); }
}

.result-character-unlock {
  background: linear-gradient(135deg, #fef3c7, #fde68a);
  padding: 20px 40px;
  border-radius: 16px;
  animation: glow 2s ease-in-out infinite;
}

[data-theme='dark'] .result-character-unlock {
  background: linear-gradient(135deg, #78350f, #92400e);
}

@keyframes glow {
  0%, 100% { box-shadow: 0 0 20px rgba(251, 191, 36, 0.4); }
  50% { box-shadow: 0 0 40px rgba(251, 191, 36, 0.8); }
}

.unlock-text {
  font-size: 18px;
  font-weight: 700;
  color: #92400e;
  margin: 0;
}

[data-theme='dark'] .unlock-text {
  color: #fef3c7;
}

.result-changes {
  width: 100%;
}

.changes-group {
  display: flex;
  flex-direction: column;
  gap: 12px;
  margin-bottom: 12px;
}

.changes-group:last-child {
  margin-bottom: 0;
}

.change-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 16px 20px;
  background: var(--bg-tertiary);
  border-radius: 12px;
  opacity: 0;
  animation: floatUp 0.5s cubic-bezier(0.34, 1.56, 0.64, 1) forwards;
}

@keyframes floatUp {
  0% { opacity: 0; transform: translateX(-30px); }
  100% { opacity: 1; transform: translateX(0); }
}

.change-item.positive {
  background: linear-gradient(90deg, rgba(34, 197, 94, 0.1), var(--bg-tertiary));
  border-left: 4px solid #22c55e;
}

.change-item.negative {
  background: linear-gradient(90deg, rgba(239, 68, 68, 0.1), var(--bg-tertiary));
  border-left: 4px solid #ef4444;
}

.change-icon {
  font-size: 24px;
}

.change-name {
  flex: 1;
  font-weight: 600;
  color: var(--text-primary);
  font-size: 15px;
}

.change-value {
  font-weight: 800;
  font-size: 20px;
  margin-right: 8px;
}

.change-item.positive .change-value {
  color: #22c55e;
  animation: valuePop 0.5s ease-out 0.3s both;
}

.change-item.negative .change-value {
  color: #ef4444;
  animation: valuePop 0.5s ease-out 0.3s both;
}

@keyframes valuePop {
  0% { transform: scale(0); }
  70% { transform: scale(1.3); }
  100% { transform: scale(1); }
}

.change-label {
  font-size: 13px;
  color: var(--text-secondary);
  font-weight: 500;
}
</style>