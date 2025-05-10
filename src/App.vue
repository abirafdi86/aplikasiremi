<template>
  <div class="score-tracker-container">
    <!-- Tampilan berbeda untuk host/admin dan viewer -->
    <div v-if="liveMode && !isHost" class="viewer-mode">
      <!-- Header Viewer Mode -->
      <div class="viewer-header">
        <h1 class="title">Live Score</h1>
        <div class="live-status">
          <span class="status-indicator" :class="connectionStatus"></span>
          <span v-if="connectionStatus === 'connected'" class="status-text connected">
            Terhubung ke Live Score
          </span>
          <span v-else-if="connectionStatus === 'connecting'" class="status-text connecting">
            Menghubungkan ke Live Score...
          </span>
          <span v-else-if="connectionStatus === 'waiting'" class="status-text waiting">
            Menunggu data dari host...
          </span>
          <span v-else-if="connectionStatus === 'failed'" class="status-text failed">
            Host telah meninggalkan room
          </span>
          <span v-else-if="connectionStatus === 'error'" class="status-text error">
            Error saat memuat data
          </span>
          <span v-if="lastUpdateTime" class="last-update">
            Update terakhir: {{ formatTime(lastUpdateTime) }}
          </span>
        </div>
      </div>

      <!-- Tampilan terima kasih ketika host telah meninggalkan room -->
      <div v-if="connectionStatus === 'failed'" class="thank-you-container">
        <div class="thank-you-message">
          <h2>Terima Kasih Telah Menonton!</h2>
          <p>Host telah mengakhiri sesi Live Score.</p>
          <p>Semoga permainan berikutnya lebih menyenangkan!</p>
        </div>
      </div>

      <!-- Score Summary untuk Viewer -->
      <div v-if="participants.length > 0 && connectionStatus !== 'failed'" class="viewer-scores-summary">
        <div v-for="participant in participants" :key="participant.name" class="viewer-participant-total">
          <span class="participant-name">{{ participant.name }}:</span>
          <span class="participant-total-score" :style="{ color: getScoreColor(participant.name) }">
            {{ calculateTotalScore(participant.name) }}
          </span>
          <span v-if="participant.name === lowestScoringParticipant && secondLowestScoringParticipant"
            class="participant-score-difference">
            (Selisih: {{ calculateScoreDifference() }})
          </span>
        </div>
      </div>

      <!-- Charts untuk Viewer -->
      <div v-if="participants.length > 0 && connectionStatus !== 'failed'" class="viewer-charts">
        <div v-for="(participant, index) in participants" :key="participant.name" class="viewer-chart-container">
          <h2 class="chart-title">{{ participant.name }}</h2>
          <div class="line-chart" :ref="`chart_${participant.name}`">
            <div class="chart-zero-line"></div>
            <div class="chart-grid">
              <div class="line-track" :style="{ backgroundColor: getChartBackgroundColor(index) }">
                <svg class="chart-svg" viewBox="0 0 100 100" preserveAspectRatio="none">
                  <polyline :points="generatePolylinePoints(participant.name)"
                    :stroke="getParticipantColor(participant.name)" fill="none" stroke-width="2" />
                </svg>

                <div v-for="(point, pointIndex) in calculateCumulativePoints(participant.name)" :key="pointIndex"
                  class="line-point" :style="{
                    left: `${(pointIndex / (Math.max(scores.length - 1, 1))) * 100}%`,
                    bottom: calculatePointPosition(participant.name, point),
                    backgroundColor: getParticipantColor(participant.name)
                  }" :title="`Round ${pointIndex + 1}: ${point}`"></div>
              </div>
            </div>
            <div class="chart-x-axis">
              <span v-for="(_, index) in scores" :key="index">
                {{ index + 1 }}
              </span>
            </div>
            <div class="chart-y-axis">
              <span v-for="n in 5" :key="n">
                {{ formatYAxisLabel(calculateYAxisValue(n)) }}
              </span>
            </div>
          </div>
        </div>
      </div>

      <!-- Add this to the viewer-mode div, after the viewer-charts div -->
      <div v-if="participants.length > 0 && connectionStatus !== 'failed'" class="viewer-leaderboard">
        <h2 class="viewer-section-title">Leaderboard Pemain</h2>

        <div v-if="!playerStats || Object.keys(playerStats).length === 0" class="no-leaderboard">
          <p>Belum ada data statistik pemain.</p>
        </div>

        <div v-else class="viewer-leaderboard-content">
          <div class="viewer-leaderboard-filters">
            <div class="filter-group">
              <label for="viewer-sort-by">Urutkan:</label>
              <select id="viewer-sort-by" v-model="leaderboardSortBy">
                <option value="winRate">Win Rate</option>
                <option value="totalGames">Total Games</option>
                <option value="avgScore">Avg. Skor</option>
                <option value="totalWins">Total Menang</option>
              </select>
            </div>
            <div class="filter-group">
              <label for="viewer-min-games">Min. games:</label>
              <select id="viewer-min-games" v-model="leaderboardMinGames">
                <option value="0">Semua</option>
                <option value="3">3+</option>
                <option value="5">5+</option>
                <option value="10">10+</option>
              </select>
            </div>
          </div>

          <div class="viewer-leaderboard-table-container">
            <table class="viewer-leaderboard-table">
              <thead>
                <tr>
                  <th>#</th>
                  <th>Pemain</th>
                  <th>Level</th>
                  <th>Win Rate</th>
                  <th>Games</th>
                  <th>W/L</th>
                  <th>Avg</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="(player, index) in sortedLeaderboard" :key="player.name"
                  :class="getLeaderboardRowClass(index)">
                  <td class="rank-cell">{{ index + 1 }}</td>
                  <td class="player-name-cell">{{ player.name }}</td>
                  <td class="player-level-cell">
                    <span class="player-level" :class="'level-' + player.level.toLowerCase()">
                      {{ player.level }}
                    </span>
                  </td>
                  <td class="win-rate-cell">{{ formatPercent(player.winRate) }}</td>
                  <td>{{ player.totalGames }}</td>
                  <td>{{ player.wins }}/{{ player.losses }}</td>
                  <td>{{ formatNumber(player.avgScore) }}</td>
                </tr>
              </tbody>
            </table>
          </div>

          <div class="viewer-level-legend">
            <div class="level-legend-title">Level Pemain:</div>
            <div class="level-legend-items">
              <div class="level-legend-item">
                <span class="player-level level-newbie">Newbie</span>
              </div>
              <div class="level-legend-item">
                <span class="player-level level-beginner">Beginner</span>
              </div>
              <div class="level-legend-item">
                <span class="player-level level-intermediate">Inter</span>
              </div>
              <div class="level-legend-item">
                <span class="player-level level-advanced">Adv</span>
              </div>
              <div class="level-legend-item">
                <span class="player-level level-pro">Pro</span>
              </div>
              <div class="level-legend-item">
                <span class="player-level level-legend">Legend</span>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Tampilan Host/Admin Mode -->
    <div v-else>
      <h1 class="title">Skor Player</h1>

      <div class="live-score-controls">
        <div v-if="!liveMode" class="create-room-section">
          <button @click="createLiveRoom" class="create-room-btn">
            <span class="btn-icon">📡</span> Mulai Live Score
          </button>
        </div>

        <div v-if="liveMode" class="live-mode-active">
          <div class="live-status">
            <span class="status-indicator" :class="connectionStatus"></span>
            <span v-if="isHost" style="color: #6B7280;">Host Mode</span>
            <span v-else>Viewer Mode</span>
            <span v-if="lastUpdateTime" class="last-update">
              Update terakhir: {{ formatTime(lastUpdateTime) }}
            </span>
          </div>

          <div v-if="isHost" class="share-link">
            <span style="color: #6B7280;">Link Tonton:</span>
            <input type="text" readonly :value="watchUrl" class="watch-url-input" />
            <button @click="copyWatchUrl" class="copy-url-btn">Salin</button>
            <span class="viewers-count">
              {{ viewers }} penonton
            </span>
          </div>

          <button @click="stopLiveMode" class="stop-live-btn">
            <span class="btn-icon">⏹️</span> Hentikan Live Score
          </button>
        </div>
      </div>

      <!-- Player name input section - No change needed -->
      <div v-if="!participants.length" class="player-name-input">
        <h2 class="player-name-title">Masukkan Nama Player</h2>
        <div class="player-name-grid">
          <div v-for="(_, index) in 4" :key="index" class="player-name-group">
            <label :for="`player-${index + 1}`">Player {{ index + 1 }}</label>
            <input :id="`player-${index + 1}`" type="text" v-model="playerNameInputs[index]"
              :placeholder="`Nama Player ${index + 1}`" />
          </div>
        </div>
        <div class="random-position-button-container">
          <button @click="generateRandomPositions" class="random-positions-btn">
            Acak Posisi Player
          </button>
        </div>
        <button @click="savePlayerNames" class="save-players-btn">
          Simpan Nama Player
        </button>
      </div>

      <!-- Position Grid - No structural change needed -->
      <div v-if="showPositionGrid" class="position-grid-container">
        <h2 class="position-grid-title">Pilih Posisi Player</h2>
        <div class="position-grid">
          <div v-for="(cell, index) in positionGrid" :key="index" class="position-grid-cell"
            :class="{ 'occupied': cell !== null, 'selectable': cell === null }" @click="selectPosition(index)">
            <span style="color: gray;">{{ cell !== null ? cell : 'Pilih' }}</span>
          </div>
        </div>
        <div class="random-position-button-container">
          <button @click="resetPositionGrid" class="reset-positions-btn">
            Reset Posisi
          </button>
        </div>
      </div>

      <div v-if="participants.length > 0">
        <!-- Score input section -->
        <div class="input-grid">
          <div v-for="participant in participants" :key="participant.name" class="input-group">
            <label :for="participant.name">{{ participant.name }}</label>
            <div class="input-with-buttons">
              <input :id="participant.name" type="number" v-model.number="currentScore[participant.name]"
                :placeholder="`Skor ${participant.name}`" />
              <div class="input-buttons">
                <button @click="markScorePositive(participant.name)" class="thumb-button thumb-up"
                  :class="{ 'active': scoreMarks[participant.name] === 'positive' }" title="Tandai Positif">
                  👍
                </button>
                <button @click="markScoreNegative(participant.name)" class="thumb-button thumb-down"
                  :class="{ 'active': scoreMarks[participant.name] === 'negative' }" title="Tandai Negatif">
                  👎
                </button>
              </div>
            </div>
          </div>
        </div>

        <button @click="addScores" class="add-score-btn">
          Tambah Skor
        </button>

        <!-- Score summary section -->
        <div class="scores-summary">
          <div v-for="participant in participants" :key="participant.name" class="participant-total">
            <span class="participant-name">{{ participant.name }}:</span>
            <span class="participant-total-score" :style="{ color: getScoreColor(participant.name) }">
              {{ calculateTotalScore(participant.name) }}
            </span>

            <span v-if="participant.name === lowestScoringParticipant && secondLowestScoringParticipant"
              class="participant-score-difference">
              Selisih:
              <span style="font-weight: bold;">{{ calculateScoreDifference() }}</span>
            </span>
          </div>
        </div>

        <!-- Charts section - Modified to display all player charts in one row -->
        <div class="individual-charts">
          <div v-for="(participant, index) in participants" :key="participant.name" class="individual-chart-container">
            <h2 class="chart-title">{{ participant.name }}</h2>
            <div class="line-chart" :ref="`chart_${participant.name}`">
              <div class="chart-zero-line"></div>
              <div class="chart-grid">
                <div class="line-track" :style="{ backgroundColor: getChartBackgroundColor(index) }">
                  <svg class="chart-svg" viewBox="0 0 100 100" preserveAspectRatio="none">
                    <polyline :points="generatePolylinePoints(participant.name)"
                      :stroke="getParticipantColor(participant.name)" fill="none" stroke-width="2" />
                  </svg>

                  <div v-for="(point, pointIndex) in calculateCumulativePoints(participant.name)" :key="pointIndex"
                    class="line-point" :style="{
                      left: `${(pointIndex / (Math.max(scores.length - 1, 1))) * 100}%`,
                      bottom: calculatePointPosition(participant.name, point),
                      backgroundColor: getParticipantColor(participant.name)
                    }" :title="`Round ${pointIndex + 1}: ${point}`"></div>
                </div>
              </div>
              <div class="chart-x-axis">
                <span v-for="(_, index) in scores" :key="index">
                  {{ index + 1 }}
                </span>
              </div>
              <div class="chart-y-axis">
                <span v-for="n in 5" :key="n">
                  {{ formatYAxisLabel(calculateYAxisValue(n)) }}
                </span>
              </div>
            </div>
          </div>
        </div>

        <!-- Action buttons section -->
        <div class="action-buttons">
          <button @click="saveGameToHistory" class="save-history-btn">
            Simpan ke Riwayat
          </button>
          <button @click="downloadReport" class="download-btn">
            Download Laporan (JPG)
          </button>
          <button @click="resetData" class="reset-data-btn">
            Reset Data
          </button>
        </div>

        <!-- Hidden canvas for report generation -->
        <div style="display: none;">
          <canvas ref="reportCanvas" width="1400" height="2000"></canvas>
        </div>
      </div>
    </div>

    <div v-if="!liveMode || isHost" class="leaderboard-panel">
      <div class="leaderboard-header">
        <h2 class="leaderboard-title">Leaderboard Pemain</h2>
        <button @click="toggleLeaderboardPanel" class="toggle-leaderboard-btn">
          {{ showLeaderboardPanel ? 'Tutup' : 'Lihat Leaderboard' }}
        </button>
      </div>

      <div v-if="showLeaderboardPanel" class="leaderboard-content">
        <div v-if="!playerStats || Object.keys(playerStats).length === 0" class="no-leaderboard">
          <p>Belum ada data statistik pemain.</p>
        </div>

        <div v-else>
          <div class="leaderboard-filters">
            <div class="filter-group">
              <label for="sort-by">Urutkan berdasarkan:</label>
              <select id="sort-by" v-model="leaderboardSortBy">
                <option value="winRate">Persentase Menang</option>
                <option value="totalGames">Total Permainan</option>
                <option value="avgScore">Rata-rata Skor</option>
                <option value="totalWins">Total Kemenangan</option>
              </select>
            </div>
            <div class="filter-group">
              <label for="min-games">Minimal permainan:</label>
              <select id="min-games" v-model="leaderboardMinGames">
                <option value="0">Semua Pemain</option>
                <option value="3">3+ Permainan</option>
                <option value="5">5+ Permainan</option>
                <option value="10">10+ Permainan</option>
              </select>
            </div>
          </div>

          <div class="leaderboard-table-container">
            <table class="leaderboard-table">
              <thead>
                <tr>
                  <th>Rank</th>
                  <th>Pemain</th>
                  <th>Level</th>
                  <th>Win Rate</th>
                  <th>Total Permainan</th>
                  <th>Menang</th>
                  <th>Kalah</th>
                  <th>Avg. Skor</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="(player, index) in sortedLeaderboard" :key="player.name"
                  :class="getLeaderboardRowClass(index)">
                  <td class="rank-cell">#{{ index + 1 }}</td>
                  <td class="player-name-cell">{{ player.name }}</td>
                  <td class="player-level-cell">
                    <span class="player-level" :class="'level-' + player.level.toLowerCase()">
                      {{ player.level }}
                    </span>
                  </td>
                  <td class="win-rate-cell">{{ formatPercent(player.winRate) }}</td>
                  <td>{{ player.totalGames }}</td>
                  <td>{{ player.wins }}</td>
                  <td>{{ player.losses }}</td>
                  <td>{{ formatNumber(player.avgScore) }}</td>
                </tr>
              </tbody>
            </table>
            <div class="leaderboard-actions">
              <button @click="resetPlayerStats" class="reset-stats-btn">
                Reset Semua Statistik
              </button>
            </div>
          </div>


          <div class="level-explanation">
            <h3>Klasifikasi Level Pemain:</h3>
            <div class="level-grid">
              <div class="level-item">
                <span class="player-level level-newbie">Newbie</span>
                <p>Pemain baru dengan pengalaman terbatas. Win rate &lt; 30% atau kurang dari 5 permainan.</p>
              </div>
              <div class="level-item">
                <span class="player-level level-beginner">Beginner</span>
                <p>Pemain dengan pemahaman dasar. Win rate 30-45% dan minimal 5 permainan.</p>
              </div>
              <div class="level-item">
                <span class="player-level level-intermediate">Intermediate</span>
                <p>Pemain dengan pengalaman sedang. Win rate 45-60% dan minimal 10 permainan.</p>
              </div>
              <div class="level-item">
                <span class="player-level level-advanced">Advanced</span>
                <p>Pemain berpengalaman dengan strategi baik. Win rate 60-75% dan minimal 15 permainan.</p>
              </div>
              <div class="level-item">
                <span class="player-level level-pro">Pro</span>
                <p>Pemain ahli dengan penguasaan permainan. Win rate &gt; 75% dan minimal 20 permainan.</p>
              </div>
              <div class="level-item">
                <span class="player-level level-legend">Legend</span>
                <p>Pemain legendaris. Win rate &gt; 85% dengan minimal 30 permainan.</p>
              </div>
            </div>
          </div>

          <div class="player-stats-detail" v-if="selectedPlayerStats">
            <h3>Statistik Detail: {{ selectedPlayerStats.name }}</h3>
            <!-- Detailed player stats can be added here in the future -->
          </div>
        </div>
      </div>
    </div>

    <!-- History Panel - For host mode only -->
    <div v-if="!liveMode || isHost" class="history-panel">
      <div class="history-header">
        <h2 class="history-title">Riwayat Permainan</h2>
        <button @click="toggleHistoryPanel" class="toggle-history-btn">
          {{ showHistoryPanel ? 'Tutup' : 'Lihat Riwayat' }}
        </button>
      </div>

      <div v-if="showHistoryPanel" class="history-content">
        <div v-if="gameHistory.length === 0" class="no-history">
          <p>Belum ada riwayat permainan.</p>
        </div>

        <div v-else class="history-list">
          <div v-for="(history, index) in gameHistory" :key="index" class="history-item">
            <div class="history-item-header">
              <span class="history-date">{{ formatHistoryDate(history.date) }}</span>
              <div class="history-actions">
                <button @click="loadGameHistory(index)" class="load-history-btn" title="Muat Permainan">
                  <span>⏮️</span>
                </button>
                <button @click="deleteGameHistory(index)" class="delete-history-btn" title="Hapus">
                  <span>🗑️</span>
                </button>
              </div>
            </div>

            <div class="history-players">
              <div v-for="player in history.participants" :key="player.name" class="history-player">
                <span class="history-player-name">{{ player.name }}:</span>
                <span class="history-player-score" :style="{ color: getScoreColorForHistory(player.name, history) }">
                  {{ calculateHistoryTotalScore(player.name, history) }}
                </span>
              </div>
            </div>

            <div class="history-rounds">
              <span class="history-rounds-count">{{ history.scores.length }} ronde</span>
              <button @click="toggleHistoryDetails(index)" class="toggle-details-btn">
                {{ expandedHistory === index ? 'Tutup Detail' : 'Lihat Detail' }}
              </button>
            </div>

            <div v-if="expandedHistory === index" class="history-details">
              <table class="history-scores-table">
                <thead>
                  <tr>
                    <th>Ronde</th>
                    <th v-for="player in history.participants" :key="player.name">{{ player.name }}</th>
                  </tr>
                </thead>
                <tbody>
                  <tr v-for="(scoreSet, roundIndex) in history.scores" :key="roundIndex">
                    <td>{{ roundIndex + 1 }}</td>
                    <td v-for="player in history.participants" :key="player.name"
                      :class="getScoreMarkClass(player.name, scoreSet)">
                      {{ getPlayerScore(player.name, scoreSet) }}
                    </td>
                  </tr>
                </tbody>
              </table>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Confirmation Dialog -->
    <div v-if="showConfirmDialog" class="confirm-dialog-overlay">
      <div class="confirm-dialog">
        <h3>{{ confirmDialogTitle }}</h3>
        <p>{{ confirmDialogMessage }}</p>
        <div class="confirm-dialog-buttons">
          <button @click="confirmAction()" class="confirm-yes-btn">Ya</button>
          <button @click="cancelAction()" class="confirm-no-btn">Tidak</button>
        </div>
      </div>
    </div>

    <p class="copyright">© 2025 by Abi Rafdi</p>
  </div>
