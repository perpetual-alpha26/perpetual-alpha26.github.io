---
layout: default
title: Register
---

<div class="page-content">
    <h1>Register Your Team</h1>
    <p>Registration for Perpetual Alpha is open! Gather your team of 1 to 4 members and enroll using the form below.</p>

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
                            <th style="text-align: center;">Available ICAIF</th>
                        </tr>
                    </thead>
                    <tbody>
                        <!-- Member 1 (Required) -->
                        <tr>
                            <td style="font-weight: bold; color: var(--accent-color);">1)</td>
                            <td><input type="text" name="member1_name" required placeholder="Required"></td>
                            <td><input type="text" name="member1_surname" required placeholder="Required"></td>
                            <td><input type="email" name="member1_email" required placeholder="Required"></td>
                            <td><input type="text" name="member1_affiliation" required placeholder="Required"></td>
                            <td style="text-align: center;"><input type="checkbox" name="member1_icaif"></td>
                        </tr>
                        <!-- Member 2 -->
                        <tr>
                            <td style="font-weight: bold; color: var(--text-secondary);">2)</td>
                            <td><input type="text" name="member2_name"></td>
                            <td><input type="text" name="member2_surname"></td>
                            <td><input type="email" name="member2_email"></td>
                            <td><input type="text" name="member2_affiliation"></td>
                            <td style="text-align: center;"><input type="checkbox" name="member2_icaif"></td>
                        </tr>
                        <!-- Member 3 -->
                        <tr>
                            <td style="font-weight: bold; color: var(--text-secondary);">3)</td>
                            <td><input type="text" name="member3_name"></td>
                            <td><input type="text" name="member3_surname"></td>
                            <td><input type="email" name="member3_email"></td>
                            <td><input type="text" name="member3_affiliation"></td>
                            <td style="text-align: center;"><input type="checkbox" name="member3_icaif"></td>
                        </tr>
                        <!-- Member 4 -->
                        <tr>
                            <td style="font-weight: bold; color: var(--text-secondary);">4)</td>
                            <td><input type="text" name="member4_name"></td>
                            <td><input type="text" name="member4_surname"></td>
                            <td><input type="email" name="member4_email"></td>
                            <td><input type="text" name="member4_affiliation"></td>
                            <td style="text-align: center;"><input type="checkbox" name="member4_icaif"></td>
                        </tr>
                    </tbody>
                </table>
            </div>
            
            <div style="margin-top: 2rem; text-align: center;">
                <button type="submit" class="btn">Submit Registration</button>
            </div>
        </form>
    </div>

    <h2 style="margin-top: 4rem;">Questions?</h2>
    <p>If you have any questions regarding the competition, eligibility, or technical requirements, please don't hesitate to reach out to the organizers.</p>
    <p><strong>Contact Email:</strong> <a href="mailto:{{ site.email }}">{{ site.email }}</a></p>
</div>
