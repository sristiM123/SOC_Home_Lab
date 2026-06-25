# How to publish this to GitHub and update it daily

A plain, step-by-step guide. Follow once to publish, then use the short daily loop.

---

## One-time setup

1. **Create the repo on GitHub.**
   Go to github.com → New repository → name it `home-soc-lab` → set it **Public** (recruiters need to see it) → do NOT add a README (you already have one) → Create.

2. **On your machine, in the folder containing these files:**

   ```bash
   git init
   git add .
   git commit -m "Initial commit: home SOC lab — Wazuh setup through attack-and-detect"
   git branch -M main
   git remote add origin https://github.com/<your-username>/home-soc-lab.git
   git push -u origin main
   ```

   Replace `<your-username>` with your GitHub username. If it asks for a password, use a **Personal Access Token** (GitHub → Settings → Developer settings → Personal access tokens), not your account password.

3. **Add your screenshots.** Drop your saved dashboard/terminal images into the `screenshots/` folder, then in each writeup add a line like:
   ```markdown
   ![SSH brute-force spike](../../screenshots/ssh-bruteforce-spike.png)
   ```
   Commit and push again.

---

## The daily loop (takes 5 minutes)

Each day you learn something:

1. Copy `TEMPLATE.md` into the right `docs/` folder (or a new one for a new topic).
2. Fill in the six short sections — goal, what I did, what broke, evidence, what I learned, SOC relevance.
3. Add any screenshot to `screenshots/`.
4. Update the status table in the main `README.md` (change "Planned" to "Done", etc.).
5. Push:
   ```bash
   git add .
   git commit -m "Add: <topic> writeup"
   git push
   ```

---

## The matching LinkedIn post

After each push, write a short LinkedIn post (3–5 lines):

- **One specific thing** you learned (not "I studied today").
- A line on why it matters for SOC work.
- A link to that day's writeup in the repo.

Example:
> Today in my home SOC lab: learned that a log reaching the SIEM isn't the same as an alert. Logs are ingested; alerts only fire when a rule matches. Searching by filename finds nothing — events are grouped by rule. Small distinction, big source of confusion for beginners. Full writeup → [repo link]
>
> #CyberSecurity #SOC #BlueTeam #Wazuh #SIEM

---

## A note on honesty (it matters for credibility)

Write what actually happened, including the failures and wrong turns. "I thought it was X, it turned out to be Y, here's how I diagnosed it" reads as *more* competent to technical reviewers than a flawless tutorial. Don't overstate — e.g. note where a built-in rule already did the job and your custom rule was practice. That nuance is exactly what separates someone who understands from someone who copied steps, and interviewers can tell the difference.
