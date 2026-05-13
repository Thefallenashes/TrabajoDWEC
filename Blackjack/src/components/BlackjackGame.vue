<script setup>
import { ref, computed } from 'vue'

// --- Deck ---
const SUITS = ['♠', '♥', '♦', '♣']
const RANKS = ['A', '2', '3', '4', '5', '6', '7', '8', '9', '10', 'J', 'Q', 'K']

function createDeck() {
  const deck = []
  for (const suit of SUITS) {
    for (const rank of RANKS) {
      deck.push({ suit, rank })
    }
  }
  return shuffle(deck)
}

function shuffle(deck) {
  const d = [...deck]
  for (let i = d.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1))
    ;[d[i], d[j]] = [d[j], d[i]]
  }
  return d
}

function cardValue(rank) {
  if (['J', 'Q', 'K'].includes(rank)) return 10
  if (rank === 'A') return 11
  return parseInt(rank)
}

function handScore(hand) {
  let score = hand.reduce((sum, c) => sum + cardValue(c.rank), 0)
  let aces = hand.filter(c => c.rank === 'A').length
  while (score > 21 && aces > 0) {
    score -= 10
    aces--
  }
  return score
}

function isRedSuit(suit) {
  return suit === '♥' || suit === '♦'
}

// --- Game State ---
const deck = ref([])
const playerHand = ref([])
const dealerHand = ref([])
const dealerHideSecond = ref(true)
const gamePhase = ref('idle') // idle | playing | dealerTurn | result
const message = ref('')
const playerWins = ref(0)
const dealerWins = ref(0)
const draws = ref(0)

const playerScore = computed(() => handScore(playerHand.value))
const dealerScore = computed(() => {
  if (dealerHideSecond.value) {
    return cardValue(dealerHand.value[0]?.rank ?? '2')
  }
  return handScore(dealerHand.value)
})
const dealerVisibleScore = computed(() => dealerHideSecond.value ? '?' : dealerScore.value)

function deal() {
  const d = createDeck()
  const pH = [d.pop(), d.pop()]
  const dH = [d.pop(), d.pop()]
  deck.value = d
  playerHand.value = pH
  dealerHand.value = dH
  dealerHideSecond.value = true
  gamePhase.value = 'playing'
  message.value = ''

  // Check immediate blackjack
  if (handScore(pH) === 21) {
    endRound()
  }
}

function hit() {
  if (gamePhase.value !== 'playing') return
  playerHand.value.push(deck.value.pop())
  if (playerScore.value > 21) {
    endRound()
  }
}

function stand() {
  if (gamePhase.value !== 'playing') return
  gamePhase.value = 'dealerTurn'
  dealerHideSecond.value = false
  runDealer()
}

async function runDealer() {
  // Dealer hits on soft 16 or less, stands on 17+
  while (handScore(dealerHand.value) < 17) {
    dealerHand.value.push(deck.value.pop())
  }
  endRound()
}

function endRound() {
  dealerHideSecond.value = false
  gamePhase.value = 'result'

  const pScore = handScore(playerHand.value)
  const dScore = handScore(dealerHand.value)
  const pBlackjack = pScore === 21 && playerHand.value.length === 2
  const dBlackjack = dScore === 21 && dealerHand.value.length === 2

  if (pScore > 21) {
    message.value = '¡Te pasaste! La IA gana.'
    dealerWins.value++
  } else if (dScore > 21) {
    message.value = '¡La IA se pasó! Tú ganas.'
    playerWins.value++
  } else if (pBlackjack && !dBlackjack) {
    message.value = '¡BLACKJACK! Tú ganas.'
    playerWins.value++
  } else if (dBlackjack && !pBlackjack) {
    message.value = '¡Blackjack de la IA! La IA gana.'
    dealerWins.value++
  } else if (pScore > dScore) {
    message.value = '¡Ganaste!'
    playerWins.value++
  } else if (dScore > pScore) {
    message.value = 'La IA gana.'
    dealerWins.value++
  } else {
    message.value = '¡Empate!'
    draws.value++
  }
}

function resetStats() {
  playerWins.value = 0
  dealerWins.value = 0
  draws.value = 0
  gamePhase.value = 'idle'
  playerHand.value = []
  dealerHand.value = []
  message.value = ''
}
</script>

