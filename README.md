# Chat App With Laravel Reverb And Vue 3 (Inertia)

- This is a simple real-time chat app built with Laravel Reverb and Vue 3 to test the capabilities of the Laravel Reverb package.
- Why I make this testing project is that I couldn't find any example of using Laravel Reverb with Vue 3 to build a simple chat app.
- The only a example I found is the project of the URL below, and this project is almost a copy of this example but using Jetstream (inertia). Appreciate the author of the project, Harish Kumar.
  - https://qirolab.com/posts/building-real-time-chat-applications-with-laravel-reverb-and-vue-3 

- The article on Qiita
  - https://qiita.com/KosukeTakagi/items/fabf9d7e33ccc21b9590

## Steps

### 1. Create a new Laravel project and Install Jetstream (Inertia) and Laravel Reverb

```bash
composer create-project laravel/laravel vue-chat

cd vue-chat

composer require laravel/jetstream


php artisan jetstream:install inertia

php artisan install:broadcasting

npm run build
```
I recommend to check if you can see the login page of Jetstream turning the server on with this command `php artisan serve` and if the laravel reverb can be on with this command `php artisan reverb:start --debug` with no error.


### 2. Creating the ChatMessage Model & Migration

```bash
php artisan make:model ChatMessage -m
```

- Open the migration file and update it as follows:

```php

<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('chat_messages', function (Blueprint $table) {
            $table->id();
            $table->foreignId('receiver_id');
            $table->foreignId('sender_id');
            $table->text('text');
            $table->timestamps();
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('chat_messages');
    }
};

```

- Migrate it by this command:
```bash
php artisan migrate
```

- Edit ChatMessage.php model file as follows:

```php
<?php
namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

class ChatMessage extends Model
{
    use HasFactory;

    protected $fillable = [
        'sender_id',
        'receiver_id',
        'text'
    ];

    public function sender()
    {
        return $this->belongsTo(User::class, 'sender_id');
    }

    public function receiver()
    {
        return $this->belongsTo(User::class, 'receiver_id');
    }
}
```

### 3. Defining Routes
- Open the `routes/web.php` file and update it as follows:

```php
<?php

use App\Events\MessageSent;
use App\Models\ChatMessage;
use App\Models\User;
use Illuminate\Foundation\Application;
use Illuminate\Support\Facades\Route;
use Inertia\Inertia;

Route::get('/', function () {
    return Inertia::render('Welcome', [
        'canLogin' => Route::has('login'),
        'canRegister' => Route::has('register'),
        'laravelVersion' => Application::VERSION,
        'phpVersion' => PHP_VERSION,
    ]);
});

Route::middleware([
    'auth:sanctum',
    config('jetstream.auth_session'),
    'verified',
])->group(function () {

    Route::get('/dashboard', function () {
        return Inertia::render('Dashboard', [
            'users' => User::query()->where('id', '!=', auth()->id())->get()
        ]);
    })->middleware(['auth'])->name('dashboard');

    Route::get('/chat/{friend}', function (User $friend) {
        return Inertia::render('Chat', [
            'friend' => $friend,
            'user' => auth()->user()
        ]);
    })->middleware(['auth'])->name('chat');

    Route::get('/messages/{friend}', function (User $friend) {
        return ChatMessage::query()
            ->where(function ($query) use ($friend) {
                $query->where('sender_id', auth()->id())
                    ->where('receiver_id', $friend->id);
            })
            ->orWhere(function ($query) use ($friend) {
                $query->where('sender_id', $friend->id)
                    ->where('receiver_id', auth()->id());
            })
            ->with(['sender', 'receiver'])
            ->orderBy('id', 'asc')
            ->get();
    })->middleware(['auth']);

    Route::post('/messages/{friend}', function (User $friend) {
        $message = ChatMessage::create([
            'sender_id' => auth()->id(),
            'receiver_id' => $friend->id,
            'text' => request()->input('message')
        ]);

        broadcast(new MessageSent($message));

        return $message;
    });
});
```


### 4. Creating the MessageSent Event
- Create and Edit `app/Events/MessageSent.php` file as follows:

```php

<?php
namespace App\Events;

use App\Models\ChatMessage;
use Illuminate\Broadcasting\Channel;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Queue\SerializesModels;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Contracts\Broadcasting\ShouldBroadcastNow;

class MessageSent implements ShouldBroadcastNow
{
    use Dispatchable;
    use InteractsWithSockets;
    use SerializesModels;

    /**
     * Create a new event instance.
     */
    public function __construct(public ChatMessage $message)
    {
        //
    }

    /**
     * Get the channels the event should broadcast on.
     *
     * @return array<int, Channel>
     */
    public function broadcastOn(): array
    {
        return [
            new PrivateChannel("chat.{$this->message->receiver_id}"),
        ];
    }
}
```

### 5. Defining Private Channels in channels.php

- Open the `routes/channels.php` file and add the following code:

```php
Broadcast::channel('chat.{id}', function ($user, $id) {
    return (int) $user->id === (int) $id;
});
```



