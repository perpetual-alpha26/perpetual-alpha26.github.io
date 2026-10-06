---
layout: default
title: Leaderboard
---

{% if jekyll.environment == "development" %}{% assign lb_url = site.data.leaderboard.dev_data_url | relative_url %}{% else %}{% assign lb_url = site.data.leaderboard.data_url %}{% endif %}
<div class="page-content lb">
    <div class="lb-head">
        <h1>Leaderboard</h1>
        <p class="lb-status" id="lb-status" aria-live="polite"><span class="lb-dot"></span><span id="lb-status-text">Loading…</span></p>
    </div>

    <div class="glass-card lb-card">
        <table class="lb-table">
            <thead>
                <tr>
                    <th class="lb-rank">#</th>
                    <th class="lb-team">Team</th>
                    <th class="lb-num" data-sort="return"><button type="button">PnL</button></th>
                    <th class="lb-num" data-sort="volatility"><button type="button">Volatility</button></th>
                </tr>
            </thead>
            <tbody id="lb-body"></tbody>
        </table>
        <p class="lb-empty" id="lb-empty" hidden></p>
    </div>

    <p class="lb-note" id="lb-note"></p>
</div>

<script>
(function () {
    const DATA_URL = {{ lb_url | jsonify }};
    const REFRESH_MS = {{ site.data.leaderboard.refresh_seconds | default: 60 }} * 1000;

    // Sort keys: PnL best first is highest, volatility best first is lowest.
    const SORTS = {
        return: { get: (t) => t.return, dir: -1, fmt: (x) => (x > 0 ? "+" : x < 0 ? "−" : "") + Math.abs(x * 100).toFixed(2) + "%",
                  empty: "No team has traded yet." },
        volatility: { get: (t) => t.volatility, dir: 1, fmt: (x) => (x * 100).toFixed(2) + "%",
                      empty: "Volatility appears after two full days of trading." },
    };
    let data = null;
    let sortKey = "return";

    const $ = (id) => document.getElementById(id);
    const el = (tag, cls, text) => {
        const n = document.createElement(tag);
        if (cls) n.className = cls;
        if (text != null) n.textContent = text;
        return n;
    };

    document.querySelectorAll("th[data-sort] button").forEach((b) =>
        b.addEventListener("click", () => { sortKey = b.parentElement.dataset.sort; render(); }));

    function render() {
        document.querySelectorAll("th[data-sort]").forEach((th) => {
            const on = th.dataset.sort === sortKey;
            th.classList.toggle("on", on);
            if (on) th.setAttribute("aria-sort", SORTS[sortKey].dir < 0 ? "descending" : "ascending");
            else th.removeAttribute("aria-sort");
        });
        const body = $("lb-body");
        body.replaceChildren();
        if (!data) return;
        const s = SORTS[sortKey];
        // Ranked teams with a value, best first; then the others by name.
        const byName = (a, b) => a.name.localeCompare(b.name);
        const top = data.teams.filter((t) => t.ranked && s.get(t) != null).sort((a, b) => s.dir * (s.get(a) - s.get(b)));
        const rest = data.teams.filter((t) => !top.includes(t)).sort((a, b) => (b.ranked - a.ranked) || byName(a, b));
        // A team that never traded has nothing to show; at the end, unranked teams keep their figures.
        const shows = (t) => t.ranked || data.status === "final";
        [...top, ...rest].forEach((t, i) => {
            const tr = el("tr", i < top.length ? "" : "lb-unranked");
            tr.appendChild(el("td", "lb-rank", i < top.length ? String(i + 1) : "–"));
            tr.appendChild(el("td", "lb-team", t.name));
            for (const key of ["return", "volatility"]) {
                const v = shows(t) ? SORTS[key].get(t) : null;
                let cls = "lb-num" + (key === sortKey ? " on" : "");
                if (key === "return" && v) cls += v > 0 ? " up" : " down";
                tr.appendChild(el("td", cls, v == null ? "–" : SORTS[key].fmt(v)));
            }
            body.appendChild(tr);
        });
        const empty = $("lb-empty");
        empty.hidden = top.length > 0 || data.status === "upcoming";
        empty.textContent = s.empty;
    }

    function status() {
        const s = $("lb-status");
        const txt = $("lb-status-text");
        s.className = "lb-status";
        if (!data) return;
        const phase = data.phase ? data.phase + " · " : "";
        const when = (iso) => new Date(iso).toLocaleString("en-GB", { day: "numeric", month: "short", hour: "2-digit", minute: "2-digit", timeZone: "UTC" }) + " UTC";
        if (data.status === "upcoming") {
            txt.textContent = phase + "Starts " + when(data.from);
        } else if (data.status === "final") {
            s.classList.add("final");
            txt.textContent = phase + "Final standings";
        } else {
            s.classList.add("live");
            const mins = Math.max(0, Math.round((Date.now() - new Date(data.generated_at)) / 60000));
            txt.textContent = (phase || "Live · ") + "updated " + (mins < 1 ? "just now" : mins + " min ago");
        }
    }

    function note() {
        if (!data) return;
        $("lb-note").textContent =
            "PnL is the net return on initial capital, updated every 5 minutes. " +
            "Volatility is the standard deviation of daily equity returns over completed days (00:00 UTC): lower is better. " +
            "Select a column to rank by it. " +
            `At the end, only teams that traded on at least ${data.min_days} days are ranked.`;
    }

    async function load() {
        try {
            if (!DATA_URL) throw new Error("data_url is not set in _data/leaderboard.yml");
            const res = await fetch(DATA_URL + (DATA_URL.includes("?") ? "&" : "?") + "t=" + Date.now(), { cache: "no-store" });
            if (!res.ok) throw new Error("HTTP " + res.status);
            data = await res.json();
            render();
            note();
        } catch (e) {
            console.error("leaderboard:", e);
            if (!data) {
                $("lb-status-text").textContent = "The leaderboard is not available yet.";
                $("lb-empty").hidden = false;
                $("lb-empty").textContent = "Check back when the competition starts.";
            }
        }
        status();
    }

    render();
    load();
    setInterval(() => { if (!document.hidden) load(); }, REFRESH_MS);
    setInterval(status, 30000);
    document.addEventListener("visibilitychange", () => { if (!document.hidden) load(); });
})();
</script>
