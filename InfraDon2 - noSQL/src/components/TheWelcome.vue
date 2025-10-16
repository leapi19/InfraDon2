<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'

// Interface pour un jeu
declare interface Game {
  _id: string
  _rev?: string
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

// Champs pour le formulaire d'ajout
const newGameTitle = ref('')
const newGameEditor = ref('')
const newGameCountry = ref('')
const newGameRelease = ref<number | null>(null)

// Initialisation de la base de données
const initDatabase = () => {
  console.log('=> Connexion à la base de données')
  const db = new PouchDB('http://admin:admin@localhost:5984/database-infradon')
  storage.value = db
  console.log('Connecté à la collection : ' + db.name)
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

// Ajout d'un nouveau jeu
const addGame = async () => {
  if (!storage.value || !newGameTitle.value || !newGameEditor.value || !newGameRelease.value) {
    alert('Veuillez remplir tous les champs obligatoires !')
    return
  }

  // Crée un nouvel objet jeu
  const gameObj = {
    _id: `game_${Date.now()}`, // ID unique basé sur la date
    biblio: {
      games: [
        {
          title: newGameTitle.value,
          editor: newGameEditor.value,
          country: newGameCountry.value || undefined,
          release: newGameRelease.value,
        },
      ],
    },
  } as Game

  try {
    await storage.value.put(gameObj)
    console.log('Jeu ajouté avec succès !')
    // Réinitialise le formulaire
    newGameTitle.value = ''
    newGameEditor.value = ''
    newGameCountry.value = ''
    newGameRelease.value = null
    // Recharge les données
    fetchData()
  } catch (error) {
    console.error("Erreur lors de l'ajout du jeu :", error)
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

  <div>
    <h2>Ajouter un nouveau jeu</h2>
    <form @submit.prevent="addGame">
      <div>
        <label>Titre : </label>
        <input v-model="newGameTitle" required />
      </div>
      <div>
        <label>Éditeur : </label>
        <input v-model="newGameEditor" required />
      </div>
      <div>
        <label>Pays : </label>
        <input v-model="newGameCountry" />
      </div>
      <div>
        <label>Année de sortie : </label>
        <input type="number" v-model.number="newGameRelease" required />
      </div>
      <button type="submit">Ajouter le jeu</button>
    </form>
  </div>

  <div v-for="game in gamesData" :key="game._id" style="margin-top: 20px">
    <div v-for="(g, index) in game.biblio.games" :key="index">
      <h2>{{ g.title }}</h2>
      <p>Éditeur : {{ g.editor }}</p>
      <p v-if="g.country">Pays : {{ g.country }}</p>
      <p>Année de sortie : {{ g.release }}</p>
    </div>
  </div>
</template>
