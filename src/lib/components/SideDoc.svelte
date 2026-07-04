<script lang="ts">
	import { onMount, tick } from 'svelte';
	import { settings } from '$lib/models/settings';
	import { sideDocText } from '$lib/models/store';

	type CardTab = {
		id: string;
		name: string;
		content: string;
	};

	let tabs: CardTab[] = [];
	let activeTabId: string = '';
	let editableEl: HTMLDivElement;
	let renamingTabId: string | null = null;

	function newId(): string {
		return Math.random().toString(36).slice(2, 10);
	}

	onMount(() => {
		const raw = $sideDocText || '';
		try {
			const parsed = JSON.parse(raw);
			if (Array.isArray(parsed) && parsed.length > 0) {
				tabs = parsed;
			} else {
				throw new Error('not a tab array');
			}
		} catch {
			// legacy plain HTML from before tabs existed — migrate it into one tab
			tabs = [{ id: newId(), name: 'Card 1', content: raw }];
		}
		activeTabId = tabs[0].id;
		loadActiveTabIntoEditor();
	});

	function loadActiveTabIntoEditor() {
		const tab = tabs.find((t) => t.id === activeTabId);
		if (editableEl) {
			editableEl.innerHTML = tab?.content ?? '';
		}
	}

	function saveEditorIntoActiveTab() {
		const tab = tabs.find((t) => t.id === activeTabId);
		if (tab && editableEl) {
			tab.content = editableEl.innerHTML;
		}
	}

	function persist() {
		saveEditorIntoActiveTab();
		sideDocText.set(JSON.stringify(tabs));
		tabs = tabs; // trigger reactivity for tab list (e.g. renamed titles)
	}

	async function selectTab(id: string) {
		if (id === activeTabId) return;
		saveEditorIntoActiveTab();
		activeTabId = id;
		await tick();
		loadActiveTabIntoEditor();
		persist();
	}

	async function addTab() {
		saveEditorIntoActiveTab();
		const tab: CardTab = { id: newId(), name: `Card ${tabs.length + 1}`, content: '' };
		tabs = [...tabs, tab];
		activeTabId = tab.id;
		await tick();
		loadActiveTabIntoEditor();
		persist();
	}

	async function removeTab(id: string, event: MouseEvent) {
		event.stopPropagation();
		const wasActive = id === activeTabId;
		tabs = tabs.filter((t) => t.id !== id);
		if (tabs.length === 0) {
			tabs = [{ id: newId(), name: 'Card 1', content: '' }];
		}
		if (wasActive) {
			activeTabId = tabs[0].id;
			await tick();
			loadActiveTabIntoEditor();
		}
		persist();
	}

	function startRename(id: string, event: MouseEvent) {
	event.stopPropagation();
	renamingTabId = id;
}

function commitRename(id: string, newName: string) {
	const tab = tabs.find((t) => t.id === id);
	if (tab) {
		tab.name = newName.trim() || tab.name;
	}
	renamingTabId = null;
	persist();
}

function handleRenameBlur(id: string, event: FocusEvent) {
	const input = event.target as HTMLInputElement;
	commitRename(id, input.value);
}