</template>



<script>
import { ref, onMounted, onUnmounted } from 'vue';
// Import Firebase Realtime Database
import { initializeApp } from "firebase/app";
import { getDatabase, ref as dbRef, set, onValue, off, remove, update, onDisconnect } from "firebase/database";
import { getAnalytics } from "firebase/analytics";

// Firebase configuration
const firebaseConfig = {
  apiKey: "AIzaSyA_f9cxg8zVG-5qDPgy4RJdzGHjTjLx3AI",
  authDomain: "aplikasi-remi.firebaseapp.com",
  databaseURL: "https://aplikasi-remi-default-rtdb.firebaseio.com",
  projectId: "aplikasi-remi",
  storageBucket: "aplikasi-remi.firebasestorage.app",
  messagingSenderId: "5127421335",
  appId: "1:5127421335:web:b1027de89db3650e4558d2",
  measurementId: "G-1TMTQ9PF6K"
};

// Initialize Firebase
const app = initializeApp(firebaseConfig);
const analytics = getAnalytics(app);
const database = getDatabase(app);

export default {
  data() {
    return {
      playerNameInputs: ['', '', '', ''],
      participants: [],
      showPositionGrid: false,
      positionGrid: [null, null, null, null],
      playerPositions: {},
      participantColors: [
        '#3B82F6', // Blue
        '#10B981', // Green
        '#F43F5E', // Rose
        '#8B5CF6'  // Purple
      ],
      currentScore: {},
      scoreMarks: {},
      scores: [],
      liveMode: false,
      isHost: false,
      roomId: '',
      viewers: 0,
      lastUpdateTime: null,
      connectionStatus: 'disconnected',
      dbListeners: [], // Track database listeners,
      gameHistory: [],
      showHistoryPanel: false,
      expandedHistory: null,

      // Confirmation dialog
      showConfirmDialog: false,
      confirmDialogTitle: '',
      confirmDialogMessage: '',
      pendingAction: null,
      pendingActionParams: null,
      playerStats: {},
      showLeaderboardPanel: false,
      leaderboardSortBy: 'winRate',
      leaderboardMinGames: '0',
      selectedPlayerStats: null,
    }
  },
  computed: {
    sortedLeaderboard() {
      // Convert playerStats object to array for sorting
      let players = Object.keys(this.playerStats).map(name => {
        const stats = this.playerStats[name];
        const level = this.calculatePlayerLevel(stats);

        return {
          name,
          level,
          totalGames: stats.totalGames,
          wins: stats.wins,
          losses: stats.totalGames - stats.wins,
          winRate: stats.totalGames > 0 ? stats.wins / stats.totalGames : 0,
          avgScore: stats.totalGames > 0 ? stats.totalScore / stats.totalGames : 0
        };
      });

      // Filter by minimum games
      const minGames = parseInt(this.leaderboardMinGames);
      if (minGames > 0) {
        players = players.filter(player => player.totalGames >= minGames);
      }

      // Sort by selected criterion
      if (this.leaderboardSortBy === 'winRate') {
        players.sort((a, b) => b.winRate - a.winRate);
      } else if (this.leaderboardSortBy === 'totalGames') {
        players.sort((a, b) => b.totalGames - a.totalGames);
      } else if (this.leaderboardSortBy === 'avgScore') {
        players.sort((a, b) => b.avgScore - a.avgScore);
      } else if (this.leaderboardSortBy === 'totalWins') {
        players.sort((a, b) => b.wins - a.wins);
      }

      return players;
    },
    watchUrl() {
      if (!this.roomId) return '';
      return `${window.location.origin}${window.location.pathname}?room=${this.roomId}`;
    },
    chartRange() {
      if (this.scores.length === 0) return { min: 0, max: 100 }

      const cumulativeScores = this.participants.map(participant =>
        this.calculateCumulativePoints(participant.name)
      )

      const allScores = cumulativeScores.flat()
      const min = Math.min(...allScores)
      const max = Math.max(...allScores)

      // Add some padding
      const padding = Math.max(Math.abs(min), Math.abs(max)) * 0.1
      return {
        min: min - padding,
        max: max + padding
      }
    },
    scoreRanking() {
      // Menghitung dan mengurutkan skor total setiap player
      const totalScores = this.participants.map(participant => ({
        name: participant.name,
        score: this.calculateTotalScore(participant.name)
      }));

      // Urutkan dari skor tertinggi ke terendah
      return totalScores.sort((a, b) => b.score - a.score);
    },
    highestScoringParticipant() {
      return this.scoreRanking.length > 0 ? this.scoreRanking[0].name : null;
    },
    lowestScoringParticipant() {
      return this.scoreRanking.length > 0 ? this.scoreRanking[this.scoreRanking.length - 1].name : null;
    },
    secondLowestScoringParticipant() {
      // Mengembalikan player dengan skor terendah kedua jika ada lebih dari 1 player
      return (this.scoreRanking.length > 1) ? this.scoreRanking[this.scoreRanking.length - 2].name : null;
    }
  },
  created() {
    // Load saved data when component is created
    this.loadSavedState();

    // Load history data
    this.loadHistoryFromLocalStorage();

    // Load player stats
    this.loadPlayerStatsFromLocalStorage();

    // Check URL for room parameter
    this.checkForRoomInUrl();
  },
  beforeUnmount() {
    // Detach all Firebase listeners
    this.detachAllListeners();
  },
  methods: {
    saveGameToHistory() {
      if (this.participants.length === 0 || this.scores.length === 0) {
        return;
      }

      // Update player stats
      this.updatePlayerStats();

      const gameRecord = {
        date: new Date().toISOString(),
        participants: JSON.parse(JSON.stringify(this.participants)),
        scores: JSON.parse(JSON.stringify(this.scores))
      };

      this.gameHistory.push(gameRecord);
      this.saveHistoryToLocalStorage();

      // Show notification
      alert('Permainan telah disimpan ke riwayat!');
    },

    // Add this method to view detailed stats for a specific player
    viewPlayerStats(playerName) {
      if (this.playerStats[playerName]) {
        this.selectedPlayerStats = {
          name: playerName,
          ...this.playerStats[playerName]
        };
      }
    },

    // Add Reset Stats button functionality
    resetPlayerStats() {
      if (confirm('Apakah Anda yakin ingin menghapus semua statistik pemain? Tindakan ini tidak dapat dibatalkan.')) {
        this.playerStats = {};
        this.savePlayerStatsToLocalStorage();
        this.selectedPlayerStats = null;
        alert('Statistik pemain telah direset.');
      }
    },

    // Modify executeLoadGameHistory to not update stats when loading history
    executeLoadGameHistory(index) {
      const historyItem = this.gameHistory[index];

      this.participants = JSON.parse(JSON.stringify(historyItem.participants));
      this.scores = JSON.parse(JSON.stringify(historyItem.scores));

      // Reset current input values
      this.currentScore = {};
      this.scoreMarks = {};
      this.participants.forEach(participant => {
        this.currentScore[participant.name] = null;
        this.scoreMarks[participant.name] = null;
      });

      // Save to localStorage
      this.saveParticipantsToLocalStorage();
      this.saveToLocalStorage();

      // If in live mode and is host, sync to Firebase
      if (this.liveMode && this.isHost) {
        this.syncToFirebase();
      }

      // Close history panel
      this.showHistoryPanel = false;
      this.expandedHistory = null;

      // Show notification
      alert('Permainan berhasil dimuat dari riwayat!');
    },

    toggleLeaderboardPanel() {
      this.showLeaderboardPanel = !this.showLeaderboardPanel;
    },

    calculatePlayerLevel(stats) {
      const { totalGames, wins } = stats;
      const winRate = totalGames > 0 ? wins / totalGames : 0;

      // Level classification based on win rate and experience
      if (totalGames >= 30 && winRate > 0.85) {
        return 'Legend';
      } else if (totalGames >= 20 && winRate > 0.75) {
        return 'Pro';
      } else if (totalGames >= 15 && winRate > 0.60) {
        return 'Advanced';
      } else if (totalGames >= 10 && winRate > 0.45) {
        return 'Intermediate';
      } else if (totalGames >= 5 && winRate > 0.30) {
        return 'Beginner';
      } else {
        return 'Newbie';
      }
    },

    getLeaderboardRowClass(index) {
      if (index === 0) return 'rank-first';
      if (index === 1) return 'rank-second';
      if (index === 2) return 'rank-third';
      return '';
    },

    formatPercent(value) {
      return (value * 100).toFixed(1) + '%';
    },

    formatNumber(value) {
      return value.toFixed(1);
    },

    // Update player statistics after a game ends
    updatePlayerStats() {
      // If there are no participants or scores, don't update stats
      if (!this.participants.length || !this.scores.length) return;

      // Calculate the result of the game
      const gameResults = this.participants.map(participant => ({
        name: participant.name,
        score: this.calculateTotalScore(participant.name),
        position: 0 // Will be set below
      }));

      // Sort by score (highest to lowest) to determine positions
      gameResults.sort((a, b) => b.score - a.score);

      // Assign positions (1-based, with ties handled)
      let currentPosition = 1;
      let previousScore = null;

      gameResults.forEach((result, index) => {
        if (index > 0 && result.score < previousScore) {
          currentPosition = index + 1;
        }
        result.position = currentPosition;
        previousScore = result.score;
      });

      // Create playerStats if it doesn't exist
      if (!this.playerStats) {
        this.playerStats = {};
      }

      // Update stats for each player
      gameResults.forEach(result => {
        const playerName = result.name;
        const isWinner = result.position === 1;

        // Initialize player stats if they don't exist
        if (!this.playerStats[playerName]) {
          this.playerStats[playerName] = {
            totalGames: 0,
            wins: 0,
            totalScore: 0,
            bestScore: -Infinity,
            worstScore: Infinity,
            winningStreak: 0,
            currentStreak: 0,
            lastPlayed: null
          };
        }

        const stats = this.playerStats[playerName];

        // Update general stats
        stats.totalGames += 1;
        stats.totalScore += result.score;
        stats.bestScore = Math.max(stats.bestScore, result.score);
        stats.worstScore = Math.min(stats.worstScore, result.score);
        stats.lastPlayed = new Date().toISOString();

        // Update win stats
        if (isWinner) {
          stats.wins += 1;
          stats.currentStreak += 1;
          stats.winningStreak = Math.max(stats.winningStreak, stats.currentStreak);
        } else {
          stats.currentStreak = 0;
        }
      });

      // Save updated stats to localStorage
      this.savePlayerStatsToLocalStorage();
    },

    savePlayerStatsToLocalStorage() {
      localStorage.setItem('scoreTrackerPlayerStats', JSON.stringify(this.playerStats));
    },

    loadPlayerStatsFromLocalStorage() {
      const savedStats = localStorage.getItem('scoreTrackerPlayerStats');
      if (savedStats) {
        try {
          this.playerStats = JSON.parse(savedStats);
        } catch (e) {
          console.error('Error parsing saved player stats:', e);
          this.playerStats = {};
        }
      }
    },


    toggleHistoryPanel() {
      this.showHistoryPanel = !this.showHistoryPanel;
      if (!this.showHistoryPanel) {
        this.expandedHistory = null;
      }
    },

    toggleHistoryDetails(index) {
      this.expandedHistory = this.expandedHistory === index ? null : index;
    },

    formatHistoryDate(dateString) {
      const date = new Date(dateString);
      return date.toLocaleDateString('id-ID', {
        year: 'numeric',
        month: 'long',
        day: 'numeric',
        hour: '2-digit',
        minute: '2-digit'
      });
    },

    calculateHistoryTotalScore(playerName, historyItem) {
      return historyItem.scores.reduce((total, scoreSet) => {
        const playerScore = scoreSet.find(s => s.name === playerName);
        return total + (playerScore ? playerScore.score : 0);
      }, 0);
    },

    getScoreColorForHistory(playerName, historyItem) {
      // Get all player total scores in this history item
      const playerScores = historyItem.participants.map(p => ({
        name: p.name,
        score: this.calculateHistoryTotalScore(p.name, historyItem)
      }));

      // Sort by score (highest to lowest)
      const sortedScores = playerScores.sort((a, b) => b.score - a.score);

      const highestPlayer = sortedScores[0]?.name;
      const lowestPlayer = sortedScores[sortedScores.length - 1]?.name;
      const secondLowestPlayer = sortedScores[sortedScores.length - 2]?.name;

      if (playerName === highestPlayer) {
        return '#10B981'; // Green for highest score
      } else if (playerName === lowestPlayer) {
        return '#EF4444'; // Red for lowest score
      } else if (playerName === secondLowestPlayer) {
        return '#FBBF24'; // Yellow for second lowest
      } else {
        return '#888888'; // Gray for others
      }
    },

    getPlayerScore(playerName, scoreSet) {
      const playerScore = scoreSet.find(s => s.name === playerName);
      return playerScore ? playerScore.score : 0;
    },

    getScoreMarkClass(playerName, scoreSet) {
      const playerScore = scoreSet.find(s => s.name === playerName);
      if (!playerScore) return '';

      if (playerScore.mark === 'positive') {
        return 'score-positive';
      } else if (playerScore.mark === 'negative') {
        return 'score-negative';
      }
      return '';
    },

    saveGameToHistory() {
      if (this.participants.length === 0 || this.scores.length === 0) {
        return;
      }

      const gameRecord = {
        date: new Date().toISOString(),
        participants: JSON.parse(JSON.stringify(this.participants)),
        scores: JSON.parse(JSON.stringify(this.scores))
      };

      this.gameHistory.push(gameRecord);
      this.saveHistoryToLocalStorage();

      // Show notification
      alert('Permainan telah disimpan ke riwayat!');
    },

    loadGameHistory(index) {
      if (index < 0 || index >= this.gameHistory.length) {
        return;
      }

      // If there's current game data, confirm before loading
      if (this.participants.length > 0 && this.scores.length > 0) {
        this.showConfirmDialog = true;
        this.confirmDialogTitle = 'Muat Permainan dari Riwayat';
        this.confirmDialogMessage = 'Memuat permainan dari riwayat akan menggantikan data permainan saat ini. Lanjutkan?';
        this.pendingAction = this.executeLoadGameHistory;
        this.pendingActionParams = index;
      } else {
        this.executeLoadGameHistory(index);
      }
    },

    executeLoadGameHistory(index) {
      const historyItem = this.gameHistory[index];

      this.participants = JSON.parse(JSON.stringify(historyItem.participants));
      this.scores = JSON.parse(JSON.stringify(historyItem.scores));

      // Reset current input values
      this.currentScore = {};
      this.scoreMarks = {};
      this.participants.forEach(participant => {
        this.currentScore[participant.name] = null;
        this.scoreMarks[participant.name] = null;
      });

      // Save to localStorage
      this.saveParticipantsToLocalStorage();
      this.saveToLocalStorage();

      // If in live mode and is host, sync to Firebase
      if (this.liveMode && this.isHost) {
        this.syncToFirebase();
      }

      // Close history panel
      this.showHistoryPanel = false;
      this.expandedHistory = null;

      // Show notification
      alert('Permainan berhasil dimuat dari riwayat!');
    },

    deleteGameHistory(index) {
      this.showConfirmDialog = true;
      this.confirmDialogTitle = 'Hapus Riwayat Permainan';
      this.confirmDialogMessage = 'Apakah Anda yakin ingin menghapus riwayat permainan ini?';
      this.pendingAction = this.executeDeleteGameHistory;
      this.pendingActionParams = index;
    },

    executeDeleteGameHistory(index) {
      if (index < 0 || index >= this.gameHistory.length) {
        return;
      }

      this.gameHistory.splice(index, 1);
      this.saveHistoryToLocalStorage();

      // Reset expanded detail view if needed
      if (this.expandedHistory === index) {
        this.expandedHistory = null;
      } else if (this.expandedHistory > index) {
        this.expandedHistory--;
      }
    },

    saveHistoryToLocalStorage() {
      localStorage.setItem('scoreTrackerHistory', JSON.stringify(this.gameHistory));
    },

    loadHistoryFromLocalStorage() {
      const savedHistory = localStorage.getItem('scoreTrackerHistory');
      if (savedHistory) {
        try {
          this.gameHistory = JSON.parse(savedHistory);
        } catch (e) {
          console.error('Error parsing saved history:', e);
          this.gameHistory = [];
        }
      }
    },

    // Confirmation dialog methods
    confirmAction() {
      if (this.pendingAction && typeof this.pendingAction === 'function') {
        this.pendingAction(this.pendingActionParams);
      }
      this.showConfirmDialog = false;
      this.pendingAction = null;
      this.pendingActionParams = null;
    },

    cancelAction() {
      this.showConfirmDialog = false;
      this.pendingAction = null;
      this.pendingActionParams = null;
    },

    formatTime(date) {
      if (!date) return '';

      const now = new Date();
      const diff = Math.floor((now - date) / 1000);

      if (diff < 5) {
        return 'baru saja';
      } else if (diff < 60) {
        return `${diff} detik yang lalu`;
      } else if (diff < 3600) {
        return `${Math.floor(diff / 60)} menit yang lalu`;
      } else {
        return `${Math.floor(diff / 3600)} jam yang lalu`;
      }
    },

    copyWatchUrl() {
      const input = document.querySelector('.watch-url-input');
      if (!input) return;

      input.select();
      document.execCommand('copy');

      alert('Link berhasil disalin ke clipboard!');
    },

    createLiveRoom() {
      // Check if there's a previously created room ID in localStorage
      const existingRoomId = localStorage.getItem('scoreTrackerRoomId');

      if (existingRoomId) {
        // Use the existing room ID if available
        this.roomId = existingRoomId;
      } else {
        // Generate a unique room ID (timestamp + random string)
        this.roomId = Date.now().toString(36) + Math.random().toString(36).substr(2, 5);
        // Store it in localStorage for future use
        localStorage.setItem('scoreTrackerRoomId', this.roomId);
      }

      this.isHost = true;
      this.liveMode = true;
      this.connectionStatus = 'connected';

      // Create room in Firebase
      const roomRef = dbRef(database, `rooms/${this.roomId}`);

      set(roomRef, {
        created: new Date().toISOString(),
        lastActive: new Date().toISOString(),
        host: true,
        viewers: 0,
        data: {
          participants: this.participants,
          scores: this.scores,
          lastUpdate: new Date().toISOString()
        }
      }).then(() => {
        console.log('Room created successfully');

        // Set up presence system for host
        const hostConnectionRef = dbRef(database, `rooms/${this.roomId}/hostConnected`);
        const hostRef = dbRef(database, `rooms/${this.roomId}/host`);

        // When host disconnects, update status
        onDisconnect(hostConnectionRef).set(false);
        onDisconnect(hostRef).set(false);

        // Set the host as connected
        set(hostConnectionRef, true);

        // Set up listener for viewer count
        const viewersRef = dbRef(database, `rooms/${this.roomId}/viewers`);
        const viewersListener = onValue(viewersRef, (snapshot) => {
          const data = snapshot.val();

          // Check if data is an object (individual viewers) or a number
          if (data && typeof data === 'object') {
            // Count the number of keys in the object (each key represents a viewer)
            this.viewers = Object.keys(data).length;
          } else {
            // If it's a number or null/undefined, use it directly
            this.viewers = data || 0;
          }
        });

        this.dbListeners.push({ ref: viewersRef, listener: viewersListener });

        // Start syncing data periodically
        this.syncInterval = setInterval(() => {
          this.syncToFirebase();
        }, 3000);

        // Show alert with watch URL
        this.$nextTick(() => {
          alert(`Room berhasil dibuat!\nBagikan link ini untuk ditonton: ${this.watchUrl}`);
        });
      }).catch(error => {
        console.error('Error creating room:', error);
        this.connectionStatus = 'error';
        alert('Gagal membuat room. Silakan coba lagi.');
        this.stopLiveMode();
      });
    },

    joinLiveRoom(roomId) {
      if (!roomId) return;

      this.roomId = roomId;
      this.isHost = false;
      this.liveMode = true;
      this.connectionStatus = 'connecting';

      // Check if room exists in Firebase
      const roomRef = dbRef(database, `rooms/${this.roomId}`);

      onValue(roomRef, (snapshot) => {
        const roomData = snapshot.val();

        if (!roomData) {
          this.connectionStatus = 'failed';
          console.error('Room not found');
          return;
        }

        if (!roomData.host) {
          this.connectionStatus = 'failed';
          console.error('Host disconnected');
          return;
        }

        // Register as viewer
        this.registerViewer();

        // Set up listener for data updates
        const dataRef = dbRef(database, `rooms/${this.roomId}/data`);
        const dataListener = onValue(dataRef, (dataSnapshot) => {
          const data = dataSnapshot.val();

          if (!data) {
            this.connectionStatus = 'waiting';
            return;
          }

          try {
            // Update local data
            this.participants = data.participants || [];
            this.scores = data.scores || [];
            this.lastUpdateTime = new Date(data.lastUpdate);

            // Add this - Update player stats if provided by host
            if (data.playerStats) {
              this.playerStats = data.playerStats;
            }

            this.connectionStatus = 'connected';
          } catch (e) {
            console.error('Error parsing data:', e);
            this.connectionStatus = 'error';
          }
        });

        this.dbListeners.push({ ref: dataRef, listener: dataListener });

        // Set up listener for host status
        const hostRef = dbRef(database, `rooms/${this.roomId}/host`);
        const hostListener = onValue(hostRef, (hostSnapshot) => {
          const hostConnected = hostSnapshot.val();

          if (!hostConnected) {
            this.connectionStatus = 'failed';
          }
        });

        this.dbListeners.push({ ref: hostRef, listener: hostListener });
      }, {
        onlyOnce: true
      }).catch(error => {
        console.error('Error joining room:', error);
        this.connectionStatus = 'error';
        alert('Gagal terhubung ke room. Silakan coba lagi.');
        this.stopLiveMode();
      });
    },

    syncToFirebase() {
      if (!this.roomId || !this.isHost) return;

      // Update the data in Firebase
      const dataRef = dbRef(database, `rooms/${this.roomId}/data`);

      update(dataRef, {
        participants: this.participants,
        scores: this.scores,
        playerStats: this.playerStats, // Add this line to sync player stats
        lastUpdate: new Date().toISOString()
      }).then(() => {
        // Update last active timestamp
        const roomRef = dbRef(database, `rooms/${this.roomId}`);
        update(roomRef, {
          lastActive: new Date().toISOString()
        });

        this.lastUpdateTime = new Date();
      }).catch(error => {
        console.error('Error syncing data:', error);
      });
    },

    registerViewer() {
      if (!this.roomId || this.isHost) return;

      // Generate a viewer ID
      const viewerId = Math.random().toString(36).substr(2, 9);

      // Register viewer in Firebase
      const viewerRef = dbRef(database, `rooms/${this.roomId}/viewers/${viewerId}`);
      const totalViewersRef = dbRef(database, `rooms/${this.roomId}/viewers`);

      // Update viewer count
      onValue(totalViewersRef, (snapshot) => {
        let count = snapshot.val() || 0;

        if (typeof count === 'object') {
          // If viewers is an object (with viewer IDs), count the keys
          count = Object.keys(count).length;
        } else {
          // If viewers is a number, increment it
          count = count + 1;
        }

        set(totalViewersRef, count);
      }, {
        onlyOnce: true
      });

      // Set viewer data
      set(viewerRef, {
        joined: new Date().toISOString(),
        lastActive: new Date().toISOString()
      });

      // Clean up when viewer disconnects
      onDisconnect(viewerRef).remove();

      // Set up interval to keep viewer active
      const keepAliveInterval = setInterval(() => {
        update(viewerRef, {
          lastActive: new Date().toISOString()
        });
      }, 5000);

      // Store interval to clear it when leaving
      this.viewerKeepAliveInterval = keepAliveInterval;
    },

    detachAllListeners() {
      // Clean up all Firebase listeners
      this.dbListeners.forEach(({ ref, listener }) => {
        off(ref, listener);
      });

      this.dbListeners = [];

      // Clear intervals
      if (this.syncInterval) {
        clearInterval(this.syncInterval);
        this.syncInterval = null;
      }

      if (this.viewerKeepAliveInterval) {
        clearInterval(this.viewerKeepAliveInterval);
        this.viewerKeepAliveInterval = null;
      }
    },

    stopLiveMode() {
      // Update player stats if there's data and we're the host
      if (this.isHost && this.participants.length > 0 && this.scores.length > 0) {
        this.updatePlayerStats();
      }

      // Save current game to history if there's data and we're the host
      if (this.isHost && this.participants.length > 0 && this.scores.length > 0) {
        this.saveGameToHistory();
      }

      // Detach all Firebase listeners
      this.detachAllListeners();

      if (this.isHost && this.roomId) {
        // Remove room data if host
        const roomRef = dbRef(database, `rooms/${this.roomId}`);
        update(roomRef, {
          host: false
        }).catch(error => {
          console.error('Error updating host status:', error);
        });

        // Clear the stored room ID when explicitly stopping
        localStorage.removeItem('scoreTrackerRoomId');
      }

      this.liveMode = false;
      this.roomId = '';
      this.isHost = false;
      this.connectionStatus = 'disconnected';
    },

    calculateScoreDifference() {
      if (!this.lowestScoringParticipant || !this.secondLowestScoringParticipant) return 0;

      const lowestScore = this.calculateTotalScore(this.lowestScoringParticipant);
      const secondLowestScore = this.calculateTotalScore(this.secondLowestScoringParticipant);

      return Math.abs(lowestScore - secondLowestScore);
    },
    generateRandomPositions() {
      // Validate that all player names are entered
      const validNames = this.playerNameInputs
        .map(name => name.trim())
        .filter(name => name !== '')

      if (validNames.length < 2) {
        alert('Harap masukkan minimal 2 nama player!')
        return
      }

      // Reset the position grid and available names
      this.positionGrid = [null, null, null, null]
      this.availableNames = this.shuffleArray(validNames)
      this.showPositionGrid = true
    },
    resetPositionGrid() {
      // Reset the position grid
      this.positionGrid = [null, null, null, null]
      this.availableNames = this.shuffleArray(this.playerNameInputs.filter(name => name.trim() !== ''))
    },
    selectPosition(index) {
      // If the selected grid cell is already occupied, do nothing
      if (this.positionGrid[index] !== null) return

      // If there are no available names left, do nothing
      if (this.availableNames.length === 0) return

      // Place the first available name in the selected grid cell
      const playerToPlace = this.availableNames.shift()
      this.positionGrid[index] = playerToPlace

      // Check if all players have been placed
      if (this.positionGrid.filter(cell => cell !== null).length === this.availableNames.length + this.positionGrid.filter(cell => cell !== null).length) {
        // If all positions are filled, allow saving
        // Do nothing special here, user must explicitly click save button
      }
    },

    savePlayerPositions() {
      // Save the players with their positions
      this.participants = this.positionGrid
        .filter(name => name !== null)
        .map(name => ({ name, position: this.positionGrid.indexOf(name) }))

      // Hide the position grid
      this.showPositionGrid = false

      // Initialize currentScore and scoreMarks for new participants
      this.currentScore = {}
      this.scoreMarks = {}
      this.participants.forEach(participant => {
        this.currentScore[participant.name] = null
        this.scoreMarks[participant.name] = null
      })

      // Save participant names to localStorage
      this.saveParticipantsToLocalStorage()
    },

    shuffleArray(array) {
      // Fisher-Yates shuffle algorithm
      const shuffled = [...array]
      for (let i = shuffled.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1));
        [shuffled[i], shuffled[j]] = [shuffled[j], shuffled[i]]
      }
      return shuffled
    },

    savePlayerNames() {
      // If using position grid, ensure all positions are filled
      if (this.showPositionGrid) {
        const unfilledPositions = this.positionGrid.filter(cell => cell === null)
        if (unfilledPositions.length > 0) {
          alert('Harap isi semua posisi player!')
          return
        }
      }

      // Use position grid names if available, otherwise use original inputs
      const validNames = this.showPositionGrid
        ? this.positionGrid.filter(name => name !== null)
        : this.playerNameInputs.filter(name => name.trim() !== '')

      if (validNames.length < 2) {
        alert('Harap masukkan minimal 2 nama player!')
        return
      }

      // Save participants
      this.participants = validNames.map(name => ({ name }))

      // Reset position grid and flags
      this.showPositionGrid = false
      this.positionGrid = [null, null, null, null]

      // Initialize currentScore and scoreMarks for new participants
      this.currentScore = {}
      this.scoreMarks = {}
      this.participants.forEach(participant => {
        this.currentScore[participant.name] = null
        this.scoreMarks[participant.name] = null
      })

      // Save participant names to localStorage
      this.saveParticipantsToLocalStorage()

      // If in live mode and is host, sync to Firebase
      if (this.liveMode && this.isHost) {
        this.syncToFirebase();
      }
    },

    saveParticipantsToLocalStorage() {
      // Save participant names to localStorage
      localStorage.setItem('scoreTrackerParticipants', JSON.stringify(
        this.participants.map(p => p.name)
      ))
    },

    loadSavedState() {
      // Try to load participants from localStorage
      const savedParticipants = localStorage.getItem('scoreTrackerParticipants')
      const savedScores = localStorage.getItem('scoreTrackerData')

      if (savedParticipants) {
        // Restore participants
        const participantNames = JSON.parse(savedParticipants)
        this.participants = participantNames.map(name => ({ name }))

        // Initialize currentScore and scoreMarks
        this.currentScore = {}
        this.scoreMarks = {}
        this.participants.forEach(participant => {
          this.currentScore[participant.name] = null
          this.scoreMarks[participant.name] = null
        })
      }

      // Load scores if available
      if (savedScores) {
        try {
          this.scores = JSON.parse(savedScores)
        } catch (e) {
          console.error('Error parsing saved scores:', e)
          this.scores = []
        }
      }
    },

    addScores() {
      // All the original code from your addScores method
      const missingScores = this.participants.some(participant => {
        return this.currentScore[participant.name] === null ||
          this.currentScore[participant.name] === undefined ||
          this.currentScore[participant.name] === '';
      });

      if (missingScores) {
        alert('Harap isi semua skor player sebelum menambahkan!');
        return;
      }

      const newScoreEntry = this.participants.map(participant => ({
        name: participant.name,
        score: this.currentScore[participant.name] || 0,
        mark: this.scoreMarks[participant.name]
      }));

      this.scores.push(newScoreEntry);

      // Save to localStorage
      this.saveToLocalStorage();

      // If in live mode and is host, sync to Firebase
      if (this.liveMode && this.isHost) {
        this.syncToFirebase();
      }

      // Reset input fields and marks
      this.participants.forEach(participant => {
        this.currentScore[participant.name] = null;
        this.scoreMarks[participant.name] = null;
      });
    },

    markScorePositive(participantName) {
      // Jika sudah positif, batalkan; jika negatif atau belum ditandai, ubah ke positif
      if (this.scoreMarks[participantName] === 'positive') {
        this.scoreMarks[participantName] = null;
      } else {
        this.scoreMarks[participantName] = 'positive';
      }
    },

    markScoreNegative(participantName) {
      // Jika sudah negatif, batalkan; jika positif atau belum ditandai, ubah ke negatif
      if (this.scoreMarks[participantName] === 'negative') {
        this.scoreMarks[participantName] = null;
      } else {
        this.scoreMarks[participantName] = 'negative';
      }
    },

    calculateTotalScore(participantName) {
      return this.scores.reduce((total, scoreSet) => {
        const participantScore = scoreSet.find(s => s.name === participantName)
        return total + (participantScore ? participantScore.score : 0)
      }, 0)
    },

    calculateCumulativePoints(participantName) {
      const cumulativePoints = []
      let runningTotal = 0

      this.scores.forEach(scoreSet => {
        const participantScore = scoreSet.find(s => s.name === participantName)
        runningTotal += participantScore ? participantScore.score : 0
        cumulativePoints.push(runningTotal)
      })

      return cumulativePoints
    },

    calculatePointPosition(participantName, point) {
      const range = this.chartRange
      const percentage = ((point - range.min) / (range.max - range.min)) * 100
      return `${percentage}%`
    },

    generatePolylinePoints(participantName) {
      const cumulativePoints = this.calculateCumulativePoints(participantName)
      if (cumulativePoints.length <= 1) {
        // Handle case with only one point
        const point = cumulativePoints[0] || 0
        const range = this.chartRange
        const y = 100 - ((point - range.min) / (range.max - range.min)) * 100
        return `0,${y} 100,${y}`
      }

      const range = this.chartRange

      return cumulativePoints.map((point, index) => {
        const x = (index / (this.scores.length - 1)) * 100
        const y = 100 - ((point - range.min) / (range.max - range.min)) * 100
        return `${x},${y}`
      }).join(' ')
    },

    calculateYAxisValue(position) {
      const range = this.chartRange
      // Interpolate between min and max
      return range.max - (position - 1) * ((range.max - range.min) / 4)
    },

    formatYAxisLabel(value) {
      // Round to 2 decimal places
      return Math.round(value * 100) / 100
    },

    getChartBackgroundColor(index) {
      // Light, transparent version of participant color
      const color = this.getParticipantColor(this.participants[index].name)
      return `${color}11`
    },

    getParticipantColor(participantName) {
      if (!this.scores.length) return '#888888'; // Default warna abu-abu jika belum ada skor

      if (participantName === this.highestScoringParticipant) {
        return '#10B981'; // Hijau untuk skor tertinggi
      } else if (participantName === this.lowestScoringParticipant) {
        return '#EF4444'; // Merah untuk skor terendah
      } else if (participantName === this.secondLowestScoringParticipant) {
        return '#FBBF24'; // Kuning untuk skor terendah kedua
      } else {
        return '#888888'; // Abu-abu untuk yang lainnya
      }
    },

    getScoreColor(participantName) {
      if (!this.scores.length) return '#888888'; // Default warna abu-abu jika belum ada skor

      if (participantName === this.highestScoringParticipant) {
        return '#10B981'; // Hijau untuk skor tertinggi
      } else if (participantName === this.lowestScoringParticipant) {
        return '#EF4444'; // Merah untuk skor terendah
      } else if (participantName === this.secondLowestScoringParticipant) {
        return '#FBBF24'; // Kuning untuk skor terendah kedua
      } else {
        return '#888888'; // Abu-abu untuk yang lainnya
      }
    },

    saveToLocalStorage() {
      // Save the scores array to localStorage
      localStorage.setItem('scoreTrackerData', JSON.stringify(this.scores));

      // If in live mode and is host, also sync to Firebase
      if (this.liveMode && this.isHost) {
        this.syncToFirebase();
      }
    },

    checkForRoomInUrl() {
      const urlParams = new URLSearchParams(window.location.search);
      const roomId = urlParams.get('room');

      if (roomId) {
        // Set a short timeout to ensure Firebase is fully initialized
        setTimeout(() => {
          this.joinLiveRoom(roomId);
        }, 1000);
      }
    },

    resetData() {
      if (confirm('Apakah Anda yakin ingin menghapus semua data skor?')) {
        // Update player stats if there's data
        if (this.participants.length > 0 && this.scores.length > 0) {
          this.updatePlayerStats();
        }

        // Save current game to history if there's data
        if (this.participants.length > 0 && this.scores.length > 0) {
          this.saveGameToHistory();
        }

        // Clear scores array
        this.scores = []

        // Clear localStorage
        localStorage.removeItem('scoreTrackerData')
        localStorage.removeItem('scoreTrackerParticipants')

        // Also clear room ID if we're resetting everything
        if (!this.liveMode || confirm('Apakah Anda juga ingin menghapus room live score saat ini?')) {
          localStorage.removeItem('scoreTrackerRoomId')

          // If in live mode, stop it
          if (this.liveMode) {
            this.stopLiveMode();
          }
        }

        // Reset participants and input fields
        this.participants = []
        this.playerNameInputs = ['', '', '', '']
        this.currentScore = {}
        this.scoreMarks = {}

        // If in live mode and is host, sync cleared data to Firebase
        if (this.liveMode && this.isHost) {
          this.syncToFirebase();
        }
      }
    },

    async downloadReport() {
      if (this.scores.length === 0) {
        alert('Belum ada data skor untuk diunduh!')
        return
      }

      // Menghitung ukuran canvas yang diperlukan berdasarkan jumlah data
      const baseHeight = 500 // Tinggi dasar untuk header dan tabel
      const detailTableHeight = 170 + (this.scores.length * 30) // Tinggi untuk tabel detail
      const chartHeight = 250 // Tinggi per grafik
      const chartSpacing = 320 // Spasi antar grafik
      const totalChartHeight = this.participants.length * chartSpacing

      // Total tinggi canvas
      const canvasHeight = baseHeight + detailTableHeight + totalChartHeight + 100

      // Sesuaikan ukuran canvas
      const canvas = this.$refs.reportCanvas
      canvas.width = 1000
      canvas.height = canvasHeight

      const ctx = canvas.getContext('2d')

      // Bersihkan canvas
      ctx.fillStyle = 'white'
      ctx.fillRect(0, 0, canvas.width, canvas.height)

      // Tambahkan judul
      ctx.fillStyle = '#333'
      ctx.font = 'bold 30px Arial'
      ctx.textAlign = 'center'
      ctx.fillText('LAPORAN SKOR PESERTA', canvas.width / 2, 50)

      // Tambahkan tanggal laporan
      const today = new Date()
      const dateString = today.toLocaleDateString('id-ID', {
        year: 'numeric',
        month: 'long',
        day: 'numeric'
      })
      ctx.font = '16px Arial'
      ctx.fillText(dateString, canvas.width / 2, 80)

      // Tambahkan tabel skor total
      ctx.font = 'bold 18px Arial'
      ctx.textAlign = 'center'
      ctx.fillText('Skor Total Player', canvas.width / 2, 130)

      // Gambar tabel skor total
      const totalScoreY = 160
      ctx.beginPath()
      ctx.rect(150, totalScoreY, 700, 50)
      ctx.stroke()

      // Header tabel
      this.participants.forEach((participant, index) => {
        const x = 150 + (700 / this.participants.length) * index
        const width = 700 / this.participants.length

        // Buat kolom
        ctx.beginPath()
        if (index > 0) {
          ctx.moveTo(x, totalScoreY)
          ctx.lineTo(x, totalScoreY + 50)
          ctx.stroke()
        }

        // Isi nama player
        ctx.font = 'bold 16px Arial'
        ctx.textAlign = 'center'
        ctx.fillText(participant.name, x + width / 2, totalScoreY + 25)
      })

      // Baris skor total
      const totalScoreRowY = totalScoreY + 50
      ctx.beginPath()
      ctx.rect(150, totalScoreRowY, 700, 50)
      ctx.stroke()

      // Isi skor total
      this.participants.forEach((participant, index) => {
        const x = 150 + (700 / this.participants.length) * index
        const width = 700 / this.participants.length

        // Buat kolom
        ctx.beginPath()
        if (index > 0) {
          ctx.moveTo(x, totalScoreRowY)
          ctx.lineTo(x, totalScoreRowY + 50)
          ctx.stroke()
        }

        // Isi skor dengan warna sesuai peringkat
        const totalScore = this.calculateTotalScore(participant.name)
        ctx.font = 'bold 20px Arial'
        ctx.textAlign = 'center'

        // Tentukan warna berdasarkan peringkat
        if (participant.name === this.highestScoringParticipant) {
          ctx.fillStyle = '#10B981'; // Hijau untuk skor tertinggi
        } else if (participant.name === this.lowestScoringParticipant) {
          ctx.fillStyle = '#EF4444'; // Merah untuk skor terendah
        } else {
          ctx.fillStyle = '#888888'; // Abu-abu untuk yang lainnya
        }

        ctx.fillText(totalScore.toString(), x + width / 2, totalScoreRowY + 30)
        ctx.fillStyle = '#333'
      })

      // Tambahkan tabel skor per ronde
      ctx.font = 'bold 18px Arial'
      ctx.textAlign = 'center'
      ctx.fillText('Detail Skor Per Ronde', canvas.width / 2, totalScoreRowY + 100)

      // Gambar tabel skor detail
      const detailY = totalScoreRowY + 130
      const headerHeight = 40
      const rowHeight = 30
      const totalHeight = headerHeight + (this.scores.length * rowHeight)

      // Header kolom ronde
      ctx.beginPath()
      ctx.rect(50, detailY, 100, headerHeight)
      ctx.stroke()
      ctx.font = 'bold 14px Arial'
      ctx.textAlign = 'center'
      ctx.fillText('Ronde', 100, detailY + 25)

      // Header kolom player
      this.participants.forEach((participant, index) => {
        const x = 150 + (700 / this.participants.length) * index
        const width = 700 / this.participants.length

        ctx.beginPath()
        ctx.rect(x, detailY, width, headerHeight)
        ctx.stroke()
        ctx.fillText(participant.name, x + width / 2, detailY + 25)
      })

      // Baris data per ronde
      this.scores.forEach((scoreSet, roundIndex) => {
        const y = detailY + headerHeight + (roundIndex * rowHeight)

        // Kolom nomor ronde
        ctx.beginPath()
        ctx.rect(50, y, 100, rowHeight)
        ctx.stroke()
        ctx.font = '14px Arial'
        ctx.textAlign = 'center'
        ctx.fillText(`Ronde ${roundIndex + 1}`, 100, y + 20)

        // Kolom skor per player
        this.participants.forEach((participant, partIndex) => {
          const x = 150 + (700 / this.participants.length) * partIndex
          const width = 700 / this.participants.length

          ctx.beginPath()
          ctx.rect(x, y, width, rowHeight)
          ctx.stroke()

          const participantScore = scoreSet.find(s => s.name === participant.name)
          const score = participantScore ? participantScore.score : 0

          // Tentukan warna berdasarkan penanda (positif/negatif)
          if (participantScore && participantScore.mark === 'positive') {
            // Kotak hijau dengan teks putih
            ctx.fillStyle = '#10B981'; // Hijau untuk latar belakang
            ctx.fillRect(x, y, width, rowHeight); // Isi kotak dengan warna hijau
            ctx.fillStyle = 'white'; // Warna teks putih
          } else if (participantScore && participantScore.mark === 'negative') {
            // Kotak merah dengan teks putih
            ctx.fillStyle = '#EF4444'; // Merah untuk latar belakang
            ctx.fillRect(x, y, width, rowHeight); // Isi kotak dengan warna merah
            ctx.fillStyle = 'white'; // Warna teks putih
          } else {
            ctx.fillStyle = '#333333'; // Hitam (default) untuk skor tanpa penanda
          }

          ctx.fillText(score.toString(), x + width / 2, y + 20)
          ctx.fillStyle = '#333'; // Reset warna untuk elemen berikutnya
        })
      })

      // Tambahkan keterangan warna di BAWAH tabel
      const legendY = detailY + totalHeight + 30;

      ctx.font = '14px Arial'
      ctx.textAlign = 'left'

      // Keterangan untuk warna hijau
      ctx.fillStyle = '#10B981'
      ctx.fillRect(150, legendY, 15, 15)
      ctx.fillStyle = '#333'
      ctx.fillText('Sang Penutup', 175, legendY + 13)

      // Keterangan untuk warna merah
      ctx.fillStyle = '#EF4444'
      ctx.fillRect(350, legendY, 15, 15)
      ctx.fillStyle = '#333'
      ctx.fillText('Kena Tutup', 375, legendY + 13)

      // Gambar grafik untuk setiap player
      const chartStartY = detailY + totalHeight + 150

      ctx.font = 'bold 18px Arial'
      ctx.textAlign = 'center'
      ctx.fillText('Grafik Perkembangan Skor', canvas.width / 2, chartStartY - 40)

      // Gambar grafik untuk setiap player
      for (let i = 0; i < this.participants.length; i++) {
        const participant = this.participants[i]
        const y = chartStartY + (i * chartSpacing)

        // Judul grafik
        ctx.font = 'bold 16px Arial'
        ctx.fillText(`Grafik Skor ${participant.name}`, canvas.width / 2, y - 20)

        // Container grafik
        ctx.beginPath()
        ctx.rect(100, y, 800, chartHeight)
        ctx.stroke()

        // Menggambar sumbu x dan y
        ctx.beginPath()
        ctx.moveTo(100, y + chartHeight / 2)  // sumbu x (garis tengah)
        ctx.lineTo(900, y + chartHeight / 2)
        ctx.setLineDash([5, 5])
        ctx.stroke()
        ctx.setLineDash([])

        ctx.beginPath()
        ctx.moveTo(100, y)  // sumbu y
        ctx.lineTo(100, y + chartHeight)
        ctx.stroke()

        // Label sumbu x (ronde)
        if (this.scores.length > 0) {
          const xStep = 800 / (this.scores.length - 1 || 1)
          for (let j = 0; j < this.scores.length; j++) {
            const xPos = 100 + j * xStep
            ctx.font = '12px Arial'
            ctx.textAlign = 'center'
            ctx.fillText(`${j + 1}`, xPos, y + chartHeight + 15)
          }
        }

        // Gambar grafik
        const points = this.calculateCumulativePoints(participant.name)
        if (points.length > 0) {
          const range = this.chartRange
          const yScale = chartHeight / (range.max - range.min)

          // Tentukan warna grafik berdasarkan peringkat
          let lineColor;
          if (participant.name === this.highestScoringParticipant) {
            lineColor = '#10B981'; // Hijau untuk skor tertinggi
          } else if (participant.name === this.lowestScoringParticipant) {
            lineColor = '#EF4444'; // Merah untuk skor terendah
          } else if (participant.name === this.secondLowestScoringParticipant) {
            lineColor = '#FBBF24'; // Kuning untuk skor terendah kedua
          } else {
            lineColor = '#888888'; // Abu-abu untuk yang lainnya
          }

          // Gambar garis
          ctx.beginPath()
          ctx.strokeStyle = lineColor
          ctx.lineWidth = 3

          for (let j = 0; j < points.length; j++) {
            const xPos = 100 + (j * (800 / (points.length - 1 || 1)))
            const yPos = y + chartHeight - ((points[j] - range.min) * yScale)

            if (j === 0) {
              ctx.moveTo(xPos, yPos)
            } else {
              ctx.lineTo(xPos, yPos)
            }
          }
          ctx.stroke()

          // Gambar titik-titik
          for (let j = 0; j < points.length; j++) {
            const xPos = 100 + (j * (800 / (points.length - 1 || 1)))
            const yPos = y + chartHeight - ((points[j] - range.min) * yScale)

            ctx.beginPath()
            ctx.fillStyle = lineColor
            ctx.arc(xPos, yPos, 5, 0, 2 * Math.PI)
            ctx.fill()

            // Tambahkan label nilai
            ctx.fillStyle = '#333'
            ctx.font = '12px Arial'
            ctx.textAlign = 'center'
            ctx.fillText(points[j].toString(), xPos, yPos - 10)
          }
        }

        // Reset gaya
        ctx.strokeStyle = '#000'
        ctx.lineWidth = 1
      }

      if (this.highestScoringParticipant) {
        ctx.font = 'bold 18px Arial'
        ctx.textAlign = 'center'
        ctx.fillStyle = '#10B981' // Warna hijau
        ctx.fillText(`Selamat kepada sang pemenang ${this.highestScoringParticipant}!`, canvas.width / 2, chartStartY + (this.participants.length * chartSpacing) + 50)

        if (this.lowestScoringParticipant) {
          ctx.font = 'bold 16px Arial'
          ctx.fillStyle = '#EF4444' // Warna merah
          ctx.fillText(`Tetap bersabar dan jadilah mesin pengocok yang baik untuk ${this.lowestScoringParticipant}!`, canvas.width / 2, chartStartY + (this.participants.length * chartSpacing) + 80)
        }
      }

      // Convert canvas to image and download
      try {
        const dataUrl = canvas.toDataURL('image/jpeg', 0.95) // Gunakan kualitas tinggi (95%)
        const link = document.createElement('a')
        link.href = dataUrl
        link.download = `Laporan_Skor_Player_${dateString.replace(/\s/g, '_')}.jpg`
        link.click()
      } catch (error) {
        console.error('Error generating report:', error)
        alert('Terjadi kesalahan saat membuat laporan. Silakan coba lagi.')
      }
    }
  }
}
</script>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Poppins:wght@300;400;500;600;700&display=swap');

