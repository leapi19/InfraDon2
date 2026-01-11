<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'
import PouchFind from 'pouchdb-find'

PouchDB.plugin(PouchFind)

// Interface pour un jeu (sans les commentaires intégrés)
interface Game {
  _id: string
  _rev?: string
  type: 'game'
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
  _attachments?: PouchDB.Core.Attachments
}

// Interface pour un commentaire (collection séparée)
interface Comment {
  _id: string
  _rev?: string
  type: 'comment'
  gameId: string
  author: string
  content: string
  createdAt: number
}

// Référence aux bases de données
const gamesDB = ref<PouchDB.Database>()
const commentsDB = ref<PouchDB.Database>()

// Données stockées
const gamesData = ref<Game[]>([])
const commentsData = ref<Comment[]>([])

// Champs pour le formulaire d'ajout de jeu
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
  commentId: string
  gameId: string
  content: string
  author: string
} | null>(null)

// Référence pour le tri
const sortKey = ref<'title' | 'likes'>('title')

// Affichage des commentaires (tous ou premier uniquement)
const showAllComments = ref<{ [gameId: string]: boolean }>({})

// Afficher uniquement les 10 jeux les plus likés
const showOnlyTop10 = ref(false)

// Pour gérer les URLs des images attachées
const attachmentUrls = ref<{ [gameId: string]: string }>({})

const remoteURL = 'http://admin:admin@localhost:5984'

// --- Initialisation des bases ---
const initDatabase = async () => {
  // Base de données pour les jeux
  const localGamesDB = new PouchDB('games-collection')
  gamesDB.value = localGamesDB
  console.log('Connecté à la collection jeux : ' + localGamesDB.name)

  // Base de données pour les commentaires
  const localCommentsDB = new PouchDB('comments-collection')
  commentsDB.value = localCommentsDB
  console.log('Connecté à la collection commentaires : ' + localCommentsDB.name)

  // Index pour la collection jeux
  await localGamesDB.createIndex({
    index: {
      fields: ['type', 'title'],
      name: 'index-games-title',
      ddoc: 'index-games-title-doc',
    },
  })

  await localGamesDB.createIndex({
    index: {
      fields: ['type', 'likes'],
      name: 'index-games-likes',
      ddoc: 'index-games-likes-doc',
    },
  })

  // Index pour la collection commentaires
  await localCommentsDB.createIndex({
    index: {
      fields: ['type', 'gameId'],
      name: 'index-comments-gameid',
      ddoc: 'index-comments-gameid-doc',
    },
  })

  console.log('Index créés ✔')
}

// --- Réplication ---
const replicateFromDistant = () => {
  if (!gamesDB.value || !commentsDB.value) return

  console.log('Réplication FROM distante')

  // Répliquer les jeux
  gamesDB.value.replicate
    .from(`${remoteURL}/games-collection`)
    .on('complete', () => {
      console.log('Réplication jeux FROM terminée')
    })
    .on('error', (err) => console.error('Erreur réplication jeux FROM :', err))

  // Répliquer les commentaires
  commentsDB.value.replicate
    .from(`${remoteURL}/comments-collection`)
    .on('complete', () => {
      console.log('Réplication commentaires FROM terminée')
      fetchData()
    })
    .on('error', (err) => console.error('Erreur réplication commentaires FROM :', err))
}

const replicateToDistant = () => {
  if (!gamesDB.value || !commentsDB.value) return

  console.log('↗️ Réplication TO distante')

  // Répliquer les jeux
  gamesDB.value.replicate
    .to(`${remoteURL}/games-collection`)
    .on('complete', () => console.log('✔ Réplication jeux TO terminée'))
    .on('error', (err) => console.error('✗ Erreur réplication jeux TO :', err))

  // Répliquer les commentaires
  commentsDB.value.replicate
    .to(`${remoteURL}/comments-collection`)
    .on('complete', () => console.log('✔ Réplication commentaires TO terminée'))
    .on('error', (err) => console.error('✗ Erreur réplication commentaires TO :', err))
}

// --- Tri Côté Client ---
const applySort = () => {
  if (sortKey.value === 'likes') {
    gamesData.value.sort((a, b) => {
      if (b.likes !== a.likes) {
        return b.likes - a.likes
      }
      return a.title.localeCompare(b.title)
    })
  } else {
    gamesData.value.sort((a, b) => a.title.localeCompare(b.title))
  }
}

