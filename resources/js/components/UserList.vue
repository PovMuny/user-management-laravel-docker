<template>
    <div class="min-h-screen bg-gray-50">

        <!-- Navbar -->
        <nav class="bg-white border-b px-8 py-3 flex items-center justify-between">
            <span class="font-bold text-gray-800 text-lg">User Management</span>
            <div class="flex items-center space-x-4">
                <span class="text-sm text-gray-500">
                    Signed in as <strong class="text-gray-700">{{ authUser?.name }}</strong>
                </span>
                <button @click="logout" class="bg-red-500 hover:bg-red-600 text-white text-sm px-3 py-1.5 rounded-lg transition font-medium">
                    Logout
                </button>
            </div>
        </nav>

    <div class="p-8">
        <div class="max-w-5xl mx-auto">

            <!-- Header -->
            <div class="flex items-center justify-between mb-6">
                <h1 class="text-3xl font-bold text-gray-800">Users</h1>
                <button @click="openCreateModal" class="bg-blue-600 hover:bg-blue-700 text-white px-4 py-2 rounded-lg font-medium transition">
                    + Add User
                </button>
            </div>

            <!-- Search Bar -->
            <div class="mb-4">
                <input
                    v-model="search"
                    @input="onSearch"
                    type="text"
                    placeholder="Search by name or email..."
                    class="w-full border border-gray-300 rounded-lg px-4 py-2 focus:outline-none focus:ring-2 focus:ring-blue-400"
                />
            </div>

            <!-- Alert Message -->
            <div v-if="alert.message" :class="['mb-4 px-4 py-3 rounded-lg text-sm font-medium', alert.type === 'success' ? 'bg-green-100 text-green-800' : 'bg-red-100 text-red-800']">
                {{ alert.message }}
            </div>

            <!-- Table -->
            <div class="bg-white rounded-xl shadow overflow-hidden">
                <table class="w-full text-sm text-left">
                    <thead class="bg-gray-100 text-gray-600 uppercase text-xs">
                        <tr>
                            <th class="px-6 py-3">ID</th>
                            <th class="px-6 py-3">Name</th>
                            <th class="px-6 py-3">Email</th>
                            <th class="px-6 py-3">Created</th>
                            <th class="px-6 py-3 text-center">Actions</th>
                        </tr>
                    </thead>
                    <tbody>
                        <tr v-if="loading">
                            <td colspan="5" class="text-center py-10 text-gray-400">Loading...</td>
                        </tr>
                        <tr v-else-if="users.length === 0">
                            <td colspan="5" class="text-center py-10 text-gray-400">No users found.</td>
                        </tr>
                        <tr v-for="user in users" :key="user.id" class="border-t hover:bg-gray-50 transition">
                            <td class="px-6 py-3 text-gray-500">{{ user.id }}</td>
                            <td class="px-6 py-3 font-medium text-gray-800">{{ user.name }}</td>
                            <td class="px-6 py-3 text-gray-600">{{ user.email }}</td>
                            <td class="px-6 py-3 text-gray-400">{{ formatDate(user.created_at) }}</td>
                            <td class="px-6 py-3 text-center space-x-2">
                                <button @click="openEditModal(user)" class="bg-yellow-400 hover:bg-yellow-500 text-white px-3 py-1 rounded text-xs font-medium transition">
                                    Edit
                                </button>
                                <button @click="confirmDelete(user)" class="bg-red-500 hover:bg-red-600 text-white px-3 py-1 rounded text-xs font-medium transition">
                                    Delete
                                </button>
                            </td>
                        </tr>
                    </tbody>
                </table>
            </div>

            <!-- Pagination -->
            <div v-if="pagination.last_page > 1" class="flex items-center justify-between mt-6">
                <p class="text-sm text-gray-500">
                    Showing page {{ pagination.current_page }} of {{ pagination.last_page }}
                    ({{ pagination.total }} users total)
                </p>
                <div class="flex space-x-1">
                    <button
                        v-for="page in pagination.last_page"
                        :key="page"
                        @click="goToPage(page)"
                        :class="['px-3 py-1 rounded text-sm font-medium transition', page === pagination.current_page ? 'bg-blue-600 text-white' : 'bg-white border text-gray-600 hover:bg-gray-100']"
                    >
                        {{ page }}
                    </button>
                </div>
            </div>

        </div>
    </div>
    </div>

    <!-- Modal: Create / Edit -->
    <div v-if="modal.show" class="fixed inset-0 bg-black/50 flex items-center justify-center z-50">
        <div class="bg-white rounded-xl shadow-xl w-full max-w-md mx-4 p-6">

            <h2 class="text-xl font-bold text-gray-800 mb-5">
                {{ modal.mode === 'create' ? 'Add New User' : 'Edit User' }}
            </h2>

            <form @submit.prevent="submitForm">

                <!-- Name -->
                <div class="mb-4">
                    <label class="block text-sm font-medium text-gray-700 mb-1">Name</label>
                    <input
                        v-model="form.name"
                        type="text"
                        placeholder="Full name"
                        class="w-full border rounded-lg px-3 py-2 focus:outline-none focus:ring-2 focus:ring-blue-400"
                        :class="errors.name ? 'border-red-400' : 'border-gray-300'"
                    />
                    <p v-if="errors.name" class="text-red-500 text-xs mt-1">{{ errors.name[0] }}</p>
                </div>

                <!-- Email -->
                <div class="mb-4">
                    <label class="block text-sm font-medium text-gray-700 mb-1">Email</label>
                    <input
                        v-model="form.email"
                        type="email"
                        placeholder="email@example.com"
                        class="w-full border rounded-lg px-3 py-2 focus:outline-none focus:ring-2 focus:ring-blue-400"
                        :class="errors.email ? 'border-red-400' : 'border-gray-300'"
                    />
                    <p v-if="errors.email" class="text-red-500 text-xs mt-1">{{ errors.email[0] }}</p>
                </div>

                <!-- Password -->
                <div class="mb-6">
                    <label class="block text-sm font-medium text-gray-700 mb-1">
                        Password
                        <span v-if="modal.mode === 'edit'" class="text-gray-400 font-normal">(leave blank to keep current)</span>
                    </label>
                    <input
                        v-model="form.password"
                        type="password"
                        placeholder="Min 8 characters"
                        class="w-full border rounded-lg px-3 py-2 focus:outline-none focus:ring-2 focus:ring-blue-400"
                        :class="errors.password ? 'border-red-400' : 'border-gray-300'"
                    />
                    <p v-if="errors.password" class="text-red-500 text-xs mt-1">{{ errors.password[0] }}</p>
                </div>

                <!-- Buttons -->
                <div class="flex justify-end space-x-3">
                    <button type="button" @click="closeModal" class="px-4 py-2 rounded-lg border border-gray-300 text-gray-600 hover:bg-gray-50 transition text-sm">
                        Cancel
                    </button>
                    <button type="submit" :disabled="submitting" class="px-4 py-2 rounded-lg bg-blue-600 hover:bg-blue-700 text-white font-medium transition text-sm disabled:opacity-50">
                        {{ submitting ? 'Saving...' : (modal.mode === 'create' ? 'Create User' : 'Save Changes') }}
                    </button>
                </div>

            </form>
        </div>
    </div>

    <!-- Modal: Delete Confirm -->
    <div v-if="deleteModal.show" class="fixed inset-0 bg-black/50 flex items-center justify-center z-50">
        <div class="bg-white rounded-xl shadow-xl w-full max-w-sm mx-4 p-6 text-center">
            <div class="text-red-500 text-5xl mb-3">&#9888;</div>
            <h2 class="text-lg font-bold text-gray-800 mb-2">Delete User?</h2>
            <p class="text-gray-500 text-sm mb-6">
                Are you sure you want to delete <strong>{{ deleteModal.user?.name }}</strong>?
                This action cannot be undone.
            </p>
            <div class="flex justify-center space-x-3">
                <button @click="deleteModal.show = false" class="px-4 py-2 rounded-lg border border-gray-300 text-gray-600 hover:bg-gray-50 transition text-sm">
                    Cancel
                </button>
                <button @click="deleteUser" :disabled="submitting" class="px-4 py-2 rounded-lg bg-red-500 hover:bg-red-600 text-white font-medium transition text-sm disabled:opacity-50">
                    {{ submitting ? 'Deleting...' : 'Yes, Delete' }}
                </button>
            </div>
        </div>
    </div>
