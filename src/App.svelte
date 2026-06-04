<script lang="ts">
  import { onMount } from "svelte";

  let username = $state("");
  let tiles: { x: number; y: number; color: string; count: number }[] = $state([]);
  let loading = $state(false);
  let error = $state("");

  const COLORS = ["#2d2d44", "#3b3b5c", "#4a4a74", "#5a5a8c", "#6b6ba4", "#7c7cbc", "#8d8dd4"];

  async function fetchCommits() {
    if (!username.trim()) return;
    loading = true; error = "";
    try {
      const res = await fetch(`https://api.github.com/users/${username}/events/public`);
      const events = await res.json();
      if (!Array.isArray(events)) { error = "User not found"; loading = false; return; }

      const byDate: Record<string, number> = {};
      for (const e of events) {
        const date = e.created_at?.slice(0, 10);
        if (date) byDate[date] = (byDate[date] || 0) + 1;
      }

      const result = Object.entries(byDate).map(([date, count], i) => ({
        x: i % 52,
        y: Math.floor(i / 52),
        color: COLORS[Math.min(count, COLORS.length - 1)],
        count,
      }));
      tiles = result;
    } catch {
      error = "Failed to fetch. Try again.";
    }
    loading = false;
  }

  onMount(() => {
    const params = new URLSearchParams(window.location.search);
    const u = params.get("u");
    if (u) { username = u; fetchCommits(); }
  });
</script>

<main style="max-width:960px;margin:0 auto;padding:40px;font-family:monospace">
  <h1 style="font-size:24px;font-weight:800">Commit.fun</h1>
  <p style="color:#666;margin-bottom:24px">Your GitHub activity, rendered as an 8-bit game map.</p>

  <div style="display:flex;gap:10px;margin-bottom:24px">
    <input bind:value={username} onkeydown={(e) => e.key === "Enter" && fetchCommits()}
      placeholder="GitHub username" style="flex:1;padding:10px 16px;border:2px solid #e0e0e0;border-radius:8px;font-size:14px;font-family:monospace" />
    <button onclick={fetchCommits} disabled={loading}
      style="padding:10px 24px;background:#2563eb;color:#fff;border:none;border-radius:8px;font-weight:600;font-family:monospace">
      {loading ? "..." : "Render"}
    </button>
  </div>

  {#if error}
    <p style="color:#ef4444">{error}</p>
  {/if}

  {#if tiles.length > 0}
    <canvas id="gameMap" width={960} height={520} style="border:1px solid #e0e0e0;border-radius:8px"></canvas>
    {@const canvas = document.getElementById("gameMap") as HTMLCanvasElement | null}
    {@const ctx = canvas?.getContext("2d")}
    {#if ctx}
      {@const _ = (() => {
        ctx.clearRect(0, 0, 960, 520);
        for (const t of tiles) {
          const size = 14;
          const padding = 2;
          ctx.fillStyle = t.color;
          ctx.fillRect(t.x * (size + padding) + 8, t.y * (size + padding) + 8, size, size);
        }
        ctx.fillStyle = "#666";
        ctx.font = "10px monospace";
        ctx.fillText(`@${username}`, 8, 500);
      })()}
    {/if}
  {/if}
</main>
