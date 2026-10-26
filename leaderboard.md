---
layout: default
title: Leaderboard
---

{% if jekyll.environment == "development" %}{% assign lb_url = site.data.leaderboard.dev_data_url | relative_url %}{% assign lb_practice_url = site.data.leaderboard.dev_practice_url | relative_url %}{% assign lb_chart_url = site.data.leaderboard.dev_chart_url | relative_url %}{% assign lb_practice_chart_url = site.data.leaderboard.dev_practice_chart_url | relative_url %}{% else %}{% assign lb_url = site.data.leaderboard.data_url %}{% assign lb_practice_url = site.data.leaderboard.practice_url %}{% assign lb_chart_url = site.data.leaderboard.chart_url %}{% assign lb_practice_chart_url = site.data.leaderboard.practice_chart_url %}{% endif %}
<div class="page-content lb">
    <div class="lb-head">
        <h1>Leaderboard</h1>
        <p class="lb-status" id="lb-status" aria-live="polite"><span class="lb-dot"></span><span id="lb-status-text">Loading…</span></p>
    </div>

    <nav class="lb-tabs" id="lb-tabs" aria-label="Standings" hidden>
        <button type="button" data-tab="live" id="lb-tab-live">Current phase</button>
        <button type="button" data-tab="practice">Practice (final)</button>
    </nav>

    <p class="lb-sort" id="lb-sort" hidden>Ranked by
        <button type="button" data-sort="score">Score</button>
        <button type="button" data-sort="return">PnL</button>
        <button type="button" data-sort="volatility">Volatility</button>
    </p>

    <div class="glass-card lb-card">
        <table class="lb-table" id="lb-table">
            <thead>
                <tr>
                    <th class="lb-rank">#</th>
                    <th class="lb-team">Team</th>
                    <th class="lb-num" data-col="return" title="Time-weighted net return up to the latest daily sample (12:00 UTC)">PnL</th>
                    <th class="lb-num" data-col="volatility" title="Volatility over the period: daily deviation × √n, the one in the score">Volatility</th>
                    <th class="lb-num" data-col="score" title="0.8 × Q + 0.2 × J (J = 0 until the jury votes)">Score</th>
                </tr>
            </thead>
            <tbody id="lb-body"></tbody>
        </table>
        <!-- Phones: one card per team, five columns of 8-9 decimals do not fit. -->
        <ol class="lb-cards" id="lb-cards"></ol>
        <p class="lb-empty" id="lb-empty" hidden></p>
    </div>

    <section class="lb-chart" id="lb-chart" hidden>
        <h2>Top 10 by PnL</h2>
        <div id="lb-chart-box"></div>
        <p class="lb-disclaimer">This chart shows, among the teams that have already traded, the 10 with the highest PnL at their latest equity snapshot, one point every 10 minutes; it is refreshed every 5 minutes, so the teams in it can change at every update. It is not the ranking: the standings and the score use only the daily sample at 12:00 UTC, so the order here can differ from the standings. Between two daily samples the lines are for information only.</p>
    </section>

    <section class="lb-formulas" id="lb-formulas" hidden>
        <h2>How the leaderboard is computed</h2>
        <ol id="lb-formulas-list"></ol>
    </section>
    <p class="lb-disclaimer">Standings are provisional until the organizers confirm the final results. The organizers may recompute the scores, re-apply the formula or change it where needed for a fair assessment of the competition, or to correct technical problems in the data or in the calculations.</p>
</div>

