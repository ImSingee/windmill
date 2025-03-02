<script lang="ts">
	import { Button } from '../common'

	import { SUPPORTED_LANGUAGES, copilot } from './lib'
	import type { SupportedLanguage } from '$lib/common'
	import { sendUserToast } from '$lib/toast'
	import type Editor from '../Editor.svelte'
	import Popup from '../common/popup/Popup.svelte'
	import { dbSchemas, copilotInfo, type DBSchema, workspaceStore } from '$lib/stores'
	import type DiffEditor from '../DiffEditor.svelte'
	import { scriptLangToEditorLang } from '$lib/scripts'
	import type SimpleEditor from '../SimpleEditor.svelte'
	import Tooltip from '../Tooltip.svelte'
	import ToggleButtonGroup from '../common/toggleButton-v2/ToggleButtonGroup.svelte'
	import ToggleButton from '../common/toggleButton-v2/ToggleButton.svelte'
	import { writable } from 'svelte/store'
	import { WindmillIcon } from '../icons'
	import HighlightCode from '../HighlightCode.svelte'
	import LoadingIcon from '../apps/svelte-select/lib/LoadingIcon.svelte'
	import { sleep } from '$lib/utils'
	import { autoPlacement } from '@floating-ui/core'
	import { Ban, Bot, Check, ExternalLink, HistoryIcon, Wand2, X } from 'lucide-svelte'
	import { fade } from 'svelte/transition'
	import { isInitialCode } from '$lib/script_helpers'
	import { twMerge } from 'tailwind-merge'

	// props
	export let iconOnly: boolean = false
	export let lang: SupportedLanguage | 'frontend'
	export let editor: Editor | SimpleEditor | undefined
	export let diffEditor: DiffEditor | undefined
	export let inlineScript = false
	export let args: Record<string, any>

	// state
	let funcDesc: string = ''
	let genLoading: boolean = false
	let input: HTMLInputElement | undefined
	let generatedCode = writable<string>('')
	let dbSchema: DBSchema | undefined = undefined
	let abortController: AbortController | undefined = undefined
	let blockPopupOpen = false
	let mode: 'gen' | 'edit' = 'gen'

	let button: HTMLButtonElement | undefined

	async function onGenerate(closePopup: () => void) {
		if (funcDesc.length <= 0) {
			return
		}
		savePrompt()
		try {
			genLoading = true
			blockPopupOpen = true
			abortController = new AbortController()
			if (mode === 'edit') {
				await copilot(
					{
						language: lang,
						description: funcDesc,
						code: editor?.getCode() || '',
						dbSchema: dbSchema,
						type: 'edit',
						workspace: $workspaceStore!
					},
					generatedCode,
					abortController
				)
			} else {
				await copilot(
					{
						language: lang,
						description: funcDesc,
						dbSchema: dbSchema,
						type: 'gen',
						workspace: $workspaceStore!
					},
					generatedCode,
					abortController
				)
			}
			setupDiff()
			diffEditor?.setModified($generatedCode)
			blockPopupOpen = false
			await sleep(500)
			closePopup()
			await sleep(300)
			showDiff()
			funcDesc = ''
		} catch (err) {
			if (!abortController?.signal.aborted) {
				if (err?.message) {
					sendUserToast('Failed to generate code: ' + err.message, true)
				} else {
					sendUserToast('Failed to generate code', true)
					console.error(err)
				}
			}
		} finally {
			genLoading = false
			blockPopupOpen = false
		}
	}

	function acceptDiff() {
		editor?.setCode(diffEditor?.getModified() || '')
		editor?.format()
		clear()
	}

	function rejectDiff() {
		clear()
	}

	function setupDiff() {
		diffEditor?.setupModel(lang === 'frontend' ? 'javascript' : scriptLangToEditorLang(lang))
		diffEditor?.setOriginal(editor?.getCode() || '')
	}

	function showDiff() {
		diffEditor?.show()
		editor?.hide()
	}

	function hideDiff() {
		editor?.show()
		diffEditor?.hide()
	}

	$: input && setTimeout(() => input?.focus(), 100)

	function clear() {
		$generatedCode = ''
	}

	$: lang && clear()

	$: !$generatedCode && hideDiff()

	function updateSchema(lang, args, dbSchemas) {
		const schemaRes = lang === 'graphql' ? args.api : args.database
		if (typeof schemaRes === 'string') {
			const schemaPath = schemaRes.replace('$res:', '')
			if (schemaPath in dbSchemas && dbSchemas[schemaPath].lang === lang) {
				dbSchema = dbSchemas[schemaPath]
			} else {
				dbSchema = undefined
			}
		} else {
			dbSchema = undefined
		}
	}

	$: updateSchema(lang, args, $dbSchemas)

	let codeDiv: HTMLDivElement | undefined
	let lastScrollHeight = 0
	function updateScroll() {
		if (codeDiv && lastScrollHeight !== codeDiv.scrollHeight) {
			codeDiv.scrollTop = codeDiv.scrollHeight
			lastScrollHeight = codeDiv.scrollHeight
		}
	}

	let promptHistory: string[] = []
	function getPromptHistory() {
		try {
			promptHistory = JSON.parse(localStorage.getItem('prompts-' + lang) || '[]')
		} catch (e) {
			console.error('error interacting with local storage', e)
		}
	}

	function savePrompt() {
		if (promptHistory.includes(funcDesc)) {
			return
		}
		promptHistory.unshift(funcDesc)
		while (promptHistory.length > 5) {
			promptHistory.pop()
		}
		try {
			localStorage.setItem('prompts-' + lang, JSON.stringify(promptHistory))
		} catch (e) {
			console.error('error interacting with local storage', e)
		}
	}

	function clearPromptHistory() {
		promptHistory = []
		try {
			localStorage.setItem('prompts-' + lang, JSON.stringify(promptHistory))
		} catch (e) {
			console.error('error interacting with local storage', e)
		}
	}
	$: lang && getPromptHistory()

	$: $generatedCode && updateScroll()
