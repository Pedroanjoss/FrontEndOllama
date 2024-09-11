<template>
  <q-page class="flex flex-center">
    <div class="container">
      <q-input v-model="prompt" label="Enter your prompt" class="input-box" />

      <div class="button-container">
        <q-btn @click="sendPrompt" :loading="loading" label="Send" color="primary" />
      </div>

      <q-card v-if="formattedResponse" class="response-card q-mt-md">
        <q-card-section>
          <p v-html="formattedResponse" class="response-text"></p>
        </q-card-section>
      </q-card>
    </div>
  </q-page>
</template>

<script setup>
import { ref } from 'vue';

const prompt = ref('');
const formattedResponse = ref('');
const loading = ref(false);

const sendPrompt = async () => {
  loading.value = true;
  formattedResponse.value = ''; // Clear previous response

  try {
    const res = await fetch('http://localhost:3031', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ text: prompt.value }),
    });

    const reader = res.body.getReader();
    const decoder = new TextDecoder();
    let done = false;
    let text = '';

    while (!done) {
      const { value, done: readerDone } = await reader.read();
      done = readerDone;

      if (value) {
        const chunk = decoder.decode(value, { stream: !done });
        const lines = chunk.split('\n').filter(Boolean);

        for (const line of lines) {
          try {
            const parsed = JSON.parse(line);
            if (parsed.response) {
              text += parsed.response;
              formattedResponse.value = formatText(text);
            }
          } catch (e) {
            console.error('Error parsing JSON chunk:', e);
          }
        }
      }
    }
  } catch (error) {
    console.error('Error sending prompt:', error);
  } finally {
    loading.value = false;
  }
};

function formatText(text) {
  return text.replace(/\n/g, '<br>');
}
</script>

<style scoped>
.container {
  max-width: 600px;
  width: 100%;
  padding: 20px;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.input-box {
  width: 100%;
  font-size: 1.2rem;
}

.button-container {
  display: flex;
  justify-content: center;
  margin-top: 20px;
}

.q-btn {
  width: 150px;
  height: 50px;
  font-size: 1.1rem;
}

.response-card {
  width: 100%;
  margin-top: 30px;
}

.response-text {
  font-size: 1.2rem;
  line-height: 1.6;
}
</style>
