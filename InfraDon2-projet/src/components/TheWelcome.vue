<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'
import PouchDBFind from 'pouchdb-find'
PouchDB.plugin(PouchDBFind)

interface Comment { id: string; text: string; createdAt: string }
interface Message { id: string; text: string; createdAt: string; likes: number; comments: Comment[] }
interface Character {
  name: string
  age: number
  affiliation: string
  lightsaber: boolean
  likes: number
  messages?: Message[]
}

// pour pouchdb
interface CharacterDoc {
  _id: string
  _rev?: string
  type: 'character'
  name: string
  age: number
  affiliation: string
  affiliationLower?: string
  lightsaber: boolean
  likes: number
}

interface MessageDoc {
  _id: string
  _rev?: string
  type: 'message'
  characterId: string
  text: string
  textLower: string
  createdAt: string
  likes: number
}

interface CommentDoc {
  _id: string
  _rev?: string
  type: 'comment'
  messageId: string
  text: string
  createdAt: string
}

const COUCH_REMOTE = 'http://RomainBlanchard:admin@localhost:5984/coachdb'

const storage = ref<any>(null)
const characters = ref<{ data: Character; docId: string; docRev: string }[]>([])
const isOnline = ref(true)
let syncHandler: any = null

const newCharacter = ref<Character>({ name: '', age: 0, affiliation: '', lightsaber: false, likes: 0, messages: [] })

const editingIndex = ref<number | null>(null)
const editingCharacter = ref<Character | null>(null)

const newMessageText = ref<Record<number, string>>({})
const editingMessageIndex = ref<{ charIndex: number; msgIndex: number } | null>(null)
const editingMessageText = ref('')

const newCommentText = ref<Record<string, string>>({})
const editingCommentIndex = ref<{ charIndex: number; msgIndex: number; commentIndex: number } | null>(null)
const editingCommentText = ref('')

const visibleComments = ref<Record<string, boolean>>({})

const searchMessageText = ref('')
// Mode global "Top 10 messages par likes"
const sortByLikes = ref(false)
const searchAffiliation = ref('')

// Tri par likes par personnage (clé: characterId)
const orderByLikesForChar = ref<Record<string, boolean>>({})

const iso = () => new Date().toISOString()

// =====================
// Gestion générique des conflits
// =====================
const saveWithConflictRetry = async <T extends { _id: string; _rev?: string }>(
  getLatestDoc: () => Promise<T>,
  mergeFn: (latest: T) => T,
  maxRetries = 3
): Promise<T | null> => {
  if (!storage.value) return null
  let attempt = 0
  let lastError: any = null
  while (attempt < maxRetries) {
    try {
      const latestDoc = await getLatestDoc()
      const mergedDoc = mergeFn(latestDoc)
      const res = await storage.value.put(mergedDoc)
      return { ...mergedDoc, _rev: res.rev }
    } catch (err: any) {
      lastError = err
      if (err?.status === 409) {
        console.warn('Conflit détecté, tentative de résolution...', { attempt: attempt + 1, id: (err as any)?.id })
        attempt++
        continue
      } else {
        console.error('Erreur non liée à un conflit :', err)
        break
      }
    }
  }
  console.error('Échec de résolution de conflit après plusieurs tentatives :', lastError)
  return null
}

const removeWithRetry = async (id: string, maxRetries = 3) => {
  if (!storage.value) return false
  let attempt = 0
  let lastError: any = null
  while (attempt < maxRetries) {
    try {
      const d = await storage.value.get(id)
      await storage.value.remove(d)
      return true
    } catch (err: any) {
      lastError = err
      if (err?.status === 409) {
        attempt++
        continue
      } else {
        console.error('Erreur suppression doc :', err)
        break
      }
    }
  }
  console.error('Échec de suppression après plusieurs tentatives :', lastError)
  return false
}

// =====================

const initDatabase = async () => {
  const localDB = new PouchDB('infradon-blaro')
  storage.value = localDB
  await createIndexes()
  await fetchData()
  startSync()
}