</template>

<script setup>
import { ref, reactive, onMounted } from 'vue'
import axios from 'axios'

const props = defineProps({ authUser: Object })
const emit  = defineEmits(['logged-out'])

// --- State ---
const users      = ref([])
const loading    = ref(false)
const submitting = ref(false)
const search     = ref('')
const pagination = reactive({ total: 0, per_page: 10, current_page: 1, last_page: 1 })
const alert      = reactive({ message: '', type: 'success' })

const modal = reactive({ show: false, mode: 'create' })
const form  = reactive({ id: null, name: '', email: '', password: '' })
const errors = ref({})

const deleteModal = reactive({ show: false, user: null })

let searchTimer = null

// --- Fetch Users ---
async function fetchUsers(page = 1) {
    loading.value = true
    try {
        const response = await axios.get('/api/users', {
            params: { page, search: search.value, per_page: 10 }
        })
        users.value      = response.data.users
        Object.assign(pagination, response.data.pagination)
    } catch (err) {
        if (err.response?.status === 401) {
            emit('logged-out')
        } else {
            showAlert('Failed to load users.', 'error')
        }
    } finally {
        loading.value = false
    }
}

// --- Logout ---
async function logout() {
    try {
        await axios.post('/api/logout')
    } finally {
        emit('logged-out')
    }
}

