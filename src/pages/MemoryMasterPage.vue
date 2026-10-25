<template>
  <q-page class="flex flex-center text-white q-pa-md memory-page">
    <!-- Painel central (Frente) em azul escuro -->
    <div
      class="game-container column items-center q-gutter-md q-pa-lg"
      style="max-width: 650px; width: 100%"
    >
      <!-- Cabeçalho Organizado -->
      <div class="column items-center q-gutter-sm w-full q-px-md" style="width: 100%">
        <div class="text-h5 font-weight-bold text-white">Memory Master</div>

        <!-- Tentativas na linha seguinte, centrado -->
        <div class="row justify-center items-center">
          <q-chip color="dark" text-color="white" icon="star"
            >Tentativas: {{ attempts }} / 20</q-chip
          >
        </div>

        <!-- Botões centrados na linha seguinte -->
        <div class="row justify-center items-center q-gutter-sm flex-wrap q-mt-xs">
          <q-btn
            color="orange"
            icon="refresh"
            label="Novo Jogo"
            @click="initGame"
            class="action-btn"
          />
          <q-btn
            color="info"
            icon="info"
            label="Acerca de"
            @click="showAbout = true"
            class="action-btn"
          />
          <q-btn
            color="negative"
            icon="exit_to_app"
            label="Sair"
            @click="exitGame"
            class="action-btn"
          />
        </div>
      </div>

      <!-- Grelha do Jogo -->
      <div class="memory-grid">
        <div
          v-for="(card, index) in cards"
          :key="index"
          class="card-container"
          :class="{ flipped: card.isFlipped || card.isMatched }"
          @click="flipCard(index)"
        >
          <div class="card-inner">
            <!-- Verso da Carta (Linha 3, Coluna 0) -->
            <div class="card-face card-back">
              <img src="images/imagensMemory.png" class="atlas-img" :style="getSpriteStyle(3, 0)" />
            </div>

            <!-- Frente da Carta -->
            <div class="card-face card-front">
              <img
                src="images/imagensMemory.png"
                class="atlas-img"
                :style="getSpriteStyle(card.row, card.col)"
              />
            </div>
          </div>
        </div>
      </div>

      <!-- Ecrã de Vitória -->
      <q-dialog v-model="won" persistent>
        <q-card class="bg-dark text-white q-pa-lg text-center dialog-effect victory-dialog">
          <q-card-section>
            <q-icon name="military_tech" color="yellow" size="64px" class="q-mb-sm" />
            <div class="text-h4 text-yellow font-weight-bold">Parabéns! Venceu!</div>
            <div class="text-subtitle1 q-mt-md">
              Completou o jogo em
              <span class="text-bold text-orange">{{ attempts }}</span> tentativas!
            </div>
          </q-card-section>
          <q-card-actions align="center">
            <q-btn
              color="yellow"
              text-color="dark"
              icon="replay"
              label="Jogar Novamente"
              @click="initGame"
              v-close-popup
            />
            <q-btn color="grey-7" label="Menu Principal" @click="exitGame" v-close-popup />
          </q-card-actions>
        </q-card>
      </q-dialog>

      <!-- Ecrã de Game Over -->
      <q-dialog v-model="gameOver" persistent>
        <q-card class="bg-dark text-white q-pa-lg text-center dialog-effect gameover-dialog">
          <q-card-section>
            <q-icon name="sentiment_very_dissatisfied" color="red" size="64px" class="q-mb-sm" />
            <div class="text-h4 text-red font-weight-bold">Fim do Jogo!</div>
            <div class="text-subtitle1 q-mt-md">
              Atingiu o limite máximo de <span class="text-bold text-red">20</span> tentativas.
              Tente novamente!
            </div>
          </q-card-section>
          <q-card-actions align="center">
            <q-btn
              color="red"
              icon="refresh"
              label="Tentar Novamente"
              @click="initGame"
              v-close-popup
            />
            <q-btn color="grey-7" label="Menu Principal" @click="exitGame" v-close-popup />
          </q-card-actions>
        </q-card>
      </q-dialog>

      <!-- Diálogo Acerca de -->
      <q-dialog v-model="showAbout">
        <q-card
          class="bg-dark text-white q-pa-lg text-center dialog-effect"
          style="min-width: 320px"
        >
          <q-card-section>
            <q-icon name="sports_esports" color="orange" size="48px" class="q-mb-sm" />
            <div class="text-h5 text-orange font-weight-bold">Memory Master</div>
            <div class="text-subtitle2 q-mt-xs text-grey-4">Desenvolvido por Luis Manuel Gimón</div>
            <div class="text-body2 q-mt-md">
              Jogo de memória interativo construído com Quasar, Vue.js e TypeScript, utilizando
              técnicas avançadas de sprites em atlas de imagens.
            </div>
          </q-card-section>
          <q-card-actions align="center">
            <q-btn color="orange" label="Fechar" v-close-popup />
          </q-card-actions>
        </q-card>
      </q-dialog>
    </div>
  </q-page>
</template>

