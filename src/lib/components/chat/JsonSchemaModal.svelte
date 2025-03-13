<script lang="ts">
  import { createEventDispatcher } from 'svelte';
  import { toast } from 'svelte-sonner';
  import { getContext } from 'svelte';
  import type { Writable } from 'svelte/store';
  import type { i18n as i18nType } from 'i18next';
  
  const i18n: Writable<i18nType> = getContext('i18n');
  const dispatch = createEventDispatcher();
  
  export let show = false;
  export let jsonSchema = '';
  export let chatId = '';
  
  let jsonSchemaInput = jsonSchema;
  let isValid = true;
  let errorMessage = '';
  
  // Debug logs for initialization
  console.log('JsonSchemaModal initialized with schema:', jsonSchema);
  console.log('JsonSchemaModal initialized with chatId:', chatId);
  
  // Load schema from localStorage if available and not already provided
  $: {
    if (chatId && !jsonSchema) {
      const savedSchema = localStorage.getItem(`jsonSchema_${chatId}`);
      if (savedSchema) {
        console.log(`Loading JSON schema from localStorage for chat ${chatId}:`, savedSchema);
        jsonSchemaInput = savedSchema;
        jsonSchema = savedSchema;
        dispatch('change', { jsonSchema: savedSchema });
      } else {
        console.log(`No saved JSON schema found in localStorage for chat ${chatId}`);
      }
    }
  }
  
  function validateJson(json) {
    try {
      if (json.trim() === '') {
        return { isValid: true, error: '' };
      }
      JSON.parse(json);
      return { isValid: true, error: '' };
    } catch (e) {
      return { isValid: false, error: e.message };
    }
  }
  
  function handleSave() {
    const { isValid: valid, error } = validateJson(jsonSchemaInput);
    isValid = valid;
    errorMessage = error;
    
    if (isValid) {
      jsonSchema = jsonSchemaInput;
      
      // Save to localStorage with chat-specific key
      if (chatId) {
        console.log(`Saving JSON schema to localStorage for chat ${chatId}:`, jsonSchema);
        localStorage.setItem(`jsonSchema_${chatId}`, jsonSchema);
      } else {
        console.log('No chatId available, cannot save JSON schema to localStorage');
      }
      
      dispatch('change', { jsonSchema });
      dispatch('save', { jsonSchema });
      dispatch('close');
      show = false;
      toast.success('JSON Schema saved successfully');
    }
  }
  
  function handleClose() {
    show = false;
    dispatch('close');
  }

  function handleKeydown(event: KeyboardEvent) {
    if (event.key === 'Escape') {
      handleClose();
    }
  }

  // When the modal is shown, check if there's a schema in localStorage
  $: if (show && jsonSchema === '') {
    // Try to load from temp storage if no schema is provided
    const tempSchema = localStorage.getItem('temp_jsonSchema');
    if (tempSchema) {
      jsonSchema = tempSchema;
      console.log('Loaded JSON schema from temp storage in modal:', tempSchema);
      validateJson(jsonSchema);
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

<div 
  class={`fixed inset-0 z-50 flex items-center justify-center ${show ? '' : 'hidden'}`}
  on:keydown={handleKeydown}
>
  <div 
    class="absolute inset-0 bg-black/50" 
    on:click={handleClose}
    on:keydown={(e) => e.key === 'Enter' && handleClose()}
    role="button"
    tabindex="0"
    aria-label="Close modal"
  ></div>
  <div class="relative bg-gray-900 rounded-lg shadow-lg w-full max-w-2xl p-6 max-h-[80vh] overflow-auto">
    <h2 class="text-xl font-bold mb-4">{$i18n.t('JSON Schema for Structured Output')}</h2>
    
    <div class="mb-4">
      <p class="text-sm text-gray-400 mb-2">
        {$i18n.t('Enter a JSON schema to format the model response. This will be sent to Ollama using the format parameter.')}
      </p>
      <textarea
        class="w-full h-64 bg-gray-800 border border-gray-700 rounded-md p-2 font-mono text-sm"
        bind:value={jsonSchema}
        on:input={validateJson}
        placeholder={placeholderSchema}
      ></textarea>
      {#if !isValid}
        <p class="text-red-500 text-sm mt-1">{errorMessage}</p>
      {/if}
    </div>
    
    <div class="flex justify-end space-x-2">
      <button
        class="px-4 py-2 bg-gray-700 hover:bg-gray-600 rounded-md"
        on:click={handleClose}
      >
        {$i18n.t('Cancel')}
      </button>
      <button
        class="px-4 py-2 bg-blue-600 hover:bg-blue-500 rounded-md disabled:opacity-50 disabled:cursor-not-allowed"
        on:click={handleSave}
        disabled={!isValid}
      >
        {$i18n.t('Save')}
      </button>
    </div>
  </div>
</div> 