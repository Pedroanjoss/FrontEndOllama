<template>
  <q-page class="flex flex-center">
    <div class="container">
      <!-- Seção de interação com a IA -->
      <div class="section">
        <q-input v-model="prompt" label="Digite seu prompt para a IA" class="input-box" />
        <div class="button-container">
          <q-btn @click="sendPrompt" :loading="loading" label="Enviar para IA" color="primary" />
        </div>
        <q-card v-if="aiResponse" class="response-card q-mt-md">
          <q-card-section>
            <p v-html="aiResponse" class="response-text"></p>
          </q-card-section>
        </q-card>
      </div>

      <!-- Seção de transcrição do YouTube -->
      <div class="section q-mt-lg">
        <q-input v-model="youtubeUrl" label="Digite a URL do vídeo do YouTube" class="input-box" />
        <div class="button-container">
          <q-btn @click="fetchTranscript" :loading="loadingTranscript" label="Obter Transcrição" color="primary" />
        </div>
        <q-card v-if="transcriptionResponse" class="response-card q-mt-md">
          <q-card-section>
            <p v-html="transcriptionResponse" class="response-text"></p>
          </q-card-section>
        </q-card>
      </div>
    </div>
  </q-page>
</template>

<script setup>
import { ref } from 'vue';

const prompt = ref(''); // Input para a IA
const youtubeUrl = ref(''); // Input para a URL do vídeo
const aiResponse = ref(''); // Resposta da IA
const transcriptionResponse = ref(''); // Resposta da transcrição
const loading = ref(false); // Estado de carregamento para a IA
const loadingTranscript = ref(false); // Estado de carregamento para a transcrição

// Função para enviar o prompt para a IA
const sendPrompt = async () => {
  loading.value = true;
  aiResponse.value = ''; // Limpa a resposta anterior

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
              aiResponse.value = formatText(text);
            }
          } catch (e) {
            console.error('Erro ao processar o JSON:', e);
          }
        }
      }
    }
  } catch (error) {
    console.error('Erro ao enviar o prompt:', error);
  } finally {
    loading.value = false;
  }
};

// Função para buscar a transcrição do vídeo do YouTube
const fetchTranscript = async () => {
  loadingTranscript.value = true;
  transcriptionResponse.value = ''; // Limpa a resposta anterior

  try {
    const res = await fetch('http://localhost:3031/transcribe', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ url: youtubeUrl.value }), // Envia a URL do vídeo
    });

    transcriptionResponse.value = await res.text();
  } catch (error) {
    console.error('Erro ao buscar a transcrição:', error);
  } finally {
    loadingTranscript.value = false;
  }
};

// Função para formatar o texto da resposta
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
  width: 180px; /* Aumenta a largura dos botões */
  height: 50px;
  font-size: 1rem; /* Tamanho da fonte ajustado */
  padding: 0 20px; 
}

.response-card {
  width: 100%;
  margin-top: 30px;
}

.response-text {
  font-size: 1.2rem;
  line-height: 1.6;
}

.section {
  width: 100%;
  margin-bottom: 30px;
}
</style>