.thank-you-container {
  text-align: center;
  padding: 40px;
  margin: 30px auto;
  background-color: #b8bfc7;
  border-radius: 10px;
  max-width: 600px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}

.viewer-leaderboard {
  margin-top: 2rem;
  padding: 1rem;
  background-color: #f9fafb;
  border-radius: 0.5rem;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
}

.viewer-section-title {
  font-size: 1.25rem;
  color: #374151;
  margin-top: 0;
  margin-bottom: 1rem;
  text-align: center;
}

.viewer-leaderboard-content {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.viewer-leaderboard-filters {
  display: flex;
  flex-wrap: wrap;
  gap: 1rem;
  margin-bottom: 0.5rem;
  padding: 0.5rem;
  background-color: #f3f4f6;
  border-radius: 0.375rem;
  justify-content: center;
}

.viewer-leaderboard-filters .filter-group {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.viewer-leaderboard-filters label {
  font-size: 0.75rem;
  color: #4b5563;
}

.viewer-leaderboard-filters select {
  padding: 0.25rem 0.5rem;
  border: 1px solid #d1d5db;
  border-radius: 0.25rem;
  background-color: white;
  font-size: 0.75rem;
  color: #374151;
}

.viewer-leaderboard-table-container {
  overflow-x: auto;
  margin-bottom: 0.75rem;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.05);
  border-radius: 0.375rem;
  background-color: white;
}

.viewer-leaderboard-table {
  color: black;
  width: 100%;
  border-collapse: collapse;
  font-size: 0.75rem;
}

.viewer-leaderboard-table th,
.viewer-leaderboard-table td {
  padding: 0.5rem;
  text-align: center;
  border-bottom: 1px solid #e5e7eb;
}

.viewer-leaderboard-table th {
  background-color: #f9fafb;
  font-weight: 600;
  color: #374151;
  position: sticky;
  top: 0;
}

.viewer-leaderboard-table tr:last-child td {
  border-bottom: none;
}

.viewer-leaderboard-table .player-name-cell {
  text-align: left;
}

.viewer-level-legend {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
  justify-content: center;
  align-items: center;
  padding: 0.5rem;
  background-color: white;
  border-radius: 0.375rem;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.05);
}

