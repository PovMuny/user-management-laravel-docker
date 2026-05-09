<template>
    <div>
        <Register v-if="page === 'register'" @registered="onLoggedIn" @go-login="page = 'login'" />
        <Login    v-else-if="page === 'login'" @logged-in="onLoggedIn" @go-register="page = 'register'" />
        <UserList v-else @logged-out="onLoggedOut" :auth-user="authUser" />
    </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import axios from 'axios'
import Login    from './Login.vue'
import Register from './Register.vue'
import UserList from './UserList.vue'

const page     = ref('login')
const authUser = ref(null)

onMounted(() => {
    const token = localStorage.getItem('api_token')
    if (token) {
        axios.defaults.headers.common['Authorization'] = `Bearer ${token}`
        const user = localStorage.getItem('auth_user')
        if (user) {
            authUser.value = JSON.parse(user)
            page.value = 'app'
        }
    }
})

function onLoggedIn({ token, user }) {
    localStorage.setItem('api_token', token)
    localStorage.setItem('auth_user', JSON.stringify(user))
    axios.defaults.headers.common['Authorization'] = `Bearer ${token}`
    authUser.value = user
    page.value = 'app'
}

function onLoggedOut() {
    localStorage.removeItem('api_token')
    localStorage.removeItem('auth_user')
    delete axios.defaults.headers.common['Authorization']
    authUser.value = null
    page.value = 'login'
}
</script>
