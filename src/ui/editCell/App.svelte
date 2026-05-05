<script>
	import { onMount, tick } from 'svelte';

	let editor
	let urlInput
	let colElement
	let rowElement
	let link = ""

	const alphabet = [
		'', 'A', 'B', 'C', 'D', 'E', 'F', 'G', 'H', 'I', 'J', 'K', 'L', 'M', 'N', 'O', 'P', 'Q', 'R', 'S', 'T', 'U', 'V', 'W', 'X', 'Y', 'Z'
	]

	let data, colIndex, rowIndex, msg;
	let columnType = 'text'
	let options = []
	let urlValue = ''
	let settings = { navigateOnEnter: false }
	let textareaListenersAttached = false
	let documentListenersAttached = false
	let attachedTextarea = null


	function setCaretPosition(ctrl, pos) {
		// Modern browsers
		if (ctrl.setSelectionRange) {
			ctrl.focus();
			ctrl.setSelectionRange(pos, pos);

			// IE8 and below
		} else if (ctrl.createTextRange) {
			var range = ctrl.createTextRange();
			range.collapse(true);
			range.moveEnd('character', pos);
			range.moveStart('character', pos);
			range.select();
		}
	}

	function checkIfLink(text) {
		if (text) {
			// Removed : for now from expression because causes error in figma
			var expression = /^(https?:\/\/)?(www\.)?[-a-zA-Z0-9%._\+~#=]{2,256}\.[a-z]{2,6}\b([-a-zA-Z0-9@:%_\+.~#()?&//=]*)$/gi;
			var regex = new RegExp(expression);
			var t = text;

			if (t.match(regex)) {
				return true
			} else {
				return false
			}
		}

	}

	function postData(payload) {
		parent.postMessage({ pluginMessage: { type: 'data-received', data: Object.assign({ columnType }, payload) } }, '*')
	}

	function postNextCell(target) {
		parent.postMessage({ pluginMessage: { type: 'next-cell', data: { colIndex, rowIndex }, target } }, '*')
	}

	function sendCurrent() {
		if (columnType === 'link') {
			postData({ data: editor ? editor.value : '', url: urlInput ? urlInput.value : '' })
		} else if (columnType === 'dropdown') {
			postData({ data })
		} else {
			let v = editor ? editor.value : ''
			postData({ data: v, link: checkIfLink(v) })
		}
	}

	function updateRows(input) {
		if (!input) return
		input.parentNode.dataset.replicatedValue = input.value
		var textareaHeight = input.clientHeight
		parent.postMessage({ pluginMessage: { type: 'resize-ui', data: { textareaHeight } } }, '*');

		input.addEventListener('input', (e) => {
			input.parentNode.dataset.replicatedValue = input.value
			var textareaHeight = input.clientHeight
			link = checkIfLink(input.value)

			postData({ data: input.value, link })

			parent.postMessage({ pluginMessage: { type: 'resize-ui', data: { textareaHeight } } }, '*');
		})

		input.addEventListener('keydown', (e) => {
			var textareaHeight = input.clientHeight
			link = checkIfLink(input.value)
			if ((e.key === 'Enter' && e.metaKey) || (e.key === 'Enter' && e.ctrlKey)) {
				postData({ data: input.value, link })
			}
			else if (e.key !== 'Enter' && e.which === 37 && e.which === 38 && e.which === 39 && e.which === 38 && e.which === 40) {
				postData({ data: input.value, link })
			}
		})
	}

	function attachTextareaListeners(input, settingsArg) {
		if (!input) return
		if (textareaListenersAttached && attachedTextarea === input) return
		attachedTextarea = input
		textareaListenersAttached = true

		if (settingsArg?.navigateOnEnter) {
			input.addEventListener('keydown', function (e) {
				link = checkIfLink(input.value)

				if (e.key === 'Enter') {
					e.preventDefault()
				}

				if ((e.key === 'Enter' && !e.shiftKey && !e.ctrlKey && !e.metaKey) || ((e.metaKey || e.ctrlKey) && e.which === 40)) {
					postData({ data: input.value, link })
					postNextCell([null, 1])
				}

				if ((e.key === 'Enter' && e.shiftKey) || ((e.metaKey || e.ctrlKey) && e.which === 38)) {
					postData({ data: input.value, link })
					postNextCell([null, -1])
				}

			});
		}

		else {
			input.addEventListener('keydown', function (e) {
				link = checkIfLink(input.value)

				if (e.key === 'Enter') {
					e.preventDefault()
				}

				if (((e.metaKey || e.ctrlKey) && e.which === 40)) {
					postData({ data: input.value, link })
					postNextCell([null, 1])
				}


				if (((e.metaKey || e.ctrlKey) && e.which === 38)) {
					postData({ data: input.value, link })
					postNextCell([null, -1])
				}

			});

			input.addEventListener('keydown', function (e) {
				link = checkIfLink(input.value)

				// User presses enter key?
				if (e.key === 'Enter' && !(e.ctrlKey || e.metaKey)) {
					e.preventDefault()
					postData({ data: input.value, link })
					parent.postMessage({ pluginMessage: { type: 'close-plugin' } }, '*');
				}

			});

		}
	}

	function attachDocumentListeners() {
		if (documentListenersAttached) return
		documentListenersAttached = true

		document.addEventListener('keydown', function (e) {
			// If user presses tab
			if ((e.key === "Tab" && !e.shiftKey) || ((e.metaKey || e.ctrlKey) && e.which === 39)) {
				e.preventDefault()
				sendCurrent()
				postNextCell([1, null])
			}
			if (e.key === "Escape") {
				sendCurrent()
				parent.postMessage({ pluginMessage: { type: 'close-plugin' } }, '*');
			}
		});

		document.addEventListener('keydown', function (e) {
			// If user presses tab and shift key
			if ((e.key === "Tab" && e.shiftKey) || ((e.metaKey || e.ctrlKey) && e.which === 37)) {
				e.preventDefault()
				sendCurrent()
				postNextCell([-1, null])
			}
		});
	}

	function selectOption(label) {
		data = label
		postData({ data: label })
		parent.postMessage({ pluginMessage: { type: 'close-plugin' } }, '*')
	}

	function clearOption() {
		data = ''
		postData({ data: '' })
	}

	async function onLoad(event) {
		msg = event.data.pluginMessage

		if (msg.type === "post-data") {
			let prevType = columnType
			;({ data, colIndex, rowIndex } = msg.data)
			columnType = msg.data.columnType || 'text'
			options = msg.data.options || []
			urlValue = msg.data.link || ''
			if (prevType !== columnType) {
				// editor element will be a different node — reset attach flags
				textareaListenersAttached = false
				attachedTextarea = null
			}
		}

		if (msg.type === "show-ui") {
			settings = msg.settings || { navigateOnEnter: false }
			attachDocumentListeners()
		}

		// Wait for Svelte to render the new UI before grabbing refs
		await tick()

		if (columnType === 'text') {
			if (editor) {
				attachTextareaListeners(editor, settings)
				editor.value = data == null ? '' : data
				editor.focus()
				updateRows(editor)
				setCaretPosition(editor, editor.value.length)
			}
		} else if (columnType === 'link') {
			if (editor) {
				editor.value = data == null ? '' : data
				editor.focus()
				setCaretPosition(editor, editor.value.length)
			}
			if (urlInput) {
				urlInput.value = urlValue || ''
			}
			parent.postMessage({ pluginMessage: { type: 'resize-ui', data: { textareaHeight: 80 } } }, '*');
		} else if (columnType === 'dropdown') {
			parent.postMessage({ pluginMessage: { type: 'resize-ui', data: { textareaHeight: Math.max(60, (options.length + 1) * 36) } } }, '*');
		}

		if (colElement) colElement.innerHTML = `${alphabet[colIndex]}`
		if (rowElement) rowElement.innerHTML = `${rowIndex}`
	}

	function onLinkLabelInput() {
		postData({ data: editor.value, url: urlInput.value })
	}

	function onLinkUrlInput() {
		postData({ data: editor.value, url: urlInput.value })
	}

	onMount(async () => {
		parent.postMessage({ pluginMessage: { type: "window-loaded" } }, '*');
	});




</script>

<svelte:window on:message={onLoad} />

<body>
	<div class="m-xsmall type--small">
		<p>Cell <span id="colElement" bind:this="{colElement}"></span><span id="rowElement" bind:this="{rowElement}"></span></p>

		{#if columnType === 'text'}
			<div class="input grow-wrap">
				<textarea id="editor" bind:this="{editor}" class="textarea" rows="1" style="max-height: 277px; min-height: auto;"></textarea>
			</div>
			<p class="type-small secondary-text">Press <key>ctrl/cmd + enter</key> to create a new line</p>
		{:else if columnType === 'link'}
			<div class="link-fields">
				<label class="field-label">
					<span>Label</span>
					<input class="field-input" type="text" bind:this="{editor}" on:input={onLinkLabelInput} placeholder="Link label" />
				</label>
				<label class="field-label">
					<span>URL</span>
					<input class="field-input" type="text" bind:this="{urlInput}" on:input={onLinkUrlInput} placeholder="https://example.com" />
				</label>
			</div>
		{:else if columnType === 'dropdown'}
			{#if options.length === 0}
				<p class="type-small secondary-text">No options defined. Open the column menu and choose "Manage options…" to add some.</p>
			{:else}
				<div class="dropdown-options">
					{#each options as opt}
						<button
							class="dropdown-option"
							class:selected={data === opt.label}
							style="background: {opt.color.bg}; color: {opt.color.fg};"
							on:click={() => selectOption(opt.label)}>
							{opt.label}
						</button>
					{/each}
					{#if data}
						<button class="clear-option" on:click={clearOption}>Clear selection</button>
					{/if}
				</div>
			{/if}
		{/if}
	</div>
</body>

<style>
	.grow-wrap {
		display: grid;
	}

	.grow-wrap::after {
		content: attr(data-replicated-value) " ";
		white-space: pre-wrap;
		visibility: hidden;
	}

	.grow-wrap>textarea {
		resize: none;
		overflow: hidden;
	}

	.grow-wrap>textarea,
	.grow-wrap::after {
		padding: 8px;
		font: inherit;
		max-height: 250px;
		min-height: auto;
		grid-area: 1 / 1 / 2 / 2;
		letter-spacing: 0.12px;
	}
	.grow-wrap::after {
		margin: 2px 0;
	}

	.link-fields {
		display: flex;
		flex-direction: column;
		gap: 8px;
	}
	.field-label {
		font-size: 11px;
		color: var(--figma-color-text-secondary);
		display: flex;
		flex-direction: column;
		gap: 2px;
	}
	.field-input {
		padding: 6px 8px;
		border: 1px solid var(--figma-color-border);
		border-radius: 2px;
		background: var(--figma-color-bg);
		color: var(--figma-color-text);
		font: inherit;
	}
	.field-input:focus {
		outline: none;
		border-color: var(--figma-color-bg-brand);
	}

	.dropdown-options {
		display: flex;
		flex-direction: column;
		gap: 4px;
		margin-top: 4px;
	}
	.dropdown-option {
		text-align: left;
		padding: 6px 10px;
		border: 2px solid transparent;
		border-radius: 4px;
		font: inherit;
		font-weight: 500;
		cursor: pointer;
	}
	.dropdown-option.selected {
		border-color: var(--figma-color-bg-brand);
	}
	.clear-option {
		margin-top: 4px;
		padding: 4px;
		background: transparent;
		border: none;
		color: var(--figma-color-text-secondary);
		cursor: pointer;
		font: inherit;
		text-align: left;
	}
	.clear-option:hover {
		color: var(--figma-color-text);
	}
</style>
