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
  likes: number
  comments: Comment[]
  createdAt: string
}

interface Character {
  name: string
  age: number
  affiliation: string
  lightsaber: boolean
  messages?: Message[]
}

interface CharacterDoc {
  _id: string
  _rev?: string
  characters: Character[]
}


const storage = ref<any>(null)
const characters = ref<{ data: Character; docId: string; docRev: string }[]>([]);
const isOnline = ref(true)
let syncHandler: any = null



const newCharacter = ref<Character>({
  name: '',
  age: 0,
  affiliation: '',
  lightsaber: false,
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


const searchMessageText = ref('')

const sortByLikes = ref(false)


const initDatabase = () => {
  console.log('=> Connexion à la base de données')
  const localDB = new PouchDB('infradon-blaro')
  storage.value = localDB
  console.log('Base locale :', localDB?.name);

  fetchData()
  createIndex()
}

const startSync = () => {
  if (!storage.value) return
  if (syncHandler) return

  console.log("Synchronisation ACTIVÉE")
  syncHandler = storage.value.sync('http://RomainBlanchard:admin@localhost:5984/infradon-blaro', {
    live: true,
    retry: true
  })
  .on('change', () => {
    console.log("Données synchronisées")
    fetchData()
  })
  .on('error', (err: any) => {
    console.error("Erreur sync :", err)
  })
}

const stopSync = () => {
  if (syncHandler && syncHandler.cancel) {
    console.log("Synchronisation STOPPÉE")
    syncHandler.cancel()
    syncHandler = null
  }
}

const toggleOnline = () => {
  isOnline.value = !isOnline.value
  console.log("Mode en ligne ?", isOnline.value)

  if (isOnline.value) {
    startSync()
  } else {
    stopSync()
  }
}
const createIndex = async () => {
  if (!storage.value) return
  try {
    await storage.value.createIndex({
      index: { fields: ['characters.affiliation'] }
    })
    await storage.value.createIndex({
      index: { fields: ['characters.messages.text'] }
    })
    await storage.value.createIndex({
      index: { fields: ['characters.messages.likes'] }
    })
    console.log('Index créés')
  } catch (err) {
    console.error('Erreur création index :', err)
  }
}


const fetchData = async () => {
  if (!storage.value) return

  try {
    const result = await storage.value.allDocs({ include_docs: true })
    console.log('=> Données récupérées :', result)

    const allCharacters: { data: Character; docId: string; docRev: string }[] = []
    
    result.rows.forEach((row: any) => {
      if (row.doc.characters && Array.isArray(row.doc.characters)) {
        row.doc.characters.forEach((char: Character) => {
          if (!char.messages) char.messages = []
          allCharacters.push({
            data: char,
            docId: row.doc._id,
            docRev: row.doc._rev
          })
        })
      }
    })
    
    characters.value = allCharacters
  } catch (err) {
    console.error('Erreur lors de la récupération :', err)
  }
}

const addCharacter = async () => {
  if (!storage.value) return
  try {

    const newDoc = {
      _id: new Date().toISOString(),
      characters: [{ ...newCharacter.value, messages: [] }]
    }

    const response = await storage.value.put(newDoc)
    console.log('Personnage ajouté avec succès')

    characters.value.push({
      data: { ...newCharacter.value, messages: [] },
      docId: newDoc._id,
      docRev: response.rev
    })

    newCharacter.value = {
      name: '',
      age: 0,
      affiliation: '',
      lightsaber: false,
      messages: []
    }
  } catch (err) {
    console.error('Erreur lors de l\'ajout :', err)
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
    
    doc.characters[0] = editingCharacter.value
    

    const response = await storage.value.put(doc)
    console.log('Personnage modifié avec succès')
    

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
      likes: 0,
      comments: [],
      createdAt: new Date().toISOString()
    }

    if (!doc.characters[0].messages) doc.characters[0].messages = []
    doc.characters[0].messages.push(newMessage)

    const response = await storage.value.put(doc)
    
    characters.value[charIndex].data.messages = doc.characters[0].messages
    characters.value[charIndex].docRev = response.rev
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

    doc.characters[0].messages![msgIndex].text = editingMessageText.value

    const response = await storage.value.put(doc)
    
    characters.value[charIndex].data.messages = doc.characters[0].messages
    characters.value[charIndex].docRev = response.rev
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

    doc.characters[0].messages!.splice(msgIndex, 1)

    const response = await storage.value.put(doc)
    
    characters.value[charIndex].data.messages = doc.characters[0].messages
    characters.value[charIndex].docRev = response.rev
  } catch (err) {
    console.error('Erreur suppression message :', err)
  }
}

const likeMessage = async (charIndex: number, msgIndex: number) => {
  if (!storage.value) return

  try {
    const char = characters.value[charIndex]
    const doc: CharacterDoc = await storage.value.get(char.docId)

    doc.characters[0].messages![msgIndex].likes++

    const response = await storage.value.put(doc)
    
    characters.value[charIndex].data.messages = doc.characters[0].messages
    characters.value[charIndex].docRev = response.rev
  } catch (err) {
    console.error('Erreur like message :', err)
  }
}

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

    doc.characters[0].messages![msgIndex].comments.push(newComment)

    const response = await storage.value.put(doc)
    
    characters.value[charIndex].data.messages = doc.characters[0].messages
    characters.value[charIndex].docRev = response.rev
    newCommentText.value[key] = ''
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

    doc.characters[0].messages![msgIndex].comments[commentIndex].text = editingCommentText.value

    const response = await storage.value.put(doc)
    
    characters.value[charIndex].data.messages = doc.characters[0].messages
    characters.value[charIndex].docRev = response.rev
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

    doc.characters[0].messages![msgIndex].comments.splice(commentIndex, 1)

    const response = await storage.value.put(doc)
    
    characters.value[charIndex].data.messages = doc.characters[0].messages
    characters.value[charIndex].docRev = response.rev
  } catch (err) {
    console.error('Erreur suppression commentaire :', err)
  }
}


