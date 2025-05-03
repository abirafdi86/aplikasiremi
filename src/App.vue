<template>
  <div class="score-tracker-container">
    <h1 class="title">Skor Player</h1>

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
</template>

<script>
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
      scores: []
    }
  },
  computed: {
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
    this.loadSavedState()
  },
  methods: {
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
      // Validate and prepare scores
      const newScoreEntry = this.participants.map(participant => ({
        name: participant.name,
        score: this.currentScore[participant.name] || 0,
        mark: this.scoreMarks[participant.name] // Tambahkan informasi penanda (positif/negatif)
      }))

      // Add to scores history
      this.scores.push(newScoreEntry)

      // Save to localStorage
      this.saveToLocalStorage()

      // Reset input fields and marks
      this.participants.forEach(participant => {
        this.currentScore[participant.name] = null
        this.scoreMarks[participant.name] = null
      })
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
      localStorage.setItem('scoreTrackerData', JSON.stringify(this.scores))
    },

    resetData() {
      if (confirm('Apakah Anda yakin ingin menghapus semua data skor?')) {
        // Clear scores array
        this.scores = []

        // Clear localStorage
        localStorage.removeItem('scoreTrackerData')
        localStorage.removeItem('scoreTrackerParticipants')

        // Reset participants and input fields
        this.participants = []
        this.playerNameInputs = ['', '', '', '']
        this.currentScore = {}
        this.scoreMarks = {}
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
  width: 30%;
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
  width: 30%;
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