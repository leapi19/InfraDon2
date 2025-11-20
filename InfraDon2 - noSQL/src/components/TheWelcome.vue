<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'
import PouchFind from 'pouchdb-find'

PouchDB.plugin(PouchFind)

// Interface pour un jeu
interface Game {
  _id: string
  _rev?: string
  title: string
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

const searchTitle = ref('')

const remoteURL = 'http://admin:admin@localhost:5984/database-infradon'

// --- Initialisation de la base ---
const initDatabase = async () => {
  const localDB = new PouchDB('database-infradon')
  storage.value = localDB
  console.log('Connecté à la base : ' + localDB.name)

  // Création d’un index sur le champ "title" à la racine
  await localDB.createIndex({
    index: {
      fields: ['title'],
      name: 'index-title',
      ddoc: 'index-title-doc',
    },
  })
  console.log("Index 'title' créé ✔")
}

// --- Réplication ---
const replicateFromDistant = () => {
  if (!storage.value) return
  console.log('Réplication FROM distante')
  storage.value.replicate
    .from(remoteURL)
    .on('complete', () => {
      console.log('Réplication FROM terminée')
      fetchData()
    })
    .on('error', (err) => console.error('Erreur réplication FROM :', err))
}

const replicateToDistant = () => {
  if (!storage.value) return
  console.log('↗️ Réplication TO distante')
  storage.value.replicate
    .to(remoteURL)
    .on('complete', () => console.log('✔ Réplication TO terminée'))
    .on('error', (err) => console.error('✗ Erreur réplication TO :', err))
}

// --- Récupération des données ---
const fetchData = async () => {
  if (!storage.value) return
  try {
    const result = await storage.value.allDocs({ include_docs: true })
    gamesData.value = result.rows.map((row) => row.doc as Game).filter((doc) => doc && doc.title)
    console.log('Données récupérées :', gamesData.value)
  } catch (err) {
    console.error('Erreur fetchData :', err)
  }
}

// --- Recherche indexée ---
const searchByTitle = async () => {
  if (!storage.value) return
  if (!searchTitle.value.trim()) {
    fetchData()
    return
  }
  try {
    const result = await storage.value.find({
      selector: { title: { $regex: RegExp(searchTitle.value, 'i') } },
      use_index: 'index-title-doc',
    })
    gamesData.value = result.docs as Game[]
    console.log('Résultats filtrés :', gamesData.value)
  } catch (err) {
    console.error('Erreur recherche indexée :', err)
  }
}

// --- Ajout ---
const addGame = async () => {
  if (!storage.value || !newGameTitle.value || !newGameEditor.value || !newGameRelease.value) {
    alert('Veuillez remplir tous les champs obligatoires')
    return
  }
  const gameObj: Game = {
    _id: `game_${Date.now()}`,
    title: newGameTitle.value,
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
  }
  try {
    await storage.value.put(gameObj)
    console.log('Jeu ajouté ✔')
    newGameTitle.value = ''
    newGameEditor.value = ''
    newGameCountry.value = ''
    newGameRelease.value = null
    fetchData()
  } catch (err) {
    console.error('Erreur ajout :', err)
  }
}

// --- Modification ---
const startEdit = (game: Game) => {
  const g = game.biblio.games[0]
  if (!g) return
  editMode.value = true
  editingGameId.value = game._id
  editGameTitle.value = g.title
  editGameEditor.value = g.editor
  editGameCountry.value = g.country || ''
  editGameRelease.value = g.release
}

const cancelEdit = () => {
  editMode.value = false
  editingGameId.value = null
  editGameTitle.value = ''
  editGameEditor.value = ''
  editGameCountry.value = ''
  editGameRelease.value = null
}

const saveEdit = async () => {
  if (
    !storage.value ||
    !editingGameId.value ||
    !editGameTitle.value ||
    !editGameEditor.value ||
    !editGameRelease.value
  ) {
    alert('Veuillez remplir tous les champs obligatoires')
    return
  }
  try {
    const doc = (await storage.value.get(editingGameId.value)) as Game
    const updated: Game = {
      ...doc,
      title: editGameTitle.value,
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
    await storage.value.put(updated)
    console.log('Jeu modifié ✔')
    cancelEdit()
    fetchData()
  } catch (err) {
    console.error('Erreur modification :', err)
  }
}

// --- Suppression ---
const deleteGame = async (game: Game) => {
  if (!storage.value) return
  if (!confirm(`Supprimer "${game.title}" ?`)) return
  try {
    await storage.value.remove(game._id, game._rev!)
    console.log('Jeu supprimé ✔')
    fetchData()
  } catch (err) {
    console.error('Erreur suppression :', err)
  }
}

// --- Factory pour générer des jeux ---
const generateGames = async (count: number) => {
  if (!storage.value) return
  const docs: Game[] = []
  for (let i = 0; i < count; i++) {
    docs.push({
      _id: `game_${Date.now()}_${i}`,
      title: `Game ${i}`,
      biblio: {
        games: [{ title: `Game ${i}`, editor: `Editor ${i}`, release: 2000 + (i % 20) }],
      },
    })
  }
  await storage.value.bulkDocs(docs)
  console.log(`${count} jeux générés ✔`)
  fetchData()
}

onMounted(async () => {
  console.log('=> Composant initialisé')
  await initDatabase()
  fetchData()
})
</script>

<template>
  <h1>Liste des jeux</h1>

  <!-- Recherche -->
  <div style="margin-bottom: 20px">
    <input v-model="searchTitle" placeholder="Recherche par titre..." />
    <button @click="searchByTitle">Rechercher</button>
    <button
      @click="
        () => {
          searchTitle = ''
          fetchData()
        }
      "
    >
      Réinitialiser
    </button>
  </div>

  <!-- Réplication -->
  <div style="margin-bottom: 20px">
    <button @click="replicateFromDistant">Replicate FROM</button>
    <button @click="replicateToDistant">Replicate TO</button>
  </div>

  <!-- Factory -->
  <div style="margin-bottom: 20px">
    <button @click="generateGames(50)">Générer 50 jeux</button>
  </div>

  <!-- Formulaire d'ajout -->
  <div v-if="!editMode">
    <h2>Ajouter un jeu</h2>
    <form @submit.prevent="addGame">
      <div><label>Titre: </label><input v-model="newGameTitle" required /></div>
      <div><label>Éditeur: </label><input v-model="newGameEditor" required /></div>
      <div><label>Pays: </label><input v-model="newGameCountry" /></div>
      <div>
        <label>Année: </label><input type="number" v-model.number="newGameRelease" required />
      </div>
      <button type="submit">Ajouter</button>
    </form>
  </div>

  <!-- Formulaire modification -->
  <div v-if="editMode">
    <h2>Modifier le jeu</h2>
    <form @submit.prevent="saveEdit">
      <div><label>Titre: </label><input v-model="editGameTitle" required /></div>
      <div><label>Éditeur: </label><input v-model="editGameEditor" required /></div>
      <div><label>Pays: </label><input v-model="editGameCountry" /></div>
      <div>
        <label>Année: </label><input type="number" v-model.number="editGameRelease" required />
      </div>
      <button type="submit">Sauvegarder</button>
      <button type="button" @click="cancelEdit">Annuler</button>
    </form>
  </div>

  <!-- Liste -->
  <div
    v-for="game in gamesData"
    :key="game._id"
    style="margin-top: 10px; border: 1px solid #ccc; padding: 10px"
  >
    <div v-for="(g, i) in game.biblio.games" :key="i">
      <h2>{{ g.title }}</h2>
      <p>Éditeur: {{ g.editor }}</p>
      <p v-if="g.country">Pays: {{ g.country }}</p>
      <p>Année: {{ g.release }}</p>
      <button @click="startEdit(game)" :disabled="editMode">Modifier</button>
      <button @click="deleteGame(game)" style="margin-left: 10px; color: white; background: red">
        Supprimer
      </button>
    </div>
  </div>
</template>