const searchMessages = async () => {
  if (!storage.value) return

  if (!searchMessageText.value) {
    fetchData()
    return
  }

  try {
    const result = await storage.value.find({
      selector: {
        'characters.messages': {
          $elemMatch: {
            text: { $regex: searchMessageText.value }
          }
        }
      }
    })

    const allCharacters: { data: Character; docId: string; docRev: string }[] = []
    result.docs.forEach((doc: any) => {
      if (doc.characters && Array.isArray(doc.characters)) {
        doc.characters.forEach((char: Character) => {
          if (!char.messages) char.messages = []
          allCharacters.push({
            data: char,
            docId: doc._id,
            docRev: doc._rev
          })
        })
      }
    })
    characters.value = allCharacters
  } catch (err) {
    console.error('Erreur recherche messages :', err)
  }
}

const toggleSortByLikes = async () => {
  sortByLikes.value = !sortByLikes.value
  
  if (!storage.value) return

  try {
    // Création d'une vue pour trier par likes
    const designDoc = {
      _id: '_design/messages',
      views: {
        by_likes: {
          map: function (doc: any) {
            if (doc.characters) {
              doc.characters.forEach(function (char: any) {
                if (char.messages) {
                  char.messages.forEach(function (msg: any) {
                    // @ts-ignore
                    emit(msg.likes, { char: char, docId: doc._id, docRev: doc._rev })
                  })
                }
              })
            }
          }.toString()
        }
      }
    }

    // Essayer de créer la vue (ignorer si elle existe déjà)
    try {
      await storage.value.put(designDoc)
    } catch (e: any) {
      if (e.status !== 409) throw e // 409 = document existe déjà
    }

    if (sortByLikes.value) {
      const result = await storage.value.query('messages/by_likes', {
        descending: true,
        include_docs: true
      })

      const allCharacters: { data: Character; docId: string; docRev: string }[] = []
      const seenDocs = new Set()

      result.rows.forEach((row: any) => {
        if (row.doc && !seenDocs.has(row.doc._id)) {
          seenDocs.add(row.doc._id)
          if (row.doc.characters && Array.isArray(row.doc.characters)) {
            row.doc.characters.forEach((char: Character) => {
              if (!char.messages) char.messages = []

              char.messages.sort((a, b) => b.likes - a.likes)
              allCharacters.push({
                data: char,
                docId: row.doc._id,
                docRev: row.doc._rev
              })
            })
          }
        }
      })
      characters.value = allCharacters
    } else {
      fetchData()
    }
  } catch (err) {
    console.error('Erreur tri par likes :', err)
    fetchData()
  }
}


