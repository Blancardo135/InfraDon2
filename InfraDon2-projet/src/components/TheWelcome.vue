<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'
import PouchDBFind from 'pouchdb-find'
PouchDB.plugin(PouchDBFind)

interface Comment {
  id: string;
  text: string;
  createdAt: string;
}

interface Message {
  id: string;
  text: string;
  createdAt: string;
  likes: number;
  comments: Comment[];
}

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
  mediaContentType?: string
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
  mediaContentType?: string
}

interface CommentDoc {
  _id: string
  _rev?: string
  type: 'comment'
  messageId: string
  text: string
  createdAt: string
}
// Choix d'architecture : 2 bases distantes (characters + messages) plutôt qu'une seule.
// Objectif performance : limiter la taille des réplications et éviter que les messages
// (potentiellement très nombreux et avec attachments) ne ralentissent les lectures/requêtes
// sur les personnages (CRUD plus léger).
const COUCH_REMOTE_CHARACTERS = 'http://RomainBlanchard:admin@localhost:5984/coachdb-characters'
const COUCH_REMOTE_MESSAGES   = 'http://RomainBlanchard:admin@localhost:5984/coachdb-messages'


const storageCharacters = ref<PouchDB.Database | null>(null)
const storageMessages   = ref<PouchDB.Database | null>(null)

const remoteCharacters = ref<PouchDB.Database | null>(null)
const remoteMessages   = ref<PouchDB.Database | null>(null)

const characters = ref<{ data: Character; docId: string; docRev: string }[]>([])
const isOnline = ref(true)

let syncChars: any = null
let syncMsgs: any = null

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

const newMessageText = ref<Record<number, string>>({})
const editingMessageIndex = ref<{ charIndex: number; msgIndex: number } | null>(null)
const editingMessageText = ref('')

const newCommentText = ref<Record<string, string>>({})
const editingCommentIndex = ref<{ charIndex: number; msgIndex: number; commentIndex: number } | null>(null)
const editingCommentText = ref('')

const visibleComments = ref<Record<string, boolean>>({})
const searchMessageText = ref('')
const sortByLikes = ref(false)
const searchAffiliation = ref('')
const orderByLikesForChar = ref<Record<string, boolean>>({})
const topMessagesPage = ref(0)
const TOP_PAGE_SIZE = 10


const messageMediaFile = ref<Record<string, File | null>>({})
const messageMediaUrl = ref<Record<string, string | null>>({})

const characterMediaFile = ref<Record<string, File | null>>({})
const characterMediaUrl  = ref<Record<string, string | null>>({})

const iso = () => new Date().toISOString()

const LIMIT_CHARACTERS = 5000
const LIMIT_MESSAGES_PER_CHARACTER = 5000
const LIMIT_COMMENTS = 5000


//gestion conflit avec retry+merge, je récupère la version la plus récente, car les modif peuvent être faites depuis deux clients.
const saveWithConflictRetry = async <
  T extends { _id: string; _rev?: string }
