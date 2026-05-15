# 📅 Day 77 — Wednesday, 30 July 2026
# ✅ Week 11 Review + Portfolio Final Polish

---

## 🎯 Today's Goal

Week 11 is done — you've learned Git advanced, CI/CD, professional repo setup, system design, and behavioral interview prep. Today you review the week, polish your portfolio one last time, and create a plan for the remaining 4 weeks.

---

## ☀️ Morning Block (2 hours): Week 11 Quick Assessment

### 10 Review Questions

**Q1.** When would you use `git rebase` instead of `git merge`?

**Q2.** What command undoes the last commit but keeps your changes staged?

**Q3.** What does `git reflog` do and why is it useful?

**Q4.** Write a GitHub Actions trigger that runs daily at 8 AM SGT (0:00 UTC).

**Q5.** Name 3 things pre-commit hooks can automatically check.

**Q6.** What is branch protection and why do teams use it?

**Q7.** In a system design interview, what are the 4 steps of the framework?

**Q8.** What does STAR stand for in behavioral interviews?

**Q9.** Name 2 tradeoffs you'd discuss in a data pipeline system design.

**Q10.** Why would you choose Athena over Redshift for a small startup?

<details>
<summary>🔑 Answers</summary>

**A1:** Use rebase to update your feature branch with latest main (keeps linear history). Use merge to combine a completed feature branch into main (preserves history).

**A2:** `git reset --soft HEAD~1`

**A3:** reflog shows a log of all HEAD movements — it's the safety net that lets you recover from mistakes like accidental hard resets. Records are kept for ~30 days.

**A4:**
```yaml
on:
  schedule:
    - cron: '0 0 * * *'
```

**A5:** Trailing whitespace, Python formatting (black), SQL linting (sqlfluff), private key detection, merge conflict markers, YAML/JSON validity.

**A6:** Branch protection prevents direct pushes to important branches (like main). Requires PR review, CI checks to pass, and prevents force pushes. Used to maintain code quality and prevent accidental breakage.

**A7:** 1) Clarify requirements, 2) Propose high-level architecture, 3) Deep dive into components, 4) Discuss tradeoffs and scaling.

**A8:** Situation, Task, Action, Result.

**A9:** Cost vs performance (Athena vs Redshift), batch vs streaming (daily vs real-time), build vs buy (self-managed vs managed services).

**A10:** Athena: pay per query ($5/TB scanned), no infrastructure to manage, perfect for 5-10 analysts. Redshift: $180+/month minimum, overkill for small teams, better for heavy concurrent usage.
</details>

---

## 🌤️ Afternoon Block (2 hours): Portfolio Final Polish

### GitHub Profile Audit

Check your GitHub profile page (github.com/YOUR_USERNAME):

- [ ] Profile picture set
- [ ] Bio mentions "Data Engineer" or "Aspiring Data Engineer"
- [ ] Location set (Singapore or Malaysia)
- [ ] Pinned repositories show your best 6 projects
- [ ] README.md on your profile repo (github.com/YOUR_USERNAME/YOUR_USERNAME)

### Profile README (if you don't have one)

Create a repo named exactly your GitHub username. Add a README.md:

```markdown
# Hi, I'm [Your Name] 👋

🎓 Aspiring Data Engineer based in Singapore

## 🛠️ Tech Stack
![Python](https://img.shields.io/badge/Python-3776AB?style=flat&logo=python&logoColor=white)
![SQL](https://img.shields.io/badge/SQL-4479A1?style=flat&logo=postgresql&logoColor=white)
![AWS](https://img.shields.io/badge/AWS-232F3E?style=flat&logo=amazon-aws&logoColor=white)
![dbt](https://img.shields.io/badge/dbt-FF694B?style=flat)
![Airflow](https://img.shields.io/badge/Airflow-017CEE?style=flat&logo=apache-airflow&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=white)

## 🚀 Featured Projects

| Project | Description | Tech Stack |
|---------|-------------|-----------|
| [Food Delivery Warehouse](link) | Star schema + SCD + Airflow + AWS | SQL, dbt, Airflow, AWS |
| [Serverless Pipeline](link) | S3 → Glue → Athena analytics | PySpark, AWS Glue, Athena |
| [Crypto Pipeline](link) | API → Python ETL → PostgreSQL | Python, Pandas, API |
| [Food Delivery dbt](link) | Analytics with dbt | dbt, SQL, GitHub Actions |
| [Retail Analysis](link) | SQL analysis of retail data | SQL, PostgreSQL |

## 📊 Currently Learning
- Advanced SQL optimization
- Data quality frameworks
- System design for data platforms

## 📫 How to reach me
- LinkedIn: [your-linkedin]
- Email: [your-email]
```

### Pinned Repos Checklist

Pin your best 6 repos. Order matters — put the most impressive first:

1. `food-delivery-warehouse` (most comprehensive: star schema + SCD + Airflow + AWS)
2. `makanexpress-serverless` (shows cloud skills: Glue + Athena + PySpark)
3. `food-delivery-dbt` (shows modern data stack: dbt + CI/CD)
4. `crypto-data-pipeline` (shows Python + API + ETL)
5. `retail-sales-analysis` (shows SQL foundation)
6. Your GitHub Pages portfolio site

---

## 🌙 Evening (1 hour): Plan for Weeks 12-15

### What's Coming

| Week | Focus | Key Output |
|------|-------|-----------|
| 12 | Resume + LinkedIn + Portfolio Polish | Application-ready profile |
| 13 | Interview Prep (SQL live coding) | 130+ problems practiced |
| 14 | Interview Prep (system design + behavioral) | Applying to jobs |
| 15 | Final Push — Apply everywhere | Interviews ongoing |

### 📝 Today's Checklist

- [ ] Completed Week 11 review questions (8+/10)
- [ ] GitHub profile updated (bio, photo, location)
- [ ] Profile README created/updated
- [ ] Pinned repos reordered (best first)
- [ ] All repos have CI badges passing ✅
- [ ] Plan reviewed for Weeks 12-15
- [ ] Ready to start Week 12 tomorrow

---

*Week 11 complete! 11 of 15 weeks done. The home stretch begins — resume, interview prep, and job applications.* 🏠