.level-legend-title {
  font-size: 0.75rem;
  color: #4b5563;
  margin-right: 0.5rem;
}

.level-legend-items {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
}

.level-legend-item .player-level {
  font-size: 0.65rem;
  padding: 0.15rem 0.35rem;
}

.viewer-mode {
  max-width: 100%;
  padding: 10px;
}

.viewer-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 20px;
  background-color: #b8bfc7;
  padding: 10px;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.viewer-header .title {
  margin: 0;
  font-size: 1.5rem;
}

.viewer-header .stop-live-btn.small {
  font-size: 0.9rem;
  padding: 5px 10px;
  background-color: #6c757d;
}

.viewer-scores-summary {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  gap: 15px;
  margin: 20px 0;
  padding: 15px;
  background-color: #f8f9fa;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.viewer-participant-total {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 10px 15px;
  border-radius: 30px;
  background-color: white;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
}

.viewer-participant-total .participant-name {
  font-weight: bold;
}

.viewer-participant-total .participant-total-score {
  font-size: 1.4rem;
  font-weight: bold;
}

.viewer-charts {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 20px;
  margin-top: 30px;
}

.leaderboard-actions {
  margin-top: 1.5rem;
  display: flex;
  justify-content: flex-end;
  padding: 1em;
}

.reset-stats-btn {
  background-color: #ef4444;
  color: white;
  border: none;
  border-radius: 0.375rem;
  padding: 0.5rem 1rem;
  font-size: 0.875rem;
  cursor: pointer;
  transition: background-color 0.2s;
}

