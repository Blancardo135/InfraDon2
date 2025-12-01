<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'
import PouchDBFind from 'pouchdb-find'
PouchDB.plugin(PouchDBFind)

interface Comment {
  id: string
  text: string
  createdAt: string
}

interface Message {
  id: string
  text: string
  comments: Comment[]
  createdAt: string
}

interface Character {
  name: string
  age: number
  affiliation: string
  lightsaber: boolean
  likes: number
  messages?: Message[]
}

interface CharacterDoc {
  _id: string
  _rev?: string
  type?: string 
  characters?: Character[]
 
  name?: string
  age?: number
  affiliation?: string
  lightsaber?: boolean
  likes?: number
  messages?: Message[]
}

const COUCH_REMOTE = 'http://RomainBlanchard:admin@localhost:5984/coachdb';

const storage = ref<any>(null)
const characters = ref<{ data: Character; docId: string; docRev: string }[]>([])
const isOnline = ref(true)
let syncHandler: any = null

const newCharacter = ref<Character>({
  name: '',
  age: 0,
  affiliation: '',
  lightsaber: false,
  likes: 0,
  messages: []
})

const editingIndex = ref<number | null>(null)
const editingCharacter = ref<Character | null>(null)

const newMessageText = ref<{ [key: number]: string }>({})
const editingMessageIndex = ref<{ charIndex: number; msgIndex: number } | null>(null)
const editingMessageText = ref('')

const newCommentText = ref<{ [key: string]: string }>({})
const editingCommentIndex = ref<{ charIndex: number; msgIndex: number; commentIndex: number } | null>(null)
const editingCommentText = ref('')


const visibleComments = ref<{ [key: string]: boolean }>({})

const searchMessageText = ref('')
const sortByLikes = ref(false)
const searchAffiliation = ref('')

function extractCharactersFromDoc(doc: CharacterDoc) {
  const out: { data: Character; docId: string; docRev?: string }[] = []

  if (doc.type === 'character') {
    const c: Character = {
      name: doc.name || '',
      age: doc.age || 0,
      affiliation: doc.affiliation || '',
      lightsaber: !!doc.lightsaber,
      likes: doc.likes === undefined ? 0 : doc.likes,
      messages: Array.isArray(doc.messages) ? doc.messages : []
    }
    out.push({ data: c, docId: doc._id, docRev: doc._rev })
    return out
  }

  if (Array.isArray((doc as any).characters)) {
    (doc as any).characters.forEach((char: any) => {
      const c: Character = {
        name: char.name || '',
        age: char.age || 0,
        affiliation: char.affiliation || '',
        lightsaber: !!char.lightsaber,
        likes: char.likes === undefined ? 0 : char.likes,
        messages: Array.isArray(char.messages) ? char.messages : []
      }
      out.push({ data: c, docId: doc._id, docRev: doc._rev })
    })
    return out
  }
  return out
}

const initDatabase = () => {
  console.log('=> Connexion à la base de données')
  const localDB = new PouchDB('infradon-blaro')
  storage.value = localDB
  console.log('Base locale :', localDB?.name)

 
  createIndex()
  fetchData()
  startSync()
}

const startSync = () => {
  if (!storage.value) return
  if (syncHandler) return

  console.log('Synchronisation ACTIVÉE ->', COUCH_REMOTE)
  syncHandler = storage.value.sync(COUCH_REMOTE, {
    live: true,
    retry: true
  })
    .on('change', (info: any) => {
      console.log('Données synchronisées', info)
      if (sortByLikes.value) {
        fetchData(true)
      } else {
        fetchData()
      }
    })
    .on('paused', (err: any) => {
    })
    .on('active', () => {

    })
    .on('error', (err: any) => {
      console.error('Erreur sync :', err)
    })
}

const stopSync = () => {
  if (syncHandler && syncHandler.cancel) {
    console.log('Synchronisation STOPPÉE')
    syncHandler.cancel()
    syncHandler = null
  }
}

const toggleOnline = () => {
  isOnline.value = !isOnline.value
  console.log('Mode en ligne ?', isOnline.value)

  if (isOnline.value) startSync()
  else stopSync()
}