const changeSortKey = (key: 'title' | 'likes') => {
  sortKey.value = key
  applySort()
}

// Toggle affichage de tous les commentaires pour un jeu
const toggleShowAllComments = (gameId: string) => {
  showAllComments.value[gameId] = !showAllComments.value[gameId]
}

// Obtenir les commentaires à afficher (premier ou tous)
const getCommentsToDisplay = (gameId: string): Comment[] => {
  const comments = commentsData.value.filter((c) => c.gameId === gameId)

  if (showAllComments.value[gameId]) {
    return comments
  }
  return comments.length > 0 ? [comments[0]] : []
}

// Filtrer pour n'afficher que les 10 jeux les plus likés
const getGamesToDisplay = () => {
  if (showOnlyTop10.value) {
    return gamesData.value.slice(0, 10)
  }
  return gamesData.value
}

// --- Récupération des données ---
const fetchData = async () => {
  if (!gamesDB.value || !commentsDB.value) return
  try {
    // Récupérer les jeux
    const gamesResult = await gamesDB.value.find({
      selector: { type: 'game' },
    })
    gamesData.value = gamesResult.docs.map((doc) => {
      const game = doc as Game
      game.likes = game.likes ?? 0
      return game
    })
    applySort()

    // Récupérer les commentaires
    const commentsResult = await commentsDB.value.find({
      selector: { type: 'comment' },
    })
    commentsData.value = commentsResult.docs as Comment[]

    // Charger les attachments pour chaque jeu
    for (const game of gamesData.value) {
      await loadAttachment(game)
    }

    console.log(
      'Données récupérées :',
      gamesData.value.length,
      'jeux,',
      commentsData.value.length,
      'commentaires',
    )
  } catch (err) {
    console.error('Erreur fetchData :', err)
  }
}

// --- Recherche indexée ---
const searchByTitle = async () => {
  if (!gamesDB.value) return
  if (!searchTitle.value.trim()) {
    fetchData()
    return
  }
  try {
    const result = await gamesDB.value.find({
      selector: {
        type: 'game',
        title: { $regex: RegExp(searchTitle.value, 'i') },
      },
      use_index: 'index-games-title-doc',
    })
    gamesData.value = result.docs.map((doc) => {
      const game = doc as Game
      game.likes = game.likes ?? 0
      return game
    })
    applySort()
    console.log('Résultats filtrés :', gamesData.value.length)
  } catch (err) {
    console.error('Erreur recherche indexée :', err)
  }
}

