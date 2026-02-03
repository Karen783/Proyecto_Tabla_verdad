<script>
	/* =========================
	   ESTADO
	========================= */
	let expression = '';
	let variables = [];
	let table = [];

	const operators = [
		{ label: '¬', value: '¬' },
		{ label: '∧', value: '∧' },
		{ label: '∨', value: '∨' },
		{ label: '→', value: '→' },
		{ label: '↔', value: '↔' },
		{ label: '(', value: '(' },
		{ label: ')', value: ')' }
	];

	/* =========================
	   UTILIDADES
	========================= */

	function insertSymbol(symbol) {
		expression += symbol;
	}

	function clearExpression() {
		expression = '';
		variables = [];
		table = [];
	}

	function extractVariables(expr) {
		const matches = expr.match(/[a-z]/gi) || [];
		return [...new Set(matches)].sort();
	}

	function generateCombinations(vars) {
		const rows = [];
		const total = 2 ** vars.length;

		for (let i = 0; i < total; i++) {
			const row = {};
			vars.forEach((v, index) => {
				row[v] = Boolean((i >> (vars.length - index - 1)) & 1);
			});
			rows.push(row);
		}
		return rows;
	}

	function normalizeExpression(expr, values) {
		let e = expr;

		// variables
		for (const [v, val] of Object.entries(values)) {
			e = e.replaceAll(v, val ? 'true' : 'false');
		}

		// operadores lógicos
		e = e
			.replaceAll('¬', '!')
			.replaceAll('∧', '&&')
			.replaceAll('∨', '||')
			.replaceAll('→', '(!A || B)')
			.replaceAll('↔', '(A === B)');

		// implicación y bicondicional (manual)
		e = e.replace(/(!?true|!?false|\([^()]+\))\s*→\s*(!?true|!?false|\([^()]+\))/g,
			'(!$1 || $2)'
		);

		e = e.replace(/(!?true|!?false|\([^()]+\))\s*↔\s*(!?true|!?false|\([^()]+\))/g,
			'($1 === $2)'
		);

		return e;
	}

	function evaluateExpression(expr) {
		return Function(`"use strict"; return (${expr});`)();
	}

	/* =========================
	   ACCIÓN PRINCIPAL
	========================= */

	function generateTable() {
		try {
			variables = extractVariables(expression);
			const combinations = generateCombinations(variables);

			table = combinations.map(values => {
				const normalized = normalizeExpression(expression, values);
				const result = evaluateExpression(normalized);
				return { ...values, result };
			});
		} catch (err) {
			alert('Expresión inválida');
			console.error(err);
		}
	}
</script>

<main class="min-h-screen bg-base-200 flex items-center justify-center p-4">
	<div class="card w-full max-w-4xl bg-base-100 shadow-xl">
		<div class="card-body gap-6">

			<h1 class="card-title text-2xl justify-center">
				Generador de Tablas de Verdad
			</h1>

			<!-- INPUT -->
			<input
				class="input input-bordered text-lg font-mono"
				bind:value={expression}
				placeholder="Ej: (p ∧ q) → r"
			/>

			<!-- TECLADO -->
			<div>
				<p class="font-semibold mb-2">Operadores</p>
				<div class="grid grid-cols-4 gap-2">
					{#each operators as op}
						<button
							class="btn btn-outline font-mono"
							on:click={() => insertSymbol(op.value)}
						>
							{op.label}
						</button>
					{/each}
				</div>
			</div>

			<!-- BOTONES -->
			<div class="flex justify-end gap-2">
				<button class="btn btn-ghost" on:click={clearExpression}>
					Limpiar
				</button>
				<button class="btn btn-primary" on:click={generateTable}>
					Generar tabla
				</button>
			</div>

			<!-- TABLA -->
			{#if table.length}
				<div class="divider">Tabla de verdad</div>

				<div class="overflow-x-auto">
					<table class="table table-zebra text-center">
						<thead>
							<tr>
								{#each variables as v}
									<th>{v}</th>
								{/each}
								<th>Resultado</th>
							</tr>
						</thead>
						<tbody>
							{#each table as row}
								<tr>
									{#each variables as v}
										<td>{row[v] ? 'V' : 'F'}</td>
									{/each}
									<td class="font-bold">
										{row.result ? 'V' : 'F'}
									</td>
								</tr>
							{/each}
						</tbody>
					</table>
				</div>
			{/if}

		</div>
	</div>
</main>

