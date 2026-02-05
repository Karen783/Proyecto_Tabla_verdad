<script>
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
		let expresiones = [...new Set(matches)].filter((v) => v !== 't' && v !== 'c').sort();
		return expresiones;
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

	function normalizeExpression(expr, values) {
		let e = expr;

		// constantes
		e = e.replaceAll('t', 'true').replaceAll('c', 'false');

		// variables
		for (const [v, val] of Object.entries(values)) {
			e = e.replace(new RegExp(`\\b${v}\\b`, 'g'), val ? 'true' : 'false');
		}

		// implicación
		while (e.includes('→')) {
			e = e.replace(/(\([^()]+\)|true|false)\s*→\s*(\([^()]+\)|true|false)/, '(!$1 || $2)');
		}

		// bicondicional
		while (e.includes('↔')) {
			e = e.replace(/(\([^()]+\)|true|false)\s*↔\s*(\([^()]+\)|true|false)/, '($1 === $2)');
		}

		// operadores básicos
		e = e.replaceAll('¬', '!').replaceAll('∧', '&&').replaceAll('∨', '||');

		return e;
	}

	function evaluateExpression(expr) {
		return Function(`"use strict"; return (${expr});`)();
	}

	function generateTable() {
		try {
			variables = extractVariables(expression);

			if (variables.length > 10) {
				alert('Máximo 10 variables permitidas');
				return;
			}

			const combinations = generateCombinations(variables);

			table = combinations.map((values) => {
				const normalized = normalizeExpression(expression, values);
				const result = evaluateExpression(normalized);
				return { ...values, result };
			});

			const results = table.map((r) => r.result);
		} catch (err) {
			alert('Expresión inválida');
		}
	}
</script>

<main class="flex min-h-screen items-center justify-center bg-base-200 p-4">
	<div class="card w-full max-w-4xl bg-base-100 shadow-xl">
		<div class="card-body gap-6">
			<h1 class="card-title justify-center text-2xl">Generador de Tablas de Verdad</h1>

			<input
				class=" input-primary input font-mono text-lg"
				bind:value={expression}
				on:input={() => (expression = expression.toLowerCase())}
				placeholder="Ej: (p ∧ q) → r"
			/>

			<div>
				<p class="mb-2 font-semibold">Operadores</p>
				<div class="grid grid-cols-4 gap-2">
					{#each operators as op}
						<button class="btn btn-block font-mono" on:click={() => insertSymbol(op.value)}>
							{op.label}
						</button>
					{/each}
				</div>
			</div>

            <div class="flex justify-between gap-2">
               <button class="btn btn-secondary flex-1" on:click={clearExpression}> Limpiar </button>
               <button class="btn btn-primary flex-1" on:click={generateTable}> Generar tabla </button>
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
		</div>
	</div>
</main>
