<script lang="ts">
	import { onMount } from 'svelte';

	// Audio context
	let audioContext: AudioContext | null = null;
	let masterGain: GainNode | null = null;
	let delayNode: DelayNode | null = null;
	let delayFeedback: GainNode | null = null;
	let delayMix: GainNode | null = null;
	let reverbNode: ConvolverNode | null = null;
	let reverbMix: GainNode | null = null;

	// Pattern state
	const STEPS = 16;
	const INSTRUMENTS = [
		{ name: 'Kick', freq: 45, color: 'bg-red-500' },
		{ name: 'Snare', freq: 200, color: 'bg-blue-500' },
		{ name: 'Clap', freq: 1000, color: 'bg-cyan-500' },
		{ name: 'Hi-Hat C', freq: 8000, color: 'bg-yellow-500' },
		{ name: 'Hi-Hat O', freq: 10000, color: 'bg-green-500' },
		{ name: 'Tom Hi', freq: 300, color: 'bg-purple-500' },
		{ name: 'Tom Mid', freq: 150, color: 'bg-pink-500' },
		{ name: 'Tom Low', freq: 80, color: 'bg-orange-500' }
	];

	// Synth notes (C2-B2)
	const SYNTH_NOTES = [
		{ name: 'C', freq: 65.41 },
		{ name: 'C#', freq: 69.30 },
		{ name: 'D', freq: 73.42 },
		{ name: 'D#', freq: 77.78 },
		{ name: 'E', freq: 82.41 },
		{ name: 'F', freq: 87.31 },
		{ name: 'F#', freq: 92.50 },
		{ name: 'G', freq: 98.00 },
		{ name: 'G#', freq: 103.83 },
		{ name: 'A', freq: 110.00 },
		{ name: 'A#', freq: 116.54 },
		{ name: 'B', freq: 123.47 }
	];

	// Chord types with interval patterns (semitones from root)
	const CHORD_TYPES = [
		{ name: 'Major', intervals: [0, 4, 7] },
		{ name: 'Minor', intervals: [0, 3, 7] },
		{ name: 'Sus2', intervals: [0, 2, 7] },
		{ name: 'Sus4', intervals: [0, 5, 7] },
		{ name: 'Dim', intervals: [0, 3, 6] },
		{ name: 'Aug', intervals: [0, 4, 8] },
		{ name: 'Maj7', intervals: [0, 4, 7, 11] },
		{ name: 'Min7', intervals: [0, 3, 7, 10] }
	];

	// Basic Channel "Quadrant Dub" inspired default pattern
	let pattern = $state<boolean[][]>([
		[true, false, false, false, true, false, false, false, true, false, false, false, true, false, false, false], // Kick: Four-on-floor
		[false, false, false, false, false, false, false, false, false, false, false, false, false, false, false, false], // Snare: Silent
		[false, false, false, false, false, false, false, false, false, false, false, false, false, false, false, false], // Clap: Silent
		[false, true, false, true, false, true, false, true, false, true, false, true, false, true, false, true], // Hi-Hat C: Sparse offbeats
		[false, false, false, false, false, false, false, false, false, false, false, false, false, false, false, false], // Hi-Hat O: Silent
		[false, false, false, false, false, false, false, false, false, false, false, false, false, false, false, false], // Tom Hi: Silent
		[false, false, false, false, false, false, false, false, false, false, false, false, false, false, false, false], // Tom Mid: Silent
		[false, false, false, false, false, false, false, false, false, false, false, false, false, false, false, false]  // Tom Low: Silent
	]);

	let instrumentVolumes = $state<number[]>([0.8, 0.7, 0.7, 0.3, 0.7, 0.7, 0.7, 0.7]); // Quieter hats for subtlety
	let instrumentMuted = $state<boolean[]>(INSTRUMENTS.map(() => false));
	let instrumentPitch = $state<number[]>([0.7, 1.0, 1.0, 1.0, 1.0, 1.0, 1.0, 1.0]); // Lower kick pitch

	// Synth pattern state - Basic Channel minimal chord progression
	let synthPattern = $state<number[]>([0, -1, 0, -1, 0, -1, 0, -1, 7, -1, 7, -1, 0, -1, 0, -1]); // Root to Fifth sparse pattern
	let synthChordType = $state(1); // Minor chord (typical Basic Channel)
	let synthVolume = $state(0.2); // Subtle synth volume
	let synthMuted = $state(false);
	let synthDelay = $state(0.65); // Heavy delay for dub atmosphere
	let synthReverb = $state(0.75); // Deep reverb for space

	let isPlaying = $state(false);
	let currentStep = $state(0);
	let bpm = $state(122); // Classic Basic Channel tempo
	let volume = $state(0.7);

	let intervalId: number | null = null;

	// Song management - 16 pattern slots
	type Song = {
		name: string;
		pattern: boolean[][];
		instrumentVolumes: number[];
		instrumentMuted: boolean[];
		instrumentPitch: number[];
		synthPattern: number[];
		synthChordType: number;
		synthVolume: number;
		synthMuted: boolean;
		synthDelay: number;
		synthReverb: number;
		bpm: number;
		volume: number;
	};

	let patternSlots = $state<(Song | null)[]>(Array(16).fill(null));
	let activePatternIndex = $state<number>(0);
	let queuedPatternIndex = $state<number>(-1);
	let copyMode = $state(false);
	let copySourceIndex = $state<number>(-1);
	let songMode = $state(false);
	let loopCounter = $state(0);
	let fileInput: HTMLInputElement | null = null;

	onMount(() => {
		audioContext = new (window.AudioContext || (window as any).webkitAudioContext)();
		masterGain = audioContext.createGain();
		masterGain.gain.value = volume;
		masterGain.connect(audioContext.destination);

		// Create delay effect - Basic Channel style
		delayNode = audioContext.createDelay(1.0);
		delayNode.delayTime.value = 0.369; // Dotted eighth note at 122 BPM
		delayFeedback = audioContext.createGain();
		delayFeedback.gain.value = 0.5; // More feedback for dub echoes
		delayMix = audioContext.createGain();
		delayMix.gain.value = 0;

		delayNode.connect(delayFeedback);
		delayFeedback.connect(delayNode);
		delayNode.connect(delayMix);
		delayMix.connect(masterGain);

		// Create reverb effect (simple impulse response)
		reverbNode = audioContext.createConvolver();
		reverbMix = audioContext.createGain();
		reverbMix.gain.value = 0;

		// Generate simple impulse response for reverb
		const sampleRate = audioContext.sampleRate;
		const length = sampleRate * 2; // 2 second reverb
		const impulse = audioContext.createBuffer(2, length, sampleRate);
		const impulseL = impulse.getChannelData(0);
		const impulseR = impulse.getChannelData(1);

		for (let i = 0; i < length; i++) {
			const decay = Math.exp(-i / (sampleRate * 0.5));
			impulseL[i] = (Math.random() * 2 - 1) * decay;
			impulseR[i] = (Math.random() * 2 - 1) * decay;
		}

		reverbNode.buffer = impulse;
		reverbNode.connect(reverbMix);
		reverbMix.connect(masterGain);

		return () => {
			if (intervalId) clearInterval(intervalId);
			if (audioContext) audioContext.close();
		};
	});

	function toggleStep(instrument: number, step: number) {
		pattern[instrument][step] = !pattern[instrument][step];
	}

	function toggleMute(instrument: number) {
		instrumentMuted[instrument] = !instrumentMuted[instrument];
	}

	function toggleSynthNote(step: number, noteIndex: number) {
		if (synthPattern[step] === noteIndex) {
			synthPattern[step] = -1; // Remove note
		} else {
			synthPattern[step] = noteIndex; // Set note
		}
	}

	function playSound(freq: number, instrument: number) {
		if (!audioContext || !masterGain) return;

		const now = audioContext.currentTime;
		const oscillator = audioContext.createOscillator();
		const gainNode = audioContext.createGain();
		const noiseBuffer = audioContext.createBuffer(1, audioContext.sampleRate * 0.1, audioContext.sampleRate);
		const noiseData = noiseBuffer.getChannelData(0);

		// Apply instrument volume and pitch
		const instVolume = instrumentVolumes[instrument];
		const pitchMultiplier = instrumentPitch[instrument];
		const adjustedFreq = freq * pitchMultiplier;

		// Create different sound types based on instrument
		if (instrument === 0) {
			// Kick - deep sine wave with slower pitch envelope for Basic Channel sound
			oscillator.type = 'sine';
			oscillator.frequency.setValueAtTime(120 * pitchMultiplier, now);
			oscillator.frequency.exponentialRampToValueAtTime(adjustedFreq, now + 0.08);
			gainNode.gain.setValueAtTime(0.9 * instVolume, now);
			gainNode.gain.exponentialRampToValueAtTime(0.01, now + 0.5);
		} else if (instrument === 1 || instrument === 2) {
			// Snare/Clap - use noise with bandpass filter
			for (let i = 0; i < noiseData.length; i++) {
				noiseData[i] = Math.random() * 2 - 1;
			}
			const noiseSource = audioContext.createBufferSource();
			noiseSource.buffer = noiseBuffer;
			
			const filter = audioContext.createBiquadFilter();
			filter.type = 'bandpass';
			filter.frequency.value = adjustedFreq;
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
			filter.frequency.value = adjustedFreq;
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
			oscillator.frequency.value = adjustedFreq;
			gainNode.gain.setValueAtTime(0.6 * instVolume, now);
			gainNode.gain.exponentialRampToValueAtTime(0.01, now + 0.2);
		}

		oscillator.connect(gainNode);
		gainNode.connect(masterGain!);

		oscillator.start(now);
		oscillator.stop(now + 0.5);
	}

	function playSynthChord(noteIndex: number) {
		if (!audioContext || !masterGain || synthMuted) return;
		if (!delayNode || !delayMix || !reverbNode || !reverbMix) return;

		const now = audioContext!.currentTime;
		const baseFreq = SYNTH_NOTES[noteIndex].freq;
		const chordType = CHORD_TYPES[synthChordType];

		// Create a dry gain node to mix dry signal
		const dryGain = audioContext.createGain();
		dryGain.gain.value = 1 - Math.max(synthDelay, synthReverb) * 0.5; // Reduce dry when effects are active
		dryGain.connect(masterGain);

		// Play each note in the chord
		chordType.intervals.forEach(interval => {
			const freq = baseFreq * Math.pow(2, interval / 12);
			const oscillator = audioContext!.createOscillator();
			const gainNode = audioContext!.createGain();

			oscillator.type = 'sawtooth';
			oscillator.frequency.value = freq;

			// ADSR envelope - slower, more sustained for dub atmosphere
			gainNode.gain.setValueAtTime(0, now);
			gainNode.gain.linearRampToValueAtTime(synthVolume * 0.25, now + 0.05); // Slower attack
			gainNode.gain.linearRampToValueAtTime(synthVolume * 0.18, now + 0.15); // Decay
			gainNode.gain.linearRampToValueAtTime(synthVolume * 0.15, now + 0.8); // Long sustain
			gainNode.gain.linearRampToValueAtTime(0, now + 1.0); // Longer release

			oscillator.connect(gainNode);
			
			// Route to dry
			gainNode.connect(dryGain);
			
			// Route to delay if enabled
			if (synthDelay > 0) {
				const delayGain = audioContext!.createGain();
				delayGain.gain.value = synthDelay;
				gainNode.connect(delayGain);
				delayGain.connect(delayNode!);
			}
			
			// Route to reverb if enabled
			if (synthReverb > 0) {
				const reverbGain = audioContext!.createGain();
				reverbGain.gain.value = synthReverb;
				gainNode.connect(reverbGain);
				reverbGain.connect(reverbNode!);
			}

			oscillator.start(now);
			oscillator.stop(now + 1.0);
		});
	}

	function step() {
		INSTRUMENTS.forEach((instrument, i) => {
			if (pattern[i][currentStep] && !instrumentMuted[i]) {
				playSound(instrument.freq, i);
			}
		});

		// Play synth if note is set
		if (synthPattern[currentStep] !== -1) {
			playSynthChord(synthPattern[currentStep]);
		}

		currentStep = (currentStep + 1) % STEPS;
		
		// Check for pattern switch at 4-bar boundary (every 16 steps)
		if (currentStep === 0) {
			if (songMode) {
				// Song mode: count loops and advance to next pattern after 4 loops
				loopCounter++;
				if (loopCounter >= 4) {
					loopCounter = 0;
					// Find next available pattern
					const nextIndex = findNextAvailablePattern(activePatternIndex);
					if (nextIndex !== activePatternIndex) {
						// Auto-save current pattern before switching
						saveCurrentPatternToSlot(activePatternIndex);
						
						if (patternSlots[nextIndex]) {
							loadPattern(nextIndex);
						} else {
							clearPatternData();
						}
						activePatternIndex = nextIndex;
					}
				}
			} else if (queuedPatternIndex !== -1) {
				// Manual pattern switch
				loopCounter = 0; // Reset loop counter
				// Auto-save current pattern before switching
				saveCurrentPatternToSlot(activePatternIndex);
				
				const queuedIndex = queuedPatternIndex;
				queuedPatternIndex = -1;
				if (patternSlots[queuedIndex]) {
					loadPattern(queuedIndex);
				} else {
					// Switch to empty pattern
					clearPatternData();
				}
				activePatternIndex = queuedIndex;
			}
		}
	}

	function findNextAvailablePattern(currentIndex: number): number {
		// Find next filled pattern slot, wrapping around
		for (let i = 1; i <= 16; i++) {
			const nextIndex = (currentIndex + i) % 16;
			if (patternSlots[nextIndex]) {
				return nextIndex;
			}
		}
		// If no other patterns found, stay on current
		return currentIndex;
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
		if (confirm('Clear all drums and synth patterns?')) {
			pattern = INSTRUMENTS.map(() => Array(STEPS).fill(false));
			synthPattern = Array(STEPS).fill(-1);
		}
	}

	function clearDrumPattern() {
		if (confirm('Clear drum pattern?')) {
			pattern = INSTRUMENTS.map(() => Array(STEPS).fill(false));
		}
	}

	function clearSynthPattern() {
		if (confirm('Are you sure you want to clear the synth pattern?')) {
			synthPattern = Array(STEPS).fill(-1);
		}
	}

	function randomDrumPattern() {
		// Detroit techno-inspired patterns
		pattern = INSTRUMENTS.map((_, instIdx) => {
			const steps = Array(STEPS).fill(false);
			
			if (instIdx === 0) {
				// Kick: Four-on-the-floor with occasional variations
				for (let i = 0; i < STEPS; i += 4) {
					steps[i] = true;
					if (Math.random() > 0.7) steps[i + 2] = true; // Occasional off-beat
				}
			} else if (instIdx === 1) {
				// Snare: Backbeat on 2 and 4 with variations
				steps[4] = true;
				steps[12] = true;
				if (Math.random() > 0.6) steps[6] = true;
				if (Math.random() > 0.6) steps[14] = true;
			} else if (instIdx === 2) {
				// Clap: Sparse accents
				if (Math.random() > 0.5) steps[8] = true;
			} else if (instIdx === 3) {
				// Hi-Hat Closed: Steady 16th pattern with gaps
				for (let i = 0; i < STEPS; i++) {
					if (i % 2 === 1) steps[i] = true; // Off-beats
					else if (Math.random() > 0.6) steps[i] = true; // Some on-beats
				}
			} else if (instIdx === 4) {
				// Hi-Hat Open: Sparse openings for groove
				for (let i = 0; i < STEPS; i++) {
					if (i % 4 === 2 && Math.random() > 0.5) steps[i] = true;
				}
			} else if (instIdx === 5 || instIdx === 6 || instIdx === 7) {
				// Toms: Sparse fills and accents
				for (let i = 0; i < STEPS; i++) {
					if (Math.random() > 0.85) steps[i] = true;
				}
			}
			
			return steps;
		});
		
		// Randomize volumes with kick, snare, clap, and hats dominating
		instrumentVolumes = INSTRUMENTS.map((_, instIdx) => {
			if (instIdx === 0) {
				// Kick: High volume (0.75-0.95)
				return 0.75 + Math.random() * 0.2;
			} else if (instIdx === 1) {
				// Snare: High volume (0.7-0.9)
				return 0.7 + Math.random() * 0.2;
			} else if (instIdx === 2) {
				// Clap: High volume (0.65-0.85)
				return 0.65 + Math.random() * 0.2;
			} else if (instIdx === 3 || instIdx === 4) {
				// Hats: High volume (0.7-0.9)
				return 0.7 + Math.random() * 0.2;
			} else {
				// Toms: Lower volume (0.4-0.6)
				return 0.4 + Math.random() * 0.2;
			}
		});
	}

	function randomPattern() {
		randomDrumPattern();
		randomSynthPattern();
	}

	function randomSynthPattern() {
		// Create dub techno-style chord progressions
		synthPattern = Array(STEPS).fill(-1);
		
		// Dub techno typically uses minor and sus chords
		const dubChords = [1, 2, 3]; // Minor, Sus2, Sus4
		synthChordType = dubChords[Math.floor(Math.random() * dubChords.length)];
		
		// Randomize delay (30-80%) and reverb (40-90%) for dub atmosphere
		synthDelay = 0.3 + Math.random() * 0.5;
		synthReverb = 0.4 + Math.random() * 0.5;
		
		// Simple chord progressions typical of dub techno (max 6 active steps)
		const progressions = [
			// Two-chord progression (4 steps)
			[0, -1, -1, -1, 7, -1, -1, -1, 0, -1, -1, -1, 7, -1, -1, -1], // Root to Fifth
			// Minimal one-chord (4 steps)
			[0, -1, 0, -1, -1, -1, -1, -1, 0, -1, 0, -1, -1, -1, -1, -1], // Sparse root
			// Hypnotic two-note pattern (4 steps)
			[0, -1, -1, -1, 5, -1, -1, -1, 0, -1, -1, -1, 5, -1, -1, -1], // Very sparse
			// Classic dub pattern (6 steps)
			[0, 0, -1, -1, 7, -1, -1, -1, 0, 0, -1, -1, 7, -1, -1, -1], // Root-Fifth with rests
			// Three-chord minimal (6 steps)
			[0, -1, 5, -1, 7, -1, -1, -1, 0, -1, 5, -1, 7, -1, -1, -1], // Root, Fourth, Fifth
			// Ultra minimal (3 steps)
			[0, -1, -1, -1, -1, -1, -1, -1, 7, -1, -1, -1, -1, -1, -1, -1] // Root and Fifth only
		];
		
		const progression = progressions[Math.floor(Math.random() * progressions.length)];
		synthPattern = [...progression];
	}

	function saveSong() {
		const name = prompt('Enter pattern name:');
		if (!name) return;
		
		const song: Song = {
			name,
			pattern: pattern.map(row => [...row]),
			instrumentVolumes: [...instrumentVolumes],
			instrumentMuted: [...instrumentMuted],
			instrumentPitch: [...instrumentPitch],
			synthPattern: [...synthPattern],
			synthChordType,
			synthVolume,
			synthMuted,
			synthDelay,
			synthReverb,
			bpm,
			volume
		};
		
		// Find first empty slot or use current slot
		let slotIndex = activePatternIndex;
		const emptySlot = patternSlots.findIndex(slot => slot === null);
		if (emptySlot !== -1) {
			slotIndex = emptySlot;
		}
		
		patternSlots[slotIndex] = song;
		activePatternIndex = slotIndex;
	}

	function saveProject() {
		const name = prompt('Enter project name:');
		if (!name) return;
		
		// Auto-save current pattern first
		saveCurrentPatternToSlot(activePatternIndex);
		
		const project = {
			name,
			version: '1.0',
			patterns: patternSlots,
			activePatternIndex
		};
		
		// Save to localStorage
		try {
			localStorage.setItem(`groovebox-project-${name}`, JSON.stringify(project));
			alert(`Project "${name}" saved successfully!`);
		} catch (e) {
			alert('Failed to save project: ' + e);
		}
	}

	function loadPattern(index: number) {
		const song = patternSlots[index];
		if (!song) return;
		
		pattern = song.pattern.map(row => [...row]);
		instrumentVolumes = [...song.instrumentVolumes];
		instrumentMuted = [...song.instrumentMuted];
		instrumentPitch = [...song.instrumentPitch];
		synthPattern = [...song.synthPattern];
		synthChordType = song.synthChordType;
		synthVolume = song.synthVolume;
		synthMuted = song.synthMuted;
		synthDelay = song.synthDelay;
		synthReverb = song.synthReverb;
		bpm = song.bpm;
		volume = song.volume;
	}

	function selectPattern(index: number) {
		// Auto-save current pattern before switching (even if not yet saved)
		saveCurrentPatternToSlot(activePatternIndex);
		
		if (isPlaying) {
			// Queue the pattern change for next 4-bar boundary
			queuedPatternIndex = index;
		} else {
			// Switch immediately if not playing
			if (patternSlots[index]) {
				loadPattern(index);
			} else {
				// Switch to empty pattern - clear current pattern
				clearPatternData();
			}
			activePatternIndex = index;
		}
	}

	function saveCurrentPatternToSlot(index: number) {
		// Check if pattern has any content
		const hasContent = pattern.some(row => row.some(step => step)) || 
		                   synthPattern.some(note => note !== -1);
		
		if (!hasContent && !patternSlots[index]) {
			// Empty pattern and not yet saved - don't save
			return;
		}
		
		// Get existing name or create default name
		const existingSong = patternSlots[index];
		const name = existingSong ? existingSong.name : `Pattern ${index + 1}`;
		
		patternSlots[index] = {
			name,
			pattern: pattern.map(row => [...row]),
			instrumentVolumes: [...instrumentVolumes],
			instrumentMuted: [...instrumentMuted],
			instrumentPitch: [...instrumentPitch],
			synthPattern: [...synthPattern],
			synthChordType,
			synthVolume,
			synthMuted,
			synthDelay,
			synthReverb,
			bpm,
			volume
		};
	}

	function clearPatternData() {
		pattern = INSTRUMENTS.map(() => Array(STEPS).fill(false));
		instrumentVolumes = INSTRUMENTS.map(() => 0.7);
		instrumentMuted = INSTRUMENTS.map(() => false);
		instrumentPitch = [0.7, 1.0, 1.0, 1.0, 1.0, 1.0, 1.0, 1.0];
		synthPattern = Array(STEPS).fill(-1);
		synthChordType = 1;
		synthVolume = 0.2;
		synthMuted = false;
		synthDelay = 0.65;
		synthReverb = 0.75;
		bpm = 122;
		volume = 0.7;
	}

	function copyPattern(fromIndex: number, toIndex: number) {
		const song = patternSlots[fromIndex];
		if (!song) return;
		
		const copiedSong: Song = {
			name: `${song.name} (copy)`,
			pattern: song.pattern.map(row => [...row]),
			instrumentVolumes: [...song.instrumentVolumes],
			instrumentMuted: [...song.instrumentMuted],
			instrumentPitch: [...song.instrumentPitch],
			synthPattern: [...song.synthPattern],
			synthChordType: song.synthChordType,
			synthVolume: song.synthVolume,
			synthMuted: song.synthMuted,
			synthDelay: song.synthDelay,
			synthReverb: song.synthReverb,
			bpm: song.bpm,
			volume: song.volume
		};
		
		patternSlots[toIndex] = copiedSong;
	}

	function deletePattern(index: number) {
		if (patternSlots[index] && confirm(`Delete pattern ${index + 1}: "${patternSlots[index]!.name}"?`)) {
			patternSlots[index] = null;
			if (activePatternIndex === index) {
				// Find next available pattern or reset to 0
				const nextPattern = patternSlots.findIndex(slot => slot !== null);
				activePatternIndex = nextPattern !== -1 ? nextPattern : 0;
			}
		}
	}

	function duplicateToNextEmpty(fromIndex: number) {
		const song = patternSlots[fromIndex];
		if (!song) return;
		
		const emptyIndex = patternSlots.findIndex((slot, idx) => slot === null && idx > fromIndex);
		if (emptyIndex === -1) {
			// No empty slot after this one, find first empty slot
			const firstEmpty = patternSlots.findIndex(slot => slot === null);
			if (firstEmpty === -1) {
				alert('No empty slots available');
				return;
			}
			copyPattern(fromIndex, firstEmpty);
		} else {
			copyPattern(fromIndex, emptyIndex);
		}
	}

	function startCopyMode(index: number) {
		if (!patternSlots[index]) return;
		copyMode = true;
		copySourceIndex = index;
	}

	function handlePatternClick(index: number) {
		if (copyMode) {
			// Copy mode: copy from source to this slot
			if (index !== copySourceIndex) {
				copyPattern(copySourceIndex, index);
			}
			copyMode = false;
			copySourceIndex = -1;
		} else {
			// Normal mode: select pattern
			selectPattern(index);
		}
	}

	function exportSong() {
		// Auto-save current pattern first
		saveCurrentPatternToSlot(activePatternIndex);
		
		const name = prompt('Enter project name for export:', 'groovebox-project');
		if (!name) return;
		
		const project = {
			name,
			version: '1.0',
			patterns: patternSlots,
			activePatternIndex,
			exportDate: new Date().toISOString()
		};
		
		const json = JSON.stringify(project, null, 2);
		const blob = new Blob([json], { type: 'application/json' });
		const url = URL.createObjectURL(blob);
		const a = document.createElement('a');
		a.href = url;
		a.download = `${name}.json`;
		a.click();
		URL.revokeObjectURL(url);
	}

	function importSong() {
		fileInput?.click();
	}

	function handleFileImport(event: Event) {
		const input = event.target as HTMLInputElement;
		const file = input.files?.[0];
		if (!file) return;
		
		const reader = new FileReader();
		reader.onload = (e) => {
			try {
				const data = JSON.parse(e.target?.result as string);
				
				// Check if it's a project file (with patterns array) or single pattern
				if (data.patterns && Array.isArray(data.patterns)) {
					// Import entire project
					if (confirm('Import entire project? This will replace all current patterns.')) {
						patternSlots = data.patterns;
						activePatternIndex = data.activePatternIndex || 0;
						// Load the active pattern
						if (patternSlots[activePatternIndex]) {
							loadPattern(activePatternIndex);
						}
						alert('Project imported successfully!');
					}
				} else {
					// Import single pattern (legacy format)
					const song: Song = data;
					
					// Find first empty slot
					const emptySlot = patternSlots.findIndex(slot => slot === null);
					if (emptySlot !== -1) {
						patternSlots[emptySlot] = song;
						loadPattern(emptySlot);
						activePatternIndex = emptySlot;
						alert('Pattern imported to slot ' + (emptySlot + 1));
					} else {
						alert('All pattern slots are full');
					}
				}
			} catch (error) {
				alert('Error importing file: ' + error);
			}
		};
		reader.readAsText(file);
		input.value = '';
	}

	$effect(() => {
		if (masterGain) {
			masterGain.gain.value = volume;
		}
	});

	$effect(() => {
		if (delayMix) {
			delayMix.gain.value = synthDelay;
		}
	});

	$effect(() => {
		if (reverbMix) {
			reverbMix.gain.value = synthReverb;
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
						onclick={() => { songMode = !songMode; if (songMode) loopCounter = 0; }}
						class="rounded-lg px-4 py-2 transition-all {songMode ? 'bg-red-600 ring-2 ring-red-400' : 'bg-gray-800 hover:bg-gray-700'}"
						aria-label="Toggle song mode"
						title="Song Mode: Loop each pattern 4x and advance"
					>
						{songMode ? '🔁' : '▶️'}
					</button>
					<button
						onclick={randomPattern}
						class="rounded-lg bg-gray-800 px-4 py-2 transition-all hover:bg-gray-700 text-lg"
						aria-label="Random all patterns"
						title="Randomize drums and synth"
					>
						🎲
					</button>
					<button
						onclick={clearPattern}
						class="rounded-lg bg-gray-800 px-4 py-2 transition-all hover:bg-gray-700 text-lg"
						aria-label="Clear all patterns"
						title="Clear drums and synth"
					>
						🗑️
					</button>
				</div>

				<!-- Song Management -->
				<div class="flex gap-2">
					<button
						onclick={saveSong}
						class="rounded-lg bg-green-700 px-4 py-2 transition-all hover:bg-green-600"
						aria-label="Save pattern"
						title="Save current pattern to slot"
					>
						<svg class="h-5 w-5" fill="currentColor" viewBox="0 0 20 20">
							<path d="M7.707 10.293a1 1 0 10-1.414 1.414l3 3a1 1 0 001.414 0l3-3a1 1 0 00-1.414-1.414L11 11.586V6h5a2 2 0 012 2v7a2 2 0 01-2 2H4a2 2 0 01-2-2V8a2 2 0 012-2h5v5.586l-1.293-1.293zM9 4a1 1 0 012 0v2H9V4z"/>
						</svg>
					</button>
					<button
						onclick={saveProject}
						class="rounded-lg bg-green-800 px-4 py-2 transition-all hover:bg-green-700"
						aria-label="Save project"
						title="Save all patterns to browser storage"
					>
						💾
					</button>
					<button
						onclick={exportSong}
						class="rounded-lg bg-blue-700 px-4 py-2 transition-all hover:bg-blue-600"
						aria-label="Export project"
						title="Export all patterns to file"
					>
						<svg class="h-5 w-5" fill="currentColor" viewBox="0 0 20 20">
							<path fill-rule="evenodd" d="M3 17a1 1 0 011-1h12a1 1 0 110 2H4a1 1 0 01-1-1zm3.293-7.707a1 1 0 011.414 0L9 10.586V3a1 1 0 112 0v7.586l1.293-1.293a1 1 0 111.414 1.414l-3 3a1 1 0 01-1.414 0l-3-3a1 1 0 010-1.414z" clip-rule="evenodd" />
						</svg>
					</button>
					<button
						onclick={importSong}
						class="rounded-lg bg-purple-700 px-4 py-2 transition-all hover:bg-purple-600"
						aria-label="Import project"
						title="Import project or pattern from file"
					>
						<svg class="h-5 w-5" fill="currentColor" viewBox="0 0 20 20">
							<path fill-rule="evenodd" d="M3 17a1 1 0 011-1h12a1 1 0 110 2H4a1 1 0 01-1-1zM6.293 6.707a1 1 0 010-1.414l3-3a1 1 0 011.414 0l3 3a1 1 0 01-1.414 1.414L11 5.414V13a1 1 0 11-2 0V5.414L7.707 6.707a1 1 0 01-1.414 0z" clip-rule="evenodd" />
						</svg>
					</button>
					<input
						type="file"
						accept=".json"
						bind:this={fileInput}
						onchange={handleFileImport}
						class="hidden"
					/>
				</div>
			</div>
		</div>

		<!-- Pattern Slots (16 slots at top) -->
		<div class="mb-6 rounded-lg bg-gray-900 p-4">
			<div class="flex items-center justify-between mb-3">
				<h3 class="text-sm font-semibold text-gray-400">PATTERNS</h3>
				{#if queuedPatternIndex !== -1}
					<span class="text-xs text-yellow-500">Pattern {queuedPatternIndex + 1} queued</span>
				{/if}
			</div>
			<div class="grid grid-cols-16 gap-1">
				{#each patternSlots as slot, idx}
					<div class="relative h-16 rounded transition-all {copyMode && idx === copySourceIndex
							? 'bg-blue-700 ring-2 ring-blue-400'
							: activePatternIndex === idx 
								? 'bg-red-600 ring-2 ring-white' 
								: queuedPatternIndex === idx
									? 'bg-yellow-600 ring-2 ring-yellow-400'
									: slot 
										? 'bg-gray-700 hover:bg-gray-600' 
										: 'bg-gray-800 hover:bg-gray-750 text-gray-600'}"
						role="button"
						tabindex="0"
						onclick={() => handlePatternClick(idx)}
						onkeydown={(e) => { if (e.key === 'Enter' || e.key === ' ') handlePatternClick(idx); }}
					>
						<div class="flex flex-col items-center justify-center h-full cursor-pointer">
							<span class="text-xs font-bold">{idx + 1}</span>
							{#if slot}
								<span class="text-[0.6rem] truncate w-full px-1 text-center">{slot.name}</span>
							{:else}
								<span class="text-[0.6rem]">{copyMode ? 'paste' : 'new'}</span>
							{/if}
						</div>
						{#if slot && !copyMode}
							<div class="absolute top-0.5 right-0.5 flex gap-0.5">
								<button
									onclick={(e) => { e.stopPropagation(); deletePattern(idx); }}
									class="w-5 h-5 rounded-full bg-black/70 hover:bg-red-600 flex items-center justify-center text-white"
									aria-label="Delete pattern {idx + 1}"
									title="Delete pattern"
								>
									<svg class="h-3 w-3" fill="currentColor" viewBox="0 0 20 20">
										<path fill-rule="evenodd" d="M4.293 4.293a1 1 0 011.414 0L10 8.586l4.293-4.293a1 1 0 111.414 1.414L11.414 10l4.293 4.293a1 1 0 01-1.414 1.414L10 11.414l-4.293 4.293a1 1 0 01-1.414-1.414L8.586 10 4.293 5.707a1 1 0 010-1.414z" clip-rule="evenodd" />
									</svg>
								</button>
								<button
									onclick={(e) => { e.stopPropagation(); duplicateToNextEmpty(idx); }}
									class="w-5 h-5 rounded-full bg-black/70 hover:bg-green-600 flex items-center justify-center text-white text-xs font-bold"
									aria-label="Duplicate pattern {idx + 1} to next empty slot"
									title="Duplicate to next empty slot"
								>
									+
								</button>
								<button
									onclick={(e) => { e.stopPropagation(); startCopyMode(idx); }}
									class="w-5 h-5 rounded-full bg-black/70 hover:bg-blue-600 flex items-center justify-center text-white text-xs font-bold"
									aria-label="Copy pattern {idx + 1}"
									title="Copy pattern to any slot"
								>
									⎘
								</button>
							</div>
						{/if}
					</div>
				{/each}
			</div>
			{#if copyMode}
				<div class="mb-4 text-center text-sm text-blue-400 bg-gray-900 p-2 rounded">
					Copy mode: Click destination slot to paste pattern "{patternSlots[copySourceIndex]?.name}"
					<button onclick={() => { copyMode = false; copySourceIndex = -1; }} class="ml-4 px-2 py-1 bg-gray-800 hover:bg-gray-700 rounded">
						Cancel
					</button>
				</div>
			{/if}
		</div>

		<!-- Controls -->
		<div class="mb-6 grid grid-cols-2 gap-4 rounded-lg bg-gray-900 p-4">
			<div>
				<label for="bpm-slider" class="mb-2 block text-sm font-medium text-gray-400">BPM</label>
				<div class="flex items-center gap-3">
					<input
						id="bpm-slider"
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
				<label for="master-volume-slider" class="mb-2 block text-sm font-medium text-gray-400">Master Volume</label>
				<div class="flex items-center gap-3">
					<input
						id="master-volume-slider"
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
		<div class="rounded-lg bg-gray-900 p-6 mb-6">
			<div class="mb-4 flex items-center justify-between">
				<div class="flex items-center gap-4">
					<h2 class="text-lg font-semibold">Drum Sequencer</h2>
					<div class="flex gap-2">
						<button
							onclick={randomDrumPattern}
							class="rounded-lg bg-gray-800 px-4 py-2 transition-all hover:bg-gray-700 text-lg"
							title="Randomize drums"
						>
							🎲
						</button>
						<button
							onclick={clearDrumPattern}
							class="rounded-lg bg-gray-800 px-4 py-2 transition-all hover:bg-gray-700 text-lg"
							title="Clear drums"
						>
							🗑️
						</button>
					</div>
				</div>
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
						<!-- Mute button -->
						<button
							onclick={() => toggleMute(instIdx)}
							class="h-8 w-8 shrink-0 rounded transition-all {instrumentMuted[instIdx]
								? 'bg-gray-700 text-gray-500'
								: 'bg-gray-800 text-white hover:bg-gray-700'}"
							aria-label="{instrumentMuted[instIdx] ? 'Unmute' : 'Mute'} {instrument.name}"
						>
							<svg class="h-4 w-4 mx-auto" fill="currentColor" viewBox="0 0 20 20">
								{#if instrumentMuted[instIdx]}
									<path fill-rule="evenodd" d="M9.383 3.076A1 1 0 0110 4v12a1 1 0 01-1.707.707L4.586 13H2a1 1 0 01-1-1V8a1 1 0 011-1h2.586l3.707-3.707a1 1 0 011.09-.217zM12.293 7.293a1 1 0 011.414 0L15 8.586l1.293-1.293a1 1 0 111.414 1.414L16.414 10l1.293 1.293a1 1 0 01-1.414 1.414L15 11.414l-1.293 1.293a1 1 0 01-1.414-1.414L13.586 10l-1.293-1.293a1 1 0 010-1.414z" clip-rule="evenodd" />
								{:else}
									<path fill-rule="evenodd" d="M9.383 3.076A1 1 0 0110 4v12a1 1 0 01-1.707.707L4.586 13H2a1 1 0 01-1-1V8a1 1 0 011-1h2.586l3.707-3.707a1 1 0 011.09-.217zM14.657 2.929a1 1 0 011.414 0A9.972 9.972 0 0119 10a9.972 9.972 0 01-2.929 7.071 1 1 0 01-1.414-1.414A7.971 7.971 0 0017 10c0-2.21-.894-4.208-2.343-5.657a1 1 0 010-1.414zm-2.829 2.828a1 1 0 011.415 0A5.983 5.983 0 0115 10a5.984 5.984 0 01-1.757 4.243 1 1 0 01-1.415-1.415A3.984 3.984 0 0013 10a3.983 3.983 0 00-1.172-2.828 1 1 0 010-1.415z" clip-rule="evenodd" />
								{/if}
							</svg>
						</button>

						<!-- Instrument volume -->
						<div class="w-32 shrink-0">
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

						<!-- Instrument pitch -->
						<div class="w-24 shrink-0">
							<input
								type="range"
								bind:value={instrumentPitch[instIdx]}
								min="0.5"
								max="2"
								step="0.01"
								class="w-full"
								aria-label="{instrument.name} pitch"
							/>
						</div>

						<!-- Instrument label -->
						<div class="w-20 shrink-0 text-right">
							<span class="text-sm font-medium {instrumentMuted[instIdx] ? 'text-gray-500' : ''}">{instrument.name}</span>
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

		<!-- Synth Sequencer -->
		<div class="rounded-lg bg-gray-900 p-6">
			<div class="mb-4 flex items-center justify-between">
				<h2 class="text-lg font-semibold">Synth Sequencer</h2>
				<div class="flex items-center gap-4">
					<!-- Chord Type Selector -->
					<div class="flex items-center gap-2">
						<label class="text-sm text-gray-400">Chord:</label>
						<select
							bind:value={synthChordType}
							class="rounded bg-gray-800 px-3 py-1 text-sm text-white border border-gray-700 focus:border-red-500 outline-none"
						>
							{#each CHORD_TYPES as chordType, idx}
								<option value={idx}>{chordType.name}</option>
							{/each}
						</select>
					</div>

					<!-- Pattern Controls -->
					<div class="flex gap-2">
						<button
							onclick={randomSynthPattern}
							class="rounded-lg bg-gray-800 px-4 py-2 transition-all hover:bg-gray-700 text-lg"
							aria-label="Random synth pattern"
							title="Randomize synth"
						>
							🎲
						</button>
						<button
							onclick={clearSynthPattern}
							class="rounded-lg bg-gray-800 px-4 py-2 transition-all hover:bg-gray-700 text-lg"
							aria-label="Clear synth pattern"
							title="Clear synth"
						>
							🗑️
						</button>
					</div>
				</div>
			</div>

			<!-- Synth Controls -->
			<div class="mb-4 space-y-3">
				<div class="flex items-center gap-4">
					<!-- Mute button -->
					<button
						onclick={() => synthMuted = !synthMuted}
						class="h-8 w-8 rounded transition-all {synthMuted
							? 'bg-gray-700 text-gray-500'
							: 'bg-gray-800 text-white hover:bg-gray-700'}"
						aria-label="{synthMuted ? 'Unmute' : 'Mute'} Synth"
					>
						<svg class="h-4 w-4 mx-auto" fill="currentColor" viewBox="0 0 20 20">
							{#if synthMuted}
								<path fill-rule="evenodd" d="M9.383 3.076A1 1 0 0110 4v12a1 1 0 01-1.707.707L4.586 13H2a1 1 0 01-1-1V8a1 1 0 011-1h2.586l3.707-3.707a1 1 0 011.09-.217zM12.293 7.293a1 1 0 011.414 0L15 8.586l1.293-1.293a1 1 0 111.414 1.414L16.414 10l1.293 1.293a1 1 0 01-1.414 1.414L15 11.414l-1.293 1.293a1 1 0 01-1.414-1.414L13.586 10l-1.293-1.293a1 1 0 010-1.414z" clip-rule="evenodd" />
							{:else}
								<path fill-rule="evenodd" d="M9.383 3.076A1 1 0 0110 4v12a1 1 0 01-1.707.707L4.586 13H2a1 1 0 01-1-1V8a1 1 0 011-1h2.586l3.707-3.707a1 1 0 011.09-.217zM14.657 2.929a1 1 0 011.414 0A9.972 9.972 0 0119 10a9.972 9.972 0 01-2.929 7.071 1 1 0 01-1.414-1.414A7.971 7.971 0 0017 10c0-2.21-.894-4.208-2.343-5.657a1 1 0 010-1.414zm-2.829 2.828a1 1 0 011.415 0A5.983 5.983 0 0115 10a5.984 5.984 0 01-1.757 4.243 1 1 0 01-1.415-1.415A3.984 3.984 0 0013 10a3.983 3.983 0 00-1.172-2.828 1 1 0 010-1.415z" clip-rule="evenodd" />
							{/if}
						</svg>
					</button>

					<!-- Volume -->
					<div class="flex-1">
						<div class="flex items-center gap-3">
							<label class="text-sm text-gray-400 w-16">Volume</label>
							<input
								type="range"
								bind:value={synthVolume}
								min="0"
								max="1"
								step="0.01"
								class="flex-1"
							/>
							<span class="w-16 text-right font-mono text-sm">{Math.round(synthVolume * 100)}%</span>
						</div>
					</div>
				</div>

				<!-- Effects Row -->
				<div class="flex items-center gap-4">
					<div class="w-8 shrink-0"></div>
					<!-- Delay -->
					<div class="flex-1">
						<div class="flex items-center gap-3">
							<label class="text-sm text-gray-400 w-16">Delay</label>
							<input
								type="range"
								bind:value={synthDelay}
								min="0"
								max="1"
								step="0.01"
								class="flex-1"
							/>
							<span class="w-16 text-right font-mono text-sm">{Math.round(synthDelay * 100)}%</span>
						</div>
					</div>
				</div>

				<div class="flex items-center gap-4">
					<div class="w-8 shrink-0"></div>
					<!-- Reverb -->
					<div class="flex-1">
						<div class="flex items-center gap-3">
							<label class="text-sm text-gray-400 w-16">Reverb</label>
							<input
								type="range"
								bind:value={synthReverb}
								min="0"
								max="1"
								step="0.01"
								class="flex-1"
							/>
							<span class="w-16 text-right font-mono text-sm">{Math.round(synthReverb * 100)}%</span>
						</div>
					</div>
				</div>
			</div>

			<!-- Synth Grid -->
			<div class="space-y-1">
				{#each SYNTH_NOTES as note, noteIdx}
					<div class="flex items-center gap-2">
						<!-- Left spacing to match drum grid -->
						<div class="w-8 shrink-0"></div>
						<div class="w-32 shrink-0"></div>
						<div class="w-24 shrink-0"></div>

						<!-- Note label -->
						<div class="w-20 shrink-0 text-right">
							<span class="text-sm font-medium {synthMuted ? 'text-gray-500' : ''}">{note.name}</span>
						</div>

						<!-- Steps -->
						<div class="flex flex-1 gap-1">
							{#each Array(4) as _, bar}
								<div class="flex gap-0.5">
									{#each Array(4) as _, beat}
										{@const stepIdx = bar * 4 + beat}
										<button
											onclick={() => toggleSynthNote(stepIdx, noteIdx)}
											class="relative h-8 w-10 rounded transition-all {synthPattern[stepIdx] === noteIdx
												? 'bg-green-500 shadow-lg'
												: 'bg-gray-800 hover:bg-gray-700'} {currentStep === stepIdx && isPlaying
												? 'ring-2 ring-white'
												: ''}"
										>
											{#if synthPattern[stepIdx] === noteIdx}
												<div class="absolute inset-0 flex items-center justify-center">
													<div class="h-2 w-2 rounded-full bg-white opacity-90"></div>
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
	</div>>
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