const createIndexes = async () => {
  if (!storage.value) return
  try {
    await storage.value.createIndex({ index: { fields: ['type'] } })
    await storage.value.createIndex({ index: { fields: ['type', 'likes'] } }) // tri global messages par likes
    await storage.value.createIndex({ index: { fields: ['type', 'affiliationLower'] } })
    await storage.value.createIndex({ index: { fields: ['type', 'characterId'] } })
    await storage.value.createIndex({ index: { fields: ['type', 'characterId', 'likes'] } }) // tri par personnage par likes
    await storage.value.createIndex({ index: { fields: ['type', 'characterId', 'createdAt'] } }) // tri par personnage par date
    await storage.value.createIndex({ index: { fields: ['type', 'textLower'] } })
    await storage.value.createIndex({ index: { fields: ['type', 'textLower', 'createdAt'] } }) // recherche + tri par date
    await storage.value.createIndex({ index: { fields: ['type', 'messageId'] } })
    console.log('Index Mango créés')
  } catch (err) {
    console.error('Erreur création index :', err)
  }
}

const startSync = () => {
  if (!storage.value || syncHandler) return
  syncHandler = storage.value
    .sync(COUCH_REMOTE, { live: true, retry: true })
    .on('change', async () => { await fetchData(sortByLikes.value) }) // conserve la mécanique
    .on('error', (err: any) => console.error('Erreur sync :', err))
}
const stopSync = () => { if (syncHandler?.cancel) { syncHandler.cancel(); syncHandler = null } }
const toggleOnline = () => { isOnline.value = !isOnline.value; isOnline.value ? startSync() : stopSync() }

const mapCharacterDoc = (d: CharacterDoc) => ({
  data: { name: d.name, age: d.age, affiliation: d.affiliation, lightsaber: !!d.lightsaber, likes: d.likes ?? 0, messages: [] },
  docId: d._id,
  docRev: d._rev || ''
})

const loadMessagesForCharacter = async (characterId: string, charIndex: number) => {
  if (!storage.value) return
  const orderByLikes = !!orderByLikesForChar.value[characterId]

  // Tri côté DB: par likes desc OU par date desc
  const selector: any = { type: 'message', characterId }
  const sort = orderByLikes
    ? [{ type: 'asc' }, { characterId: 'asc' }, { likes: 'desc' }]
    : [{ type: 'asc' }, { characterId: 'asc' }, { createdAt: 'desc' }]

  if (orderByLikes) selector.likes = { $gte: 0 }
  else selector.createdAt = { $gte: null }

  const { docs: messageDocs } = await storage.value.find({
    selector,
    sort,
    limit: 9999
  })
  const messages = messageDocs as MessageDoc[]

  const messageIds = messages.map(m => m._id)
  let commentsByMsg = new Map<string, CommentDoc[]>()
  if (messageIds.length) {
    const { docs: commentDocs } = await storage.value.find({
      selector: { type: 'comment', messageId: { $in: messageIds } },
      limit: 9999
    })
    commentsByMsg = (commentDocs as CommentDoc[]).reduce((acc, c) => {
      const arr = acc.get(c.messageId) || []
      arr.push(c)
      acc.set(c.messageId, arr)
      return acc
    }, new Map<string, CommentDoc[]>())
  }

  const uiMessages: Message[] = messages.map(m => ({
    id: m._id,
    text: m.text,
    createdAt: m.createdAt,
    likes: m.likes ?? 0,
    comments: (commentsByMsg.get(m._id) || [])
      .sort((a, b) => new Date(a.createdAt).getTime() - new Date(b.createdAt).getTime())
      .map(c => ({ id: c._id, text: c.text, createdAt: c.createdAt }))
  }))
  if (characters.value[charIndex]?.docId === characterId) {
    characters.value[charIndex].data.messages = uiMessages
  }
}
const reloadAllMessages = async () => {
  for (let i = 0; i < characters.value.length; i++) {
    await loadMessagesForCharacter(characters.value[i].docId, i)
  }
}

