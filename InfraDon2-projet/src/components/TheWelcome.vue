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
  // type MIME du média attaché (si présent)
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

const COUCH_REMOTE = 'http://RomainBlanchard:admin@localhost:5984/coachdb'

const storage = ref<any>(null)
const remote = ref<any>(null)

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

const sortByLikes = ref(false)
const searchAffiliation = ref('')

const orderByLikesForChar = ref<Record<string, boolean>>({})

// -----------------------------------------------------------------------------
// AFFICHAGE DES 10 DOCUMENTS LES PLUS "LIKÉS" + PAGINATION
// -----------------------------------------------------------------------------
// Exigences :
//  - Afficher les 10 documents les plus “likés“
//  - Offrir la possibilité d’afficher les 10 documents suivants sur le même principe
//
// Ici, on considère que le "document" à trier par likes est un message (type: 'message').
// On s’appuie sur PouchDB.find + index Mango (type, likes) pour déléguer tri/pagination
// à la base (et ne PAS faire les tris en TypeScript).
//
// On évite ainsi de charger tous les messages (pas de allDocs+tri en mémoire) pour
// n’en afficher que 10. On ne récupère que la "page" utile depuis la DB.
//
const topMessagesPage = ref(0) // 0 = première page, 1 = page suivante, etc.
const TOP_PAGE_SIZE = 10

// Fichiers sélectionnés pour chaque message (clé = messageId)
const messageMediaFile = ref<Record<string, File | null>>({})

// URLs temporaires pour prévisualiser les médias (clé = messageId)
const messageMediaUrl = ref<Record<string, string | null>>({})

const iso = () => new Date().toISOString()

// -----------------------------------------------------------------------------
// saveWithConflictRetry
// -----------------------------------------------------------------------------
// Utilise db.get + db.put pour gérer les conflits de révision (409).
// Question "Quelles méthodes PouchDB sont utiles ?" :
//  - db.get : pour récupérer la dernière révision du doc
//  - db.put : pour appliquer les modifications
//  - Gestion des conflits côté client via une fonction de merge (mergeFn).
// -----------------------------------------------------------------------------
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

const initDatabase = async () => {
  const localDB = new PouchDB('infradon-blaro')
  storage.value = localDB

  remote.value = new PouchDB(COUCH_REMOTE)
  console.log('initDatabase: local= infradon-blaro, remote=', COUCH_REMOTE)

  await createIndexes()
  await fetchData()
  startSync()
}

/**
 * Index Mango nécessaires pour déléguer les tris / recherches à PouchDB
 * plutôt que de trier/filtrer en TS.
 *
 * Question : "Est-ce que allDocs({ include_docs: true }) fait sens ?"
 * - Dans cette appli, NON :
 *   * allDocs(include_docs) chargerait potentiellement toute la base, y compris
 *     des documents qui ne sont pas nécessaires pour l’écran courant.
 *   * On devrait ensuite filtrer/ trier en TypeScript, ce qui va à l’encontre
 *     de la consigne : "Les traitements de recherches et de tries ne doivent
 *     pas être exécuté en TS."
 *
 * Choix :
 * - Utiliser db.find({ selector, sort }) + db.createIndex(...) pour :
 *   * Récupérer uniquement les documents utiles (messages d’un personnage, top 10, etc.)
 *   * Déléguer les tris/filtrages à PouchDB (et donc, à la base distante lors de la réplication)
 * - Les index (type, likes, characterId, textLower...) sont conçus pour couvrir
 *   les cas d’usage du TP : recherche, tri par likes, filtre par affiliation, etc.
 */
const createIndexes = async () => {
  if (!storage.value) return
  try {
    await storage.value.createIndex({ index: { fields: ['type'] } })
    await storage.value.createIndex({ index: { fields: ['type', 'likes'] } })
    await storage.value.createIndex({ index: { fields: ['type', 'affiliationLower'] } })
    await storage.value.createIndex({ index: { fields: ['type', 'characterId'] } })
    await storage.value.createIndex({ index: { fields: ['type', 'characterId', 'likes'] } })
    await storage.value.createIndex({ index: { fields: ['type', 'characterId', 'createdAt'] } })
    await storage.value.createIndex({ index: { fields: ['type', 'textLower'] } })
    await storage.value.createIndex({ index: { fields: ['type', 'textLower', 'createdAt'] } })
    await storage.value.createIndex({ index: { fields: ['type', 'messageId'] } })
    console.log('Index Mango créés')
  } catch (err) {
    console.error('Erreur création index :', err)
  }
}

