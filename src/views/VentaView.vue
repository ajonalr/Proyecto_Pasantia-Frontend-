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
import { useVenta } from '@/composables/useVenta';




const {

    busqueda,
    articulosObtenidos,
    carrito,
    loading,
    procesando,
    busquedaRealizada,
    clienteSeleccionado,
    snackbar,
    formatPrecio,
    buscarArticulos,
    agregarAlCarrito,
    recalcularTotalFila,
    eliminarDelCarrito,
    procesarVenta

} = useVenta()
</script>