const fetchData = async (onlyTopMessagesByLikes = false) => {
  if (!storage.value) return
  try {
    if (onlyTopMessagesByLikes) {
      // Top 10 messages par likes (tri côté DB)
      const { docs: msgDocs } = await storage.value.find({
        selector: { type: 'message', likes: { $gte: 0 } },
        sort: [{ likes: 'desc' }],
        limit: 10
      })
      const messages = msgDocs as MessageDoc[]
      if (!messages.length) { characters.value = []; return }

      const characterIds = Array.from(new Set(messages.map(m => m.characterId)))
      const { docs: charDocs } = await storage.value.find({
        selector: { _id: { $in: characterIds } },
        limit: characterIds.length
      })
      const charMap = new Map<string, number>()
      characters.value = (charDocs as CharacterDoc[])
        .filter(d => (d as any).type === 'character')
        .map((d, idx) => {
          charMap.set(d._id, idx)
          return mapCharacterDoc(d)
        })

      const msgIds = messages.map(m => m._id)
      const { docs: cmtDocs } = await storage.value.find({
        selector: { type: 'comment', messageId: { $in: msgIds } },
        limit: 9999
      })
      const commentsByMsg = (cmtDocs as CommentDoc[]).reduce((acc, c) => {
        const arr = acc.get(c.messageId) || []
        arr.push(c)
        acc.set(c.messageId, arr)
        return acc
      }, new Map<string, CommentDoc[]>())

      for (const m of messages) {
        const idx = charMap.get(m.characterId)
        if (idx === undefined) continue
        const msgUI: Message = {
          id: m._id,
          text: m.text,
          createdAt: m.createdAt,
          likes: m.likes ?? 0,
          comments: (commentsByMsg.get(m._id) || [])
            .sort((a, b) => new Date(a.createdAt).getTime() - new Date(b.createdAt).getTime())
            .map(c => ({ id: c._id, text: c.text, createdAt: c.createdAt }))
        }
        const arr = characters.value[idx].data.messages || []
        arr.push(msgUI)
        characters.value[idx].data.messages = arr
      }
    } else {
      // Liste complète des personnages + messages
      const { docs } = await storage.value.find({
        selector: { type: 'character' },
        limit: 9999
      })
      characters.value = (docs as CharacterDoc[]).map(mapCharacterDoc)
      await reloadAllMessages()
    }
  } catch (err) {
    console.error('Erreur fetch data :', err)
  }
}

// CRUD
const addCharacter = async () => {
  if (!storage.value) return
  try {
    const doc: CharacterDoc = {
      _id: 'character:' + iso(),
      type: 'character',
      name: newCharacter.value.name,
      age: newCharacter.value.age,
      affiliation: newCharacter.value.affiliation,
      affiliationLower: (newCharacter.value.affiliation || '').toLowerCase(),
      lightsaber: newCharacter.value.lightsaber,
      likes: newCharacter.value.likes || 0
    }
    const res = await storage.value.put(doc)
    characters.value.push(mapCharacterDoc({ ...doc, _rev: res.rev }))
    newCharacter.value = { name: '', age: 0, affiliation: '', lightsaber: false, likes: 0, messages: [] }
  } catch (err) {
    console.error("Erreur ajout personnage :", err)
  }
}

const startEdit = (index: number) => { editingIndex.value = index; editingCharacter.value = { ...characters.value[index].data } }
const cancelEdit = () => { editingIndex.value = null; editingCharacter.value = null }