### 6. Creating the pages and components

- Arrange dashboard page by editing `resources/js/Pages/Dashboard.vue` file as follows:

```vue

<script setup>
import AppLayout from '@/Layouts/AppLayout.vue';

defineProps(
    {
        users: Array,
    }
);

</script>

<template>
    <AppLayout title="Dashboard">
        <template #header>
            <h2 class="font-semibold text-xl text-gray-800 leading-tight">
                Dashboard
            </h2>
        </template>

        <div class="py-12">
            <div class="max-w-7xl mx-auto sm:px-6 lg:px-8">
                <div class="bg-white overflow-hidden shadow-xl sm:rounded-lg">
                    <div v-for="user in users" class="overflow-hidden bg-white shadow-sm sm:rounded-lg">
                        <div class="p-6">
                            <div class="flex items-center">
                                <a :href="`chat/${user.id}`">
                                    <div class="ml-4">
                                        <div class="text-sm font-medium text-gray-900">{{ user.name }}</div>
                                        <div class="text-sm text-gray-500">{{ user.email }}</div>
                                    </div>
                                </a>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </AppLayout>
</template>

```

- Create the Chat component by editing the file `resources/js/Components/Chat.vue` as follows:

```vue

<script setup>
import axios from "axios";
import { nextTick, onMounted, ref, watch } from "vue";

const props = defineProps({
    friend: {
        type: Object,
        required: true,
    },
    currentUser: {
        type: Object,
        required: true,
    },
});

const messages = ref([]);
const newMessage = ref("");
const messagesContainer = ref(null);
const isFriendTyping = ref(false);
const isFriendTypingTimer = ref(null);

watch(
    messages,
    () => {
        nextTick(() => {
            messagesContainer.value.scrollTo({
                top: messagesContainer.value.scrollHeight,
                behavior: "smooth",
            });
        });
    },
    { deep: true }
);

const sendMessage = () => {
    if (newMessage.value.trim() !== "") {
        axios
            .post(`/messages/${props.friend.id}`, {
                message: newMessage.value,
            })
            .then((response) => {
                messages.value.push(response.data);
                newMessage.value = "";
            });
    }
};

const sendTypingEvent = () => {
    Echo.private(`chat.${props.friend.id}`).whisper("typing", {
        userID: props.currentUser.id,
    });
};

onMounted(() => {
    axios.get(`/messages/${props.friend.id}`).then((response) => {
        console.log(response.data);
        messages.value = response.data;
    });

    Echo.private(`chat.${props.currentUser.id}`)
        .listen("MessageSent", (response) => {
            messages.value.push(response.message);
        })
        .listenForWhisper("typing", (response) => {
            isFriendTyping.value = response.userID === props.friend.id;

            if (isFriendTypingTimer.value) {
                clearTimeout(isFriendTypingTimer.value);
            }

            isFriendTypingTimer.value = setTimeout(() => {
                isFriendTyping.value = false;
            }, 1000);
        });
});
</script>

<template>
    <div>
        <div class="flex flex-col justify-end h-80">
            <div ref="messagesContainer" class="p-4 overflow-y-auto max-h-fit">
                <div
                    v-for="message in messages"
                    :key="message.id"
                    class="flex items-center mb-2"
                >
                    <div
                        v-if="message.sender_id === currentUser.id"
                        class="p-2 ml-auto text-white bg-blue-500 rounded-lg"
                    >
                        {{ message.text }}
                    </div>
                    <div v-else class="p-2 mr-auto bg-gray-200 rounded-lg">
                        {{ message.text }}
                    </div>
                </div>
            </div>
        </div>
        <div class="flex items-center">
            <input
                type="text"
                v-model="newMessage"
                @keydown="sendTypingEvent"
                @keyup.enter="sendMessage"
                placeholder="Type a message..."
                class="flex-1 px-2 py-1 border rounded-lg"
            />
            <button
                @click="sendMessage"
                class="px-4 py-1 ml-2 text-white bg-blue-500 rounded-lg"
            >
                Send
            </button>
        </div>
        <small v-if="isFriendTyping" class="text-gray-700">
            {{ friend.name }} is typing...
        </small>
    </div>
</template>

<style scoped>

</style>
```

- Create the Chat page by editing the file `resources/js/Pages/Chat.vue` as follows:

```vue

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
```

### 7. Produce test user data

- Edit the seeder file `database/seeders/DatabaseSeeder.php` as follows.

```php

<?php

namespace Database\Seeders;

use App\Models\User;
use Illuminate\Database\Seeder;

class DatabaseSeeder extends Seeder
{
    /**
     * Seed the application's database.
     */
    public function run(): void
    {
        User::factory(10)->create();
    }
}
```

- Run the seeder by this command:
```bash
php artisan db:seed
```

### 8. Running the Application
- Run the application by these commands on different terminals:
```bash
php artisan serve
```
```bash
php artisan reverb:start --debug
```
```bash
npm run dev
```

![The Chat image when succeeded to build](./screenshot_success.png?raw=true "Chat")
