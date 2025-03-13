<script lang="ts">
  import { createEventDispatcher, onMount, onDestroy } from 'svelte';
  import { toast } from 'svelte-sonner';
  import { getContext } from 'svelte';
  import type { Writable } from 'svelte/store';
  import type { i18n as i18nType } from 'i18next';
  import Modal from '../common/Modal.svelte';
  
  const i18n: Writable<i18nType> = getContext('i18n');
  const dispatch = createEventDispatcher();
  
  export let show = false;
  export let jsonSchema = '';
  export let chatId = '';
  
  let jsonSchemaInput = jsonSchema;
  let isValid = true;
  let errorMessage = '';
  let previousChatId = '';
  
  // Debug logs for initialization
  console.log('JsonSchemaModal initialized with schema:', jsonSchema);
  console.log('JsonSchemaModal initialized with chatId:', chatId);
  
  // Clean up on component destroy
  onDestroy(() => {
    document.body.classList.remove('modal-open');
  });
  
  // Watch for changes in chatId to clear schema when a new chat is created
  $: if (chatId && chatId !== previousChatId) {
    console.log(`Chat ID changed from ${previousChatId} to ${chatId}`);
    
    // If this is a new chat (not just initial load), clear the schema
    if (previousChatId !== '') {
      console.log('New chat detected, clearing JSON schema');
      jsonSchema = '';
      jsonSchemaInput = '';
      
      // Notify parent components
      dispatch('change', { jsonSchema: '' });
      dispatch('jsonSchemaChange', { jsonSchema: '' });
      
      // Clear localStorage - be aggressive in removing all schema data
      localStorage.removeItem(`jsonSchema_${chatId}`);
      localStorage.removeItem(`jsonSchema_${previousChatId}`);
      localStorage.removeItem('ollama-json-schema');
      localStorage.removeItem('temp_jsonSchema');
      
      // Clear any other JSON schema entries
      const allKeys = Object.keys(localStorage);
      for (const key of allKeys) {
        if (key.startsWith('jsonSchema_')) {
          console.log('Removing schema for key:', key);
          localStorage.removeItem(key);
        }
      }
    }
    
    previousChatId = chatId;
  }
  
  // Also watch for changes in jsonSchema from parent component
  $: if (jsonSchema !== jsonSchemaInput) {
    console.log('jsonSchema changed externally:', jsonSchema);
    jsonSchemaInput = jsonSchema;
  }
  
  // Load schema from localStorage if available and not already provided
  $: {
    if (chatId && !jsonSchema) {
      const savedSchema = localStorage.getItem(`jsonSchema_${chatId}`);
      if (savedSchema) {
        console.log(`Loading JSON schema from localStorage for chat ${chatId}:`, savedSchema);
        jsonSchemaInput = savedSchema;
        jsonSchema = savedSchema;
        dispatch('change', { jsonSchema: savedSchema });
        dispatch('jsonSchemaChange', { jsonSchema: savedSchema });
      } else {
        console.log(`No saved JSON schema found in localStorage for chat ${chatId}`);
      }
    }
  }
  
  function validateJson(json: string) {
    try {
      if (json.trim() === '') {
        return { isValid: true, error: '' };
      }
      JSON.parse(json);
      return { isValid: true, error: '' };
    } catch (e) {
      const error = e instanceof Error ? e.message : String(e);
      return { isValid: false, error };
    }
  }
  
  function handleSave() {
    const { isValid: valid, error } = validateJson(jsonSchema);
    isValid = valid;
    errorMessage = error;
    
    if (isValid) {
      // If the schema is empty, treat it as a clear operation
      if (jsonSchema.trim() === '') {
        handleClear();
        return;
      }
      
      // Save to localStorage with chat-specific key
      if (chatId) {
        console.log(`Saving JSON schema to localStorage for chat ${chatId}:`, jsonSchema);
        localStorage.setItem(`jsonSchema_${chatId}`, jsonSchema);
        
        // Also save to a global key for persistence across all chats
        localStorage.setItem('ollama-json-schema', jsonSchema);
      } else {
        console.log('No chatId available, saving to global schema only');
        localStorage.setItem('ollama-json-schema', jsonSchema);
      }
      
      dispatch('change', { jsonSchema });
      dispatch('jsonSchemaChange', { jsonSchema });
      dispatch('save', { jsonSchema });
      dispatch('close');
      show = false;
      toast.success('JSON Schema saved successfully');
    }
  }
  
  function handleClear() {
    // Clear the schema
    jsonSchema = '';
    jsonSchemaInput = '';
    
    // Remove from localStorage
    if (chatId) {
      localStorage.removeItem(`jsonSchema_${chatId}`);
    }
    localStorage.removeItem('ollama-json-schema');
    
    // Notify parent components
    dispatch('change', { jsonSchema: '' });
    dispatch('jsonSchemaChange', { jsonSchema: '' });
    dispatch('save', { jsonSchema: '' });
    dispatch('close');
    show = false;
    toast.success('JSON Schema cleared successfully');
  }
  
  function handleClose() {
    show = false;
    dispatch('close');
  }

  // When the modal is shown, check if there's a schema in localStorage
  $: if (show) {
    // First try to load from chat-specific storage
    if (chatId) {
      const savedSchema = localStorage.getItem(`jsonSchema_${chatId}`);
      if (savedSchema) {
        jsonSchema = savedSchema;
        jsonSchemaInput = savedSchema;
        console.log('Loaded JSON schema from chat-specific storage:', savedSchema);
      } else {
        // If no schema found for this chat, reset both values
        jsonSchema = '';
        jsonSchemaInput = '';
      }
    }
    
    // If no chat-specific schema, try the global schema
    if (!jsonSchema) {
      const globalSchema = localStorage.getItem('ollama-json-schema');
      if (globalSchema) {
        jsonSchema = globalSchema;
        jsonSchemaInput = globalSchema;
        console.log('Loaded JSON schema from global storage:', globalSchema);
      } else {
        // If no global schema found, reset both values
        jsonSchema = '';
        jsonSchemaInput = '';
      }
    }
    
    // Validate the schema
    if (jsonSchema) {
      const validation = validateJson(jsonSchema);
      isValid = validation.isValid;
      errorMessage = validation.error;
    }
  }

  // Define the placeholder as a variable to avoid syntax issues
  const placeholderSchema = `{
  "type": "object",
  "properties": {
    "name": { "type": "string" },
    "age": { "type": "number" },
    "interests": { 
      "type": "array",
      "items": { "type": "string" }
    }
  }
}`;
</script>