const saveEdit = async (index: number) => {
  if (!storage.value || !editingCharacter.value) return
  try {
    const curr = characters.value[index]
    const updated = await saveWithConflictRetry<CharacterDoc>(
      () => storage.value.get(curr.docId),
      latest => {
        latest.name = editingCharacter.value!.name
        latest.age = editingCharacter.value!.age
        latest.affiliation = editingCharacter.value!.affiliation
        latest.affiliationLower = (editingCharacter.value!.affiliation || '').toLowerCase()
        latest.lightsaber = editingCharacter.value!.lightsaber
        if (typeof editingCharacter.value!.likes === 'number') {
          // éviter d'écraser une éventuelle mise à jour distante des likes
          latest.likes = Math.max(latest.likes ?? 0, editingCharacter.value!.likes ?? 0)
        }
        return latest
      }
    )
    if (updated) {
      characters.value[index] = mapCharacterDoc(updated)
      await loadMessagesForCharacter(curr.docId, index)
    }
    cancelEdit()
  } catch (err) {
    console.error('Erreur modification personnage :', err)
  }
}

const deleteCharacter = async (index: number) => {
  if (!storage.value) return
  if (!confirm('Supprimer ce personnage (et ses messages/commentaires) ?')) return
  try {
    const curr = characters.value[index]
    const characterId = curr.docId
    const { docs: msgDocs } = await storage.value.find({ selector: { type: 'message', characterId }, limit: 9999 })
    const msgs = msgDocs as MessageDoc[]
    if (msgs.length) {
      const msgIds = msgs.map(m => m._id)
      const { docs: cmtDocs } = await storage.value.find({ selector: { type: 'comment', messageId: { $in: msgIds } }, limit: 9999 })
      const cmts = cmtDocs as CommentDoc[]
      if (cmts.length) await storage.value.bulkDocs(cmts.map(c => ({ ...c, _deleted: true })))
      await storage.value.bulkDocs(msgs.map(m => ({ ...m, _deleted: true })))
    }
    await removeWithRetry(characterId)
    characters.value.splice(index, 1)
  } catch (err) {
    console.error('Erreur suppression personnage :', err)
  }
}

const addMessage = async (charIndex: number) => {
  if (!storage.value) return
  const text = (newMessageText.value[charIndex] || '').trim()
  if (!text) return
  try {
    const char = characters.value[charIndex]
    const msg: MessageDoc = {
      _id: 'message:' + iso(),
      type: 'message',
      characterId: char.docId,
      text,
      textLower: text.toLowerCase(),
      createdAt: iso(),
      likes: 0
    }
    await storage.value.put(msg)
    await loadMessagesForCharacter(char.docId, charIndex)
    newMessageText.value[charIndex] = ''
  } catch (err) {
    console.error('Erreur ajout message :', err)
  }
}

const startEditMessage = (charIndex: number, msgIndex: number) => {
  editingMessageIndex.value = { charIndex, msgIndex }
  editingMessageText.value = characters.value[charIndex].data.messages![msgIndex].text
}
const cancelEditMessage = () => { editingMessageIndex.value = null; editingMessageText.value = '' }

const saveEditMessage = async () => {
  if (!storage.value || !editingMessageIndex.value) return
  try {
    const { charIndex, msgIndex } = editingMessageIndex.value
    const msg = characters.value[charIndex].data.messages![msgIndex]
    const updated = await saveWithConflictRetry<MessageDoc>(
      () => storage.value.get(msg.id),
      latest => {
        latest.text = editingMessageText.value
        latest.textLower = editingMessageText.value.toLowerCase()
        return latest
      }
    )
    if (updated) {
      await loadMessagesForCharacter(characters.value[charIndex].docId, charIndex)
    }
    cancelEditMessage()
  } catch (err) {
    console.error('Erreur modification message :', err)
  }
}

const deleteMessage = async (charIndex: number, msgIndex: number) => {
  if (!storage.value) return
  if (!confirm('Supprimer ce message (et ses commentaires) ?')) return
  try {
    const char = characters.value[charIndex]
    const msg = char.data.messages![msgIndex]
    const { docs: cmtDocs } = await storage.value.find({ selector: { type: 'comment', messageId: msg.id }, limit: 9999 })
    const cmts = cmtDocs as CommentDoc[]
    if (cmts.length) await storage.value.bulkDocs(cmts.map(c => ({ ...c, _deleted: true })))
    await removeWithRetry(msg.id)
    await loadMessagesForCharacter(char.docId, charIndex)
  } catch (err) {
    console.error('Erreur suppression message :', err)
  }
}