.reset-stats-btn:hover {
  background-color: #dc2626;
}

.viewer-chart-container {
  background-color: white;
  padding: 15px;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}


.status-indicator {
  display: inline-block;
  width: 12px;
  height: 12px;
  border-radius: 50%;
  margin-right: 5px;
}

.status-indicator.connected {
  background-color: #10B981;
  /* Hijau */
  box-shadow: 0 0 8px #10B981;
}

.status-indicator.connecting,
.status-indicator.waiting {
  background-color: #F59E0B;
  /* Kuning */
  box-shadow: 0 0 8px #F59E0B;
  animation: pulse 1.5s infinite;
}

.status-indicator.failed,
.status-indicator.error {
  background-color: #EF4444;
  /* Merah */
  box-shadow: 0 0 8px #EF4444;
}

.status-indicator.disconnected {
  background-color: #6B7280;
  /* Abu-abu */
}

@keyframes pulse {
  0% {
    opacity: 1;
  }

  50% {
    opacity: 0.5;
  }

  100% {
    opacity: 1;
  }
}

.live-score-controls {
  margin: 20px 0;
  padding: 15px;
  border-radius: 8px;
  background-color: #f8f9fa;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
}

.create-room-btn,
.stop-live-btn {
  background-color: #3B82F6;
  color: white;
  border: none;
  border-radius: 4px;
  padding: 10px 15px;
  font-size: 14px;
  font-weight: bold;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background-color 0.2s;
}