/**
 * Réplication bidirectionnelle en live entre base locale et CouchDB distante.
 *
 * Question : "Faut-il tout répliquer ?"
 * - Dans ce TP, on a choisi OUI :
 *   * volume de données limité
 *   * on souhaite un mode offline complet (toute l’application fonctionne sans réseau)
 *   * cohérence simple : la base locale = miroir fonctionnel de la base distante
 *
 * Remarque :
 * - Pour une application réelle avec beaucoup de données, on pourrait :
 *   * faire de la réplication filtrée (seulement les docs d'un utilisateur, etc.)
 *   * utiliser plusieurs bases (ex: /messages-recents, /messages-archives)
 *   pour réduire les coûts réseau/stockage.
 */
const startSync = () => {
  if (!storage.value || !remote.value || syncHandler) {
    console.log('startSync: pas démarré (storage=', !!storage.value, 'remote=', !!remote.value, 'syncHandler=', !!syncHandler, ')')
    return
  }
  console.log('startSync: démarrage sync bidirectionnelle avec', COUCH_REMOTE)
  // Méthodes PouchDB utiles ici :
  // - db.sync(remote, { live: true, retry: true }) pour garder la base locale
  //   toujours à jour (offline-first) sans faire d’appels manuels en continu.
  syncHandler = storage.value
    .sync(remote.value, { live: true, retry: true })
    .on('change', async (info: any) => {
      console.log('sync change:', info)
      await fetchData(sortByLikes.value)
    })
    .on('error', (err: any) => console.error('Erreur sync :', err))
}

const stopSync = () => {
  if (syncHandler?.cancel) {
    console.log('stopSync: arrêt de la réplication live')
    syncHandler.cancel()
    syncHandler = null
  }
}

const toggleOnline = () => {
  isOnline.value = !isOnline.value
  console.log('toggleOnline: isOnline =', isOnline.value)
  isOnline.value ? startSync() : stopSync()
}

const mapCharacterDoc = (d: CharacterDoc) => ({
  data: { name: d.name, age: d.age, affiliation: d.affiliation, lightsaber: !!d.lightsaber, likes: d.likes ?? 0, messages: [] },
  docId: d._id,
  docRev: d._rev || ''
})

/**
 * Charge (si présent) l’attachment "media" pour un message donné et produit une URL locale (createObjectURL).
 *
 * Efficacité & réponses aux questions :
 * - On NE fait pas de allDocs({ attachments: true, include_docs: true }) qui renverrait
 *   tous les blobs de la base (inefficace en mémoire + réseau).
 * - On appelle db.getAttachment(messageId, 'media') uniquement pour les messages
 *   effectivement affichés. Cela réduit les échanges au strict minimum nécessaire.
 *
 * Méthodes utilisées :
 * - db.getAttachment : pour récupérer le Blob correspondant à l’attachment
 * - URL.createObjectURL (API Web) : pour générer une URL utilisable dans <img> etc.
 * - URL.revokeObjectURL : pour libérer la ressource quand elle n’est plus utile.
 */