// --- Ajout ---
const addGame = async () => {
  if (!gamesDB.value || !newGameTitle.value || !newGameEditor.value || !newGameRelease.value) {
    alert('Veuillez remplir tous les champs obligatoires')
    return
  }
  const gameObj: Omit<Game, '_attachments'> = {
    _id: `game_${Date.now()}`,
    type: 'game',
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
    likes: 0,
  }
  try {
    await gamesDB.value.put(gameObj)
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

// --- Modification du jeu ---
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
    !gamesDB.value ||
    !editingGameId.value ||
    !editGameTitle.value ||
    !editGameEditor.value ||
    !editGameRelease.value
  ) {
    alert('Veuillez remplir tous les champs obligatoires')
    return
  }
  try {
    const doc = (await gamesDB.value.get(editingGameId.value)) as Game
    const updated: Omit<Game, '_attachments'> & {
      _rev: string
      _attachments?: PouchDB.Core.Attachments
    } = {
      ...doc,
      _rev: doc._rev!,
      type: 'game',
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
    await gamesDB.value.put(updated)
    console.log('Jeu modifié ✔')
    cancelEdit()
    fetchData()
  } catch (err) {
    console.error('Erreur modification :', err)
  }
}

// --- Suppression du jeu ---
const deleteGame = async (game: Game) => {
  if (!gamesDB.value || !commentsDB.value) return
  if (!confirm(`Supprimer "${game.title}" ?`)) return
  try {
    // Supprimer tous les commentaires associés
    const comments = commentsData.value.filter((c) => c.gameId === game._id)
    for (const comment of comments) {
      await commentsDB.value.remove(comment._id, comment._rev!)
    }

    // Supprimer le jeu
    await gamesDB.value.remove(game._id, game._rev!)
    console.log('Jeu supprimé ✔')
    fetchData()
  } catch (err) {
    console.error('Erreur suppression :', err)
  }
}

// --- Liker un jeu ---
const likeGame = async (game: Game) => {
  if (!gamesDB.value) return
  try {
    const doc = (await gamesDB.value.get(game._id)) as Game
    const updated: Omit<Game, '_attachments'> & {
      _rev: string
      _attachments?: PouchDB.Core.Attachments
    } = {
      ...doc,
      _rev: doc._rev!,
      type: 'game',
      likes: (doc.likes || 0) + 1,
    }
    await gamesDB.value.put(updated)
    console.log('Jeu liké ✔')
    fetchData()
  } catch (err) {
    console.error('Erreur like :', err)
  }
}

// --- Ajouter un commentaire ---
const addComment = async (gameId: string) => {
  if (!commentsDB.value || !newCommentAuthor.value.trim() || !newCommentContent.value.trim()) {
    alert('Veuillez remplir votre nom et le commentaire.')
    return
  }
  try {
    const comment: Comment = {
      _id: `comment_${Date.now()}`,
      type: 'comment',
      gameId: gameId,
      author: newCommentAuthor.value,
      content: newCommentContent.value,
      createdAt: Date.now(),
    }
    await commentsDB.value.put(comment)
    console.log('Commentaire ajouté ✔')
    newCommentAuthor.value = ''
    newCommentContent.value = ''
    fetchData()
  } catch (err) {
    console.error('Erreur ajout commentaire :', err)
  }
}

// --- Suppression d'un commentaire ---
const deleteComment = async (comment: Comment) => {
  if (!commentsDB.value || !confirm('Supprimer ce commentaire ?')) return
  try {
    await commentsDB.value.remove(comment._id, comment._rev!)
    console.log('Commentaire supprimé ✔')
    fetchData()
  } catch (err) {
    console.error('Erreur suppression commentaire :', err)
  }
}

// --- Modification d'un commentaire ---
const startEditComment = (comment: Comment) => {
  if (editMode.value) return alert('Veuillez finir la modification du jeu en cours.')
  editingComment.value = {
    commentId: comment._id,
    gameId: comment.gameId,
    content: comment.content,
    author: comment.author,
  }
}

const cancelEditComment = () => {
  editingComment.value = null
}

const saveEditComment = async () => {
  if (!commentsDB.value || !editingComment.value || !editingComment.value.content.trim()) return

  try {
    const doc = (await commentsDB.value.get(editingComment.value.commentId)) as Comment
    const updated: Comment = {
      ...doc,
      _rev: doc._rev!,
      type: 'comment',
      content: editingComment.value.content,
    }

    await commentsDB.value.put(updated)
    console.log('Commentaire modifié ✔')
    cancelEditComment()
    fetchData()
  } catch (err) {
    console.error('Erreur modification commentaire :', err)
  }
}

// --- Gestion des attachments (médias) ---
const loadAttachment = async (game: Game) => {
  if (!gamesDB.value) return

  const oldUrl = attachmentUrls.value[game._id]
  if (oldUrl) {
    URL.revokeObjectURL(oldUrl)
  }

  try {
    if (game._attachments && Object.keys(game._attachments).length > 0) {
      const attachmentName = Object.keys(game._attachments)[0]
      if (attachmentName) {
        const blob = (await gamesDB.value.getAttachment(
          game._id,
          attachmentName,
        )) as unknown as Blob
        attachmentUrls.value[game._id] = URL.createObjectURL(blob)
      }
    }
  } catch (err) {
    console.error('Erreur chargement attachment:', err)
  }
}

const addAttachment = async (game: Game, event: Event) => {
  if (!gamesDB.value) return

  const input = event.target as HTMLInputElement
  const file = input.files?.[0]

  if (!file) return

  if (!file.type.startsWith('image/')) {
    alert('Veuillez sélectionner une image')
    return
  }

  try {
    const doc = (await gamesDB.value.get(game._id)) as Game

    await gamesDB.value.putAttachment(game._id, 'media', doc._rev!, file, file.type)

    console.log('Media ajouté ✔')
    fetchData()
  } catch (err) {
    console.error('Erreur ajout media:', err)
  }
}

const deleteAttachment = async (game: Game) => {
  if (!gamesDB.value) return
  if (!confirm('Supprimer le média ?')) return

  try {
    const doc = (await gamesDB.value.get(game._id)) as Game

    if (doc._attachments && Object.keys(doc._attachments).length > 0) {
      const attachmentName = Object.keys(doc._attachments)[0]

      if (attachmentName && doc._rev) {
        await gamesDB.value.removeAttachment(game._id, attachmentName, doc._rev)

        const oldUrl = attachmentUrls.value[game._id]
        if (oldUrl) {
          URL.revokeObjectURL(oldUrl)
          delete attachmentUrls.value[game._id]
        }

        console.log('Media supprimé ✔')
        fetchData()
      }
    }
  } catch (err) {
    console.error('Erreur suppression media:', err)
  }
}

// Obtenir le nombre de commentaires pour un jeu
const getCommentsCount = (gameId: string): number => {
  return commentsData.value.filter((c) => c.gameId === gameId).length
}

onMounted(async () => {
  console.log('=> Composant initialisé')
  await initDatabase()
  fetchData()
})
</script>

<template>
  <h1>🎮 Liste des jeux - 2 Collections</h1>

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

  <!-- Tri -->
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

  <!-- Bouton pour afficher les 10 plus likés -->
  <div style="margin-bottom: 20px">
    <button
      @click="showOnlyTop10 = !showOnlyTop10"
      :style="{
        background: showOnlyTop10 ? '#28a745' : '#6c757d',
        color: 'white',
        fontWeight: 'bold',
      }"
    >
      {{ showOnlyTop10 ? '✨ Top 10 activé' : '✨ Afficher Top 10' }}
    </button>
  </div>

  <!-- Réplication -->
  <div style="margin-bottom: 20px">
    <button @click="replicateFromDistant">⬇️ Replicate FROM</button>
    <button @click="replicateToDistant">⬆️ Replicate TO</button>
  </div>

  <hr />

  <!-- Formulaire modification commentaire -->
  <div v-if="editingComment" class="edit-comment-form">
    <h3>✏️ Modifier un commentaire</h3>
    <p>
      Auteur: <strong>{{ editingComment.author }}</strong>
    </p>
    <textarea
      v-model="editingComment.content"
      rows="4"
      required
      style="width: 100%; padding: 10px; margin: 10px 0"
    ></textarea>
    <br />
    <button @click="saveEditComment">Sauvegarder</button>
    <button @click="cancelEditComment" style="margin-left: 10px">Annuler</button>
  </div>

  <!-- Formulaire ajout jeu -->
  <div v-if="!editMode && !editingComment">
    <h2>Ajouter un jeu</h2>
    <div>
      <div><label>Titre: </label><input v-model="newGameTitle" required /></div>
      <div><label>Éditeur: </label><input v-model="newGameEditor" required /></div>
      <div><label>Pays: </label><input v-model="newGameCountry" /></div>
      <div>
        <label>Année: </label><input type="number" v-model.number="newGameRelease" required />
      </div>
      <button @click="addGame">Ajouter</button>
    </div>
  </div>

  <!-- Formulaire modification jeu -->
  <div v-if="editMode">
    <h2>Modifier le jeu</h2>
    <div>
      <div><label>Titre: </label><input v-model="editGameTitle" required /></div>
      <div><label>Éditeur: </label><input v-model="editGameEditor" required /></div>
      <div><label>Pays: </label><input v-model="editGameCountry" /></div>
      <div>
        <label>Année: </label><input type="number" v-model.number="editGameRelease" required />
      </div>
      <button @click="saveEdit">Sauvegarder</button>
      <button @click="cancelEdit">Annuler</button>
    </div>
  </div>

  <hr />

  <!-- Liste des jeux -->
  <div v-for="game in getGamesToDisplay()" :key="game._id" class="game-card">
    <div v-for="(g, i) in game.biblio.games" :key="i">
      <h2>{{ g.title }}</h2>

      <!-- Affichage du média si présent -->
      <div v-if="attachmentUrls[game._id]" class="media-container">
        <img :src="attachmentUrls[game._id]" :alt="g.title" class="game-image" />
        <button
          @click="deleteAttachment(game)"
          class="delete-media-btn"
          :disabled="editMode || !!editingComment"
        >
          🗑️ Supprimer le média
        </button>
      </div>

      <!-- Formulaire d'ajout de média -->
      <div v-else class="media-upload">
        <label class="file-upload-label">
          📷 Ajouter une image
          <input
            type="file"
            accept="image/*"
            @change="addAttachment(game, $event)"
            :disabled="editMode || !!editingComment"
            style="display: none"
          />
        </label>
      </div>

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

      <h4 class="comments-section">💬 Commentaires ({{ getCommentsCount(game._id) }})</h4>

      <div v-if="getCommentsCount(game._id) > 0">
        <div v-for="c in getCommentsToDisplay(game._id)" :key="c._id" class="comment-item">
          <p class="comment-author">
            {{ c.author }}
            <span class="comment-date">({{ new Date(c.createdAt).toLocaleDateString() }})</span>
          </p>
          <p class="comment-content">{{ c.content }}</p>
          <button @click="startEditComment(c)" :disabled="editMode || !!editingComment">
            Modifier
          </button>
          <button
            @click="deleteComment(c)"
            class="delete-comment-btn"
            :disabled="editMode || !!editingComment"
          >
            Supprimer
          </button>
        </div>

        <!-- Bouton pour afficher tous les commentaires -->
        <div v-if="getCommentsCount(game._id) > 1" style="margin-top: 10px">
          <button
            @click="toggleShowAllComments(game._id)"
            style="background: #17a2b8; color: white"
            :disabled="editMode || !!editingComment"
          >
            {{
              showAllComments[game._id]
                ? '▲ Masquer les commentaires'
                : `▼ Afficher tous les commentaires (${getCommentsCount(game._id)})`
            }}
          </button>
        </div>
      </div>

      <p v-else style="color: #6c757d">Aucun commentaire pour l'instant.</p>

      <div class="add-comment-form">
        <h5>Ajouter un commentaire</h5>
        <div>
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
          <button @click="addComment(game._id)" :disabled="editMode || !!editingComment">
            Poster
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
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

