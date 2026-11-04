<script setup>
import { ref, computed, onMounted } from 'vue'

const todosLosPokemon = ref([])
const resultados = ref(null) // null = mostrar lista completa
const busqueda = ref('')
const tipoSeleccionado = ref('')
const mensaje = ref('')
const error = ref('')
const cargando = ref(false)

const listaMostrada = computed(() => {
  const base = resultados.value ?? todosLosPokemon.value
  if (!tipoSeleccionado.value) return base
  return base.filter(p =>
    p.types.some(t => t.type.name === tipoSeleccionado.value)
  )
})

const contador = computed(() => {
  if (cargando.value) return 'Cargando...'
  if (resultados.value) return `${listaMostrada.value.length} resultado(s)`
  if (tipoSeleccionado.value)
    return `${listaMostrada.value.length} Pokémon de tipo ${tipoSeleccionado.value}`
  return `${todosLosPokemon.value.length} / 151`
})

const tipos = [
  ['normal', 'Normal'], ['fire', 'Fuego'], ['water', 'Agua'],
  ['electric', 'Eléctrico'], ['grass', 'Planta'], ['ice', 'Hielo'],
  ['fighting', 'Lucha'], ['poison', 'Veneno'], ['ground', 'Tierra'],
  ['flying', 'Volador'], ['psychic', 'Psíquico'], ['bug', 'Bicho'],
  ['rock', 'Roca'], ['ghost', 'Fantasma'], ['dragon', 'Dragón'],
  ['dark', 'Siniestro'], ['steel', 'Acero'], ['fairy', 'Hada']
]

function stat(p, nombre) {
  return p.stats.find(s => s.stat.name === nombre)?.base_stat ?? 0
}

function imagen(p) {
  return p.sprites.other['official-artwork'].front_default || p.sprites.front_default
}

async function cargarPokemonIniciales() {
  cargando.value = true
  error.value = ''
  mensaje.value = 'Cargando Pokédex completa...'
  resultados.value = null
  tipoSeleccionado.value = ''
  busqueda.value = ''

  try {
    const resBase = await fetch('https://pokeapi.co/api/v2/pokemon?limit=151')
    const datosBase = await resBase.json()
    // Promise.all hace las 151 peticiones en paralelo (mucho más rápido)
    todosLosPokemon.value = await Promise.all(
      datosBase.results.map(r => fetch(r.url).then(res => res.json()))
    )
    mensaje.value = ''
  } catch (e) {
    error.value = 'Error al cargar la lista completa de Pokémon.'
    mensaje.value = ''
  } finally {
    cargando.value = false
  }
}

async function buscarPokemon() {
  const nombre = busqueda.value.toLowerCase().trim()
  if (!nombre) {
    alert('Por favor escribe el nombre de un Pokémon.')
    return
  }
  error.value = ''
  mensaje.value = 'Buscando en la PokeAPI...'
  tipoSeleccionado.value = ''

  try {
    const res = await fetch('https://pokeapi.co/api/v2/pokemon/' + nombre)
    if (!res.ok) {
      error.value = `No se encontró el Pokémon "${nombre}".`
      resultados.value = []
    } else {
      resultados.value = [await res.json()]
    }
  } catch (e) {
    error.value = 'Error al conectarse a la API.'
  } finally {
    mensaje.value = ''
  }
}

async function cargarAleatorio() {
  error.value = ''
  mensaje.value = 'Cargando aleatorio...'
  tipoSeleccionado.value = ''
  const numero = Math.floor(Math.random() * 1010) + 1

  try {
    const res = await fetch('https://pokeapi.co/api/v2/pokemon/' + numero)
    const datos = await res.json()
    resultados.value = [datos]
    busqueda.value = datos.name
  } catch (e) {
    error.value = 'Error al obtener Pokémon aleatorio.'
  } finally {
    mensaje.value = ''
  }
}

onMounted(cargarPokemonIniciales)
</script>

<template>
  <div class="app-container">
    <div class="hero-banner">
      <div class="hero-content">
        <h1 class="pokemon-logo">Pokémon</h1>
      </div>
    </div>

    <div class="main-layout">
      <aside class="sidebar">
        <div class="user-profile">
          <div class="user">
          <img src="https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcQosM3prtK4UTYRUANfeEHS9hRQFfql4Qj-lEDyWnNkBw&s=10" width="100" height="100" alt="Entrenador" />          </div>
          <div class="user-info">
            <span class="user-name">ENTRENADOR</span>
            <span class="user-level">Level 1</span>
          </div>
        </div>

        <div class="sidebar-divider"></div>

        <form class="sidebar-form" @submit.prevent>
          <div class="form-group">
            <label for="searchInput">Buscar Pokémon:</label>
            <input
              type="text"
              id="searchInput"
              v-model="busqueda"
              placeholder="Ej: pikachu, charizard..."
              @keyup.enter="buscarPokemon"
            />
            <input type="button" value="Buscar" @click="buscarPokemon" />
          </div>

          <div class="form-group">
            <label for="typeFilter">Filtrar por tipo:</label>
            <select id="typeFilter" v-model="tipoSeleccionado">
              <option value="">Todos los tipos</option>
              <option v-for="[valor, nombre] in tipos" :key="valor" :value="valor">
                {{ nombre }}
              </option>
            </select>
          </div>

          <div class="form-group">
            <input type="button" value="🎲 Aleatorio" class="btn-mint" @click="cargarAleatorio" />
            <input type="button" value="🔄 Restablecer Lista" class="btn-alt" @click="cargarPokemonIniciales" />
          </div>
        </form>
      </aside>

      <main class="content-area">
        <div class="content-header">
          <h2>POKÉMON</h2>
          <span class="pokemon-count">{{ contador }}</span>
        </div>

        <div v-if="mensaje" class="mensaje">{{ mensaje }}</div>
        <div v-else-if="error" class="error">{{ error }}</div>
        <div v-else-if="listaMostrada.length === 0" class="mensaje">
          No se encontraron resultados.
        </div>

        <div v-else class="pokemon-grid">
          <div v-for="p in listaMostrada" :key="p.id" class="card-pokemon">
            <div class="cp-label">{{ stat(p, 'hp') }} CP</div>
            <img class="pokemon-img" :src="imagen(p)" :alt="p.name" />
            <div class="pokemon-name">{{ p.name }}</div>
            <div class="types-container">
              <span
                v-for="t in p.types"
                :key="t.type.name"
                class="type-badge"
                :class="'type-' + t.type.name"
              >
                {{ t.type.name }}
              </span>
            </div>
            <div class="stats-box">
              <div><span>Ataque</span> <strong>{{ stat(p, 'attack') }}</strong></div>
              <div><span>Defensa</span> <strong>{{ stat(p, 'defense') }}</strong></div>
            </div>
          </div>
        </div>
      </main>
    </div>
  </div>
</template>