// --- Search (debounced 400ms) ---
function onSearch() {
    clearTimeout(searchTimer)
    searchTimer = setTimeout(() => fetchUsers(1), 400)
}

// --- Pagination ---
function goToPage(page) {
    fetchUsers(page)
}

// --- Modal: Create ---
function openCreateModal() {
    modal.mode     = 'create'
    modal.show     = true
    form.id        = null
    form.name      = ''
    form.email     = ''
    form.password  = ''
    errors.value   = {}
}

// --- Modal: Edit ---
function openEditModal(user) {
    modal.mode     = 'edit'
    modal.show     = true
    form.id        = user.id
    form.name      = user.name
    form.email     = user.email
    form.password  = ''
    errors.value   = {}
}

function closeModal() {
    modal.show = false
}

// --- Submit: Create or Update ---
async function submitForm() {
    submitting.value = true
    errors.value     = {}
    try {
        if (modal.mode === 'create') {
            await axios.post('/api/users', {
                name: form.name, email: form.email, password: form.password
            })
            showAlert('User created successfully.', 'success')
        } else {
            const payload = { name: form.name, email: form.email }
            if (form.password) payload.password = form.password
            await axios.put(`/api/users/${form.id}`, payload)
            showAlert('User updated successfully.', 'success')
        }
        closeModal()
        fetchUsers(pagination.current_page)
    } catch (err) {
        if (err.response?.status === 422) {
            errors.value = err.response.data.errors
        } else {
            showAlert('Something went wrong. Please try again.', 'error')
        }
    } finally {
        submitting.value = false
    }
}

// --- Delete ---
function confirmDelete(user) {
    deleteModal.user = user
    deleteModal.show = true
}

async function deleteUser() {
    submitting.value = true
    try {
        await axios.delete(`/api/users/${deleteModal.user.id}`)
        deleteModal.show = false
        showAlert('User deleted successfully.', 'success')
        const page = users.value.length === 1 && pagination.current_page > 1
            ? pagination.current_page - 1
            : pagination.current_page
        fetchUsers(page)
    } catch {
        showAlert('Failed to delete user.', 'error')
    } finally {
        submitting.value = false
    }
}

// --- Alert ---
function showAlert(message, type = 'success') {
    alert.message = message
    alert.type    = type
    setTimeout(() => { alert.message = '' }, 4000)
}

// --- Format date ---
function formatDate(dateStr) {
    return new Date(dateStr).toLocaleDateString('en-GB', {
        day: '2-digit', month: 'short', year: 'numeric'
    })
}

onMounted(() => fetchUsers())
</script>