<Modal bind:show size="md" className="bg-gray-900 rounded-md">
  <div class="p-6">
    <h2 class="text-xl font-bold mb-4 text-white">{$i18n.t('JSON Schema for Structured Output')}</h2>
    
    <div class="mb-4">
      <p class="text-sm text-gray-400 mb-2">
        {$i18n.t('Enter a JSON schema to format the model response. This will be sent to Ollama using the format parameter.')}
      </p>
      <textarea
        class="w-full h-64 bg-gray-800 border border-gray-700 rounded-md p-2 font-mono text-sm text-white"
        bind:value={jsonSchema}
        on:input={(e) => {
          const result = validateJson(e.currentTarget.value);
          isValid = result.isValid;
          errorMessage = result.error;
        }}
        placeholder={placeholderSchema}
      ></textarea>
      {#if !isValid}
        <p class="text-red-500 text-sm mt-1">{errorMessage}</p>
      {/if}
    </div>
    
    <div class="flex justify-end space-x-2">
      <button
        class="px-4 py-2 bg-red-600 hover:bg-red-500 rounded-md text-white"
        on:click={handleClear}
      >
        {$i18n.t('Clear')}
      </button>
      <button
        class="px-4 py-2 bg-gray-700 hover:bg-gray-600 rounded-md text-white"
        on:click={handleClose}
      >
        {$i18n.t('Cancel')}
      </button>
      <button
        class="px-4 py-2 bg-blue-600 hover:bg-blue-500 rounded-md text-white disabled:opacity-50 disabled:cursor-not-allowed"
        on:click={handleSave}
        disabled={!isValid}
      >
        {$i18n.t('Save')}
      </button>
    </div>
  </div>
</Modal> 