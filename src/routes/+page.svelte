<script lang="ts">
	import { onMount } from 'svelte';

	// ===== TYPE DEFINITIONS =====
	interface KeyPosition {
		x: number;
		y: number;
	}

	interface KeyboardSettings {
		theme: 'dark' | 'light' | 'neon';
		keyShape: 'rounded' | 'square' | 'circular';
		is3D: boolean;
		primaryColor: string;
		hoverColor: string;
		textColor: string;
	}

	interface LineSettings {
		color: string;
		width: number;
		style: 'solid' | 'dashed' | 'dotted';
		glow: boolean;
		dotSize: number;
	}

	interface Settings {
		keyboard: KeyboardSettings;
		lines: LineSettings;
	}

	// ===== CANVAS AND DRAWING STATE =====
	// Canvas element and 2D context for drawing lines
	let canvas: HTMLCanvasElement | undefined = $state();
	let ctx: CanvasRenderingContext2D | null = null;

	// ===== TYPING STATE MANAGEMENT =====
	// Array to track the sequence of letters typed (for line drawing)
	let typedSequence: string[] = [];
	// Current displayed text (includes spaces, formatted for display)
	let currentText: string = $state('');

	// ===== KEYBOARD VISIBILITY AND ANIMATION =====
	// Controls keyboard fade in/out behavior
	let keyboardVisible: boolean = true;
	let keyboardOpacity: number = $state(1);
	let showExportButtons: boolean = $state(false);
	let showSettings: boolean = $state(false);

	// ===== TIMING CONTROLS =====
	// Timers for auto-fade and export button appearance
	let fadeTimeout: number | undefined;
	let exportTimeout: number | undefined;
	let isHovering: boolean = false;

	// ===== CUSTOMIZATION SETTINGS =====
	// All user-configurable options for keyboard appearance and line styling
	let settings: Settings = $state({
		keyboard: {
			theme: 'dark', // dark, light, neon - changes overall color scheme
			keyShape: 'rounded', // rounded, square, circular - key border radius
			is3D: false, // adds shadow and hover scale effects
			primaryColor: 'dark', // background color for dark theme keys
			hoverColor: '#4b5563', // hover state color
			textColor: '#ffffff' // key text color
		},
		lines: {
			color: '#ffffff', // line stroke color
			width: 4, // line thickness in pixels
			style: 'solid', // solid, dashed, dotted - line pattern
			glow: true, // adds shadow blur effect
			dotSize: 5 // size of dots at key positions (currently not drawn)
		}
	});

	// ===== KEYBOARD LAYOUT DEFINITION =====
	// Standard QWERTY layout arranged in rows
	const keyboardLayout: string[][] = [
		['Q', 'W', 'E', 'R', 'T', 'Y', 'U', 'I', 'O', 'P'],
		['A', 'S', 'D', 'F', 'G', 'H', 'J', 'K', 'L'],
		['Z', 'X', 'C', 'V', 'B', 'N', 'M']
	];

	// ===== KEY POSITION TRACKING =====
	// Stores x,y coordinates of each key for line drawing
	let keyPositions: Record<string, KeyPosition> = {};

	// ===== INITIALIZATION =====
	onMount(() => {
		if (canvas) {
			ctx = canvas.getContext('2d');
			if (ctx) {
				canvas.width = window.innerWidth;
				canvas.height = window.innerHeight;

				// Calculate initial key positions for line drawing
				calculateKeyPositions();

				// Handle window resize - recalculate positions and redraw
				const handleResize = (): void => {
					if (canvas && ctx) {
						canvas.width = window.innerWidth;
						canvas.height = window.innerHeight;
						calculateKeyPositions();
						redrawLines();
					}
				};

				window.addEventListener('resize', handleResize);

				return () => {
					window.removeEventListener('resize', handleResize);
				};
			}
		}
	});

	// ===== KEY POSITION CALCULATION =====
	// Maps each keyboard key to its screen coordinates for line drawing
	function calculateKeyPositions(): void {
		const keyboard = document.querySelector('.keyboard');
		if (!keyboard) return;

		const keys = keyboard.querySelectorAll('.key[data-letter]') as NodeListOf<HTMLElement>;
		keys.forEach((key) => {
			const rect = key.getBoundingClientRect();
			const letter = key.getAttribute('data-letter');
			if (letter) {
				keyPositions[letter] = {
					x: rect.left + rect.width / 2,
					y: rect.top + rect.height / 2
				};
			}
		});
	}

	// ===== MAIN KEY HANDLING =====
	// Processes all keyboard input and routes to appropriate handlers
	function handleKeyPress(event: KeyboardEvent): void {
		const key = event.key.toUpperCase();

		if (key.match(/^[A-Z]$/)) {
			// Only alphabetical keys are added to the name
			addLetter(key);
		} else if (key === ' ') {
			// Space creates word breaks in display text
			addSpace();
		} else if (key === 'BACKSPACE') {
			// Delete removes last character
			removeLetter();
		} else if (key === 'ENTER') {
			// Enter confirms typing is complete
			confirmTyping();
		}
		// All other keys are ignored and won't affect the displayed name
	}

	// ===== LETTER INPUT HANDLING =====
	// Adds a letter to both display text and drawing sequence
	function addLetter(letter: string): void {
		currentText += letter.toLowerCase();
		typedSequence.push(letter);

		// Reset all timers and show keyboard
		clearTimeout(fadeTimeout);
		clearTimeout(exportTimeout);

		keyboardOpacity = 1;
		showExportButtons = false;

		// Recalculate positions and redraw lines
		setTimeout(() => {
			calculateKeyPositions();
			redrawLines();
		}, 10);

		// Start the auto-fade timer
		startFadeTimer();
	}

	// ===== SPACE HANDLING =====
	// Adds space to display text but not to drawing sequence
	function addSpace(): void {
		if (currentText.length > 0 && !currentText.endsWith(' ')) {
			currentText += ' ';
			// Don't add space to typed sequence for line drawing

			// Reset timers and restart fade countdown
			clearTimeout(fadeTimeout);
			clearTimeout(exportTimeout);
			keyboardOpacity = 1;
			showExportButtons = false;
			startFadeTimer();
		}
	}

	// ===== DELETE HANDLING =====
	// Removes last character from both display and drawing sequence
	function removeLetter(): void {
		if (currentText.length > 0) {
			const lastChar = currentText[currentText.length - 1];
			currentText = currentText.slice(0, -1);

			// Only remove from typed sequence if it's a letter (not space)
			if (lastChar.match(/[a-z]/i)) {
				typedSequence.pop();
			}

			redrawLines();

			// Reset timers and continue if text remains
			clearTimeout(fadeTimeout);
			clearTimeout(exportTimeout);
			keyboardOpacity = 1;
			showExportButtons = false;

			if (currentText.length > 0) {
				startFadeTimer();
			}
		}
	}

	// ===== TYPING COMPLETION =====
	// Immediately fades keyboard and shows export options
	function confirmTyping(): void {
		clearTimeout(fadeTimeout);
		clearTimeout(exportTimeout);
		keyboardOpacity = 0.1; // Fade out more to leave name and lines visible
		showExportButtons = true;
	}

	// ===== AUTO-FADE TIMER SYSTEM =====
	// Manages keyboard fade-out after inactivity
	function startFadeTimer(): void {
		fadeTimeout = setTimeout(() => {
			if (!isHovering) {
				keyboardOpacity = 0.3;

				// After additional delay, show export buttons
				exportTimeout = setTimeout(() => {
					if (!isHovering && keyboardOpacity === 0.3) {
						showExportButtons = true;
					}
				}, 1000);
			}
		}, 2000);
	}

	// ===== MOUSE HOVER HANDLING =====
	// Prevents auto-fade when user hovers over keyboard
	function handleMouseEnter(): void {
		isHovering = true;
		clearTimeout(fadeTimeout);
		clearTimeout(exportTimeout);
		keyboardOpacity = 1;
		showExportButtons = false;
	}

	function handleMouseLeave(): void {
		isHovering = false;
		if (typedSequence.length > 0) {
			startFadeTimer();
		}
	}

	// ===== LINE DRAWING ENGINE =====
	// Renders connecting lines between typed letters on canvas
	function redrawLines(): void {
		if (!ctx || !canvas) return;

		// Clear previous drawing
		ctx.clearRect(0, 0, canvas.width, canvas.height);

		if (typedSequence.length === 0) return;

		// Apply current line style settings
		ctx.strokeStyle = settings.lines.color;
		ctx.lineWidth = settings.lines.width;
		ctx.lineCap = 'round';

		// Apply glow effect if enabled
		if (settings.lines.glow) {
			ctx.shadowColor = settings.lines.color;
			ctx.shadowBlur = 10;
		} else {
			ctx.shadowBlur = 0;
		}

		// Set line dash pattern based on style
		if (settings.lines.style === 'dashed') {
			ctx.setLineDash([10, 5]);
		} else if (settings.lines.style === 'dotted') {
			ctx.setLineDash([2, 8]);
		} else {
			ctx.setLineDash([]);
		}

		// Draw connecting lines between consecutive letters
		if (typedSequence.length >= 2) {
			ctx.beginPath();

			for (let i = 0; i < typedSequence.length - 1; i++) {
				const currentKey = typedSequence[i];
				const nextKey = typedSequence[i + 1];

				const currentPos = keyPositions[currentKey];
				const nextPos = keyPositions[nextKey];

				if (currentPos && nextPos) {
					if (i === 0) {
						ctx.moveTo(currentPos.x, currentPos.y);
					}
					ctx.lineTo(nextPos.x, nextPos.y);
				}
			}

			ctx.stroke();
		}
	}

	// ===== EXPORT FUNCTIONALITY =====
	// SVG export - creates scalable vector graphics file
	function exportAsSVG(): void {
		if (typedSequence.length < 2) return;

		let svgContent = `<svg width="${canvas?.width}" height="${canvas?.height}" xmlns="http://www.w3.org/2000/svg">`;

		// Add glow filter definition if enabled
		if (settings.lines.glow) {
			svgContent += `<defs><filter id="glow"><feGaussianBlur stdDeviation="3" result="coloredBlur"/><feMerge><feMergeNode in="coloredBlur"/><feMergeNode in="SourceGraphic"/></feMerge></filter></defs>`;
		}

		// Build path data for connecting lines
		let pathData = '';
		for (let i = 0; i < typedSequence.length - 1; i++) {
			const currentKey = typedSequence[i];
			const nextKey = typedSequence[i + 1];

			const currentPos = keyPositions[currentKey];
			const nextPos = keyPositions[nextKey];

			if (currentPos && nextPos) {
				if (i === 0) {
					pathData += `M ${currentPos.x} ${currentPos.y} `;
				}
				pathData += `L ${nextPos.x} ${nextPos.y} `;
			}
		}

		// Apply line styling to SVG path
		const strokeDasharray =
			settings.lines.style === 'dashed'
				? '10,5'
				: settings.lines.style === 'dotted'
					? '2,8'
					: 'none';
		const filter = settings.lines.glow ? 'filter="url(#glow)"' : '';

		svgContent += `<path d="${pathData}" stroke="${settings.lines.color}" stroke-width="${settings.lines.width}" stroke-dasharray="${strokeDasharray}" fill="none" ${filter}/>`;

		// Add dots at key positions (currently not visible in UI)
		typedSequence.forEach((letter) => {
			const pos = keyPositions[letter];
			if (pos) {
				svgContent += `<circle cx="${pos.x}" cy="${pos.y}" r="${settings.lines.dotSize}" fill="${settings.lines.color}" ${filter}/>`;
			}
		});

		svgContent += '</svg>';

		// Download the SVG file
		const blob = new Blob([svgContent], { type: 'image/svg+xml' });
		const url = URL.createObjectURL(blob);
		const a = document.createElement('a');
		a.href = url;
		a.download = `typing-pattern-${currentText.replace(/\s+/g, '-')}.svg`;
		a.click();
		URL.revokeObjectURL(url);
	}

	// PNG export - creates raster image from canvas
	function exportAsPNG(): void {
		if (!canvas) return;

		const link = document.createElement('a');
		link.download = `typing-pattern-${currentText.replace(/\s+/g, '-')}.png`;
		link.href = canvas.toDataURL();
		link.click();
	}

	// ===== UTILITY FUNCTIONS =====
	// Resets the entire application state
	function clearCanvas(): void {
		typedSequence = [];
		currentText = '';
		showExportButtons = false;
		keyboardOpacity = 1;
		clearTimeout(fadeTimeout);
		clearTimeout(exportTimeout);
		if (ctx && canvas) {
			ctx.clearRect(0, 0, canvas.width, canvas.height);
		}
	}

	// ===== SETTINGS MANAGEMENT =====
	// Toggles settings panel visibility
	function toggleSettings(): void {
		showSettings = !showSettings;
	}

	// Applies setting changes and redraws
	function updateSettings(): void {
		setTimeout(() => {
			calculateKeyPositions();
			redrawLines();
		}, 10);
	}

	// ===== DYNAMIC STYLING =====
	// Generates CSS classes for keyboard keys based on current settings
	function getKeyClasses(): string {
		let baseClasses = 'key w-14 h-10 border font-mono transition-all duration-150 ';

		// Apply theme-based colors
		if (settings.keyboard.theme === 'light') {
			baseClasses += 'bg-gray-200 hover:bg-gray-300 border-gray-400 text-black ';
		} else if (settings.keyboard.theme === 'neon') {
			baseClasses +=
				'bg-purple-900 hover:bg-purple-800 border-cyan-400 text-cyan-300 shadow-lg shadow-cyan-500/20 ';
		} else {
			baseClasses += `border-gray-600 text-white `;
		}

		// Apply shape styling
		if (settings.keyboard.keyShape === 'square') {
			baseClasses += 'rounded-none ';
		} else if (settings.keyboard.keyShape === 'circular') {
			baseClasses += 'rounded-full ';
		} else {
			baseClasses += 'rounded ';
		}

		// Apply 3D effects
		if (settings.keyboard.is3D) {
			baseClasses += 'shadow-lg transform hover:scale-105 ';
		}

		return baseClasses;
	}

	// Generates inline styles for dark theme customization
	function getKeyStyle(): string {
		if (settings.keyboard.theme === 'dark') {
			return `background-color: ${settings.keyboard.primaryColor}; color: ${settings.keyboard.textColor};`;
		}
		return '';
	}