//
const searchAffiliation = ref('')

const searchByAffiliation = async () => {
  if (!storage.value) return

  if (!searchAffiliation.value) {
    fetchData()
    return
  }

  try {
    const result = await storage.value.find({
      selector: {
        'characters': {
          $elemMatch: {
            affiliation: { $eq: searchAffiliation.value }
          }
        }
      }
    })

    const allCharacters: { data: Character; docId: string; docRev: string }[] = []
    result.docs.forEach((doc: any) => {
      if (doc.characters && Array.isArray(doc.characters)) {
        doc.characters.forEach((char: Character) => {
          if (char.affiliation.toLowerCase() === searchAffiliation.value.toLowerCase()) {
            if (!char.messages) char.messages = []
            allCharacters.push({
              data: char,
              docId: doc._id,
              docRev: doc._rev
            })
          }
        })
      }
    })
    characters.value = allCharacters
  } catch (err) {
    console.error('Erreur recherche affiliation :', err)
  }
}


onMounted(() => {
  initDatabase()
  createIndex()
  fetchData()
  startSync()
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

    <input
      v-model="newCharacter.name"
      placeholder="Nom"
      class="border p-2 rounded w-full"
      required
    />

    <input
      v-model.number="newCharacter.age"
      placeholder="Âge"
      type="number"
      class="border p-2 rounded w-full"
      required
    />

    <input
      v-model="newCharacter.affiliation"
      placeholder="Affiliation"
      class="border p-2 rounded w-full"
      required
    />

    <label class="flex items-center space-x-2">
      <input type="checkbox" v-model="newCharacter.lightsaber" />
      <span>Possède un sabre laser</span>
    </label>

    <button
      type="submit"
      class="bg-blue-600 text-white px-4 py-2 rounded hover:bg-blue-700 transition"
    >
      Ajouter
    </button>
  </form>
<div class="p-4 mb-6 border rounded bg-gray-50 space-y-2">
  <h2 class="font-semibold text-lg mb-2">Recherche par affiliation</h2>
  <div class="flex items-center space-x-2">
    <input
      v-model="searchAffiliation"
      placeholder="Ex: Jedi"
      class="border p-2 rounded w-full"
    />
    <button
      @click="searchByAffiliation"
      class="bg-purple-600 text-white px-4 py-2 rounded hover:bg-purple-700 transition"
    >
      Rechercher
    </button>
  </div>
</div>

<div class="p-4 mb-6 border rounded bg-gray-50 space-y-2">
  <h2 class="font-semibold text-lg mb-2">Recherche de messages</h2>
  <div class="flex items-center space-x-2">
    <input
      v-model="searchMessageText"
      placeholder="Rechercher un message..."
      class="border p-2 rounded w-full"
    />
    <button
      @click="searchMessages"
      class="bg-indigo-600 text-white px-4 py-2 rounded hover:bg-indigo-700 transition"
    >
      Rechercher
    </button>
  </div>
  <button
    @click="toggleSortByLikes"
    class="mt-2 px-4 py-2 rounded text-white"
    :class="sortByLikes ? 'bg-yellow-600' : 'bg-gray-500'"
  >
    {{ sortByLikes ? 'Tri par likes ✔' : 'Trier par likes' }}
  </button>
</div>

  <section>
    <article
      v-for="(char, index) in characters"
      :key="char.docId"
      class="p-4 border-b hover:bg-gray-100 transition"
    >

      <div v-if="editingIndex !== index">
        <h2 class="font-semibold text-lg">{{ char.data.name }}</h2>
        <p>Âge : {{ char.data.age }}</p>
        <p>Affiliation : {{ char.data.affiliation }}</p>
        <p>Sabre laser : {{ char.data.lightsaber ? 'Oui' : 'Non' }}</p>
        
        <div class="mt-3 space-x-2">
          <button
            @click="startEdit(index)"
            class="bg-green-600 text-white px-3 py-1 rounded hover:bg-green-700 transition text-sm"
          >
            Modifier
          </button>
          <button
            @click="deleteCharacter(index)"
            class="bg-red-600 text-white px-3 py-1 rounded hover:bg-red-700 transition text-sm"
          >
            Supprimer
          </button>
        </div>

 
        <div class="mt-4 pl-4 border-l-2 border-gray-300">
          <h3 class="font-semibold">Messages</h3>
          
          <div class="flex items-center space-x-2 mt-2">
            <input
              v-model="newMessageText[index]"
              placeholder="Nouveau message..."
              class="border p-2 rounded flex-1"
            />
            <button
              @click="addMessage(index)"
              class="bg-blue-500 text-white px-3 py-1 rounded text-sm"
            >
              Ajouter
            </button>
          </div>

          <div
            v-for="(msg, msgIndex) in char.data.messages"
            :key="msg.id"
            class="mt-3 p-3 bg-white border rounded"
          >
            <div v-if="editingMessageIndex?.charIndex === index && editingMessageIndex?.msgIndex === msgIndex">
              <input
                v-model="editingMessageText"
                class="border p-2 rounded w-full"
              />
              <div class="mt-2 space-x-2">
                <button @click="saveEditMessage" class="bg-blue-500 text-white px-2 py-1 rounded text-xs">Enregistrer</button>
                <button @click="cancelEditMessage" class="bg-gray-500 text-white px-2 py-1 rounded text-xs">Annuler</button>
              </div>
            </div>
            <div v-else>
              <p>{{ msg.text }}</p>
              <p class="text-sm text-gray-500"> {{ msg.likes }} likes</p>
              <div class="mt-2 space-x-2">
                <button @click="likeMessage(index, msgIndex)" class="bg-pink-500 text-white px-2 py-1 rounded text-xs">❤️ Like</button>
                <button @click="startEditMessage(index, msgIndex)" class="bg-green-500 text-white px-2 py-1 rounded text-xs">Modifier</button>
                <button @click="deleteMessage(index, msgIndex)" class="bg-red-500 text-white px-2 py-1 rounded text-xs">Supprimer</button>
              </div>
              <div class="mt-3 pl-3 border-l border-gray-200">
                <p class="text-sm font-semibold">Commentaires</p>
                
                <div class="flex items-center space-x-2 mt-1">
                  <input
                    v-model="newCommentText[`${index}-${msgIndex}`]"
                    placeholder="Commenter..."
                    class="border p-1 rounded flex-1 text-sm"
                  />
                  <button
                    @click="addComment(index, msgIndex)"
                    class="bg-blue-400 text-white px-2 py-1 rounded text-xs"
                  >
                    Ajouter
                  </button>
                </div>

                <div
                  v-for="(comment, commentIndex) in msg.comments"
                  :key="comment.id"
                  class="mt-2 p-2 bg-gray-50 rounded text-sm"
                >
                  <div v-if="editingCommentIndex?.charIndex === index && editingCommentIndex?.msgIndex === msgIndex && editingCommentIndex?.commentIndex === commentIndex">
                    <input
                      v-model="editingCommentText"
                      class="border p-1 rounded w-full text-sm"
                    />
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
              </div>
            </div>
          </div>
        </div>
      </div>
      <div v-else class="space-y-2">
        <input
          v-model="editingCharacter!.name"
          placeholder="Nom"
          class="border p-2 rounded w-full"
          required
        />

        <input
          v-model.number="editingCharacter!.age"
          placeholder="Âge"
          type="number"
          class="border p-2 rounded w-full"
          required
        />

        <input
          v-model="editingCharacter!.affiliation"
          placeholder="Affiliation"
          class="border p-2 rounded w-full"
          required
        />

        <label class="flex items-center space-x-2">
          <input type="checkbox" v-model="editingCharacter!.lightsaber" />
          <span>Possède un sabre laser</span>
        </label>

        <div class="mt-3 space-x-2">
          <button
            @click="saveEdit(index)"
            class="bg-blue-600 text-white px-3 py-1 rounded hover:bg-blue-700 transition text-sm"
          >
            Enregistrer
          </button>
          <button
            @click="cancelEdit"
            class="bg-gray-600 text-white px-3 py-1 rounded hover:bg-gray-700 transition text-sm"
          >
            Annuler
          </button>
        </div>
      </div>
    </article>
  </section>
</template>