<script>
	// ── Tipos y constantes ──────────────────────────────────────────
	const OPERACIONES = [
		{ label: '∪  Unión', value: 'union' },
		{ label: '∩  Intersección', value: 'intersection' },
		{ label: '−  Diferencia (A−B)', value: 'difference' },
		{ label: '△  Dif. Simétrica', value: 'symmetric_difference' }
	];

	const OP_ICONS = {
		union: '∪',
		intersection: '∩',
		difference: '−',
		symmetric_difference: '△'
	};

	const OP_DESC = {
		union: 'Todo lo que está en A, en B, o en ambos. Ningún elemento se pierde.',
		intersection: 'Solo los elementos que están en A y en B al mismo tiempo.',
		difference: 'Lo que está en A pero no en B. No es conmutativa: A−B ≠ B−A.',
		symmetric_difference: 'Los elementos exclusivos de cada conjunto. Equivale a (A∪B) − (A∩B).'
	};

	// ── Estado ──────────────────────────────────────────────────────
	let conjuntos = []; // [{ nombre: string, elementos: Set }]
	let seleccionados = []; // índices seleccionados
	let operacion = 'union';
	let editandoIdx = null;
	let editNombre = '';
	let errorMsg = '';

	// ── Helpers de lógica ───────────────────────────────────────────
	function esValido(token) {
		return /^[A-Za-z0-9_]+$/.test(token);
	}

	function parsearLinea(linea) {
		linea = linea.trim();
		if (!linea) return null;
		if (linea.includes(',')) {
			const tokens = linea.split(',').map((t) => t.trim());
			if (tokens.some((t) => !esValido(t))) return null;
			return tokens.sort().join(','); // subconjunto como string ordenado
		}
		return esValido(linea) ? linea : null;
	}

	function elementoStr(elem) {
		if (elem.includes(',')) return '{ ' + elem + ' }';
		return elem;
	}

	function operarConjuntos(sets, op) {
		let resultado = new Set(sets[0]);
		for (let i = 1; i < sets.length; i++) {
			const b = sets[i];
			if (op === 'union') {
				b.forEach((e) => resultado.add(e));
			} else if (op === 'intersection') {
				resultado = new Set([...resultado].filter((e) => b.has(e)));
			} else if (op === 'difference') {
				resultado = new Set([...resultado].filter((e) => !b.has(e)));
			} else if (op === 'symmetric_difference') {
				const union = new Set([...resultado, ...b]);
				const inter = new Set([...resultado].filter((e) => b.has(e)));
				resultado = new Set([...union].filter((e) => !inter.has(e)));
			}
		}
		return resultado;
	}

	// ── Resultado reactivo ──────────────────────────────────────────
	$: setsSeleccionados = seleccionados.map((i) => conjuntos[i]);

	$: resultado = (() => {
		if (setsSeleccionados.length < 2) return null;
		const sets = setsSeleccionados.map((c) => c.elementos);
		return operarConjuntos(sets, operacion);
	})();

	$: resultadoTexto = (() => {
		if (!resultado) return '';
		if (resultado.size === 0) return '∅  (conjunto vacío)';
		return '{ ' + [...resultado].map(elementoStr).sort().join(',  ') + ' }';
	})();

	// ── Carga de archivo ────────────────────────────────────────────
	async function cargarArchivo(event) {
		const file = event.target.files[0];
		if (!file) return;
		event.target.value = ''; // reset input

		const texto = await file.text();
		const lineas = texto.split('\n');
		const elementos = new Set();
		const invalidas = [];

		lineas.forEach((linea, i) => {
			if (!linea.trim()) return;
			const elem = parsearLinea(linea);
			if (elem === null) invalidas.push(i + 1);
			else elementos.add(elem);
		});

		if (elementos.size === 0) {
			errorMsg = 'El archivo no tiene elementos válidos.';
			return;
		}

		// Nombre único basado en el nombre del archivo
		let nombre = file.name.replace(/\.[^.]+$/, '');
		const existentes = new Set(conjuntos.map((c) => c.nombre));
		let n = 2;
		let base = nombre;
		while (existentes.has(nombre)) nombre = `${base} (${n++})`;

		conjuntos = [...conjuntos, { nombre, elementos }];
		errorMsg = invalidas.length ? `Se ignoraron ${invalidas.length} línea(s) inválida(s).` : '';
	}

	function eliminar(idx) {
		conjuntos = conjuntos.filter((_, i) => i !== idx);
		seleccionados = seleccionados.filter((i) => i !== idx).map((i) => (i > idx ? i - 1 : i));
	}

	function iniciarEdicion(idx) {
		editandoIdx = idx;
		editNombre = conjuntos[idx].nombre;
	}

	function guardarEdicion() {
		if (editNombre.trim()) {
			conjuntos[editandoIdx].nombre = editNombre.trim();
			conjuntos = [...conjuntos]; // trigger reactividad
		}
		editandoIdx = null;
	}

	function toggleSeleccion(idx) {
		if (seleccionados.includes(idx)) {
			seleccionados = seleccionados.filter((i) => i !== idx);
		} else {
			seleccionados = [...seleccionados, idx];
		}
	}
