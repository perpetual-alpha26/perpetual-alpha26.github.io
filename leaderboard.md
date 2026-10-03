---
layout: default
title: Leaderboard
---

<div class="page-content">
    <div class="page-header" style="text-align: center; margin: 4rem 0 2rem 0;">
        <h1 style="font-size: 2.5rem; margin-bottom: 1rem; color: var(--accent-color);">Registered Teams</h1>
        <p style="color: var(--text-secondary); max-width: 600px; margin: 0 auto;">The following teams are officially registered. A green light indicates that the team has successfully completed the onboarding process submitting the registration form.</p>
    </div>
    
    <div class="teams-grid" style="display: grid; grid-template-columns: repeat(auto-fill, minmax(300px, 1fr)); gap: 1.5rem; margin-top: 3rem; margin-bottom: 5rem;">
        {% for team in site.data.teams %}
        <div class="glass-card team-card" data-teamname="{{ team.name | downcase | escape }}" data-teamemail="{{ team.email | downcase | escape }}" style="padding: 1.5rem; display: flex; justify-content: space-between; align-items: center; transition: all 0.3s ease;">
            <span class="team-name" style="font-weight: 600; font-size: 1.1rem;">{{ team.name }}</span>
            <div class="traffic-light red" title="Onboarding Pending"></div>
        </div>
        {% endfor %}
    </div>
</div>

<script>
    document.addEventListener("DOMContentLoaded", function() {
        const csvUrl = "https://docs.google.com/spreadsheets/d/e/2PACX-1vTicD-3XH4OKrikRKMxyG92AgnMFEF7kL-7u4zU12Le-k9S8EodKUvKxyOuKeDP2gi5KXXD1wDVZAI0/pub?gid=49022248&single=true&output=csv";
        
        fetch(csvUrl)
            .then(response => response.text())
            .then(csvText => {
                // Parse CSV (Handling quotes)
                const rows = [];
                let row = [];
                let inQuotes = false;
                let val = '';
                for (let i = 0; i < csvText.length; i++) {
                    const char = csvText[i];
                    if (inQuotes) {
                        if (char === '"') {
                            if (i + 1 < csvText.length && csvText[i+1] === '"') {
                                val += '"';
                                i++;
                            } else {
                                inQuotes = false;
                            }
                        } else {
                            val += char;
                        }
                    } else {
                        if (char === '"') {
                            inQuotes = true;
                        } else if (char === ',') {
                            row.push(val);
                            val = '';
                        } else if (char === '\n' || char === '\r') {
                            row.push(val);
                            rows.push(row);
                            val = '';
                            row = [];
                            if (char === '\r' && i + 1 < csvText.length && csvText[i+1] === '\n') {
                                i++; // skip \n after \r
                            }
                        } else {
                            val += char;
                        }
                    }
                }
                if (val !== '' || row.length > 0) {
                    row.push(val);
                    rows.push(row);
                }

                if (rows.length < 2) return;
                
                // Find "Team Name" and "Member 1 - Email" column indices
                const headers = rows[0].map(h => h.trim().toLowerCase());
                let teamColIdx = -1;
                let emailColIdx = -1;
                for (let i = 0; i < headers.length; i++) {
                    if (headers[i] === "team name") {
                        teamColIdx = i;
                    } else if (headers[i] === "member 1 - email") {
                        emailColIdx = i;
                    }
                }
                
                // Extract onboarded team names and emails
                const onboardedNames = new Set();
                const onboardedEmails = new Set();
                for (let i = 1; i < rows.length; i++) {
                    if (teamColIdx !== -1 && rows[i][teamColIdx]) {
                        onboardedNames.add(rows[i][teamColIdx].trim().toLowerCase());
                    }
                    if (emailColIdx !== -1 && rows[i][emailColIdx]) {
                        onboardedEmails.add(rows[i][emailColIdx].trim().toLowerCase());
                    }
                }

                // Update UI
                const teamCards = document.querySelectorAll('.team-card');
                teamCards.forEach(card => {
                    const tName = card.getAttribute('data-teamname');
                    const tEmail = card.getAttribute('data-teamemail');
                    if (onboardedNames.has(tName) || onboardedEmails.has(tEmail)) {
                        const light = card.querySelector('.traffic-light');
                        if (light) {
                            light.classList.remove('red');
                            light.classList.add('green');
                            light.setAttribute('title', 'Onboarded');
                        }
                    }
                });
            })
            .catch(error => console.error("Error fetching onboarding data:", error));
    });
</script>