const createIndex = async () => {
  if (!storage.value) return
  try {

    await storage.value.createIndex({ index: { fields: ['type', 'affiliation'] } })
    await storage.value.createIndex({ index: { fields: ['type', 'likes'] } })
    await storage.value.createIndex({ index: { fields: ['type', 'messages.text'] } })
    console.log('Index créés')
  } catch (err) {
    console.error('Erreur création index :', err)
  }
}
const fetchData = async (onlyTop = false) => {
  if (!storage.value) return

  try {
    const result = await storage.value.allDocs({ include_docs: true })

    const allCharacters: { data: Character; docId: string; docRev: string }[] = []

    result.rows.forEach((row: any) => {
      if (!row.doc) return
      const extracted = extractCharactersFromDoc(row.doc as CharacterDoc)
      extracted.forEach((entry) => {
        if (!entry.data.messages) entry.data.messages = []
        if (entry.data.likes === undefined) entry.data.likes = 0
        allCharacters.push({
          data: entry.data,
          docId: entry.docId,
          docRev: entry.docRev || row.doc._rev
        })
      })
    })

    if (onlyTop) {
      allCharacters.sort((a, b) => (b.data.likes || 0) - (a.data.likes || 0))
      characters.value = allCharacters.slice(0, 10)
    } else {
      characters.value = allCharacters
    }
  } catch (err) {
    console.error('Erreur lors de la récupération :', err)
  }
}

//crud
const addCharacter = async () => {
  if (!storage.value) return
  try {
    const doc: CharacterDoc = {
      _id: 'character:' + new Date().toISOString(),
      type: 'character',
      name: newCharacter.value.name,
      age: newCharacter.value.age,
      affiliation: newCharacter.value.affiliation,
      lightsaber: newCharacter.value.lightsaber,
      likes: newCharacter.value.likes || 0,
      messages: Array.isArray(newCharacter.value.messages) ? newCharacter.value.messages : []
    }
    const response = await storage.value.put(doc)
    characters.value.push({
      data: {
        name: doc.name!,
        age: doc.age!,
        affiliation: doc.affiliation!,
        lightsaber: !!doc.lightsaber,
        likes: doc.likes || 0,
        messages: doc.messages || []
      },
      docId: doc._id,
      docRev: response.rev
    })
    newCharacter.value = {
      name: '',
      age: 0,
      affiliation: '',
      lightsaber: false,
      likes: 0,
      messages: []
    }
  } catch (err) {
    console.error("Erreur lors de l'ajout :", err)
  }
}

const startEdit = (index: number) => {
  editingIndex.value = index
  editingCharacter.value = { ...characters.value[index].data }
}

const cancelEdit = () => {
  editingIndex.value = null
  editingCharacter.value = null
}

const saveEdit = async (index: number) => {
  if (!storage.value || !editingCharacter.value) return
  try {
    const charToUpdate = characters.value[index]
    const doc: CharacterDoc = await storage.value.get(charToUpdate.docId)

    if (doc.type === 'character') {
      doc.name = editingCharacter.value.name
      doc.age = editingCharacter.value.age
      doc.affiliation = editingCharacter.value.affiliation
      doc.lightsaber = editingCharacter.value.lightsaber
      // on ne touche pas likes/messages ici si non fourni
      if (!doc.messages) doc.messages = editingCharacter.value.messages || []
      if (editingCharacter.value.likes !== undefined) doc.likes = editingCharacter.value.likes
    } else if (Array.isArray(doc.characters)) {
      const idx = doc.characters.findIndex((c: any) =>
        c.name === charToUpdate.data.name &&
        c.age === charToUpdate.data.age &&
        c.affiliation === charToUpdate.data.affiliation
      )
      const replaceIndex = idx >= 0 ? idx : 0
      doc.characters[replaceIndex] = { ...editingCharacter.value }
    } else {
      doc.type = 'character'
      doc.name = editingCharacter.value.name
      doc.age = editingCharacter.value.age
      doc.affiliation = editingCharacter.value.affiliation
      doc.lightsaber = editingCharacter.value.lightsaber
      doc.likes = editingCharacter.value.likes || 0
      doc.messages = editingCharacter.value.messages || []
    }

    const response = await storage.value.put(doc)
    characters.value[index] = {
      data: { ...editingCharacter.value },
      docId: charToUpdate.docId,
      docRev: response.rev
    }
    cancelEdit()
  } catch (err) {
    console.error('Erreur lors de la modification :', err)
  }
}

