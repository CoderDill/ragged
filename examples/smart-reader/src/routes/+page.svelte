<script lang="ts">
	import type { SearchResult } from '$lib/types';
	import { onMount } from 'svelte';
	import { Ragged, t } from '../../../../ragged/main';
	import { searchWiki } from '$lib/api/wiki';
	import type { EventHistory } from '$lib/components/Console.types';
	import Console from '$lib/components/Console.svelte';

	// STATE

	let r: Ragged;
	let commandInput = '';
	let assistantOutput: string | undefined = undefined;
	let results: SearchResult[] = [];
	let isProcessing = false;

	let history: EventHistory[] = [];

	// ACTIONS

	const doSearch = async (text: string) => {
		try {
			results = await searchWiki(text);
			history = [
				...history,
				{
					type: 'wiki-search',
					data: {
						titles: results.map((r) => r.title)
					}
				}
			];
		} catch (e) {
			console.error('Error while searching', e);
		}
	};

	// RAGGED ENTRYPOINT

	const processCommand = async (command: string) => {
		if (isProcessing) return;

		history = [
			...history,
			{
				type: 'chat',
				data: {
					sender: 'user',
					message: command
				}
			}
		];

		isProcessing = true;
		console.log('command', command);
		console.log('results', results);

		let input: string =
			'# Command\n\nExecute the user command below, taking the Currently Displayed Results into account.';
		input += `# User Command\n\n${command}\n\n`;
		input += `# Currently Displayed Results\n\n${JSON.stringify(results)}\n\n`;

		const p$ = r.chat(input, {
			tools,
			requestOverrides: {
				model: 'gpt-4-turbo'
			}
		});

		// HANDLING RAGGED EVENTS

		p$.subscribe({
			next: (s) => {
				history = [
					...history,
					{
						type: 'ragged',
						data: s
					}
				];

				if (s.type === 'finished') {
					assistantOutput = undefined;
					if (s.data.length > 0) {
						history = [
							...history,
							{
								type: 'chat',
								data: {
									sender: 'ai',
									message: s.data
								}
							}
						];
					}
				}

				// handle tools
				if (s.type === 'tool.inputs') {
					if (s.data.name === 'search-wikipedia') {
						doSearch(s.data.arguments.searchTerm);
					}
				}

				// output the assistant response
				if (s.type === 'text.joined') {
					assistantOutput = s.data;
				}
			},
			complete: () => {
				isProcessing = false;
			}
		});
	};

	// LLM TOOLS
	const tools = [
		t
			.tool()
			.title('search-wikipedia')
			.description(
				'Search Wikipedia for a term. This will return a list of results. It will also display the results to the user. You must only call this function when you are explicitly asked to run a search.'
			)
			.inputs({
				searchTerm: t.string().description('The term to search for on Wikipedia.').isRequired()
			})
	];

	// SETUP

	onMount(() => {
		r = new Ragged({
			provider: 'openai',
			config: {
				apiKey: import.meta.env.VITE_OPENAI_CREDS,
				dangerouslyAllowBrowser: true
			}
		});
	});
</script>

<div class="h-full w-full p-4 flex flex-row gap-2">
	<div class="flex flex-col gap-4 bg-base-200 p-4 border border-accent w-2/5">
		<h1 class="text-xl font-bold">Smart Reader</h1>
		<div class="flex flex-col gap-4 max-h-[90vh] overflow-scroll">
			<div class="flex flex-col gap-4">
				<form
					class="form w-3xl flex flex-col gap-2"
					on:submit={(e) => processCommand(commandInput)}
				>
					<label for="command-ai" class="label font-bold">Command AI</label>
					<input
						id="command-ai"
						name="command-ai"
						class="input input-bordered"
						placeholder="Command AI"
						bind:value={commandInput}
						disabled={isProcessing}
					/>
					<input
						type="submit"
						class="btn btn-primary w-fit"
						value={isProcessing ? '⌛' : 'Command'}
					/>
				</form>
			</div>
			<div class="flex flex-col">
				<h2 class="text-lg font-bold">Assistant Output</h2>
				<!-- <pre class="whitespace-pre-wrap">{assistantOutput}</pre> -->
				<Console {history} incomingStream={assistantOutput} />
			</div>
		</div>
	</div>
	<div class="flex flex-col w-full gap-4 border border-accent p-4 bg-base-200">
		<div class="flex flex-col gap-4 max-h-[90vh] overflow-scroll">
			{#if results.length === 0}
				<p>No results found</p>
			{:else}
				{#each results as result}
					<div class="border-2 border-base-300 p-4">
						<h2 class="text-lg font-bold">{result.title}</h2>
						<p>{result.extract}</p>
					</div>
				{/each}
			{/if}
		</div>
	</div>
</div>
