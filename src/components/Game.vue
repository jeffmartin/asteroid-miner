<template>
  <div class="clicker-game">
    <h1>Idle Miner</h1>
    <div class="nav-bar">
      <button @click="currentView = 'game'" :class="{ active: currentView === 'game' }">⛏ Game</button>
      <button @click="currentView = 'achievements'" :class="{ active: currentView === 'achievements' }">🏆 Achievements</button>
    </div>
    <div v-if="currentView === 'game'">
    <button @click="toggleMute" class="mute-button">
      {{ audio.muted ? "🔇 Unmute" : "🔊 Mute" }}
    </button>
    <p>Mined Resources: {{ resources }}</p>
    <p>Resources per Click: {{ clickValue }}</p>
    <p>Auto-Miners: {{ autoMiners }} ({{ totalAutoMinerRate }} resources/sec)</p>

    <div class="mine-button-wrapper">
      <button @click="mineResource" class="mine-button">Mine</button>
      
      <transition-group name="float" tag="div" class="click-feedback">
        <div 
          v-for="item in clickFeedback" 
          :key="item.id" 
          class="float-text"
        >
          +{{ item.value }}
        </div>
      </transition-group>
    </div>

    <div class="upgrades">
      <h2>Upgrades</h2>

      <div class="upgrade-section">
        <p>Pickaxe Upgrade (Cost: {{ upgradeCost }})</p>
        <button :disabled="resources < upgradeCost" @click="buyUpgrade">
          Buy Pickaxe Upgrade
        </button>
      </div>

      <div class="upgrade-section">
        <div class="miners">
          <h2>Auto-Miners</h2>
          <div v-for="(miner, index) in minerTypes" :key="miner.name" class="miner-card">
            <p>
              {{ miner.name }} — {{ miner.owned }} owned<br />
              Produces {{ miner.rate }}/sec each<br />
              Cost: {{ minerCost(miner) }}
            </p>
            <button 
              :disabled="resources < minerCost(miner)"
              @click="buyMiner(index)"
              >
              Buy {{ miner.name }}
            </button>
          </div>
          <div class="sprite-gallery">
            <h2>Miners at Work</h2>
            <div class="sprite-row" v-for="miner in minerTypes" :key="miner.name">
              <div
                class="sprite"
                v-for="n in miner.owned"
                :key="n + '-' + miner.name"
              >
              <img :src="miner.image" :alt="miner.name" />
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
    <button @click="resetGame">Reset Game</button>
    </div>
    <div v-if="currentView === 'achievements'" class="achievements-tab">
      <h2>Achievements</h2>
      <div class="achievement-list">
        <div
          class="achievement-card"
          v-for="ach in achievements"
          :key="ach.id"
          :class="{ unlocked: ach.unlocked }"
          >
          <h3>{{ ach.unlocked ? ach.name : "???" }}</h3>
          <p>{{ ach.unlocked ? ach.description : "Unlock to reveal." }}</p>
        </div>
      </div>
    </div>

    <transition name="fade">
    <div v-if="currentAchievement" class="achievement-popup">
      <h3>🏆 Achievement Unlocked!</h3>
      <strong>{{ currentAchievement.name }}</strong>
      <p>{{ currentAchievement.description }}</p>
    </div>
</transition>
  </div>
</template>

<script>
let idCounter = 0;