const loadMessageMediaUrl = async (messageId: string) => {
  if (!storage.value) return
  try {
    const blob = await storage.value.getAttachment(messageId, 'media')
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

const loadMessagesForCharacter = async (characterId: string, charIndex: number) => {
  if (!storage.value) return
  const orderByLikes = !!orderByLikesForChar.value[characterId]

  const selector: any = { type: 'message', characterId }
  const sort = orderByLikes
    ? [{ type: 'asc' }, { characterId: 'asc' }, { likes: 'desc' }]
    : [{ type: 'asc' }, { characterId: 'asc' }, { createdAt: 'desc' }]

  if (orderByLikes) selector.likes = { $gte: 0 }
  else selector.createdAt = { $gte: null }

  // Ici, on utilise db.find plutôt que allDocs :
  //  - db.find permet de sélectionner uniquement les messages d’un personnage,
  //    avec un tri côté DB (par date ou par likes).
  //  - Cela évite de charger tous les messages de tous les personnages.
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

  for (const m of uiMessages) {
    await loadMessageMediaUrl(m.id)
  }
}

const reloadAllMessages = async () => {
  for (let i = 0; i < characters.value.length; i++) {
    await loadMessagesForCharacter(characters.value[i].docId, i)
  }
}

/**
 *
 * Question : "Est-ce que allDocs(include_docs) fait sens ici ?"
 *  - Non, car allDocs chargerait potentiellement des centaines de messages pour
 *    ensuite ne garder que 10 en mémoire. On préfère laisser la DB ne renvoyer
 *    que les 10 nécessaires via limit/skip.
 */
const fetchData = async (onlyTopMessagesByLikes = false) => {
  if (!storage.value) return
  try {
    if (onlyTopMessagesByLikes) {
      const { docs: msgDocs } = await storage.value.find({
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
      const { docs: charDocs } = await storage.value.find({
        selector: { _id: { $in: characterIds } },
        limit: characterIds.length
      })

      const charDocsFiltered = (charDocs as CharacterDoc[]).filter(d => d.type === 'character')
      const charMap = new Map<string, number>()

      characters.value = charDocsFiltered.map((d, idx) => {
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

        await loadMessageMediaUrl(msgUI.id)
      }
    } else {
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

const addCharacter = async () => {
  if (!storage.value){
    console.warn('addCharacter: storage.value est null')
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

    const res = await storage.value.put(doc)
    console.log('addCharacter: écrit en local', doc._id, 'rev=', res.rev)

    if (remote.value) {
      // Question pour l'utilité des méthodes pouchDB
      // - db.replicate.to(remote) : ici, permet de pousser explicitement les nouvelles
      //   données vers le serveur en plus du sync live, pour un retour plus rapide.
      try {
        const info = await storage.value.replicate.to(remote.value)
        console.log('addCharacter: réplication vers CouchDB OK', info)
      } catch (repErr) {
        console.error('addCharacter: erreur réplication vers CouchDB', repErr)
      }
    } else {
      console.warn('addCharacter: remote DB non initialisée, pas de réplication immédiate')
    }

    characters.value.push(mapCharacterDoc({ ...doc, _rev: res.rev }))
    newCharacter.value = { name: '', age: 0, affiliation: '', lightsaber: false, likes: 0, messages: [] }
  } catch (err) {
    console.error('Erreur ajout personnage :', err)
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

    if (remote.value) {
      try {
        const info = await storage.value.replicate.to(remote.value)
        console.log('deleteCharacter: réplication suppressions vers CouchDB OK', info)
      } catch (err) {
        console.error('deleteCharacter: erreur réplication suppressions vers CouchDB', err)
      }
    }
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

    if (remote.value) {
      try {
        const info = await storage.value.replicate.to(remote.value)
        console.log('addMessage: réplication vers CouchDB OK', info)
      } catch (err) {
        console.error('addMessage: erreur réplication vers CouchDB', err)
      }
    }

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

    if (remote.value) {
      try {
        const info = await storage.value.replicate.to(remote.value)
        console.log('deleteMessage: réplication suppressions vers CouchDB OK', info)
      } catch (err) {
        console.error('deleteMessage: erreur réplication suppressions vers CouchDB', err)
      }
    }
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

      if (remote.value) {
        try {
          const info = await storage.value.replicate.to(remote.value)
          console.log('likeCharacter: réplication vers CouchDB OK', info)
        } catch (err) {
          console.error('likeCharacter: erreur réplication vers CouchDB', err)
        }
      }
    }
  } catch (err) {
    console.error('Erreur like personnage :', err)
  }
}

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
        await fetchData(true)
      } else {
        await loadMessagesForCharacter(characters.value[charIndex].docId, charIndex)
      }

      if (remote.value) {
        try {
          const info = await storage.value.replicate.to(remote.value)
          console.log('likeMessage: réplication vers CouchDB OK', info)
        } catch (err) {
          console.error('likeMessage: erreur réplication vers CouchDB', err)
        }
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

    if (remote.value) {
      try {
        const info = await storage.value.replicate.to(remote.value)
        console.log('addComment: réplication vers CouchDB OK', info)
      } catch (err) {
        console.error('addComment: erreur réplication vers CouchDB', err)
      }
    }
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

      if (remote.value) {
        try {
          const info = await storage.value.replicate.to(remote.value)
          console.log('saveEditComment: réplication vers CouchDB OK', info)
        } catch (err) {
          console.error('saveEditComment: erreur réplication vers CouchDB', err)
        }
      }
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

    if (remote.value) {
      try {
        const info = await storage.value.replicate.to(remote.value)
        console.log('deleteComment: réplication suppressions vers CouchDB OK', info)
      } catch (err) {
        console.error('deleteComment: erreur réplication suppressions vers CouchDB', err)
      }
    }
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
  if (!q) { sortByLikes.value = false; topMessagesPage.value = 0; await fetchData(); return }
  try {
    const upper = q + '\uffff'
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
      characters.value[idx].data.messages = arr

      await loadMessageMediaUrl(msgUI.id)
    }
  } catch (err) {
    console.error('Erreur recherche messages :', err)
  }
}

const searchByAffiliation = async () => {
  if (!storage.value) return
  const val = (searchAffiliation.value || '').trim().toLowerCase()
  if (!val) { sortByLikes.value = false; topMessagesPage.value = 0; await fetchData(); return }
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
  if (!storage.value) return
  topMessagesPage.value++
  await fetchData(true)
}

const prevTopMessagesPage = async () => {
  if (!storage.value) return
  if (topMessagesPage.value > 0) {
    topMessagesPage.value--
    await fetchData(true)
  }
}

/**
 *
 * Questions utilité et efficacité :
 * - db.putAttachment :
 *      permet d'associer un Blob (File du navigateur) au document sans créer
 *        un sous-document spécifique.
 * - Je ne duplique pas le Blob dans un autre doc => moins de données.
 */
const attachMediaToMessage = async (msgId: string) => {
  if (!storage.value) return
  const file = messageMediaFile.value[msgId]
  if (!file) return

  try {
    const doc = await storage.value.get<MessageDoc>(msgId)

    const res = await storage.value.putAttachment(
      msgId,
      'media',
      doc._rev,
      file,
      file.type || 'application/octet-stream'
    )

    const updatedDoc = await storage.value.get<MessageDoc>(msgId)
    updatedDoc.mediaContentType = file.type || 'application/octet-stream'
    const res2 = await storage.value.put(updatedDoc)

    await loadMessageMediaUrl(msgId)

    if (remote.value) {
      try {
        const info = await storage.value.replicate.to(remote.value)
        console.log('attachMediaToMessage: réplication vers CouchDB OK', info)
      } catch (err) {
        console.error('attachMediaToMessage: erreur réplication vers CouchDB', err)
      }
    }

    console.log('Attachment ajouté pour le message', msgId, res2.rev)
  } catch (err) {
    console.error('Erreur attachMediaToMessage :', err)
  }
}

/**
 * Supprime l’attachment "media" associé à un message.
 *
 * choix :
 * - j'utilise db.removeAttachment pour ne pas supprimer le document complet.
 * - je libère aussi l’URL locale (URL.revokeObjectURL) pour la mémoire.
 */
const removeMediaFromMessage = async (msgId: string) => {
  if (!storage.value) return

  if (!confirm('Supprimer le média associé à ce message ?')) return

  try {
    const doc = await storage.value.get<MessageDoc>(msgId)
    const res = await storage.value.removeAttachment(msgId, 'media', doc._rev)

    const refreshed = await storage.value.get<MessageDoc>(msgId)
    delete refreshed.mediaContentType
    const res2 = await storage.value.put(refreshed)

    if (messageMediaUrl.value[msgId]) {
      URL.revokeObjectURL(messageMediaUrl.value[msgId] as string)
      messageMediaUrl.value[msgId] = null
    }

    if (remote.value) {
      try {
        const info = await storage.value.replicate.to(remote.value)
        console.log('removeMediaFromMessage: réplication vers CouchDB OK', info)
      } catch (err) {
        console.error('removeMediaFromMessage: erreur réplication vers CouchDB', err)
      }
    }

    console.log('Attachment supprimé pour le message', msgId, res2.rev)
  } catch (err) {
    console.error('Erreur removeMediaFromMessage :', err)
  }
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

    <div v-if="sortByLikes" class="mt-2 flex items-center space-x-2">
      <button
        @click="prevTopMessagesPage"
        class="px-3 py-1 rounded bg-gray-500 text-white text-xs disabled:opacity-50"
        :disabled="topMessagesPage === 0"
      >
        10 précédents
      </button>
      <button
        @click="nextTopMessagesPage"
        class="px-3 py-1 rounded bg-gray-700 text-white text-xs"
      >
        10 suivants
      </button>
      <span class="text-xs text-gray-600">Page {{ topMessagesPage + 1 }}</span>
    </div>
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

              
              <div class="mt-2">
                <label class="text-xs text-gray-600 block mb-1">Média associé (image, audio, etc.)</label>
                <div class="flex items-center space-x-2">
                  <input
                    type="file"
                    class="text-xs"
                    @change="(e: Event) => {
                      const input = e.target as HTMLInputElement
                      if (input.files && input.files[0]) {
                        messageMediaFile[msg.id] = input.files[0]
                      }
                    }"
                  />
                  <button
                    @click="attachMediaToMessage(msg.id)"
                    class="bg-blue-400 text-white px-2 py-1 rounded text-xs"
                  >
                    Associer
                  </button>
                  <button
                    v-if="messageMediaUrl[msg.id]"
                    @click="removeMediaFromMessage(msg.id)"
                    class="bg-red-400 text-white px-2 py-1 rounded text-xs"
                  >
                    Supprimer média
                  </button>
                </div>

                <div v-if="messageMediaUrl[msg.id]" class="mt-2">
                  <img
                    :src="messageMediaUrl[msg.id] as string"
                    alt="Média du message"
                    class="max-w-xs max-h-48 border rounded"
                  />
                </div>
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

                <!--
                  Exigences :
                  - "Afficher pour chaque document, le dernier commentaire associé"
                    => on affiche par défaut le dernier via msg.comments.slice(-1)
                  - "Pour chaque document, offrir la possibilité de voir tous les commentaires"
                    => si visibleComments[...] est true, on affiche msg.comments en entier.
                -->
                <div
                  v-for="(comment, commentIndex) in (visibleComments[`${index}-${msgIndex}`]
                    ? msg.comments
                    : msg.comments.slice(-1))"
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

<style scoped>
:host,
:root {
  display: block;
  padding-top: 2rem;
}

h1 {
  margin-bottom: 2rem;
  letter-spacing: -0.03em;
}

form,
.p-4.mb-6.border.rounded.bg-gray-50 {
  margin-bottom: 2.5rem;
  padding: 1.5rem;
}

form h2,
.p-4.mb-6.border.rounded.bg-gray-50 h2 {
  margin-bottom: 1rem;
}

form input,
form label,
.p-4.mb-6.border.rounded.bg-gray-50 input,
.p-4.mb-6.border.rounded.bg-gray-50 button {
  margin-top: 0.5rem;
}

section {
  display: flex;
  flex-direction: column;
  gap: 2.25rem;
}

article {
  padding: 1.75rem;
  background: linear-gradient(180deg, #ffffff, #fafafa);
  border-radius: 16px;
  border: 1px solid #e5e7eb;
  box-shadow: 0 12px 28px rgba(0, 0, 0, 0.06);
  transition: transform 0.15s ease, box-shadow 0.15s ease;
}

article:hover {
  transform: translateY(-2px);
  box-shadow: 0 20px 42px rgba(0, 0, 0, 0.08);
}

article h2 {
  margin-bottom: 0.5rem;
  font-size: 1.05rem;
}

article p {
  margin-top: 0.25rem;
  font-size: 0.85rem;
}

button {
  margin-top: 0.4rem;
  font-size: 0.72rem;
  padding: 0.3rem 0.65rem;
  border-radius: 6px;
}

article .space-x-2 > * {
  margin-right: 0.4rem;
}

article .border-l-2 {
  margin-top: 1.5rem;
  padding-left: 1.25rem;
}

article .flex.items-center.space-x-2.mt-2 {
  margin-top: 1rem;
}

article .bg-white {
  margin-top: 1.25rem;
  padding: 1rem;
  border-radius: 12px;
}

article .border-l {
  margin-top: 1rem;
  padding-left: 1rem;
}

.bg-gray-50 {
  margin-top: 0.6rem;
  padding: 0.6rem;
  border-radius: 8px;
}

.text-blue-600 {
  margin-top: 0.75rem;
  display: inline-block;
}

.p-4.mb-6.border.rounded.bg-gray-50 {
  margin-top: 2rem;
}

button.bg-green-600,
button.bg-gray-600 {
  margin-bottom: 2rem;
}
</style>