</script>

<!-- ── UI ──────────────────────────────────────────────────────────── -->
<main class="min-h-screen bg-base-200 p-6">
	<div class="mx-auto flex max-w-3xl flex-col gap-6">
		<!-- Header -->
		<div class="flex items-center justify-between">
			<div>
				<a href="/" class="text-sm text-slate-500 hover:underline">← Volver</a>
				<h1 class="mt-1 text-3xl font-extrabold text-slate-800">Calculadora de Conjuntos</h1>
			</div>

			<!-- Botón cargar archivo (dispara input oculto) -->
			<label class="btn cursor-pointer border-none bg-slate-500 text-white hover:bg-slate-600">
				＋ Agregar conjunto
				<input type="file" accept=".txt" class="hidden" onchange={cargarArchivo} />
			</label>
		</div>

		{#if errorMsg}
			<div class="alert text-sm alert-warning">{errorMsg}</div>
		{/if}

		<!-- Lista de conjuntos -->
		<div class="card bg-base-100 shadow">
			<div class="card-body gap-3 p-4">
				<h2 class="text-lg font-bold text-slate-700">Conjuntos cargados</h2>

				{#if conjuntos.length === 0}
					<p class="py-6 text-center text-sm text-slate-400 italic">
						No hay conjuntos aún. Carga un archivo .txt donde cada línea sea un elemento.
					</p>
				{:else}
					{#each conjuntos as c, i}
						<div class="flex items-center gap-3 rounded-xl bg-base-200 p-3">
							<!-- Checkbox de selección -->
							<input
								type="checkbox"
								class="checkbox checkbox-sm"
								checked={seleccionados.includes(i)}
								onchange={() => toggleSeleccion(i)}
							/>

							<!-- Nombre (editable inline) -->
							{#if editandoIdx === i}
								<input
									class="input-bordered input input-sm w-32"
									bind:value={editNombre}
									onblur={guardarEdicion}
									onkeydown={(e) => e.key === 'Enter' && guardarEdicion()}
									autofocus
								/>
							{:else}
								<span class="w-32 truncate font-bold text-slate-700">{c.nombre}</span>
							{/if}

							<!-- Tags de elementos (preview) -->
							<div class="flex flex-1 flex-wrap gap-1">
								{#each [...c.elementos].slice(0, 6) as elem}
									<span class="badge border-none bg-purple-200 badge-sm text-purple-900">
										{elementoStr(elem)}
									</span>
								{/each}
								{#if c.elementos.size > 6}
									<span class="badge badge-ghost badge-sm italic">
										+{c.elementos.size - 6} más
									</span>
								{/if}
							</div>

							<!-- Acciones -->
							<button class="btn btn-ghost btn-xs" onclick={() => iniciarEdicion(i)}>✏️</button>
							<button class="btn btn-ghost btn-xs" onclick={() => eliminar(i)}>🗑️</button>
						</div>
					{/each}
				{/if}
			</div>
		</div>

		<!-- Panel de operaciones (visible si hay 2+ conjuntos) -->
		{#if conjuntos.length >= 2}
			<div class="card bg-base-100 shadow">
				<div class="card-body gap-4">
					<h2 class="text-lg font-bold text-slate-700">Operaciones</h2>

					<!-- Selector de operación -->
					<div class="flex flex-wrap gap-3">
						{#each OPERACIONES as op}
							<label class="flex cursor-pointer items-center gap-2">
								<input
									type="radio"
									class="radio radio-sm"
									name="operacion"
									value={op.value}
									bind:group={operacion}
								/>
								<span class="text-sm font-medium">{op.label}</span>
							</label>
						{/each}
					</div>

					<!-- Resultado -->
					{#if setsSeleccionados.length < 2}
						<div class="alert text-sm alert-info">
							☑️ Selecciona al menos 2 conjuntos con los checkboxes de arriba.
						</div>
					{:else}
						<div class="flex flex-col gap-2 rounded-xl bg-base-200 p-4">
							<p class="text-sm font-bold text-purple-800">
								{setsSeleccionados.map((c) => c.nombre).join(` ${OP_ICONS[operacion]} `)}
								→ {resultado?.size ?? 0} elemento(s)
							</p>
							<p class="text-xs text-slate-500 italic">{OP_DESC[operacion]}</p>
							<pre
								class="rounded-lg bg-base-100 p-3 text-sm break-words whitespace-pre-wrap text-slate-800">{resultadoTexto}
                                </pre>
						</div>
					{/if}
				</div>
			</div>
		{/if}
	</div>
</main>