const deleteCharacter = async (index: number) => {
  if (!storage.value) return
  const confirmDelete = confirm('Êtes-vous sûr de vouloir supprimer ce personnage ?')
  if (!confirmDelete) return

  try {
    const charToDelete = characters.value[index]
    const doc = await storage.value.get(charToDelete.docId)
    await storage.value.remove(doc)
    console.log('Personnage supprimé avec succès')
    characters.value.splice(index, 1)
  } catch (err) {
    console.error('Erreur lors de la suppression :', err)
  }
}


const addMessage = async (charIndex: number) => {
  if (!storage.value || !newMessageText.value[charIndex]) return
  try {
    const char = characters.value[charIndex]
    const doc: CharacterDoc = await storage.value.get(char.docId)

    const newMessage: Message = {
      id: new Date().toISOString(),
      text: newMessageText.value[charIndex],
      comments: [],
      createdAt: new Date().toISOString()
    }

    if (doc.type === 'character') {
      if (!Array.isArray(doc.messages)) doc.messages = []
      doc.messages.push(newMessage)
    } else if (Array.isArray(doc.characters) && doc.characters.length > 0) {
      if (!Array.isArray(doc.characters[0].messages)) doc.characters[0].messages = []
      doc.characters[0].messages.push(newMessage)
    } else {
      doc.type = 'character'
      doc.messages = [newMessage]
      doc.name = doc.name || 'unknown'
      doc.age = doc.age || 0
      doc.affiliation = doc.affiliation || ''
      doc.lightsaber = !!doc.lightsaber
      doc.likes = doc.likes || 0
    }

    const response = await storage.value.put(doc)
    const extracted = extractCharactersFromDoc(doc)
    characters.value = characters.value.map(c =>
      c.docId === doc._id ? { data: extracted[0].data, docId: doc._id, docRev: response.rev } : c
    )
    newMessageText.value[charIndex] = ''
  } catch (err) {
    console.error('Erreur ajout message :', err)
  }
}

const startEditMessage = (charIndex: number, msgIndex: number) => {
  editingMessageIndex.value = { charIndex, msgIndex }
  editingMessageText.value = characters.value[charIndex].data.messages![msgIndex].text
}

const cancelEditMessage = () => {
  editingMessageIndex.value = null
  editingMessageText.value = ''
}

const saveEditMessage = async () => {
  if (!storage.value || !editingMessageIndex.value) return
  try {
    const { charIndex, msgIndex } = editingMessageIndex.value
    const char = characters.value[charIndex]
    const doc: CharacterDoc = await storage.value.get(char.docId)

    if (doc.type === 'character') {
      if (!doc.messages) doc.messages = []
      doc.messages[msgIndex].text = editingMessageText.value
    } else if (Array.isArray(doc.characters) && doc.characters[0]) {
      if (!doc.characters[0].messages) doc.characters[0].messages = []
      doc.characters[0].messages[msgIndex].text = editingMessageText.value
    }

    const response = await storage.value.put(doc)
    const extracted = extractCharactersFromDoc(doc)
    characters.value = characters.value.map(c =>
      c.docId === doc._id ? { data: extracted[0].data, docId: doc._id, docRev: response.rev } : c
    )
    cancelEditMessage()
  } catch (err) {
    console.error('Erreur modification message :', err)
  }
}

const deleteMessage = async (charIndex: number, msgIndex: number) => {
  if (!storage.value) return
  if (!confirm('Supprimer ce message ?')) return
  try {
    const char = characters.value[charIndex]
    const doc: CharacterDoc = await storage.value.get(char.docId)

    if (doc.type === 'character') {
      doc.messages!.splice(msgIndex, 1)
    } else if (Array.isArray(doc.characters) && doc.characters[0]) {
      doc.characters[0].messages!.splice(msgIndex, 1)
    }

    const response = await storage.value.put(doc)
    const extracted = extractCharactersFromDoc(doc)
    characters.value = characters.value.map(c =>
      c.docId === doc._id ? { data: extracted[0].data, docId: doc._id, docRev: response.rev } : c
    )
  } catch (err) {
    console.error('Erreur suppression message :', err)
  }
}

