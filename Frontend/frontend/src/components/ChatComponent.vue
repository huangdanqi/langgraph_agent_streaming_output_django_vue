<template>
  <div class="chat-container">
    <div class="messages-container">
      <!-- Render each message with a unique id attribute -->
      <div
        v-for="message in messages"
        :key="message.id"
        :id="'message-' + message.id"
        :class="['message', message.sender.toLowerCase()]"
      >
        <strong v-if="message.sender !== 'generating'">{{ message.sender }}</strong>
        <p>{{ message.message }}</p>
      </div>

      <!-- "Generating..." indicator -->
      <div v-if="isGenerating" class="message generating">
        <p>Generating{{ generatingDots }}</p>
      </div>

      <!-- Dummy element to scroll to the bottom -->
      <div ref="messagesEndRef"></div>
    </div>

    <!-- Input form -->
    <form @submit.prevent="sendMessage" class="input-form">
      <input
        type="text"
        v-model="inputMessage"
        placeholder="Type your message..."
      />
      <button type="submit">Send</button>
    </form>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount, watch, nextTick } from 'vue';

// Simple counter to generate unique ids for messages
let messageIdCounter = 1;

const inputMessage = ref('');
const messages = ref([]); 
const ws = ref(null);
const messagesEndRef = ref(null);

// Holds the current bot message for the active request
const currentBotMessage = ref(null);
const isGenerating = ref(false); // Flag to show the "Generating..." indicator
const generatingDots = ref('.');
let generatingInterval = null;

const setupWebSocket = () => {
  ws.value = new WebSocket('ws://127.0.0.1:8000/ws/chat/');
  ws.value.onopen = () => console.log('WebSocket connected');
  ws.value.onerror = (error) => console.error('WebSocket error:', error);
  ws.value.onclose = () => console.log('WebSocket closed.');

  ws.value.onmessage = (event) => {
    const data = JSON.parse(event.data);

    // If no current bot message exists, create one and assign a new id
    if (!currentBotMessage.value) {
      currentBotMessage.value = { id: messageIdCounter++, sender: 'Bot', message: '' };
      messages.value.push(currentBotMessage.value);
      // Hide the "Generating..." indicator once the first chunk arrives
      isGenerating.value = false;
    }

    // Append the chunk to the current bot message
    currentBotMessage.value.message += data.agent;

    // Scroll to the bottom after updating the message
    nextTick(() => {
      messagesEndRef.value?.scrollIntoView({ behavior: 'smooth' });
    });

    // Optionally, if you receive a signal that this response is complete,
    // you can finalize the message (here we're not creating a new object)
    if (data.isFinished) {
      // You can do any final processing here if needed.
    }
  };
};

const sendMessage = () => {
  if (!inputMessage.value.trim()) return;

  // Create a new user message with a unique id
  messages.value.push({
    id: messageIdCounter++,
    sender: 'User',
    message: inputMessage.value,
  });

  // Reset current bot message so that a new one is created for the response
  currentBotMessage.value = null;
  // Show "Generating..." indicator with dynamic dots
  isGenerating.value = true;
  generatingDots.value = '.';
  startGeneratingDots();

  // Send the user's message via WebSocket
  ws.value.send(JSON.stringify({ message: inputMessage.value }));
  inputMessage.value = '';
};

const startGeneratingDots = () => {
  generatingInterval = setInterval(() => {
    generatingDots.value =
      generatingDots.value.length === 3 ? '.' : generatingDots.value + '.';
  }, 500);
};

const stopGeneratingDots = () => {
  clearInterval(generatingInterval);
  generatingDots.value = '';
};

watch(isGenerating, (newValue) => {
  if (!newValue) {
    stopGeneratingDots();
  }
});

watch(messages, () => {
  nextTick(() => {
    messagesEndRef.value?.scrollIntoView({ behavior: 'smooth' });
  });
});

onMounted(() => {
  setupWebSocket();
});

onBeforeUnmount(() => {
  if (ws.value) ws.value.close();
});
</script>

<style scoped>

/* Ensure that the html and body elements fill the entire height of the screen */
html, body {
  margin: 0;
  padding: 0;
  height: 100%; /* Ensure the html and body take up the full height */
  display: flex;
  justify-content: center; /* Horizontally center the chat container */
  align-items: center; /* Vertically center the chat container */
  background: #f0f2f5;
  font-family: Arial, sans-serif;
}

.chat-container {
  width: 80%;  /* Set the width to 80% of the page */
  height: 90vh;  /* Set the height to 80% of the viewport height */
  max-width: 800px;  /* Optional max-width */
  background: white;
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.1);
  border-radius: 10px;
  padding: 20px;
  display: flex;
  flex-direction: column;
  justify-content: flex-end; /* Push input form to the bottom */
  position: absolute;
  top: 50%;  /* Center the container vertically */
  left: 50%;  /* Center the container horizontally */
  transform: translate(-50%, -50%);  /* Adjust for true center */
  overflow: hidden;
}
/* Container for the messages */
.messages-container {
  flex: 1;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  padding-bottom: 100px; /* Space to avoid overlap with the input */
  max-height: calc(100vh - 200px); /* Ensure the height is responsive */
}

/* Styling for the message bubbles */
.message {
  max-width: 75%;
  padding: 12px;
  border-radius: 10px;
  margin-bottom: 10px;
  word-wrap: break-word;
  text-align: left;
  width: auto;
}

/* User messages (Right side) */
.user {
  align-self: flex-end;
  background: rgba(0, 188, 255, 0.4); /* Cyan with opacity */
  color: black;
  border-radius: 20px;
  padding: 12px;
  margin-bottom: 10px;
  word-wrap: break-word;
  text-align: left;
  font-family:'Franklin Gothic Medium', 'Arial Narrow', Arial, sans-serif;
}

/* Bot messages (Left side) */
.bot {
  align-self: flex-start;
  background: rgba(203, 117, 11, 0.2); /* Yellow with opacity */
  color: black;
  border-radius: 20px;
  padding: 12px;
  margin-bottom: 10px;
  word-wrap: break-word;
  text-align: left;
  font-family:'Franklin Gothic Medium', 'Arial Narrow', Arial, sans-serif;
}

/* Generating message style */
.generating {
  color: black;
  font-style: italic;
  font-weight: bold;
  align-self: center;
  width: auto;
  padding: 12px;
  border-radius: 10px;
  text-align: center;
  background: none;  /* Remove the background color */
}

/* Input form stays at the bottom */
.input-form {
  display: flex;
  position: absolute;
  bottom: 10px;
  left: 0;
  width: 100%;
  padding: 10px;
  background: white;
  border-top: 1px solid #ddd;
}

input {
  flex: 1;
  padding: 12px;
  border: none;
  border-radius: 20px;
  background: #f0f0f0;
  font-size: 16px;
  outline: none;
}

button {
  padding: 12px 20px;
  margin-left: 10px;
  margin-right:15px ;
  border: none;
  border-radius: 20px;
  background: #ff7eb3;
  color: white;
  font-size: 16px;
  cursor: pointer;
}

button:hover {
  background: #ff758c;
}
</style>