<script setup lang="ts">
import { ref, onMounted } from 'vue';
import { useRouter } from 'vue-router';

interface Card {
  id: number;
  row: number;
  col: number;
  isFlipped: boolean;
  isMatched: boolean;
}

const router = useRouter();

// Mapeamento apenas dos ícones válidos (linhas 0, 1 e 2 do atlas)
const availableIcons = [
  { row: 0, col: 0 }, // Cão
  { row: 0, col: 1 }, // Gato
  { row: 0, col: 2 }, // Papagaio
  { row: 0, col: 3 }, // Leão
  { row: 1, col: 0 }, // Elefante
  { row: 1, col: 1 }, // Vaca
  { row: 1, col: 2 }, // Cavalo
  { row: 1, col: 3 }, // Porco
  { row: 2, col: 0 }, // Trator
  { row: 2, col: 1 }, // Casa
  { row: 2, col: 2 }, // Árvore
  { row: 2, col: 3 }, // Girassol
];

const cards = ref<Card[]>([]);
const attempts = ref(0);
const won = ref(false);
const gameOver = ref(false);
const showAbout = ref(false);
let firstSelection: number | null = null;
let lockBoard = false;

// Alinhamento perfeito do sprite por transform
const getSpriteStyle = (row: number, col: number) => {
  return {
    transform: `translate(${-col * 25}%, ${-row * 25}%)`,
  };
};

// Inicializar e embaralhar o jogo
const initGame = () => {
  attempts.value = 0;
  won.value = false;
  gameOver.value = false;
  firstSelection = null;
  lockBoard = false;

  const shuffledIcons = [...availableIcons].sort(() => 0.5 - Math.random());
  const selectedPairs = shuffledIcons.slice(0, 8);

  const deck = [...selectedPairs, ...selectedPairs]
    .sort(() => 0.5 - Math.random())
    .map((item, index) => ({
      id: index,
      row: item.row,
      col: item.col,
      isFlipped: false,
      isMatched: false,
    }));

  cards.value = deck;
};

// Ação do botão Sair
const exitGame = () => {
  router.push('/');
};

// Lógica de clique na carta
const flipCard = (index: number) => {
  if (lockBoard || gameOver.value || won.value) return;
  const card = cards.value[index];
  if (!card || card.isFlipped || card.isMatched) return;

  card.isFlipped = true;

  if (firstSelection === null) {
    firstSelection = index;
    return;
  }

  attempts.value++;

  const selectedIndex = firstSelection;
  const firstCard = cards.value[selectedIndex];

  if (!firstCard) return;

  // Verificar se formam par
  if (firstCard.row === card.row && firstCard.col === card.col) {
    firstCard.isMatched = true;
    card.isMatched = true;
    firstSelection = null;

    // Verificar condição de vitória
    if (cards.value.every((c) => c.isMatched)) {
      setTimeout(() => {
        won.value = true;
      }, 400);
    }
  } else {
    // Verificar se atingiu o limite de 20 tentativas após erro
    if (attempts.value >= 20) {
      lockBoard = true;
      setTimeout(() => {
        gameOver.value = true;
      }, 500);
      return;
    }

    // Não coincidem: virar de volta após pequeno atraso
    lockBoard = true;
    setTimeout(() => {
      firstCard.isFlipped = false;
      card.isFlipped = false;
      firstSelection = null;
      lockBoard = false;
    }, 1000);
  }
};

onMounted(() => {
  initGame();
});
</script>

<style scoped>
/* Fundo geral da página (margens) em Azul Escuro */
.memory-page {
  background-color: #0b2265 !important;
  min-height: 100vh;
  width: 100%;
}

/* Painel central (Frente) num Azul mais escuro */
.game-container {
  background-color: #1565c0 !important;
  border-radius: 20px;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.4);
}

.memory-grid {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 12px;
  width: 100%;
  max-width: 480px;
  aspect-ratio: 1;
}

.card-container {
  perspective: 1000px;
  cursor: pointer;
}

.card-inner {
  position: relative;
  width: 100%;
  height: 100%;
  transition: transform 0.5s;
  transform-style: preserve-3d;
}

.card-container.flipped .card-inner {
  transform: rotateY(180deg);
}

.card-face {
  position: absolute;
  width: 100%;
  height: 100%;
  backface-visibility: hidden;
  border-radius: 12px;
  overflow: hidden;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.3);
  border: 2px solid rgba(255, 255, 255, 0.2);
  background-color: #fff;
}

.card-back {
  background-color: #1976d2;
}

.card-front {
  transform: rotateY(180deg);
}

.atlas-img {
  position: absolute;
  top: 0;
  left: 0;
  width: 400%;
  height: 400%;
  max-width: none;
  pointer-events: none;
}

/* Botões uniformes com fonte reduzida */
.action-btn {
  width: 135px;
  font-size: 11px;
}

.dialog-effect {
  border-radius: 16px;
  min-width: 320px;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.5);
}

.victory-dialog {
  border: 2px solid #fbc02d;
}

.gameover-dialog {
  border: 2px solid #e53935;
}
</style>