const likeCharacter = async (charIndex: number) => {
  if (!storage.value) return
  try {
    const curr = characters.value[charIndex]
    const updated = await saveWithConflictRetry<CharacterDoc>(
      () => storage.value.get(curr.docId),
      latest => {
        latest.likes = (latest.likes ?? 0) + 1
        return latest
      }
    )
    if (updated) {
      characters.value[charIndex] = mapCharacterDoc(updated)
      if (sortByLikes.value) setTimeout(() => fetchData(true), 120)
      else await loadMessagesForCharacter(curr.docId, charIndex)
    }
  } catch (err) {
    console.error('Erreur like personnage :', err)
  }
}

// Like d'un message (avec gestion des conflits)
const likeMessage = async (charIndex: number, msgIndex: number) => {
  if (!storage.value) return
  try {
    const msg = characters.value[charIndex].data.messages![msgIndex]
    const updated = await saveWithConflictRetry<MessageDoc>(
      () => storage.value.get(msg.id),
      latest => {
        latest.likes = (latest.likes ?? 0) + 1
        return latest
      }
    )
    if (updated) {
      if (sortByLikes.value) {
        await fetchData(true) // en mode Top 10 global
      } else {
        await loadMessagesForCharacter(characters.value[charIndex].docId, charIndex)
      }
    }
  } catch (err) {
    console.error('Erreur like message :', err)
  }
}

