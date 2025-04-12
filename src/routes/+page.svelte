<script lang="ts">
	import { onMount } from 'svelte';
	import { v4 as uuid } from 'uuid';
	import Icon from '@iconify/svelte';
	import Util from '$lib/components/Util.svelte';

	type Path = {
		strokeWidth: number;
		strokeColor: string;
		strokeStyle: 'solid' | 'dashed' | 'dotted';
		position: {
			x: number;
			y: number;
		}[];
	};

	type Canvas = {
		size: {
			width: number;
			height: number;
		};
		position: {
			x: number;
			y: number;
		};
		scale: number;
	};

	let isDrawing = $state(false);

	let canvas: HTMLCanvasElement | null = $state(null);
	let ctx: CanvasRenderingContext2D | null = $state(null);

	let paths: Record<string, Path> | null = $state(null);
	let currentPath: Path | null = $state(null);

	let tool: 'pen' | 'eraser' = $state('pen');
	let eraser: 'object' | 'line' = $state('object');

	let undo: { id: string; path: Path }[] = $state([]);
	let redo: { id: string; path: Path }[] = $state([]);
	let doLimit = $state(30);

	let canvasInfo: Canvas = $state({
		size: {
			width: 2894,
			height: 4093
		},
		position: {
			x: 0,
			y: 0
		},
		scale: 1
	});

	let strokeWidth: number = $state(10);
	let strokeColor: string = $state('#000');
	let strokeStyle: 'solid' | 'dashed' | 'dotted' = $state('solid');

	let dpr = 1;

	let open = $state({
		strokeWidth: false,
		strokeColor: false
	});

	const tabsPos: {
		[key: string]: {
			origin: {
				x: 'left' | 'right';
				y: 'top' | 'bottom';
			};
			x: number;
			y: number;
		};
	} = {
		pen: {
			origin: {
				x: 'left',
				y: 'top'
			},
			x: 10,
			y: 10
		},
		tool: {
			origin: {
				x: 'left',
				y: 'bottom'
			},
			x: 10,
			y: 10
		},
		style: {
			origin: {
				x: 'right',
				y: 'bottom'
			},
			x: 0,
			y: 0
		},
		history: {
			origin: {
				x: 'right',
				y: 'top'
			},
			x: 10,
			y: 10
		}
	};

	const MARGIN = 5;

	const isPointInPath = (path: Path, x: number, y: number): boolean => {
		for (let i = 0; i < path.position.length - 1; i++) {
			const point1 = path.position[i];
			const point2 = path.position[i + 1];
			if (isPointOnLine(point1, point2, x, y, path.strokeWidth)) {
				return true;
			}
		}
		return false;
	};

	const isPointOnLine = (
		point1: { x: number; y: number },
		point2: { x: number; y: number },
		x: number,
		y: number,
		strokeWidth: number
	): boolean => {
		const A = x - point1.x;
		const B = y - point1.y;
		const C = point2.x - point1.x;
		const D = point2.y - point1.y;

		const dot = A * C + B * D;
		const lenSq = C * C + D * D;
		const param = lenSq !== 0 ? dot / lenSq : -1;

		let nearestX, nearestY;

		if (param < 0) {
			nearestX = point1.x;
			nearestY = point1.y;
		} else if (param > 1) {
			nearestX = point2.x;
			nearestY = point2.y;
		} else {
			nearestX = point1.x + param * C;
			nearestY = point1.y + param * D;
		}

		const distance = Math.hypot(x - nearestX, y - nearestY);
		return distance <= strokeWidth + MARGIN;
	};

	const pointerEvent = (e: PointerEvent) => {
		if (!ctx || !canvas) return;

		const rect = canvas.getBoundingClientRect();
		const x = (((e.clientX - rect.left) / rect.width) * canvasInfo.size.width) / canvasInfo.scale;
		const y = (((e.clientY - rect.top) / rect.height) * canvasInfo.size.height) / canvasInfo.scale;

		if (e.type === 'pointerdown') {
			isDrawing = true;
			if (tool === 'pen') {
				currentPath = { strokeWidth, strokeColor, strokeStyle, position: [{ x, y }] };
			} else if (tool === 'eraser') {
				handleEraser(x, y);
			}
		} else if (e.type === 'pointermove' && isDrawing) {
			if (tool === 'pen') {
				if (!currentPath || !isDrawing) return;
				currentPath.position.push({ x, y });
				redraw();
			} else if (tool === 'eraser') {
				handleEraser(x, y);
			}
		} else if (e.type === 'pointerup' || e.type === 'pointerout') {
			if (tool === 'pen') {
				if (isDrawing && currentPath) {
					let id = uuid();
					while ((paths && id in paths) || id in undo) {
						id = uuid();
					}
					undo.push({
						id,
						path: currentPath
					});
					if (undo.length > doLimit) {
						undo.shift();
					}
					redo = [];
					paths = {
						...paths,
						[id]: currentPath
					};
					currentPath = null;
					redraw();
				}
				isDrawing = false;
			}
		}
	};

	const handleEraser = (x: number, y: number) => {
		if (!paths) return;

		if (eraser === 'object') {
			for (const [id, path] of Object.entries(paths)) {
				if (isPointInPath(path, x, y)) {
					delete paths[id];
					undo.push({ id, path });
					if (undo.length > doLimit) undo.shift();
					break;
				}
			}
		} else if (eraser === 'line') {
			for (const [id, path] of Object.entries(paths)) {
				const newPosition = [];
				let isModified = false;

				for (let i = 0; i < path.position.length - 1; i++) {
					const point1 = path.position[i];
					const point2 = path.position[i + 1];

					// 線分が消しゴムの範囲内にあるか判定
					if (!isPointOnLine(point1, point2, x, y, path.strokeWidth)) {
						newPosition.push(point1);
					} else {
						isModified = true;

						// 消しゴムの範囲に応じて新しいパスを作成
						const midX = (point1.x + point2.x) / 2;
						const midY = (point1.y + point2.y) / 2;

						// 前後の点を追加して消した感じを出す
						newPosition.push({ x: point1.x, y: point1.y });
						newPosition.push({ x: midX, y: midY });
					}
				}

				// 最後の点を追加
				if (path.position.length > 0) {
					newPosition.push(path.position[path.position.length - 1]);
				}

				// パスが変更された場合、元のパスを保存して更新
				if (isModified) {
					undo.push({ id, path });
					if (undo.length > doLimit) undo.shift();
					if (newPosition.length === 0) {
						delete paths[id];
					} else {
						paths[id] = { ...path, position: newPosition };
					}
				}
			}
		}
		redraw();
	};

	const redraw = () => {
		if (!ctx || !canvas) return;

		ctx.save();
		ctx.setTransform(1, 0, 0, 1, 0, 0);
		ctx.clearRect(0, 0, canvas.width, canvas.height);
		ctx.restore();

		ctx.save();

		if (paths) {
			Object.values(paths).forEach((path) => {
				if (!ctx || !paths) return;
				ctx.beginPath();
				ctx.strokeStyle = path.strokeColor;
				ctx.lineWidth = path.strokeWidth;
				ctx.setLineDash(
					path.strokeStyle === 'dashed' ? [5, 5] : path.strokeStyle === 'dotted' ? [1, 2] : []
				);
				path.position.forEach((point, index) => {
					if (!ctx || !canvas) return;
					if (index === 0) {
						ctx.moveTo(point.x, point.y);
					} else {
						ctx.lineTo(point.x, point.y);
					}
				});
				ctx.stroke();
			});
		}

		if (currentPath && currentPath.position.length > 0) {
			ctx.beginPath();
			ctx.strokeStyle = currentPath.strokeColor;
			ctx.lineWidth = currentPath.strokeWidth;
			ctx.setLineDash(
				currentPath.strokeStyle === 'dashed'
					? [5, 5]
					: currentPath.strokeStyle === 'dotted'
						? [1, 2]
						: []
			);
			currentPath.position.forEach((point, index) => {
				if (!ctx || !canvas) return;
				if (index === 0) {
					ctx.moveTo(point.x, point.y);
				} else {
					ctx.lineTo(point.x, point.y);
				}
			});
			ctx.stroke();
		}

		ctx.restore();
	};

	const undoAction = () => {
		if (undo.length > 0) {
			const lastUndo = undo.pop()!;
			redo.push(lastUndo);

			if (!paths) return;

			const remainingPaths = Object.entries(paths).filter(([key]) => key !== lastUndo.id);
			paths = Object.fromEntries(remainingPaths);

			redraw();
		}
	};

	const redoAction = () => {
		if (redo.length > 0) {
			const lastRedo = redo.pop()!;
			undo.push(lastRedo);

			paths = {
				...paths,
				[lastRedo.id]: lastRedo.path
			};

			redraw();
		}
	};

	$effect(() => {
		if (canvas && ctx) {
			canvas.width = canvasInfo.size.width * dpr;
			canvas.height = canvasInfo.size.height * dpr;
			ctx.scale(dpr, dpr);
			ctx.clearRect(0, 0, canvas.width, canvas.height);
			redraw();
		}
	});

	onMount(() => {
		if (canvas && typeof window !== 'undefined') {
			dpr = window.devicePixelRatio || 1;
			canvas.width = canvasInfo.size.width * dpr;
			canvas.height = canvasInfo.size.height * dpr;

			ctx = canvas.getContext('2d');
			if (ctx) {
				ctx.scale(dpr, dpr);
				ctx.strokeStyle = strokeColor;
				ctx.lineJoin = 'round';
				ctx.lineCap = 'round';
				ctx.lineWidth = strokeWidth;
			}

			window.addEventListener('resize', redraw);
		}
	});
