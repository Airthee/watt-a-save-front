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
            <datepicker v-model="startDate" :clear-button="true" name="startDate" />
          </div>
        </div>
        <div class="field">
          <label class="label">Date de fin</label>
          <div class="control">
            <datepicker v-model="endDate" :clear-button="true" name="endDate" />
          </div>
        </div>
        <div class="field">
          <label class="label">Classe</label>
          <div class="control">
            <div class="select">
              <select v-model="classroomId" name="classroom">
                <option value="0">
                  Selectionner une classe
                </option>
                <option v-for="cr in allClassrooms" :key="cr.id" v-html="cr.name" :value="cr.id" />
              </select>
            </div>
          </div>
        </div>

        <div>
          <line-chart :chart-data="datacollection" />
        </div>
      </div>
    </div>
  </section>
</template>

<script>
import Datepicker from 'vuejs-datepicker'
import { Line } from 'vue-chartjs'

export default {
  components: {
    Datepicker
  },

  extends: Line,

  data () {
    return {
      startDate: new Date('2020-01-01'),
      endDate: new Date(),
      classroomId: 0,
      allClassrooms: [],
      mesures: []
    }
  },

  computed: {
    datacollection () {
      // If end date, check that it is > to the start date (trigger error)
      if (this.endDate && this.startDate && this.endDate < this.startDate) {
        throw new Error('End date must be after start date')
      }

      // Filter data
      const dataFiltered = this.mesures.filter((mesure) => {
        // mesure.bench.id
        if (this.classroomId && mesure.bench.classroom.id !== this.classroomId) { return false }

        // mesure.createdAt
        if (this.startDate && mesure.createdAt < this.startDate) { return false }
        if (this.endDate && mesure.createdAt > this.endDate) { return false }

        // If all tests passed
        return true
      })

      // Draw data
      return {
        labels: 
      }
    }
  },

  mounted () {
    // Chargement des données
    // Récupération de toutes les salles de classes
    // Recupération des données de consommation (aujourd'hui)
    const promiseLoadClassrooms = this.loadClassrooms()
    const promiseLoadDate = this.loadData()

    // Draw chart when data loaded
    Promise.all([promiseLoadClassrooms, promiseLoadDate]).then(() => {
      this.drawChart()
    })
  },

  methods: {
    loadClassrooms () {
      return new Promise((resolve, reject) => {
        for (let i = 1; i <= 10; i++) {
          this.allClassrooms.push({
            id: i,
            name: `S${i}`
          })
        }
        resolve()
      })
    },
    loadData () {
      return new Promise((resolve, reject) => {
        let date
        this.allClassrooms.forEach((classroom) => {
          for (let benchId = 1; benchId <= 5; benchId++) {
            date = new Date()
            for (let mesureId = 1; mesureId <= 1000; mesureId++) {
              this.mesures.push({
                id: mesureId,
                bench: {
                  id: benchId,
                  classroom
                },
                serialNumber: `AAA-BBB-${benchId}`,
                power: Math.random() * 5, // Between 0 and 5
                createdAt: this.$moment(date).format('YYYY-MM-DD HH:mm:ss')
              })
              date.setSeconds(date.getSeconds() + 1)
            }
          }
        })
        resolve()
      })
    }
  }
}
</script>

<style>

</style>
