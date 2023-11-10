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
		console.log('done');
		playing = false;
		fadeout();
		// start fade
	}

	function fadeout() {
		oscillators.forEach(({ oscillator }) => {
			gainNode.gain.linearRampToValueAtTime(0.0001, oscillator.context.currentTime + 5);
		});
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

		gainNode = audio.createGain();

		gainNode.gain.value = 0.2;

		gainNode.connect(audio.destination);

		for (let i = 0; i < nodes.length; i++) {
			// add 3 oscillators per note, starting at a frequency randomly between 200 - 400
			for (let j = 0; j < 3; j++) {
				const oscillator = audio.createOscillator();
				oscillator.type = 'triangle';
				const startingValue = 0 + Math.random() * 200;
				const store = tweenedStore(startingValue, oscillator);
				oscillators.push({ oscillator, freq: store, node: nodes[i] });
				oscillator.connect(gainNode);
				oscillator.start();
			}
		}

		setTimeout(() => {
			for (const oscillator of oscillators) {
				const f = findNode(oscillator.node)?.freq;
				if (!f) continue;
				// give f a lil jitter
				// oscillator.oscillator.frequency.exponentialRampToValueAtTime(
				// 	f + Math.random() * 25 - 12.5,
				// 	oscillator.oscillator.context.currentTime + 5
				// );
				// setTimeout(fadeout, 5 * 1000);
				// Two ways to do this - above is "proper" way, but with our tweened store way we can set the value directly
				oscillator.freq.set(f + Math.random() * 25 - 12.5).then(() => {
					doneCount++;
				});
				// oscillator.freq.set(f);
			}
		}, 2500);

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
