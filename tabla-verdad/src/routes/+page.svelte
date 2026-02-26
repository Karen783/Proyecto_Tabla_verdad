<script>
	let expression = '';
	let variables = [];
	let table = [];
	let subExpressions = [];

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
        return [...new Set(matches)]
            .filter((v) => v !== 't' && v !== 'c')
            .sort();
    }

	function generateCombinations(vars) {
        const rows = [];
        const total = Math.pow(2, vars.length);
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
        let e = ` ${expr} `; 

        // 1. Reemplazar constantes
        e = e.replace(/\bt\b/g, ' true ').replace(/\bc\b/g, ' false ');

        // 2. Reemplazar variables 
        for (const [v, val] of Object.entries(values)) {
            const regex = new RegExp(`\\b${v}\\b`, 'g');
            e = e.replace(regex, val ? ' true ' : ' false ');
        }

        // 3. Operadores básicos
        e = e.replaceAll('¬', ' ! ')
             .replaceAll('∧', ' && ')
             .replaceAll('∨', ' || ');

        // 4. Implicación y Bicondicional 
        let prev;
        do {
            prev = e;
            e = e.replace(/(\([^()]+\)|true|false)\s*→\s*(\([^()]+\)|true|false)/g, '(! $1 || $2)');
            e = e.replace(/(\([^()]+\)|true|false)\s*↔\s*(\([^()]+\)|true|false)/g, '($1 === $2)');
        } while (e !== prev);

        return e;
    }

	function evaluateExpression(expr) {
        try {
            return !!(new Function(`return ${expr}`)());
        } catch (e) {
            return false;
        }
    }

	function generateTable() {
        if (!expression) return;
        try {
            variables = extractVariables(expression);
            
            const parenRegex = /\(([^()]+)\)/g;
            let match;
            subExpressions = [];
            while ((match = parenRegex.exec(expression)) !== null) {
                if (match[1].length > 1) subExpressions.push(match[1]);
            }
            subExpressions = [...new Set(subExpressions)];

            const combinations = generateCombinations(variables);

            table = combinations.map((values) => {
                const row = { ...values };
                
                // Evaluar columnas intermedias
                subExpressions.forEach(sub => {
                    row[sub] = evaluateExpression(normalizeExpression(sub, values));
                });

                // Evaluar resultado final
                const finalNorm = normalizeExpression(expression, values);
                row.result = evaluateExpression(finalNorm);
                
                return row;
            });
        } catch (err) {
            alert('Error en la expresión');
        }
    }
</script>

<main class="flex min-h-screen items-center justify-center bg-base-200 p-4">
	<div class="card w-full max-w-4xl bg-base-100 shadow-xl">
		<div class="card-body gap-6">
			<h1 class="card-title justify-center text-2xl">Generador de Tablas de Verdad</h1>

			<input
				class="input-primary input font-mono text-lg w-full"
				bind:value={expression}
				on:input={() => (expression = expression.toLowerCase())}
				placeholder="Ej: (p ∧ q) → r"
			/>

			<div>
				<p class="mb-2 font-semibold">Operadores</p>
				<div class="grid grid-cols-4 gap-2">
					{#each operators as op}
						<button class="btn btn-soft font-mono" on:click={() => insertSymbol(op.value)}>
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
									<th class="bg-base-300">{v}</th>
								{/each}

								{#each subExpressions as sub}
									<th class="text-secondary italic">{sub}</th>
								{/each}

								<th class="bg-primary text-primary-content">Resultado</th>
							</tr>
						</thead>
						<tbody>
							{#each table as row}
								<tr>
									{#each variables as v}
										<td>{row[v] ? 'V' : 'F'}</td>
									{/each}

									{#each subExpressions as sub}
										<td class="text-secondary">{row[sub] ? 'V' : 'F'}</td>
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