button:not(.delete-btn):not(.delete-comment-btn):not(.delete-media-btn) {
  background-color: #007bff;
  color: white;
}

button:not(.delete-btn):not(.delete-comment-btn):not(.delete-media-btn):hover {
  background-color: #0056b3;
}

.delete-btn {
  background: #dc3545 !important;
  color: white;
  margin-left: 10px;
}

.delete-btn:hover:not(:disabled) {
  background: #c82333 !important;
}

.delete-comment-btn {
  background: darkred !important;
  color: white;
  margin-left: 10px;
}

.delete-comment-btn:hover:not(:disabled) {
  background: #8b0000 !important;
}

div > div {
  margin-bottom: 10px;
}

label {
  display: inline-block;
  width: 100px;
  font-weight: 600;
}

.game-card {
  margin-top: 20px !important;
  border: 1px solid #e0e0e0 !important;
  padding: 20px !important;
  border-radius: 8px;
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.05);
  background-color: #f0e6ff !important;
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
.comments-section {
  color: #6c757d;
  border-bottom: 1px solid #eee;
  padding-bottom: 5px;
  margin-bottom: 15px;
  font-size: 1.1em;
}
.comment-item {
  border-left: 4px solid #17a2b8 !important;
  padding: 10px 15px !important;
  margin-bottom: 10px !important;
  background: #f8f9fa !important;
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
  color: #333 !important;
}
.add-comment-form {
  margin-top: 20px;
  padding: 15px;
  border: 1px dashed #17a2b8;
  background: #000000;
  border-radius: 6px;
}
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
.media-container {
  margin: 15px 0;
  text-align: center;
}
.game-image {
  max-width: 100%;
  max-height: 300px;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  margin-bottom: 10px;
}
.delete-media-btn {
  background: #dc3545 !important;
  color: white;
  padding: 8px 15px;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  font-weight: bold;
}
.delete-media-btn:hover:not(:disabled) {
  background: #c82333 !important;
}
.media-upload {
  margin: 15px 0;
}
.file-upload-label {
  display: inline-block;
  padding: 10px 20px;
  background: #28a745;
  color: white;
  border-radius: 5px;
  cursor: pointer;
  font-weight: bold;
  transition: background-color 0.2s;
}
.file-upload-label:hover {
  background: #218838;
}
.file-upload-label input[type='file']:disabled + label {
  opacity: 0.6;
  cursor: not-allowed;
}
</style>