function handleRenameKeydown(event: KeyboardEvent) {
	if (event.key === 'Enter') {
		(event.target as HTMLInputElement).blur();
	}
}

	function handlePaste(event: ClipboardEvent) {
		event.preventDefault();

		const html = event.clipboardData?.getData('text/html');
		const plain = event.clipboardData?.getData('text/plain') ?? '';

		const selection = window.getSelection();
		if (!selection || selection.rangeCount === 0) {
			persist();
			return;
		}
		const range = selection.getRangeAt(0);
		range.deleteContents();

		let lastNode: Node | null = null;

		if (html) {
			const fragment = sanitizeHtmlToFragment(html);
			lastNode = fragment.lastChild;
			range.insertNode(fragment);
		} else {
			const textNode = document.createTextNode(plain);
			lastNode = textNode;
			range.insertNode(textNode);
		}

		if (lastNode) {
			const newRange = document.createRange();
			newRange.setStartAfter(lastNode);
			newRange.collapse(true);
			selection.removeAllRanges();
			selection.addRange(newRange);
		}

		persist();
	}

	function isNearWhiteOrTransparent(color: string): boolean {
		if (!color) return true;
		if (color === 'transparent' || color === 'rgba(0, 0, 0, 0)') return true;
		const match = color.match(/\d+(\.\d+)?/g);
		if (!match) return false;
		const [r, g, b, a] = match.map(Number);
		if (a === 0) return true;
		return r > 240 && g > 240 && b > 240;
	}

	function sanitizeHtmlToFragment(html: string): DocumentFragment {
		const doc = new DOMParser().parseFromString(html, 'text/html');
		const allowedTags = new Set(['B', 'STRONG', 'I', 'EM', 'U', 'SPAN', 'BR', 'DIV', 'P']);

		function clean(node: Node) {
			[...node.childNodes].forEach((child) => {
				if (child.nodeType === 1) {
					const el = child as HTMLElement;

					if (!allowedTags.has(el.tagName)) {
						while (el.firstChild) el.parentNode?.insertBefore(el.firstChild, el);
						el.parentNode?.removeChild(el);
						return;
					}

					const weight = el.style.fontWeight;
					const decoration = el.style.textDecoration;
					const fontStyle = el.style.fontStyle;
					const bgColor = el.style.backgroundColor;

					el.removeAttribute('style');
					el.removeAttribute('class');
					el.removeAttribute('id');

					const isBold = weight === 'bold' || (!!weight && parseInt(weight) >= 600);
					el.style.setProperty('font-weight', isBold ? 'bold' : 'normal', 'important');

					if (decoration && decoration.includes('underline')) {
						el.style.setProperty('text-decoration', 'underline', 'important');
					}
					if (fontStyle === 'italic') {
						el.style.setProperty('font-style', 'italic', 'important');
					}

					const isRealHighlight = !isNearWhiteOrTransparent(bgColor);

					if (isRealHighlight) {
						el.style.setProperty('background-color', bgColor, 'important');
						el.style.setProperty('color', 'black', 'important');
					} else {
						el.style.setProperty('color', 'black', 'important');
					}

					clean(el);
				}
			});
		}

		clean(doc.body);

		const fragment = document.createDocumentFragment();
		[...doc.body.childNodes].forEach((node) => {
			fragment.appendChild(document.importNode(node, true));
		});
		return fragment;
	}
</script>

<div class="panel">
	<div class="tab-bar">
		{#each tabs as tab (tab.id)}
			<div
				class="tab"
				class:active={tab.id === activeTabId}
				on:click={() => selectTab(tab.id)}
				on:dblclick={(e) => startRename(tab.id, e)}
			>
				{#if renamingTabId === tab.id}
					<input
    class="tab-rename-input"
    value={tab.name}
    autofocus
    on:click={(e) => e.stopPropagation()}
    on:blur={(e) => handleRenameBlur(tab.id, e)}
    on:keydown={handleRenameKeydown}
/>
				{:else}
					<span class="tab-name">{tab.name}</span>
					<button class="tab-close" on:click={(e) => removeTab(tab.id, e)}>×</button>
				{/if}
			</div>
		{/each}
		<button class="add-tab" on:click={addTab}>+</button>
	</div>

	<div class="document">
		<div
			class="text"
			contenteditable="true"
			bind:this={editableEl}
			on:paste={handlePaste}
			on:input={persist}
			class:customScrollbar={settings.data.customScrollbar.value}
		></div>
	</div>
</div>

<style>
	.panel {
		display: flex;
		flex-direction: column;
		width: 100%;
		height: 100%;
		box-sizing: border-box;
	}

	.tab-bar {
		display: flex;
		align-items: center;
		gap: 4px;
		overflow-x: auto;
		flex-shrink: 0;
		padding-bottom: 6px;
	}

	.tab {
		display: flex;
		align-items: center;
		gap: 4px;
		padding: 4px 8px;
		border-radius: var(--border-radius);
		background: var(--background);
		color: var(--text-weak);
		cursor: pointer;
		white-space: nowrap;
		font-size: 0.8rem;
		flex-shrink: 0;
	}

	.tab.active {
		background: white;
		color: black;
		font-weight: bold;
	}

	.tab-close {
		background: none;
		border: none;
		cursor: pointer;
		color: inherit;
		font-size: 0.9rem;
		line-height: 1;
		padding: 0 2px;
	}

	.tab-rename-input {
		font-size: 0.8rem;
		border: none;
		outline: none;
		background: white;
		color: black;
		width: 80px;
	}

	.add-tab {
		background: var(--background);
		color: var(--text-weak);
		border: none;
		border-radius: var(--border-radius);
		cursor: pointer;
		padding: 4px 10px;
		font-size: 0.9rem;
		flex-shrink: 0;
	}

	.document {
		background: white;
		width: 100%;
flex: 1;
		min-height: 0;
		border-radius: var(--border-radius);
		box-sizing: border-box;
	}
	.text {
		height: 100%;
		width: 100%;

		outline: none;
		margin: none;

		overflow-y: scroll;

		line-height: 1.5em;
		font-size: 0.9rem;
		color: black;

		background: none;
		border: none;
		text-decoration: inherit;
		box-sizing: border-box;

		padding: var(--padding);
	}

	.text:empty::before {
		content: 'paste a card here';
		color: #888;
	}
</style>