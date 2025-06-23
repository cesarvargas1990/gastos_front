<template>
  <div class="max-w-5xl mx-auto p-4">
    <h1 class="text-3xl font-bold text-center text-indigo-600 mb-6">Registro de Gastos</h1>

    <form @submit.prevent="agregarGasto" class="bg-white shadow-md rounded-lg p-4 grid gap-4 md:grid-cols-3 sm:grid-cols-1">
      <input v-model="nuevoGasto.descripcion" type="text" placeholder="Descripción" required class="border rounded px-3 py-2 w-full" />
      <input v-model.number="nuevoGasto.valor" type="number" placeholder="Valor" required class="border rounded px-3 py-2 w-full" />
      <input v-model="nuevoGasto.fecha" type="date" required class="border rounded px-3 py-2 w-full" />
      <button type="submit" class="md:col-span-3 bg-indigo-600 text-white py-2 px-4 rounded hover:bg-indigo-700 transition">Agregar Gasto</button>
    </form>

    <div class="overflow-x-auto mt-6">
      <table class="min-w-full bg-white rounded-lg shadow-md">
        <thead class="bg-indigo-100 text-indigo-800">
          <tr>
            <th class="px-4 py-2 text-left cursor-pointer" @click="ordenarPor('rowIndex')">
            # {{ orden.campo === 'rowIndex' ? (orden.ascendente ? '⬆️' : '⬇️') : '' }}
            </th>
            <th class="px-4 py-2 text-left">Descripción</th>
            <th class="px-4 py-2 text-left cursor-pointer" @click="ordenarPor('valor')">
              Valor {{ orden.campo === 'valor' ? (orden.ascendente ? '⬆️' : '⬇️') : '' }}
            </th>
            <th class="px-4 py-2 text-left cursor-pointer" @click="ordenarPor('fecha')">
              Fecha {{ orden.campo === 'fecha' ? (orden.ascendente ? '⬆️' : '⬇️') : '' }}
            </th>
            <th class="px-4 py-2">Acciones</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(gasto, index) in gastosOrdenados" :key="index" class="border-t">
            <td class="px-4 py-2">{{ gasto.rowIndex }}</td>
            <td class="px-4 py-2">{{ gasto.descripcion }}</td>
            <td class="px-4 py-2">{{ gasto.valor}}</td>
            <td class="px-4 py-2">{{ gasto.fecha }}</td>
            <td class="px-4 py-2 text-center space-x-2">
              <button @click="abrirModalEditar(gasto)" class="text-yellow-600 hover:underline">Editar</button>
              <button @click="eliminar(gasto.rowIndex)" class="text-red-600 hover:underline">Eliminar</button>
            </td>
          </tr>
        </tbody>
      </table>
    </div>

    <!-- Modal centrado -->
    <div v-if="modalActivo" class="fixed inset-0 bg-black bg-opacity-40 flex items-center justify-center z-50">
      <div class="bg-white p-6 rounded-lg shadow-lg w-full max-w-md">
        <h2 class="text-xl font-semibold mb-4">Editar Gasto</h2>
        <input v-model="editado.descripcion" type="text" placeholder="Descripción" class="border rounded px-3 py-2 w-full mb-2" />
        <input v-model.number="editado.valor"  type="number" placeholder="Valor" class="border rounded px-3 py-2 w-full mb-2" />
        <input v-model="editado.fecha" type="date" class="border rounded px-3 py-2 w-full mb-4" />
        <div class="flex justify-end space-x-2">
          <button @click="guardarEdicion" class="bg-indigo-600 text-white px-4 py-2 rounded hover:bg-indigo-700">Guardar</button>
          <button @click="cerrarModal" class="bg-gray-300 px-4 py-2 rounded hover:bg-gray-400">Cancelar</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'
import axios from 'axios'

const API_BASE = 'http://localhost:3000'

const gastos = ref([])
const nuevoGasto = ref({ descripcion: '', valor: 0, fecha: '' })
const modalActivo = ref(false)
const indexEditando = ref(null)
const editado = ref({ descripcion: '', valor: 0, fecha: '' })
const orden = ref({ campo: '', ascendente: true })

const cargarGastos = async () => {
  const { data } = await axios.get(`${API_BASE}/listado-gastos`)
  gastos.value = data
}

const formatearParaGuardar = (fechaISO) => {
  const date = new Date(fechaISO)
  const dd = String(date.getDate()).padStart(2, '0')
  const mm = String(date.getMonth() + 1).padStart(2, '0')
  const yyyy = date.getFullYear()
  return `${dd}/${mm}/${yyyy}`
}

const agregarGasto = async () => {
  const payload = {
    descripcion: nuevoGasto.value.descripcion,
    valor: nuevoGasto.value.valor,
    fecha: formatearParaGuardar(nuevoGasto.value.fecha)
  }
  await axios.post(`${API_BASE}/agregar-gasto`, payload)
  nuevoGasto.value = { descripcion: '', valor: 0, fecha: '' }
  await cargarGastos()
}

const convertirADateInputCompatible = (fechaStr) => {
  const [dd, mm, yyyy] = fechaStr.split('/')
  return `${yyyy}-${mm.padStart(2, '0')}-${dd.padStart(2, '0')}`
}


const abrirModalEditar = (gastoe) => {
//alert(JSON.stringify(gastoe))
//alert(gastoe.valor.trim().slice(1))
//alert (index)
  indexEditando.value = gastoe.rowIndex
  //const gasto = gastos.value[index]
  //alert (gastosOrdenados.value[index])
  editado.value = {
    descripcion: gastoe.descripcion,
    valor: gastoe.valor.trim().slice(1).replace(',', '').replace('.', ''),
    fecha: convertirADateInputCompatible(gastoe.fecha)
  }
  modalActivo.value = true
}

const cerrarModal = () => {
  modalActivo.value = false
  indexEditando.value = null
}

const guardarEdicion = async () => {
  const index = indexEditando.value
  const payload = {
    descripcion: editado.value.descripcion,
    valor: editado.value.valor,
    fecha: editado.value.fecha
  }
  await axios.put(`${API_BASE}/editar-gasto/${index}`, payload)
  cerrarModal()
  await cargarGastos()
}

const eliminar = async (index) => {
  if (confirm('¿Seguro que deseas eliminar este gasto?')) {
    await axios.delete(`${API_BASE}/eliminar-gasto/${index}`)
    await cargarGastos()
  }
}

const formatearFecha = (timestamp) => {
  const fecha = new Date(timestamp)
  const dia = String(fecha.getDate()).padStart(2, '0')
  const mes = String(fecha.getMonth() + 1).padStart(2, '0')
  const anio = fecha.getFullYear()
  return `${dia}/${mes}/${anio}`
}

const ordenarPor = (campo) => {
  if (orden.value.campo === campo) {
    orden.value.ascendente = !orden.value.ascendente
  } else {
    orden.value.campo = campo
    orden.value.ascendente = true
  }
}

const gastosOrdenados = computed(() => {
  const copia = [...gastos.value]
  if (!orden.value.campo) return copia
  return copia.sort((a, b) => {
    const valorA = a[orden.value.campo]
    const valorB = b[orden.value.campo]
    return orden.value.ascendente ? valorA - valorB : valorB - valorA
  })
})

onMounted(cargarGastos)
</script>
