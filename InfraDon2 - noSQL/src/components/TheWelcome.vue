<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'
import PouchFind from 'pouchdb-find'

PouchDB.plugin(PouchFind)

// Interface pour un jeu mise à jour avec likes et commentaires
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
  likes: number
  comments: Array<{
    author: string
    content: string
    date: number // Timestamp pour l'ordre et l'ID du commentaire
  }>
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

// Champs pour le formulaire de modification du JEU
const editMode = ref(false)
const editingGameId = ref<string | null>(null)
const editGameTitle = ref('')
const editGameEditor = ref('')
const editGameCountry = ref('')
const editGameRelease = ref<number | null>(null)

// Champs pour la recherche
const searchTitle = ref('')

// Champs pour les commentaires
const newCommentAuthor = ref('')
const newCommentContent = ref('')

// État pour la modification d'un commentaire
const editingComment = ref<{
  gameId: string
  date: number
  content: string
  author: string
} | null>(null)

// Référence pour le tri
const sortKey = ref<'title' | 'likes'>('title')

const remoteURL = 'http://admin:admin@localhost:5984/database-infradon'

// --- Initialisation de la base ---
const initDatabase = async () => {
  const localDB = new PouchDB('database-infradon')
  storage.value = localDB
  console.log('Connecté à la base : ' + localDB.name) // Création d’un index sur le champ "title"

  await localDB.createIndex({
    index: {
      fields: ['title'],
      name: 'index-title',
      ddoc: 'index-title-doc',
    },
  }) // Création d’un index sur le champ "likes" (pour la flexibilité, même si le tri est client)
  await localDB.createIndex({
    index: {
      fields: ['likes'],
      name: 'index-likes',
      ddoc: 'index-likes-doc',
    },
  })
  console.log("Index 'title' et 'likes' créés ✔")
}

// --- Réplication -- -
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

// --- Tri Côté Client ---
const applySort = () => {
  if (sortKey.value === 'likes') {
    // Tri par likes décroissant (plus de likes en premier)
    gamesData.value.sort((a, b) => {
      if (b.likes !== a.likes) {
        return b.likes - a.likes
      }
      return a.title.localeCompare(b.title) // Stabilité par titre
    })
  } else {
    // Tri par titre croissant (par défaut)
    gamesData.value.sort((a, b) => a.title.localeCompare(b.title))
  }
}

const changeSortKey = (key: 'title' | 'likes') => {
  sortKey.value = key
  applySort()
}

