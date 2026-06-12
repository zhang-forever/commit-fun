<script lang="ts">
  import { onMount } from "svelte";

  let username = $state("");
  let tiles: { x: number; y: number; color: string; count: number; date: string }[] = $state([]);
  let loading = $state(false);
  let error = $state("");
  let canvas: HTMLCanvasElement | undefined = $state();
  let ctx: CanvasRenderingContext2D | null = $state(null);
  let selectedIdx = $state(-1);
  let hoveredTile: { x: number; y: number; count: number; date: string } | null = $state(null);

  const COLORS = [
    "#161b22", "#0e4429", "#006d32", "#26a641", "#39d353",
  ];

  const TILE_SIZE = 13;
  const GAP = 3;
  const OFFSET_X = 28;
  const OFFSET_Y = 16;
  const COLS = 53;

  function getColor(count: number): string {
    if (count === 0) return COLORS[0];
    if (count <= 2) return COLORS[1];
    if (count <= 5) return COLORS[2];
    if (count <= 10) return COLORS[3];
    return COLORS[4];
  }

  function drawMap() {
    if (!ctx || !canvas) return;
    const w = canvas.width;
    const h = canvas.height;
    ctx.clearRect(0, 0, w, h);

    // Background
    ctx.fillStyle = "#ffffff";
    ctx.fillRect(0, 0, w, h);

    // Day-of-week labels
    ctx.fillStyle = "#666";
    ctx.font = "9px monospace";
    const days = ["", "Mon", "", "Wed", "", "Fri", ""];
    days.forEach((d, i) => {
      if (d) ctx.fillText(d, 1, OFFSET_Y + i * (TILE_SIZE + GAP) + 10);
    });

    // Tiles
    for (let i = 0; i < tiles.length; i++) {
      const t = tiles[i];
      const tx = OFFSET_X + t.x * (TILE_SIZE + GAP);
      const ty = OFFSET_Y + t.y * (TILE_SIZE + GAP);

      ctx.fillStyle = t.color;
      ctx.beginPath();
      ctx.roundRect(tx, ty, TILE_SIZE, TILE_SIZE, 2);
      ctx.fill();

      // Selection highlight
      if (i === selectedIdx) {
        ctx.strokeStyle = "#2563eb";
        ctx.lineWidth = 2;
        ctx.beginPath();
        ctx.roundRect(tx - 1, ty - 1, TILE_SIZE + 2, TILE_SIZE + 2, 3);
        ctx.stroke();
      }
    }

    // Tooltip
    if (hoveredTile) {
      const tx = OFFSET_X + hoveredTile.x * (TILE_SIZE + GAP);
      const ty = OFFSET_Y + hoveredTile.y * (TILE_SIZE + GAP) - 28;
      const text = `${hoveredTile.date}: ${hoveredTile.count} event${hoveredTile.count !== 1 ? "s" : ""}`;
      ctx.font = "10px monospace";
      const tw = ctx.measureText(text).width + 12;
      ctx.fillStyle = "#1f2937";
      ctx.beginPath();
      ctx.roundRect(tx - 4, ty - 2, tw, 18, 4);
      ctx.fill();
      ctx.fillStyle = "#fff";
      ctx.fillText(text, tx + 2, ty + 12);
    }

    // Username
    ctx.fillStyle = "#666";
    ctx.font = "10px monospace";
    const labelY = OFFSET_Y + (Math.floor((tiles.length - 1) / COLS) + 1) * (TILE_SIZE + GAP) + 14;
    ctx.fillText(`@${username}`, OFFSET_X, Math.min(labelY, h - 12));
  }

  async function fetchCommits() {
    if (!username.trim()) return;
    loading = true;
    error = "";
    selectedIdx = -1;
    try {
      const res = await fetch(
        `https://api.github.com/users/${encodeURIComponent(username.trim())}/events/public`
      );
      if (res.status === 403) {
        error = "Rate limited. Try again later.";
        loading = false;
        return;
      }
      if (!res.ok) {
        error = `API error (${res.status}). User not found or unavailable.`;
        loading = false;
        return;
      }
      const events = await res.json();
      if (!Array.isArray(events)) {
        error = "User not found or no public events.";
        loading = false;
        return;
      }

      const byDate: Record<string, number> = {};
      for (const e of events) {
        const date = e.created_at?.slice(0, 10);
        if (date) byDate[date] = (byDate[date] || 0) + 1;
      }

      const sorted = Object.entries(byDate).sort(([a], [b]) => a.localeCompare(b));
      tiles = sorted.map(([date, count], i) => ({
        x: i % COLS,
        y: Math.floor(i / COLS),
        color: getColor(count),
        count,
        date,
      }));
    } catch {
      error = "Network error. Check your connection and try again.";
    }
    loading = false;
  }

  function handleKeydown(e: KeyboardEvent) {
    if (tiles.length === 0) return;
    const maxIdx = tiles.length - 1;
    let idx = selectedIdx;

    switch (e.key) {
      case "ArrowRight":
        idx = Math.min(idx + 1, maxIdx);
        e.preventDefault();
        break;
      case "ArrowLeft":
        idx = Math.max(idx - 1, 0);
        e.preventDefault();
        break;
      case "ArrowDown":
        idx = Math.min(idx + COLS, maxIdx);
        e.preventDefault();
        break;
      case "ArrowUp":
        idx = Math.max(idx - COLS, 0);
        e.preventDefault();
        break;
      case "Home":
        idx = 0;
        e.preventDefault();
        break;
      case "End":
        idx = maxIdx;
        e.preventDefault();
        break;
      default:
        return;
    }

    if (selectedIdx === -1) idx = 0;
    selectedIdx = idx;
    const t = tiles[idx];
    hoveredTile = { x: t.x, y: t.y, count: t.count, date: t.date };
    drawMap();
  }

  function handleCanvasClick(e: MouseEvent) {
    if (!canvas) return;
    const rect = canvas.getBoundingClientRect();
    const mx = e.clientX - rect.left;
    const my = e.clientY - rect.top;

    for (let i = 0; i < tiles.length; i++) {
      const t = tiles[i];
      const tx = OFFSET_X + t.x * (TILE_SIZE + GAP);
      const ty = OFFSET_Y + t.y * (TILE_SIZE + GAP);
      if (mx >= tx && mx <= tx + TILE_SIZE && my >= ty && my <= ty + TILE_SIZE) {
        selectedIdx = i;
        hoveredTile = { x: t.x, y: t.y, count: t.count, date: t.date };
        drawMap();
        return;
      }
    }
    selectedIdx = -1;
    hoveredTile = null;
    drawMap();
  }

  function handleCanvasMove(e: MouseEvent) {
    if (!canvas) return;
    const rect = canvas.getBoundingClientRect();
    const mx = e.clientX - rect.left;
    const my = e.clientY - rect.top;
    let found = false;

    for (const t of tiles) {
      const tx = OFFSET_X + t.x * (TILE_SIZE + GAP);
      const ty = OFFSET_Y + t.y * (TILE_SIZE + GAP);
      if (mx >= tx && mx <= tx + TILE_SIZE && my >= ty && my <= ty + TILE_SIZE) {
        hoveredTile = { x: t.x, y: t.y, count: t.count, date: t.date };
        found = true;
        break;
      }
    }
    if (!found) hoveredTile = null;
    drawMap();
  }

  onMount(() => {
    if (canvas) ctx = canvas.getContext("2d");
    const params = new URLSearchParams(window.location.search);
    const u = params.get("u");
    if (u) {
      username = u;
      fetchCommits();
    }
  });

  $effect(() => {
    if (tiles.length > 0) drawMap();
  });