</script>

{#if genLoading}
	<div transition:fade class="fixed z-[4999] inset-0 bg-gray-500/75" />
{/if}

{#if $generatedCode.length > 0 && !genLoading}
	{#if inlineScript}
		<div class="flex gap-1">
			<Button
				title="Discard generated code"
				btnClasses="!font-medium px-2 w-7"
				size="xs"
				color="red"
				on:click={rejectDiff}
				variant="contained"
				startIcon={{ icon: X }}
				iconOnly
			/>
			<Button
				title="Accept generated code"
				btnClasses="!font-medium px-2 w-7"
				size="xs"
				color="green"
				on:click={acceptDiff}
				iconOnly
				startIcon={{ icon: Check }}
			/>
		</div>
	{:else}
		<div class="flex gap-1 px-2">
			<Button
				title="Discard generated code"
				btnClasses="!font-medium px-2"
				size="xs"
				color="red"
				on:click={rejectDiff}
				variant="contained"
				startIcon={{ icon: X }}
				{iconOnly}
			>
				Discard
			</Button><Button
				title="Accept generated code"
				btnClasses="!font-medium px-2"
				size="xs"
				color="green"
				on:click={acceptDiff}
				startIcon={{ icon: Check }}
				{iconOnly}
			>
				Accept
			</Button>
		</div>
	{/if}
{/if}
{#if ($generatedCode.length === 0 || genLoading) && SUPPORTED_LANGUAGES.has(lang)}
	<Popup
		floatingConfig={{
			middleware: [
				autoPlacement({
					allowedPlacements: ['bottom-start', 'bottom-end', 'top-start', 'top-end', 'top', 'bottom']
				})
			]
		}}
		let:close
		blockOpen={blockPopupOpen}
	>
		<svelte:fragment slot="button">
			{#if inlineScript}
				<Button
					size="xs"
					color={genLoading ? 'red' : 'light'}
					btnClasses={genLoading ? '!px-3 z-[5000]' : '!px-2'}
					nonCaptureEvent={!genLoading}
					on:click={genLoading ? () => abortController?.abort() : undefined}
					bind:element={button}
					iconOnly
					title="Generate code from Prompt"
					startIcon={genLoading
						? { icon: Ban }
						: { icon: Wand2, classes: 'text-violet-800 dark:text-violet-400' }}
				/>
			{:else}
				<Button
					title="Generate code from prompt"
					btnClasses={twMerge(
						'!font-medium',
						genLoading ? 'z-[5000]' : 'text-violet-800 dark:text-violet-400'
					)}
					size="xs"
					color={genLoading ? 'red' : 'light'}
					spacingSize="md"
					startIcon={genLoading ? undefined : { icon: Wand2 }}
					propagateEvent
					on:click={genLoading
						? () => abortController?.abort()
						: () => {
								if (editor) {
									if (isInitialCode(editor.getCode())) {
										mode = 'gen'
									} else {
										mode = 'edit'
									}
								}
						  }}
					bind:element={button}
					{iconOnly}
				>
					{#if genLoading}
						<WindmillIcon
							white={true}
							class="mr-1 text-white"
							height="16px"
							width="20px"
							spin="veryfast"
						/>
						Stop
					{:else}
						AI Gen
					{/if}
				</Button>
			{/if}
		</svelte:fragment>
		<div class="block text-primary">
			{#if genLoading}
				<div class="w-[42rem] min-h-[3rem] max-h-[34rem] overflow-y-scroll" bind:this={codeDiv}>
					{#if $generatedCode.length > 0}
						<div class="overflow-x-scroll">
							<HighlightCode language={lang} code={$generatedCode} /></div
						>
					{:else}
						<LoadingIcon />
					{/if}
				</div>
			{:else if $copilotInfo.exists_openai_resource_path}
				<div class="flex flex-col gap-4">
					<div class="flex flex-row justify-between items-center">
						<ToggleButtonGroup class="w-auto shrink-0" bind:selected={mode}>
							<ToggleButton value={'gen'} label="Generate from scratch" small light />
							<ToggleButton value={'edit'} label="Edit existing code" small light />
						</ToggleButtonGroup>

						<div class="text-[0.6rem] text-secondary opacity-60 flex flex-row items-center gap-0.5">
							GPT-4 Turbo<Bot size={14} />
						</div>
					</div>
					<div class="flex w-96">
						<input
							type="text"
							bind:this={input}
							bind:value={funcDesc}
							on:keypress={({ key }) => {
								if (key === 'Enter' && funcDesc.length > 0) {
									onGenerate(() => close(input || null))
								}
							}}
							placeholder={mode === 'edit'
								? 'Describe the changes you want'
								: 'Describe what the script should do'}
						/>
						<Button
							size="xs"
							color="light"
							buttonType="button"
							btnClasses="!p-1 !w-[38px] !ml-2 text-violet-800 dark:text-violet-400 bg-violet-100 dark:bg-gray-700"
							title="Generate code from prompt"
							aria-label="Generate"
							on:click={() => {
								onGenerate(() => close(input || null))
							}}
							disabled={funcDesc.length <= 0}
							iconOnly
							startIcon={{ icon: Wand2 }}
						/>
					</div>
					{#if promptHistory.length > 0}
						<div class="w-96 flex flex-col gap-1">
							{#each promptHistory as p}
								<Button
									size="xs2"
									color="light"
									btnClasses="justify-start overflow-x-scroll no-scrollbar"
									startIcon={{ icon: HistoryIcon, classes: 'shrink-0' }}
									on:click={() => {
										funcDesc = p
									}}>{p}</Button
								>
							{/each}
							<button
								class="underline text-xs text-start px-2 text-secondary font-normal"
								on:click={clearPromptHistory}>clear history</button
							>
						</div>
					{/if}

					{#if ['postgresql', 'mysql', 'snowflake', 'bigquery', 'mssql', 'graphql'].includes(lang) && dbSchema?.lang === lang}
						<div class="flex flex-row items-center justify-between gap-2 w-96">
							<div class="flex flex-row items-center">
								<p class="text-xs text-secondary">
									Context: {lang === 'graphql' ? 'GraphQL' : 'DB'} schema
								</p>
								<Tooltip>
									We pass the selected schema to GPT-4 Turbo for better script generation.
								</Tooltip>
							</div>
							{#if dbSchema.lang !== 'graphql' && (dbSchema.schema?.public || dbSchema.schema?.PUBLIC || dbSchema.schema?.dbo)}
								<ToggleButtonGroup class="w-auto shrink-0" bind:selected={dbSchema.publicOnly}>
									<ToggleButton
										value={true}
										label={(dbSchema.schema?.dbo ? 'Dbo' : 'Public') + ' schema'}
										small
										light
									/>
									<ToggleButton value={false} label="All schemas" small light />
								</ToggleButtonGroup>
							{/if}
						</div>
					{/if}
				</div>
			{:else}
				<p class="text-sm">
					Enable Windmill AI in the <a
						href="/workspace_settings?tab=openai"
						target="_blank"
						class="inline-flex flex-row items-center gap-1"
					>
						workspace settings <ExternalLink size={16} />
					</a>
				</p>
			{/if}
		</div>
	</Popup>
{/if}
