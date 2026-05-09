<template>
    <div class="min-h-screen bg-gray-50 flex items-center justify-center px-4">
        <div class="bg-white rounded-xl shadow-lg w-full max-w-md p-8">

            <!-- Title -->
            <div class="text-center mb-8">
                <h1 class="text-3xl font-bold text-gray-800">User Management</h1>
                <p class="text-gray-500 text-sm mt-1">Create your account</p>
            </div>

            <form @submit.prevent="submit">

                <!-- Name -->
                <div class="mb-4">
                    <label class="block text-sm font-medium text-gray-700 mb-1">Full Name</label>
                    <input
                        v-model="form.name"
                        type="text"
                        placeholder="Your full name"
                        class="w-full border rounded-lg px-3 py-2 text-sm focus:outline-none focus:ring-2 focus:ring-blue-400"
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
                        class="w-full border rounded-lg px-3 py-2 text-sm focus:outline-none focus:ring-2 focus:ring-blue-400"
                        :class="errors.email ? 'border-red-400' : 'border-gray-300'"
                    />
                    <p v-if="errors.email" class="text-red-500 text-xs mt-1">{{ errors.email[0] }}</p>
                </div>

                <!-- Password -->
                <div class="mb-4">
                    <label class="block text-sm font-medium text-gray-700 mb-1">Password</label>
                    <input
                        v-model="form.password"
                        type="password"
                        placeholder="Min 8 characters"
                        class="w-full border rounded-lg px-3 py-2 text-sm focus:outline-none focus:ring-2 focus:ring-blue-400"
                        :class="errors.password ? 'border-red-400' : 'border-gray-300'"
                    />
                    <p v-if="errors.password" class="text-red-500 text-xs mt-1">{{ errors.password[0] }}</p>
                </div>

                <!-- Confirm Password -->
                <div class="mb-6">
                    <label class="block text-sm font-medium text-gray-700 mb-1">Confirm Password</label>
                    <input
                        v-model="form.password_confirmation"
                        type="password"
                        placeholder="Re-enter password"
                        class="w-full border rounded-lg px-3 py-2 text-sm focus:outline-none focus:ring-2 focus:ring-blue-400"
                        :class="errors.password_confirmation ? 'border-red-400' : 'border-gray-300'"
                    />
                </div>

                <button
                    type="submit"
                    :disabled="loading"
                    class="w-full bg-blue-600 hover:bg-blue-700 text-white font-medium py-2 rounded-lg transition text-sm disabled:opacity-50"
                >
                    {{ loading ? 'Creating account...' : 'Create Account' }}
                </button>
            </form>

            <p class="text-center text-sm text-gray-500 mt-6">
                Already have an account?
                <button @click="$emit('go-login')" class="text-blue-600 hover:underline font-medium">Sign In</button>
            </p>

        </div>
    </div>
</template>

<script setup>
import { ref, reactive } from 'vue'
import axios from 'axios'

const emit = defineEmits(['registered', 'go-login'])

const form    = reactive({ name: '', email: '', password: '', password_confirmation: '' })
const errors  = ref({})
const loading = ref(false)

async function submit() {
    loading.value = true
    errors.value  = {}

    try {
        const response = await axios.post('/api/register', {
            name:                  form.name,
            email:                 form.email,
            password:              form.password,
            password_confirmation: form.password_confirmation,
        })
        emit('registered', {
            token: response.data.token,
            user:  response.data.user,
        })
    } catch (err) {
        if (err.response?.status === 422) {
            errors.value = err.response.data.errors
        } else {
            errors.value = { name: ['Something went wrong. Please try again.'] }
        }
    } finally {
        loading.value = false
    }
}
</script>