// --- Récupération des données ---
const fetchData = async () => {
  if (!storage.value) return
  try {
    const result = await storage.value.allDocs({ include_docs: true })
    gamesData.value = result.rows
      .map((row) => {
        const doc = row.doc as Game
        if (doc && doc.title) {
          // Initialisation des nouveaux champs si le doc ne les a pas (rétrocompatibilité)
          doc.likes = doc.likes ?? 0
          doc.comments = doc.comments ?? []
          return doc
        }
        return null
      })
      .filter((doc): doc is Game => doc !== null)
    applySort() // Appliquer le tri après la récupération
    console.log('Données récupérées :', gamesData.value.length)
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
    gamesData.value = result.docs.map((doc) => {
      const game = doc as Game
      game.likes = game.likes ?? 0
      game.comments = game.comments ?? []
      return game
    })
    applySort() // Appliquer le tri après la recherche
    console.log('Résultats filtrés :', gamesData.value.length)
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
    likes: 0, // Nouveau champ
    comments: [], // Nouveau champ
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

// --- Modification du jeu (fonctions existantes) ---
const startEdit = (game: Game) => {
  if (editingComment.value) return alert('Veuillez finir la modification du commentaire en cours.')
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

// --- Suppression du jeu (fonction existante) ---
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

// --- Liker un jeu ---
const likeGame = async (game: Game) => {
  if (!storage.value) return
  try {
    const doc = (await storage.value.get(game._id)) as Game
    const updated: Game = {
      ...doc,
      _rev: doc._rev,
      likes: (doc.likes || 0) + 1,
    }
    await storage.value.put(updated)
    console.log('Jeu liké ✔')
    fetchData()
  } catch (err) {
    console.error('Erreur like :', err)
  }
}

// --- Ajouter un commentaire ---
const addComment = async (game: Game) => {
  if (!storage.value || !newCommentAuthor.value.trim() || !newCommentContent.value.trim()) {
    alert('Veuillez remplir votre nom et le commentaire.')
    return
  }
  try {
    const doc = (await storage.value.get(game._id)) as Game
    const newComment = {
      author: newCommentAuthor.value,
      content: newCommentContent.value,
      date: Date.now(),
    }
    const updated: Game = {
      ...doc,
      _rev: doc._rev,
      comments: [...(doc.comments || []), newComment],
    }
    await storage.value.put(updated)
    console.log('Commentaire ajouté ✔') // Réinitialisation et rafraîchissement
    newCommentAuthor.value = ''
    newCommentContent.value = ''
    fetchData()
  } catch (err) {
    console.error('Erreur ajout commentaire :', err)
  }
}

// --- Suppression d'un commentaire ---
const deleteComment = async (game: Game, commentDate: number) => {
  if (!storage.value || !confirm('Supprimer ce commentaire ?')) return
  try {
    const doc = (await storage.value.get(game._id)) as Game
    const updated: Game = {
      ...doc,
      _rev: doc._rev,
      comments: (doc.comments || []).filter((c) => c.date !== commentDate),
    }
    await storage.value.put(updated)
    console.log('Commentaire supprimé ✔')
    fetchData()
  } catch (err) {
    console.error('Erreur suppression commentaire :', err)
  }
}

// --- Modification d'un commentaire ---
const startEditComment = (game: Game, comment: Game['comments'][number]) => {
  if (editMode.value) return alert('Veuillez finir la modification du jeu en cours.')
  editingComment.value = {
    gameId: game._id,
    date: comment.date,
    content: comment.content,
    author: comment.author,
  }
}

const cancelEditComment = () => {
  editingComment.value = null
}

const saveEditComment = async () => {
  if (!storage.value || !editingComment.value || !editingComment.value.content.trim()) return

  try {
    const doc = (await storage.value.get(editingComment.value.gameId)) as Game
    const updatedComments = (doc.comments || []).map((c) => {
      if (c.date === editingComment.value!.date) {
        return { ...c, content: editingComment.value!.content }
      }
      return c
    })

    const updated: Game = {
      ...doc,
      _rev: doc._rev,
      comments: updatedComments,
    }

    await storage.value.put(updated)
    console.log('Commentaire modifié ✔')
    cancelEditComment()
    fetchData()
  } catch (err) {
    console.error('Erreur modification commentaire :', err)
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
      likes: Math.floor(Math.random() * 100), // Likes aléatoires
      comments: [
        { author: 'AI', content: `Super jeu numéro ${i}!`, date: Date.now() + i },
        { author: 'User', content: `Un peu surestimé.`, date: Date.now() + i + 1000 },
      ],
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

   
  <div style="margin-bottom: 20px">
        <label>Trier par :</label>    
    <button
      @click="changeSortKey('title')"
      :style="{ fontWeight: sortKey === 'title' ? 'bold' : 'normal' }"
    >
            Titre    
    </button>
       
    <button
      @click="changeSortKey('likes')"
      :style="{ fontWeight: sortKey === 'likes' ? 'bold' : 'normal' }"
    >
            👍 Likes    
    </button>
     
  </div>

         
  <div style="margin-bottom: 20px">
            <button @click="replicateFromDistant">Replicate FROM</button>        
    <button @click="replicateToDistant">Replicate TO</button>    
  </div>

         
  <div style="margin-bottom: 20px">
            <button @click="generateGames(50)">Générer 50 jeux</button>    
  </div>

   
  <hr />

   
  <div v-if="editingComment" class="edit-comment-form">
       
    <h3>
            ✏️ Modifier un commentaire sur "{{
        gamesData.find((g) => g._id === editingComment?.gameId)?.title
      }}"    
    </h3>
       
    <p>Auteur: **{{ editingComment.author }}**</p>
       
    <form @submit.prevent="saveEditComment">
           
      <textarea
        v-model="editingComment.content"
        rows="4"
        cols="50"
        required
        style="width: 100%"
      ></textarea>
            <br />
            <button type="submit">Sauvegarder Commentaire</button>      
      <button type="button" @click="cancelEditComment" style="margin-left: 10px">Annuler</button>  
       
    </form>
     
  </div>

       
  <div v-if="!editMode && !editingComment">
               
    <h2>Ajouter un jeu</h2>
               
    <form @submit.prevent="addGame">
                       
      <div><label>Titre: </label><input v-model="newGameTitle" required /></div>
                       
      <div><label>Éditeur: </label><input v-model="newGameEditor" required /></div>
                       
      <div><label>Pays: </label><input v-model="newGameCountry" /></div>
                       
      <div>
                        <label>Année: </label
        ><input type="number" v-model.number="newGameRelease" required />            
      </div>
                  <button type="submit">Ajouter</button>        
    </form>
           
  </div>

       
  <div v-if="editMode">
               
    <h2>Modifier le jeu</h2>
               
    <form @submit.prevent="saveEdit">
                       
      <div><label>Titre: </label><input v-model="editGameTitle" required /></div>
                       
      <div><label>Éditeur: </label><input v-model="editGameEditor" required /></div>
                       
      <div><label>Pays: </label><input v-model="editGameCountry" /></div>
                       
      <div>
                        <label>Année: </label
        ><input type="number" v-model.number="editGameRelease" required />            
      </div>
                  <button type="submit">Sauvegarder</button>            
      <button type="button" @click="cancelEdit">Annuler</button>        
    </form>
           
  </div>

   
  <hr />

   
  <div v-for="game in gamesData" :key="game._id" class="game-card">
       
    <div v-for="(g, i) in game.biblio.games" :key="i">
           
      <h2>{{ g.title }}</h2>
           
      <p>Éditeur: {{ g.editor }}</p>
           
      <p v-if="g.country">Pays: {{ g.country }}</p>
           
      <p>Année: {{ g.release }}</p>

           
      <div style="margin: 10px 0">
               
        <button @click="likeGame(game)" :disabled="editMode || !!editingComment">
                    👍 Liker ({{ game.likes }})        
        </button>
             
      </div>

           
      <button @click="startEdit(game)" :disabled="editMode || !!editingComment">
                Modifier Jeu      
      </button>
           
      <button @click="deleteGame(game)" class="delete-btn" :disabled="editMode || !!editingComment">
                Supprimer Jeu      
      </button>

           
      <hr style="margin: 15px 0" />

           
      <h4 class="comments-section">💬 Commentaires ({{ game.comments.length }})</h4>
           
      <div v-if="game.comments.length">
               
        <div v-for="comment in game.comments" :key="comment.date" class="comment-item">
                   
          <p class="comment-author">
                        {{ comment.author }}            
            <span class="comment-date">({{ new Date(comment.date).toLocaleDateString() }})</span>  
                   
          </p>
                   
          <p class="comment-content">{{ comment.content }}</p>
                   
          <button @click="startEditComment(game, comment)" :disabled="editMode || !!editingComment">
                        Modifier          
          </button>
                   
          <button
            @click="deleteComment(game, comment.date)"
            class="delete-comment-btn"
            :disabled="editMode || !!editingComment"
          >
                        Supprimer          
          </button>
                 
        </div>
             
      </div>
           
      <p v-else style="color: #6c757d">Aucun commentaire pour l'instant.</p>

           
      <div class="add-comment-form">
               
        <h5>Ajouter un commentaire</h5>
               
        <form @submit.prevent="addComment(game)">
                   
          <div>
            <label>Votre nom: </label
            ><input v-model="newCommentAuthor" required :disabled="editMode || !!editingComment" />
          </div>
                   
          <div>
                        <label>Commentaire: </label
            ><textarea
              v-model="newCommentContent"
              rows="3"
              required
              style="width: 100%"
              :disabled="editMode || !!editingComment"
            ></textarea>
                     
          </div>
                    <button type="submit" :disabled="editMode || !!editingComment">Poster</button>  
               
        </form>
             
      </div>
         
    </div>
     
  </div>
</template>

<style scoped>
/* Style global et boutons de contrôle */
body {
  font-family: 'Arial', sans-serif;
  background-color: #f4f7f6;
  color: #333;
  padding: 20px;
}

h1 {
  color: #007bff;
  border-bottom: 2px solid #007bff;
  padding-bottom: 10px;
}

h2 {
  color: #333;
  margin-top: 15px;
  font-size: 1.5em;
}

/* Styles pour les inputs et boutons */
input,
textarea {
  padding: 10px;
  margin: 5px 0 10px 0;
  border: 1px solid #ccc;
  border-radius: 4px;
  box-sizing: border-box;
  font-size: 1em;
}

button {
  padding: 10px 15px;
  margin-right: 10px;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  font-weight: bold;
  transition:
    background-color 0.2s,
    transform 0.1s;
}

button:hover:not(:disabled) {
  transform: translateY(-1px);
}

button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

/* Boutons de couleur par défaut (ex: Rechercher, Ajouter, Modifier, Liker) */
button:not(.delete-btn):not(.delete-comment-btn) {
  background-color: #007bff;
  color: white;
}

button:not(.delete-btn):not(.delete-comment-btn):hover {
  background-color: #0056b3;
}

/* Boutons de suppression spécifiques */
.delete-btn {
  background: #dc3545 !important;
  color: white;
  margin-left: 10px; /* Conserve l'espacement initial */
}

.delete-btn:hover:not(:disabled) {
  background: #c82333 !important;
}

.delete-comment-btn {
  background: darkred !important;
  color: white;
  margin-left: 10px; /* Conserve l'espacement initial */
}

.delete-comment-btn:hover:not(:disabled) {
  background: #8b0000 !important;
}

/* Formulaires d'ajout et de modification */
form div {
  margin-bottom: 10px;
}

form label {
  display: inline-block;
  width: 100px;
  font-weight: 600;
}

/* Conteneur du jeu */
.game-card {
  margin-top: 20px !important;
  border: 1px solid #e0e0e0 !important;
  padding: 20px !important;
  border-radius: 8px;
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.05);
  background-color: blueviolet !important;
}

.game-card h2 {
  color: #333;
  margin-top: 0;
  margin-bottom: 15px;
  font-size: 1.8em;
}

.game-card p {
  margin: 5px 0;
  font-size: 0.95em;
}

/* Section Commentaires */
.comments-section {
  color: #6c757d;
  border-bottom: 1px solid #eee;
  padding-bottom: 5px;
  margin-bottom: 15px;
  font-size: 1.1em;
}

.comment-item {
  border-left: 4px solid #17a2b8 !important; /* Couleur pour les commentaires */
  padding: 10px 15px !important;
  margin-bottom: 10px !important;
  background: #4d5357 !important;
  border-radius: 4px;
}

.comment-author {
  font-weight: bold;
  color: #17a2b8;
  margin: 0;
}

.comment-date {
  font-size: 0.8em;
  color: #6c757d;
}

.comment-content {
  margin: 5px 0 10px 0 !important;
}

/* Formulaire d'ajout de commentaire */
.add-comment-form {
  margin-top: 20px;
  padding: 15px;
  border: 1px dashed #17a2b8;
  background: #000000;
  border-radius: 6px;
}

/* Formulaire de modification de commentaire (orange) */
.edit-comment-form {
  border: 2px solid #ffc107 !important;
  background: #fffbe6 !important;
  padding: 15px;
  margin-bottom: 20px;
  border-radius: 8px;
}

.edit-comment-form h3 {
  color: #ffc107;
  border-bottom: 1px solid #ffc107;
  padding-bottom: 5px;
}
</style>
