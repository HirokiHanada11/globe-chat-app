<template>
    <app-layout>
        <template #header>
            <h2 class="font-semibold text-xl text-gray-800 leading-tight">
                Create New Chat Room
            </h2>
        </template>

        <div class="py-12">
            <div class="max-w-7xl mx-auto sm:px-6 lg:px-8">
                <div class="bg-white overflow-hidden shadow-xl sm:rounded-lg">
                    <new-room-form 
                    v-on:roomcreated="returnToChat()" 
                    />
                </div>
            </div>
        </div>
    </app-layout>
</template>

<script>
    import AppLayout from '@/Layouts/AppLayout'
    import NewRoomForm from './NewRoomForm.vue'

    export default {
        components: {
            AppLayout,
            NewRoomForm,
        },
        methods: {
            returnToChat () {
                window.location = '/chatrooms';
            }
        }
    }
</script>