const likeCharacter = async (charIndex: number) => {
  if (!storage.value) return
  try {
    const char = characters.value[charIndex]
    const doc: CharacterDoc = await storage.value.get(char.docId)

    if (doc.type === 'character') {
      if (doc.likes === undefined) doc.likes = 0
      doc.likes++
    } else if (Array.isArray(doc.characters) && doc.characters[0]) {
      if (doc.characters[0].likes === undefined) doc.characters[0].likes = 0
      doc.characters[0].likes++
    }

    const response = await storage.value.put(doc)
    const extracted = extractCharactersFromDoc(doc)
    characters.value = characters.value.map(c =>
      c.docId === doc._id ? { data: extracted[0].data, docId: doc._id, docRev: response.rev } : c
    )

    if (sortByLikes.value) {
      setTimeout(() => fetchData(true), 120)
    }
  } catch (err) {
    console.error('Erreur like personnage :', err)
  }
}

/**
 * Commentaires
 */
const addComment = async (charIndex: number, msgIndex: number) => {
  const key = `${charIndex}-${msgIndex}`
  if (!storage.value || !newCommentText.value[key]) return
  try {
    const char = characters.value[charIndex]
    const doc: CharacterDoc = await storage.value.get(char.docId)
    const newComment: Comment = {
      id: new Date().toISOString(),
      text: newCommentText.value[key],
      createdAt: new Date().toISOString()
    }

    if (doc.type === 'character') {
      if (!doc.messages) doc.messages = []
      if (!doc.messages[msgIndex]) doc.messages[msgIndex] = { id: new Date().toISOString(), text: '', comments: [], createdAt: new Date().toISOString() }
      if (!doc.messages[msgIndex].comments) doc.messages[msgIndex].comments = []
      doc.messages[msgIndex].comments.push(newComment)
    } else if (Array.isArray(doc.characters) && doc.characters[0]) {
      if (!doc.characters[0].messages) doc.characters[0].messages = []
      if (!doc.characters[0].messages[msgIndex]) doc.characters[0].messages[msgIndex] = { id: new Date().toISOString(), text: '', comments: [], createdAt: new Date().toISOString() }
      if (!doc.characters[0].messages[msgIndex].comments) doc.characters[0].messages[msgIndex].comments = []
      doc.characters[0].messages[msgIndex].comments.push(newComment)
    }

    const response = await storage.value.put(doc)
    const extracted = extractCharactersFromDoc(doc)
    characters.value = characters.value.map(c =>
      c.docId === doc._id ? { data: extracted[0].data, docId: doc._id, docRev: response.rev } : c
    )

    newCommentText.value[key] = ''
    visibleComments.value[key] = true
  } catch (err) {
    console.error('Erreur ajout commentaire :', err)
  }
}

const startEditComment = (charIndex: number, msgIndex: number, commentIndex: number) => {
  editingCommentIndex.value = { charIndex, msgIndex, commentIndex }
  editingCommentText.value = characters.value[charIndex].data.messages![msgIndex].comments[commentIndex].text
}

const cancelEditComment = () => {
  editingCommentIndex.value = null
  editingCommentText.value = ''
}

const saveEditComment = async () => {
  if (!storage.value || !editingCommentIndex.value) return
  try {
    const { charIndex, msgIndex, commentIndex } = editingCommentIndex.value
    const char = characters.value[charIndex]
    const doc: CharacterDoc = await storage.value.get(char.docId)

    if (doc.type === 'character') {
      doc.messages![msgIndex].comments[commentIndex].text = editingCommentText.value
    } else if (Array.isArray(doc.characters) && doc.characters[0]) {
      doc.characters[0].messages![msgIndex].comments[commentIndex].text = editingCommentText.value
    }

    const response = await storage.value.put(doc)
    const extracted = extractCharactersFromDoc(doc)
    characters.value = characters.value.map(c =>
      c.docId === doc._id ? { data: extracted[0].data, docId: doc._id, docRev: response.rev } : c
    )
    cancelEditComment()
  } catch (err) {
    console.error('Erreur modification commentaire :', err)
  }
}

