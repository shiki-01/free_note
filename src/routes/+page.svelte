<script lang="ts">
	import { onMount } from 'svelte';
    import { v4 as uuid } from 'uuid';
    import Icon from '@iconify/svelte';

	let isDrawing = $state(false);

	let canvas: HTMLCanvasElement | null = $state(null);
	let ctx: CanvasRenderingContext2D | null = $state(null);

    let paths: Record<string, { x: number; y: number }[]> | null = $state(null);
    let currentPath: { x: number; y: number }[] = $state([]);

    let strokeWidth = 2;
    let strokeColor = '#000';

    let dpr = 1;

	const tabsPos = {
		pen: {
			origin: {
				x: 'top',
				y: 'right'
			},
			x: 10,
			y: 10
		},
		tool: {
			origin: {
				x: 'top',
				y: 'left'
			},
			x: 0,
			y: 0
		},
		style: {
			origin: {
				x: 'top',
				y: 'left'
			},
			x: 0,
			y: 0
		}
	};

	const pointerEvent = (e: PointerEvent) => {
        if (!ctx || !canvas) return;

        const rect = canvas.getBoundingClientRect();
        const x = (e.clientX - rect.left) * 1;
        const y = (e.clientY - rect.top) * 1;

        if (e.type === 'pointerdown') {
            isDrawing = true;
            currentPath = [{ x, y }];
        } else if (e.type === 'pointermove' && isDrawing) {
            currentPath.push({ x, y });
            redraw();
        } else if (e.type === 'pointerup' || e.type === 'pointerout') {
            if (isDrawing) {
                paths = {
                    ...paths,
                    [uuid()]: currentPath
                }
                currentPath = [];
            }
            isDrawing = false;
        }
    };

    const redraw = () => {
        if (!ctx || !canvas) return;

        ctx.clearRect(0, 0, canvas.width, canvas.height);

        if (paths) {
            Object.keys(paths).forEach((path) => {
            if (!ctx || !paths) return;
            ctx.beginPath();
            paths[path].forEach((point, index) => {
                if (!ctx) return;
                if (index === 0) {
                    ctx.moveTo(point.x, point.y);
                } else {
                    ctx.lineTo(point.x, point.y);
                }
            });
            ctx.stroke();
        });
        }

        if (currentPath.length > 0) {
            ctx.beginPath();
            currentPath.forEach((point, index) => {
                if (!ctx) return;
                if (index === 0) {
                    ctx.moveTo(point.x, point.y);
                } else {
                    ctx.lineTo(point.x, point.y);
                }
            });
            ctx.stroke();
        }
    };

    onMount(() => {
        if (canvas && typeof window !== 'undefined') {
            dpr = window.devicePixelRatio || 1;
            canvas.width = canvas.clientWidth * dpr;
            canvas.height = canvas.clientHeight * dpr;

            ctx = canvas.getContext('2d');
            if (ctx) {
                ctx.scale(dpr, dpr);
                ctx.strokeStyle = strokeColor;
                ctx.lineJoin = 'round';
                ctx.lineCap = 'round';
                ctx.lineWidth = strokeWidth;
            }
        }
    });
</script>

<div class="w:100% h:100% rel">
	<canvas
		bind:this={canvas}
		class="w:100% h:100% abs"
		onpointerdown={pointerEvent}
		onpointermove={pointerEvent}
		onpointerup={pointerEvent}
		onpointerout={pointerEvent}
	></canvas>
	<div
		style="
            top: {tabsPos.pen.origin.x === 'top' ? `${tabsPos.pen.y}px` : `calc(100% - ${tabsPos.pen.y}px); transform: translateY(100%)`};
            right: {tabsPos.pen.origin.y === 'left' ? `${tabsPos.pen.x}px` : `calc(100% - ${tabsPos.pen.x}px); transform: translateX(100%)`};
        "
		class="w:64px h:200px abs bg:#eee flex flex-col justify-content:center r:8px p:12px"
	>
        <Icon icon="mdi:freehand-line" class="w:32px h:32px" />
    </div>
</div>
