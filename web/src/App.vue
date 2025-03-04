<template>
	<Timer :count="count" />
	<div class="w-full min-h-screen h-full flex items-center justify-center m-0">
		<div class="m-auto p-2 w-full md:w-9/12">
			<h1 class="text-white text-2xl uppercase mb-2 tracking-wider text-center">
				<Icon icon="mdi:trophy" class="inline align-[-.125em] mr-1" />Scoreboard
			</h1>
			<div class="flex items-end space-x-2 border-white border rounded-md p-2">
				<div class="flex-1 mb-2">
					<label for="inputTimer">Timer Start Value (HH:MM:SS)</label>
					<div class="flex items-center space-x-2">
						<div class="flex-1">
							<input
								type="text"
								ref="inputTimer"
								id="inputTimer"
								placeholder="04:00:00"
								class="w-full p-2 h-12 border rounded-md"
								v-model="settings.time"
							/>
						</div>
					</div>
				</div>

				<div class="flex-1 mb-2">
					<label for="inputMessage">Scrolling Message</label>
					<div class="flex items-center space-x-2">
						<div class="flex-1">
							<input
								type="text"
								ref="inputMessage"
								id="inputMessage"
								placeholder="This will scroll across the screen"
								class="w-full p-2 h-12 border rounded-md"
								v-model="settings.message"
							/>
						</div>
					</div>
				</div>

				<input
					type="color"
					ref="foreground"
					class="w-12 h-12 border rounded appearance-none mb-2"
					v-model="settings.foreground"
				/>
				<input
					type="color"
					ref="background"
					class="w-12 h-12 border rounded mb-2"
					v-model="settings.background"
				/>
			</div>
			<button id="generate" @click="generate" class="mt-2 mb-4">
				Generate Canvas
			</button>
			<label>Canvas Buffer</label>
			<div class="overflow-x-scroll mb-4 border bg-zinc-700">
				<canvas ref="bufferCanvas" class="hidden"></canvas>
				<canvas
					ref="renderCanvas"
					class="h-16"
					@drop.prevent="handleDrop"
					@dragenter.prevent
					@dragover.prevent
				></canvas>
			</div>
			<div>
				<label>Scroll Speed ({{ settings.scrollSpeed }})</label>
				<div>
					<input
						type="range"
						min="0"
						max="32"
						v-model="settings.scrollSpeed"
						class="w-full bg-zinc-400 rounded-lg appearance-none mb-4"
					/>
				</div>
			</div>
			<button id="publish" @click="publishAndSave">Publish Canvas</button>

			<hr class="my-4 border-zinc-600" />
			<label>Status: {{ status }}</label>
		</div>
	</div>
</template>

<style scoped></style>

<script setup lang="ts">
import Timer from './components/timer.vue';

let bufferHeight: number = 10;

interface Settings {
	message: string;
	time: string;
	foreground: string;
	background: string;
	scrollSpeed: number;
}

import { onMounted, ref } from 'vue';
import { Icon } from '@iconify/vue';
import { Scoreboard } from './lib/scoreboard';

const scoreboard = new Scoreboard();

const bufferCanvas = ref<HTMLCanvasElement | null>(null);
const renderCanvas = ref<HTMLCanvasElement | null>(null);

const status = ref('');

const count = ref(0);

let renderContext: CanvasRenderingContext2D;
let bufferContext: CanvasRenderingContext2D;

const settings = ref<Settings>({
	message: '',
	time: '',
	foreground: '#efefef',
	background: '#000000',
	scrollSpeed: 0,
});

// Tokens for text replacement during publish.
const textReplacements = new Map<string, string>();