export default {
  name: 'Game',
  data() {
    return {
      currentView: "game", // or "achievements"
      autoMiners:0,
        resources: 0,
        clickValue: 1,
        upgradeLevel: 0,
        baseUpgradeCost: 10,
        intervalId: null,
        saveIntervalId: null,
        clickFeedback: [],
        minerTypes: [
        {
            name: "Shovel Guy",
            baseCost: 50,
            rate: 1,
            owned: 0,
            image: "images/shovel.gif"
        },
        {
            name: "Drill Bot",
            baseCost: 300,
            rate: 5,
            owned: 0,
            image: "images/drill.gif"
        },
        {
            name: "Laser Excavator",
            baseCost: 1000,
            rate: 20,
            owned: 0,
            image: "images/laser.gif"
        }
        ],
        audio: {
            click: null,
            buy: null,
            bgm: null,
            muted: false
        },
        achievements: [
        {
          id: "firstClick",
          name: "First Click!",
          description: "Mined your first resource.",
          unlocked: false
        },
        {
          id: "upgradeTime",
          name: "Getting Stronger",
          description: "Buy your first upgrade.",
          unlocked: false
        },
        {
          id: "minerMania",
          name: "Auto Army",
          description: "Own 1 of each miner type.",
          unlocked: false
        },
        {
          id: "richMiner",
          name: "Filthy Rich",
          description: "Reach 1000 resources.",
          unlocked: false
        }
      ],
      achievementQueue: [],
      achievementPopupTimer: null,
      currentAchievement: null,
    };
  },

  computed: {
    upgradeCost() {
    return Math.floor(this.baseUpgradeCost * Math.pow(1.5, this.upgradeLevel));
  },
  totalAutoMinerRate() {
    return this.minerTypes.reduce(
      (sum, miner) => sum + miner.owned * miner.rate,
      0
    );
  }
    
  },
  methods: {
    checkAchievements() {
      const a = this.achievements;

      if (!a.find(x => x.id === "firstClick").unlocked && this.resources >= 1) {
        this.unlockAchievement("firstClick");
      }

      if (!a.find(x => x.id === "upgradeTime").unlocked && this.upgradeLevel >= 1) {
        this.unlockAchievement("upgradeTime");
      }

      if (!a.find(x => x.id === "minerMania").unlocked &&
        this.minerTypes.every(m => m.owned > 0)) {
        this.unlockAchievement("minerMania");
      }

      if (!a.find(x => x.id === "richMiner").unlocked && this.resources >= 1000) {
        this.unlockAchievement("richMiner");
      }
    },
    unlockAchievement(id) {
      const achievement = this.achievements.find(a => a.id === id);
      if (achievement && !achievement.unlocked) {
        achievement.unlocked = true;
        this.achievementQueue.push(achievement);
        this.saveGame();

        // Show popup for 3 seconds
        if (!this.achievementPopupTimer) {
          this.showNextAchievementPopup();
        }
      }
    },
    showNextAchievementPopup() {
      if (this.achievementQueue.length === 0) {
        this.achievementPopupTimer = null;
        return;
      }

      const achievement = this.achievementQueue.shift();
      this.currentAchievement = achievement;

      this.achievementPopupTimer = setTimeout(() => {
        this.currentAchievement = null;
        this.showNextAchievementPopup();
      }, 3000);
    },
    toggleMute() {
      this.audio.muted = !this.audio.muted;

      if (this.audio.bgm) {
        this.audio.bgm.muted = this.audio.muted;
      }
    },
    mineResource() {
      this.resources += this.clickValue;
      
      if (!this.audio.muted && this.audio.click) this.audio.click.play();

      const feedback = {
        id: idCounter++,
        value: this.clickValue
      };
      this.clickFeedback.push(feedback);
      setTimeout(() => {
        this.clickFeedback = this.clickFeedback.filter(item => item.id !== feedback.id);
      }, 1000);
      this.checkAchievements();
    },
    buyUpgrade() {
      const cost = this.upgradeCost;
      if (this.resources >= cost) {
        this.resources -= cost;
        this.upgradeLevel += 1;
        this.clickValue += 1;

        if (!this.audio.muted && this.audio.buy) this.audio.buy.play();
        this.checkAchievements();
      }
    },
    buyAutoMiner() {
      const cost = this.autoMinerCost;
      if (this.resources >= cost) {
        this.resources -= cost;
        this.autoMiners += 1;
      }
    },
    buyMiner(index) {
      const miner = this.minerTypes[index];
      const cost = Math.floor(miner.baseCost * Math.pow(1.15, miner.owned));
      if (this.resources >= cost) {
        this.resources -= cost;
        miner.owned += 1;
        this.autoMiners += 1;

      if (!this.audio.muted && this.audio.buy) this.audio.buy.play();
      this.checkAchievements();
      }
    },
    loadAudio() {
      this.audio.click = new Audio("sounds/click.mp3");
      this.audio.buy = new Audio("sounds/buy.mp3");
      this.audio.bgm = new Audio("sounds/bgm.mp3");
      this.audio.bgm.loop = true;
      this.audio.bgm.volume = 0.3;
      this.audio.bgm.play().catch(() => {
    // Autoplay blocked, wait for interaction
    });
  },
  minerCost(miner) {
    return Math.floor(miner.baseCost * Math.pow(1.15, miner.owned));
  },
  tick() {
      this.resources += this.totalAutoMinerRate;
      this.checkAchievements();
  },
  saveGame() {
    const gameData = {
      resources: this.resources,
      clickValue: this.clickValue,
      upgradeLevel: this.upgradeLevel,
      minerTypes: this.minerTypes.map(miner => ({
        owned: miner.owned
      })),
      achievements: this.achievements.map(a => ({
        id: a.id,
        unlocked: a.unlocked
      }))
    };
    localStorage.setItem('idleMinerSave', JSON.stringify(gameData));
  },
  loadGame() {
    const saved = localStorage.getItem('idleMinerSave');
    if (saved) {
      const data = JSON.parse(saved);
      this.resources = data.resources ?? 0;
      this.clickValue = data.clickValue ?? 1;
      this.upgradeLevel = data.upgradeLevel ?? 0;

      if (data.minerTypes) {
        data.minerTypes.forEach((savedMiner, index) => {
          if (this.minerTypes[index]) {
            this.minerTypes[index].owned = savedMiner.owned ?? 0;
            this.autoMiners += this.minerTypes[index].owned;
          }
          
        });
      }

      if (data.achievements) {
        data.achievements.forEach(saved => {
          const match = this.achievements.find(a => a.id === saved.id);
          if (match) match.unlocked = saved.unlocked;
        });
      }
    }
  },

  resetGame() {
        localStorage.removeItem('idleMinerSave');
        location.reload();
    }
  },
  mounted() {
    this.loadGame();
    this.loadAudio();
    this.intervalId = setInterval(this.tick, 1000);
    this.saveIntervalId = setInterval(this.saveGame, 5000); // auto-save every 5 sec
  },
  beforeUnmount() {
    clearInterval(this.intervalId);
    clearInterval(this.saveIntervalId);
    this.saveGame(); // save on exit too
  }
};
</script>

