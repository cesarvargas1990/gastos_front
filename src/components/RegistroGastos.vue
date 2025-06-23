<template>
  <div class="max-w-5xl mx-auto p-4">
    <h1 class="text-3xl font-bold text-center text-indigo-600 mb-6">Registro de Gastos</h1>


<!-- RESUMEN FINANCIERO -->
    <div class="grid md:grid-cols-2 gap-4 mb-6">
      <div v-for="(item, key) in resumenFiltrado" :key="key" class="bg-white shadow-md rounded-lg p-4 border border-indigo-100">
       
        <p class="text-gray-700 font-bold">{{ item.valor }}</p>
        <p class="text-sm text-gray-500">{{ item.descripcion }}</p>
      </div>
    </div>


   <div v-if="datosColoreados2025" class="mt-8">
  <h2 class="text-2xl font-semibold text-indigo-700 mb-4 text-center">Resumen por Mes - 2025</h2>
  <div class="overflow-x-auto">
    <table class="min-w-full bg-white rounded-lg shadow-md">
      <thead class="bg-indigo-100 text-indigo-800">
        <tr>
          <th class="px-4 py-2 text-left">Mes</th>
          <th class="px-4 py-2 text-left">Total</th>
          <th class="px-4 py-2 text-left">Disponible</th>
        </tr>
      </thead>
      <tbody>
        <tr
          v-for="(info, mes) in datosColoreados2025"
          :key="mes"
          :class="info.disponible.includes('-') ? 'bg-red-50' : 'bg-green-50'"
          class="border-t"
        >
          <td class="px-4 py-2 capitalize font-medium">{{ mes }}</td>
          <td class="px-4 py-2">{{ info.total }}</td>
          <td class="px-4 py-2">{{ info.disponible }}</td>
        </tr>
      </tbody>
    </table>
  </div>
</div>


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
            <th class="px-4 py-2 text-left cursor-pointer" @click="ordenarPor('descripcion')">
              Descripción {{ orden.campo === 'descripcion' ? (orden.ascendente ? '⬆️' : '⬇️') : '' }}
            </th>
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
const resumen = ref({})
const datosColoreados2025 = ref(null)

const nuevoGasto = ref({ descripcion: '', valor: 0, fecha: '' })
const modalActivo = ref(false)
const indexEditando = ref(null)
const editado = ref({ descripcion: '', valor: 0, fecha: '' })
const orden = ref({ campo: '', ascendente: true })


const resumenFiltrado = computed(() => {
  return Object.entries(resumen.value || {})
    .filter(([_, val]) => val?.valor && val?.descripcion && !val.valor.includes('#REF!'))
    .map(([key, val]) => ({
      clave: typeof key === 'string' ? key.replaceAll('_', ' ') : String(key),
      ...val
    }))
})

const cargarResumen = async () => {
  try {
    const { data } = await axios.get(`${API_BASE}/resumen-financiero`)
    resumen.value = data
  } catch (error) {
    console.error('Error cargando resumen financiero:', error)
  }
}

const cargarDatosColoreados = async () => {
  try {
    const { data } = await axios.get('http://localhost:3000/datos-coloreados')
    datosColoreados2025.value = data['2025'] || {}
  } catch (err) {
    console.error('Error al cargar datos-coloreados:', err)
  }
}



const cargarGastos = async () => {
  const { data } = await axios.get(`${API_BASE}/listado-gastos`)
  gastos.value = data
   await cargarResumen()
  await cargarDatosColoreados()
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
  await cargarResumen()
  await cargarDatosColoreados()
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
   await cargarResumen()
   await cargarDatosColoreados()
}

const eliminar = async (index) => {
  if (confirm('¿Seguro que deseas eliminar este gasto?')) {
    await axios.delete(`${API_BASE}/eliminar-gasto/${index}`)
    await cargarGastos()
     await cargarResumen()
     await cargarDatosColoreados()
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
    let valorA = a[orden.value.campo]
    let valorB = b[orden.value.campo]

    if (orden.value.campo === 'fecha') {
      const parseFecha = (str) => {
        const [dd, mm, yyyy] = str.split('/')
        return new Date(`${yyyy}-${mm}-${dd}`).getTime()
      }
      valorA = parseFecha(valorA)
      valorB = parseFecha(valorB)
    }

    if (orden.value.campo === 'valor') {
      const parseValor = (str) => {
        if (typeof str === 'number') return str
        return Number(str.replace(/[^0-9.-]+/g, '').replace(',', '.')) || 0
      }
      valorA = parseValor(valorA)
      valorB = parseValor(valorB)
    }

    if (orden.value.campo === 'rowIndex') {
      valorA = Number(valorA)
      valorB = Number(valorB)
    }

    if (orden.value.campo === 'descripcion') {
      return orden.value.ascendente
        ? valorA.localeCompare(valorB)
        : valorB.localeCompare(valorA)
    }

    return orden.value.ascendente ? valorA - valorB : valorB - valorA
  })
})



onMounted(() => {
   cargarGastos()
   cargarResumen()
   cargarDatosColoreados()
})
</script>