</script>

<main style="max-width:960px;margin:0 auto;padding:40px;font-family:monospace">
  <h1 style="font-size:24px;font-weight:800">Commit.fun</h1>
  <p style="color:#666;margin-bottom:24px">Your GitHub activity, rendered as an 8-bit game map.</p>

  <div style="display:flex;gap:10px;margin-bottom:24px">
    <input bind:value={username} onkeydown={(e) => e.key === "Enter" && fetchCommits()}
      placeholder="GitHub username" style="flex:1;padding:10px 16px;border:2px solid #e0e0e0;border-radius:8px;font-size:14px;font-family:monospace" />
    <button onclick={fetchCommits} disabled={loading}
      style="padding:10px 24px;background:#2563eb;color:#fff;border:none;border-radius:8px;font-weight:600;font-family:monospace;cursor:pointer;opacity:{loading ? 0.6 : 1}">
      {loading ? "Loading…" : "Render"}
    </button>
  </div>

  {#if error}
    <p style="color:#ef4444;margin-bottom:16px">⚠ {error}</p>
  {/if}

  {#if loading}
    <p style="color:#666">Fetching commits…</p>
  {/if}

  {#if tiles.length > 0}
    <!-- svelte-ignore a11y_no_noninteractive_element_interactions -->
    <canvas
      id="gameMap"
      bind:this={canvas}
      width={740}
      height={260}
      tabindex="0"
      role="grid"
      aria-label="Commit activity map. Use arrow keys to navigate tiles."
      onkeydown={handleKeydown}
      onclick={handleCanvasClick}
      onmousemove={handleCanvasMove}
      onmouseleave={() => { hoveredTile = null; drawMap(); }}
      style="border:1px solid #e0e0e0;border-radius:8px;outline:none;cursor:crosshair"
    ></canvas>
    <p style="color:#999;font-size:11px;margin-top:8px">
      Click tiles or use ←→↑↓ to navigate. Darker = more activity.
    </p>
  {:else if !loading && !error && username.trim()}
    <p style="color:#666">No events found for @{username}.</p>
  {/if}
</main>