const deleteComment = async (charIndex: number, msgIndex: number, commentIndex: number) => {
  if (!storage.value) return
  if (!confirm('Supprimer ce commentaire ?')) return
  try {
    const char = characters.value[charIndex]
    const doc: CharacterDoc = await storage.value.get(char.docId)

    if (doc.type === 'character') {
      doc.messages![msgIndex].comments.splice(commentIndex, 1)
    } else if (Array.isArray(doc.characters) && doc.characters[0]) {
      doc.characters[0].messages![msgIndex].comments.splice(commentIndex, 1)
    }

    const response = await storage.value.put(doc)
    const extracted = extractCharactersFromDoc(doc)
    characters.value = characters.value.map(c =>
      c.docId === doc._id ? { data: extracted[0].data, docId: doc._id, docRev: response.rev } : c
    )
  } catch (err) {
    console.error('Erreur suppression commentaire :', err)
  }
}


const toggleCommentVisibility = (charIndex: number, msgIndex: number) => {
  const key = `${charIndex}-${msgIndex}`
  visibleComments.value[key] = !visibleComments.value[key]
}
const searchMessages = async () => {
  if (!storage.value) return
  if (!searchMessageText.value) {
    sortByLikes.value = false
    fetchData()
    return
  }
  try {
    const needle = searchMessageText.value.toLowerCase()
    const result = await storage.value.allDocs({ include_docs: true })
    const allCharacters: { data: Character; docId: string; docRev: string }[] = []
    result.rows.forEach((row: any) => {
      if (!row.doc) return
      const extracted = extractCharactersFromDoc(row.doc as CharacterDoc)
      extracted.forEach((entry) => {
        entry.data.messages = entry.data.messages || []
        const found = entry.data.messages.some(m => (m.text || '').toLowerCase().includes(needle))
        if (found) {
          allCharacters.push({
            data: entry.data,
            docId: entry.docId,
            docRev: entry.docRev
          })
        }
      })
    })
    characters.value = allCharacters
  } catch (err) {
    console.error('Erreur recherche messages :', err)
  }
}
const searchByAffiliation = async () => {
  if (!storage.value) return
  if (!searchAffiliation.value) {
    sortByLikes.value = false
    fetchData()
    return
  }
  const needle = searchAffiliation.value.toLowerCase()
  try {
    const result = await storage.value.allDocs({ include_docs: true })
    const allCharacters: { data: Character; docId: string; docRev: string }[] = []
    result.rows.forEach((row: any) => {
      if (!row.doc) return
      const extracted = extractCharactersFromDoc(row.doc as CharacterDoc)
      extracted.forEach((entry) => {
        if ((entry.data.affiliation || '').toLowerCase() === needle) {
          allCharacters.push({
            data: entry.data,
            docId: entry.docId,
            docRev: entry.docRev
          })
        }
      })
    })
    characters.value = allCharacters
  } catch (err) {
    console.error('Erreur recherche affiliation :', err)
  }
}

const toggleSortByLikes = async () => {
  if (sortByLikes.value === true) {
    sortByLikes.value = false
    fetchData()
    return
  }
  sortByLikes.value = true

  fetchData(true)
}

onMounted(() => {
  initDatabase()
})
</script>