.create-room-btn:hover {
  background-color: #2563EB;
}

.stop-live-btn {
  background-color: #EF4444;
}

.stop-live-btn:hover {
  background-color: #DC2626;
}

.btn-icon {
  margin-right: 8px;
  font-size: 16px;
}

.live-mode-active {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.live-status {
  display: flex;
  align-items: center;
  gap: 10px;
  font-size: 14px;
  font-weight: 500;
}

.status-indicator {
  display: inline-block;
  width: 12px;
  height: 12px;
  border-radius: 50%;
  background-color: #9CA3AF;
  /* Default gray */
}

.status-indicator.connected {
  background-color: #10B981;
  /* Green */
  animation: pulse 2s infinite;
}

.status-indicator.connecting,
.status-indicator.waiting {
  background-color: #F59E0B;
  /* Amber */
  animation: pulse 1s infinite;
}

.status-indicator.failed,
.status-indicator.error,
.status-indicator.disconnected {
  background-color: #EF4444;
  /* Red */
}

@keyframes pulse {
  0% {
    opacity: 1;
  }

  50% {
    opacity: 0.5;
  }

  100% {
    opacity: 1;
  }
}

.last-update {
  color: #6B7280;
  font-size: 12px;
  margin-left: auto;
}

.share-link {
  display: flex;
  align-items: center;
  gap: 10px;
  margin: 10px 0;
}

.watch-url-input {
  flex: 1;
  padding: 8px 12px;
  border: 1px solid #D1D5DB;
  border-radius: 4px;
  font-size: 14px;
  background-color: #F3F4F6;
  color: #374151;
}

.copy-url-btn {
  background-color: #6B7280;
  color: white;
  border: none;
  border-radius: 4px;
  padding: 8px 12px;
  font-size: 13px;
  cursor: pointer;
  transition: background-color 0.2s;
}

.copy-url-btn:hover {
  background-color: #4B5563;
}

.viewers-count {
  font-size: 13px;
  color: #6B7280;
  display: flex;
  align-items: center;
  margin-left: 10px;
}

.viewers-count::before {
  content: '👁️';
  margin-right: 5px;
}

.viewer-display {
  background-color: #6a87ad;
  border-left: 4px solid #3B82F6;
  padding: 12px 15px;
  margin: 15px 0;
  border-radius: 0 4px 4px 0;
}

.connection-status {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.status-text {
  color: white !important;
  font-size: 14px;
  font-weight: 500;
  display: flex;
  align-items: center;
}

.status-text::before {
  content: '';
  display: inline-block;
  width: 10px;
  height: 10px;
  border-radius: 50%;
  margin-right: 8px;
}

/* Leaderboard Panel Styles */
.leaderboard-panel {
  margin-top: 2rem;
  border: 1px solid #e5e7eb;
  border-radius: 0.5rem;
  overflow: hidden;
  background-color: #f9fafb;
}

.leaderboard-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0.75rem 1rem;
  background-color: #f3f4f6;
  border-bottom: 1px solid #e5e7eb;
}

.leaderboard-title {
  margin: 0;
  font-size: 1.25rem;
  color: #374151;
}

.toggle-leaderboard-btn {
  background-color: #8b5cf6;
  color: white;
  border: none;
  border-radius: 0.375rem;
  padding: 0.5rem 1rem;
  font-size: 0.875rem;
  cursor: pointer;
  transition: background-color 0.2s;
}

.toggle-leaderboard-btn:hover {
  background-color: #7c3aed;
}

.leaderboard-content {
  padding: 1rem;
}

.no-leaderboard {
  text-align: center;
  padding: 2rem;
  color: #6b7280;
}

.leaderboard-filters {
  display: flex;
  flex-wrap: wrap;
  gap: 1rem;
  margin-bottom: 1rem;
  background-color: #f3f4f6;
  padding: 0.75rem;
  border-radius: 0.375rem;
}

.filter-group {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.filter-group label {
  font-size: 0.875rem;
  color: #4b5563;
}

.filter-group select {
  padding: 0.375rem 0.75rem;
  border: 1px solid #d1d5db;
  border-radius: 0.25rem;
  background-color: white;
  font-size: 0.875rem;
  color: #374151;
}

.leaderboard-table-container {
  overflow-x: auto;
  margin-bottom: 1.5rem;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
  border-radius: 0.5rem;
}

.leaderboard-table {
  color: black;
  width: 100%;
  border-collapse: collapse;
  background-color: white;
  font-size: 0.875rem;
}

.leaderboard-table th,
.leaderboard-table td {
  padding: 0.75rem;
  text-align: center;
  border-bottom: 1px solid #e5e7eb;
}

.leaderboard-table th {
  background-color: #f9fafb;
  font-weight: 600;
  color: #374151;
  position: sticky;
  top: 0;
}

.leaderboard-table tr:last-child td {
  border-bottom: none;
}

.leaderboard-table tbody tr:hover {
  background-color: #f9fafb;
}

.rank-cell {
  font-weight: 600;
}

.player-name-cell {
  font-weight: 500;
  text-align: left;
}

.player-level-cell {
  text-align: center;
}

.win-rate-cell {
  font-weight: 500;
}

.player-level {
  display: inline-block;
  padding: 0.25rem 0.5rem;
  border-radius: 9999px;
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.level-newbie {
  background-color: #e5e7eb;
  color: #4b5563;
}

.level-beginner {
  background-color: #bfdbfe;
  color: #1e40af;
}

.level-intermediate {
  background-color: #a7f3d0;
  color: #065f46;
}

.level-advanced {
  background-color: #fde68a;
  color: #92400e;
}

.level-pro {
  background-color: #fed7aa;
  color: #9a3412;
}

.level-legend {
  background-color: #fbcfe8;
  color: #831843;
}

.rank-first td {
  background-color: rgba(255, 215, 0, 0.1);
}

.rank-second td {
  background-color: rgba(192, 192, 192, 0.1);
}

.rank-third td {
  background-color: rgba(205, 127, 50, 0.1);
}

.level-explanation {
  background-color: white;
  border-radius: 0.5rem;
  padding: 1rem;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
}

.level-explanation h3 {
  margin-top: 0;
  margin-bottom: 1rem;
  font-size: 1.125rem;
  color: #374151;
}

.level-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(240px, 1fr));
  gap: 1rem;
}

.level-item {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  padding: 0.75rem;
  border-radius: 0.375rem;
  background-color: #f9fafb;
  border: 1px solid #e5e7eb;
}

.level-item .player-level {
  align-self: flex-start;
}

.level-item p {
  margin: 0;
  font-size: 0.875rem;
  color: #4b5563;
}

/* Responsive styles */
@media (max-width: 768px) {
  .viewer-leaderboard-filters {
    flex-direction: column;
    align-items: stretch;
    gap: 0.5rem;
  }

  .viewer-leaderboard-filters .filter-group {
    justify-content: space-between;
  }

  .viewer-leaderboard-table th,
  .viewer-leaderboard-table td {
    padding: 0.35rem 0.25rem;
    font-size: 0.7rem;
  }

  .level-legend-items {
    justify-content: center;
  }

  .leaderboard-filters {
    flex-direction: column;
    align-items: flex-start;
  }

  .filter-group {
    width: 100%;
  }

  .filter-group select {
    flex-grow: 1;
  }

  .leaderboard-table th,
  .leaderboard-table td {
    padding: 0.5rem;
  }

  .level-grid {
    grid-template-columns: 1fr;
  }
}

/* Manual game save button */
.save-history-btn {
  background-color: #8b5cf6;
  color: white;
  border: none;
  border-radius: 0.375rem;
  padding: 0.75rem 1.5rem;
  font-size: 1rem;
  cursor: pointer;
  transition: background-color 0.2s;
}

.save-history-btn:hover {
  background-color: #7c3aed;
}

.status-text.connected::before {
  background-color: #10B981;
  animation: pulse 2s infinite;
}

.status-text.connecting::before,
.status-text.waiting::before {
  background-color: #F59E0B;
  animation: pulse 1s infinite;
}

.status-text.failed::before,
.status-text.error::before {
  background-color: #EF4444;
}

.copyright {
  font-size: 14px;
  color: #666;
  text-align: center;
  margin-top: 70px;
  font-style: italic;
}

.score-tracker-container {
  width: 100%;
  max-width: 1400px;
  margin: 0 auto;
  padding: 20px;
  background: linear-gradient(145deg, #f0f0f0, #ffffff);
  border-radius: 20px;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.1);
  overflow-x: hidden;
  box-sizing: border-box;
  font-family: 'Poppins', sans-serif;
  transition: all 0.5s cubic-bezier(0.25, 0.46, 0.45, 0.94);
}

.score-tracker-container:hover {
  box-shadow: 0 15px 35px rgba(0, 0, 0, 0.15);
  transform: translateY(-5px);
}

.title {
  text-align: center;
  color: #333;
  margin-bottom: 30px;
  font-size: 36px;
  font-weight: 700;
  background: linear-gradient(90deg, #3B82F6, #8B5CF6);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  text-shadow: 0 5px 15px rgba(59, 130, 246, 0.2);
  letter-spacing: 1px;
  transition: transform 0.3s ease;
}

.title:hover {
  transform: scale(1.03);
}

.player-name-input {
  background: #fff;
  padding: 30px;
  border-radius: 16px;
  margin-bottom: 40px;
  box-shadow: 0 8px 20px rgba(0, 0, 0, 0.06);
  transform: translateY(0);
  transition: all 0.4s cubic-bezier(0.175, 0.885, 0.32, 1.275);
}

.player-name-input:hover {
  transform: translateY(-5px);
  box-shadow: 0 15px 30px rgba(0, 0, 0, 0.1);
}

.player-name-title {
  text-align: center;
  color: #333;
  margin-bottom: 25px;
  font-size: 24px;
  font-weight: 600;
  background: linear-gradient(90deg, #10B981, #3B82F6);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
}

.player-name-grid {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 15px;
  margin-bottom: 30px;
}

.player-name-group {
  display: flex;
  flex-direction: column;
  position: relative;
  transition: transform 0.3s ease;
}

.player-name-group:hover {
  transform: translateY(-3px);
}

.player-name-group label {
  margin-bottom: 8px;
  color: #555;
  font-weight: 500;
  transition: color 0.3s ease;
}

.player-name-group input {
  color: black;
  padding: 12px 15px;
  font-size: 15px;
  border: 2px solid #ddd;
  border-radius: 10px;
  transition: all 0.3s ease;
  background-color: #f9f9f9;
  box-shadow: inset 0 2px 4px rgba(0, 0, 0, 0.05);
}

.player-name-group input:focus {
  border-color: #3B82F6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.25);
  outline: none;
  background-color: #fff;
}

.player-name-group:hover label {
  color: #3B82F6;
}

.reset-positions-btn {
  margin-top: 2em;
  width: 100%;
  padding: 15px;
  background: linear-gradient(135deg, #F43F5E, #E11D48);
  color: white;
  border: none;
  border-radius: 12px;
  cursor: pointer;
  font-weight: 600;
  font-size: 16px;
  transition: all 0.3s cubic-bezier(0.175, 0.885, 0.32, 1.275);
  position: relative;
  overflow: hidden;
  letter-spacing: 0.5px;
}

.history-panel {
  margin-top: 2rem;
  border: 1px solid #e5e7eb;
  border-radius: 0.5rem;
  overflow: hidden;
  background-color: #f9fafb;
}

.history-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0.75rem 1rem;
  background-color: #f3f4f6;
  border-bottom: 1px solid #e5e7eb;
}

.history-title {
  margin: 0;
  font-size: 1.25rem;
  color: #374151;
}

.toggle-history-btn {
  background-color: #6366f1;
  color: white;
  border: none;
  border-radius: 0.375rem;
  padding: 0.5rem 1rem;
  font-size: 0.875rem;
  cursor: pointer;
  transition: background-color 0.2s;
}

.toggle-history-btn:hover {
  background-color: #4f46e5;
}

.history-content {
  padding: 1rem;
  max-height: 500px;
  overflow-y: auto;
}

.no-history {
  text-align: center;
  padding: 2rem;
  color: #6b7280;
}

.history-list {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.history-item {
  background-color: white;
  border: 1px solid #e5e7eb;
  border-radius: 0.375rem;
  padding: 1rem;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.05);
}

.history-item-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 0.75rem;
}

.history-date {
  font-weight: 500;
  color: #4b5563;
}

.history-actions {
  display: flex;
  gap: 0.5rem;
}

.load-history-btn,
.delete-history-btn {
  background-color: transparent;
  border: 1px solid #e5e7eb;
  border-radius: 0.25rem;
  width: 2rem;
  height: 2rem;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: background-color 0.2s;
}

.load-history-btn:hover {
  background-color: #f3f4f6;
}

.delete-history-btn:hover {
  background-color: #fee2e2;
  border-color: #fecaca;
}

.history-players {
  display: flex;
  flex-wrap: wrap;
  gap: 1rem;
  margin-bottom: 0.75rem;
}

.history-player {
  display: flex;
  align-items: center;
  gap: 0.25rem;
}

.history-player-name {
  font-weight: 500;
}

.history-player-score {
  font-weight: 600;
}

.history-rounds {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding-top: 0.75rem;
  border-top: 1px solid #e5e7eb;
}

.history-rounds-count {
  color: #6b7280;
  font-size: 0.875rem;
}

.toggle-details-btn {
  background-color: transparent;
  border: 1px solid #d1d5db;
  border-radius: 0.25rem;
  padding: 0.25rem 0.5rem;
  font-size: 0.75rem;
  color: #4b5563;
  cursor: pointer;
  transition: background-color 0.2s;
}

.toggle-details-btn:hover {
  background-color: #f3f4f6;
}

.history-details {
  margin-top: 1rem;
  overflow-x: auto;
}

.history-scores-table {
  color: black;
  width: 100%;
  border-collapse: collapse;
  font-size: 0.875rem;
}

.history-scores-table th,
.history-scores-table td {
  border: 1px solid #e5e7eb;
  padding: 0.5rem;
  text-align: center;
}

.history-scores-table th {
  background-color: #f3f4f6;
  font-weight: 500;
}

.history-scores-table tr:nth-child(even) {
  background-color: #f9fafb;
}

.score-positive {
  background-color: #10b981;
  color: white;
}

.score-negative {
  background-color: #ef4444;
  color: white;
}

/* Confirmation Dialog */
.confirm-dialog-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.confirm-dialog {
  background-color: white;
  border-radius: 0.5rem;
  padding: 1.5rem;
  max-width: 400px;
  width: 100%;
  box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1), 0 2px 4px -1px rgba(0, 0, 0, 0.06);
}

.confirm-dialog h3 {
  margin-top: 0;
  margin-bottom: 0.75rem;
  color: #111827;
}

.confirm-dialog p {
  margin-bottom: 1.5rem;
  color: #4b5563;
}

.confirm-dialog-buttons {
  display: flex;
  justify-content: flex-end;
  gap: 0.75rem;
}

.confirm-yes-btn,
.confirm-no-btn {
  padding: 0.5rem 1rem;
  border-radius: 0.375rem;
  font-size: 0.875rem;
  cursor: pointer;
}

.confirm-yes-btn {
  background-color: #ef4444;
  color: white;
  border: none;
}

.confirm-yes-btn:hover {
  background-color: #dc2626;
}

.confirm-no-btn {
  background-color: #f3f4f6;
  color: #4b5563;
  border: 1px solid #d1d5db;
}

.confirm-no-btn:hover {
  background-color: #e5e7eb;
}

.reset-positions-btn:before {
  content: '';
  position: absolute;
  top: 0;
  left: -100%;
  width: 100%;
  height: 100%;
  background: linear-gradient(90deg, transparent, rgba(255, 255, 255, 0.2), transparent);
  transition: 0.5s;
}

.reset-positions-btn:hover {
  transform: translateY(-3px);
  box-shadow: 0 7px 14px rgba(244, 63, 94, 0.3);
}

.reset-positions-btn:hover:before {
  left: 100%;
}

.reset-positions-btn:active {
  transform: translateY(1px);
}

.random-position-button-container {
  margin-bottom: 20px;
  text-align: center;
}

.random-positions-btn {
  width: 100%;
  padding: 15px;
  background: linear-gradient(135deg, #8B5CF6, #6D28D9);
  color: white;
  border: none;
  border-radius: 12px;
  cursor: pointer;
  font-weight: 600;
  font-size: 16px;
  transition: all 0.3s cubic-bezier(0.175, 0.885, 0.32, 1.275);
  position: relative;
  overflow: hidden;
  letter-spacing: 0.5px;
}

.random-positions-btn:before {
  content: '';
  position: absolute;
  top: 0;
  left: -100%;
  width: 100%;
  height: 100%;
  background: linear-gradient(90deg, transparent, rgba(255, 255, 255, 0.2), transparent);
  transition: 0.5s;
}

.random-positions-btn:hover {
  transform: translateY(-3px);
  box-shadow: 0 7px 14px rgba(139, 92, 246, 0.3);
}

.random-positions-btn:hover:before {
  left: 100%;
}

.random-positions-btn:active {
  transform: translateY(1px);
}

.save-players-btn {
  width: 100%;
  padding: 15px;
  background: linear-gradient(135deg, #10B981, #059669);
  color: white;
  border: none;
  border-radius: 12px;
  cursor: pointer;
  font-weight: 600;
  font-size: 16px;
  transition: all 0.3s cubic-bezier(0.175, 0.885, 0.32, 1.275);
  position: relative;
  overflow: hidden;
  letter-spacing: 0.5px;
}

.save-players-btn:before {
  content: '';
  position: absolute;
  top: 0;
  left: -100%;
  width: 100%;
  height: 100%;
  background: linear-gradient(90deg, transparent, rgba(255, 255, 255, 0.2), transparent);
  transition: 0.5s;
}

.save-players-btn:hover {
  transform: translateY(-3px);
  box-shadow: 0 7px 14px rgba(16, 185, 129, 0.3);
}

.save-players-btn:hover:before {
  left: 100%;
}

.save-players-btn:active {
  transform: translateY(1px);
}

.position-grid-container {
  background-color: white;
  padding: 30px;
  border-radius: 16px;
  margin-bottom: 40px;
  box-shadow: 0 8px 20px rgba(0, 0, 0, 0.06);
  transition: all 0.4s cubic-bezier(0.175, 0.885, 0.32, 1.275);
}

.position-grid-container:hover {
  transform: translateY(-5px);
  box-shadow: 0 15px 30px rgba(0, 0, 0, 0.1);
}

.position-grid-title {
  text-align: center;
  margin-bottom: 25px;
  color: #333;
  font-size: 24px;
  font-weight: 600;
  background: linear-gradient(90deg, #8B5CF6, #3B82F6);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
}

.position-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 20px;
}

.position-grid-cell {
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 20px;
  border: 3px solid #eee;
  border-radius: 12px;
  font-weight: 600;
  cursor: default;
  transition: all 0.3s cubic-bezier(0.175, 0.885, 0.32, 1.275);
  font-size: 16px;
  box-shadow: 0 3px 8px rgba(0, 0, 0, 0.05);
}

.position-grid-cell.selectable {
  border-color: #10B981;
  background-color: rgba(16, 185, 129, 0.05);
  cursor: pointer;
  position: relative;
  overflow: hidden;
}

.position-grid-cell.selectable:before {
  content: '';
  position: absolute;
  top: 0;
  left: -100%;
  width: 100%;
  height: 100%;
  background: linear-gradient(90deg, transparent, rgba(16, 185, 129, 0.1), transparent);
  transition: 0.5s;
}

.position-grid-cell.selectable:hover {
  transform: translateY(-5px);
  box-shadow: 0 10px 15px rgba(16, 185, 129, 0.2);
  background-color: rgba(16, 185, 129, 0.1);
}

.position-grid-cell.selectable:hover:before {
  left: 100%;
}

.position-grid-cell.occupied {
  background-color: #f0f0f0;
  color: #666;
  border-color: #ddd;
  transform: scale(1);
}

.input-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 15px;
  margin-bottom: 30px;
  width: 100%;
}

.input-group {
  display: flex;
  flex-direction: column;
  width: 100%;
  transition: transform 0.3s ease;
  position: relative;
}

.input-group:hover {
  transform: translateY(-3px);
}

.input-group label {
  margin-bottom: 8px;
  color: #555;
  font-weight: 500;
  transition: color 0.3s ease;
}

.input-group:hover label {
  color: #3B82F6;
}

.input-with-buttons {
  display: flex;
  position: relative;
  width: 100%;
}

.input-with-buttons input {
  width: 100%;
  background: white;
  color: black;
  padding: 12px;
  padding-right: 70px;
  border: 2px solid #ddd;
  border-radius: 10px;
  font-size: 16px;
  font-weight: 500;
  box-sizing: border-box;
  transition: all 0.3s ease;
  box-shadow: inset 0 2px 4px rgba(0, 0, 0, 0.05);
}

.input-with-buttons input:focus {
  border-color: #3B82F6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.25);
  outline: none;
}

.input-buttons {
  position: absolute;
  right: 0;
  top: 0;
  height: 100%;
  display: flex;
  align-items: center;
  padding-right: 8px;
}

.thumb-button {
  background: none;
  border: none;
  cursor: pointer;
  font-size: 20px;
  padding: 5px;
  margin: 0 3px;
  border-radius: 50%;
  transition: all 0.3s cubic-bezier(0.175, 0.885, 0.32, 1.275);
  width: 30px;
  height: 30px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.thumb-button:hover {
  transform: scale(1.3);
}

.thumb-up {
  color: #10B981;
}

.thumb-down {
  color: #EF4444;
}

.thumb-button.active {
  transform: scale(1.2);
  box-shadow: 0 3px 8px rgba(0, 0, 0, 0.15);
  animation: pulse 1.5s infinite;
}

@keyframes pulse {
  0% {
    box-shadow: 0 0 0 0 rgba(16, 185, 129, 0.4);
  }

  70% {
    box-shadow: 0 0 0 10px rgba(16, 185, 129, 0);
  }

  100% {
    box-shadow: 0 0 0 0 rgba(16, 185, 129, 0);
  }
}

.thumb-up.active {
  background-color: rgba(16, 185, 129, 0.2);
}

.thumb-down.active {
  background-color: rgba(239, 68, 68, 0.2);
  animation: pulse-red 1.5s infinite;
}

@keyframes pulse-red {
  0% {
    box-shadow: 0 0 0 0 rgba(239, 68, 68, 0.4);
  }

  70% {
    box-shadow: 0 0 0 10px rgba(239, 68, 68, 0);
  }

  100% {
    box-shadow: 0 0 0 0 rgba(239, 68, 68, 0);
  }
}

.add-score-btn {
  width: 100%;
  padding: 15px;
  background: linear-gradient(135deg, #3B82F6, #2563EB);
  color: white;
  border: none;
  border-radius: 12px;
  cursor: pointer;
  font-weight: 600;
  font-size: 16px;
  margin-bottom: 35px;
  transition: all 0.3s cubic-bezier(0.175, 0.885, 0.32, 1.275);
  position: relative;
  overflow: hidden;
  letter-spacing: 0.5px;
}

.add-score-btn:before {
  content: '';
  position: absolute;
  top: 0;
  left: -100%;
  width: 100%;
  height: 100%;
  background: linear-gradient(90deg, transparent, rgba(255, 255, 255, 0.2), transparent);
  transition: 0.5s;
}

.add-score-btn:hover {
  transform: translateY(-3px);
  box-shadow: 0 7px 14px rgba(59, 130, 246, 0.3);
}

.add-score-btn:hover:before {
  left: 100%;
}

.add-score-btn:active {
  transform: translateY(1px);
}

.scores-summary {
  display: flex;
  flex-wrap: wrap;
  justify-content: space-around;
  background: linear-gradient(145deg, #ffffff, #f8f8f8);
  padding: 15px;
  border-radius: 16px;
  margin-bottom: 25px;
  box-shadow: 0 8px 20px rgba(0, 0, 0, 0.06);
  width: 100%;
  box-sizing: border-box;
  gap: 10px;
  transition: all 0.4s cubic-bezier(0.175, 0.885, 0.32, 1.275);
}

.scores-summary:hover {
  transform: translateY(-5px);
  box-shadow: 0 15px 30px rgba(0, 0, 0, 0.1);
}

.participant-total {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 10px;
  flex: 1;
  min-width: 80px;
  max-width: 150px;
  transition: transform 0.3s ease;
  position: relative;
  border-radius: 12px;
  box-shadow: 0 3px 8px rgba(0, 0, 0, 0.05);
  background-color: white;
}

.participant-total:hover {
  transform: translateY(-5px) scale(1.05);
  box-shadow: 0 8px 15px rgba(0, 0, 0, 0.1);
  z-index: 1;
}

.participant-name {
  color: #555;
  margin-bottom: 8px;
  font-size: 18px;
  font-weight: 600;
  text-align: center;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  width: 100%;
  transition: color 0.3s ease;
}

.participant-total:hover .participant-name {
  color: #3B82F6;
}

.participant-total-score {
  font-size: 45px;
  font-weight: 700;
  transition: all 0.5s cubic-bezier(0.175, 0.885, 0.32, 1.275);
  text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.1);
}

.participant-total:hover .participant-total-score {
  transform: scale(1.1);
}

.individual-charts {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 20px;
  margin-bottom: 30px;
}

.individual-chart-container {
  background: linear-gradient(145deg, #ffffff, #f8f8f8);
  border-radius: 16px;
  padding: 25px;
  box-shadow: 0 8px 20px rgba(0, 0, 0, 0.06);
  transition: all 0.4s cubic-bezier(0.175, 0.885, 0.32, 1.275);
  position: relative;
  overflow: hidden;
}

.individual-chart-container:hover {
  transform: translateY(-5px) scale(1.02);
  box-shadow: 0 15px 30px rgba(0, 0, 0, 0.1);
}

.individual-chart-container:before {
  content: '';
  position: absolute;
  top: -100%;
  left: 0;
  width: 100%;
  height: 100%;
  background: linear-gradient(180deg, rgba(255, 255, 255, 0.2), transparent);
  transition: 0.5s;
}

.individual-chart-container:hover:before {
  top: 0;
}

.chart-title {
  text-align: center;
  margin-bottom: 25px;
  color: #333;
  font-size: 20px;
  font-weight: 600;
  background: linear-gradient(90deg, #3B82F6, #10B981);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  transition: transform 0.3s ease;
}

.individual-chart-container:hover .chart-title {
  transform: scale(1.05);
}

.line-chart {
  position: relative;
  height: 250px;
  width: 100%;
  border-left: 2px solid #ddd;
  border-bottom: 2px solid #ddd;
  margin-bottom: 40px;
  transition: all 0.4s ease;
}

.individual-chart-container:hover .line-chart {
  border-color: #3B82F6;
}

.chart-zero-line {
  position: absolute;
  left: 50px;
  right: 0;
  border-top: 1px dashed #aaa;
  top: 50%;
  transition: border-color 0.3s ease;
}

.individual-chart-container:hover .chart-zero-line {
  border-top-color: #3B82F6;
}

.chart-grid {
  position: absolute;
  top: 0;
  left: 50px;
  right: 0;
  bottom: 30px;
}

.line-track {
  position: absolute;
  width: 100%;
  height: 100%;
  top: 0;
  left: 0;
  border-radius: 8px;
  transition: all 0.4s ease;
}

.chart-svg {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  transition: all 0.5s ease;
}

.individual-chart-container:hover .chart-svg polyline {
  stroke-width: 3;
  stroke-dasharray: 1000;
  stroke-dashoffset: 1000;
  animation: dash 2s forwards;
}

@keyframes dash {
  to {
    stroke-dashoffset: 0;
  }
}

.line-point {
  position: absolute;
  width: 12px;
  height: 12px;
  border-radius: 50%;
  transform: translate(-50%, 50%);
  z-index: 10;
  cursor: pointer;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.2);
  transition: all 0.3s cubic-bezier(0.175, 0.885, 0.32, 1.275);
}

.line-point:hover {
  transform: translate(-50%, 50%) scale(1.5);
  box-shadow: 0 5px 10px rgba(0, 0, 0, 0.3);
}

.chart-x-axis {
  position: absolute;
  bottom: 0;
  left: 50px;
  right: 0;
  display: flex;
  justify-content: space-between;
  color: #666;
  font-size: 12px;
  font-weight: 500;
  transition: color 0.3s ease;
}

.chart-y-axis {
  position: absolute;
  top: 0;
  left: 0;
  bottom: 30px;
  width: 50px;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  align-items: flex-end;
  padding-right: 10px;
  color: #666;
  font-size: 12px;
  font-weight: 500;
  transition: color 0.3s ease;
}

.individual-chart-container:hover .chart-x-axis,
.individual-chart-container:hover .chart-y-axis {
  color: #333;
}

.participant-score-difference {
  font-size: 18px;
  color: #666;
  margin-top: 8px;
  text-align: center;
  width: 100%;
  transition: all 0.3s ease;
  animation: fadeIn 1s ease-in-out;
}

@keyframes fadeIn {
  0% {
    opacity: 0;
    transform: translateY(10px);
  }

  100% {
    opacity: 1;
    transform: translateY(0);
  }
}

.participant-total:hover .participant-score-difference {
  color: #3B82F6;
}

.action-buttons {
  display: flex;
  justify-content: center;
  gap: 30px;
  margin-top: 20px;
}

.download-btn {
  padding: 15px 30px;
  background: linear-gradient(135deg, #10B981, #059669);
  color: white;
  border: none;
  border-radius: 12px;
  cursor: pointer;
  font-weight: 600;
  font-size: 16px;
  transition: all 0.3s cubic-bezier(0.175, 0.885, 0.32, 1.275);
  position: relative;
  overflow: hidden;
  letter-spacing: 0.5px;
  box-shadow: 0 4px 10px rgba(16, 185, 129, 0.2);
}

.download-btn:before {
  content: '';
  position: absolute;
  top: 0;
  left: -100%;
  width: 100%;
  height: 100%;
  background: linear-gradient(90deg, transparent, rgba(255, 255, 255, 0.2), transparent);
  transition: 0.5s;
}

.download-btn:hover {
  transform: translateY(-5px);
  box-shadow: 0 10px 20px rgba(16, 185, 129, 0.3);
}

.download-btn:hover:before {
  left: 100%;
}

.download-btn:active {
  transform: translateY(1px);
}

.reset-data-btn {
  padding: 15px 30px;
  background: linear-gradient(135deg, #EF4444, #DC2626);
  color: white;
  border: none;
  border-radius: 12px;
  cursor: pointer;
  font-weight: 600;
  font-size: 16px;
  transition: all 0.3s cubic-bezier(0.175, 0.885, 0.32, 1.275);
  position: relative;
  overflow: hidden;
  letter-spacing: 0.5px;
  box-shadow: 0 4px 10px rgba(239, 68, 68, 0.2);
}

.reset-data-btn:before {
  content: '';
  position: absolute;
  top: 0;
  left: -100%;
  width: 100%;
  height: 100%;
  background: linear-gradient(90deg, transparent, rgba(255, 255, 255, 0.2), transparent);
  transition: 0.5s;
}

.reset-data-btn:hover {
  transform: translateY(-5px);
  box-shadow: 0 10px 20px rgba(239, 68, 68, 0.3);
}

.reset-data-btn:hover:before {
  left: 100%;
}

.reset-data-btn:active {
  transform: translateY(1px);
}

@media (max-width: 768px) {

  .history-item-header,
  .history-rounds {
    flex-direction: column;
    align-items: flex-start;
    gap: 0.5rem;
  }

  .history-actions {
    align-self: flex-end;
  }

  .history-players {
    flex-direction: column;
    gap: 0.5rem;
  }

  .viewer-header {
    flex-direction: column;
    align-items: flex-start;
    gap: 10px;
  }

  .viewer-header .stop-live-btn.small {
    align-self: flex-end;
  }

  .viewer-scores-summary {
    flex-direction: column;
    align-items: center;
  }

  .viewer-charts {
    grid-template-columns: 1fr;
  }

  .share-link {
    flex-direction: column;
    align-items: flex-start;
  }

  .watch-url-input {
    width: 90%;
    margin: 5px 0;
  }

  .viewers-count {
    margin-left: 0;
    margin-top: 5px;
  }

  .individual-charts {
    grid-template-columns: 1fr;
  }

  .player-name-grid {
    grid-template-columns: repeat(2, 1fr);
  }

  .input-grid {
    grid-template-columns: repeat(2, 1fr);
  }

  .action-buttons {
    flex-direction: column;
    align-items: center;
    gap: 15px;
  }

  .download-btn,
  .reset-data-btn {
    width: 100%;
  }
}

@media (max-width: 480px) {
  .player-name-grid {
    grid-template-columns: 1fr;
  }

  .input-grid {
    grid-template-columns: 1fr;
  }

  .scores-summary {
    flex-direction: column;
    align-items: center;
  }

  .participant-total {
    margin-bottom: 15px;
    max-width: 100%;
  }
}

/* Animations */
@keyframes slideIn {
  0% {
    opacity: 0;
    transform: translateY(30px);
  }

  100% {
    opacity: 1;
    transform: translateY(0);
  }
}

@keyframes fadeScale {
  0% {
    opacity: 0;
    transform: scale(0.9);
  }

  100% {
    opacity: 1;
    transform: scale(1);
  }
}

/* Apply animations to containers */
.player-name-input,
.position-grid-container,
.scores-summary,
.individual-chart-container,
.input-grid {
  animation: fadeScale 0.6s ease-out;
}

.participant-total {
  animation: slideIn 0.6s ease-out;
  animation-fill-mode: both;
}

.participant-total:nth-child(1) {
  animation-delay: 0.1s;
}

.participant-total:nth-child(2) {
  animation-delay: 0.2s;
}

.participant-total:nth-child(3) {
  animation-delay: 0.3s;
}

.participant-total:nth-child(4) {
  animation-delay: 0.4s;
}

/* Glassmorphism effects */
.scores-summary,
.individual-chart-container {
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  background: rgba(255, 255, 255, 0.85);
  border: 1px solid rgba(255, 255, 255, 0.2);
}

.position-grid-cell.selectable {
  backdrop-filter: blur(5px);
  -webkit-backdrop-filter: blur(5px);
  background: rgba(16, 185, 129, 0.05);
  border: 1px solid rgba(16, 185, 129, 0.2);
}
</style>