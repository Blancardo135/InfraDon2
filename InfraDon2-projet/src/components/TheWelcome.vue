<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'
import PouchDBFind from 'pouchdb-find'
PouchDB.plugin(PouchDBFind)


interface Character {
  name: string
  age: number
  affiliation: string
  lightsaber: boolean
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
  lightsaber: false
})


const editingIndex = ref<number | null>(null)
const editingCharacter = ref<Character | null>(null)


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

onMounted(() => {
  initDatabase()
  if (isOnline.value) startSync()
})



const createIndex = async () => {
  if (!storage.value) return
  try {
    await storage.value.createIndex({
      index: { fields: ['characters.affiliation'] }
    })
    console.log('Index créé sur characters.affiliation')
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
      characters: [newCharacter.value]
    }

    const response = await storage.value.put(newDoc)
    console.log('Personnage ajouté avec succès')

    characters.value.push({
      data: { ...newCharacter.value },
      docId: newDoc._id,
      docRev: response.rev
    })

    newCharacter.value = {
      name: '',
      age: 0,
      affiliation: '',
      lightsaber: false
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

//
const searchAffiliation = ref('')

const searchByAffiliation = () => {
  if (!searchAffiliation.value) {
    fetchData()
    return
  }

  const filtered = characters.value.filter(
    (char) => char.data.affiliation.toLowerCase() === searchAffiliation.value.toLowerCase()
  )
  characters.value = filtered
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