onMounted(async () => {
	// Load local data.
	loadData();

	if (!bufferCanvas.value || !renderCanvas.value) {
		return;
	}

	// Configure canvases.
	{
		bufferCanvas.value.width = 1024;
		bufferCanvas.value.height = bufferHeight;
		bufferContext = bufferCanvas.value.getContext('2d')!;

		renderCanvas.value.width = 8;
		renderCanvas.value.height = bufferHeight;
		renderContext = renderCanvas.value.getContext('2d', {
			willReadFrequently: true,
		})!;
	}

	// Load font.
	{
		// const font = new FontFace("ScoreboardFont", "url(/fonts/04B_20__.woff2)");
		const font = new FontFace('ScoreboardFont', 'url(/fonts/BoldPixels.ttf)');
		await font.load();
		document.fonts.add(font);
	}

	// timer length is 4 hours
	const ONE_SECOND = 1000;
	const ONE_MINUTE = ONE_SECOND * 60;
	const ONE_HOUR = ONE_MINUTE * 60;
	// let timer = ONE_HOUR * 4;

	const [hh = 0, mm = 0, ss = 0] = settings.value.time.split(':');
	let timer =
		ONE_HOUR * Number(hh) + ONE_MINUTE * Number(mm) + ONE_SECOND + Number(ss);

	/*
	 * This is a magic number based on how many characters fit on the scoreboard
	 * while the timer is also being displayed.
	 *
	 * NOTE: if the font changes, this will need to be checked!
	 */
	const MAX_LENGTH = 21;
	const message = settings.value.message;
	let offset = 0;

	setInterval(() => {
		timer -= ONE_SECOND; // reduce by 1 second

		const h = String(Math.floor(timer / ONE_HOUR)).padStart(2, '0');
		const m = String(Math.floor((timer / ONE_MINUTE) % 60)).padStart(2, '0');
		const s = String(Math.floor((timer / ONE_SECOND) % 60)).padStart(2, '0');

		textReplacements.set('{{timer}}', `${h}:${m}:${s}`);

		let msg = message.slice(
			offset,
			Math.min(MAX_LENGTH + offset, message.length),
		);

		if (msg.length < MAX_LENGTH) {
			msg += ' ' + message.slice(0, MAX_LENGTH - msg.length);
		}

		offset = offset >= message.length ? 0 : offset + 1;

		textReplacements.set('{{message}}', msg);

		generate();
		publish();
	}, 1000);

	// Generate initial canvas.
	generate();

	status.value = 'Ready.';
});

const generate = () => {
	if (!bufferCanvas.value || !renderCanvas.value) {
		return;
	}

	status.value = 'Generating canvas.';

	let text = ' {{timer}}  {{message}}';
	if (!text) return;

	// Loop through replacements and apply them.
	textReplacements.forEach((value, key) => {
		text = text.replace(key, value);
	});

	// Clear buffer.
	bufferContext.fillStyle = settings.value.background;
	bufferContext.fillRect(
		0,
		0,
		bufferCanvas.value.width,
		bufferCanvas.value.height,
	);

	// Set font and styles.
	{
		bufferContext.font = "8px 'ScoreboardFont'";
		bufferContext.fillStyle = settings.value.foreground;
		bufferContext.textRendering = 'geometricPrecision';
		bufferContext.imageSmoothingEnabled = false;

		renderContext.imageSmoothingEnabled = false;
	}

	// Draw text to buffer canvas.
	let totalWidth = 1;
	for (let i = 0; i < text.length; i++) {
		const char = text[i];
		bufferContext.fillText(char, Math.ceil(totalWidth), bufferHeight - 1);

		let metrics = bufferContext.measureText(char);
		let characterWidth = Math.ceil(metrics.width);
		totalWidth += characterWidth;
	}

	if (totalWidth > bufferCanvas.value.width) {
		status.value = 'Text input too long! Buffer truncated.';

		totalWidth = bufferCanvas.value.width;
	}

	// Calculate render canvas size.
	renderCanvas.value.width = totalWidth;
	renderContext.drawImage(bufferCanvas.value, 0, 0);
};

const publish = async () => {
	status.value = 'Publishing canvas.';

	if (
		!renderCanvas.value ||
		renderCanvas.value.width == 0 ||
		renderCanvas.value.height == 0
	) {
		status.value = 'Skipping empty buffer.';
		return;
	}

	const buffer = scoreboard.createBuffer(renderCanvas.value!, {
		scrollSpeed: settings.value.scrollSpeed,
		saveBuffer: false,
	});

	await fetch('http://webdev.local/buffer', {
		method: 'POST',
		headers: { 'Content-Type': 'application/octet-stream' },
		body: buffer,
		signal: AbortSignal.timeout(5000),
	})
		.then((response) => {
			if (response.status == 200) {
				status.value = 'Publish successful.';
			} else {
				status.value = 'Publish failed!';
			}
		})
		.catch((error) => {
			if (error.name === 'AbortError') {
				status.value = 'Fetch request timed out!';
			} else {
				status.value = 'An error occurred!';
				console.log(error);
			}
		});
};

const publishAndSave = async () => {
	await publish();
	await saveData();
};

const saveData = () => {
	localStorage.setItem('settings', JSON.stringify(settings.value));
};

const loadData = () => {
	const data = localStorage.getItem('settings');

	if (data) {
		settings.value = JSON.parse(data);
	}
};

const handleDrop = (event: DragEvent) => {
	if (!event.dataTransfer) {
		event.preventDefault();
		return;
	}

	const file = event.dataTransfer.files[0];
	if (file && file.type.startsWith('image/')) {
		const droppedImage = new Image();
		droppedImage.onload = () => {
			if (!bufferCanvas.value || !renderCanvas.value) {
				return;
			}

			const totalWidth = Math.min(droppedImage.width, bufferCanvas.value.width);
			renderCanvas.value.width = totalWidth;
			renderContext.drawImage(droppedImage, 0, 0);
		};
		droppedImage.src = URL.createObjectURL(file);
	}
};
</script>