const addComment = async (charIndex: number, msgIndex: number) => {
  if (!storage.value) return
  const key = `${charIndex}-${msgIndex}`
  const text = (newCommentText.value[key] || '').trim()
  if (!text) return
  try {
    const char = characters.value[charIndex]
    const msg = char.data.messages![msgIndex]
    const c: CommentDoc = {
      _id: 'comment:' + iso(),
      type: 'comment',
      messageId: msg.id,
      text,
      createdAt: iso()
    }
    await storage.value.put(c)
    await loadMessagesForCharacter(char.docId, charIndex)
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
const cancelEditComment = () => { editingCommentIndex.value = null; editingCommentText.value = '' }

const saveEditComment = async () => {
  if (!storage.value || !editingCommentIndex.value) return
  try {
    const { charIndex, msgIndex, commentIndex } = editingCommentIndex.value
    const char = characters.value[charIndex]
    const cmt = char.data.messages![msgIndex].comments[commentIndex]
    const updated = await saveWithConflictRetry<CommentDoc>(
      () => storage.value.get(cmt.id),
      latest => {
        latest.text = editingCommentText.value
        return latest
      }
    )
    if (updated) {
      await loadMessagesForCharacter(char.docId, charIndex)
    }
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
    const cmt = char.data.messages![msgIndex].comments[commentIndex]
    await removeWithRetry(cmt.id)
    await loadMessagesForCharacter(char.docId, charIndex)
  } catch (err) {
    console.error('Erreur suppression commentaire :', err)
  }
}

const toggleCommentVisibility = (charIndex: number, msgIndex: number) => {
  const key = `${charIndex}-${msgIndex}`
  visibleComments.value[key] = !visibleComments.value[key]
}

const toggleCharSort = async (charIndex: number) => {
  const char = characters.value[charIndex]
  const id = char.docId
  orderByLikesForChar.value[id] = !orderByLikesForChar.value[id]
  await loadMessagesForCharacter(id, charIndex)
}

const searchMessages = async () => {
  if (!storage.value) return
  const q = (searchMessageText.value || '').trim().toLowerCase()
  if (!q) { sortByLikes.value = false; await fetchData(); return }
  try {
    const upper = q + '\uffff'
    // Recherche + tri par date côté DB
    const { docs: msgDocs } = await storage.value.find({
      selector: { type: 'message', textLower: { $gte: q, $lte: upper }, createdAt: { $gte: null } },
      sort: [{ type: 'asc' }, { textLower: 'asc' }, { createdAt: 'desc' }],
      limit: 500
    })
    const messages = msgDocs as MessageDoc[]
    if (!messages.length) { characters.value = []; return }

    const characterIds = Array.from(new Set(messages.map(m => m.characterId)))
    const { docs: charDocs } = await storage.value.find({
      selector: { _id: { $in: characterIds } },
      limit: characterIds.length
    })
    const charRows = (charDocs as CharacterDoc[]).filter((d: any) => d?.type === 'character')
    const charMap = new Map<string, number>()
    characters.value = charRows.map((d: CharacterDoc, idx: number) => {
      charMap.set(d._id, idx)
      return mapCharacterDoc(d)
    })

    const msgIds = messages.map(m => m._id)
    const { docs: cmtDocs } = await storage.value.find({
      selector: { type: 'comment', messageId: { $in: msgIds } },
      limit: 9999
    })
    const commentsByMsg = (cmtDocs as CommentDoc[]).reduce((acc, c) => {
      const arr = acc.get(c.messageId) || []
      arr.push(c)
      acc.set(c.messageId, arr)
      return acc
    }, new Map<string, CommentDoc[]>())

    for (const m of messages) {
      const idx = charMap.get(m.characterId)
      if (idx === undefined) continue
      const msgUI: Message = {
        id: m._id,
        text: m.text,
        createdAt: m.createdAt,
        likes: m.likes ?? 0,
        comments: (commentsByMsg.get(m._id) || [])
          .sort((a, b) => new Date(a.createdAt).getTime() - new Date(b.createdAt).getTime())
          .map(c => ({ id: c._id, text: c.text, createdAt: c.createdAt }))
      }
      const arr = characters.value[idx].data.messages || []
      arr.push(msgUI)
      // Pas de tri en TS: ordre déjà géré par Mango
      characters.value[idx].data.messages = arr
    }
  } catch (err) {
    console.error('Erreur recherche messages :', err)
  }
}

const searchByAffiliation = async () => {
  if (!storage.value) return
  const val = (searchAffiliation.value || '').trim().toLowerCase()
  if (!val) { sortByLikes.value = false; await fetchData(); return }
  try {
    const { docs } = await storage.value.find({
      selector: { type: 'character', affiliationLower: val },
      limit: 9999
    })
    characters.value = (docs as CharacterDoc[]).map(mapCharacterDoc)
    await reloadAllMessages()
  } catch (err) {
    console.error('Erreur recherche affiliation :', err)
  }
}

const toggleSortByLikes = async () => {
  if (sortByLikes.value) { sortByLikes.value = false; await fetchData() }
  else { sortByLikes.value = true; await fetchData(true) }
}

onMounted(() => { initDatabase() })
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
      {{ sortByLikes ? 'Top 10 Messages par Likes (Cliquez pour désactiver)' : 'Afficher le Top 10 Messages par Likes' }}
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
          <div class="flex items-center justify-between">
            <h3 class="font-semibold">Messages</h3>
            <button
              @click="toggleCharSort(index)"
              class="text-xs px-2 py-1 rounded border"
              :class="orderByLikesForChar[char.docId] ? 'bg-yellow-50 border-yellow-300 text-yellow-700' : 'bg-gray-50 border-gray-300 text-gray-700'"
              :title="orderByLikesForChar[char.docId] ? 'Trier par Date (desc)' : 'Trier par Likes (desc)'"
            >
              {{ orderByLikesForChar[char.docId] ? 'Tri: Likes ↓' : 'Tri: Date ↓' }}
            </button>
          </div>
          
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
              <div class="flex items-center justify-between">
                <div>
                  <p>{{ msg.text }}</p>
                  <p class="text-sm text-gray-500">Posté le {{ new Date(msg.createdAt).toLocaleString() }}</p>
                </div>
                <button 
                  @click="likeMessage(index, msgIndex)" 
                  class="bg-pink-500 text-white px-2 py-1 rounded text-xs flex items-center space-x-1"
                >
                  <span>❤️</span>
                  <span>{{ msg.likes || 0 }}</span>
                </button>
              </div>

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