<template>
  <div class="bj-table">
    <h1 class="bj-title">♠ Blackjack ♠</h1>

    <!-- Scoreboard -->
    <div class="bj-scoreboard">
      <div class="bj-score-item">
        <span class="bj-score-label">Tú</span>
        <span class="bj-score-value">{{ playerWins }}</span>
      </div>
      <div class="bj-score-item bj-score-draws">
        <span class="bj-score-label">Empates</span>
        <span class="bj-score-value">{{ draws }}</span>
      </div>
      <div class="bj-score-item">
        <span class="bj-score-label">IA</span>
        <span class="bj-score-value">{{ dealerWins }}</span>
      </div>
    </div>

    <!-- Dealer Area -->
    <div class="bj-zone">
      <div class="bj-zone-header">
        <span class="bj-zone-label">Crupier</span>
        <span class="bj-zone-points">{{ gamePhase !== 'idle' ? dealerVisibleScore : '' }}</span>
      </div>
      <div class="bj-hand">
        <div
          v-for="(card, i) in dealerHand"
          :key="i"
          :class="['bj-card', { 'bj-card--hidden': i === 1 && dealerHideSecond, 'bj-card--red': isRedSuit(card.suit) }]"
        >
          <template v-if="!(i === 1 && dealerHideSecond)">
            <span class="bj-card-corner bj-card-corner--tl">{{ card.rank }}<br>{{ card.suit }}</span>
            <span class="bj-card-center">{{ card.suit }}</span>
            <span class="bj-card-corner bj-card-corner--br">{{ card.rank }}<br>{{ card.suit }}</span>
          </template>
          <template v-else>
            <span class="bj-card-back">🂠</span>
          </template>
        </div>
        <div v-if="gamePhase === 'idle'" class="bj-placeholder">— Sin cartas —</div>
      </div>
    </div>

    <!-- Player Area -->
    <div class="bj-zone bj-zone--player">
      <div class="bj-zone-header">
        <span class="bj-zone-label">Tú</span>
        <span class="bj-zone-points">{{ gamePhase !== 'idle' ? playerScore : '' }}</span>
      </div>
      <div class="bj-hand">
        <div
          v-for="(card, i) in playerHand"
          :key="i"
          :class="['bj-card', { 'bj-card--red': isRedSuit(card.suit) }]"
        >
          <span class="bj-card-corner bj-card-corner--tl">{{ card.rank }}<br>{{ card.suit }}</span>
          <span class="bj-card-center">{{ card.suit }}</span>
          <span class="bj-card-corner bj-card-corner--br">{{ card.rank }}<br>{{ card.suit }}</span>
        </div>
        <div v-if="gamePhase === 'idle'" class="bj-placeholder">— Sin cartas —</div>
      </div>
    </div>

    <!-- Message -->
    <div v-if="message" class="bj-message">{{ message }}</div>

    <!-- Actions -->
    <div class="bj-actions">
      <button v-if="gamePhase === 'idle' || gamePhase === 'result'" class="bj-btn bj-btn--deal" @click="deal">
        {{ gamePhase === 'result' ? 'Nueva Mano' : 'Repartir' }}
      </button>
      <button v-if="gamePhase === 'playing'" class="bj-btn bj-btn--hit" @click="hit">Pedir (Hit)</button>
      <button v-if="gamePhase === 'playing'" class="bj-btn bj-btn--stand" @click="stand">Plantarse (Stand)</button>
      <button v-if="gamePhase !== 'idle'" class="bj-btn bj-btn--reset" @click="resetStats">Reiniciar Stats</button>
    </div>
  </div>
</template>

<style scoped>
/* ============================================================
   DESKTOP BLACKJACK — ocupa 90 % del ancho, 5 % de margen
   ============================================================ */

.bj-table {
  min-height: 100vh;
  width: 90%;
  margin-left: 5%;
  margin-right: 5%;
  background: radial-gradient(ellipse 90% 70% at 50% 45%,
    #236b40 0%, #0d4a28 55%, #061e10 100%);
  display: flex;
  flex-direction: column;
  align-items: stretch;
  font-family: 'Georgia', serif;
  color: #f0e6c8;
  padding: 2.5rem 3rem 3rem;
  user-select: none;
}

/* ---- Title ---- */
.bj-title {
  font-size: 2.8rem;
  letter-spacing: 0.2em;
  text-align: center;
  text-shadow: 0 3px 14px rgba(0,0,0,0.7);
  margin-bottom: 1.2rem;
}

/* ---- Scoreboard ---- */
.bj-scoreboard {
  display: flex;
  justify-content: center;
  gap: 4rem;
  background: rgba(0,0,0,0.4);
  border-radius: 14px;
  padding: 0.9rem 3rem;
  margin-bottom: 2.2rem;
  border: 1px solid rgba(255,255,255,0.12);
  backdrop-filter: blur(4px);
}
.bj-score-item {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.15rem;
}
.bj-score-label {
  font-size: 0.78rem;
  text-transform: uppercase;
  letter-spacing: 0.14em;
  opacity: 0.65;
}
.bj-score-value {
  font-size: 2.2rem;
  font-weight: bold;
  color: #ffd700;
  line-height: 1;
}
.bj-score-draws .bj-score-value {
  color: #9e9e9e;
}

/* ---- Zones (dealer / player) ---- */
.bj-zone {
  width: 100%;
  background: rgba(0,0,0,0.22);
  border-radius: 20px;
  padding: 1.4rem 2rem;
  margin-bottom: 0.9rem;
  border: 1px solid rgba(255,255,255,0.08);
  box-shadow: inset 0 1px 0 rgba(255,255,255,0.06);
}
.bj-zone--player {
  border-color: rgba(255,215,0,0.3);
  box-shadow:
    inset 0 1px 0 rgba(255,215,0,0.08),
    0 0 24px rgba(255,215,0,0.06);
}
.bj-zone-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
}
.bj-zone-label {
  font-size: 1rem;
  text-transform: uppercase;
  letter-spacing: 0.16em;
  opacity: 0.7;
}
.bj-zone-points {
  font-size: 1.6rem;
  font-weight: bold;
  color: #ffd700;
  min-width: 2.5ch;
  text-align: right;
}

