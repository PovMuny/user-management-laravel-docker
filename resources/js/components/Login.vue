<template>
    <div class="min-h-screen bg-gray-50 flex items-center justify-center px-4">
        <div class="bg-white rounded-xl shadow-lg w-full max-w-md p-8">

            <!-- Logo / Title -->
            <div class="text-center mb-8">
                <h1 class="text-3xl font-bold text-gray-800">User Management</h1>
                <p class="text-gray-500 text-sm mt-1">Sign in to your account</p>
            </div>

            <!-- Alert -->
            <div v-if="errorMessage" class="mb-4 bg-red-50 border border-red-200 text-red-700 text-sm px-4 py-3 rounded-lg">
                {{ errorMessage }}
            </div>

            <form @submit.prevent="submit">

                <!-- Email -->
                <div class="mb-4">
                    <label class="block text-sm font-medium text-gray-700 mb-1">Email</label>
                    <input
                        v-model="form.email"
                        type="email"
                        placeholder="email@example.com"
                        class="w-full border rounded-lg px-3 py-2 text-sm focus:outline-none focus:ring-2 focus:ring-blue-400"
                        :class="errors.email ? 'border-red-400' : 'border-gray-300'"
                    />
                    <p v-if="errors.email" class="text-red-500 text-xs mt-1">{{ errors.email[0] }}</p>
                </div>

                <!-- Password -->
                <div class="mb-6">
                    <label class="block text-sm font-medium text-gray-700 mb-1">Password</label>
                    <input
                        v-model="form.password"
                        type="password"
                        placeholder="Your password"
                        class="w-full border rounded-lg px-3 py-2 text-sm focus:outline-none focus:ring-2 focus:ring-blue-400"
                        :class="errors.password ? 'border-red-400' : 'border-gray-300'"
                    />
                    <p v-if="errors.password" class="text-red-500 text-xs mt-1">{{ errors.password[0] }}</p>
                </div>

                <button
                    type="submit"
                    :disabled="loading"
                    class="w-full bg-blue-600 hover:bg-blue-700 text-white font-medium py-2 rounded-lg transition text-sm disabled:opacity-50"
                >
                    {{ loading ? 'Signing in...' : 'Sign In' }}
                </button>
            </form>

            <p class="text-center text-sm text-gray-500 mt-6">
                Don't have an account?
                <button @click="$emit('go-register')" class="text-blue-600 hover:underline font-medium">Register</button>
            </p>

        </div>
    </div>
</template>

<script setup>
import { ref, reactive } from 'vue'
import axios from 'axios'

const emit = defineEmits(['logged-in', 'go-register'])

const form         = reactive({ email: '', password: '' })
const errors       = ref({})
const errorMessage = ref('')
const loading      = ref(false)

async function submit() {
    loading.value      = true
    errors.value       = {}
    errorMessage.value = ''

    try {
        const response = await axios.post('/api/login', {
            email:    form.email,
            password: form.password,
        })
        emit('logged-in', {
            token: response.data.token,
            user:  response.data.user,
        })
    } catch (err) {
        if (err.response?.status === 422) {
            errors.value = err.response.data.errors
        } else if (err.response?.status === 401) {
            errorMessage.value = err.response.data.message
        } else {
            errorMessage.value = 'Something went wrong. Please try again.'
        }
    } finally {
        loading.value = false
    }
}
</script>