<template>
  <h1 class="text-2xl font-bold mb-4">Liste des personnages</h1>
  <div class="mb-4">
    <button
      @click="toggleOnline"
      class="px-4 py-2 rounded text-white"
      :class="isOnline ? 'bg-green-600' : 'bg-gray-600'"
    >
      {{ isOnline ? 'Mode Online ✔' : 'Mode Offline ✖' }}
    </button>
  </div>

  <form @submit.prevent="addCharacter" class="p-4 mb-6 border rounded bg-gray-50 space-y-2">
    <h2 class="font-semibold text-lg mb-2">Ajouter un personnage</h2>
    <input v-model="newCharacter.name" placeholder="Nom" class="border p-2 rounded w-full" required />
    <input v-model.number="newCharacter.age" placeholder="Âge" type="number" class="border p-2 rounded w-full" required />
    <input v-model="newCharacter.affiliation" placeholder="Affiliation" class="border p-2 rounded w-full" required />
    <label class="flex items-center space-x-2">
      <input type="checkbox" v-model="newCharacter.lightsaber" />
      <span>Possède un sabre laser</span>
    </label>
    <button type="submit" class="bg-blue-600 text-white px-4 py-2 rounded hover:bg-blue-700 transition">
      Ajouter
    </button>
  </form>

  <div class="p-4 mb-6 border rounded bg-gray-50 space-y-2">
    <h2 class="font-semibold text-lg mb-2">Recherche par affiliation</h2>
    <div class="flex items-center space-x-2">
      <input v-model="searchAffiliation" placeholder="Ex: Jedi" class="border p-2 rounded w-full" />
      <button @click="searchByAffiliation" class="bg-purple-600 text-white px-4 py-2 rounded hover:bg-purple-700 transition">
        Rechercher
      </button>
    </div>
  </div>

  <div class="p-4 mb-6 border rounded bg-gray-50 space-y-2">
    <h2 class="font-semibold text-lg mb-2">Recherche et Filtres</h2>
    <div class="flex items-center space-x-2">
      <input v-model="searchMessageText" placeholder="Rechercher un message..." class="border p-2 rounded w-full" />
      <button @click="searchMessages" class="bg-indigo-600 text-white px-4 py-2 rounded hover:bg-indigo-700 transition">
        Rechercher
      </button>
    </div>
    <button
      @click="toggleSortByLikes"
      class="mt-2 px-4 py-2 rounded text-white transition"
      :class="sortByLikes ? 'bg-yellow-600 ring-2 ring-yellow-400' : 'bg-gray-500 hover:bg-gray-600'"
    >
      {{ sortByLikes ? 'Top 10 Likes Activé (Cliquez pour désactiver)' : 'Afficher le Top 10 Likes' }}
    </button>
  </div>

  <section>
    <article
      v-for="(char, index) in characters"
      :key="char.docId + '-' + index"
      class="p-4 border-b hover:bg-gray-100 transition"
    >
      <div v-if="editingIndex !== index">
        <div class="flex items-center justify-between">
          <h2 class="font-semibold text-lg">{{ char.data.name }}</h2>
          <div class="flex items-center space-x-2">
            <button 
              @click="likeCharacter(index)" 
              class="bg-pink-500 text-white px-3 py-1 rounded hover:bg-pink-600 transition text-sm flex items-center space-x-1"
            >
              <span>❤️</span>
              <span>{{ char.data.likes || 0 }}</span>
            </button>
          </div>
        </div>
        <p>Âge : {{ char.data.age }}</p>
        <p>Affiliation : {{ char.data.affiliation }}</p>
        <p>Sabre laser : {{ char.data.lightsaber ? 'Oui' : 'Non' }}</p>
        
        <div class="mt-3 space-x-2">
          <button @click="startEdit(index)" class="bg-green-600 text-white px-3 py-1 rounded hover:bg-green-700 transition text-sm">Modifier</button>
          <button @click="deleteCharacter(index)" class="bg-red-600 text-white px-3 py-1 rounded hover:bg-red-700 transition text-sm">Supprimer</button>
        </div>

        <div class="mt-4 pl-4 border-l-2 border-gray-300">
          <h3 class="font-semibold">Messages</h3>
          
          <div class="flex items-center space-x-2 mt-2">
            <input v-model="newMessageText[index]" placeholder="Nouveau message..." class="border p-2 rounded flex-1" />
            <button @click="addMessage(index)" class="bg-blue-500 text-white px-3 py-1 rounded text-sm">Ajouter</button>
          </div>

          <div
            v-for="(msg, msgIndex) in char.data.messages"
            :key="msg.id"
            class="mt-3 p-3 bg-white border rounded"
          >
            <div v-if="editingMessageIndex?.charIndex === index && editingMessageIndex?.msgIndex === msgIndex">
              <input v-model="editingMessageText" class="border p-2 rounded w-full" />
              <div class="mt-2 space-x-2">
                <button @click="saveEditMessage" class="bg-blue-500 text-white px-2 py-1 rounded text-xs">Enregistrer</button>
                <button @click="cancelEditMessage" class="bg-gray-500 text-white px-2 py-1 rounded text-xs">Annuler</button>
              </div>
            </div>
            <div v-else>
              <p>{{ msg.text }}</p>
              <p class="text-sm text-gray-500">Posté le {{ new Date(msg.createdAt).toLocaleString() }}</p>
              <div class="mt-2 space-x-2">
                <button @click="startEditMessage(index, msgIndex)" class="bg-green-500 text-white px-2 py-1 rounded text-xs">Modifier</button>
                <button @click="deleteMessage(index, msgIndex)" class="bg-red-500 text-white px-2 py-1 rounded text-xs">Supprimer</button>
              </div>
              
              <div class="mt-3 pl-3 border-l border-gray-200">
                <p class="text-sm font-semibold">Commentaires</p>
                
                <div class="flex items-center space-x-2 mt-1">
                  <input v-model="newCommentText[`${index}-${msgIndex}`]" placeholder="Commenter..." class="border p-1 rounded flex-1 text-sm" />
                  <button @click="addComment(index, msgIndex)" class="bg-blue-400 text-white px-2 py-1 rounded text-xs">Ajouter</button>
                </div>

                <div
                  v-for="(comment, commentIndex) in (visibleComments[`${index}-${msgIndex}`] ? msg.comments : msg.comments.slice(0, 1))"
                  :key="comment.id"
                  class="mt-2 p-2 bg-gray-50 rounded text-sm"
                >
                  <div v-if="editingCommentIndex?.charIndex === index && editingCommentIndex?.msgIndex === msgIndex && editingCommentIndex?.commentIndex === commentIndex">
                    <input v-model="editingCommentText" class="border p-1 rounded w-full text-sm" />
                    <div class="mt-1 space-x-1">
                      <button @click="saveEditComment" class="bg-blue-400 text-white px-2 py-1 rounded text-xs">Enregistrer</button>
                      <button @click="cancelEditComment" class="bg-gray-400 text-white px-2 py-1 rounded text-xs">Annuler</button>
                    </div>
                  </div>
                  <div v-else>
                    <p>{{ comment.text }}</p>
                    <div class="mt-1 space-x-1">
                      <button @click="startEditComment(index, msgIndex, commentIndex)" class="bg-green-400 text-white px-2 py-1 rounded text-xs">Modifier</button>
                      <button @click="deleteComment(index, msgIndex, commentIndex)" class="bg-red-400 text-white px-2 py-1 rounded text-xs">Supprimer</button>
                    </div>
                  </div>
                </div>

                <div v-if="msg.comments.length > 1" class="mt-2">
                  <button 
                    @click="toggleCommentVisibility(index, msgIndex)"
                    class="text-blue-600 text-xs underline hover:text-blue-800"
                  >
                    {{ visibleComments[`${index}-${msgIndex}`] 
                        ? 'Masquer les commentaires' 
                        : `Voir les ${msg.comments.length - 1} autres commentaires` 
                    }}
                  </button>
                </div>

              </div>
            </div>
          </div>
        </div>
      </div>
      
      <div v-else class="space-y-2">
        <input v-model="editingCharacter!.name" placeholder="Nom" class="border p-2 rounded w-full" required />
        <input v-model.number="editingCharacter!.age" placeholder="Âge" type="number" class="border p-2 rounded w-full" required />
        <input v-model="editingCharacter!.affiliation" placeholder="Affiliation" class="border p-2 rounded w-full" required />
        <label class="flex items-center space-x-2">
          <input type="checkbox" v-model="editingCharacter!.lightsaber" />
          <span>Possède un sabre laser</span>
        </label>
        <div class="mt-3 space-x-2">
          <button @click="saveEdit(index)" class="bg-blue-600 text-white px-3 py-1 rounded hover:bg-blue-700 transition text-sm">Enregistrer</button>
          <button @click="cancelEdit" class="bg-gray-600 text-white px-3 py-1 rounded hover:bg-gray-700 transition text-sm">Annuler</button>
        </div>
      </div>
    </article>
  </section>
</template>
