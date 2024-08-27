<script>
	import { onMount } from "svelte";
	import Plotly from "plotly.js-dist";
	import { onDestroy } from "svelte";

	let inputsConfig = [
		{
			name: "tbToStore",
			label: "TB to store",
			value: 100,
			big: true,
		},
		{
			name: "k",
			label: "k (Split in Blocks)",
			value: 6,
			big: true,
		},
		{
			name: "n",
			label: "n (Block Ok to losse)",
			value: 3,
			big: true,
		},
		{
			name: "chunkSize",
			label: "Chunk Size (Byte)",
			value: 1310720,
		},
		{
			name: "chunkOverhead",
			label: "Chunk Overhead (%)",
			value: 0.0,
		},
		{
			name: "chunkAmountNeeded",
			label: "Chunk Amount Needed",
			value: 400000000,
		},
		{
			name: "parityBlockMetadataSize",
			label: "Parity Block Metadata Size (Byte)",
			value: 2373,
		},
		{
			name: "dhtMetadataSize",
			label: "DHT Metadata Size (Byte)",
			value: 674,
		},
	];

	let inputValues = {};
	inputsConfig.forEach((input) => {
		inputValues[input.name] = input.value;
	});

	let graphContainer;
	let results;

	function calculate(inputVal) {
		let {
			tbToStore,
			k,
			n,
			chunkSize,
			chunkOverhead,
			parityBlockMetadataSize,
			dhtMetadataSize,
			chunkAmountNeeded,
		} = inputVal;

		let bytesToStore = TbToBytes(tbToStore);

		let overheadFactor = n / k;
		let parityBlockSize = (chunkSize / k) * (1 + overheadFactor);
		let amountOfParityBlocks = k * chunkAmountNeeded;
		let parityAndMetaDHTMetaSize =
			amountOfParityBlocks * (dhtMetadataSize + parityBlockMetadataSize);
		let rawAndErasureCodingOverheadWithMeta =
			parityAndMetaDHTMetaSize +
			bytesToStore * (1 + chunkOverhead) * (1 + overheadFactor);

		return {
			"overhead Factor": overheadFactor,
			"overhead by Erasure Coding": [overheadFactor * 100, "%"],
			"Raw + pure Erasure Coding overhead": [
				bytesToStore * overheadFactor + bytesToStore,
				"Bytes",
				BytesToTb(bytesToStore * overheadFactor + bytesToStore),
				"TB",
			],
			"Parity Block Size": [
				parityBlockSize,
				"Bytes",
				BytesToMb(parityBlockSize),
				"MB",
			],
			"Total ParityBlockSize of Chunk": [
				parityBlockSize * k,
				"Bytes",
				BytesToMb(parityBlockSize * k),
				"MB",
			],
			"ParityMeta Size relative to ParityBlock": [
				(parityBlockMetadataSize / parityBlockSize) * 100,
				"%",
			],
			"Amount of Parity Blocks": [amountOfParityBlocks, "Blocks"],
			"ParityBlock Meta data on 100TB": [
				amountOfParityBlocks * parityBlockMetadataSize,
				"Bytes",
				BytesToMb(amountOfParityBlocks * parityBlockMetadataSize),
				"MB",
			],
			"DHTMeta Size relative to ParityBlock": [
				(dhtMetadataSize / parityBlockSize) * 100,
				"%",
			],
			"DHTMeta Size on 100TB": [
				amountOfParityBlocks * dhtMetadataSize,
				"Bytes",
				BytesToTb(amountOfParityBlocks * dhtMetadataSize),
				"TB",
			],
			"ParityMeta+DHTMeta relative ParityBlock": [
				((dhtMetadataSize + parityBlockMetadataSize) /
					parityBlockSize) *
					100,
				"%",
			],
			"ParityMeta+DHTMeta on 100TB": [
				parityAndMetaDHTMetaSize,
				"Bytes",
				BytesToTb(parityAndMetaDHTMetaSize),
				"TB",
			],
			"Raw + Erasure Coding incl. Metadata": [
				rawAndErasureCodingOverheadWithMeta,
				"Bytes",
				BytesToTb(rawAndErasureCodingOverheadWithMeta),
				"TB",
			],
			"Overhead by Erasure Coding Total": [
				((rawAndErasureCodingOverheadWithMeta - bytesToStore) /
					bytesToStore) *
					100,
				"%",
			],
			"Storage Effectiveness": [
				(1 -
					(rawAndErasureCodingOverheadWithMeta - bytesToStore) /
						bytesToStore) *
					100,
				"%",
			],
		};
	}

	function TbToBytes(tb) {
		return tb * 1024 * 1024 * 1024 * 1024;
	}

	function BytesToTb(bytes) {
		return bytes / 1024 / 1024 / 1024 / 1024;
	}

	function MbToBytes(mb) {
		return mb * 1024 * 1024;
	}

	function BytesToMb(bytes) {
		return bytes / 1024 / 1024;
	}

	let rangeKMin = 6;
	let rangeKMax = 42;
	let rangeNMin = 3;
	let rangeNMax = 21;

	const range = (start, stop, step) =>
		Array.from(
			{ length: (stop - start) / step + 1 },
			(_, i) => start + i * step,
		);

	function generateGraph() {
		results = calculate(inputValues);

		let kRange = range(rangeKMin, rangeKMax, 1);
		let nRange = range(rangeNMin, rangeNMax, 1);

		let zValues = kRange.map((k) => {
			return nRange.map((n) => {
				if (n >= k) {
					return null;
				}

				let result = calculate({
					...inputValues,
					k,
					n,
				});

				return result["Storage Effectiveness"][0]; // use "Storage Effectiveness" as the z-axis value
			});
		});

		console.log(zValues, kRange, nRange);

		let data = [
			{
				x: nRange,
				y: kRange,
				z: zValues,
				type: "surface",
				colorscale: "Viridis",
				contours: {
					z: {
						show: true,
						usecolormap: true,
						highlightcolor: "#42f462",
						project: { z: true },
					},
				},
			},
		];
		let layout = {
			title: "Storage Effectiveness Surface Plot",
			scene: {
				xaxis: { title: "X: n (Block Ok to lose)" },
				yaxis: { title: "Y: k (Split in Blocks)" },
				zaxis: { title: "Z: Storage Effectiveness (%)" },
			},
			showlegend: true,
			autosize: true,
			width: 900,
			height: 900,
			margin: {
				l: 65,
				r: 50,
				b: 65,
				t: 90,
			},
			paper_bgcolor: "rgba(34, 34, 34, 1)",
			plot_bgcolor: "rgba(34, 34, 34, 1)",
			font: {
				color: "#FFF",
			},
		};

		if (graphContainer) {
			Plotly.newPlot(graphContainer, data, layout);
		}
	}

	onMount(() => {
		generateGraph();
	});

	$: graphContainer && generateGraph();

	onDestroy(() => {
		Plotly.purge(graphContainer);
	});
