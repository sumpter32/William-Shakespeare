<script lang="ts">
	import ChatMessage from '$lib/components/ChatMessage.svelte'
	import type { ChatCompletionRequestMessage } from 'openai'
	import { SSE } from 'sse.js'

	let query: string = ''
	let answer: string = ''
	let loading: boolean = false
	let chatMessages: ChatCompletionRequestMessage[] = []
	let scrollToDiv: HTMLDivElement

	function scrollToBottom() {
		setTimeout(function () {
			scrollToDiv.scrollIntoView({ behavior: 'smooth', block: 'end', inline: 'nearest' })
		}, 100)
	}

	const handleSubmit = async () => {
		loading = true
		chatMessages = [...chatMessages, { role: 'user', content: query }]

		const eventSource = new SSE('/api/chat', {
			headers: {
				'Content-Type': 'application/json'
			},
			payload: JSON.stringify({ messages: chatMessages })
		})

		query = ''

		eventSource.addEventListener('error', handleError)

		eventSource.addEventListener('message', (e) => {
			scrollToBottom()
			try {
				loading = false
				if (e.data === '[DONE]') {
					chatMessages = [...chatMessages, { role: 'assistant', content: answer }]
					answer = ''
					return
				}

				const completionResponse = JSON.parse(e.data)
				const [{ delta }] = completionResponse.choices

				if (delta.content) {
					answer = (answer ?? '') + delta.content
				}
			} catch (err) {
				handleError(err)
			}
		})
		eventSource.stream()
		scrollToBottom()
	}

	function handleError<T>(err: T) {
		loading = false
		query = ''
		answer = ''
		console.error(err)
	}
</script>
<style>
	.parent-container {
      height: 100vh;
     }
	 
	 .chat-container {
	  height: calc(99vh - 3.5rem); /* Subtract the height of the input area */
	  padding-bottom: env(safe-area-inset-bottom); /* Add padding for the virtual keyboard on mobile devices */
	}
	.input-area {
	  position: fixed;
	  bottom: 0;
	  left: 0;
	  right: 0;
	  padding: 0 env(safe-area-inset-left) 0 env(safe-area-inset-right); /* Add padding for the virtual keyboard on mobile devices */
	}
	
  </style>
<div class="flex flex-col w-full px-0 items-center h-full parent-container">
  <div class="flex flex-col w-full px-0 items-center h-full">
	<div class="chat-container w-full bg-gray-900 rounded-md p-4 overflow-y-auto flex flex-col gap-4">
	  <div class="flex flex-col gap-2">
		<ChatMessage type="assistant" message="What do you want!" />
		{#each chatMessages as message}
		  <ChatMessage type={message.role} message={message.content} />
		{/each}
		{#if answer}
		  <ChatMessage type="assistant" message={answer} />
		{/if}
		{#if loading}
		  <ChatMessage type="assistant" message="Loading.." />
		{/if}
	  </div>
	  <div class="" bind:this={scrollToDiv} />
	</div>
	<form
	  class="input-area flex w-full rounded-md gap-4 bg-gray-900 p-4"
	  on:submit|preventDefault={() => handleSubmit()}
	>
	  <input type="text" class="input input-bordered flex-grow" bind:value={query} />
	  <button type="submit" class="btn btn-accent"> Send </button>
	</form>
  </div>
</div>