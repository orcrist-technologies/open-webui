<script lang="ts">
	import { onDestroy, onMount, tick, getContext, createEventDispatcher } from 'svelte';
	const i18n = getContext('i18n');
	const dispatch = createEventDispatcher();

	import Markdown from './Markdown.svelte';
	import CodeBlock from './CodeBlock.svelte';
	import { chatId, mobile, showArtifacts, showControls, showOverview } from '$lib/stores';
	import FloatingButtons from '../ContentRenderer/FloatingButtons.svelte';
	import { createMessagesList } from '$lib/utils';

	export let id: string;
	export let content: string;
	export let history: any;
	export let model: any = null;
	export let sources: any[] = null;

	export let save: boolean = false;
	export let floatingButtons: boolean = true;

	export let onSourceClick: () => void = () => {};
	export let onTaskClick: () => void = () => {};

	export let onAddMessages: (params: {modelId: string, parentId: string, messages: any[]}) => void = () => {};

	let contentContainerElement: HTMLDivElement;
	let floatingButtonsElement: any;
	
	// Function to check if content is valid JSON
	function isValidJSON(str: string): boolean {
		if (typeof str !== 'string') return false;
		try {
			const result = JSON.parse(str);
			return typeof result === 'object' && result !== null;
		} catch (e) {
			return false;
		}
	}
	
	// Function to format JSON with proper indentation
	function formatJSON(jsonString: string): string {
		try {
			const obj = JSON.parse(jsonString);
			return JSON.stringify(obj, null, 2);
		} catch (e) {
			return jsonString;
		}
	}
	
	// Check if the content is JSON
	let isJSON = false;
	let formattedJSON = '';
	
	$: {
		isJSON = isValidJSON(content);
		if (isJSON) {
			formattedJSON = formatJSON(content);
			console.log('Detected JSON response:', formattedJSON);
		}
	}

	const updateButtonPosition = (event: MouseEvent): void => {
		const buttonsContainerElement = document.getElementById(`floating-buttons-${id}`);
		if (
			!contentContainerElement?.contains(event.target as Node) &&
			!buttonsContainerElement?.contains(event.target as Node)
		) {
			closeFloatingButtons();
			return;
		}

		setTimeout(async () => {
			await tick();

			if (!contentContainerElement?.contains(event.target as Node)) return;

			const selection = window.getSelection();

			if (selection && selection.toString().trim().length > 0) {
				const range = selection.getRangeAt(0);
				const rect = range.getBoundingClientRect();

				const parentRect = contentContainerElement.getBoundingClientRect();

				// Adjust based on parent rect
				const top = rect.bottom - parentRect.top;
				const left = rect.left - parentRect.left;

				if (buttonsContainerElement) {
					buttonsContainerElement.style.display = 'block';

					// Calculate space available on the right
					const spaceOnRight = parentRect.width - left;
					let halfScreenWidth = $mobile ? window.innerWidth / 2 : window.innerWidth / 3;

					if (spaceOnRight < halfScreenWidth) {
						const right = parentRect.right - rect.right;
						buttonsContainerElement.style.right = `${right}px`;
						buttonsContainerElement.style.left = 'auto'; // Reset left
					} else {
						// Enough space, position using 'left'
						buttonsContainerElement.style.left = `${left}px`;
						buttonsContainerElement.style.right = 'auto'; // Reset right
					}
					buttonsContainerElement.style.top = `${top + 5}px`; // +5 to add some spacing
				}
			} else {
				closeFloatingButtons();
			}
		}, 0);
	};

	const closeFloatingButtons = (): void => {
		const buttonsContainerElement = document.getElementById(`floating-buttons-${id}`);
		if (buttonsContainerElement) {
			buttonsContainerElement.style.display = 'none';
		}

		if (floatingButtonsElement) {
			floatingButtonsElement.closeHandler();
		}
	};

	const keydownHandler = (e: KeyboardEvent): void => {
		if (e.key === 'Escape') {
			closeFloatingButtons();
		}
	};

	onMount(() => {
		if (floatingButtons) {
			contentContainerElement?.addEventListener('mouseup', updateButtonPosition as EventListener);
			document.addEventListener('mouseup', updateButtonPosition as EventListener);
			document.addEventListener('keydown', keydownHandler);
		}
	});

	onDestroy(() => {
		if (floatingButtons) {
			contentContainerElement?.removeEventListener('mouseup', updateButtonPosition as EventListener);
			document.removeEventListener('mouseup', updateButtonPosition as EventListener);
			document.removeEventListener('keydown', keydownHandler);
		}
	});
</script>

<div
	class="relative w-full"
	bind:this={contentContainerElement}
	on:mousedown={() => {
		closeFloatingButtons();
	}}
>
	{#if isJSON}
		<div class="json-response">
			<CodeBlock
				id={`json-${id}`}
				lang="json"
				code={formattedJSON}
				{save}
				token={{type: 'code', lang: 'json', text: formattedJSON}}
				onCode={(value) => {
					dispatch('code', value);
				}}
				onSave={(value) => {
					dispatch('update', {
						oldContent: content,
						newContent: value
					});
				}}
			/>
		</div>
	{:else}
		<Markdown
			{id}
			{content}
			{model}
			{save}
			sourceIds={sources?.map((source) => source.id) ?? []}
			{onSourceClick}
			{onTaskClick}
			on:update={(e) => {
				dispatch('update', e.detail);
			}}
			on:code={(e) => {
				dispatch('code', e.detail);
			}}
			on:code-run
			on:code-copy
			on:code-edit
			on:code-insert
			on:table-export
		/>
	{/if}

	{#if floatingButtons}
		<div
			id="floating-buttons-{id}"
			class="absolute z-10"
			style="display: none;"
		>
			<FloatingButtons
				bind:element={floatingButtonsElement}
				modelId={model?.id}
				messages={[{ content: content }]}
				onAdd={(data) => {
					console.log(data.modelId, data.parentId, data.messages);
					dispatch('add-messages', data);
					closeFloatingButtons();
				}}
			/>
		</div>
	{/if}
</div>
