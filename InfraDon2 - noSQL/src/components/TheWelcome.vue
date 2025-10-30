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

// Champs pour le formulaire de modification
const editMode = ref(false)
const editingGameId = ref<string | null>(null)
const editGameTitle = ref('')
const editGameEditor = ref('')
const editGameCountry = ref('')
const editGameRelease = ref<number | null>(null)

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

  const gameObj = {
    _id: `game_${Date.now()}`,
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
    newGameTitle.value = ''
    newGameEditor.value = ''
    newGameCountry.value = ''
    newGameRelease.value = null
    fetchData()
  } catch (error) {
    console.error("Erreur lors de l'ajout du jeu :", error)
  }
}

// Préparer la modification d'un jeu
const startEdit = (game: Game) => {
  const gameData = game.biblio?.games?.[0]
  if (!gameData) return

  editMode.value = true
  editingGameId.value = game._id
  editGameTitle.value = gameData.title
  editGameEditor.value = gameData.editor
  editGameCountry.value = gameData.country || ''
  editGameRelease.value = gameData.release
}

// Annuler la modification
const cancelEdit = () => {
  editMode.value = false
  editingGameId.value = null
  editGameTitle.value = ''
  editGameEditor.value = ''
  editGameCountry.value = ''
  editGameRelease.value = null
}

// Sauvegarder les modifications
const saveEdit = async () => {
  if (
    !storage.value ||
    !editingGameId.value ||
    !editGameTitle.value ||
    !editGameEditor.value ||
    !editGameRelease.value
  ) {
    alert('Veuillez remplir tous les champs obligatoires !')
    return
  }

  try {
    // Récupérer le document actuel pour obtenir le _rev
    const doc = (await storage.value.get(editingGameId.value)) as Game

    // Mettre à jour le document
    const updatedGame = {
      ...doc,
      biblio: {
        games: [
          {
            title: editGameTitle.value,
            editor: editGameEditor.value,
            country: editGameCountry.value || undefined,
            release: editGameRelease.value,
          },
        ],
      },
    }

    await storage.value.put(updatedGame)
    console.log('Jeu modifié avec succès !')
    cancelEdit()
    fetchData()
  } catch (error) {
    console.error('Erreur lors de la modification du jeu :', error)
  }
}

// Supprimer un jeu
const deleteGame = async (game: Game) => {
  if (!storage.value) return

  const gameTitle = game.biblio?.games?.[0]?.title || 'ce jeu'
  const confirmDelete = confirm(`Êtes-vous sûr de vouloir supprimer "${gameTitle}" ?`)
  if (!confirmDelete) return

  try {
    await storage.value.remove(game._id, game._rev!)
    console.log('Jeu supprimé avec succès !')
    fetchData()
  } catch (error) {
    console.error('Erreur lors de la suppression du jeu :', error)
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

  <!-- Formulaire d'ajout -->
  <div v-if="!editMode">
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

  <!-- Formulaire de modification -->
  <div v-if="editMode">
    <h2>Modifier le jeu</h2>
    <form @submit.prevent="saveEdit">
      <div>
        <label>Titre : </label>
        <input v-model="editGameTitle" required />
      </div>
      <div>
        <label>Éditeur : </label>
        <input v-model="editGameEditor" required />
      </div>
      <div>
        <label>Pays : </label>
        <input v-model="editGameCountry" />
      </div>
      <div>
        <label>Année de sortie : </label>
        <input type="number" v-model.number="editGameRelease" required />
      </div>
      <button type="submit">Sauvegarder</button>
      <button type="button" @click="cancelEdit" style="margin-left: 10px">Annuler</button>
    </form>
  </div>

  <!-- Liste des jeux -->
  <div
    v-for="game in gamesData"
    :key="game._id"
    style="margin-top: 20px; border: 1px solid #ccc; padding: 10px"
  >
    <div v-for="(g, index) in game.biblio.games" :key="index">
      <h2>{{ g.title }}</h2>
      <p>Éditeur : {{ g.editor }}</p>
      <p v-if="g.country">Pays : {{ g.country }}</p>
      <p>Année de sortie : {{ g.release }}</p>
      <div style="margin-top: 10px">
        <button @click="startEdit(game)" :disabled="editMode">Modifier</button>
        <button
          @click="deleteGame(game)"
          style="margin-left: 10px; background-color: #dc3545; color: white"
        >
          Supprimer
        </button>
      </div>
    </div>
  </div>
</template>
