---
layout: default
title: Leaderboard
---

{% if jekyll.environment == "development" %}{% assign lb_url = site.data.leaderboard.dev_data_url | relative_url %}{% assign lb_practice_url = site.data.leaderboard.dev_practice_url | relative_url %}{% else %}{% assign lb_url = site.data.leaderboard.data_url %}{% assign lb_practice_url = site.data.leaderboard.practice_url %}{% endif %}
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
                    <th class="lb-num" data-col="return" title="Net return on the initial capital up to the latest equity snapshot">PnL</th>
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

    <section class="lb-formulas" id="lb-formulas" hidden>
        <h2>How the leaderboard is computed</h2>
        <ol id="lb-formulas-list"></ol>
    </section>
    <p class="lb-disclaimer">Standings are provisional until the organizers confirm the final results. The organizers may recompute the scores, re-apply the formula or change it where needed for a fair assessment of the competition, or to correct technical problems in the data or in the calculations.</p>
</div>

<script>
(function () {
    const URLS = { live: {{ lb_url | jsonify }}, practice: {{ lb_practice_url | jsonify }} };
    const REFRESH_MS = {{ site.data.leaderboard.refresh_seconds | default: 60 }} * 1000;
    // Practice returns are often a few millionths of the capital: 8 decimals, 9 for the score.
    const PCT_DIGITS = 8;
    const SCORE_DIGITS = 9;

    const boards = { live: null, practice: null };
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
            return;
        }
        if (!live) {
            empty.textContent = "The phase starts on " + when(d.from) + ": no figures before then.";
            renderFormulas(null);
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
            ["PnL", " (return R): net return on the initial capital up to the latest equity snapshot, R = (1 + r_1) × … × (1 + r_n) × (1 + r_today) − 1, " +
                "where r_d = (E_d − transfers) / E_(d−1) − 1 and E_d is the account value at the " + cutoff + " UTC sample of day d. " +
                "Money moved in or out is not a gain or a loss."],
            ["Volatility", " σ: volatility over the period, the same days as the PnL. σ = s × √n, where s is the " + s.std + " standard deviation (n − 1) " +
                "of the n daily returns of the completed days so far (from the second day) and n their number; after a missing day the return enters as r / √(days covered)."],
            ["Financial score", " Q = (max(R, 0) + ε) / (|R| + λ × σ + c), with R the PnL, σ the volatility, λ = " + s.lambda + ", c = " + s.c + ", ε = " + s.epsilon + ". " +
                "Each team is scored on its own figures only: the score does not change when other teams join or move. A loss scores below ε / c = " + num(s.epsilon / s.c) + "."],
            ["Score", " = " + s.w_financial + " × Q + " + s.w_jury + " × J, with J the jury score from 0 to 1: 0 until the jury votes, " +
                "so until then the score is at most " + s.w_financial + ". The position (#) follows the score."],
            ["In the score", ": at least " + days + " (a trade on the competition markets) in the phase, a start sample and at least 2 daily returns. " +
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
})();
</script>
