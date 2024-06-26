<script lang="ts">
	import { getDocs } from '$lib/apis/documents';
	import {
		getRAGConfig,
		updateRAGConfig,
		getQuerySettings,
		scanDocs,
		updateQuerySettings,
		resetVectorDB,
		getEmbeddingModel,
		updateEmbeddingModel
	} from '$lib/apis/rag';

	import { documents } from '$lib/stores';
	import { onMount, getContext } from 'svelte';
	import { toast } from 'svelte-sonner';

	import Tooltip from '$lib/components/common/Tooltip.svelte';

	const i18n = getContext('i18n');

	export let saveHandler: Function;

	let scanDirLoading = false;
	let updateEmbeddingModelLoading = false;

	let showResetConfirm = false;

	let chunkSize = 0;
	let chunkOverlap = 0;
	let pdfExtractImages = true;

	let querySettings = {
		template: '',
		k: 4
	};

	let embeddingModel = '';

	const scanHandler = async () => {
		scanDirLoading = true;
		const res = await scanDocs(localStorage.token);
		scanDirLoading = false;

		if (res) {
			await documents.set(await getDocs(localStorage.token));
			toast.success($i18n.t('Scan complete!'));
		}
	};

	const embeddingModelUpdateHandler = async () => {
		if (embeddingModel.split('/').length - 1 > 1) {
			toast.error(
				$i18n.t(
					'Model filesystem path detected. Model shortname is required for update, cannot continue.'
				)
			);
			return;
		}

		console.log('Update embedding model attempt:', embeddingModel);

		updateEmbeddingModelLoading = true;
		const res = await updateEmbeddingModel(localStorage.token, {
			embedding_model: embeddingModel
		}).catch(async (error) => {
			toast.error(error);
			embeddingModel = (await getEmbeddingModel(localStorage.token)).embedding_model;
			return null;
		});
		updateEmbeddingModelLoading = false;

		if (res) {
			console.log('embeddingModelUpdateHandler:', res);
			if (res.status === true) {
				toast.success($i18n.t('Model {{embedding_model}} update complete!', res), {
					duration: 1000 * 10
				});
			}
		}
	};

	const submitHandler = async () => {
		const res = await updateRAGConfig(localStorage.token, {
			pdf_extract_images: pdfExtractImages,
			chunk: {
				chunk_overlap: chunkOverlap,
				chunk_size: chunkSize
			}
		});
		querySettings = await updateQuerySettings(localStorage.token, querySettings);
	};

	onMount(async () => {
		const res = await getRAGConfig(localStorage.token);

		if (res) {
			pdfExtractImages = res.pdf_extract_images;

			chunkSize = res.chunk.chunk_size;
			chunkOverlap = res.chunk.chunk_overlap;
		}

		embeddingModel = (await getEmbeddingModel(localStorage.token)).embedding_model;

		querySettings = await getQuerySettings(localStorage.token);
	});
</script>

<form
	class="flex flex-col h-full justify-between space-y-3 text-sm"
	on:submit|preventDefault={() => {
		submitHandler();
		saveHandler();
	}}
