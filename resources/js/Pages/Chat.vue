<script setup>
import AppLayout from '@/Layouts/AppLayout.vue';
import Chat from '@/Components/Chat.vue';

defineProps(
    {
        friend: Object,
        user: Object,
    }
);

</script>

<template>
    <AppLayout title="Chat Page">
        <template #header>
            <h2 class="font-semibold text-xl text-gray-800 leading-tight">
                Chat Page With {{ friend.name }}
            </h2>
        </template>

        <div class="py-12">
            <div class="max-w-7xl mx-auto sm:px-6 lg:px-8">
                <div class="bg-white overflow-hidden shadow-xl sm:rounded-lg">
                    <div class="py-12">
                        <div class="mx-auto max-w-7xl sm:px-6 lg:px-8">
                            <div class="overflow-hidden bg-white shadow-sm sm:rounded-lg">
                                <div class="p-6 bg-white border-b border-gray-200">
                                    <Chat
                                        :friend="friend"
                                        :currentUser="user"
                                    />
                                </div>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </AppLayout>
</template>
