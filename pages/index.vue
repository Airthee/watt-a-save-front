<template>
  <section class="section">
    <div class="container">
      <h1 class="title">
        Section
      </h1>
      <h2 class="subtitle">
        A simple container to divide your page into <strong>sections</strong>, like the one you're currently reading
      </h2>

      <div>
        <div class="field">
          <label class="label">Date de début</label>
          <div class="control">
            <datepicker @selected="updateDatacollection()" v-model="startDate" :clear-button="true" />
          </div>
        </div>
        <div class="field">
          <label class="label">Date de fin</label>
          <div class="control">
            <datepicker @selected="updateDatacollection()" v-model="endDate" :clear-button="true" />
          </div>
        </div>
        <div class="field">
          <label class="label">Classe</label>
          <div class="control">
            <div class="select">
              <select @change="updateDatacollection()" v-model="inputClassroom">
                <option :value="0">
                  Selectionner une classe
                </option>
                <option v-for="cr in allClassrooms" :key="cr.id" v-html="cr.name" :value="cr.id" />
              </select>
            </div>
          </div>
        </div>
        <LineChart :chart-data="datacollection" :options="chartOptions" class="line-chart" />
      </div>
    </div>
  </section>
</template>

<script>
import Datepicker from 'vuejs-datepicker'
import LineChart from '@/components/LineChart'

export default {
  components: {
    Datepicker,
    LineChart
  },

  data () {
    return {
      startDate: new Date('2020-01-01'),
      endDate: new Date(),
      inputClassroom: 0,
      allClassrooms: [],
      allBenches: [],
      measures: [],
      chartOptions: {
        responsive: true,
        maintainAspectRatio: false
      },
      datacollection: {
        datasets: [],
        options: this.chartOptions
      }
    }
  },

  computed: {
    classroomId () {
      return parseInt(this.inputClassroom)
    }
  },

  mounted () {
    // Chargement des données
    // Récupération de toutes les salles de classes
    // Recupération des données de consommation (aujourd'hui)
    Promise.all([this.loadClassrooms(), this.loadBenches()])
      .then(() => this.updateDatacollection())
      .then(() => this.$nuxt.$loading.finish())

    this.$nextTick(() => {
      this.$nuxt.$loading.start()
    })
  },

  methods: {
    loadClassrooms () {
      return new Promise((resolve, reject) => {
        this.$axios.$get('/rooms')
          .then((response) => {
            this.allClassrooms = response.data
            resolve()
          })
      })
    },
    loadBenches () {
      return new Promise((resolve, reject) => {
        this.$axios.$get('/benches')
          .then((response) => {
            this.allBenches = response.data
          })
        resolve()
      })
    },
    loadData () {
      return new Promise((resolve, reject) => {
        const axiosParams = {}

        // If end date, check that it is > to the start date (trigger error)
        if (this.endDate && this.startDate && this.endDate < this.startDate) {
          throw new Error('End date must be after start date')
        }

        // Filter data
        axiosParams.before = this.startDate ? this.$moment(this.startDate).format('YYYY-MM-DD HH-mm-ss') : null
        axiosParams.after = this.endDate ? this.$moment(this.endDate).format('YYYY-MM-DD HH-mm-ss') : null
        axiosParams.bench_id = null
        axiosParams.room_id = this.classroomId || null

        this.$axios.get('/measures', axiosParams)
          .then((response) => {
            const allMeasures = response.data.data
            // Group by bench
            const measuresByBench = this.groupByBench(allMeasures)

            // We went 100 measures by bench
            const finalMeasures = []
            for (let [, benchMeasures] of Object.entries(measuresByBench)) {
              // Sort by date
              benchMeasures = benchMeasures.sort((a, b) => {
                const timeA = new Date(a.date).getTime()
                const timeB = new Date(b.date).getTime()
                if (timeA > timeB) { return 1 }
                if (timeB > timeA) { return -1 }
                return 0
              })

              // Create 100 groups
              const measuresByGroup = parseInt(benchMeasures.length / 100)
              while (benchMeasures.length > 0) {
                const groupValues = benchMeasures.splice(0, measuresByGroup)
                const refValue = groupValues[0]

                // Consumption is the mean of the group consumption
                refValue.consumption = groupValues.reduce((total, { consumption }) => total + parseFloat(consumption), 0) / groupValues.length
                refValue.date = new Date(refValue.date)
                finalMeasures.push(refValue)
              }
            }
            this.measures = finalMeasures
            resolve()
          })
      })
    },
    updateDatacollection () {
      return new Promise((resolve, reject) => {
        this.loadData()
          .then(() => {
            // Data to return
            const datacollection = {}

            // Create labels
            datacollection.labels = []
            this.measures.forEach((measure) => {
              const label = this.$moment(measure.date).format('DD/MM/YYYY HH:mm:ss')
              if (!datacollection.labels.includes(label)) {
                datacollection.labels.push(label)
              }
            })

            // If classroom selected, then dataset by bench
            // else dataset by classroom
            if (this.classroomId) {
              datacollection.datasets = Object.entries(this.groupByBench(this.measures)).map(([benchId, value]) => {
                const bench = this.allBenches.find(b => parseInt(b.id) === parseInt(benchId))
                return {
                  label: bench.serial_number,
                  backgroundColor: this.randomColor(),
                  data: value.map(v => v.consumption)
                }
              })
            } else {
              datacollection.datasets = Object.entries(this.groupByClassroom(this.measures)).map(([classroomId, value]) => {
                const classroom = this.allClassrooms.find(c => parseInt(c.id) === parseInt(classroomId))
                return {
                  label: classroom.name,
                  backgroundColor: this.randomColor(),
                  data: value.map(v => v.consumption)
                }
              })
            }

            this.datacollection = datacollection
            resolve()
          })
      })
    },
    findClassroomForBenchId (benchId) {
      return this.allClassrooms.find(c => parseInt(c.id) === parseInt(benchId))
    },
    findClassroomForMeasure (measure) {
      return this.allClassrooms.find(c => parseInt(c.bench_id) === parseInt(this.findBenchForMeasure(measure).id))
    },
    findBenchForMeasure (measure) {
      return this.allBenches.find(b => parseInt(b.id) === parseInt(measure.bench_id))
    },
    findBenchesForClassroomId (classroomId) {
      return this.allBenches.filter(b => parseInt(b.id) === parseInt(classroomId))
    },
    groupByBench (data) {
      const dataByBench = {}
      data.forEach((d) => {
        if (typeof dataByBench[d.bench_id] === 'undefined') {
          dataByBench[d.bench_id] = []
        }
        dataByBench[d.bench_id].push(d)
      })
      return dataByBench
    },
    groupByClassroom (data) {
      const dataByClassroom = {}
      data.forEach((d) => {
        const bench = this.findBenchForMeasure(d)
        if (typeof dataByClassroom[bench.room_id] === 'undefined') {
          dataByClassroom[bench.room_id] = []
        }
        dataByClassroom[bench.room_id].push(d)
      })
      return dataByClassroom
    },
    randomColor: () => `#${(parseInt(Math.random() * (16 ** 6))).toString(16)}`
  }
}
</script>

<style lang="scss" scoped>
  .line-chart {
    width: 100%;
    overflow: auto;
  }
</style>
