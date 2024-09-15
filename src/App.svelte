<script>
	import { onMount } from "svelte";
	import Plotly, { add } from "plotly.js-dist";
	import { onDestroy } from "svelte";
	import Decimal from "decimal.js";

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
			value: 20,
			big: true,
		},
		{
			name: "n",
			label: "n (Block Ok to losse)",
			value: 4,
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
			name: "parityBlockMetadataSize",
			label: "Parity Block Metadata Size (Byte)",
			value: 2373,
		},
		{
			name: "dhtMetadataSize",
			label: "DHT Metadata Size (Byte)",
			value: 674,
		},
		{
			name: "annualFailureRatePercent",
			label: "Annual failure rate of drive (%)",
			value: 0.41, // x/100 = 0.0041
		},
		{
			name: "hoursToRebuild",
			label: "hours to rebuild a failed drive",
			value: 156,
		},
	];

	let inputValues = {};
	inputsConfig.forEach((input) => {
		inputValues[input.name] = input.value;
	});

	let graphContainer;
	let results;

	function factorial(n) {
		if (n === 0 || n === 1) return 1;
		let result = n;
		while (n > 1) {
			n--;
			result *= n;
		}
		return result;
	}

	function countNines(storageDurability) {
		let durabilityPercentage = storageDurability
			.mul(100)
			.toFixed(30)
			.toString(); // Adjust precision as needed

		let tmp = durabilityPercentage.replace(".", "");

		let nineCount = 0;
		for (let char of tmp) {
			if (char === "9") {
				nineCount++;
			} else {
				if (Number(char) > 4) {
					nineCount++;
				}
				break;
			}
		}

		return nineCount;
	}
	function calculate(inputVal) {
		let {
			tbToStore,
			k,
			n,
			chunkSize,
			chunkOverhead,
			parityBlockMetadataSize,
			dhtMetadataSize,
			hoursToRebuild,
			annualFailureRatePercent,
		} = inputVal;

		let bytesToStore = TbToBytes(tbToStore);
		let chunkAmountNeeded = Math.ceil(bytesToStore / chunkSize);

		let overheadFactor = n / k;
		let parityBlockSize = (chunkSize / k) * (1 + overheadFactor);
		let amountOfParityBlocks = k * chunkAmountNeeded;
		let parityAndMetaDHTMetaSize =
			amountOfParityBlocks * (dhtMetadataSize + parityBlockMetadataSize);
		let rawAndErasureCodingOverheadWithMeta =
			parityAndMetaDHTMetaSize +
			bytesToStore * (1 + chunkOverhead) * (1 + overheadFactor);

		Decimal.config({
			precision: 50, // Increase precision
			rounding: Decimal.ROUND_HALF_UP,
		});

		// Durability Calculations with Decimal.js
		let hoursPerYear = new Decimal(365).mul(24);
		let durabilityLambda = new Decimal(annualFailureRatePercent)
			.div(100)
			.mul(k)
			.div(hoursPerYear.div(hoursToRebuild));

		let e = Decimal.exp(-1 * durabilityLambda);

		let poissonProbabilityInRebuildTime = e.mul(
			Decimal.pow(durabilityLambda, n).div(factorial(n)),
		);

		let storageDurability = Decimal.pow(
			new Decimal(1).minus(poissonProbabilityInRebuildTime),
			hoursPerYear.div(hoursToRebuild),
		);

		let nines = countNines(storageDurability);

		return {
			"overhead Factor": overheadFactor,
			"overhead by Erasure Coding": [overheadFactor * 100, "%"],
			"Chunk Amount Needed": [chunkAmountNeeded, "Chunks"],
			"Raw + pure Erasure Coding overhead": [
				BytesToMb(bytesToStore * overheadFactor + bytesToStore),
				"MB",
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
			"Total ParityBlock Meta Size": [
				BytesToMb(amountOfParityBlocks * parityBlockMetadataSize),
				"MB",
				BytesToTb(amountOfParityBlocks * parityBlockMetadataSize),
				"TB",
			],
			"DHTMeta Size relative to ParityBlock": [
				(dhtMetadataSize / parityBlockSize) * 100,
				"%",
			],
			"Total DHTMeta Size": [
				BytesToMb(amountOfParityBlocks * dhtMetadataSize),
				"MB",
				BytesToTb(amountOfParityBlocks * dhtMetadataSize),
				"TB",
			],
			"ParityMeta+DHTMeta relative ParityBlock": [
				((dhtMetadataSize + parityBlockMetadataSize) /
					parityBlockSize) *
					100,
				"%",
			],
			"Total ParityMeta+DHTMeta": [
				BytesToMb(parityAndMetaDHTMetaSize),
				"MB",
				BytesToTb(parityAndMetaDHTMetaSize),
				"TB",
			],
			"Raw + Erasure Coding incl. Metadata": [
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
				(bytesToStore / rawAndErasureCodingOverheadWithMeta) * 100,
				"%",
			],
			"Storage Durability": [
				"" + nines,
				"nines",
				storageDurability.mul(100).toPrecision(15),
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

	let rangeKMin = 3;
	let rangeKMax = 64;
	let rangeNMin = 2;
	let rangeNMax = 32;

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

		let data = [
			{
				x: nRange,
				y: kRange,
				z: zValues,
				type: "surface",
				colorscale: "Viridis",
				showscale: false,
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
			title: "Storage Effectiveness for different k and n values",
			scene: {
				camera: { eye: { x: 3, y: 0.9, z: 1.5 } },
				xaxis: { title: "X: n (Block Ok to lose)" },
				yaxis: { title: "Y: k (Split in Blocks)" },
				zaxis: { title: "Z: Storage Effectiveness (%)" },
			},
			showlegend: false,
			autosize: true,
			// width: 900,
			// height: 900,
			margin: {
				l: 0,
				r: 0,
				b: 0,
				t: 45,
				pad: 0,
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

<svelte:head>
	<script async defer src="https://buttons.github.io/buttons.js"></script>
</svelte:head>

<h1>OuroborosDB Data Storage Calculator</h1>
<p>
	This calculator helps you to estimate the storage space needed for your data
	and the durability of the storage system and what are the most space
	effective erasure settings.<br />
	You can use this calculator for other software, but it is build for
	<a href="https://github.com/i5heu/ouroboros-db">OuroborosDB</a>.
</p>
<p style="display: flex; align-items: center;">
	Give this Project a Github Star:
	<a
		class="github-button"
		href="https://github.com/i5heu/ouroboros-db-overhead-calculator"
		data-color-scheme="no-preference: light; light: light; dark: dark;"
		data-size="large"
		data-show-count="true"
		aria-label="Star i5heu/ouroboros-db-overhead-calculator on GitHub"
		>Star</a
	>
</p>

Single Calculation:
<div class="inputs">
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

<details>
	<summary>Advanced Inputs</summary>
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
</details>

<div class="results">
	<h2>Results:</h2>
	{#if results}
		<p>
			For {inputValues.tbToStore} TB of data you need
			<strong>
				{results["Raw + Erasure Coding incl. Metadata"][0].toFixed(2)} TB</strong
			>
			of storage space.<br />
			This results in a storage effectiveness of
			<strong>{results["Storage Effectiveness"][0].toFixed(2)}%</strong>
			with a data durability of {results["Storage Durability"][0]} nines or
			{results["Storage Durability"][2]}%.
		</p>
		<table>
			{#each Object.keys(results) as key}
				<tr>
					<td>{key}</td>
					{#if typeof results[key] === "object"}
						{#each results[key] as result}
							{#if typeof result === "number"}
								<td>{result.toFixed(2)}</td>
							{:else}
								<td>{result}</td>
							{/if}
						{/each}
					{:else if typeof results[key] === "number"}
						<td>{results[key].toFixed(2)}</td>
					{:else}
						<td>{results[key]}</td>
					{/if}
				</tr>
			{/each}
		</table>
	{/if}
</div>

<div id="graphContainer" bind:this={graphContainer}></div>

Graph:
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

<div class="credits">
	(c) 2024 <a href="https://heidenstedt.org/">Mia Heidenstedt</a> and contributors
</div>

<style>
	:global(html) {
		background: #1e1e2f;
		margin-bottom: 5em;
		padding-bottom: 5em;
	}

	:global(body) {
		font-family: "Roboto", sans-serif;
		background: #1e1e2f;
		color: #ffffff;
		margin: 0;
		padding: 0 5px;
		max-width: 60rem;
		margin: 0 auto;
	}

	input {
		background-color: #333a4d;
		color: #ffffff;
		border: none;
		padding: 10px;
		border-radius: 5px;
		width: 100%;
		margin-right: 5px;
		margin-bottom: 15px;
		transition: all 0.3s ease-in-out;
	}

	input:focus {
		border: 2px solid #00aaff;
	}

	label {
		display: block;
		margin-bottom: 8px;
		font-weight: bold;
		color: #c9d1d9;
	}

	h1 {
		color: #00aaff;
		text-align: center;
		margin-bottom: 30px;
	}

	#graphContainer {
		background-color: #2b2b3d;
		border-radius: 8px;
		padding: 0;
		margin: 20px 0;
		box-shadow: 0 4px 8px rgba(0, 0, 0, 0.5);
		height: 40em;
		overflow: hidden;
	}

	.results {
		background-color: #1f1f2e;
		padding: 20px;
		padding-top: 0;
		border-radius: 8px;
		margin: 30px 0;
		box-shadow: 0 4px 8px rgba(0, 0, 0, 0.5);
		max-width: 100%;
		overflow-x: auto;
	}

	table {
		width: 100%;
		border-collapse: collapse;
	}

	table td {
		padding: 10px;
		border: 1px solid #333a4d;
	}

	.inputs {
		display: grid;
		gap: 1em;
		grid-template-columns: repeat(auto-fill, minmax(150px, 1fr));
		align-items: end;
		margin: 5px 0;
	}

	.input-group {
		margin-bottom: 15px;
	}

	.credits {
		height: 5rem;
		padding: 0px 5rem;
		display: flex;
		align-content: center;
		justify-content: center;
		align-items: center;
	}

	summary {
		cursor: pointer;
		padding-bottom: 1em;
	}
</style>