</script>

<div class="w:100% h:100% rel">
	<canvas
		bind:this={canvas}
		style="
			top: 50%;
			left: 50%;
			transform: translate(calc(-50% + {canvasInfo.position.x}px), calc(-50% + {canvasInfo.position
			.y}px)) scale({canvasInfo.scale});
		"
		class="abs"
		onpointerdown={pointerEvent}
		onpointermove={pointerEvent}
		onpointerup={pointerEvent}
		onpointerout={pointerEvent}
	></canvas>
	<div
		style="
            top: {tabsPos.pen.origin.y === 'top'
			? `${tabsPos.pen.y}px`
			: `calc(100% - ${tabsPos.pen.y}px); transform: translateY(-100%)`};
            left: {tabsPos.pen.origin.x === 'left'
			? `${tabsPos.pen.x}px`
			: `calc(100% - ${tabsPos.pen.x}px); transform: translateX(-100%)`};
        "
		class="w:64px abs bg:#eee flex flex:column justify-content:center r:8px"
	>
		<Util icon="mdi:freehand-line" onclick={() => (open.strokeWidth = !open.strokeWidth)}>
			{#if open.strokeWidth}
				<div
					class="w:100% h:fit abs top:50% translateY(-50%) left:calc(100%+12px) bg:#eee p:4px r:6px flex flex-col justify-content:center align-items:center"
				>
					<input type="range" min="1" max="10" bind:value={strokeWidth} class="w:100%" />
				</div>
			{/if}
		</Util>
		<Util icon="mdi:palette" onclick={() => (open.strokeColor = !open.strokeColor)}>
			{#if open.strokeColor}
				<div
					class="w:100% h:fit abs top:50% translateY(-50%) left:calc(100%+12px) bg:#eee p:4px r:6px flex flex-col justify-content:center align-items:center"
				>
					<input type="color" bind:value={strokeColor} class="w:100%" />
				</div>
			{/if}
		</Util>
	</div>
	<div
		style="
			top: {tabsPos.tool.origin.y === 'top'
			? `${tabsPos.tool.y}px`
			: `calc(100% - ${tabsPos.tool.y}px); transform: translateY(-100%)`};
			left: {tabsPos.tool.origin.x === 'left'
			? `${tabsPos.tool.x}px`
			: `calc(100% - ${tabsPos.tool.x}px); transform: translateX(-100%)`};
		"
		class="w:64px abs top:10px left:10px bg:#eee flex flex:column justify-content:center r:8px"
	>
		<Util
			icon="mdi:pencil"
			class={tool === 'pen' ? 'bg:#e7e7e7' : ''}
			onclick={() => (tool = 'pen')}
		/>
		<Util
			icon="mdi:eraser"
			class={tool === 'eraser' ? 'bg:#e7e7e7' : ''}
			onclick={() => (tool = 'eraser')}
		/>
	</div>
	<div
		style="
			top: {tabsPos.history.origin.y === 'top'
			? `${tabsPos.history.y}px`
			: `calc(100% - ${tabsPos.history.y}px); transform: translateY(-100%)`};
			left: {tabsPos.history.origin.x === 'left'
			? `${tabsPos.history.x}px`
			: `calc(100dvw - ${tabsPos.history.x}px); transform: translateX(-100%)`};
		"
		class="w:64px abs top:10px left:10px bg:#eee flex flex:column justify-content:center r:8px"
	>
		<Util icon="mdi:undo" onclick={undoAction} />
		<Util icon="mdi:redo" onclick={redoAction} />
		<Util
			icon="mdi:magnify-plus-outline"
			onclick={() => (canvasInfo.scale = canvasInfo.scale + 0.1)}
		/>
		<Util
			icon="mdi:magnify-minus-outline"
			onclick={() => (canvasInfo.scale = canvasInfo.scale - 0.1)}
		/>
	</div>
</div>