<style scoped>
.clicker-game {
  text-align: center;
  margin-top: 50px;
  font-family: sans-serif;
}

.mine-button-wrapper {
  position: relative;
  display: inline-block;
}

.mine-button {
  padding: 15px 30px;
  font-size: 22px;
  font-weight: bold;
  border-radius: 12px;
  border: none;
  background-color: #4caf50;
  color: white;
  transition: transform 0.1s ease, background-color 0.2s;
  cursor: pointer;
  touch-action: manipulation;
  user-select: none;
}

.mine-button:hover {
  background-color: #45a049;
}

.mine-button:active {
  transform: scale(0.95);
}

.click-feedback {
  position: absolute;
  top: 0;
  left: 50%;
  transform: translateX(-50%);
  pointer-events: none;
}

.float-text {
  position: absolute;
  font-size: 18px;
  color: #2ecc71;
  font-weight: bold;
  animation: floatUp 1s ease forwards;
}

@keyframes floatUp {
  0% {
    transform: translateY(0);
    opacity: 1;
  }
  100% {
    transform: translateY(-40px);
    opacity: 0;
  }
}

button {
  font-size: 18px;
  padding: 14px 20px;
  border-radius: 8px;
  margin: 8px 0;
  touch-action: manipulation; /* prevents double-tap zoom */
  cursor:pointer;
}

button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.upgrades {
  margin-top: 30px;
  border-top: 1px solid #ccc;
  padding-top: 20px;
}

.upgrade-section {
  margin: 20px 0;
}
.miners {
  margin-top: 40px;
}

.miner-card {
  border: 1px solid #ccc;
  padding: 15px;
  margin: 10px auto;
  width: 300px;
  border-radius: 10px;
  background-color: #f7f7f7;
}
.mute-button {
  position: absolute;
  top: 10px;
  right: 10px;
  padding: 5px 10px;
  font-size: 16px;
  border-radius: 6px;
  cursor: pointer;
}
.sprite-gallery {
  margin-top: 40px;
  padding: 10px;
  border-top: 2px dashed #aaa;
  text-align: center;
  float:right;
}

.sprite-row {
  display: flex;
  justify-content: center;
  flex-wrap: wrap;
  margin-bottom: 20px;
  gap: 8px;
}

.sprite {
  margin: 4px;
  width: 48px;
  height: 48px;
}

.sprite img {
  width: 40px;
  height: 40px;
}

.achievement-popup {
  position: fixed;
  bottom: 20px;
  right: 20px;
  background: #fffae6;
  border: 2px solid #ffd700;
  border-radius: 10px;
  padding: 15px 20px;
  z-index: 9999;
  box-shadow: 0 0 10px gold;
  animation: bounce-in 0.5s ease;
}

@keyframes bounce-in {
  0% {
    transform: translateY(100%);
    opacity: 0;
  }
  60% {
    transform: translateY(-10%);
    opacity: 1;
  }
  100% {
    transform: translateY(0);
  }
}

.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.5s;
}
.fade-enter, .fade-leave-to {
  opacity: 0;
}

.nav-bar {
  display: flex;
  gap: 10px;
  margin-bottom: 20px;
  flex-wrap: wrap;
  justify-content: center;
}

.nav-bar button {
  padding: 8px 16px;
  font-weight: bold;
  border-radius: 8px;
  cursor: pointer;
  background: #eee;
  border: 1px solid #ccc;
  flex: 1 1 45%;
  min-width: 120px;
}

.nav-bar .active {
  background: #ffd700;
  border-color: gold;
}

.achievements-tab {
  padding: 20px;
  background: #f4f4f4;
  border-radius: 10px;
}

.achievement-list {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(140px, 1fr));
  gap: 12px;
}
.achievement-card {
  background: #ddd;
  padding: 15px;
  border-radius: 8px;
  width: 200px;
  box-shadow: 2px 2px 5px #999;
  opacity: 0.5;
  transition: all 0.3s ease;
}

.achievement-card.unlocked {
  background: #fff9db;
  border: 2px solid gold;
  opacity: 1;
  transform: scale(1.03);
}

body {
  background-color: #ffffff;
  color: #333333;
  margin: 0;
  padding: 0;
  font-family: 'Arial', sans-serif;
  touch-action: manipulation;
}

button {
  background-color: #f0f0f0;
  color: #000000;
}

input, select, textarea {
  background-color: #ffffff;
  color: #000000;
}

.achievement-card {
  background-color: #ffffff;
  color: #000000;
}

#app {
  max-width: 480px;
  margin: 0 auto;
  padding: 16px;
}
@media (prefers-color-scheme: dark) {
  body {
    background-color: #121212;
    color: #eeeeee;
  }

  button {
    background-color: #333;
    color: #fff;
  }

  .achievement-card {
    background-color: #222;
    color: #ddd;
  }
}
</style>
