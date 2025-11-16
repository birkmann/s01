<script lang="ts">
	import { onMount } from 'svelte';

	let names = $state<string[]>(['Alice', 'Bob', 'Charlie', 'David', 'Emma', 'Frank']);
	let doneNames = $state<string[]>([]);
	let newName = $state('');
	let bulkNames = $state('');
	let bulkDoneNames = $state('');
	let showBulkInput = $state(false);
	let showBulkDoneInput = $state(false);
	let rotation = $state(0);
	let isSpinning = $state(false);
	let winner = $state<string | null>(null);
	let canvas: HTMLCanvasElement;
	let ctx: CanvasRenderingContext2D | null = null;

	const colors = ['#ef4444', '#f59e0b', '#10b981', '#3b82f6', '#8b5cf6', '#ec4899'];

	onMount(() => {
		ctx = canvas.getContext('2d');
		drawWheel();
	});

	function drawWheel() {
		if (!ctx || names.length === 0) return;

		const centerX = canvas.width / 2;
		const centerY = canvas.height / 2;
		const radius = Math.min(centerX, centerY) - 10;
		const sliceAngle = (2 * Math.PI) / names.length;

		ctx.clearRect(0, 0, canvas.width, canvas.height);

		// Draw slices
		names.forEach((name, index) => {
			const startAngle = index * sliceAngle + (rotation * Math.PI) / 180;
			const endAngle = startAngle + sliceAngle;

			// Draw slice
			ctx!.beginPath();
			ctx!.moveTo(centerX, centerY);
			ctx!.arc(centerX, centerY, radius, startAngle, endAngle);
			ctx!.closePath();
			ctx!.fillStyle = colors[index % colors.length];
			ctx!.fill();
			ctx!.strokeStyle = '#1f2937';
			ctx!.lineWidth = 3;
			ctx!.stroke();

			// Draw text
			ctx!.save();
			ctx!.translate(centerX, centerY);
			ctx!.rotate(startAngle + sliceAngle / 2);
			ctx!.textAlign = 'center';
			ctx!.fillStyle = '#ffffff';
			ctx!.font = 'bold 16px sans-serif';
			ctx!.fillText(name, radius * 0.65, 0);
			ctx!.restore();
		});

		// Draw center circle
		ctx.beginPath();
		ctx.arc(centerX, centerY, 20, 0, 2 * Math.PI);
		ctx.fillStyle = '#1f2937';
		ctx.fill();
		ctx.strokeStyle = '#ffffff';
		ctx.lineWidth = 2;
		ctx.stroke();
	}

	function spinWheel() {
		if (isSpinning || names.length === 0) return;

		// Remove previous winner from names list before spinning
		if (winner) {
			names = names.filter(name => name !== winner);
			winner = null;
			drawWheel();
		}

		isSpinning = true;

		const spinDuration = 3000 + Math.random() * 2000;
		const totalRotation = 360 * 5 + Math.random() * 360;
		const startTime = Date.now();
		const startRotation = rotation;

		function animate() {
			const elapsed = Date.now() - startTime;
			const progress = Math.min(elapsed / spinDuration, 1);

			// Easing function for smooth deceleration
			const easeOut = 1 - Math.pow(1 - progress, 3);
			rotation = (startRotation + totalRotation * easeOut) % 360;

			drawWheel();

			if (progress < 1) {
				requestAnimationFrame(animate);
			} else {
				isSpinning = false;
				selectWinner();
			}
		}

		animate();
	}

	function selectWinner() {
		const sliceAngle = 360 / names.length;
		// The pointer is at the top (270 degrees in our coordinate system)
		// We need to find which slice is pointing up
		const pointerAngle = 270; // Top of the wheel
		const adjustedRotation = (pointerAngle - rotation) % 360;
		const normalizedAngle = adjustedRotation < 0 ? adjustedRotation + 360 : adjustedRotation;
		const winnerIndex = Math.floor(normalizedAngle / sliceAngle) % names.length;
		winner = names[winnerIndex];
		
		// Move winner to done list (will happen on next spin)
		doneNames = [...doneNames, names[winnerIndex]];
	}

	function addName() {
		if (newName.trim() && !isSpinning) {
			names = [...names, newName.trim()];
			newName = '';
			drawWheel();
		}
	}

	function removeName(index: number) {
		if (!isSpinning && names.length > 1) {
			names = names.filter((_, i) => i !== index);
			drawWheel();
		}
	}

	function handleKeyPress(event: KeyboardEvent) {
		if (event.key === 'Enter') {
			addName();
		}
	}

	function uploadBulkNames() {
		if (!bulkNames.trim() || isSpinning) return;
		
		const newNames = bulkNames
			.split('\n')
			.map(name => name.trim())
			.filter(name => name.length > 0);
		
		if (newNames.length > 0) {
			names = [...names, ...newNames];
			bulkNames = '';
			showBulkInput = false;
			drawWheel();
		}
	}

	function uploadBulkDoneNames() {
		if (!bulkDoneNames.trim()) return;
		
		const newDoneNames = bulkDoneNames
			.split('\n')
			.map(name => name.trim())
			.filter(name => name.length > 0);
		
		if (newDoneNames.length > 0) {
			doneNames = [...doneNames, ...newDoneNames];
			bulkDoneNames = '';
			showBulkDoneInput = false;
		}
	}

	function clearAllNames() {
		if (!isSpinning) {
			names = [];
			drawWheel();
		}
	}

	function removeDoneName(index: number) {
		doneNames = doneNames.filter((_, i) => i !== index);
	}

	function clearAllDoneNames() {
		doneNames = [];
	}

	$effect(() => {
		drawWheel();
	});
