<script lang="ts">
	import { toast } from 'svelte-sonner';
	import { goto } from '$app/navigation';
	import {
		user,
		chats,
		settings,
		showSettings,
		chatId,
		tags,
		showSidebar,
		mobile,
		showArchivedChats
	} from '$lib/stores';
	import { onMount, getContext, tick } from 'svelte';

	const i18n = getContext('i18n');

	import { updateUserSettings } from '$lib/apis/users';
	import {
		deleteChatById,
		getChatList,
		getChatById,
		getChatListByTagName,
		updateChatById,
		getAllChatTags,
		archiveChatById,
		cloneChatById
	} from '$lib/apis/chats';
	import { WEBUI_BASE_URL } from '$lib/constants';

	import ArchivedChatsModal from './Sidebar/ArchivedChatsModal.svelte';
	import UserMenu from './Sidebar/UserMenu.svelte';
	import ChatItem from './Sidebar/ChatItem.svelte';
	import DeleteConfirmDialog from '$lib/components/common/ConfirmDialog.svelte';

	const BREAKPOINT = 768;

	let navElement;
	let search = '';

	let shiftKey = false;

	let selectedChatId = null;
	let deleteChat = null;

	let showDeleteConfirm = false;
	let showDropdown = false;
	let filteredChatList = [];

	$: filteredChatList = $chats.filter((chat) => {
		if (search === '') {
			return true;
		} else {
			let title = chat.title.toLowerCase();
			const query = search.toLowerCase();

			let contentMatches = false;
			// Access the messages within chat.chat.messages
			if (chat.chat && chat.chat.messages && Array.isArray(chat.chat.messages)) {
				contentMatches = chat.chat.messages.some((message) => {
					// Check if message.content exists and includes the search query
					return message.content && message.content.toLowerCase().includes(query);
				});
			}

			return title.includes(query) || contentMatches;
		}
	});

	onMount(async () => {
		mobile.subscribe((e) => {
			if ($showSidebar && e) {
				showSidebar.set(false);
			}

			if (!$showSidebar && !e) {
				showSidebar.set(true);
			}
		});

		showSidebar.set(window.innerWidth > BREAKPOINT);
		await chats.set(await getChatList(localStorage.token));

		let touchstart;
		let touchend;

		function checkDirection() {
			const screenWidth = window.innerWidth;
			const swipeDistance = Math.abs(touchend.screenX - touchstart.screenX);
			if (touchstart.clientX < 40 && swipeDistance >= screenWidth / 8) {
				if (touchend.screenX < touchstart.screenX) {
					showSidebar.set(false);
				}
				if (touchend.screenX > touchstart.screenX) {
					showSidebar.set(true);
				}
			}
		}

		const onTouchStart = (e) => {
			touchstart = e.changedTouches[0];
			console.log(touchstart.clientX);
		};

		const onTouchEnd = (e) => {
			touchend = e.changedTouches[0];
			checkDirection();
		};

		const onKeyDown = (e) => {
			if (e.key === 'Shift') {
				shiftKey = true;
			}
		};

		const onKeyUp = (e) => {
			if (e.key === 'Shift') {
				shiftKey = false;
			}
		};

		const onFocus = () => {};

		const onBlur = () => {
			shiftKey = false;
			selectedChatId = null;
		};

		window.addEventListener('keydown', onKeyDown);
		window.addEventListener('keyup', onKeyUp);

		window.addEventListener('touchstart', onTouchStart);
		window.addEventListener('touchend', onTouchEnd);

		window.addEventListener('focus', onFocus);
		window.addEventListener('blur', onBlur);

		return () => {
			window.removeEventListener('keydown', onKeyDown);
			window.removeEventListener('keyup', onKeyUp);

			window.removeEventListener('touchstart', onTouchStart);
			window.removeEventListener('touchend', onTouchEnd);

			window.removeEventListener('focus', onFocus);
			window.removeEventListener('blur', onBlur);
		};
	});

	// Helper function to fetch and add chat content to each chat
	const enrichChatsWithContent = async (chatList) => {
		const enrichedChats = await Promise.all(
			chatList.map(async (chat) => {
				const chatDetails = await getChatById(localStorage.token, chat.id).catch((error) => null); // Handle error or non-existent chat gracefully
				if (chatDetails) {
					chat.chat = chatDetails.chat; // Assuming chatDetails.chat contains the chat content
				}
				return chat;
			})
		);

		await chats.set(enrichedChats);
	};

	const saveSettings = async (updated) => {
		await settings.set({ ...$settings, ...updated });
		await updateUserSettings(localStorage.token, { ui: $settings });
		location.href = '/';
	};

	const deleteChatHandler = async (id) => {
		const res = await deleteChatById(localStorage.token, id).catch((error) => {
			toast.error(error);
			return null;
		});

		if (res) {
			if ($chatId === id) {
				await chatId.set('');
				await tick();
				goto('/');
			}
			await chats.set(await getChatList(localStorage.token));
		}
	};
</script>

<ArchivedChatsModal
	bind:show={$showArchivedChats}
	on:change={async () => {
		await chats.set(await getChatList(localStorage.token));
	}}
/>

<DeleteConfirmDialog
	bind:show={showDeleteConfirm}
	title={$i18n.t('Delete chat?')}
	on:confirm={() => {
		deleteChatHandler(deleteChat.id);
	}}
>
	<div class=" text-sm text-gray-500">
		{$i18n.t('This will delete')} <span class="  font-semibold">{deleteChat.title}</span>.
	</div>
</DeleteConfirmDialog>

<!-- svelte-ignore a11y-no-static-element-interactions -->

{#if $showSidebar}
	<div
		class=" fixed md:hidden z-40 top-0 right-0 left-0 bottom-0 bg-black/60 w-full min-h-screen h-screen flex justify-center overflow-hidden overscroll-contain"
		on:mousedown={() => {
			showSidebar.set(!$showSidebar);
		}}
	/>
{/if}



<style>
	.scrollbar-hidden:active::-webkit-scrollbar-thumb,
	.scrollbar-hidden:focus::-webkit-scrollbar-thumb,
	.scrollbar-hidden:hover::-webkit-scrollbar-thumb {
		visibility: visible;
	}
	.scrollbar-hidden::-webkit-scrollbar-thumb {
		visibility: hidden;
	}
</style>
