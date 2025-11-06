<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'

// Définition du type de données
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

// Références à la base et aux données
const storage = ref<any>(null)
const characters = ref<{ data: Character; docId: string; docRev: string }[]>([])

// Données du nouveau personnage à ajouter
const newCharacter = ref<Character>({
  name: '',
  age: 0,
  affiliation: '',
  lightsaber: false
})

// État pour l'édition
const editingIndex = ref<number | null>(null)
const editingCharacter = ref<Character | null>(null)

// --- Initialisation de la base de données ---
const initDatabase = () => {
  console.log('=> Connexion à la base de données')
  const localDB = new PouchDB('infradon-blaro')
  storage.value = localDB
  console.log('Base locale :', localDB?.name);

  localDB.replicate.from('http://RomainBlanchard:admin@localhost:5984/infradon-blaro', { 
    live: true, 
    retry: true 
  }).on('change', (info) => {
    console.log('Changements reçus du serveur')
    fetchData()
  }).on('complete', () => {
    console.log('Réplication initiale terminée')
    fetchData()
  }).on('error', (err) => {
    console.error('Erreur réplication pull :', err)
  })
  
   localDB.replicate.to('http://RomainBlanchard:admin@localhost:5984/infradon-blaro', { 
    live: true, 
    retry: true 
  }).on('change', (info) => {
    console.log('Changements envoyés au serveur')
  }).on('complete', () => {
    console.log('Réplication inverse terminée')
  }).on('error', (err) => {
    console.error('Erreur réplication push :', err)
  })
}

// --- Récupération des données existantes ---
const fetchData = async () => {
  if (!storage.value) return

  try {
    const result = await storage.value.allDocs({ include_docs: true })
    console.log('=> Données récupérées :', result)

    // Extraire tous les personnages avec leur référence de document
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

// --- Ajout d'un nouveau personnage/document ---
const addCharacter = async () => {
  if (!storage.value) return

  try {
    // Créer un nouveau document avec le personnage
    const newDoc = {
      _id: new Date().toISOString(), // id unique basé sur le timestamp
      characters: [newCharacter.value]
    }

    // Enregistrer dans la base CouchDB
    const response = await storage.value.put(newDoc)
    console.log('✅ Personnage ajouté avec succès')

    // Ajouter directement dans la liste locale
    characters.value.push({
      data: { ...newCharacter.value },
      docId: newDoc._id,
      docRev: response.rev
    })

    // Réinitialiser le formulaire
    newCharacter.value = {
      name: '',
      age: 0,
      affiliation: '',
      lightsaber: false
    }
  } catch (err) {
    console.error('❌ Erreur lors de l\'ajout :', err)
  }
}

// --- Activer le mode édition ---
const startEdit = (index: number) => {
  editingIndex.value = index
  editingCharacter.value = { ...characters.value[index].data }
}

// --- Annuler l'édition ---
const cancelEdit = () => {
  editingIndex.value = null
  editingCharacter.value = null
}

// --- Sauvegarder les modifications ---
const saveEdit = async (index: number) => {
  if (!storage.value || !editingCharacter.value) return

  try {
    const charToUpdate = characters.value[index]
    
    // Récupérer le document complet
    const doc: CharacterDoc = await storage.value.get(charToUpdate.docId)
    
    // Comme nous stockons un personnage par document, on met à jour le premier élément
    doc.characters[0] = editingCharacter.value
    
    // Sauvegarder dans CouchDB
    const response = await storage.value.put(doc)
    console.log('✅ Personnage modifié avec succès')
    
    // Mettre à jour localement
    characters.value[index] = {
      data: { ...editingCharacter.value },
      docId: charToUpdate.docId,
      docRev: response.rev
    }
    
    // Sortir du mode édition
    cancelEdit()
  } catch (err) {
    console.error('❌ Erreur lors de la modification :', err)
  }
}

// --- Supprimer un personnage ---
const deleteCharacter = async (index: number) => {
  if (!storage.value) return
  
  const confirmDelete = confirm('Êtes-vous sûr de vouloir supprimer ce personnage ?')
  if (!confirmDelete) return

  try {
    const charToDelete = characters.value[index]
    
    // Récupérer le document pour avoir la dernière révision
    const doc = await storage.value.get(charToDelete.docId)
    
    // Supprimer le document de CouchDB
    await storage.value.remove(doc)
    console.log('✅ Personnage supprimé avec succès')
    
    // Supprimer localement
    characters.value.splice(index, 1)
  } catch (err) {
    console.error('❌ Erreur lors de la suppression :', err)
  }
}

// --- Cycle de vie Vue ---
onMounted(() => {
  initDatabase()
  fetchData()
})
</script>

<template>
  <h1 class="text-2xl font-bold mb-4">Liste des personnages</h1>

  <!-- 🔹 Formulaire d'ajout -->
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

  <!-- 🔹 Liste des personnages -->
  <section>
    <article
      v-for="(char, index) in characters"
      :key="char.docId"
      class="p-4 border-b hover:bg-gray-100 transition"
    >
      <!-- Mode lecture -->
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

      <!-- Mode édition -->
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