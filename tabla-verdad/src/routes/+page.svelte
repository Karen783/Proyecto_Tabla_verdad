<script>
/* =========================
   ESTADO
========================= */
let classification = null;
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
	classification = null;
}

function extractVariables(expr) {
	const matches = expr.match(/[a-z]/gi) || [];
	return [...new Set(matches)]
		.filter(v => v !== 't' && v !== 'c')
		.sort();
}

function generateCombinations(vars) {
	const rows = [];
	const total = 2 ** vars.length;

	for (let i = total - 1; i >= 0; i--) {
		const row = {};
		vars.forEach((v, index) => {
			row[v] = Boolean((i >> (vars.length - index - 1)) & 1);
		});
		rows.push(row);
	}
	return rows;
}

/* =========================
   NORMALIZACIÓN LÓGICA
========================= */

function normalizeExpression(expr, values) {
	let e = expr;

	// constantes
	e = e.replaceAll('t', 'true').replaceAll('c', 'false');

	// variables
	for (const [v, val] of Object.entries(values)) {
		e = e.replaceAll(v, val ? 'true' : 'false');
	}

	// implicación
	while (e.includes('→')) {
		e = e.replace(
			/(\([^()]+\)|true|false)\s*→\s*(\([^()]+\)|true|false)/,
			'(!$1 || $2)'
		);
	}

	// bicondicional
	while (e.includes('↔')) {
		e = e.replace(
			/(\([^()]+\)|true|false)\s*↔\s*(\([^()]+\)|true|false)/,
			'($1 === $2)'
		);
	}

	// operadores básicos
	e = e
		.replaceAll('¬', '!')
		.replaceAll('∧', '&&')
		.replaceAll('∨', '||');

	return e;
}

function evaluateExpression(expr) {
	return Function(`"use strict"; return (${expr});`)();
}

/* =========================
   GENERAR TABLA
========================= */

function generateTable() {
	try {
		expression = expression.toLowerCase();

		variables = extractVariables(expression);

		if (variables.length > 10) {
			alert('Máximo 10 variables permitidas');
			return;
		}

		const combinations = generateCombinations(variables);

		table = combinations.map(values => {
			const normalized = normalizeExpression(expression, values);
			const result = evaluateExpression(normalized);
			return { ...values, result };
		});

		const results = table.map(r => r.result);

		if (results.every(r => r === true)) classification = 'T';
		else if (results.every(r => r === false)) classification = 'C';
		else classification = 'CONT';

	} catch (err) {
		classification = null;
		alert('Expresión inválida');
	}
}
</script>

<main class="min-h-screen bg-base-200 flex items-center justify-center p-4">
	<div class="card w-full max-w-4xl bg-base-100 shadow-xl">
		<div class="card-body gap-6">

			<h1 class="card-title text-2xl justify-center">
				Generador de Tablas de Verdad
			</h1>

			<input
				class="input input-bordered text-lg font-mono"
				bind:value={expression}
				on:input={() => expression = expression.toLowerCase()}
				placeholder="Ej: (p ∧ q) → r"
			/>

			<div>
				<p class="font-semibold mb-2">Operadores</p>
				<div class="grid grid-cols-4 gap-2">
					{#each operators as op}
						<button
							class="btn btn-block font-mono"
							on:click={() => insertSymbol(op.value)}
						>
							{op.label}
						</button>
					{/each}
				</div>
			</div>

			<div class="flex justify-end gap-2">
				<button class="btn btn-ghost" on:click={clearExpression}>
					Limpiar
				</button>
				<button class="btn btn-primary" on:click={generateTable}>
					Generar tabla
				</button>
			</div>

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

			{#if classification}
				<div class="alert mt-4
					{classification === 'T' ? 'alert-success' : ''}
					{classification === 'C' ? 'alert-error' : ''}
					{classification === 'CONT' ? 'alert-warning' : ''}">

					{#if classification === 'T'}
						<span class="font-bold">Tautología</span> — siempre verdadera
					{:else if classification === 'C'}
						<span class="font-bold">Contradicción</span> — siempre falsa
					{:else}
						<span class="font-bold">Contingencia</span> — depende de los valores
					{/if}
				</div>
			{/if}

		</div>
	</div>
</main>