/* ---- Card hand ---- */
.bj-hand {
  display: flex;
  flex-wrap: nowrap;
  gap: 0.8rem;
  min-height: 140px;
  align-items: center;
  overflow-x: auto;
  padding-bottom: 4px;
  scrollbar-width: thin;
  scrollbar-color: rgba(255,255,255,0.15) transparent;
}

.bj-placeholder {
  opacity: 0.25;
  font-style: italic;
  font-size: 1rem;
  padding: 0.5rem 0;
}

/* ---- Cards ---- */
.bj-card {
  position: relative;
  flex-shrink: 0;
  width: 90px;
  height: 130px;
  background: #fffef8;
  border-radius: 10px;
  box-shadow: 3px 6px 18px rgba(0,0,0,0.55);
  color: #1a1a2e;
  display: flex;
  align-items: center;
  justify-content: center;
  border: 1px solid #d0c8b0;
  transition: transform 0.18s ease, box-shadow 0.18s ease;
  animation: dealCard 0.28s cubic-bezier(0.22, 0.61, 0.36, 1);
}
.bj-card:hover {
  transform: translateY(-8px) rotate(-1deg);
  box-shadow: 4px 12px 28px rgba(0,0,0,0.65);
}
.bj-card--red {
  color: #c0392b;
}
.bj-card--hidden {
  background:
    repeating-linear-gradient(
      45deg,
      #1a237e,
      #1a237e 6px,
      #283593 6px,
      #283593 12px
    );
  border-color: #3949ab;
  box-shadow: 3px 6px 18px rgba(0,0,0,0.6);
}
.bj-card-corner {
  position: absolute;
  font-size: 0.78rem;
  font-weight: bold;
  line-height: 1.25;
  text-align: center;
}
.bj-card-corner--tl { top: 6px; left: 8px; }
.bj-card-corner--br { bottom: 6px; right: 8px; transform: rotate(180deg); }
.bj-card-center {
  font-size: 2rem;
  line-height: 1;
}
.bj-card-back {
  font-size: 3.5rem;
  color: rgba(255,255,255,0.08);
}

@keyframes dealCard {
  from { opacity: 0; transform: translateY(-30px) scale(0.8); }
  to   { opacity: 1; transform: translateY(0) scale(1); }
}

/* ---- Result message ---- */
.bj-message {
  font-size: 1.7rem;
  font-weight: bold;
  text-align: center;
  padding: 0.9rem 2.5rem;
  background: rgba(0,0,0,0.5);
  border-radius: 12px;
  margin: 0.4rem 0 1rem;
  border: 1px solid rgba(255,215,0,0.45);
  color: #ffd700;
  text-shadow: 0 2px 6px rgba(0,0,0,0.8);
  animation: fadeIn 0.3s ease-out;
  letter-spacing: 0.04em;
}
@keyframes fadeIn {
  from { opacity: 0; transform: scale(0.93); }
  to   { opacity: 1; transform: scale(1); }
}

/* ---- Action buttons ---- */
.bj-actions {
  display: flex;
  gap: 1.2rem;
  justify-content: center;
  margin-top: 0.6rem;
  flex-wrap: wrap;
}
.bj-btn {
  padding: 0.85rem 2.4rem;
  border: none;
  border-radius: 10px;
  font-size: 1.05rem;
  font-family: inherit;
  font-weight: bold;
  cursor: pointer;
  letter-spacing: 0.06em;
  transition: transform 0.12s, box-shadow 0.12s, filter 0.12s;
  box-shadow: 0 5px 14px rgba(0,0,0,0.45);
  min-width: 160px;
}
.bj-btn:focus-visible {
  outline: 3px solid #ffd700;
  outline-offset: 3px;
}
.bj-btn:active  { transform: scale(0.95) !important; }
.bj-btn:hover   { filter: brightness(1.12); transform: translateY(-3px); box-shadow: 0 8px 20px rgba(0,0,0,0.5); }

.bj-btn--deal  { background: linear-gradient(135deg, #ffd700, #f0a500); color: #1a1a1a; }
.bj-btn--hit   { background: linear-gradient(135deg, #2ecc71, #1a9e55); color: #fff; }
.bj-btn--stand { background: linear-gradient(135deg, #e74c3c, #b03a2e); color: #fff; }
.bj-btn--reset {
  background: rgba(255,255,255,0.1);
  color: #c8bfa0;
  border: 1px solid rgba(255,255,255,0.18);
  min-width: 140px;
}

/* ---- Divider line between zones ---- */
.bj-zone + .bj-zone {
  margin-top: 0;
}
</style>
