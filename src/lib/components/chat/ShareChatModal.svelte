<script lang="ts">
	import { getContext, onMount } from 'svelte';

	import { toast } from 'svelte-sonner';
	import { deleteSharedChatById, getChatById, shareChatById } from '$lib/apis/chats';
	import { chatId, modelfiles } from '$lib/stores';
	import { copyToClipboard } from '$lib/utils';

	import Modal from '../common/Modal.svelte';
	import Link from '../icons/Link.svelte';

	let chat = null;
	const i18n = getContext('i18n');

	const shareLocalChat = async () => {
		const _chat = chat;

		const sharedChat = await shareChatById(localStorage.token, $chatId);
		const chatShareUrl = `${window.location.origin}/s/${sharedChat.id}`;

		toast.success($i18n.t('Copied shared chat URL to clipboard!'));
		copyToClipboard(chatShareUrl);
		chat = await getChatById(localStorage.token, $chatId);
	};

	const shareChat = async () => {
		const _chat = chat.chat;
		console.log('share', _chat);

		toast.success($i18n.t('Redirecting you to OpenWebUI Community'));
		const url = 'https://openwebui.com';
		// const url = 'http://localhost:5173';

		const tab = await window.open(`${url}/chats/upload`, '_blank');
		window.addEventListener(
			'message',
			(event) => {
				if (event.origin !== url) return;
				if (event.data === 'loaded') {
					tab.postMessage(
						JSON.stringify({
							chat: _chat,
							modelfiles: $modelfiles.filter((modelfile) =>
								_chat.models.includes(modelfile.tagName)
							)
						}),
						'*'
					);
				}
			},
			false
		);
	};

	export let show = false;

	$: if (show) {
		(async () => {
			if ($chatId) {
				chat = await getChatById(localStorage.token, $chatId);
			} else {
				chat = null;
				console.log(chat);
			}
		})();
	}
</script>

<Modal bind:show size="sm">
	<div>
		<div class=" flex justify-between dark:text-gray-300 px-5 py-4">
			<div class=" text-lg font-medium self-center">{$i18n.t('Share Chat')}</div>
			<button
				class="self-center"
				on:click={() => {
					show = false;
				}}
			>
				<svg
					xmlns="http://www.w3.org/2000/svg"
					viewBox="0 0 20 20"
					fill="currentColor"
					class="w-5 h-5"
				>
					<path
						d="M6.28 5.22a.75.75 0 00-1.06 1.06L8.94 10l-3.72 3.72a.75.75 0 101.06 1.06L10 11.06l3.72 3.72a.75.75 0 101.06-1.06L11.06 10l3.72-3.72a.75.75 0 00-1.06-1.06L10 8.94 6.28 5.22z"
					/>
				</svg>
			</button>
		</div>
		<hr class=" dark:border-gray-800" />

		{#if chat}
			<div class="px-4 pt-4 pb-5 w-full flex flex-col justify-center">
				<div class=" text-sm dark:text-gray-300 mb-1">
					{#if chat.share_id}
						<a href="/s/{chat.share_id}" target="_blank"
							>You have shared this chat <span class=" underline">before</span>.</a
						>
						Click here to
						<button
							class="underline"
							on:click={async () => {
								const res = await deleteSharedChatById(localStorage.token, $chatId);

								if (res) {
									chat = await getChatById(localStorage.token, $chatId);
								}
							}}>delete this link</button
						> and create a new shared link.
					{:else}
						Messages you send after creating your link won't be shared. Users with the URL will be
						able to view the shared chat.
					{/if}
				</div>

				<div class="flex justify-end">
					<div class="flex flex-col items-end space-x-1 mt-1.5">
						<div class="flex gap-1">
							<button
								class=" self-center px-3.5 py-2 rounded-xl text-sm font-medium bg-gray-100 hover:bg-gray-200 text-gray-800 dark:bg-gray-850 dark:hover:bg-gray-800 dark:text-white"
								type="button"
								on:click={() => {
									shareChat();
									show = false;
								}}
							>
								{$i18n.t('Share to OpenWebUI Community')}
							</button>

							<button
								class=" self-center flex items-center gap-1 px-3.5 py-2 rounded-xl text-sm font-medium bg-emerald-600 hover:bg-emerald-500 text-white"
								type="button"
								on:click={() => {
									shareLocalChat();
									show = false;
								}}
							>
								<Link />

								{#if chat.share_id}
									{$i18n.t('Update and Copy Link')}
								{:else}
									{$i18n.t('Copy Link')}
								{/if}
							</button>
						</div>
					</div>
				</div>
			</div>
		{/if}
	</div>
</Modal>
