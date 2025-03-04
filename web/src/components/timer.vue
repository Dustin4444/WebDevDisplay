<script lang="ts" setup>
import { onMounted, type Ref, ref } from 'vue';

const timer: Ref<string> = ref('');
const canvas: Ref<HTMLCanvasElement | null> = ref(null);
let context: CanvasRenderingContext2D | null;

const CANVAS_WIDTH = 90;
const CANVAS_HEIGHT = 10;

// looooool
function leftPad(num: unknown) {
	return String(num).padStart(2, '0');
}

function createTimer() {
	const ONE_SECOND = 1000;
	const ONE_MINUTE = ONE_SECOND * 60;
	const ONE_HOUR = ONE_MINUTE * 60;

	const [hh = 0, mm = 0, ss = 0] = '04:00:00'.split(':'); // settings.value.time.split(':');
	let timeRemaining =
		ONE_HOUR * Number(hh) + ONE_MINUTE * Number(mm) + ONE_SECOND + Number(ss);

	setInterval(() => {
		timeRemaining -= ONE_SECOND; // reduce by 1 second

		const h = leftPad(Math.floor(timeRemaining / ONE_HOUR));
		const m = leftPad(Math.floor((timeRemaining / ONE_MINUTE) % 60));
		const s = leftPad(Math.floor((timeRemaining / ONE_SECOND) % 60));

		timer.value = `${h}:${m}:${s}`;

		render();
	}, 1000);
}

function resetCanvas() {
	if (!context) {
		return;
	}

	context.fillStyle = '#f00';
	context.fillRect(0, 0, CANVAS_WIDTH, CANVAS_HEIGHT);
}

function render() {
	if (!context) {
		return;
	}

	resetCanvas();

	context.font = "8px 'ScoreboardFont'";
	context.fillStyle = '#fff';
	context.textRendering = 'geometricPrecision';
	context.imageSmoothingEnabled = false;

	context.fillText(` ${timer.value} `, 0, CANVAS_HEIGHT - 1);
}

onMounted(() => {
	if (!canvas.value) {
		return;
	}

	context = canvas.value?.getContext('2d', {
		willReadFrequently: true,
	});

	canvas.value.width = CANVAS_WIDTH;
	canvas.value.height = CANVAS_HEIGHT;

	if (!context) {
		return;
	}

	createTimer();
});
</script>

<template>
	<div class="canvas-wrapper">
		<canvas ref="canvas" class="h-16"></canvas>
	</div>
</template>

<style lang="css" scoped>
.canvas-wrapper {
	background: white;
}
</style>