</script>

<div class="flex min-h-screen flex-col bg-gray-950 p-6 text-white">
	<div class="mx-auto w-full max-w-6xl">
		<h1 class="mb-8 text-center text-2xl font-bold">Wheel of Names</h1>

		<!-- Wheel -->
		<div class="mb-8 flex flex-col items-center">
			<div class="relative">
				<!-- Pointer -->
				<div
					class="absolute left-1/2 top-0 z-10 h-0 w-0 -translate-x-1/2 border-x-15 border-t-30 border-x-transparent border-t-red-500"
				></div>

				<canvas
					bind:this={canvas}
					width="400"
					height="400"
					class="rounded-full shadow-2xl"
				></canvas>
			</div>

		<button
			onclick={spinWheel}
			disabled={isSpinning || names.length === 0}
			class="mt-6 rounded-lg bg-blue-600 px-8 py-3 text-lg font-semibold transition-all hover:bg-blue-700 disabled:cursor-not-allowed disabled:opacity-50"
		>
			{isSpinning ? 'Spinning...' : 'Spin the Wheel'}
		</button>

		<div class="mt-4 h-24">
			{#if winner}
				<div class="rounded-lg bg-green-900/50 px-6 py-4 text-center">
					<p class="text-sm uppercase tracking-wider text-green-400">Winner</p>
					<p class="mt-1 text-2xl font-bold text-green-300">{winner}</p>
				</div>
			{/if}
		</div>
	</div>		<!-- Two Lists -->
		<div class="grid gap-6 lg:grid-cols-2">
			<!-- Names List -->
			<div class="flex flex-col overflow-hidden">
				<div class="mb-4 flex items-center justify-between">
					<h2 class="text-xl font-semibold">Names ({names.length})</h2>
					<div class="flex gap-2">
						<button
							onclick={() => showBulkInput = !showBulkInput}
							disabled={isSpinning}
							class="rounded-lg bg-gray-800 p-2 transition-all hover:bg-gray-700 disabled:cursor-not-allowed disabled:opacity-50"
							aria-label="Toggle bulk input"
						>
							<svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" viewBox="0 0 20 20" fill="currentColor">
								<path fill-rule="evenodd" d="M3 4a1 1 0 011-1h12a1 1 0 110 2H4a1 1 0 01-1-1zm0 4a1 1 0 011-1h12a1 1 0 110 2H4a1 1 0 01-1-1zm0 4a1 1 0 011-1h12a1 1 0 110 2H4a1 1 0 01-1-1zm0 4a1 1 0 011-1h12a1 1 0 110 2H4a1 1 0 01-1-1z" clip-rule="evenodd" />
							</svg>
						</button>
						<button
							onclick={clearAllNames}
							disabled={isSpinning || names.length === 0}
							class="rounded-lg bg-gray-800 p-2 transition-all hover:bg-gray-700 disabled:cursor-not-allowed disabled:opacity-50"
							aria-label="Clear all names"
						>
							<svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" viewBox="0 0 20 20" fill="currentColor">
								<path fill-rule="evenodd" d="M9 2a1 1 0 00-.894.553L7.382 4H4a1 1 0 000 2v10a2 2 0 002 2h8a2 2 0 002-2V6a1 1 0 100-2h-3.382l-.724-1.447A1 1 0 0011 2H9zM7 8a1 1 0 012 0v6a1 1 0 11-2 0V8zm5-1a1 1 0 00-1 1v6a1 1 0 102 0V8a1 1 0 00-1-1z" clip-rule="evenodd" />
							</svg>
						</button>
					</div>
				</div>

				{#if showBulkInput}
					<div class="mb-4 space-y-2">
						<textarea
							bind:value={bulkNames}
							placeholder="Paste names here (one per line)&#10;John&#10;Jane&#10;Alice&#10;Bob"
							rows="6"
							class="w-full rounded-lg bg-gray-800 px-4 py-2 text-white placeholder-gray-500 focus:outline-none focus:ring-2 focus:ring-gray-600"
						></textarea>
						<button
							onclick={uploadBulkNames}
							disabled={!bulkNames.trim() || isSpinning}
							class="flex w-full items-center justify-center gap-2 rounded-lg bg-gray-800 px-4 py-2 transition-all hover:bg-gray-700 disabled:cursor-not-allowed disabled:opacity-50"
							aria-label="Upload names"
						>
							<svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" viewBox="0 0 20 20" fill="currentColor">
								<path fill-rule="evenodd" d="M3 17a1 1 0 011-1h12a1 1 0 110 2H4a1 1 0 01-1-1zM6.293 6.707a1 1 0 010-1.414l3-3a1 1 0 011.414 0l3 3a1 1 0 01-1.414 1.414L11 5.414V13a1 1 0 11-2 0V5.414L7.707 6.707a1 1 0 01-1.414 0z" clip-rule="evenodd" />
							</svg>
							<span>Upload</span>
						</button>
					</div>
				{/if}

				<div class="mb-4 flex gap-2">
					<input
						type="text"
						bind:value={newName}
						onkeypress={handleKeyPress}
						placeholder="Enter a name"
						class="flex-1 rounded-lg bg-gray-800 px-4 py-2 text-white placeholder-gray-500 focus:outline-none focus:ring-2 focus:ring-gray-600"
					/>
					<button
						onclick={addName}
						disabled={!newName.trim() || isSpinning}
						class="rounded-lg bg-gray-800 p-2 transition-all hover:bg-gray-700 disabled:cursor-not-allowed disabled:opacity-50"
						aria-label="Add name"
					>
						<svg xmlns="http://www.w3.org/2000/svg" class="h-6 w-6" viewBox="0 0 20 20" fill="currentColor">
							<path fill-rule="evenodd" d="M10 5a1 1 0 011 1v3h3a1 1 0 110 2h-3v3a1 1 0 11-2 0v-3H6a1 1 0 110-2h3V6a1 1 0 011-1z" clip-rule="evenodd" />
						</svg>
					</button>
				</div>

				<div class="flex-1 space-y-2 overflow-y-auto rounded-lg bg-gray-900 p-4">
					{#each names as name, index}
						<div
							class="flex items-center justify-between rounded-lg bg-gray-800 px-4 py-3 transition-all hover:bg-gray-700"
						>
							<span class="font-medium">{name}</span>
							<button
								onclick={() => removeName(index)}
								disabled={isSpinning || names.length <= 1}
								class="text-gray-400 transition-colors hover:text-gray-300 disabled:cursor-not-allowed disabled:opacity-30"
								aria-label="Remove {name}"
							>
								<svg
									xmlns="http://www.w3.org/2000/svg"
									class="h-5 w-5"
									viewBox="0 0 20 20"
									fill="currentColor"
								>
									<path
										fill-rule="evenodd"
										d="M4.293 4.293a1 1 0 011.414 0L10 8.586l4.293-4.293a1 1 0 111.414 1.414L11.414 10l4.293 4.293a1 1 0 01-1.414 1.414L10 11.414l-4.293 4.293a1 1 0 01-1.414-1.414L8.586 10 4.293 5.707a1 1 0 010-1.414z"
										clip-rule="evenodd"
									/>
								</svg>
							</button>
						</div>
					{/each}
				</div>
			</div>

			<!-- Done List -->
			<div class="flex flex-col">
				<div class="mb-4 flex items-center justify-between">
					<h2 class="text-xl font-semibold">Done ({doneNames.length})</h2>
					<div class="flex gap-2">
						<button
							onclick={() => showBulkDoneInput = !showBulkDoneInput}
							class="rounded-lg bg-gray-800 p-2 transition-all hover:bg-gray-700"
							aria-label="Toggle bulk input"
						>
							<svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" viewBox="0 0 20 20" fill="currentColor">
								<path fill-rule="evenodd" d="M3 4a1 1 0 011-1h12a1 1 0 110 2H4a1 1 0 01-1-1zm0 4a1 1 0 011-1h12a1 1 0 110 2H4a1 1 0 01-1-1zm0 4a1 1 0 011-1h12a1 1 0 110 2H4a1 1 0 01-1-1zm0 4a1 1 0 011-1h12a1 1 0 110 2H4a1 1 0 01-1-1z" clip-rule="evenodd" />
							</svg>
						</button>
						<button
							onclick={clearAllDoneNames}
							disabled={doneNames.length === 0}
							class="rounded-lg bg-gray-800 p-2 transition-all hover:bg-gray-700 disabled:cursor-not-allowed disabled:opacity-50"
							aria-label="Clear all done names"
						>
							<svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" viewBox="0 0 20 20" fill="currentColor">
								<path fill-rule="evenodd" d="M9 2a1 1 0 00-.894.553L7.382 4H4a1 1 0 000 2v10a2 2 0 002 2h8a2 2 0 002-2V6a1 1 0 100-2h-3.382l-.724-1.447A1 1 0 0011 2H9zM7 8a1 1 0 012 0v6a1 1 0 11-2 0V8zm5-1a1 1 0 00-1 1v6a1 1 0 102 0V8a1 1 0 00-1-1z" clip-rule="evenodd" />
							</svg>
						</button>
					</div>
				</div>

				{#if showBulkDoneInput}
					<div class="mb-4 space-y-2">
						<textarea
							bind:value={bulkDoneNames}
							placeholder="Paste done names here (one per line)&#10;John&#10;Jane&#10;Alice&#10;Bob"
							rows="6"
							class="w-full rounded-lg bg-gray-800 px-4 py-2 text-white placeholder-gray-500 focus:outline-none focus:ring-2 focus:ring-gray-600"
						></textarea>
						<button
							onclick={uploadBulkDoneNames}
							disabled={!bulkDoneNames.trim()}
							class="flex w-full items-center justify-center gap-2 rounded-lg bg-gray-800 px-4 py-2 transition-all hover:bg-gray-700 disabled:cursor-not-allowed disabled:opacity-50"
							aria-label="Upload names"
						>
							<svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" viewBox="0 0 20 20" fill="currentColor">
								<path fill-rule="evenodd" d="M3 17a1 1 0 011-1h12a1 1 0 110 2H4a1 1 0 01-1-1zM6.293 6.707a1 1 0 010-1.414l3-3a1 1 0 011.414 0l3 3a1 1 0 01-1.414 1.414L11 5.414V13a1 1 0 11-2 0V5.414L7.707 6.707a1 1 0 01-1.414 0z" clip-rule="evenodd" />
							</svg>
							<span>Upload</span>
						</button>
					</div>
				{/if}

				<div class="flex-1 space-y-2 overflow-y-auto rounded-lg bg-gray-900 p-4">
					{#each doneNames as name, index}
						<div
							class="flex items-center justify-between rounded-lg bg-gray-800 px-4 py-3 transition-all hover:bg-gray-700"
						>
						<span class="font-medium {index === doneNames.length - 1 ? 'text-green-400' : ''}">{name}</span>
						<div class="flex gap-2">
							<button
								onclick={() => {
									names = [...names, name];
									doneNames = doneNames.filter((_, i) => i !== index);
									drawWheel();
								}}
								class="text-gray-400 transition-colors hover:text-gray-300"
								aria-label="Move {name} back to names"
							>
								<svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" viewBox="0 0 20 20" fill="currentColor">
									<path fill-rule="evenodd" d="M7.707 3.293a1 1 0 010 1.414L5.414 7H11a7 7 0 017 7v2a1 1 0 11-2 0v-2a5 5 0 00-5-5H5.414l2.293 2.293a1 1 0 11-1.414 1.414l-4-4a1 1 0 010-1.414l4-4a1 1 0 011.414 0z" clip-rule="evenodd" />
								</svg>
							</button>
							<button
								onclick={() => removeDoneName(index)}
								class="text-gray-400 transition-colors hover:text-gray-300"
								aria-label="Remove {name}"
							>
								<svg
									xmlns="http://www.w3.org/2000/svg"
									class="h-5 w-5"
									viewBox="0 0 20 20"
									fill="currentColor"
								>
									<path
										fill-rule="evenodd"
										d="M4.293 4.293a1 1 0 011.414 0L10 8.586l4.293-4.293a1 1 0 111.414 1.414L11.414 10l4.293 4.293a1 1 0 01-1.414 1.414L10 11.414l-4.293 4.293a1 1 0 01-1.414-1.414L8.586 10 4.293 5.707a1 1 0 010-1.414z"
										clip-rule="evenodd"
									/>
								</svg>
							</button>
						</div>
						</div>
					{/each}
				</div>
			</div>
		</div>
	</div>
</div>