<style>
.lb-term-bar { display: flex; flex-wrap: wrap; align-items: center; gap: 0.5rem 1rem; padding: 0.5rem 0.75rem; border-bottom: 1px solid #262b33; font-size: 11px; color: #9ca3af; letter-spacing: 0.04em; }
.lb-term-presets { display: inline-flex; border: 1px solid #262b33; border-radius: 4px; overflow: hidden; }
.lb-term-presets button, .lb-term-reset { min-width: 2.75rem; padding: 0.35rem 0.6rem; background: transparent; color: #9ca3af; font: inherit; font-weight: 600; letter-spacing: 0.06em; cursor: pointer; }
.lb-term-presets button + button { border-left: 1px solid #262b33; }
.lb-term-presets button[aria-pressed="true"] { background: #ff9f1a; color: #000; }
.lb-term-presets button:hover:not([aria-pressed="true"]), .lb-term-reset:hover:not(:disabled) { color: #fff; }
.lb-term-range { display: flex; flex-wrap: wrap; gap: 0.5rem; }
.lb-term-range label { display: inline-flex; align-items: center; gap: 0.4rem; text-transform: uppercase; }
.lb-term-range input { padding: 0.25rem 0.4rem; border: 1px solid #262b33; border-radius: 4px; background: #0f1319; color: #e5e7eb; font: inherit; color-scheme: dark; }
.lb-term-opts { display: inline-flex; align-items: center; gap: 0.75rem; margin-left: auto; }
.lb-term-reset { border: 1px solid #262b33; border-radius: 4px; }
.lb-term-reset:disabled { opacity: 0.4; cursor: default; }
.lb-term-presets button:focus-visible, .lb-term-reset:focus-visible, .lb-term-range input:focus-visible { outline: 1px solid #ff9f1a; outline-offset: 1px; }
.lb-term-hint { margin: 0; padding: 0.25rem 0.75rem 0.4rem; color: #6b7280; font-size: 11px; }
/* Phones: the toolbar stacks, buttons and fields big enough for a finger. */
@media (max-width: 600px) {
    .lb-term-bar { flex-direction: column; align-items: stretch; font-size: 12px; }
    /* The preset buttons stay on one row. */
    .lb-term-presets { display: grid; grid-auto-flow: column; grid-auto-columns: 1fr; }
    .lb-term-presets button, .lb-term-reset { min-height: 44px; font-size: 13px; }
    .lb-term-range { display: grid; grid-template-columns: 1fr 1fr; }
    .lb-term-bar > *, .lb-term-range label { min-width: 0; }
    .lb-term-range label { flex-direction: column; align-items: stretch; gap: 0.2rem; }
    /* datetime-local has a wide intrinsic size: without these it pushed the chart past the screen. */
    .lb-term-range input { min-height: 44px; width: 100%; min-width: 0; max-width: 100%; box-sizing: border-box; font-size: 14px; -webkit-appearance: none; appearance: none; }
    .lb-term-opts { margin-left: 0; justify-content: space-between; }
    .lb-term-reset { padding: 0 1.25rem; }
    .lb-term-zoom { display: inline-grid; grid-template-columns: 1fr 1fr; }
    .lb-term-zoom button { min-width: 3rem; font-size: 18px; }
}
.lb .lb-term .lb-term-hint { margin: 0; padding: 0.25rem 0.75rem 0.4rem; color: #6b7280; font-size: 11px; line-height: 1.4; }
.lb-term-zoom button { min-width: 2.25rem; font-size: 14px; }
.lb-term-zoom button:disabled { opacity: 0.35; cursor: default; }
</style>

<script>
(function () {
    const URLS = { live: {{ lb_url | jsonify }}, practice: {{ lb_practice_url | jsonify }} };
    const CHART_URLS = { live: {{ lb_chart_url | jsonify }}, practice: {{ lb_practice_chart_url | jsonify }} };
    const REFRESH_MS = {{ site.data.leaderboard.refresh_seconds | default: 60 }} * 1000;
    // Practice returns are often a few millionths of the capital: 8 decimals, 9 for the score.
    const PCT_DIGITS = 8;
    const SCORE_DIGITS = 9;

    const boards = { live: null, practice: null };
    const charts = { live: null, practice: null };
    let tab = location.hash === "#practice" ? "practice" : "live";
    let sort = "score";
    let failed = false;

    const $ = (id) => document.getElementById(id);
    const el = (tag, cls, text) => {
        const n = document.createElement(tag);
        if (cls) n.className = cls;
        if (text != null) n.textContent = text;
        return n;
    };

    // --- ranking: same rules as the organizer panel (pa_frontend lib/leaderboard.ts) ---

    // Volatility over the period (daily deviation × √n); an older file has the daily one only.
    const shownVol = (t) => (t.volatility_horizon !== undefined ? t.volatility_horizon : t.volatility);
    const byName = (a, b) => a.name.toLowerCase().localeCompare(b.name.toLowerCase());

    // By score: eligible teams by rank, then the others (teams that traded first, then by name), all with
    // their figures. By PnL (highest first) or volatility (lowest first): a team not ranked shows its figures
    // only when final (while live it has never traded).
    function rankTeams(d, by) {
        const line = (t, rank, shown) => ({ t, rank, shown, inScore: t.eligible != null ? t.eligible : rank !== null });
        if (by === "score") {
            const ranked = (t) => typeof t.rank === "number";
            const top = d.teams.filter(ranked).sort((a, b) => a.rank - b.rank || byName(a, b));
            const rest = d.teams.filter((t) => !ranked(t)).sort((a, b) => (b.ranked - a.ranked) || byName(a, b));
            return [...top.map((t) => line(t, t.rank, true)), ...rest.map((t) => line(t, null, true))];
        }
        const value = (t) => (by === "return" ? t.return : shownVol(t));
        const dir = by === "return" ? -1 : 1;
        const top = d.teams.filter((t) => t.ranked && value(t) != null).sort((a, b) => dir * (value(a) - value(b)) || byName(a, b));
        const rest = d.teams.filter((t) => !top.includes(t)).sort((a, b) => (b.ranked - a.ranked) || byName(a, b));
        const shown = (t) => t.ranked || d.status === "final";
        return [...top.map((t, i) => line(t, i + 1, shown(t))), ...rest.map((t) => line(t, null, shown(t)))];
    }

    const pct = (v) => (v == null || !Number.isFinite(v) ? "–" : (v >= 0 ? "+" : "") + (v * 100).toFixed(PCT_DIGITS) + "%");
    const volPct = (v) => (v == null || !Number.isFinite(v) ? "–" : (v * 100).toFixed(PCT_DIGITS) + "%");
    const score = (v) => (v == null || !Number.isFinite(v) ? "–" : v.toFixed(SCORE_DIGITS));
    // 12 significant digits: ε / c is 0.1, not 0.09999999999999999.
    const num = (x) => String(Number(x.toPrecision(12)));
    const when = (iso) => new Date(iso).toLocaleString("en-GB", { day: "numeric", month: "short", hour: "2-digit", minute: "2-digit", timeZone: "UTC" }) + " UTC";

    function figures(l) {
        const ret = l.shown ? l.t.return : null;
        const vol = l.shown ? shownVol(l.t) : null;
        return {
            ret: pct(ret), vol: volPct(vol), score: score(l.t.score),
            color: ret == null || !Number.isFinite(ret) ? "" : ret >= 0 ? "up" : "down",
        };
    }

    // --- rendering ---

    function render() {
        const d = boards[tab];
        // The practice tab exists once its file is written.
        $("lb-tabs").hidden = !boards.practice;
        $("lb-tab-live").textContent = boards.live && boards.live.phase ? boards.live.phase : "Current phase";
        document.querySelectorAll("#lb-tabs button").forEach((b) => b.setAttribute("aria-current", b.dataset.tab === tab ? "page" : "false"));

        // A file without the score (older tracker) is ordered by PnL.
        const by = sort === "score" && d && !d.scoring ? "return" : sort;
        document.querySelectorAll("#lb-sort button").forEach((b) => {
            b.hidden = b.dataset.sort === "score" && !(d && d.scoring);
            b.setAttribute("aria-pressed", b.dataset.sort === by ? "true" : "false");
        });
        document.querySelectorAll("#lb-table th[data-col]").forEach((th) => th.classList.toggle("on", th.dataset.col === by));

        const body = $("lb-body");
        const cards = $("lb-cards");
        body.replaceChildren();
        cards.replaceChildren();
        const empty = $("lb-empty");
        const live = d && d.status !== "upcoming";
        $("lb-sort").hidden = !live;
        $("lb-table").hidden = !live;
        empty.hidden = !!live;
        if (!d) {
            empty.textContent = failed ? "The leaderboard is not available yet. Check back when the competition starts." : "Loading…";
            renderFormulas(null);
            renderChart();
            return;
        }
        if (!live) {
            empty.textContent = "The phase starts on " + when(d.from) + ": no figures before then.";
            renderFormulas(null);
            renderChart();
            return;
        }
        for (const l of rankTeams(d, by)) {
            const f = figures(l);
            const tr = el("tr", l.inScore ? "" : "lb-unranked");
            tr.appendChild(el("td", "lb-rank", l.rank == null ? "–" : String(l.rank)));
            tr.appendChild(el("td", "lb-team", l.t.name));
            tr.appendChild(el("td", "lb-num " + f.color + (by === "return" ? " on" : ""), f.ret));
            tr.appendChild(el("td", "lb-num" + (by === "volatility" ? " on" : ""), f.vol));
            tr.appendChild(el("td", "lb-num lb-score" + (by === "score" ? " on" : ""), f.score));
            body.appendChild(tr);

            const li = el("li", "lb-item" + (l.inScore ? "" : " lb-unranked"));
            const top = el("div", "lb-item-head");
            top.appendChild(el("span", "lb-rank", l.rank == null ? "–" : String(l.rank)));
            top.appendChild(el("span", "lb-team", l.t.name));
            li.appendChild(top);
            const dl = el("dl", "lb-item-figs");
            for (const [label, value, cls] of [["PnL", f.ret, f.color], ["Volatility", f.vol, ""], ["Score", f.score, "lb-score"]]) {
                const row = el("div");
                row.appendChild(el("dt", "", label));
                row.appendChild(el("dd", cls, value));
                dl.appendChild(row);
            }
            li.appendChild(dl);
            cards.appendChild(li);
        }
        renderFormulas(d);
        renderChart();
    }

    // The formulas behind the table, with the parameters of the file (fractional score only).
    function renderFormulas(d) {
        const box = $("lb-formulas");
        const s = d && d.scoring;
        box.hidden = !(s && s.method === "fractional");
        if (box.hidden) return;
        const cutoff = d.from.slice(11, 16);
        const days = d.min_days + " trading day" + (d.min_days === 1 ? "" : "s");
        const items = [
            ["PnL", " (return R): time-weighted net return up to the latest daily sample, R = (1 + r_1) × … × (1 + r_n) − 1, " +
                "where r_d = (E_d − transfers) / E_(d−1) − 1 and E_d is the Hyperliquid account value (open positions at the mark price, unrealized PnL included) " +
                "at the daily sample of day d: " + sampleRule(cutoff, s.sample_after_s) + ". " +
                "Money moved in or out is not a gain or a loss. PnL, volatility and score change once a day, with the new sample."],
            ["Volatility", " σ: volatility over the period, the same days as the PnL. σ = s × √n, where s is the " + s.std + " standard deviation (n − 1) " +
                "of the n daily returns of the completed days so far (from the second day) and n their number; after a missing day the return enters as r / √(days covered)."],
            ["Financial score", " Q = (max(R, 0) + ε) / (|R| + λ × σ + c), with R the PnL, σ the volatility, λ = " + s.lambda + ", c = " + s.c + ", ε = " + s.epsilon + ". " +
                "Each team is scored on its own figures only: the score does not change when other teams join or move. A loss scores below ε / c = " + num(s.epsilon / s.c) + "."],
            ["Score", " = " + s.w_financial + " × Q + " + s.w_jury + " × J, with J the jury score from 0 to 1: 0 until the jury votes, " +
                "so until then the score is at most " + s.w_financial + ". The position (#) follows the score."],
            ["In the score", ": at least " + days + " (a trade on the competition markets) in the phase, a start sample and at least 2 daily returns. " +
                "The first daily return comes with the second daily sample, so in the first two days of a phase nobody has a rank yet. " +
                "The other teams are listed below, dimmed, with their figures."],
        ];
        const list = $("lb-formulas-list");
        list.replaceChildren();
        for (const [head, text] of items) {
            const li = el("li");
            li.appendChild(el("strong", "", head));
            li.appendChild(document.createTextNode(text));
            list.appendChild(li);
        }
    }

    function status() {
        const s = $("lb-status");
        const txt = $("lb-status-text");
        s.className = "lb-status";
        const d = boards[tab];
        if (!d) {
            if (failed) txt.textContent = "Not available yet";
            return;
        }
        const phase = d.phase ? d.phase + " · " : "";
        if (d.status === "upcoming") {
            txt.textContent = phase + "Starts " + when(d.from);
        } else if (d.status === "final") {
            s.classList.add("final");
            txt.textContent = phase + "Final standings · " + when(d.from) + " to " + when(d.to);
        } else {
            s.classList.add("live");
            const mins = Math.max(0, Math.round((Date.now() - new Date(d.generated_at)) / 60000));
            txt.textContent = (phase || "Live · ") + "updated " + (mins < 1 ? "just now" : mins + " min ago");
        }
    }

    // How the daily sample is taken (same words as the panel, pa_frontend lib/leaderboard.ts sampleRule).
    function sampleRule(cutoff, after) {
        if (after > 0) {
            const m = Math.round(after / 60);
            return "the first equity snapshot of the round that starts at " + cutoff + " UTC (taken within " + m + " minute" + (m === 1 ? "" : "s") +
                " of it; for a team the round missed, the last snapshot before " + cutoff + ")";
        }
        return "the last equity snapshot before " + cutoff + " UTC";
    }

    // --- PnL chart: same look and rules as the panel (pa_frontend components/tradingChart.tsx) ---

    const LINES = ["#ff9f1a", "#22d3ee", "#4ade80", "#f472b6", "#f5f5f5", "#3b82f6", "#f87171", "#a78bfa", "#bef264", "#9ca3af"];
    const MONTHS = ["JAN", "FEB", "MAR", "APR", "MAY", "JUN", "JUL", "AUG", "SEP", "OCT", "NOV", "DEC"];
    const SVG = "http://www.w3.org/2000/svg";
    const HOUR = 3600000, DAY = 86400000;
    // Time windows of the buttons (0: the whole phase); the first view is "1W".
    const PRESETS = [["1D", DAY], ["3D", 3 * DAY], ["1W", 7 * DAY], ["All", 0]];
    const DEFAULT_PRESET = "1W";
    // Narrowest window: the chart file has a point every 10 minutes.
    const MIN_SPAN = 2 * HOUR;
    const two = (n) => String(n).padStart(2, "0");
    const dayLabel = (t) => { const d = new Date(t); return two(d.getUTCDate()) + " " + MONTHS[d.getUTCMonth()]; };
    const hhmm = (t) => { const d = new Date(t); return two(d.getUTCHours()) + ":" + two(d.getUTCMinutes()); };
    const stamp = (t) => dayLabel(t) + " " + hhmm(t) + " UTC";
    // datetime-local inputs hold UTC here: no time zone of the viewer's device.
    const toInput = (t) => new Date(t).toISOString().slice(0, 16);
    const fromInput = (s) => (s ? Date.parse(s + ":00Z") : NaN);
    // 5 decimals: the chart file keeps 1e-7 of the capital, 0.00001%.
    const pct5 = (v) => (v >= 0 ? "+" : "") + (v * 100).toFixed(5) + "%";
    // preset: the button of the window shown, which follows the latest data; null: a window
    // zoomed or typed by hand, kept in view. gesture: a drag, pan or pinch in progress.
    const chartState = { hidden: new Set(), focus: null, cursor: null, frame: null, preset: DEFAULT_PRESET, view: null, tab: null, gesture: null };

    function svgEl(tag, attrs, text) {
        const n = document.createElementNS(SVG, tag);
        for (const k in attrs) n.setAttribute(k, attrs[k]);
        if (text != null) n.textContent = text;
        return n;
    }

    // A stable colour per team name: the hash picks a colour, a taken one moves to the next free.
    function colours(keys) {
        const out = new Map();
        const taken = new Set();
        for (const k of keys) {
            let h = 0;
            for (let i = 0; i < k.length; i++) h = (h * 31 + k.charCodeAt(i)) >>> 0;
            let c = h % LINES.length;
            for (let n = 0; n < LINES.length && taken.has(c); n++) c = (c + 1) % LINES.length;
            taken.add(c);
            out.set(k, LINES[c]);
        }
        return out;
    }

    // The lines of a chart file: one point per step (null: no snapshot), then the last snapshot.
    function chartSeries(f) {
        const t0 = Date.parse(f.start);
        const step = f.step_s * 1000;
        const seen = new Map();
        for (const t of f.teams) seen.set(t.name, (seen.get(t.name) || 0) + 1);
        return f.teams.slice().sort((a, b) => a.position - b.position).slice(0, 10).map((t) => {
            const points = [];
            t.values.forEach((v, i) => { if (v !== null) points.push({ t: t0 + i * step, v }); });
            const last = Date.parse(t.last_at);
            if (!points.length || last > points[points.length - 1].t) points.push({ t: last, v: t.pnl });
            return { key: seen.get(t.name) > 1 ? t.name + " #" + t.position : t.name, name: t.name, position: t.position, points };
        });
    }

    // Index of the last point at or before t, -1 if none.
    function indexAt(points, t) {
        let lo = 0, hi = points.length - 1, found = -1;
        while (lo <= hi) {
            const mid = (lo + hi) >> 1;
            if (points[mid].t <= t) { found = mid; lo = mid + 1; } else hi = mid - 1;
        }
        return found;
    }

    function valueAt(points, t) {
        const i = indexAt(points, t);
        return i < 0 ? null : points[i].v;
    }

    // Time axis: grid step for the window, from hours to days.
    function timeStep(span, compact) {
        const most = compact ? 4 : 9;
        for (const s of [HOUR, 2 * HOUR, 3 * HOUR, 6 * HOUR, 12 * HOUR, DAY, 2 * DAY, 4 * DAY, 7 * DAY]) if (span / s <= most) return s;
        return 7 * DAY;
    }

    function renderChart() {
        const box = $("lb-chart-box");
        const section = $("lb-chart");
        // A refresh while a date is being typed or a gesture is running would throw it away: the next one redraws.
        if (chartState.gesture || (box.contains(document.activeElement) && document.activeElement.tagName === "INPUT")) return;
        const d = boards[tab];
        const f = charts[tab];
        // Only a chart of the same window and phase as the standings shown.
        const ok = d && f && d.status !== "upcoming" && f.teams && f.teams.length && Date.parse(f.start) === Date.parse(d.from) && f.phase === d.phase;
        section.hidden = !ok;
        if (!ok) return;
        if (chartState.tab !== tab) Object.assign(chartState, { tab, preset: DEFAULT_PRESET, view: null, cursor: null });
        const series = chartSeries(f);
        const colour = colours(series.map((s) => s.key));
        const compact = box.clientWidth < 600;
        const width = compact ? 520 : 960, height = compact ? 340 : 380, font = compact ? 13 : 10;
        // Value axis in its own gutter on the left: inside the plot its labels hid the
        // latest points, which are always on the right edge.
        const pad = { top: 16, right: compact ? 100 : 150, bottom: compact ? 34 : 30, left: compact ? 92 : 70 };
        const plotW = width - pad.left - pad.right, plotH = height - pad.top - pad.bottom;
        let full0 = Infinity, full1 = -Infinity;
        for (const s of series) for (const p of s.points) { if (p.t < full0) full0 = p.t; if (p.t > full1) full1 = p.t; }
        if (!(full1 > full0)) { section.hidden = true; return; }
        const minSpan = Math.min(MIN_SPAN, full1 - full0);
        // A window inside the phase, at least minSpan wide.
        const clampView = (a, b) => {
            let span = Math.min(Math.max(b - a, minSpan), full1 - full0);
            a = Math.min(Math.max(a, full0), full1 - span);
            return { t0: a, t1: a + span };
        };
        const currentView = () => {
            if (chartState.preset) {
                const span = PRESETS.find((p) => p[0] === chartState.preset)[1];
                return { t0: span ? Math.max(full0, full1 - span) : full0, t1: full1 };
            }
            return clampView(chartState.view.t0, chartState.view.t1);
        };
        const setView = (a, b) => { chartState.preset = null; chartState.view = clampView(a, b); draw(); };
        const setPreset = (name) => { chartState.preset = name; chartState.view = null; draw(); };

        const fig = el("figure", "lb-term");
        const cap = el("figcaption", "lb-term-head");
        const left = el("span", "lb-term-title");
        left.appendChild(el("span", "lb-term-badge", "PNL"));
        left.appendChild(el("span", "lb-term-name", "Top " + series.length + " by PnL"));
        left.appendChild(el("span", "lb-term-sub", Math.round(f.step_s / 60) + "m · last snapshot"));
        cap.appendChild(left);
        cap.appendChild(el("span", "lb-term-sub", "upd " + stamp(Date.parse(f.generated_at))));
        fig.appendChild(cap);

        // Toolbar: window buttons, From / To in UTC, reset, PnL from the start of the window.
        const bar = el("div", "lb-term-bar");
        const presets = el("div", "lb-term-presets");
        presets.setAttribute("role", "group");
        presets.setAttribute("aria-label", "Time window");
        const presetButtons = PRESETS.map(([name]) => {
            const b = el("button", "", name);
            b.type = "button";
            b.addEventListener("click", () => setPreset(name));
            presets.appendChild(b);
            return [name, b];
        });
        bar.appendChild(presets);
        const range = el("div", "lb-term-range");
        const mkInput = (label) => {
            const lab = el("label", "", label);
            const inp = el("input");
            inp.type = "datetime-local";
            inp.step = 600;
            inp.min = toInput(full0);
            inp.max = toInput(full1);
            lab.appendChild(inp);
            range.appendChild(lab);
            return inp;
        };
        const fromIn = mkInput("From (UTC)");
        const toIn = mkInput("To (UTC)");
        const onRange = () => {
            const a = fromInput(fromIn.value), b = fromInput(toIn.value);
            if (isFinite(a) && isFinite(b) && b > a) setView(a, b); else draw();
        };
        fromIn.addEventListener("change", onRange);
        toIn.addEventListener("change", onRange);
        bar.appendChild(range);
        const opts = el("div", "lb-term-opts");
        // Zoom in / out around the middle of the window, a factor 2 per click.
        const zoomBy = (k) => { const mid = (vt0 + vt1) / 2, half = ((vt1 - vt0) * k) / 2; setView(mid - half, mid + half); };
        const zoomGroup = el("div", "lb-term-presets lb-term-zoom");
        zoomGroup.setAttribute("role", "group");
        zoomGroup.setAttribute("aria-label", "Zoom");
        const zoomIn = el("button", "", "+");
        zoomIn.type = "button"; zoomIn.title = "Zoom in"; zoomIn.setAttribute("aria-label", "Zoom in");
        zoomIn.addEventListener("click", () => zoomBy(0.5));
        const zoomOut = el("button", "", "\u2212");
        zoomOut.type = "button"; zoomOut.title = "Zoom out"; zoomOut.setAttribute("aria-label", "Zoom out");
        zoomOut.addEventListener("click", () => zoomBy(2));
        zoomGroup.appendChild(zoomIn);
        zoomGroup.appendChild(zoomOut);
        opts.appendChild(zoomGroup);
        const reset = el("button", "lb-term-reset", "Reset");
        reset.type = "button";
        reset.title = "Back to the last week";
        reset.addEventListener("click", () => setPreset(DEFAULT_PRESET));
        opts.appendChild(reset);
        bar.appendChild(opts);
        fig.appendChild(bar);

        const svg = svgEl("svg", { viewBox: "0 0 " + width + " " + height, role: "img", "aria-label": "PnL of the top " + series.length + " teams", class: "lb-term-svg" });
        const clipId = "lb-clip-" + tab;
        const defs = svgEl("defs", {});
        const clip = svgEl("clipPath", { id: clipId });
        clip.appendChild(svgEl("rect", { x: pad.left, y: pad.top, width: plotW, height: plotH }));
        defs.appendChild(clip);
        svg.appendChild(defs);
        svg.appendChild(svgEl("rect", { x: 0, y: 0, width, height, fill: "#07090c" }));
        const plot = svgEl("g", {});
        const cross = svgEl("g", { "pointer-events": "none" });
        const brush = svgEl("rect", { y: pad.top, height: plotH, width: 0, fill: "#ff9f1a", "fill-opacity": 0.12, stroke: "#ff9f1a", "stroke-opacity": 0.6, visibility: "hidden", "pointer-events": "none" });
        svg.appendChild(plot);
        svg.appendChild(cross);
        svg.appendChild(brush);
        fig.appendChild(svg);
        const hint = el("p", "lb-term-hint", compact ? "Zoom: + / \u2212 or pinch. Drag sideways to move, tap for values." : "Zoom: mouse wheel on the chart, + / \u2212, or drag across it. Double-click to reset.");
        fig.appendChild(hint);

        // Scales of the last draw, read by the pointer handlers.
        let vt0 = full0, vt1 = full1, x = () => 0, rows = () => [];
        const timeAtPx = (px) => vt0 + ((px - pad.left) / plotW) * (vt1 - vt0);
        const svgX = (clientX) => { const r = svg.getBoundingClientRect(); return ((clientX - r.left) / r.width) * width; };

        function draw() {
            const v = currentView();
            vt0 = v.t0; vt1 = v.t1;
            const shown = series.filter((s) => !chartState.hidden.has(s.key) && s.points.length);
            const val = (s, raw) => raw;
            let lo = 0, hi = 0;
            for (const s of shown) for (const p of s.points) if (p.t >= vt0 && p.t <= vt1) { const w = val(s, p.v); if (w < lo) lo = w; if (w > hi) hi = w; }
            const margin = (hi - lo || Math.abs(hi) || 1e-6) * 0.06;
            lo -= margin; hi += margin;
            x = (t) => pad.left + ((t - vt0) / (vt1 - vt0)) * plotW;
            const y = (w) => pad.top + plotH - ((w - lo) / (hi - lo)) * plotH;
            plot.replaceChildren();

            for (let i = 0; i < 5; i++) {
                const w = lo + ((hi - lo) * (i + 0.5)) / 5;
                plot.appendChild(svgEl("line", { x1: pad.left, x2: pad.left + plotW, y1: y(w), y2: y(w), stroke: "#141920" }));
                plot.appendChild(svgEl("text", { x: pad.left - 6, y: y(w) + font / 3, "text-anchor": "end", fill: "#6b7280", "font-size": font }, pct5(w)));
            }
            // Dashed line at every daily sample (12:00 UTC), labels on a step that fits the window.
            const first = Date.parse(d.from);
            for (let t = first + Math.ceil((vt0 - first) / DAY) * DAY; t <= vt1; t += DAY)
                plot.appendChild(svgEl("line", { x1: x(t), x2: x(t), y1: pad.top, y2: pad.top + plotH, stroke: "#262b33", "stroke-dasharray": "2 3" }));
            const step = timeStep(vt1 - vt0, compact);
            const anchor = step >= DAY ? first : 0;
            for (let t = anchor + Math.ceil((vt0 - anchor) / step) * step; t <= vt1; t += step) {
                const label = step >= DAY ? dayLabel(t) : t % DAY === 0 ? dayLabel(t) : hhmm(t);
                const tx = x(t);
                plot.appendChild(svgEl("line", { x1: tx, x2: tx, y1: pad.top + plotH, y2: pad.top + plotH + 4, stroke: "#4b5563" }));
                plot.appendChild(svgEl("text", { x: tx, y: height - 10, "text-anchor": tx - pad.left < 30 ? "start" : pad.left + plotW - tx < 30 ? "end" : "middle", fill: "#6b7280", "font-size": font }, label));
            }
            plot.appendChild(svgEl("line", { x1: pad.left, x2: pad.left + plotW, y1: y(0), y2: y(0), stroke: "#4b5563" }));
            plot.appendChild(svgEl("line", { x1: pad.left + plotW, x2: pad.left + plotW, y1: pad.top, y2: pad.top + plotH, stroke: "#262b33" }));
            const lines = svgEl("g", { "clip-path": "url(#" + clipId + ")" });
            plot.appendChild(lines);
            // Last line first: the teams higher up are drawn on top of the flat ones.
            for (const s of shown.slice().reverse()) {
                // The spacing of the series, without its last interval: the last point can
                // be the latest snapshot seconds after the last step, and taking it as the
                // spacing broke every other interval into a gap.
                let gap = Infinity;
                const last = s.points.length > 2 ? s.points.length - 1 : s.points.length;
                for (let i = 1; i < last; i++) gap = Math.min(gap, s.points[i].t - s.points[i - 1].t);
                // The points in the window and one on each side, so the line reaches the edges.
                const i0 = Math.max(0, indexAt(s.points, vt0)), i1 = Math.min(s.points.length - 1, indexAt(s.points, vt1) + 1);
                let cur = [];
                const flush = () => {
                    if (!cur.length) return;
                    const dim = chartState.focus !== null && chartState.focus !== s.key;
                    lines.appendChild(svgEl("polyline", { points: cur.join(" "), fill: "none", stroke: colour.get(s.key), "stroke-opacity": dim ? 0.15 : 1,
                        "stroke-width": chartState.focus === s.key ? 2.2 : 1.3, "stroke-linejoin": "round" }));
                    cur = [];
                };
                for (let i = i0; i <= i1; i++) {
                    const p = s.points[i];
                    if (i > i0 && p.t - s.points[i - 1].t > 3 * gap) flush();
                    cur.push(x(p.t).toFixed(1) + "," + y(val(s, p.v)).toFixed(1));
                }
                flush();
            }
            // Value at the right edge of the window, as tags on the right axis, pushed apart.
            const tagH = font + 4;
            const tags = shown.map((s) => ({ s, raw: valueAt(s.points, vt1) })).filter((r) => r.raw !== null)
                .map((r) => { const w = val(r.s, r.raw); return { s: r.s, v: w, y0: y(w), y: y(w) }; }).sort((a, b) => a.y - b.y);
            // Pushed down from the top edge, then up from the bottom edge: every tag
            // stays inside the plot height (shifting them all up moved the highest out).
            const tagTop = pad.top + tagH / 2, tagBottom = pad.top + plotH - tagH / 2;
            for (let i = 0; i < tags.length; i++) tags[i].y = Math.max(tags[i].y, tagTop, i ? tags[i - 1].y + tagH : -Infinity);
            for (let i = tags.length - 1; i >= 0; i--) tags[i].y = Math.min(tags[i].y, tagBottom, i < tags.length - 1 ? tags[i + 1].y - tagH : Infinity);
            for (const tg of tags) {
                const g = svgEl("g", { opacity: chartState.focus !== null && chartState.focus !== tg.s.key ? 0.25 : 1 });
                g.appendChild(svgEl("line", { x1: pad.left + plotW, x2: pad.left + plotW + 4, y1: Math.min(Math.max(tg.y0, pad.top), pad.top + plotH), y2: tg.y, stroke: colour.get(tg.s.key) }));
                g.appendChild(svgEl("rect", { x: pad.left + plotW + 4, y: tg.y - tagH / 2, width: pad.right - 6, height: tagH - 1, fill: colour.get(tg.s.key) }));
                g.appendChild(svgEl("text", { x: pad.left + plotW + 8, y: tg.y + font / 3, fill: "#000", "font-size": font, "font-weight": 700 },
                    compact ? pct5(tg.v) : pct5(tg.v) + " " + tg.s.name.slice(0, 9).toUpperCase()));
                plot.appendChild(g);
            }
            rows = (c) => shown.map((s) => { const raw = valueAt(s.points, c); return { s, v: raw === null ? null : val(s, raw) }; })
                .filter((r) => r.v !== null).sort((a, b) => b.v - a.v);

            // Toolbar state.
            for (const [name, b] of presetButtons) b.setAttribute("aria-pressed", chartState.preset === name ? "true" : "false");
            fromIn.value = toInput(vt0);
            toIn.value = toInput(vt1);
            // The pickers offer only a To after the From (and a From before the To), at least the narrowest window apart.
            fromIn.max = toInput(vt1 - minSpan);
            toIn.min = toInput(vt0 + minSpan);
            reset.disabled = chartState.preset === DEFAULT_PRESET;
            zoomIn.disabled = vt1 - vt0 <= minSpan + 1;
            zoomOut.disabled = vt1 - vt0 >= full1 - full0 - 1;
            if (chartState.cursor !== null && (chartState.cursor < vt0 || chartState.cursor > vt1)) chartState.cursor = null;
            drawCross();
        }

        // Crosshair: every line at the time under the pointer (mouse, pen or touch).
        function drawCross() {
            cross.replaceChildren();
            const c = chartState.cursor;
            if (c === null) return;
            const list = rows(c);
            const cx = x(c), rowH = font + 4, boxW = compact ? 330 : 250;
            const bx = cx + 12 + boxW > pad.left + plotW ? Math.max(pad.left, cx - 12 - boxW) : cx + 12;
            cross.appendChild(svgEl("line", { x1: cx, x2: cx, y1: pad.top, y2: pad.top + plotH, stroke: "#ff9f1a", "stroke-dasharray": "3 3" }));
            cross.appendChild(svgEl("rect", { x: bx, y: pad.top + 6, width: boxW, height: rowH + 8 + list.length * rowH, fill: "#0f1319", stroke: "#262b33" }));
            cross.appendChild(svgEl("text", { x: bx + 8, y: pad.top + 6 + rowH, fill: "#ff9f1a", "font-size": font, "font-weight": 700 }, stamp(c)));
            list.forEach((r, i) => {
                cross.appendChild(svgEl("rect", { x: bx + 8, y: pad.top + 10 + rowH * (i + 1), width: 8, height: 8, fill: colour.get(r.s.key) }));
                cross.appendChild(svgEl("text", { x: bx + 22, y: pad.top + 6 + rowH * (i + 2), fill: "#d4d4d8", "font-size": font }, r.s.name.slice(0, compact ? 18 : 22)));
                cross.appendChild(svgEl("text", { x: bx + boxW - 8, y: pad.top + 6 + rowH * (i + 2), "text-anchor": "end", fill: r.v >= 0 ? "#22c55e" : "#ef4444", "font-size": font }, pct5(r.v)));
            });
        }
        const setCursor = (px) => {
            chartState.cursor = px < pad.left || px > pad.left + plotW ? null : timeAtPx(px);
            if (chartState.frame === null) chartState.frame = requestAnimationFrame(() => { chartState.frame = null; drawCross(); });
        };

        // Mouse and pen: drag across the plot to zoom on that stretch, double-click to reset.
        // Touch: one finger taps for values or pans a zoomed window sideways, two fingers pinch.
        // touch-action: pan-y (CSS) leaves the vertical page scroll to the browser.
        const touches = new Map();
        svg.addEventListener("pointerdown", (e) => {
            const px = svgX(e.clientX);
            if (e.pointerType !== "touch") {
                if (e.button !== 0 || px < pad.left || px > pad.left + plotW) return;
                chartState.gesture = { kind: "brush", x0: px, x1: px };
                svg.setPointerCapture(e.pointerId);
                return;
            }
            touches.set(e.pointerId, px);
            svg.setPointerCapture(e.pointerId);
            if (touches.size === 1) chartState.gesture = { kind: "touch", x0: px, view: { t0: vt0, t1: vt1 }, moved: false };
            else if (touches.size === 2) {
                const [a, b] = [...touches.values()];
                chartState.gesture = { kind: "pinch", dist: Math.max(Math.abs(a - b), 10), mid: timeAtPx((a + b) / 2), view: { t0: vt0, t1: vt1 } };
                chartState.cursor = null;
            }
        });
        svg.addEventListener("pointermove", (e) => {
            const px = svgX(e.clientX);
            const g = chartState.gesture;
            if (e.pointerType !== "touch") {
                if (g && g.kind === "brush") {
                    g.x1 = Math.min(Math.max(px, pad.left), pad.left + plotW);
                    brush.setAttribute("x", Math.min(g.x0, g.x1));
                    brush.setAttribute("width", Math.abs(g.x1 - g.x0));
                    brush.setAttribute("visibility", "visible");
                }
                setCursor(px);
                return;
            }
            if (!touches.has(e.pointerId)) return;
            touches.set(e.pointerId, px);
            if (g && g.kind === "pinch" && touches.size === 2) {
                const [a, b] = [...touches.values()];
                const span = (g.view.t1 - g.view.t0) * g.dist / Math.max(Math.abs(a - b), 10);
                const frac = ((a + b) / 2 - pad.left) / plotW;
                chartState.preset = null;
                chartState.view = clampView(g.mid - frac * span, g.mid - frac * span + span);
                draw();
            } else if (g && g.kind === "touch") {
                const dx = px - g.x0;
                if (!g.moved && Math.abs(dx) < 8) return;
                g.moved = true;
                // Pan only a zoomed window; on the whole phase a sideways drag moves the cursor.
                if (g.view.t1 - g.view.t0 < full1 - full0 - 1) {
                    const dt = (-dx / plotW) * (g.view.t1 - g.view.t0);
                    chartState.preset = null;
                    chartState.view = clampView(g.view.t0 + dt, g.view.t1 + dt);
                    chartState.cursor = null;
                    draw();
                } else setCursor(px);
            }
        });
        const endPointer = (e) => {
            const g = chartState.gesture;
            if (e.pointerType !== "touch") {
                if (g && g.kind === "brush") {
                    brush.setAttribute("visibility", "hidden");
                    chartState.gesture = null;
                    if (Math.abs(g.x1 - g.x0) > 6) {
                        const a = timeAtPx(Math.min(g.x0, g.x1)), b = timeAtPx(Math.max(g.x0, g.x1));
                        chartState.cursor = null;
                        setView(a, b);
                    }
                }
                return;
            }
            if (!touches.has(e.pointerId)) return;
            const px = touches.get(e.pointerId);
            touches.delete(e.pointerId);
            if (g && g.kind === "touch" && !g.moved && e.type === "pointerup") {
                // A tap shows the values at that time; a tap on the same spot hides them.
                const near = chartState.cursor !== null && Math.abs(x(chartState.cursor) - px) < 24;
                if (near) { chartState.cursor = null; drawCross(); } else setCursor(px);
            }
            if (touches.size === 1 && g && g.kind === "pinch") {
                const [rest] = [...touches.values()];
                chartState.gesture = { kind: "touch", x0: rest, view: { t0: vt0, t1: vt1 }, moved: true };
            } else if (touches.size === 0) chartState.gesture = null;
        };
        svg.addEventListener("pointerup", endPointer);
        svg.addEventListener("pointercancel", endPointer);
        svg.addEventListener("pointerleave", (e) => { if (e.pointerType === "touch" || chartState.gesture) return; chartState.cursor = null; drawCross(); });
        svg.addEventListener("dblclick", () => setPreset(DEFAULT_PRESET));
        // Wheel over the plot zooms at the pointer (the trackpad pinch arrives as a wheel too);
        // a sideways swipe pans. Outside the plot the wheel scrolls the page as usual.
        svg.addEventListener("wheel", (e) => {
            const px = svgX(e.clientX);
            if (px < pad.left || px > pad.left + plotW) return;
            e.preventDefault();
            if (Math.abs(e.deltaX) > Math.abs(e.deltaY)) {
                if (vt1 - vt0 >= full1 - full0 - 1) return;
                const dt = (e.deltaX / plotW) * (vt1 - vt0);
                setView(vt0 + dt, vt1 + dt);
                return;
            }
            const at = timeAtPx(px), k = Math.exp(Math.max(-1, Math.min(1, e.deltaY / 100)) * 0.25);
            setView(at - (at - vt0) * k, at + (vt1 - at) * k);
        }, { passive: false });

        // Legend: a button per team hides or shows its line.
        const table = el("table", "lb-term-legend");
        const head = el("tr");
        for (const [txt, cls] of [["#", ""], ["Team", ""], ["PnL", "lb-num"]]) head.appendChild(el("th", cls, txt));
        const thead = el("thead"); thead.appendChild(head); table.appendChild(thead);
        const tbody = el("tbody");
        for (const s of series) {
            const last = s.points[s.points.length - 1];
            const off = chartState.hidden.has(s.key);
            const tr = el("tr", off ? "off" : "");
            tr.appendChild(el("td", "lb-term-pos", String(s.position)));
            const td = el("td");
            const b = el("button");
            b.type = "button";
            b.setAttribute("aria-pressed", off ? "false" : "true");
            b.title = off ? "Show this line" : "Hide this line";
            const sw = el("span", "lb-term-swatch");
            sw.style.background = off ? "#3a3f47" : colour.get(s.key);
            b.appendChild(sw);
            b.appendChild(document.createTextNode(s.name));
            b.addEventListener("click", () => { if (chartState.hidden.has(s.key)) chartState.hidden.delete(s.key); else chartState.hidden.add(s.key); renderChart(); });
            td.appendChild(b);
            tr.appendChild(td);
            tr.appendChild(el("td", "lb-num " + (last && last.v < 0 ? "down" : "up"), last ? pct5(last.v) : "–"));
            tbody.appendChild(tr);
        }
        table.appendChild(tbody);
        fig.appendChild(table);
        box.replaceChildren(fig);
        draw();
    }

    // --- data ---

    async function fetchBoard(url) {
        if (!url) throw new Error("data_url is not set in _data/leaderboard.yml");
        const res = await fetch(url + (url.includes("?") ? "&" : "?") + "t=" + Date.now(), { cache: "no-store" });
        if (!res.ok) throw new Error("HTTP " + res.status);
        const d = await res.json();
        if (!d || !Array.isArray(d.teams)) throw new Error("not a leaderboard");
        return d;
    }

    async function load() {
        // The practice file never changes once written: fetched until it exists (503 before the practice closes).
        const jobs = [fetchBoard(URLS.live).then((d) => { boards.live = d; failed = false; })];
        if (!boards.practice && URLS.practice) jobs.push(fetchBoard(URLS.practice).then((d) => { boards.practice = d; }));
        // The PnL chart of the phase, rewritten every 5 minutes; the practice one is frozen. A missing chart only hides the chart.
        if (CHART_URLS.live) jobs.push(fetchBoard(CHART_URLS.live).then((c) => { charts.live = c; }));
        if (!charts.practice && CHART_URLS.practice) jobs.push(fetchBoard(CHART_URLS.practice).then((c) => { charts.practice = c; }));
        const out = await Promise.allSettled(jobs);
        if (out[0].status === "rejected") {
            console.error("leaderboard:", out[0].reason);
            if (!boards.live) failed = true;
        }
        if (tab === "practice" && !boards.practice) tab = "live";
        render();
        status();
    }

    document.querySelectorAll("#lb-sort button").forEach((b) => b.addEventListener("click", () => { sort = b.dataset.sort; render(); }));
    document.querySelectorAll("#lb-tabs button").forEach((b) => b.addEventListener("click", () => {
        tab = b.dataset.tab;
        history.replaceState(null, "", tab === "practice" ? "#practice" : location.pathname + location.search);
        render();
        status();
    }));

    render();
    load();
    setInterval(() => { if (!document.hidden) load(); }, REFRESH_MS);
    setInterval(status, 30000);
    document.addEventListener("visibilitychange", () => { if (!document.hidden) load(); });
    // The chart picks its drawing (wide or narrow) from its width.
    let resizeTimer = null;
    window.addEventListener("resize", () => { clearTimeout(resizeTimer); resizeTimer = setTimeout(renderChart, 150); });
})();
</script>
