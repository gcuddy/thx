<script lang="ts">
	import { findNode } from '$lib/nodes';
	import { expoIn, quadInOut } from 'svelte/easing';
	import { tweened, type Tweened } from 'svelte/motion';

	let playing = false;

	function tweenedStore(startingValue: number, oscillator: OscillatorNode) {
		const val = tweened(startingValue, {
			duration: 1000 * 7,
			easing: expoIn
		});

		// this will cause memory leak, for testing purposes only
		val.subscribe((v) => {
			console.log('setting frequency', v);
			oscillator.frequency.value = v;
		});

		return {
			set: val.set,
			update: val.update,
			subscribe: val.subscribe
		};
	}

	const nodes = ['D1', 'D2', 'A2', 'D3', 'A3', 'D4', 'A4', 'D5', 'A5', 'D6', 'F#6'];

	let oscillatorCount = nodes.length * 3; // 33

	let doneCount = 0;

	$: if (doneCount === oscillatorCount) {
		// console.log('done'); playing = false;
		// fadeout();
		// start fade
	}

	function fadeout() {
		oscillators.forEach(({ oscillator }) => {
			gainNode.gain.linearRampToValueAtTime(0.0001, oscillator.context.currentTime + 5);
		});
		setTimeout(() => {
			playing = false;
		}, 5000);
	}

	let pannerValue = 0;

	let panner: StereoPannerNode;
	let ltrPanners: StereoPannerNode[] = [];
	let rtlPanners: StereoPannerNode[] = [];

	$: if (panner) {
		panner.pan.value = pannerValue;
	}

	const oscillators: Array<{
		oscillator: OscillatorNode;
		freq: Tweened<number>;
		node: string;
	}> = [];
	let gainNode: GainNode;

	function thx() {
		oscillators.forEach(({ oscillator }) => {
			oscillator.stop();
		});
		// clear array
		oscillators.length = 0;
		playing = true;

		const audio = new window.AudioContext();
		panner = new StereoPannerNode(audio, { pan: 0 });

		// panner.

		gainNode = audio.createGain();

		gainNode.gain.value = 0.1;

		panner.connect(gainNode);

		gainNode.connect(audio.destination);

		for (let i = 0; i < nodes.length; i++) {
			// add 3 oscillators per note, starting at a frequency randomly between 200 - 400
			for (let j = 0; j < 3; j++) {
				const oscillator = audio.createOscillator();
				oscillator.type = 'sawtooth';
				const startingValue = 0 + Math.random() * 200;
				const store = tweenedStore(startingValue, oscillator);
				if (i % 2 === 0) {
					const panner = new StereoPannerNode(audio, { pan: -1 });
					oscillator.connect(panner);
					panner.connect(gainNode);
					ltrPanners.push(panner);
				} else {
					const panner = new StereoPannerNode(audio, { pan: 1 });
					oscillator.connect(panner);
					panner.connect(gainNode);
					rtlPanners.push(panner);
				}
				oscillators.push({ oscillator, freq: store, node: nodes[i] });
                let filter = audio.createBiquadFilter();
                filter.type = "lowpass";
                // filter.frequency.value = 2000;
                filter.frequency.setTargetAtTime(2000, audio.currentTime, 0)
                oscillator.connect(filter);
                // filter.connect(gainNode);

				oscillator.connect(gainNode);
				oscillator.start();
			}
		}

		setTimeout(() => {
			for (const oscillator of oscillators) {
				const f = findNode(oscillator.node)?.freq;
				if (!f) continue;
				// give f a lil jitter
				oscillator.oscillator.frequency.exponentialRampToValueAtTime(
					f + Math.random() * 25 - 12.5,
					oscillator.oscillator.context.currentTime + 7.5
				);
				setTimeout(fadeout, 10 * 1000);
				// Two ways to do this - above is "proper" way, but with our tweened store way we can set the value directly
				// oscillator.freq.set(f + Math.random() * 25 - 12.5).then(() => {
				// 	doneCount++;
				// });
				// oscillator.freq.set(f);
			}
			ltrPanners.forEach((panner) => {
				panner.pan.linearRampToValueAtTime(1, panner.context.currentTime + 7.5);
			});
			rtlPanners.forEach((panner) => {
				panner.pan.linearRampToValueAtTime(-1, panner.context.currentTime + 7.5);
			});
		}, 2000);

		// D2, A2, D3, A3, D4, A4, D5, A5, D6, and F#6. via https://www.johndcook.com/blog/2018/06/12/mathematics-of-deep-note/

		// for (const node of nodes) {
		// 	// oscillator.frequency.value = 440 * Math.pow(2, (parseInt(node.slice(1)) - 49) / 12);

		// 	const f = findNode(node)?.freq;
		// 	if (!f) continue;
		// 	const oscillator = audio.createOscillator();
		// 	oscillator.type = 'triangle';
		// 	oscillator.frequency.value = f;

		// 	oscillator.connect(gainNode);

		// 	oscillator.start();
		// }

		// const oscillator = audio.createOscillator();
		// oscillator.type = "triangle";
		// oscillator.frequency.value = 440;
		// oscillator.connect(gainNode);

		// oscillator.start();
	}
</script>

<button disabled={playing} on:click={thx}> Play THX </button>

<input type="range" bind:value={pannerValue} min="-1" max="1" step=".01" />
