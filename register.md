---
layout: default
title: Register
---

<div class="page-content">
    <h1>Register Your Team</h1>
    <p>Registration for Perpetual Alpha is open! Gather your team of 1 to 4 members and enroll using the form below. At least one team member should be available to present at ICAIF '26 Milan.</p>

    <div class="glass-card" style="margin-top: 2rem;">
        <form action="YOUR_GETFORM_ENDPOINT" method="POST">
            <div style="overflow-x: auto;">
                <table class="registration-table">
                    <thead>
                        <tr>
                            <th></th>
                            <th>Name</th>
                            <th>Surname</th>
                            <th>Email</th>
                            <th>Affiliation</th>
                            <th style="text-align: center;">Available<br>Join<br>ICAIF</th>
                        </tr>
                    </thead>
                    <tbody>
                        <!-- Member 1 (Required) -->
                        <tr>
                            <td style="font-weight: bold; color: var(--accent-color);">1)</td>
                            <td><input type="text" name="member1_name" required placeholder="Required"></td>
                            <td><input type="text" name="member1_surname" required placeholder="Required"></td>
                            <td><input type="email" name="member1_email" required placeholder="Required" pattern="^[^@\s]+@[^@\s]+\.[^@\s]+$"></td>
                            <td><input type="text" name="member1_affiliation"></td>
                            <td style="text-align: center;"><input type="checkbox" name="member1_icaif"></td>
                        </tr>
                        <!-- Member 2 -->
                        <tr>
                            <td style="font-weight: bold; color: var(--text-secondary);">2)</td>
                            <td><input type="text" name="member2_name"></td>
                            <td><input type="text" name="member2_surname"></td>
                            <td><input type="email" name="member2_email" pattern="^[^@\s]+@[^@\s]+\.[^@\s]+$"></td>
                            <td><input type="text" name="member2_affiliation"></td>
                            <td style="text-align: center;"><input type="checkbox" name="member2_icaif"></td>
                        </tr>
                        <!-- Member 3 -->
                        <tr>
                            <td style="font-weight: bold; color: var(--text-secondary);">3)</td>
                            <td><input type="text" name="member3_name"></td>
                            <td><input type="text" name="member3_surname"></td>
                            <td><input type="email" name="member3_email" pattern="^[^@\s]+@[^@\s]+\.[^@\s]+$"></td>
                            <td><input type="text" name="member3_affiliation"></td>
                            <td style="text-align: center;"><input type="checkbox" name="member3_icaif"></td>
                        </tr>
                        <!-- Member 4 -->
                        <tr>
                            <td style="font-weight: bold; color: var(--text-secondary);">4)</td>
                            <td><input type="text" name="member4_name"></td>
                            <td><input type="text" name="member4_surname"></td>
                            <td><input type="email" name="member4_email" pattern="^[^@\s]+@[^@\s]+\.[^@\s]+$"></td>
                            <td><input type="text" name="member4_affiliation"></td>
                            <td style="text-align: center;"><input type="checkbox" name="member4_icaif"></td>
                        </tr>
                    </tbody>
                </table>
            </div>
            
            <div style="margin-top: 2rem; text-align: center;">
                <label style="color: var(--text-secondary); cursor: pointer; display: inline-flex; align-items: center; gap: 0.5rem; justify-content: center; margin-bottom: 1.5rem;">
                    <input type="checkbox" id="human-check" style="width: 1.2rem; height: 1.2rem; accent-color: var(--accent-color);">
                    Yes, I am human
                </label>
                <br>
                <button type="submit" class="btn" id="submit-btn">Submit Registration</button>
            </div>
        </form>
    </div>

    <h2 style="margin-top: 4rem;">Questions?</h2>
    <p>If you have any questions regarding the competition, eligibility, or technical requirements, please don't hesitate to reach out to the organizers.</p>
    <p><strong>Contact Email:</strong> <a href="mailto:{{ site.email }}">{{ site.email }}</a></p>
</div>

<script>
document.addEventListener('DOMContentLoaded', function() {
    const form = document.querySelector('form');
    const submitBtn = document.getElementById('submit-btn');
    const humanCheck = document.getElementById('human-check');

    const rows = [1, 2, 3, 4].map(i => ({
        name: form.querySelector(`[name="member${i}_name"]`),
        surname: form.querySelector(`[name="member${i}_surname"]`),
        email: form.querySelector(`[name="member${i}_email"]`),
        affiliation: form.querySelector(`[name="member${i}_affiliation"]`),
        icaif: form.querySelector(`[name="member${i}_icaif"]`)
    }));

    function isRowValid(row) {
        return row.name.value.trim() !== '' && 
               row.surname.value.trim() !== '' && 
               row.email.value.trim() !== '' && 
               row.email.checkValidity();
    }

    const enableRow = (row, enable) => {
        row.name.disabled = !enable;
        row.surname.disabled = !enable;
        row.email.disabled = !enable;
        row.affiliation.disabled = !enable;
        
        if (!enable) {
            row.name.value = '';
            row.surname.value = '';
            row.email.value = '';
            row.affiliation.value = '';
            row.icaif.checked = false;
        }
    };

    function updateFormState() {
        let row1Valid = isRowValid(rows[0]);
        let row2Valid = isRowValid(rows[1]);
        let row3Valid = isRowValid(rows[2]);
        let row4Valid = isRowValid(rows[3]);

        // Rule 5: Row n activated if row n-1 is completed
        enableRow(rows[1], row1Valid);
        enableRow(rows[2], row2Valid);
        enableRow(rows[3], row3Valid);

        // Rule 6: Checkbox enabled only if its row is valid
        rows[0].icaif.disabled = !row1Valid;
        rows[1].icaif.disabled = !row2Valid;
        rows[2].icaif.disabled = !row3Valid;
        rows[3].icaif.disabled = !row4Valid;

        rows.forEach(r => { if(r.icaif.disabled) r.icaif.checked = false; });

        // Rule 7: Submit button enabled if row 1 is valid AND at least one checkbox is checked AND human check is checked
        let anyChecked = rows.some(r => r.icaif.checked);
        submitBtn.disabled = !(row1Valid && anyChecked && humanCheck.checked);
        
        if (submitBtn.disabled) {
            submitBtn.style.opacity = '0.5';
            submitBtn.style.cursor = 'not-allowed';
        } else {
            submitBtn.style.opacity = '1';
            submitBtn.style.cursor = 'pointer';
        }
    }

    form.querySelectorAll('input').forEach(input => {
        input.addEventListener('input', updateFormState);
        input.addEventListener('change', updateFormState);
    });
    humanCheck.addEventListener('change', updateFormState);

    updateFormState();
});
</script>
