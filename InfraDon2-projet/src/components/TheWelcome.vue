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

// Références à la base et aux données
const storage = ref<any>(null)
const characters = ref<Character[]>([])

// Données du nouveau personnage à ajouter
const newCharacter = ref<Character>({
  name: '',
  age: 0,
  affiliation: '',
  lightsaber: false
})

// --- Initialisation de la base de données ---
const initDatabase = () => {
  console.log('=> Connexion à la base de données')
  const db = new PouchDB('http://RomainBlanchard:admin@localhost:5984/infradon-blaro')
  storage.value = db
  console.log('Connecté à la collection :', db?.name)
}

// --- Récupération des données existantes ---
const fetchData = async () => {
  if (!storage.value) return

  try {
    const result = await storage.value.allDocs({ include_docs: true })
    console.log('=> Données récupérées :', result)

    // Extraire tous les personnages de tous les documents
    const allCharacters: Character[] = result.rows.flatMap(
      (row: any) => row.doc.characters || []
    )
    characters.value = allCharacters
  } catch (err) {
    console.error('Erreur lors de la récupération :', err)
  }
}

// --- Ajout d'un nouveau personnage ---
const addCharacter = async () => {
  if (!storage.value) return

  try {
    // Créer un nouveau document avec le personnage
    const newDoc = {
      _id: new Date().toISOString(), // id unique basé sur le timestamp
      characters: [newCharacter.value]
    }

    // Enregistrer dans la base CouchDB
    await storage.value.put(newDoc)
    console.log('✅ Personnage ajouté avec succès')

    // Ajouter directement dans la liste locale (sans recharger)
    characters.value.push({ ...newCharacter.value })

    // Réinitialiser le formulaire
    newCharacter.value = {
      name: '',
      age: 0,
      affiliation: '',
      lightsaber: false
    }
  } catch (err) {
    console.error('❌ Erreur lors de l’ajout :', err)
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

  <!-- 🔹 Formulaire d’ajout -->
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
      :key="index"
      class="p-4 border-b hover:bg-gray-100 transition"
    >
      <h2 class="font-semibold text-lg">{{ char.name }}</h2>
      <p>Âge : {{ char.age }}</p>
      <p>Affiliation : {{ char.affiliation }}</p>
      <p>Sabre laser : {{ char.lightsaber ? 'Oui' : 'Non' }}</p>
    </article>
  </section>
</template>
