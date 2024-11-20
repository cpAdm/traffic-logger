<template>
  <form @submit.prevent="handleSubmit">
    <fieldset>
      <legend>Vehicle Type</legend>
      <div>
        <div v-for="option in vehicleTypes" :key="option">
          <label>
            <input v-model="vehicleType" :value="option" type="radio" />
            {{ capitalize(option) }}
          </label>
        </div>
      </div>
    </fieldset>

    <fieldset>
      <legend>Direction</legend>
      <div v-for="option in directions" :key="option">
        <label>
          <input v-model="direction" :value="option" type="radio" />
          {{ capitalize(option) }}
        </label>
      </div>
    </fieldset>

    <button type="submit">Add</button>
  </form>

  <table>
    <thead>
      <tr>
        <th>Vehicle Type</th>
        <th>Direction</th>
        <th>Timestamp</th>
        <th>Actions</th>
      </tr>
    </thead>
    <tbody>
      <tr v-for="entry in entries" :key="entry.id" :class="{ 'removed-row': entry.removed }">
        <td>
          <select
            v-model="entry.vehicleType"
            :disabled="entry.removed"
            @change="updateEntry(entry.id, 'vehicleType', $event.target)"
          >
            <option v-for="option in vehicleTypes" :key="option" :value="option">
              {{ capitalize(option) }}
            </option>
          </select>
        </td>

        <td>
          <select
            v-model="entry.direction"
            :disabled="entry.removed"
            @change="updateEntry(entry.id, 'direction', $event.target)"
          >
            <option v-for="option in directions" :key="option" :value="option">
              {{ capitalize(option) }}
            </option>
          </select>
        </td>

        <td>{{ entry.timestamp }}</td>

        <td>
          <button :disabled="entry.removed" @click="deleteEntry(entry.id)">Delete</button>
        </td>
      </tr>
    </tbody>
  </table>

  <h2>Data as CSV</h2>
  <pre><code>{{ textCSV }}</code></pre>

  <h2>Data as JSON</h2>
  <pre><code>{{
      JSON.stringify(
        entries.filter((entry) => !entry.removed),
        undefined,
        2
      )
    }}</code></pre>

  <button @click="deleteAllEntries">Reset</button>
</template>

<script lang="ts" setup>
import { computed, onMounted, ref } from 'vue'

const directions = ['left', 'right', 'straight']
const vehicleTypes = ['car', 'bus', 'cyclist', 'pedestrian']

type Entry = {
  id: string
  direction: (typeof directions)[number]
  vehicleType: (typeof vehicleTypes)[number]
  timestamp: string
  removed?: boolean
}

const direction = ref('')
const vehicleType = ref('')

const entries = ref<Entry[]>([])

const loadEntries = () => {
  const storedEntries = localStorage.getItem('trafficEntries')
  if (storedEntries) {
    entries.value = JSON.parse(storedEntries)
  }
}

const saveEntries = (newEntries: Entry[]) => {
  entries.value = newEntries
  localStorage.setItem('trafficEntries', JSON.stringify(newEntries))
}

const handleSubmit = () => {
  if (direction.value && vehicleType.value) {
    const newEntry = {
      id: Date.now().toString(),
      direction: direction.value,
      vehicleType: vehicleType.value,
      timestamp: new Date().toLocaleString(),
    }
    saveEntries([newEntry, ...entries.value])
    direction.value = 'right'
    vehicleType.value = 'car'
  }
}

const updateEntry = (id: string, field: keyof Entry, eventTarget: EventTarget | null) => {
  if (!eventTarget) {
    return
  }
  const value = (<HTMLInputElement>eventTarget).value

  const updatedEntries = entries.value.map((entry) =>
    entry.id === id ? { ...entry, [field]: value } : entry,
  )
  saveEntries(updatedEntries)
}

const deleteEntry = (id: string) => {
  const updatedEntries = entries.value.map((entry) =>
    entry.id === id ? { ...entry, removed: true } : entry,
  )
  saveEntries(updatedEntries)
}

const deleteAllEntries = () => {
  if (window.confirm('Do you really want to DELETE ALL DATA?')) {
    saveEntries([])
  }
}

const capitalize = (string: string) => string.charAt(0).toUpperCase() + string.slice(1)

const textCSV = computed(() => {
  const headers: (keyof Entry)[] = ['id', 'direction', 'vehicleType', 'timestamp']

  // Map each entry to a CSV row
  const rows = entries.value
    .filter((entry) => !entry.removed)
    .map((entry) =>
      headers
        .map((header) => {
          // Use double quotes to escape comma's
          return `"${entry[header]}"`
        })
        .join(','),
    )

  // Combine the headers and rows into a CSV string
  return [headers.join(','), ...rows].join('\n')
})

onMounted(loadEntries)
</script>

<style scoped>
.removed-row {
  text-decoration: line-through;
}

label {
  font-size: 2rem;
}
</style>
