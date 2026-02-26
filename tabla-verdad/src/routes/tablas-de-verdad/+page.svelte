<script>
    let expression = '';
    let variables = [];
    let table = [];
    let subExpressions = [];
    let clasificacion = '';

    const operators = [
        { label: '¬', value: '¬' }, { label: '∧', value: '∧' },
        { label: '∨', value: '∨' }, { label: '→', value: '→' },
        { label: '↔', value: '↔' }, { label: '(', value: '(' },
        { label: ')', value: ')' }
    ];

    function insertSymbol(symbol) { expression += symbol; }

    function clearExpression() {
        expression = ''; variables = []; table = []; subExpressions = []; clasificacion = '';
    }

    function extractVariables(expr) {
        const matches = expr.match(/[a-z]/gi) || [];
        return [...new Set(matches)].filter((v) => v !== 't' && v !== 'c').sort();
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

    // --- AQUÍ ESTÁ LA MAGIA CORREGIDA ---
    function normalizeExpression(expr, values) {
        let e = ` ${expr} `;

        // 1. Reemplazo de variables/constantes
        e = e.replace(/\bt\b/g, ' true ').replace(/\bc\b/g, ' false ');
        for (const [v, val] of Object.entries(values)) {
            e = e.replace(new RegExp(`\\b${v}\\b`, 'g'), val ? ' true ' : ' false ');
        }

        // 2. Operadores básicos
        e = e.replaceAll('¬', ' ! ').replaceAll('∧', ' && ').replaceAll('∨', ' || ');

        // 3. Resolución INTELIGENTE de Implicación (De adentro hacia afuera)
        
        // Regex para encontrar paréntesis que contienen flechas pero NO otros paréntesis dentro
        // Esto encuentra los bloques más internos: ej: (true → false)
        const innerArrowRegex = /\(([^()]*?(?:→|↔)[^()]*?)\)/;
        
        // Mientras queden paréntesis internos con flechas, los resolvemos primero
        while (innerArrowRegex.test(e)) {
            e = e.replace(innerArrowRegex, (match, content) => {
                let resolved = content;
                // Resolvemos las flechas en este bloque plano
                let prev;
                do {
                    prev = resolved;
                    resolved = resolved.replace(/([^→↔]+)\s*→\s*([^→↔]+)/, '(!($1)||$2)');
                    resolved = resolved.replace(/([^→↔]+)\s*↔\s*([^→↔]+)/, '(($1)===($2))');
                } while (prev !== resolved);
                return `(${resolved})`; // Mantenemos los paréntesis exteriores
            });
        }

        // 4. Resolver las flechas restantes (Nivel superior / sin paréntesis)
        let prev;
        do {
            prev = e;
            e = e.replace(/([^→↔]+)\s*→\s*([^→↔]+)/, '(!($1)||$2)');
            e = e.replace(/([^→↔]+)\s*↔\s*([^→↔]+)/, '(($1)===($2))');
        } while (prev !== e);

        return e;
    }

    function evaluateExpression(expr) {
        try { return !!(new Function(`return ${expr}`)()); } catch (e) { return false; }
    }

    function analizarResultado(datos) {
        if (datos.length === 0) return '';
        const resultados = datos.map(r => r.result);
        if (resultados.every(res => res === true)) return 'Tautología';
        if (resultados.every(res => res === false)) return 'Contradicción';
        return 'Contingencia';
    }

    function generateTable() {
        if (!expression) return;
        try {
            variables = extractVariables(expression);
            
            // Sub-expresiones para columnas intermedias
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
                subExpressions.forEach(sub => {
                    row[sub] = evaluateExpression(normalizeExpression(sub, values));
                });
                const finalNorm = normalizeExpression(expression, values);
                row.result = evaluateExpression(finalNorm);
                return row;
            });

            clasificacion = analizarResultado(table);
        } catch (err) {
            alert('Error. Revisa paréntesis.');
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
				<p class="mb-2 font-semibold text-sm opacity-70">Operadores</p>
				<div class="grid grid-cols-4 gap-2">
					{#each operators as op}
						<button 
							class="btn btn-soft border-none bg-base-200 hover:bg-base-300 text-base-content font-mono shadow-sm" 
							on:click={() => insertSymbol(op.value)}>
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
                <div class="divider">Análisis Lógico</div>

                <div class="flex flex-col items-center gap-2 mb-4">
                    <div class="badge badge-lg p-4 font-bold
                        {clasificacion === 'Tautología' ? 'badge-success text-white' : 
                         clasificacion === 'Contradicción' ? 'badge-error text-white' : 
                         'badge-warning text-warning-content'}">
                        {clasificacion}
                    </div>
                    <p class="text-xs opacity-60 italic">
                        {#if clasificacion === 'Tautología'}
                            (La expresión es siempre verdadera para cualquier valor)
                        {:else if clasificacion === 'Contradicción'}
                            (La expresión es siempre falsa)
                        {:else}
                            (El resultado depende de los valores de las variables)
                        {/if}
                    </p>
                </div>

                <div class="overflow-x-auto rounded-lg border border-base-300">
                    <table class="table table-zebra text-center">
                        <thead>
                            <tr class="bg-base-200">
                                {#each variables as v}
                                    <th class="border-x border-base-300">{v}</th>
                                {/each}

                                {#each subExpressions as sub}
                                    <th class="text-secondary italic border-x border-base-300">{sub}</th>
                                {/each}

                                <th class="bg-primary text-primary-content border-l border-primary">Resultado</th>
                            </tr>
                        </thead>
                        <tbody>
                            {#each table as row}
                                <tr>
                                    {#each variables as v}
                                        <td class="border-x border-base-300">{row[v] ? 'V' : 'F'}</td>
                                    {/each}

                                    {#each subExpressions as sub}
                                        <td class="text-secondary opacity-80 border-x border-base-300">
                                            {row[sub] ? 'V' : 'F'}
                                        </td>
                                    {/each}

                                    <td class="font-bold border-l border-base-300
                                        {row.result ? 'text-success' : 'text-error'}">
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