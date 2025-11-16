<script lang="ts">
	import { onMount } from 'svelte';

	// Audio context
	let audioContext: AudioContext | null = null;
	let masterGain: GainNode | null = null;

	// Pattern state
	const STEPS = 16;
	const INSTRUMENTS = [
		{ name: 'Kick', freq: 60, color: 'bg-red-500' },
		{ name: 'Snare', freq: 200, color: 'bg-blue-500' },
		{ name: 'Clap', freq: 1000, color: 'bg-cyan-500' },
		{ name: 'Hi-Hat C', freq: 8000, color: 'bg-yellow-500' },
		{ name: 'Hi-Hat O', freq: 10000, color: 'bg-green-500' },
		{ name: 'Tom Hi', freq: 300, color: 'bg-purple-500' },
		{ name: 'Tom Mid', freq: 150, color: 'bg-pink-500' },
		{ name: 'Tom Low', freq: 80, color: 'bg-orange-500' }
	];

	let pattern = $state<boolean[][]>(
		INSTRUMENTS.map(() => Array(STEPS).fill(false))
	);

	let instrumentVolumes = $state<number[]>(INSTRUMENTS.map(() => 0.7));

	let isPlaying = $state(false);
	let currentStep = $state(0);
	let bpm = $state(120);
	let volume = $state(0.7);

	let intervalId: number | null = null;

	onMount(() => {
		audioContext = new (window.AudioContext || (window as any).webkitAudioContext)();
		masterGain = audioContext.createGain();
		masterGain.gain.value = volume;
		masterGain.connect(audioContext.destination);

		return () => {
			if (intervalId) clearInterval(intervalId);
			if (audioContext) audioContext.close();
		};
	});

	function toggleStep(instrument: number, step: number) {
		pattern[instrument][step] = !pattern[instrument][step];
	}

	function playSound(freq: number, instrument: number) {
		if (!audioContext || !masterGain) return;

		const now = audioContext.currentTime;
		const oscillator = audioContext.createOscillator();
		const gainNode = audioContext.createGain();
		const noiseBuffer = audioContext.createBuffer(1, audioContext.sampleRate * 0.1, audioContext.sampleRate);
		const noiseData = noiseBuffer.getChannelData(0);

		// Apply instrument volume
		const instVolume = instrumentVolumes[instrument];

		// Create different sound types based on instrument
		if (instrument === 0) {
			// Kick - use sine wave with pitch envelope
			oscillator.type = 'sine';
			oscillator.frequency.setValueAtTime(150, now);
			oscillator.frequency.exponentialRampToValueAtTime(freq, now + 0.05);
			gainNode.gain.setValueAtTime(0.8 * instVolume, now);
			gainNode.gain.exponentialRampToValueAtTime(0.01, now + 0.3);
		} else if (instrument === 1 || instrument === 2) {
			// Snare/Clap - use noise with bandpass filter
			for (let i = 0; i < noiseData.length; i++) {
				noiseData[i] = Math.random() * 2 - 1;
			}
			const noiseSource = audioContext.createBufferSource();
			noiseSource.buffer = noiseBuffer;
			
			const filter = audioContext.createBiquadFilter();
			filter.type = 'bandpass';
			filter.frequency.value = freq;
			filter.Q.value = 1;
			
			noiseSource.connect(filter);
			filter.connect(gainNode);
			
			gainNode.gain.setValueAtTime(0.5 * instVolume, now);
			gainNode.gain.exponentialRampToValueAtTime(0.01, now + 0.15);
			
			noiseSource.start(now);
			noiseSource.stop(now + 0.15);
			gainNode.connect(masterGain!);
			return;
		} else if (instrument === 3 || instrument === 4) {
			// Hi-hats - use noise with highpass filter
			for (let i = 0; i < noiseData.length; i++) {
				noiseData[i] = Math.random() * 2 - 1;
			}
			const noiseSource = audioContext.createBufferSource();
			noiseSource.buffer = noiseBuffer;
			
			const filter = audioContext.createBiquadFilter();
			filter.type = 'highpass';
			filter.frequency.value = freq;
			filter.Q.value = 1;
			
			noiseSource.connect(filter);
			filter.connect(gainNode);
			
			const duration = instrument === 3 ? 0.05 : 0.15; // Closed vs Open
			gainNode.gain.setValueAtTime(0.3 * instVolume, now);
			gainNode.gain.exponentialRampToValueAtTime(0.01, now + duration);
			
			noiseSource.start(now);
			noiseSource.stop(now + duration);
			gainNode.connect(masterGain!);
			return;
		} else {
			// Toms - use triangle wave
			oscillator.type = 'triangle';
			oscillator.frequency.value = freq;
			gainNode.gain.setValueAtTime(0.6 * instVolume, now);
			gainNode.gain.exponentialRampToValueAtTime(0.01, now + 0.2);
		}

		oscillator.connect(gainNode);
		gainNode.connect(masterGain!);

		oscillator.start(now);
		oscillator.stop(now + 0.5);
	}

	function step() {
		INSTRUMENTS.forEach((instrument, i) => {
			if (pattern[i][currentStep]) {
				playSound(instrument.freq, i);
			}
		});
		currentStep = (currentStep + 1) % STEPS;
	}

	function togglePlay() {
		if (isPlaying) {
			if (intervalId) clearInterval(intervalId);
			intervalId = null;
			currentStep = 0;
		} else {
			const interval = (60 / bpm / 4) * 1000;
			step();
			intervalId = window.setInterval(step, interval);
		}
		isPlaying = !isPlaying;
	}

	function clearPattern() {
		if (confirm('Are you sure you want to clear the pattern?')) {
			pattern = INSTRUMENTS.map(() => Array(STEPS).fill(false));
		}
	}

	function randomPattern() {
		pattern = INSTRUMENTS.map(() => 
			Array(STEPS).fill(false).map(() => Math.random() > 0.7)
		);
	}

	$effect(() => {
		if (masterGain) {
			masterGain.gain.value = volume;
		}
	});

	$effect(() => {
		if (isPlaying && intervalId) {
			clearInterval(intervalId);
			const interval = (60 / bpm / 4) * 1000;
			intervalId = window.setInterval(step, interval);
		}
	});