>(
  db: PouchDB.Database | null,
  getLatestDoc: () => Promise<T>,
  mergeFn: (latest: T) => T,
  maxRetries = 3
): Promise<T | null> => {
  if (!db) return null

  let attempt = 0
  let lastError: any = null

  while (attempt < maxRetries) {
    try {
      const latestDoc = await getLatestDoc()
      const mergedDoc = mergeFn(latestDoc)
      const res = await db.put(mergedDoc)
      return { ...mergedDoc, _rev: res.rev }
    } catch (err: any) {
      lastError = err
      if (err?.status === 409) {
        console.warn('Conflit détecté, tentative de résolution...', {
          attempt: attempt + 1,
          id: (err as any)?.id
        })
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

const removeWithRetry = async (
  db: PouchDB.Database | null,
  id: string,
  maxRetries = 3
) => {
  if (!db) return false

  let attempt = 0
  let lastError: any = null

  while (attempt < maxRetries) {
    try {
      const d = await db.get(id)
      await db.remove(d)
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

const generateFakeCharacter = (): Character => {
  const names = ['Luke', 'Leia', 'Han', 'Rey', 'Finn', 'Obi-Wan', 'Anakin', 'Padmé', 'Mando', 'Ahsoka']
  const affiliations = ['Jedi', 'Sith', 'Rebelles', 'Empire', 'Mandalorien', 'Résistance']
  const name = names[Math.floor(Math.random() * names.length)]
  const affiliation = affiliations[Math.floor(Math.random() * affiliations.length)]
  const age = 18 + Math.floor(Math.random() * 50)
  const lightsaber = affiliation === 'Jedi' || affiliation === 'Sith'
  return {
    name: `${name} ${Math.floor(Math.random() * 1000)}`,
    age,
    affiliation,
    lightsaber,
    likes: 0,
    messages: []
  }
}

const addFakeCharacter = async () => {
  newCharacter.value = generateFakeCharacter()
  await addCharacter()
}

// je crée les index avant de faire des requêtes Mango (find),
// sinon PouchDB devra scanner davantage de documents.
const initDatabase = async () => {
 
  const localChars = new PouchDB('infradon-blaro-characters')
  storageCharacters.value = localChars
  remoteCharacters.value = new PouchDB(COUCH_REMOTE_CHARACTERS)

  
  const localMsgs = new PouchDB('infradon-blaro-messages')
  storageMessages.value = localMsgs
  remoteMessages.value = new PouchDB(COUCH_REMOTE_MESSAGES)

  console.log(
    'initDatabase:',
    'localChars=infradon-blaro-characters, remoteChars=',
    COUCH_REMOTE_CHARACTERS,
    'localMsgs=infradon-blaro-messages, remoteMsgs=',
    COUCH_REMOTE_MESSAGES
  )
//createIndex me permet de préparer les index nécessaires pour les requêtes mango.
//toutes les recherches doivent etre executées côté DB.
  await createIndexesCharacters()
  await createIndexesMessages()
  await fetchData()
  startSync()
}

const createIndexesCharacters = async () => {
  const db = storageCharacters.value
  if (!db) return
  try {
    await db.createIndex({ index: { fields: ['type'] } })
    await db.createIndex({ index: { fields: ['type', 'affiliationLower'] } })
    await db.createIndex({ index: { fields: ['type', 'messageId'] } })
    await db.createIndex({ index: { fields: ['type', 'messageId', 'createdAt'] } })
    console.log('Index Mango créés (characters/comments)')
  } catch (err) {
    console.error('Erreur création index characters/comments :', err)
  }
}

const createIndexesMessages = async () => {
  const db = storageMessages.value
  if (!db) return
  try {
    await db.createIndex({ index: { fields: ['type'] } })
    await db.createIndex({ index: { fields: ['type', 'characterId'] } })
    await db.createIndex({ index: { fields: ['type', 'characterId', 'likes'] } })
    await db.createIndex({ index: { fields: ['type', 'characterId', 'createdAt'] } })
    await db.createIndex({ index: { fields: ['type', 'textLower'] } })
    await db.createIndex({ index: { fields: ['type', 'textLower', 'createdAt'] } })
    await db.createIndex({ index: { fields: ['type', 'likes'] } })
    console.log('Index Mango créés (messages)')
  } catch (err) {
    console.error('Erreur création index messages :', err)
  }
}
//jutilise db.sync(remote, { live:true, retry:true }) pour une synchronisation bidirectionnelle
//comme ça l'appli reste a jour et je peux utiliser le mode off/online

//Pour la question "tout répliquer ?" Oui, je réplique ici toutes les données des deux bases pour simplifier et garantir l'accès offline.
const startSync = () => {
  const dbChars = storageCharacters.value
  const dbMsgs  = storageMessages.value
  const rChars  = remoteCharacters.value
  const rMsgs   = remoteMessages.value

  if (!dbChars || !dbMsgs || !rChars || !rMsgs) {
    console.log(
      'startSync: pas démarré (dbChars=',
      !!dbChars,
      'dbMsgs=',
      !!dbMsgs,
      'rChars=',
      !!rChars,
      'rMsgs=',
      !!rMsgs,
      ')'
    )
    return
  }

  if (!syncChars) {
    console.log('startSync: démarrage sync bidirectionnelle characters/comments avec', COUCH_REMOTE_CHARACTERS)
    syncChars = dbChars
      .sync(rChars, { live: true, retry: true })
      .on('change', async (info: any) => {
        console.log('sync chars change:', info)
        await handleSyncChange(info, 'chars')
      })
      .on('error', (err: any) => console.error('Erreur sync chars :', err))
  }

  if (!syncMsgs) {
    console.log('startSync: démarrage sync bidirectionnelle messages avec', COUCH_REMOTE_MESSAGES)
    syncMsgs = dbMsgs
      .sync(rMsgs, { live: true, retry: true })
      .on('change', async (info: any) => {
        console.log('sync msgs change:', info)
        await handleSyncChange(info, 'msgs')
      })
      .on('error', (err: any) => console.error('Erreur sync msgs :', err))
  }
}

const stopSync = () => {
  if (syncChars?.cancel) {
    console.log('stopSync: arrêt de la réplication live characters/comments')
    syncChars.cancel()
    syncChars = null
  }
  if (syncMsgs?.cancel) {
    console.log('stopSync: arrêt de la réplication live messages')
    syncMsgs.cancel()
    syncMsgs = null
  }
}

const toggleOnline = () => {
  isOnline.value = !isOnline.value
  console.log('toggleOnline: isOnline =', isOnline.value)
  isOnline.value ? startSync() : stopSync()
}

const mapCharacterDoc = (d: CharacterDoc) => ({
  data: {
    name: d.name,
    age: d.age,
    affiliation: d.affiliation,
    lightsaber: !!d.lightsaber,
    likes: d.likes ?? 0,
    messages: []
  },
  docId: d._id,
  docRev: d._rev || ''
})


const loadMessageMediaUrl = async (messageId: string) => {
  const dbMsgs = storageMessages.value
  if (!dbMsgs) return

  try {
    const blob = await dbMsgs.getAttachment(messageId, 'media')
    if (blob) {
      if (messageMediaUrl.value[messageId]) {
        URL.revokeObjectURL(messageMediaUrl.value[messageId] as string)
      }
      messageMediaUrl.value[messageId] = URL.createObjectURL(blob)
    }
  } catch (err: any) {
    if (err?.status !== 404) {
      console.error('Erreur getAttachment pour message', messageId, err)
    }
  }
}



const loadCharacterMediaUrl = async (characterId: string) => {
  const dbChars = storageCharacters.value
  if (!dbChars) return

  try {
    const blob = await dbChars.getAttachment(characterId, 'media')
    if (blob) {
      if (characterMediaUrl.value[characterId]) {
        URL.revokeObjectURL(characterMediaUrl.value[characterId] as string)
      }
      characterMediaUrl.value[characterId] = URL.createObjectURL(blob)
    }
  } catch (err: any) {
    if (err?.status !== 404) {
      console.error('Erreur getAttachment pour personnage', characterId, err)
    }
  }
}



const loadMessagesForCharacter = async (characterId: string, charIndex: number) => {
  const dbMsgs  = storageMessages.value
  const dbChars = storageCharacters.value
  if (!dbMsgs || !dbChars) return

  const orderByLikes = !!orderByLikesForChar.value[characterId]
  const selector: any = { type: 'message', characterId }

  const sort = orderByLikes
    ? [{ type: 'asc' }, { characterId: 'asc' }, { likes: 'desc' }]
    : [{ type: 'asc' }, { characterId: 'asc' }, { createdAt: 'desc' }]

  if (orderByLikes) selector.likes = { $gte: 0 }
  else selector.createdAt = { $gte: null }

  const { docs: messageDocs } = await dbMsgs.find({
    selector,
    sort,
    limit: LIMIT_MESSAGES_PER_CHARACTER
  })

  const messages = messageDocs as MessageDoc[]

  const uiMessages: Message[] = []

  for (const m of messages) {
    const { docs: lastCommentDocs } = await dbChars.find({
      selector: {
        type: 'comment',
        messageId: m._id,
        createdAt: { $gte: null }
      },
      sort: [{ type: 'asc' }, { messageId: 'asc' }, { createdAt: 'desc' }],
      limit: 1
    })

    const c = lastCommentDocs[0] as CommentDoc | undefined

    uiMessages.push({
      id: m._id,
      text: m.text,
      createdAt: m.createdAt,
      likes: m.likes ?? 0,
      comments: c ? [{ id: c._id, text: c.text, createdAt: c.createdAt }] : []
    })
  }

  if (characters.value[charIndex]?.docId === characterId) {
    characters.value[charIndex].data.messages = uiMessages
  }

  for (const m of uiMessages) {
    await loadMessageMediaUrl(m.id)
  }
}

const reloadAllMessages = async () => {
  for (let i = 0; i < characters.value.length; i++) {
    await loadMessagesForCharacter(characters.value[i].docId, i)
  }
}

//Pour la question "est-ce que allDocs avec "include_docs" fait sens?"
//je dirai que c'est pratique si on a besoin de tout récupérer rapidement si on pas de filtre.
//Mais ici, mango + index est plus adapté.
const fetchData = async (onlyTopMessagesByLikes = false) => {
  const dbChars = storageCharacters.value
  const dbMsgs  = storageMessages.value
  if (!dbChars || !dbMsgs) return

  try {
    if (onlyTopMessagesByLikes) {
      const { docs: msgDocs } = await dbMsgs.find({
        selector: { type: 'message', likes: { $gte: 0 } },
        sort: [{ type: 'asc' }, { likes: 'desc' }],
        limit: TOP_PAGE_SIZE,
        skip: topMessagesPage.value * TOP_PAGE_SIZE
      })

      const messages = msgDocs as MessageDoc[]
      if (!messages.length) {
        characters.value = []
        return
      }

      const characterIds = Array.from(new Set(messages.map(m => m.characterId)))

      const { docs: charDocs } = await dbChars.find({
        selector: { _id: { $in: characterIds } },
        limit: characterIds.length
      })

      const charDocsFiltered = (charDocs as CharacterDoc[]).filter(d => d.type === 'character')
      const charMap = new Map<string, number>()

      characters.value = charDocsFiltered.map((d, idx) => {
        charMap.set(d._id, idx)
        return mapCharacterDoc(d)
      })

    
      for (const c of characters.value) {
        await loadCharacterMediaUrl(c.docId)
      }

      for (const m of messages) {
        const idx = charMap.get(m.characterId)
        if (idx === undefined) continue

        const { docs: lastCommentDocs } = await dbChars.find({
          selector: {
            type: 'comment',
            messageId: m._id,
            createdAt: { $gte: null }
          },
          sort: [{ type: 'asc' }, { messageId: 'asc' }, { createdAt: 'desc' }],
          limit: 1
        })

        const c = lastCommentDocs[0] as CommentDoc | undefined

        const msgUI: Message = {
          id: m._id,
          text: m.text,
          createdAt: m.createdAt,
          likes: m.likes ?? 0,
          comments: c ? [{ id: c._id, text: c.text, createdAt: c.createdAt }] : []
        }

        const arr = characters.value[idx].data.messages || []
        arr.push(msgUI)
        characters.value[idx].data.messages = arr

        await loadMessageMediaUrl(msgUI.id)
      }
    } else {
      const { docs } = await dbChars.find({
        selector: { type: 'character' },
        limit: LIMIT_CHARACTERS
      })
      characters.value = (docs as CharacterDoc[]).map(mapCharacterDoc)

      
      for (const c of characters.value) {
        await loadCharacterMediaUrl(c.docId)
      }

      await reloadAllMessages()
    }
  } catch (err) {
    console.error('Erreur fetch data :', err)
  }
}

const fetchCharactersOnly = async () => {
  const dbChars = storageCharacters.value
  if (!dbChars) return
  const { docs } = await dbChars.find({
    selector: { type: 'character' },
    limit: LIMIT_CHARACTERS
  })
  characters.value = (docs as CharacterDoc[]).map(mapCharacterDoc)

  for (const c of characters.value) {
    await loadCharacterMediaUrl(c.docId)
  }
}

const refreshCharacterById = async (characterId: string) => {
  const dbChars = storageCharacters.value
  if (!dbChars) return

  const idx = characters.value.findIndex(c => c.docId === characterId)
  try {
    const d = await dbChars.get<CharacterDoc>(characterId)
    if (d.type !== 'character') return
    const mapped = mapCharacterDoc(d)
    if (idx >= 0) {
      characters.value[idx] = mapped
      await loadCharacterMediaUrl(characterId)
    } else {
      await fetchCharactersOnly()
    }
  } catch (err: any) {
    if (err?.status === 404) {
      if (idx >= 0) characters.value.splice(idx, 1)
      return
    }
    console.error('refreshCharacterById erreur:', err)
  }
}

const refreshMessagesForCharacterId = async (characterId: string) => {
  const idx = characters.value.findIndex(c => c.docId === characterId)
  if (idx < 0) return
  await loadMessagesForCharacter(characterId, idx)
}

const refreshCommentsForMessageId = async (messageId: string) => {
  for (let charIndex = 0; charIndex < characters.value.length; charIndex++) {
    const msgs = characters.value[charIndex].data.messages || []
    const found = msgs.find(m => m.id === messageId)
    if (found) {
      await loadMessagesForCharacter(characters.value[charIndex].docId, charIndex)
      return
    }
  }
}

let syncRefreshTimer: any = null
let pendingCharIds = new Set<string>()
let pendingMsgIds = new Set<string>()
let pendingNeedFullReload = false

const scheduleTargetedRefresh = () => {
  if (syncRefreshTimer) clearTimeout(syncRefreshTimer)
  syncRefreshTimer = setTimeout(async () => {
    syncRefreshTimer = null

    if (sortByLikes.value) {
      pendingCharIds.clear()
      pendingMsgIds.clear()
      pendingNeedFullReload = false
      await fetchData(true)
      return
    }

    if (pendingNeedFullReload) {
      pendingNeedFullReload = false
      pendingCharIds.clear()
      pendingMsgIds.clear()
      await fetchData(false)
      return
    }

    const charIds = Array.from(pendingCharIds)
    const msgIds = Array.from(pendingMsgIds)
    pendingCharIds.clear()
    pendingMsgIds.clear()

    for (const cid of charIds) {
      await refreshCharacterById(cid)
      await refreshMessagesForCharacterId(cid)
    }
    for (const mid of msgIds) {
      await refreshCommentsForMessageId(mid)
    }
  }, 120)
}

const handleSyncChange = async (info: any, source: 'chars' | 'msgs') => {
  if (!isOnline.value) return

  const changes = info?.change?.docs
  if (!Array.isArray(changes) || !changes.length) {
    pendingNeedFullReload = true
    scheduleTargetedRefresh()
    return
  }

  for (const d of changes) {
    const id: string = d?._id
    const type: string = d?.type

    if (source === 'chars') {
      if (type === 'character' && id) {
        pendingCharIds.add(id)
      } else if (type === 'comment' && d?.messageId) {
        pendingMsgIds.add(d.messageId)
      } else {
        pendingNeedFullReload = true
      }
    }

    if (source === 'msgs') {
      if (type === 'message' && id) {
        const characterId: string | undefined = d?.characterId
        if (characterId) pendingCharIds.add(characterId)
        else pendingNeedFullReload = true
      } else {
        pendingNeedFullReload = true
      }
    }
  }

  scheduleTargetedRefresh()
}

const addCharacter = async () => {
  const dbChars = storageCharacters.value
  if (!dbChars) {
    console.warn('addCharacter: storageCharacters.value est null')
    return
  }

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

    const res = await dbChars.put(doc)
    console.log('addCharacter: écrit en local', doc._id, 'rev=', res.rev)

    characters.value.push(mapCharacterDoc({ ...doc, _rev: res.rev }))

    newCharacter.value = {
      name: '',
      age: 0,
      affiliation: '',
      lightsaber: false,
      likes: 0,
      messages: []
    }
  } catch (err) {
    console.error('Erreur ajout personnage :', err)
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
  const dbChars = storageCharacters.value
  if (!dbChars || !editingCharacter.value) return

  try {
    const curr = characters.value[index]

    const updated = await saveWithConflictRetry<CharacterDoc>(
      dbChars,
      () => dbChars.get(curr.docId),
      latest => {
        latest.name = editingCharacter.value!.name
        latest.age = editingCharacter.value!.age
        latest.affiliation = editingCharacter.value!.affiliation
        latest.affiliationLower = (editingCharacter.value!.affiliation || '').toLowerCase()
        latest.lightsaber = editingCharacter.value!.lightsaber

        if (typeof editingCharacter.value!.likes === 'number') {
          latest.likes = Math.max(latest.likes ?? 0, editingCharacter.value!.likes ?? 0)
        }
        return latest
      }
    )

    if (updated) {
      characters.value[index] = mapCharacterDoc(updated)
      await loadMessagesForCharacter(curr.docId, index)
      await loadCharacterMediaUrl(curr.docId)
    }

    cancelEdit()
  } catch (err) {
    console.error('Erreur modification personnage :', err)
  }
}

const deleteCharacter = async (index: number) => {
  const dbChars = storageCharacters.value
  const dbMsgs  = storageMessages.value
  if (!dbChars || !dbMsgs) return

  if (!confirm('Supprimer ce personnage (et ses messages/commentaires) ?')) return

  try {
    const curr = characters.value[index]
    const characterId = curr.docId

    
    const { docs: msgDocs } = await dbMsgs.find({
      selector: { type: 'message', characterId },
      limit: LIMIT_MESSAGES_PER_CHARACTER
    })

    const msgs = msgDocs as MessageDoc[]

    if (msgs.length) {
      const msgIds = msgs.map(m => m._id)

      
      const { docs: cmtDocs } = await dbChars.find({
        selector: { type: 'comment', messageId: { $in: msgIds } },
        limit: LIMIT_COMMENTS
      })

      const cmts = cmtDocs as CommentDoc[]
      if (cmts.length) {
        await dbChars.bulkDocs(cmts.map(c => ({ ...c, _deleted: true })))
      }

      await dbMsgs.bulkDocs(msgs.map(m => ({ ...m, _deleted: true })))
    }

    
    if (characterMediaUrl.value[characterId]) {
      URL.revokeObjectURL(characterMediaUrl.value[characterId] as string)
      characterMediaUrl.value[characterId] = null
    }

    await removeWithRetry(dbChars, characterId)
    characters.value.splice(index, 1)
  } catch (err) {
    console.error('Erreur suppression personnage :', err)
  }
}

const addMessage = async (charIndex: number) => {
  const dbMsgs = storageMessages.value
  if (!dbMsgs) return

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

    await dbMsgs.put(msg)
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

const cancelEditMessage = () => {
  editingMessageIndex.value = null
  editingMessageText.value = ''
}

const saveEditMessage = async () => {
  const dbMsgs = storageMessages.value
  if (!dbMsgs || !editingMessageIndex.value) return

  try {
    const { charIndex, msgIndex } = editingMessageIndex.value
    const msg = characters.value[charIndex].data.messages![msgIndex]

    const updated = await saveWithConflictRetry<MessageDoc>(
      dbMsgs,
      () => dbMsgs.get(msg.id),
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
  const dbMsgs  = storageMessages.value
  const dbChars = storageCharacters.value
  if (!dbMsgs || !dbChars) return

  if (!confirm('Supprimer ce message (et ses commentaires) ?')) return

  try {
    const char = characters.value[charIndex]
    const msg  = char.data.messages![msgIndex]

    const { docs: cmtDocs } = await dbChars.find({
      selector: { type: 'comment', messageId: msg.id },
      limit: LIMIT_COMMENTS
    })

    const cmts = cmtDocs as CommentDoc[]
    if (cmts.length) {
      await dbChars.bulkDocs(cmts.map(c => ({ ...c, _deleted: true })))
    }

    
    if (messageMediaUrl.value[msg.id]) {
      URL.revokeObjectURL(messageMediaUrl.value[msg.id] as string)
      messageMediaUrl.value[msg.id] = null
    }

    await removeWithRetry(dbMsgs, msg.id)
    await loadMessagesForCharacter(char.docId, charIndex)
  } catch (err) {
    console.error('Erreur suppression message :', err)
  }
}

const likeCharacter = async (charIndex: number) => {
  const dbChars = storageCharacters.value
  if (!dbChars) return

  try {
    const curr = characters.value[charIndex]

    const updated = await saveWithConflictRetry<CharacterDoc>(
      dbChars,
      () => dbChars.get(curr.docId),
      latest => {
        latest.likes = (latest.likes ?? 0) + 1
        return latest
      }
    )

    if (updated) {
      characters.value[charIndex] = mapCharacterDoc(updated)

      if (sortByLikes.value) {
        setTimeout(() => fetchData(true), 120)
      } else {
        await loadMessagesForCharacter(curr.docId, charIndex)
        await loadCharacterMediaUrl(curr.docId)
      }
    }
  } catch (err) {
    console.error('Erreur like personnage :', err)
  }
}

const likeMessage = async (charIndex: number, msgIndex: number) => {
  const dbMsgs = storageMessages.value
  if (!dbMsgs) return

  try {
    const msg = characters.value[charIndex].data.messages![msgIndex]

    const updated = await saveWithConflictRetry<MessageDoc>(
      dbMsgs,
      () => dbMsgs.get(msg.id),
      latest => {
        latest.likes = (latest.likes ?? 0) + 1
        return latest
      }
    )

    if (updated) {
      if (sortByLikes.value) {
        await fetchData(true)
      } else {
        await loadMessagesForCharacter(characters.value[charIndex].docId, charIndex)
      }
    }
  } catch (err) {
    console.error('Erreur like message :', err)
  }
}

const addComment = async (charIndex: number, msgIndex: number) => {
  const dbChars = storageCharacters.value
  if (!dbChars) return

  const key = `${charIndex}-${msgIndex}`
  const text = (newCommentText.value[key] || '').trim()
  if (!text) return

  try {
    const char = characters.value[charIndex]
    const msg  = char.data.messages![msgIndex]

    const c: CommentDoc = {
      _id: 'comment:' + iso(),
      type: 'comment',
      messageId: msg.id,
      text,
      createdAt: iso()
    }

    await dbChars.put(c)
    await loadMessagesForCharacter(char.docId, charIndex)

    newCommentText.value[key] = ''
    visibleComments.value[key] = true
  } catch (err) {
    console.error('Erreur ajout commentaire :', err)
  }
}

const startEditComment = (charIndex: number, msgIndex: number, commentIndex: number) => {
  editingCommentIndex.value = { charIndex, msgIndex, commentIndex }
  editingCommentText.value =
    characters.value[charIndex].data.messages![msgIndex].comments[commentIndex].text
}

const cancelEditComment = () => {
  editingCommentIndex.value = null
  editingCommentText.value = ''
}

const saveEditComment = async () => {
  const dbChars = storageCharacters.value
  if (!dbChars || !editingCommentIndex.value) return

  try {
    const { charIndex, msgIndex, commentIndex } = editingCommentIndex.value
    const char = characters.value[charIndex]
    const cmt  = char.data.messages![msgIndex].comments[commentIndex]

    const updated = await saveWithConflictRetry<CommentDoc>(
      dbChars,
      () => dbChars.get(cmt.id),
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
  const dbChars = storageCharacters.value
  if (!dbChars) return

  if (!confirm('Supprimer ce commentaire ?')) return

  try {
    const char = characters.value[charIndex]
    const cmt  = char.data.messages![msgIndex].comments[commentIndex]

    await removeWithRetry(dbChars, cmt.id)
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
  const dbChars = storageCharacters.value
  const dbMsgs  = storageMessages.value
  if (!dbChars || !dbMsgs) return

  const q = (searchMessageText.value || '').trim().toLowerCase()
  if (!q) {
    sortByLikes.value = false
    topMessagesPage.value = 0
    await fetchData()
    return
  }

  try {
    const upper = q + '\uffff'

    const { docs: msgDocs } = await dbMsgs.find({
      selector: {
        type: 'message',
        textLower: { $gte: q, $lte: upper },
        createdAt: { $gte: null }
      },
      sort: [{ type: 'asc' }, { textLower: 'asc' }, { createdAt: 'desc' }],
      limit: 500
    })

    const messages = msgDocs as MessageDoc[]
    if (!messages.length) {
      characters.value = []
      return
    }

    const characterIds = Array.from(new Set(messages.map(m => m.characterId)))

    const { docs: charDocs } = await dbChars.find({
      selector: { _id: { $in: characterIds } },
      limit: characterIds.length
    })

    const charRows = (charDocs as CharacterDoc[]).filter((d: any) => d?.type === 'character')
    const charMap = new Map<string, number>()

    characters.value = charRows.map((d: CharacterDoc, idx: number) => {
      charMap.set(d._id, idx)
      return mapCharacterDoc(d)
    })

    
    for (const c of characters.value) {
      await loadCharacterMediaUrl(c.docId)
    }

    for (const m of messages) {
      const idx = charMap.get(m.characterId)
      if (idx === undefined) continue

      const { docs: lastCommentDocs } = await dbChars.find({
        selector: {
          type: 'comment',
          messageId: m._id,
          createdAt: { $gte: null }
        },
        sort: [{ type: 'asc' }, { messageId: 'asc' }, { createdAt: 'desc' }],
        limit: 1
      })

      const c = lastCommentDocs[0] as CommentDoc | undefined

      const msgUI: Message = {
        id: m._id,
        text: m.text,
        createdAt: m.createdAt,
        likes: m.likes ?? 0,
        comments: c ? [{ id: c._id, text: c.text, createdAt: c.createdAt }] : []
      }

      const arr = characters.value[idx].data.messages || []
      arr.push(msgUI)
      characters.value[idx].data.messages = arr

      await loadMessageMediaUrl(msgUI.id)
    }
  } catch (err) {
    console.error('Erreur recherche messages :', err)
  }
}
//find sur affiliation, comme ça pas de filtrage en typescript et recherche insensible a la casse.
const searchByAffiliation = async () => {
  const dbChars = storageCharacters.value
  if (!dbChars) return

  const val = (searchAffiliation.value || '').trim().toLowerCase()
  if (!val) {
    sortByLikes.value = false
    topMessagesPage.value = 0
    await fetchData()
    return
  }

  try {
    const { docs } = await dbChars.find({
      selector: { type: 'character', affiliationLower: val },
      limit: LIMIT_CHARACTERS
    })

    characters.value = (docs as CharacterDoc[]).map(mapCharacterDoc)

    
    for (const c of characters.value) {
      await loadCharacterMediaUrl(c.docId)
    }

    await reloadAllMessages()
  } catch (err) {
    console.error('Erreur recherche affiliation :', err)
  }
}

const toggleSortByLikes = async () => {
  if (sortByLikes.value) {
    sortByLikes.value = false
    topMessagesPage.value = 0
    await fetchData()
  } else {
    sortByLikes.value = true
    topMessagesPage.value = 0
    await fetchData(true)
  }
}

const nextTopMessagesPage = async () => {
  const dbMsgs = storageMessages.value
  if (!dbMsgs) return
  topMessagesPage.value++
  await fetchData(true)
}

const prevTopMessagesPage = async () => {
  const dbMsgs = storageMessages.value
  if (!dbMsgs) return
  if (topMessagesPage.value > 0) {
    topMessagesPage.value--
    await fetchData(true)
  }
}



const attachMediaToMessage = async (msgId: string) => {
  const dbMsgs = storageMessages.value
  if (!dbMsgs) return

  const file = messageMediaFile.value[msgId]
  if (!file) {
    alert('Veuillez choisir un fichier avant de l’associer au message.')
    return
  }

  try {
    const doc = await dbMsgs.get<MessageDoc>(msgId)
      //je stocke le média comme attachment couchdb/pouchdb, pour éviter que ce soit trop lourd j'ai fait deux collections.
    await dbMsgs.putAttachment(
      msgId,
      'media',
      doc._rev,
      file,
      file.type || 'application/octet-stream'
    )

    const updatedDoc = await dbMsgs.get<MessageDoc>(msgId)
    updatedDoc.mediaContentType = file.type || 'application/octet-stream'
    await dbMsgs.put(updatedDoc)

    await loadMessageMediaUrl(msgId)
    messageMediaFile.value[msgId] = null

    console.log('Attachment ajouté pour le message', msgId)
  } catch (err) {
    console.error('Erreur attachMediaToMessage :', err)
  }
}

const removeMediaFromMessage = async (msgId: string) => {
  const dbMsgs = storageMessages.value
  if (!dbMsgs) return

  if (!confirm('Supprimer le média associé à ce message ?')) return

  try {
    const doc = await dbMsgs.get<MessageDoc>(msgId)

    await dbMsgs.removeAttachment(msgId, 'media', doc._rev)
    const refreshed = await dbMsgs.get<MessageDoc>(msgId)
    delete refreshed.mediaContentType
    await dbMsgs.put(refreshed)

    if (messageMediaUrl.value[msgId]) {
      URL.revokeObjectURL(messageMediaUrl.value[msgId] as string)
      messageMediaUrl.value[msgId] = null
    }

    console.log('Attachment supprimé pour le message', msgId)
  } catch (err) {
    console.error('Erreur removeMediaFromMessage :', err)
  }
}



const attachMediaToCharacter = async (characterId: string) => {
  const dbChars = storageCharacters.value
  if (!dbChars) return

  const file = characterMediaFile.value[characterId]
  if (!file) {
    alert('Veuillez choisir un fichier avant de l’associer au personnage.')
    return
  }

  try {
    const doc = await dbChars.get<CharacterDoc>(characterId)

    await dbChars.putAttachment(
      characterId,
      'media',
      doc._rev,
      file,
      file.type || 'application/octet-stream'
    )

    const updatedDoc = await dbChars.get<CharacterDoc>(characterId)
    updatedDoc.mediaContentType = file.type || 'application/octet-stream'
    await dbChars.put(updatedDoc)

    await loadCharacterMediaUrl(characterId)
    characterMediaFile.value[characterId] = null

    console.log('Attachment ajouté pour le personnage', characterId)
  } catch (err) {
    console.error('Erreur attachMediaToCharacter :', err)
  }
}

const removeMediaFromCharacter = async (characterId: string) => {
  const dbChars = storageCharacters.value
  if (!dbChars) return

  if (!confirm('Supprimer le média associé à ce personnage ?')) return

  try {
    const doc = await dbChars.get<CharacterDoc>(characterId)

    await dbChars.removeAttachment(characterId, 'media', doc._rev)
    const refreshed = await dbChars.get<CharacterDoc>(characterId)
    delete refreshed.mediaContentType
    await dbChars.put(refreshed)

    if (characterMediaUrl.value[characterId]) {
      URL.revokeObjectURL(characterMediaUrl.value[characterId] as string)
      characterMediaUrl.value[characterId] = null
    }

    console.log('Attachment supprimé pour le personnage', characterId)
  } catch (err) {
    console.error('Erreur removeMediaFromCharacter :', err)
  }
}



onMounted(() => {
  initDatabase()
})
</script>

<template>
  <div class="app">
   
    <header class="header">
      <h1>Personnages</h1>
      <button @click="toggleOnline" class="status-btn" :class="{ online: isOnline }">
        {{ isOnline ? 'En ligne' : 'Hors ligne' }}
      </button>
    </header>

    
    <section class="panel">
      <h2>Nouveau personnage</h2>
      <form @submit.prevent="addCharacter" class="form-row">
        <input v-model="newCharacter.name" placeholder="Nom" required />
        <input v-model.number="newCharacter.age" placeholder="Âge" type="number" required />
        <input v-model="newCharacter.affiliation" placeholder="Affiliation" required />
        <label class="checkbox">
          <input type="checkbox" v-model="newCharacter.lightsaber" />
          <span>Sabre</span>
        </label>
        <button type="submit" class="btn primary">Ajouter</button>
      </form>
    </section>

   
    <section class="panel">
      <h2>Recherche</h2>
      <div class="filters">
        <div class="filter-group">
          <input v-model="searchAffiliation" placeholder="Affiliation..." />
          <button @click="searchByAffiliation" class="btn">Chercher</button>
        </div>
        <div class="filter-group">
          <input v-model="searchMessageText" placeholder="Message..." />
          <button @click="searchMessages" class="btn">Chercher</button>
        </div>
        <button @click="toggleSortByLikes" class="btn" :class="{ active: sortByLikes }">
          {{ sortByLikes ? 'Top 10 activé' : 'Top 10 likés' }}
        </button>
        <div v-if="sortByLikes" class="pagination">
          <button @click="prevTopMessagesPage" :disabled="topMessagesPage === 0" class="btn sm">←</button>
          <span>{{ topMessagesPage + 1 }}</span>
          <button @click="nextTopMessagesPage" class="btn sm">→</button>
        </div>
      </div>
    </section>

   
    <section class="characters-grid">
      <article v-for="(char, index) in characters" :key="char.docId" class="character-card">
        
        
        <template v-if="editingIndex !== index">
          
          <div class="card-header">
            <div class="avatar">
              <span>{{ char.data.name.charAt(0) }}</span>
            </div>
            <div class="info">
              <h3>{{ char.data.name }}</h3>
              <p>
                {{ char.data.age }} ans • {{ char.data.affiliation }}
                <span v-if="char.data.lightsaber"></span>
              </p>
            </div>
            <button @click="likeCharacter(index)" class="like-btn">
              ❤️ {{ char.data.likes || 0 }}
            </button>
          </div>

          
          <div class="card-actions">
            <div class="btn-row">
              <button @click="startEdit(index)" class="btn sm success">Modifier</button>
              <button @click="deleteCharacter(index)" class="btn sm danger">✕ Supprimer</button>
            </div>
          </div>

          
          <div class="media-section">
            <div class="media-header">
              <span>Média associé</span>
            </div>
            <div class="media-controls">
              <input type="file" :id="'cf-' + char.docId" hidden
                @change="(e: Event) => {
                  const t = e.target as HTMLInputElement
                  if (t.files?.[0]) characterMediaFile[char.docId] = t.files[0]
                }" />
              <label :for="'cf-' + char.docId" class="btn sm ghost">Choisir un media</label>
              <button v-if="characterMediaFile[char.docId]" @click="attachMediaToCharacter(char.docId)" class="btn sm primary">Envoyer</button>
              <button v-if="characterMediaUrl[char.docId]" @click="removeMediaFromCharacter(char.docId)" class="btn sm danger">Supprimer</button>
            </div>
            <div v-if="characterMediaUrl[char.docId]" class="media-preview">
              <img :src="characterMediaUrl[char.docId] as string" alt="Média du personnage" />
            </div>
          </div>

          <!-- Section Messages -->
          <div class="messages-section">
            <div class="section-header">
              <span>Messages</span>
              <button @click="toggleCharSort(index)" class="btn xs" :class="{ active: orderByLikesForChar[char.docId] }">
                {{ orderByLikesForChar[char.docId] ? '♥' : '📅' }}
              </button>
            </div>

            <div class="new-message">
              <input v-model="newMessageText[index]" placeholder="Nouveau message..." />
              <button @click="addMessage(index)" class="btn sm primary">+</button>
            </div>

            
            <div class="messages-list">
              <div v-for="(msg, msgIndex) in char.data.messages" :key="msg.id" class="message">
                
                
                <template v-if="editingMessageIndex?.charIndex === index && editingMessageIndex?.msgIndex === msgIndex">
                  <div class="edit-row">
                    <input v-model="editingMessageText" />
                    <button @click="saveEditMessage" class="btn xs primary">✓</button>
                    <button @click="cancelEditMessage" class="btn xs">✕</button>
                  </div>
                </template>

                
                <template v-else>
                  <div class="message-header">
                    <p class="text">{{ msg.text }}</p>
                    <button @click="likeMessage(index, msgIndex)" class="like-btn sm">♥ {{ msg.likes || 0 }}</button>
                  </div>
                  <div class="message-meta">
                    <span>{{ new Date(msg.createdAt).toLocaleDateString() }}</span>
                    <div class="msg-actions">
                      <button @click="startEditMessage(index, msgIndex)" class="btn xs ghost">✎</button>
                      <button @click="deleteMessage(index, msgIndex)" class="btn xs ghost">✕</button>
                    </div>
                  </div>

                  
                  <div class="media-attach">
                    <div class="media-controls inline">
                      <input type="file" :id="'mf-' + msg.id" hidden
                        @change="(e: Event) => {
                          const t = e.target as HTMLInputElement
                          if (t.files?.[0]) messageMediaFile[msg.id] = t.files[0]
                        }" />
                      <label :for="'mf-' + msg.id" class="btn xs ghost">📎</label>
                      <button v-if="messageMediaFile[msg.id]" @click="attachMediaToMessage(msg.id)" class="btn xs primary">↑</button>
                      <button v-if="messageMediaUrl[msg.id]" @click="removeMediaFromMessage(msg.id)" class="btn xs danger">🗑</button>
                    </div>
                    <div v-if="messageMediaUrl[msg.id]" class="media-preview sm">
                      <img :src="messageMediaUrl[msg.id] as string" alt="Média du message" />
                    </div>
                  </div>

                  <!-- Commentaires -->
                  <div class="comments">
                    <div class="new-comment">
                      <input v-model="newCommentText[`${index}-${msgIndex}`]" placeholder="Commenter..." />
                      <button @click="addComment(index, msgIndex)" class="btn xs">→</button>
                    </div>

                    <div v-for="(comment, cIdx) in (visibleComments[`${index}-${msgIndex}`] ? msg.comments : msg.comments.slice(-1))" 
                         :key="comment.id" class="comment">
                      <template v-if="editingCommentIndex?.charIndex === index && editingCommentIndex?.msgIndex === msgIndex && editingCommentIndex?.commentIndex === cIdx">
                        <input v-model="editingCommentText" />
                        <button @click="saveEditComment" class="btn xs primary">✓</button>
                        <button @click="cancelEditComment" class="btn xs">✕</button>
                      </template>
                      <template v-else>
                        <span>{{ comment.text }}</span>
                        <div class="comment-actions">
                          <button @click="startEditComment(index, msgIndex, cIdx)" class="btn xs ghost">✎</button>
                          <button @click="deleteComment(index, msgIndex, cIdx)" class="btn xs ghost">✕</button>
                        </div>
                      </template>
                    </div>

                    <button v-if="msg.comments.length > 1" @click="toggleCommentVisibility(index, msgIndex)" class="link">
                      {{ visibleComments[`${index}-${msgIndex}`] ? 'Masquer' : `+${msg.comments.length - 1} commentaires` }}
                    </button>
                  </div>
                </template>
              </div>
            </div>
          </div>
        </template>

        
        <template v-else>
          <form @submit.prevent="saveEdit(index)" class="edit-form">
            <input v-model="editingCharacter!.name" placeholder="Nom" required />
            <input v-model.number="editingCharacter!.age" placeholder="Âge" type="number" required />
            <input v-model="editingCharacter!.affiliation" placeholder="Affiliation" required />
            <label class="checkbox">
              <input type="checkbox" v-model="editingCharacter!.lightsaber" />
              <span>Sabre laser</span>
            </label>
            <div class="btn-row">
              <button type="submit" class="btn primary">Enregistrer</button>
              <button type="button" @click="cancelEdit" class="btn">Annuler</button>
            </div>
          </form>
        </template>
      </article>
    </section>
  </div>
</template>

<style scoped>

* {
  box-sizing: border-box;
}

.app {
  --bg: #0d0d0d;
  --surface: #1a1a1a;
  --surface-2: #242424;
  --surface-3: #2e2e2e;
  --border: #333;
  --border-light: #444;
  --text: #eee;
  --text-dim: #999;
  --text-muted: #666;
  --primary: #3b82f6;
  --primary-dim: #2563eb;
  --success: #22c55e;
  --danger: #ef4444;
  --warning: #f59e0b;
  --pink: #ec4899;
  
  max-width: 800px;
  margin: 0 auto;
  padding: 1rem;
  background: var(--bg);
  color: var(--text);
  font-family: system-ui, -apple-system, sans-serif;
  font-size: 13px;
  min-height: 100vh;
}


.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0.75rem 0;
  margin-bottom: 1rem;
  border-bottom: 1px solid var(--border);
}

.header h1 {
  font-size: 1.25rem;
  font-weight: 600;
  margin: 0;
}

.status-btn {
  padding: 0.35rem 0.75rem;
  border-radius: 1rem;
  border: 1px solid var(--border);
  background: var(--surface);
  color: var(--text-dim);
  font-size: 11px;
  cursor: pointer;
  transition: all 0.2s;
}

.status-btn.online {
  background: var(--success);
  color: #fff;
  border-color: var(--success);
  box-shadow: 0 0 8px rgba(34, 197, 94, 0.4);
}


.panel {
  background: var(--surface);
  border: 1px solid var(--border);
  border-radius: 10px;
  padding: 0.875rem;
  margin-bottom: 0.875rem;
}

.panel h2 {
  font-size: 0.8rem;
  font-weight: 600;
  color: var(--text-dim);
  margin: 0 0 0.625rem 0;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.form-row {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
  align-items: center;
}

.form-row input[type="text"],
.form-row input[type="number"] {
  flex: 1;
  min-width: 100px;
}


input[type="text"],
input[type="number"],
input:not([type]) {
  padding: 0.5rem 0.75rem;
  background: var(--surface-2);
  border: 1px solid var(--border);
  border-radius: 6px;
  color: var(--text);
  font-size: 12px;
  transition: all 0.15s;
}

input::placeholder {
  color: var(--text-muted);
}

input:focus {
  outline: none;
  border-color: var(--primary);
  box-shadow: 0 0 0 2px rgba(59, 130, 246, 0.2);
}

.checkbox {
  display: flex;
  align-items: center;
  gap: 0.375rem;
  font-size: 12px;
  color: var(--text-dim);
  cursor: pointer;
}

.checkbox input {
  width: 14px;
  height: 14px;
  accent-color: var(--primary);
}


.btn {
  padding: 0.5rem 0.875rem;
  border: 1px solid var(--border);
  border-radius: 6px;
  background: var(--surface-2);
  color: var(--text);
  font-size: 11px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.15s;
  white-space: nowrap;
}

.btn:hover {
  background: var(--surface-3);
  border-color: var(--border-light);
}

.btn.sm { padding: 0.35rem 0.625rem; font-size: 10px; }
.btn.xs { padding: 0.25rem 0.5rem; font-size: 10px; }

.btn.primary {
  background: var(--primary);
  border-color: var(--primary);
  color: #fff;
}
.btn.primary:hover { background: var(--primary-dim); }

.btn.success {
  background: var(--success);
  border-color: var(--success);
  color: #fff;
}

.btn.danger {
  background: var(--danger);
  border-color: var(--danger);
  color: #fff;
}

.btn.ghost {
  background: transparent;
  border-color: transparent;
}
.btn.ghost:hover {
  background: var(--surface-3);
}

.btn.active {
  background: var(--warning);
  border-color: var(--warning);
  color: #000;
}

.btn:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}

.btn-row {
  display: flex;
  gap: 0.375rem;
}

.link {
  background: none;
  border: none;
  color: var(--primary);
  font-size: 11px;
  cursor: pointer;
  padding: 0.25rem 0;
}
.link:hover { text-decoration: underline; }


.filters {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
  align-items: center;
}

.filter-group {
  display: flex;
  gap: 0.25rem;
}

.filter-group input {
  width: 120px;
}

.pagination {
  display: flex;
  align-items: center;
  gap: 0.375rem;
  font-size: 11px;
  color: var(--text-dim);
}


.characters-grid {
  display: grid;
  gap: 0.75rem;
}


.character-card {
  background: var(--surface);
  border: 1px solid var(--border);
  border-radius: 10px;
  padding: 0.875rem;
  transition: all 0.2s;
}

.character-card:hover {
  border-color: var(--border-light);
}


.card-header {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  margin-bottom: 0.625rem;
}

.avatar {
  width: 36px;
  height: 36px;
  border-radius: 50%;
  background: #1568b0;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 600;
  font-size: 14px;
  color: #fff;
  flex-shrink: 0;
}

.info {
  flex: 1;
  min-width: 0;
}

.info h3 {
  margin: 0;
  font-size: 0.95rem;
  font-weight: 600;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.info p {
  margin: 0.125rem 0 0;
  font-size: 11px;
  color: var(--text-dim);
}

.like-btn {
  padding: 0.35rem 0.625rem;
  border-radius: 1rem;
  border: none;
  background: linear-gradient(135deg, var(--pink), #db2777);
  color: #fff;
  font-size: 11px;
  font-weight: 500;
  cursor: pointer;
  transition: transform 0.15s;
}

.like-btn:hover {
  transform: scale(1.05);
}

.like-btn.sm {
  padding: 0.2rem 0.5rem;
  font-size: 10px;
}


.card-actions {
  display: flex;
  justify-content: flex-start;
  align-items: center;
  padding: 0.5rem 0;
  border-bottom: 1px solid var(--border);
  margin-bottom: 0.625rem;
}


.media-section {
  background: var(--surface-2);
  border: 1px solid var(--border);
  border-radius: 8px;
  padding: 0.625rem;
  margin-bottom: 0.75rem;
}

.media-header {
  font-size: 11px;
  font-weight: 600;
  color: var(--text-dim);
  margin-bottom: 0.5rem;
  text-transform: uppercase;
  letter-spacing: 0.3px;
}

.media-controls {
  display: flex;
  gap: 0.375rem;
  align-items: center;
  flex-wrap: wrap;
}

.media-controls.inline {
  margin-bottom: 0.375rem;
}

.media-preview {
  margin-top: 0.5rem;
  border-radius: 6px;
  overflow: hidden;
  background: var(--surface-3);
  display: inline-block;
}

.media-preview img {
  display: block;
  max-width: 200px;
  max-height: 150px;
  object-fit: contain;
  border-radius: 6px;
}

.media-preview.sm img {
  max-width: 140px;
  max-height: 100px;
}


.media-attach {
  margin-top: 0.5rem;
  padding-top: 0.5rem;
  border-top: 1px dashed var(--border);
}


.messages-section {
  margin-top: 0.5rem;
}

.section-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 0.5rem;
  font-size: 11px;
  font-weight: 600;
  color: var(--text-dim);
  text-transform: uppercase;
  letter-spacing: 0.3px;
}

.new-message {
  display: flex;
  gap: 0.375rem;
  margin-bottom: 0.5rem;
}

.new-message input {
  flex: 1;
}

.messages-list {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}


.message {
  background: var(--surface-2);
  border: 1px solid var(--border);
  border-radius: 8px;
  padding: 0.625rem;
}

.message-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  gap: 0.5rem;
}

.message-header .text {
  margin: 0;
  font-size: 12px;
  line-height: 1.4;
  flex: 1;
}

.message-meta {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 0.375rem;
  font-size: 10px;
  color: var(--text-muted);
}

.msg-actions {
  display: flex;
  gap: 0.125rem;
}

.edit-row {
  display: flex;
  gap: 0.375rem;
  align-items: center;
}

.edit-row input {
  flex: 1;
}


.comments {
  margin-top: 0.5rem;
  padding-top: 0.5rem;
  border-top: 1px dashed var(--border);
}

.new-comment {
  display: flex;
  gap: 0.25rem;
  margin-bottom: 0.375rem;
}

.new-comment input {
  flex: 1;
  padding: 0.35rem 0.5rem;
  font-size: 11px;
}

.comment {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.5rem;
  padding: 0.35rem 0.5rem;
  background: var(--surface-3);
  border-radius: 4px;
  margin-bottom: 0.25rem;
  font-size: 11px;
}

.comment span {
  flex: 1;
  color: var(--text-dim);
}

.comment input {
  flex: 1;
  padding: 0.25rem 0.5rem;
  font-size: 11px;
}

.comment-actions {
  display: flex;
  gap: 0.125rem;
}


.edit-form {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 0.5rem;
}

.edit-form .checkbox {
  grid-column: span 2;
}

.edit-form .btn-row {
  grid-column: span 2;
  justify-content: flex-start;
}


@media (max-width: 600px) {
  .app {
    padding: 0.75rem;
  }
  
  .form-row {
    flex-direction: column;
  }
  
  .form-row input {
    width: 100%;
  }
  
  .filters {
    flex-direction: column;
    align-items: stretch;
  }
  
  .filter-group {
    width: 100%;
  }
  
  .filter-group input {
    flex: 1;
    width: auto;
  }
  
  .card-header {
    flex-wrap: wrap;
  }
  
  .card-actions {
    flex-direction: column;
    gap: 0.5rem;
    align-items: flex-start;
  }
  
  .edit-form {
    grid-template-columns: 1fr;
  }
  
  .edit-form .checkbox,
  .edit-form .btn-row {
    grid-column: span 1;
  }

  .media-preview img {
    max-width: 100%;
    max-height: 120px;
  }

  .media-preview.sm img {
    max-width: 100%;
    max-height: 80px;
  }
}


::-webkit-scrollbar {
  width: 6px;
  height: 6px;
}

::-webkit-scrollbar-track {
  background: var(--bg);
}

::-webkit-scrollbar-thumb {
  background: var(--border-light);
  border-radius: 3px;
}

::-webkit-scrollbar-thumb:hover {
  background: var(--text-muted);
}
</style>