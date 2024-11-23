<template>
  <h1>Logged {{ activeEntries.length }} entries</h1>

  <form @submit.prevent="addVehicle">
    <fieldset>
      <legend>Vehicle Type</legend>
      <label v-for="option in vehicleTypes" :key="option">
        <input v-model="vehicleType" :value="option" type="radio" />
        <CarIcon v-if="option === 'car'" />
        <BusSideIcon v-else-if="option === 'bus'" />
        <BikeIcon v-else-if="option === 'cyclist'" />
        <WalkIcon v-else-if="option === 'pedestrian'" />
        {{ capitalize(option) }}
      </label>
    </fieldset>

    <fieldset>
      <legend>Direction</legend>
      <label v-for="option in directions" :key="option">
        <input v-model="direction" :value="option" type="radio" />
        <ArrowLeftIcon v-if="option === 'left'" />
        <ArrowUpIcon v-else-if="option === 'straight'" />
        <ArrowRightIcon v-else-if="option === 'right'" />
        {{ capitalize(option) }}
      </label>
    </fieldset>

    <button type="submit">Add entry</button>
  </form>

  <button class="download" @click="downloadCSVData()">Download CSV</button>
  <button class="download" @click="downloadJSONData()">Download JSON</button>

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
          <button :disabled="entry.removed" class="error" @click="deleteEntry(entry.id)">
            Delete
          </button>
        </td>
      </tr>
    </tbody>
  </table>

  <button class="error" @click="deleteAllEntries">Reset all data</button>
</template>

<script lang="ts" setup>
import { computed, onMounted, ref } from 'vue'
import ArrowUpIcon from '@/assets/ArrowUpIcon.vue'
import ArrowLeftIcon from '@/assets/ArrowLeftIcon.vue'
import ArrowRightIcon from '@/assets/ArrowRightIcon.vue'
import CarIcon from '@/assets/CarIcon.vue'
import BusSideIcon from '@/assets/BusSideIcon.vue'
import WalkIcon from '@/assets/WalkIcon.vue'
import BikeIcon from '@/assets/BikeIcon.vue'

const directions = ['left', 'straight', 'right']
const defaultDirection = 'straight'
const vehicleTypes = ['car', 'bus', 'cyclist', 'pedestrian']
const defaultVehicleType = 'car'

type Entry = {
  id: string
  direction: (typeof directions)[number]
  vehicleType: (typeof vehicleTypes)[number]
  timestamp: string
  removed?: boolean
}

const direction = ref(defaultDirection)
const vehicleType = ref(defaultVehicleType)

const entries = ref<Entry[]>([])
const activeEntries = computed(() => entries.value.filter((entry) => !entry.removed))

function loadEntries() {
  const storedEntries = localStorage.getItem('trafficEntries')
  if (storedEntries) {
    entries.value = JSON.parse(storedEntries)
  }
}

function saveEntries(newEntries: Entry[]) {
  entries.value = newEntries
  localStorage.setItem('trafficEntries', JSON.stringify(newEntries))
}

function addVehicle() {
  if (direction.value && vehicleType.value) {
    const newEntry = {
      id: Date.now().toString(),
      direction: direction.value,
      vehicleType: vehicleType.value,
      timestamp: new Date().toLocaleString(),
    }
    saveEntries([newEntry, ...entries.value])
    direction.value = defaultDirection
    vehicleType.value = defaultVehicleType
  }
}

function updateEntry(id: string, field: keyof Entry, eventTarget: EventTarget | null) {
  if (!eventTarget) {
    return
  }
  const value = (<HTMLInputElement>eventTarget).value

  const updatedEntries = entries.value.map((entry) =>
    entry.id === id ? { ...entry, [field]: value } : entry,
  )
  saveEntries(updatedEntries)
}

function deleteEntry(id: string) {
  const updatedEntries = entries.value.map((entry) =>
    entry.id === id ? { ...entry, removed: true } : entry,
  )
  saveEntries(updatedEntries)
}

function deleteAllEntries() {
  if (window.confirm('Do you really want to DELETE ALL DATA?')) {
    saveEntries([])
  }
}

function capitalize(string: string) {
  return string.charAt(0).toUpperCase() + string.slice(1)
}

function downloadData(data: BlobPart, type: string) {
  const blob = new Blob([data], { type: type })

  const link = document.createElement('a')
  link.href = URL.createObjectURL(blob)
  link.download = 'traffic-log'
  link.click()

  URL.revokeObjectURL(link.href)
}

function downloadJSONData() {
  const data = JSON.stringify(activeEntries.value, undefined, 2)
  downloadData(data, ' application/json')
}

function downloadCSVData() {
  const headers: (keyof Entry)[] = ['id', 'direction', 'vehicleType', 'timestamp']

  // Map each entry to a CSV row
  const rows = activeEntries.value.map((entry) =>
    // Use double quotes to escape comma's
    headers.map((header) => `"${entry[header]}"`).join(','),
  )

  // Combine the headers and rows into a CSV string
  const textCSV = [headers.join(','), ...rows].join('\n')

  downloadData(textCSV, 'text/csv')
}

onMounted(loadEntries)
</script>

<style scoped>
.removed-row {
  text-decoration: line-through;
}

label {
  cursor: pointer;
  display: block;
  font-size: 2rem;

  svg {
    width: 2rem;
    vertical-align: bottom;
  }

  input[type='radio'] {
    display: none; /* Hide default radio button */
  }

  input:checked + svg {
    fill: #0083ef;
  }
}

button {
  cursor: pointer;
  border-radius: 1rem;
  background-color: transparent;

  &[type='submit'] {
    color: #0083ef;
    border: 2px solid #0083ef;
    font-size: 1.5rem;
    padding: 0.5rem;
    margin: 1rem 1rem 1rem 0;
  }

  &.download {
    padding: 0.5rem 1rem;
    margin: 1rem 1rem 1rem 0;
  }

  &.error {
    border-radius: 5px;
    color: #ef000c;
    border-color: #ef000c;

    &:disabled {
      color: darkgrey;
      border-color: darkgrey;
      text-decoration: line-through;
    }
  }
}
</style>