</script>

<div class="flex min-h-screen flex-col bg-gray-950 p-6 text-white">
	<div class="mx-auto w-full max-w-7xl">
		<!-- Header -->
		<div class="mb-6 flex items-center justify-between border-b border-gray-800 pb-4">
			<div>
				<h1 class="text-3xl font-bold tracking-wider text-red-500">TR-909</h1>
				<p class="text-sm text-gray-500">RHYTHM COMPOSER</p>
			</div>
			
			<!-- Transport Controls -->
			<div class="flex items-center gap-4">
				<button
					onclick={togglePlay}
					class="flex h-16 w-16 items-center justify-center rounded-full bg-red-600 text-white shadow-lg transition-all hover:bg-red-700 active:scale-95"
				>
					{#if isPlaying}
						<svg class="h-8 w-8" fill="currentColor" viewBox="0 0 24 24">
							<rect x="6" y="4" width="4" height="16" />
							<rect x="14" y="4" width="4" height="16" />
						</svg>
					{:else}
						<svg class="h-8 w-8" fill="currentColor" viewBox="0 0 24 24">
							<path d="M8 5v14l11-7z" />
						</svg>
					{/if}
				</button>

				<div class="flex gap-2">
					<button
						onclick={randomPattern}
						class="rounded-lg bg-gray-800 px-4 py-2 transition-all hover:bg-gray-700"
						aria-label="Random pattern"
					>
						<svg class="h-5 w-5" fill="currentColor" viewBox="0 0 20 20">
							<path d="M5 4a2 2 0 00-2 2v6H2V6a4 4 0 014-4h6v1H6zM15 4h-2v1h2v2h1V5a2 2 0 00-2-2zm0 11h1v-2h-1v2zm-11 1a2 2 0 002 2h6v-1H6v-2H5v1z"/>
						</svg>
					</button>
					<button
						onclick={clearPattern}
						class="rounded-lg bg-gray-800 px-4 py-2 transition-all hover:bg-gray-700"
						aria-label="Clear pattern"
					>
						<svg class="h-5 w-5" fill="currentColor" viewBox="0 0 20 20">
							<path fill-rule="evenodd" d="M9 2a1 1 0 00-.894.553L7.382 4H4a1 1 0 000 2v10a2 2 0 002 2h8a2 2 0 002-2V6a1 1 0 100-2h-3.382l-.724-1.447A1 1 0 0011 2H9zM7 8a1 1 0 012 0v6a1 1 0 11-2 0V8zm5-1a1 1 0 00-1 1v6a1 1 0 102 0V8a1 1 0 00-1-1z" clip-rule="evenodd" />
						</svg>
					</button>
				</div>
			</div>
		</div>

		<!-- Controls -->
		<div class="mb-6 grid grid-cols-2 gap-4 rounded-lg bg-gray-900 p-4">
			<div>
				<label class="mb-2 block text-sm font-medium text-gray-400">BPM</label>
				<div class="flex items-center gap-3">
					<input
						type="range"
						bind:value={bpm}
						min="60"
						max="200"
						class="flex-1"
					/>
					<span class="w-16 text-right font-mono text-lg">{bpm}</span>
				</div>
			</div>
			<div>
				<label class="mb-2 block text-sm font-medium text-gray-400">Master Volume</label>
				<div class="flex items-center gap-3">
					<input
						type="range"
						bind:value={volume}
						min="0"
						max="1"
						step="0.01"
						class="flex-1"
					/>
					<span class="w-16 text-right font-mono text-lg">{Math.round(volume * 100)}%</span>
				</div>
			</div>
		</div>

		<!-- Step Sequencer -->
		<div class="rounded-lg bg-gray-900 p-6">
			<div class="mb-4 flex items-center justify-between">
				<h2 class="text-lg font-semibold">Step Sequencer</h2>
				<div class="flex gap-1">
					{#each Array(4) as _, bar}
						<div class="flex gap-0.5">
							{#each Array(4) as _, beat}
								{@const step = bar * 4 + beat}
								<div
									class="h-2 w-8 rounded-sm {currentStep === step && isPlaying
										? 'bg-red-500'
										: step % 4 === 0
											? 'bg-gray-700'
											: 'bg-gray-800'}"
								></div>
							{/each}
						</div>
					{/each}
				</div>
			</div>

			<!-- Grid -->
			<div class="space-y-2">
				{#each INSTRUMENTS as instrument, instIdx}
					<div class="flex items-center gap-2">
						<!-- Instrument volume -->
						<div class="w-32">
							<input
								type="range"
								bind:value={instrumentVolumes[instIdx]}
								min="0"
								max="1"
								step="0.01"
								class="w-full"
								aria-label="{instrument.name} volume"
							/>
						</div>

						<!-- Instrument label -->
						<div class="w-20 text-right">
							<span class="text-sm font-medium">{instrument.name}</span>
						</div>

						<!-- Steps -->
						<div class="flex flex-1 gap-1">
							{#each Array(4) as _, bar}
								<div class="flex gap-0.5">
									{#each Array(4) as _, beat}
										{@const stepIdx = bar * 4 + beat}
										<button
											onclick={() => toggleStep(instIdx, stepIdx)}
											class="group relative h-10 w-10 rounded transition-all {pattern[instIdx][stepIdx]
												? `${instrument.color} shadow-lg`
												: 'bg-gray-800 hover:bg-gray-700'} {currentStep === stepIdx && isPlaying
												? 'ring-2 ring-white'
												: ''}"
										>
											{#if pattern[instIdx][stepIdx]}
												<div class="absolute inset-0 flex items-center justify-center">
													<div class="h-3 w-3 rounded-full bg-white opacity-90"></div>
												</div>
											{/if}
											{#if stepIdx % 4 === 0}
												<div class="absolute -top-1 left-1/2 h-1 w-1 -translate-x-1/2 rounded-full bg-gray-600"></div>
											{/if}
										</button>
									{/each}
								</div>
							{/each}
						</div>
					</div>
				{/each}
			</div>
		</div>

		<!-- Footer Info -->
		<div class="mt-6 text-center text-xs text-gray-600">
			<p>Use your mouse to tap steps • Adjust BPM and Volume • Press Play to start</p>
		</div>
	</div>
</div>

<style>
	input[type='range'] {
		-webkit-appearance: none;
		appearance: none;
		height: 6px;
		background: #374151;
		border-radius: 3px;
		outline: none;
	}

	input[type='range']::-webkit-slider-thumb {
		-webkit-appearance: none;
		appearance: none;
		width: 20px;
		height: 20px;
		background: #ef4444;
		border-radius: 50%;
		cursor: pointer;
		transition: all 0.15s ease;
	}

	input[type='range']::-webkit-slider-thumb:hover {
		background: #dc2626;
		transform: scale(1.1);
	}

	input[type='range']::-moz-range-thumb {
		width: 20px;
		height: 20px;
		background: #ef4444;
		border: none;
		border-radius: 50%;
		cursor: pointer;
		transition: all 0.15s ease;
	}

	input[type='range']::-moz-range-thumb:hover {
		background: #dc2626;
		transform: scale(1.1);
	}
</style>
