<template>
    <v-container>
        <v-row>
            <v-col cols="12" md="6">
                <v-card class="pa-4" elevation="2" title="Buscar Artículos">
                    <v-text-field v-model="busqueda" append-inner-icon="mdi-magnify"
                        label="Buscar por nombre o código..." variant="outlined" density="comfortable"
                        @keyup.enter="buscarArticulos" @click:append-inner="buscarArticulos"
                        :loading="loading"></v-text-field>

                    <v-list v-if="articulosObtenidos.length > 0" lines="two">
                        <v-list-item v-for="item in articulosObtenidos" :key="item.id"
                            :title="item.nombre || 'Sin nombre'">
                            <template v-slot:subtitle>
                                Stock: {{ item.stock || 0 }} | Precio: Q{{ formatPrecio(item.precioVenta) }}
                            </template>
                            <template v-slot:append>
                                <v-btn size="small" color="primary" variant="tonal"
                                    :disabled="!item.stock || item.stock <= 0" @click="agregarAlCarrito(item)">
                                    Agregar
                                </v-btn>
                            </template>
                        </v-list-item>
                    </v-list>


                    <template>
                        <select name="cliente_di" v-model="clienteSeleccionado" id="">
                            <option v-for="cliente in clientesDB" :value="cliente.id">cliente.nombre</option>
                            <!-- <option value="1">Consumidor Final</option>
                            <option value="2">Ariel </option>
                            <option value="3">Angelo</option> -->
                        </select>
                    </template>

                    <v-alert v-if="busqueda && articulosObtenidos.length === 0 && busquedaRealizada && !loading"
                        type="info" variant="tonal" text="No se encontraron artículos" class="mt-2"></v-alert>
                </v-card>
            </v-col>

            <v-col cols="12" md="6">
                <v-card class="pa-4" elevation="2" title="Detalle de Venta">
                    <v-table density="compact" v-if="carrito.length > 0">
                        <thead>
                            <tr>
                                <th>Artículo</th>
                                <th width="100px">Cant.</th>
                                <th>Precio</th>
                                <th>Total</th>
                                <th></th>
                            </tr>
                        </thead>
                        <tbody>
                            <tr v-for="(prod, index) in carrito" :key="index">
                                <td>{{ prod.nombre || 'Sin nombre' }}</td>
                                <td>
                                    <v-text-field v-model.number="prod.cantidad" type="number" min="1"
                                        :max="prod.stock || 999" density="compact" variant="underlined" hide-details
                                        @update:model-value="recalcularTotalFila(prod)"></v-text-field>
                                </td>
                                <td>Q{{ formatPrecio(prod.precioVenta) }}</td>
                                <td>Q{{ formatPrecio(prod.total) }}</td>
                                <td>
                                    <v-btn icon="mdi-delete" color="error" variant="text" size="small"
                                        @click="eliminarDelCarrito(index)"></v-btn>
                                </td>
                            </tr>
                        </tbody>
                    </v-table>

                    <v-alert v-else type="info" variant="tonal" text="No hay productos en el carrito"></v-alert>

                    <v-divider class="my-4"></v-divider>

                    <div class="d-flex justify-space-between align-center">
                        <span class="text-h6 font-weight-bold">Total: Q{{ formatPrecio(totalVenta) }}</span>
                        <v-btn color="success" size="large" :disabled="carrito.length === 0" :loading="procesando"
                            @click="procesarVenta">
                            Cobrar
                        </v-btn>
                    </div>
                </v-card>
            </v-col>
        </v-row>

        <v-snackbar v-model="snackbar.show" :color="snackbar.color" :timeout="3000">
            {{ snackbar.message }}
        </v-snackbar>
    </v-container>
</template>

<script setup>
import { ref, computed } from 'vue'
import axios from 'axios'

const API_URL = 'http://localhost:3000'


const busqueda = ref('')
const articulosObtenidos = ref([])
const carrito = ref([])
const loading = ref(false)
const procesando = ref(false)
const busquedaRealizada = ref(false)
const clienteSeleccionado = ref(1)

const snackbar = ref({
    show: false,
    message: '',
    color: 'success'
})

// Función helper para formatear precios de manera segura
const formatPrecio = (precio) => {
    if (precio === undefined || precio === null || precio === '') return '0.00'
    const num = Number(precio)
    return isNaN(num) ? '0.00' : num.toFixed(2)
}

const mostrarMensaje = (message, color = 'success') => {
    snackbar.value = { show: true, message, color }
}

