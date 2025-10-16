<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'

// Interface pour un jeu
declare interface Game {
  _id: string
  _rev: string
  biblio: {
    games: Array<{
      title: string
      editor: string
      country?: string
      release: number
    }>
  }
}

// Référence à la base de données
const storage = ref<PouchDB.Database>()
// Données stockées (liste de jeux)
const gamesData = ref<Game[]>([])

// Initialisation de la base de données
const initDatabase = () => {
  console.log('=> Connexion à la base de données')
  const db = new PouchDB('http://admin:admin@localhost:5984/database-infradon')
  if (db) {
    console.log('Connecté à la collection : ' + db.name)
    storage.value = db
  } else {
    console.warn('Échec lors de la connexion à la base de données')
  }
}

// Récupération des données
const fetchData = async () => {
  if (!storage.value) return
  try {
    const result = await storage.value.allDocs({ include_docs: true })
    gamesData.value = result.rows
      .map((row) => row.doc as Game)
      .filter((doc) => doc.biblio && doc.biblio.games)
    console.log('Données récupérées :', gamesData.value)
  } catch (error) {
    console.error('Erreur lors de la récupération des données :', error)
  }
}

onMounted(() => {
  console.log('=> Composant initialisé')
  initDatabase()
  fetchData()
})
</script>

<template>
  <h1>Games List</h1>
  <div v-for="game in gamesData" v-bind:key="game._id">
    <div v-for="(g, index) in game.biblio.games" :key="index">
      <h2>{{ g.title }}</h2>
      <p>Éditeur : {{ g.editor }}</p>
      <p v-if="g.country">Pays : {{ g.country }}</p>
      <p>Année de sortie : {{ g.release }}</p>
    </div>
  </div>
</template>
