<script lang="ts">
	import { onMount } from 'svelte';
	import { settings } from '$lib/models/settings';
	import { sideDocText } from '$lib/models/store';

	let editableEl: HTMLDivElement;

	onMount(() => {
		if (editableEl) {
			editableEl.innerHTML = $sideDocText || '';
		}
	});

	function handlePaste(event: ClipboardEvent) {
		event.preventDefault();

		const html = event.clipboardData?.getData('text/html');
		const plain = event.clipboardData?.getData('text/plain') ?? '';

		const selection = window.getSelection();
		if (!selection || selection.rangeCount === 0) {
			updateStore();
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

		updateStore();
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

	function updateStore() {
		if (editableEl) {
			sideDocText.set(editableEl.innerHTML);
		}
	}
</script>

<div class="document">
	<div
		class="text"
		contenteditable="true"
		bind:this={editableEl}
		on:paste={handlePaste}
		on:input={updateStore}
		class:customScrollbar={settings.data.customScrollbar.value}
	></div>
</div>

<style>
	.document {
    background: white;
    width: 100%;
    height: 100%;
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
    padding-bottom: calc(var(--main-height) * 0.6);
}

	.text:empty::before {
		content: 'paste a card here';
		color: var(--text-weak);
	}
</style>