</script>

<svelte:window onkeydown={handleKeyPress} />

<main
	class="relative min-h-screen overflow-hidden bg-black text-white"
	style="background-color: #1a1a1a;"
>
	<canvas bind:this={canvas} class="pointer-events-none absolute inset-0 z-30"></canvas>

	<button
		class="fixed top-4 right-4 z-40 rounded-lg bg-gray-800 p-3 transition-colors duration-200 hover:bg-gray-700"
		onclick={toggleSettings}
	>
		⚙️
	</button>

	{#if showSettings}
		<div class="bg-opacity-50 fixed inset-0 z-50 flex items-center justify-center bg-black p-4">
			<div class="max-h-[80vh] w-full max-w-md overflow-y-auto rounded-lg bg-gray-900 p-6">
				<div class="mb-6 flex items-center justify-between">
					<h2 class="text-xl font-semibold">Settings</h2>
					<button onclick={toggleSettings} class="text-gray-400 hover:text-white">✕</button>
				</div>

				<div class="mb-6">
					<h3 class="mb-3 text-lg font-medium">Keyboard</h3>

					<div class="space-y-4">
						<div>
							<label for="theme-select" class="mb-2 block text-sm font-medium">Theme</label>
							<select
								id="theme-select"
								bind:value={settings.keyboard.theme}
								onchange={updateSettings}
								class="w-full rounded bg-gray-800 p-2"
							>
								<option value="dark">Dark</option>
								<option value="light">Light</option>
								<option value="neon">Neon</option>
							</select>
						</div>

						<div>
							<label for="key-shape-select" class="mb-2 block text-sm font-medium">Key Shape</label>
							<select
								id="key-shape-select"
								bind:value={settings.keyboard.keyShape}
								onchange={updateSettings}
								class="w-full rounded bg-gray-800 p-2"
							>
								<option value="rounded">Rounded</option>
								<option value="square">Square</option>
								<option value="circular">Circular</option>
							</select>
						</div>

						<div>
							<label class="flex items-center space-x-2">
								<input
									type="checkbox"
									bind:checked={settings.keyboard.is3D}
									onchange={updateSettings}
									class="rounded"
								/>
								<span>3D Effect</span>
							</label>
						</div>

						{#if settings.keyboard.theme === 'dark'}
							<div>
								<label for="primary-color" class="mb-2 block text-sm font-medium"
									>Primary Color</label
								>
								<input
									id="primary-color"
									type="color"
									bind:value={settings.keyboard.primaryColor}
									onchange={updateSettings}
									class="h-10 w-full rounded"
								/>
							</div>
						{/if}
					</div>
				</div>

				<div>
					<h3 class="mb-3 text-lg font-medium">Lines</h3>

					<div class="space-y-4">
						<div>
							<label for="line-color" class="mb-2 block text-sm font-medium">Color</label>
							<input
								id="line-color"
								type="color"
								bind:value={settings.lines.color}
								onchange={updateSettings}
								class="h-10 w-full rounded"
							/>
						</div>

						<div>
							<label for="line-width" class="mb-2 block text-sm font-medium"
								>Width: {settings.lines.width}px</label
							>
							<input
								id="line-width"
								type="range"
								min="1"
								max="10"
								bind:value={settings.lines.width}
								onchange={updateSettings}
								class="w-full"
							/>
						</div>

						<div>
							<label for="line-style" class="mb-2 block text-sm font-medium">Style</label>
							<select
								id="line-style"
								bind:value={settings.lines.style}
								onchange={updateSettings}
								class="w-full rounded bg-gray-800 p-2"
							>
								<option value="solid">Solid</option>
								<option value="dashed">Dashed</option>
								<option value="dotted">Dotted</option>
							</select>
						</div>

						<div>
							<label class="flex items-center space-x-2">
								<input
									type="checkbox"
									bind:checked={settings.lines.glow}
									onchange={updateSettings}
									class="rounded"
								/>
								<span>Glow Effect</span>
							</label>
						</div>

						<div>
							<label for="dot-size" class="mb-2 block text-sm font-medium"
								>Dot Size: {settings.lines.dotSize}px</label
							>
							<input
								id="dot-size"
								type="range"
								min="2"
								max="10"
								bind:value={settings.lines.dotSize}
								onchange={updateSettings}
								class="h-10 w-full rounded"
							/>
						</div>
					</div>
				</div>

				<div class="mt-6 border-t border-gray-700 pt-4">
					<button
						onclick={toggleSettings}
						class="w-full rounded-lg bg-blue-600 px-4 py-2 font-medium text-white shadow-lg transition-colors duration-200 hover:bg-blue-700"
					>
						Save Settings
					</button>
				</div>
			</div>
		</div>
	{/if}

	<div class="relative z-20 flex min-h-screen flex-col items-center justify-center p-8">
		<div class="mb-12 text-center">
			<h1 class="mb-4 text-2xl font-light text-gray-400">Enter your name</h1>
			<div class="flex min-h-[3rem] items-center justify-center font-mono text-4xl tracking-wider">
				{currentText || ''}
				<span class="animate-pulse">|</span>
			</div>
		</div>

		<div
			class="keyboard transition-opacity duration-500 ease-in-out"
			style="opacity: {keyboardOpacity}"
			onmouseenter={handleMouseEnter}
			onmouseleave={handleMouseLeave}
			role="presentation"
		>
			{#each keyboardLayout as row, rowIndex}
				<div class="mb-2 flex justify-center gap-2">
					{#each row as letter}
						<button
							class={getKeyClasses()}
							style={getKeyStyle()}
							data-letter={letter}
							onclick={() => addLetter(letter)}
						>
							{letter}
						</button>
					{/each}
				</div>
			{/each}
		</div>

		{#if showExportButtons}
			<div
				class="animate-fade-in fixed bottom-8 left-1/2 z-40 flex -translate-x-1/2 transform gap-4"
			>
				<button
					class="rounded-lg bg-white px-6 py-3 font-medium text-black shadow-lg transition-colors duration-200 hover:bg-gray-100"
					onclick={exportAsSVG}
				>
					Export as SVG
				</button>
				<button
					class="rounded-lg bg-white px-6 py-3 font-medium text-black shadow-lg transition-colors duration-200 hover:bg-gray-100"
					onclick={exportAsPNG}
				>
					Export as PNG
				</button>
			</div>
		{/if}
	</div>
</main>

<style>
	@keyframes fade-in {
		from {
			opacity: 0;
			transform: translateX(-50%) translateY(20px);
		}
		to {
			opacity: 1;
			transform: translateX(-50%) translateY(0);
		}
	}

	.animate-fade-in {
		animation: fade-in 0.3s ease-out;
	}
</style>
