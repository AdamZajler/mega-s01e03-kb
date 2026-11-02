<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { groupByCause, readAs, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  // One row per problem in a merged ticket, counted the desk's way (causes.ts): by cause, one id
  // one report. Reports the desk read as a one-day move stay counted under their cause and get a
  // row of their own below it, because they are a problem of their own.
  type Problem = 'parser' | 'hint' | 'unplaced' | 'one-day move';
  let { ticket, issues = {} }: { ticket: number; issues?: Partial<Record<Problem, string>> } = $props();

  const merged = $derived(uniqueById(reports as Report[]).filter((r) => r.mergedInto === ticket));
  const groups = $derived(groupByCause(merged, tenantOf));
  const rows = $derived(
    groups.flatMap((g) => {
      const moved = g.reports.filter((r) => notes.oneDayMove.includes(r.id));
      const days = moved.map((r) => r.receivedAt.slice(0, 10)).sort();
      const row = { problem: g.cause as Problem, n: g.reports.length, tenants: g.tenants, first: g.firstSeen, last: g.lastSeen, within: '' };
      if (!readAs(g, notes.oneDayMove)) return [row];
      return [
        row,
        { problem: 'one-day move' as Problem, n: moved.length, tenants: new Set(moved.map((r) => r.tenant)).size, first: days[0], last: days.at(-1)!, within: g.cause },
      ];
    }),
  );
</script>

<p class="sum"><strong>{merged.length}</strong> reports merged into #{ticket} · <strong>{groups.length}</strong> {groups.length === 1 ? 'problem' : 'problems'}</p>

<table>
  <thead><tr><th>problem</th><th>reports</th><th>tenants</th><th>seen</th><th>ticket</th></tr></thead>
  <tbody>
    {#each rows as r (r.problem)}
      <tr class:within={r.within}>
        <td><code>{r.problem}</code>{#if r.within} <span class="note">counted under <code>{r.within}</code></span>{/if}</td>
        <td>{r.n}</td>
        <td>{r.tenants}</td>
        <td>{r.first} → {r.last}</td>
        <td>{#if issues[r.problem]}<a href={issues[r.problem]}>#{issues[r.problem]?.split('/').at(-1)}</a>{:else}none yet{/if}</td>
      </tr>
    {/each}
  </tbody>
</table>

<style>
  .sum { font-size: 1.1rem; }
  .note { color: var(--muted); font-size: 0.85em; }
  .within td:first-child { padding-left: 1.5rem; }
</style>