>
	<div class=" space-y-3 pr-1.5 overflow-y-scroll max-h-[22rem]">
		<div>
			<div class=" mb-2 text-sm font-medium">{$i18n.t('General Settings')}</div>

			<div class="  flex w-full justify-between">
				<div class=" self-center text-xs font-medium">
					{$i18n.t('Scan for documents from {{path}}', { path: '/data/docs' })}
				</div>

				<button
					class=" self-center text-xs p-1 px-3 bg-gray-100 dark:bg-gray-800 dark:hover:bg-gray-700 rounded-lg flex flex-row space-x-1 items-center {scanDirLoading
						? ' cursor-not-allowed'
						: ''}"
					on:click={() => {
						scanHandler();
						console.log('check');
					}}
					type="button"
					disabled={scanDirLoading}
				>
					<div class="self-center font-medium">{$i18n.t('Scan')}</div>

					{#if scanDirLoading}
						<div class="ml-3 self-center">
							<svg
								class=" w-3 h-3"
								viewBox="0 0 24 24"
								fill="currentColor"
								xmlns="http://www.w3.org/2000/svg"
								><style>
									.spinner_ajPY {
										transform-origin: center;
										animation: spinner_AtaB 0.75s infinite linear;
									}
									@keyframes spinner_AtaB {
										100% {
											transform: rotate(360deg);
										}
									}
								</style><path
									d="M12,1A11,11,0,1,0,23,12,11,11,0,0,0,12,1Zm0,19a8,8,0,1,1,8-8A8,8,0,0,1,12,20Z"
									opacity=".25"
								/><path
									d="M10.14,1.16a11,11,0,0,0-9,8.92A1.59,1.59,0,0,0,2.46,12,1.52,1.52,0,0,0,4.11,10.7a8,8,0,0,1,6.66-6.61A1.42,1.42,0,0,0,12,2.69h0A1.57,1.57,0,0,0,10.14,1.16Z"
									class="spinner_ajPY"
								/></svg
							>
						</div>
					{/if}
				</button>
			</div>
		</div>

		<hr class=" dark:border-gray-700" />

		<div class="space-y-2">
			<div>
				<div class=" mb-2 text-sm font-medium">{$i18n.t('Update Embedding Model')}</div>
				<div class="flex w-full">
					<div class="flex-1 mr-2">
						<input
							class="w-full rounded-lg py-2 px-4 text-sm dark:text-gray-300 dark:bg-gray-850 outline-none"
							placeholder={$i18n.t('Update embedding model (e.g. {{model}})', {
								model: embeddingModel.slice(-40)
							})}
							bind:value={embeddingModel}
						/>
					</div>
					<button
						class="px-2.5 bg-gray-100 hover:bg-gray-200 text-gray-800 dark:bg-gray-850 dark:hover:bg-gray-800 dark:text-gray-100 rounded-lg transition"
						on:click={() => {
							embeddingModelUpdateHandler();
						}}
						disabled={updateEmbeddingModelLoading}
					>
						{#if updateEmbeddingModelLoading}
							<div class="self-center">
								<svg
									class=" w-4 h-4"
									viewBox="0 0 24 24"
									fill="currentColor"
									xmlns="http://www.w3.org/2000/svg"
									><style>
										.spinner_ajPY {
											transform-origin: center;
											animation: spinner_AtaB 0.75s infinite linear;
										}
										@keyframes spinner_AtaB {
											100% {
												transform: rotate(360deg);
											}
										}
									</style><path
										d="M12,1A11,11,0,1,0,23,12,11,11,0,0,0,12,1Zm0,19a8,8,0,1,1,8-8A8,8,0,0,1,12,20Z"
										opacity=".25"
									/><path
										d="M10.14,1.16a11,11,0,0,0-9,8.92A1.59,1.59,0,0,0,2.46,12,1.52,1.52,0,0,0,4.11,10.7a8,8,0,0,1,6.66-6.61A1.42,1.42,0,0,0,12,2.69h0A1.57,1.57,0,0,0,10.14,1.16Z"
										class="spinner_ajPY"
									/></svg
								>
							</div>
						{:else}
							<svg
								xmlns="http://www.w3.org/2000/svg"
								viewBox="0 0 16 16"
								fill="currentColor"
								class="w-4 h-4"
							>
								<path
									d="M8.75 2.75a.75.75 0 0 0-1.5 0v5.69L5.03 6.22a.75.75 0 0 0-1.06 1.06l3.5 3.5a.75.75 0 0 0 1.06 0l3.5-3.5a.75.75 0 0 0-1.06-1.06L8.75 8.44V2.75Z"
								/>
								<path
									d="M3.5 9.75a.75.75 0 0 0-1.5 0v1.5A2.75 2.75 0 0 0 4.75 14h6.5A2.75 2.75 0 0 0 14 11.25v-1.5a.75.75 0 0 0-1.5 0v1.5c0 .69-.56 1.25-1.25 1.25h-6.5c-.69 0-1.25-.56-1.25-1.25v-1.5Z"
								/>
							</svg>
						{/if}
					</button>
				</div>

				<div class="mt-2 mb-1 text-xs text-gray-400 dark:text-gray-500">
					{$i18n.t(
						'Warning: If you update or change your embedding model, you will need to re-import all documents.'
					)}
				</div>

				<hr class=" dark:border-gray-700 my-3" />

				<div class=" ">
					<div class=" text-sm font-medium">{$i18n.t('Chunk Params')}</div>

					<div class=" flex">
						<div class="  flex w-full justify-between">
							<div class="self-center text-xs font-medium min-w-fit">{$i18n.t('Chunk Size')}</div>

							<div class="self-center p-3">
								<input
									class=" w-full rounded-lg py-1.5 px-4 text-sm dark:text-gray-300 dark:bg-gray-850 outline-none"
									type="number"
									placeholder={$i18n.t('Enter Chunk Size')}
									bind:value={chunkSize}
									autocomplete="off"
									min="0"
								/>
							</div>
						</div>

						<div class="flex w-full">
							<div class=" self-center text-xs font-medium min-w-fit">
								{$i18n.t('Chunk Overlap')}
							</div>

							<div class="self-center p-3">
								<input
									class="w-full rounded-lg py-1.5 px-4 text-sm dark:text-gray-300 dark:bg-gray-850 outline-none"
									type="number"
									placeholder={$i18n.t('Enter Chunk Overlap')}
									bind:value={chunkOverlap}
									autocomplete="off"
									min="0"
								/>
							</div>
						</div>
					</div>

					<div class="pr-2">
						<div class="flex justify-between items-center text-xs">
							<div class=" text-xs font-medium">{$i18n.t('PDF Extract Images (OCR)')}</div>

							<button
								class=" text-xs font-medium text-gray-500"
								type="button"
								on:click={() => {
									pdfExtractImages = !pdfExtractImages;
								}}>{pdfExtractImages ? $i18n.t('On') : $i18n.t('Off')}</button
							>
						</div>
					</div>
				</div>

				<hr class=" dark:border-gray-700 my-3" />

				<div>
					<div class=" text-sm font-medium">{$i18n.t('Query Params')}</div>

					<div class=" flex">
						<div class="  flex w-full justify-between">
							<div class="self-center text-xs font-medium flex-1">{$i18n.t('Top K')}</div>

							<div class="self-center p-3">
								<input
									class=" w-full rounded-lg py-1.5 px-4 text-sm dark:text-gray-300 dark:bg-gray-850 outline-none"
									type="number"
									placeholder={$i18n.t('Enter Top K')}
									bind:value={querySettings.k}
									autocomplete="off"
									min="0"
								/>
							</div>
						</div>
					</div>

					<div>
						<div class=" mb-2.5 text-sm font-medium">{$i18n.t('RAG Template')}</div>
						<textarea
							bind:value={querySettings.template}
							class="w-full rounded-lg px-4 py-3 text-sm dark:text-gray-300 dark:bg-gray-850 outline-none resize-none"
							rows="4"
						/>
					</div>
				</div>

				<hr class=" dark:border-gray-700 my-3" />

				{#if showResetConfirm}
					<div class="flex justify-between rounded-md items-center py-2 px-3.5 w-full transition">
						<div class="flex items-center space-x-3">
							<svg
								xmlns="http://www.w3.org/2000/svg"
								viewBox="0 0 16 16"
								fill="currentColor"
								class="w-4 h-4"
							>
								<path d="M2 3a1 1 0 0 1 1-1h10a1 1 0 0 1 1 1v1a1 1 0 0 1-1 1H3a1 1 0 0 1-1-1V3Z" />
								<path
									fill-rule="evenodd"
									d="M13 6H3v6a2 2 0 0 0 2 2h6a2 2 0 0 0 2-2V6ZM5.72 7.47a.75.75 0 0 1 1.06 0L8 8.69l1.22-1.22a.75.75 0 1 1 1.06 1.06L9.06 9.75l1.22 1.22a.75.75 0 1 1-1.06 1.06L8 10.81l-1.22 1.22a.75.75 0 0 1-1.06-1.06l1.22-1.22-1.22-1.22a.75.75 0 0 1 0-1.06Z"
									clip-rule="evenodd"
								/>
							</svg>
							<span>{$i18n.t('Are you sure?')}</span>
						</div>

						<div class="flex space-x-1.5 items-center">
							<button
								class="hover:text-white transition"
								on:click={() => {
									const res = resetVectorDB(localStorage.token).catch((error) => {
										toast.error(error);
										return null;
									});

									if (res) {
										toast.success($i18n.t('Success'));
									}

									showResetConfirm = false;
								}}
							>
								<svg
									xmlns="http://www.w3.org/2000/svg"
									viewBox="0 0 20 20"
									fill="currentColor"
									class="w-4 h-4"
								>
									<path
										fill-rule="evenodd"
										d="M16.704 4.153a.75.75 0 01.143 1.052l-8 10.5a.75.75 0 01-1.127.075l-4.5-4.5a.75.75 0 011.06-1.06l3.894 3.893 7.48-9.817a.75.75 0 011.05-.143z"
										clip-rule="evenodd"
									/>
								</svg>
							</button>
							<button
								class="hover:text-white transition"
								on:click={() => {
									showResetConfirm = false;
								}}
							>
								<svg
									xmlns="http://www.w3.org/2000/svg"
									viewBox="0 0 20 20"
									fill="currentColor"
									class="w-4 h-4"
								>
									<path
										d="M6.28 5.22a.75.75 0 00-1.06 1.06L8.94 10l-3.72 3.72a.75.75 0 101.06 1.06L10 11.06l3.72 3.72a.75.75 0 101.06-1.06L11.06 10l3.72-3.72a.75.75 0 00-1.06-1.06L10 8.94 6.28 5.22z"
									/>
								</svg>
							</button>
						</div>
					</div>
				{:else}
					<button
						class=" flex rounded-md py-2 px-3.5 w-full hover:bg-gray-200 dark:hover:bg-gray-800 transition"
						on:click={() => {
							showResetConfirm = true;
						}}
					>
						<div class=" self-center mr-3">
							<svg
								xmlns="http://www.w3.org/2000/svg"
								viewBox="0 0 16 16"
								fill="currentColor"
								class="w-4 h-4"
							>
								<path
									fill-rule="evenodd"
									d="M3.5 2A1.5 1.5 0 0 0 2 3.5v9A1.5 1.5 0 0 0 3.5 14h9a1.5 1.5 0 0 0 1.5-1.5v-7A1.5 1.5 0 0 0 12.5 4H9.621a1.5 1.5 0 0 1-1.06-.44L7.439 2.44A1.5 1.5 0 0 0 6.38 2H3.5Zm6.75 7.75a.75.75 0 0 0 0-1.5h-4.5a.75.75 0 0 0 0 1.5h4.5Z"
									clip-rule="evenodd"
								/>
							</svg>
						</div>
						<div class=" self-center text-sm font-medium">{$i18n.t('Reset Vector Storage')}</div>
					</button>
				{/if}
			</div>
		</div>
	</div>
	<div class="flex justify-end pt-3 text-sm font-medium">
		<button
			class=" px-4 py-2 bg-emerald-600 hover:bg-emerald-700 text-gray-100 transition rounded"
			type="submit"
		>
			{$i18n.t('Save')}
		</button>
	</div>
</form>