const buscarArticulos = async () => {
    if (!busqueda.value.trim()) {
        mostrarMensaje('Ingresa un término de búsqueda', 'warning')
        return
    }

    loading.value = true
    busquedaRealizada.value = true

    try {
        const response = await axios.get(`${API_URL}/articulos/search`, {
            params: { q: busqueda.value }
        })

        console.log('Respuesta del backend:', response.data)

        if (Array.isArray(response.data)) {
            // Asegurar que los campos numéricos sean realmente números
            articulosObtenidos.value = response.data.map(item => ({
                ...item,
                id: item.id || item.codigo,
                stock: parseInt(item.stock) || 0,
                precioVenta: parseFloat(item.precioVenta) || 0,
                precioCosto: parseFloat(item.precioCosto) || 0
            }))

            console.log('Artículos procesados:', articulosObtenidos.value)

            if (articulosObtenidos.value.length === 0) {
                mostrarMensaje('No se encontraron artículos', 'info')
            }
        } else {
            console.error('La respuesta no es un array:', response.data)
            articulosObtenidos.value = []
            mostrarMensaje('Formato de respuesta inesperado', 'error')
        }
    } catch (error) {
        console.error("Error al buscar artículos:", error)
        mostrarMensaje('Error al buscar artículos', 'error')
        articulosObtenidos.value = []
    } finally {
        loading.value = false
    }
}

const agregarAlCarrito = (item) => {
    const itemId = item.id || item.codigo

    // Asegurar que los valores numéricos sean números
    const precioVenta = parseFloat(item.precio_venta) || 0
    const precioCosto = parseFloat(item.precio_costo) || 0
    const stock = parseInt(item.stock) || 0

    const productoExistente = carrito.value.find(p => p.articuloId === itemId)

    if (productoExistente) {
        if (productoExistente.cantidad < stock) {
            productoExistente.cantidad++
            recalcularTotalFila(productoExistente)
            mostrarMensaje(`Se agregó otra unidad de ${item.nombre}`, 'success')
        } else {
            mostrarMensaje(`Stock máximo alcanzado para ${item.nombre}`, 'warning')
        }
    } else {
        carrito.value.push({
            articuloId: itemId,
            nombre: item.nombre || 'Sin nombre',
            stock: stock,
            cantidad: 1,
            precioCosto: precioCosto,
            precioVenta: precioVenta,
            descuento: 0,
            total: precioVenta // Inicialmente 1 * precioVenta
        })
        mostrarMensaje(`${item.nombre} agregado al carrito`, 'success')
    }

    console.log('Carrito actual:', carrito.value)
}

const recalcularTotalFila = (prod) => {
    // Asegurar que cantidad sea un número válido
    prod.cantidad = parseInt(prod.cantidad) || 1

    if (prod.cantidad > prod.stock) prod.cantidad = prod.stock
    if (prod.cantidad < 1) prod.cantidad = 1

    prod.total = (prod.cantidad * parseFloat(prod.precioVenta)) - parseFloat(prod.descuento || 0)
}

const eliminarDelCarrito = (index) => {
    const producto = carrito.value[index]
    carrito.value.splice(index, 1)
    mostrarMensaje(`${producto.nombre || 'Producto'} eliminado del carrito`, 'info')
}

const totalVenta = computed(() => {
    return carrito.value.reduce((total, prod) => {
        return total + (parseFloat(prod.total) || 0)
    }, 0)
})

const procesarVenta = async () => {
    if (carrito.value.length === 0) {
        mostrarMensaje('El carrito está vacío', 'warning')
        return
    }

    procesando.value = true

    try {
        const payload = {
            productos: carrito.value.map(p => ({
                cantidad: parseInt(p.cantidad) || 1,
                precioCosto: parseFloat(p.precioCosto) || 0,
                precioVenta: parseFloat(p.precioVenta) || 0,
                descuento: parseFloat(p.descuento) || 0,
                total: parseFloat(p.total) || 0,
                articuloId: p.articuloId,
                clienteId: clienteSeleccionado.value
            }))
        }

        console.log('Enviando venta a:', `${API_URL}/ventas/store`)
        console.log('Payload:', JSON.stringify(payload, null, 2))

        // Prueba con /store si tu ruta es así
        const response = await axios.post(`${API_URL}/ventas/store`, payload)
        // O prueba con esta línea si tu ruta es diferente:
        // const response = await axios.post(`${API_URL}/ventas`, payload)

        mostrarMensaje(`¡Venta registrada exitosamente! Factura: ${response.data.factura}`, 'success')

        // Limpiar todo
        carrito.value = []
        busqueda.value = ''
        articulosObtenidos.value = []
        busquedaRealizada.value = false

    } catch (error) {
        console.error('Error procesando la venta:', error)

        if (error.response) {
            // El servidor respondió con un error
            console.error('Respuesta del servidor:', error.response.data)
            mostrarMensaje(`Error: ${error.response.data.error || 'Error del servidor'}`, 'error')
        } else if (error.request) {
            // No se recibió respuesta
            console.error('No se recibió respuesta:', error.request)
            mostrarMensaje('No se pudo conectar con el servidor', 'error')
        } else {
            mostrarMensaje('Error al configurar la petición', 'error')
        }
    } finally {
        procesando.value = false
    }
}
</script>