</script>

<body>
	<h1>Data Storage Calculator</h1>
	<div class="bigInputs">
		{#each inputsConfig as input}
			{#if input.big}
				<div class="input-group {input.big ? 'big' : 'small'}">
					<label for={input.name}>{input.label}</label>
					<input
						type="number"
						bind:value={inputValues[input.name]}
						on:change={generateGraph}
						step={input.step || "any"}
					/>
				</div>
			{/if}
		{/each}
	</div>
	<div class="inputs">
		{#each inputsConfig as input}
			{#if !input.big}
				<div class="input-group {input.big ? 'big' : 'small'}">
					<label for={input.name}>{input.label}</label>
					<input
						type="number"
						bind:value={inputValues[input.name]}
						on:change={generateGraph}
						step={input.step || "any"}
					/>
				</div>
			{/if}
		{/each}
	</div>

	<div class="results">
		<h2>Results</h2>
		{#if results}
			<table>
				{#each Object.keys(results) as key}
					<tr>
						<td>{key}</td>
						{#if typeof results[key] === "object"}
							{#each results[key] as result}
								<td>{result}</td>
							{/each}
						{:else}
							<td>{results[key]}</td>
						{/if}
					</tr>
				{/each}
			</table>
		{/if}
	</div>

	<div class="inputs">
		<div class="input-group">
			<label for="kMin">k range min</label>
			<input
				name="kMin"
				type="number"
				bind:value={rangeKMin}
				on:change={generateGraph}
				step="1"
			/>
		</div>
		<div class="input-group">
			<label for="kMax">k range max</label>
			<input
				name="kMax"
				type="number"
				bind:value={rangeKMax}
				on:change={generateGraph}
				step="1"
			/>
		</div>
		<div class="input-group">
			<label for="nMin">n range min</label>
			<input
				name="nMin"
				type="number"
				bind:value={rangeNMin}
				on:change={generateGraph}
				step="1"
			/>
		</div>
		<div class="input-group">
			<label for="nMax">n range max</label>
			<input
				name="nMax"
				type="number"
				bind:value={rangeNMax}
				on:change={generateGraph}
				step="1"
			/>
		</div>
	</div>

	<div id="graphContainer" bind:this={graphContainer}></div>
</body>

<style>
	:global(html) {
		background-color: #222;
	}

	:global(body) {
		color: #fff !important;
	}

	.inputs {
		display: flex;
		flex-wrap: wrap;
	}

	input {
		background-color: #444;
		color: #fff;
		border: 1px solid #666;
		padding: 5px;
		width: 100%;
		margin-bottom: 10px;
	}

	label {
		display: block;
		margin-bottom: 5px;
	}

	#graphContainer {
		width: 100%;
		height: 400px;
	}

	.input-group {
		display: inline-block;
		margin-bottom: 15px;
		padding: 1em;
	}

	.input-group.big {
		width: 50%;
		margin: 0 auto;
		display: block;
	}

	.input-group.small {
		font-size: 0.8em;
	}
</style>
