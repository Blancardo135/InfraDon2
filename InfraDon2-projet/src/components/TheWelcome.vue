<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'

// Définition du type de données
interface Character {
  name: string;
  age: number;
  affiliation: string;
  lightsaber: boolean;
}

// Référence à la base de données
const storage = ref<any>(null)
const characters = ref<Character[]>([])

// Initialisation de la base de données
const initDatabase = () => {
  console.log('=> Connexion à la base de données');
  const db = new PouchDB('http://RomainBlanchard:admin@localhost:5984/infradon-blaro')
  storage.value = db
  console.log("Connecté à la collection :", db?.name)
}
console.log("Hey")
// Récupération des données
const fetchData = async () => {
  if (!storage.value) return

  try {
    // Récupère tous les documents de la base
    const result = await storage.value.allDocs({ include_docs: true })
    console.log('=> Données récupérées :', result)

    // On parcourt chaque document et on extrait le tableau "characters"
    const allCharacters: Character[] = result.rows.flatMap((row: any) => row.doc.characters || [])
    characters.value = allCharacters

  } catch (err) {
    console.error('Erreur lors de la récupération :', err)
  }
}

onMounted(() => {
  initDatabase()
  fetchData()
})
</script>

<template>
  <h1>Liste des personnages</h1>

  <article v-for="(char, index) in characters" :key="index" class="p-4 border-b">
    <h2>{{ char.name }}</h2>
    <p>Âge : {{ char.age }}</p>
    <p>Affiliation : {{ char.affiliation }}</p>
    <p>Sabre laser : {{ char.lightsaber ? 'Oui' : 'Non' }}</p>
  </article>
</template>



