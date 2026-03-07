# 28 — Behavioral & Situational Interview Questions

---

## Q1. Tell me about yourself.

**Framework:** Present → Past → Future

**Answer:**
"I'm a mobile app engineer with [X] years of experience building production Flutter/Android/iOS apps. Currently at [Company], I lead the development of [app name] which serves [X] users. Previously, I worked at [Company] where I built [feature/app]. I'm passionate about clean architecture, performance optimization, and delivering great user experiences. I'm excited about this role because [specific reason about the company/team]."

---

## Q2. Describe a challenging technical problem you solved.

**Framework:** STAR (Situation, Task, Action, Result)

**Example Answer:**
- **S:** Our Flutter app had severe jank on the feed screen — frames were dropping to 20 FPS on mid-range devices.
- **T:** I needed to identify the bottleneck and get performance to 60 FPS without redesigning the feature.
- **A:** I used Flutter DevTools to profile. Found three issues: (1) unnecessary rebuilds due to `setState` at the top level, (2) un-cached network images being re-decoded, (3) complex `ClipRRect` on every list item. I refactored to use `Selector` from Provider for targeted rebuilds, added `CachedNetworkImage` with `memCacheWidth`, and replaced `ClipRRect` with pre-rounded images.
- **R:** Frame rate improved from 20 FPS to stable 60 FPS. Memory usage dropped 35%. App store rating improved from 3.8 to 4.5.

---

## Q3. How do you handle disagreements with team members about technical decisions?

**Answer:**
"I approach disagreements as opportunities to find the best solution. I start by actively listening to understand the other person's perspective and reasoning. Then I present my viewpoint with objective criteria — data, benchmarks, user impact, or maintenance cost. If we can't agree, I suggest prototyping both approaches or doing a time-boxed spike. Ultimately, I value team alignment over being right, so I'm willing to commit to the team's decision even if I initially disagreed, as long as we've had a fair discussion."

---

## Q4. Tell me about a time you had to learn a new technology quickly.

**Example:**
"When our team decided to migrate from Provider to Riverpod, I had one sprint to get up to speed and guide the team. I spent the first two days reading the official documentation and building a sample app. Then I created a migration guide document for the team, set up a shared example project, and led a knowledge-sharing session. I also pair-programmed with team members during the first few migrations. By the end of the sprint, the team was comfortable with Riverpod, and we migrated 60% of our state management without any production issues."

---

## Q5. How do you prioritize tasks when you have multiple deadlines?

**Answer:**
"I use the Eisenhower Matrix: urgent+important first, important+not-urgent scheduled next. For mobile specifically, I prioritize: (1) production bugs affecting users, (2) features blocking other teams, (3) sprint commitments, (4) tech debt. I communicate proactively with stakeholders when priorities conflict and negotiate realistic timelines. I also break large tasks into smaller deliverables so there's always measurable progress."

---

## Q6. Describe a time you made a mistake in production. What happened?

**Example:**
"I once pushed a release that caused a crash on Android 10 devices due to a missing null check in a platform channel. Our crash rate spiked from 0.1% to 5%. I immediately identified the issue via Crashlytics, prepared a hotfix within an hour, and pushed an expedited release. I then implemented three changes to prevent recurrence: (1) added comprehensive null safety checks, (2) added a pre-release checklist including testing on multiple OS versions, (3) set up automated crash rate monitoring alerts. The crash rate returned to 0.08% — better than before."

---

## Q7. How do you ensure code quality in your team?

**Answer:**
- **Code reviews** — every PR reviewed by at least one peer
- **Automated testing** — unit tests for business logic, widget tests for UI, integration tests for critical flows
- **CI/CD** — lint, format, test on every commit
- **Architecture guidelines** — documented patterns (Clean Architecture, naming conventions)
- **Pair programming** — for complex features or onboarding
- **Tech debt sprints** — dedicated time for refactoring
- **Static analysis** — `flutter analyze`, custom lint rules

---

## Q8. How do you handle working with a difficult stakeholder?

**Answer:**
"I focus on understanding their underlying needs. Often 'difficult' stakeholders are frustrated because of past experiences or misaligned expectations. I schedule regular check-ins to build trust, use demos instead of just status updates (seeing is believing), and translate technical constraints into business terms. I also set clear expectations about timelines and trade-offs upfront, so there are fewer surprises."

---

## Q9. Where do you see yourself in 5 years?

**Answer:**
"I want to grow into a senior/lead mobile engineer role where I can influence architecture decisions, mentor junior developers, and drive technical strategy. I'm particularly interested in [specific area — performance, design systems, cross-platform]. I also want to contribute to the open-source Flutter/mobile community through talks or packages."

---

## Q10. Why are you leaving your current role?

**Tips:**
- Be positive — focus on what you're moving TOWARD, not away from
- Don't badmouth current employer
- Be specific about growth

**Example:** "I've learned a lot at [Company], but I'm looking for a role where I can work on a product at a larger scale with more complex technical challenges. Your team's work on [specific product/technology] is exactly the kind of problem I want to solve."

---

## Q11. How do you stay updated with new mobile technologies?

**Answer:**
- Follow Flutter/Android/iOS official blogs and release notes
- Flutter Weekly, Android Weekly, iOS Dev Weekly newsletters
- Watch conference talks (Flutter Forward, Google I/O, WWDC, DroidCon)
- Read source code of popular open-source packages
- Build side projects with new technologies
- Participate in communities (Reddit, Discord, Stack Overflow)
- Follow key developers on X/Twitter and GitHub

---

## Q12. How do you handle a situation where a feature deadline is unrealistic?

**Answer:**
"I assess the gap between scope and timeline, then present options to stakeholders: (1) reduce scope — what's the MVP that delivers core value? (2) extend timeline — what's a realistic date? (3) increase resources — can we parallelize with more engineers? I use data to support my assessment — story points, past velocity, technical complexity. I've found that most stakeholders appreciate transparency and prefer a realistic plan over over-promising."

---

## Q13. Describe your experience with code reviews.

**Answer:**
"I give and receive code reviews on every PR. When reviewing, I focus on: correctness (does it work?), architecture (does it fit the codebase?), edge cases, and testability. I avoid style nitpicks — that's what linters are for. When receiving reviews, I appreciate all feedback and treat it as a learning opportunity. I ask clarifying questions rather than getting defensive. I believe code reviews are one of the highest-ROI activities for code quality and team growth."

---

## Q14. Tell me about a time you had to make a trade-off between speed and quality.

**Example:**
"We had a critical feature needed for a marketing launch. The ideal approach was a full architectural refactor, but we had two weeks. I proposed a phased approach: Phase 1 (2 weeks) — implement the feature with the existing architecture, add good test coverage, and document known tech debt. Phase 2 (next sprint) — refactor to the proper architecture. This let us hit the deadline while ensuring the code was maintainable. We completed Phase 2 the following sprint without issues."

---

## Q15. What questions do you have for us?

**Great questions to ask:**
1. "What does the mobile architecture look like today, and what are the plans for improving it?"
2. "How does the mobile team collaborate with backend, design, and product?"
3. "What's the testing strategy and CI/CD pipeline like?"
4. "What's the biggest technical challenge the mobile team is currently facing?"
5. "How do you handle technical debt?"
6. "What does the code review process look like?"
7. "What does growth and progression look like for this role?"
8. "What's the on-call/incident response process like?"

---

## Q16. Why do you want to work at this company specifically?

**Answer (framework):**

Structure: **Company research + alignment with your goals**

1. **Product/Mission**: "I'm impressed by [specific product/feature]. I use it daily and have ideas for improving [aspect]."
2. **Technology**: "Your team's work on [open-source project / blog post / tech talk] aligns with my interests in [area]."
3. **Growth**: "The role offers [specific opportunity] which matches my goal to [grow in specific area]."
4. **Culture**: "I value [specific company value] and saw it reflected in [specific example — blog, Glassdoor, interview experience]."

**Red flags to avoid:** Don't say "I need a job" or only mention salary/perks.

---

## Q17. Describe a project you're most proud of.

**Answer (STAR):**

**Situation:** "At [Company], our app had a 2.3-star rating due to performance issues and crashes."

**Task:** "I was tasked with leading the performance overhaul initiative."

**Action:**
- Profiled the app using DevTools/Instruments, found 3 major bottlenecks
- Implemented lazy loading, image caching, and background data sync
- Rewrote the core feed using slivers for better scroll performance
- Added crash reporting (Crashlytics) and monitoring
- Set up CI/CD with automated performance benchmarks

**Result:** "Crash-free rate went from 94% to 99.7%. App rating improved to 4.5 stars in 3 months. App startup time reduced by 60%."

---

## Q18. How do you mentor junior developers?

**Answer:**

**Approach:**
1. **Pair programming**: Regular sessions on real features, not toy examples
2. **Code reviews**: Explain the "why", not just "change this". Ask questions instead of dictating.
3. **Gradual responsibility**: Start with bug fixes → small features → architecture decisions
4. **Documentation**: Maintain architecture docs and ADRs (Architecture Decision Records)
5. **Safe space**: Encourage questions, normalize saying "I don't know"

**Example:** "I mentored a junior dev by having them own a feature from design to deployment. I reviewed their architecture proposal, guided their implementation through PRs, and let them present the feature in sprint review. Within 3 months, they were confidently shipping features independently."

---

## Q19. Tell me about a time you failed.

**Answer (STAR):**

**Situation:** "I pushed a release without adequate testing on older devices."

**Task:** "The app crashed on Android 9 devices due to an API incompatibility I missed."

**Action:**
- Immediately rolled back via staged rollout (only 5% of users affected)
- Root-caused: used API 30+ method without version check
- Fixed with backward-compatible implementation
- Added device matrix testing to CI pipeline
- Created a pre-release checklist for the team

**Result:** "The incident lasted 2 hours. We added automated testing on API 28+ emulators, and haven't had a device-specific crash since. I learned the importance of testing on minimum supported API level."

---

## Q20. How do you handle cross-functional collaboration?

**Answer:**

**Working with designers:**
- Review designs early, flag technical constraints (animations, custom widgets)
- Ask for edge case designs (empty states, error states, loading states)
- Use design systems with shared tokens (colors, spacing, typography)

**Working with backend:**
- Co-design API contracts early (OpenAPI/Swagger)
- Discuss pagination, error codes, rate limiting upfront
- Mock APIs for parallel development

**Working with QA:**
- Share test plans, explain edge cases
- Provide debug builds with logging
- Automate regression tests to free QA for exploratory testing

**Working with product:**
- Ask clarifying questions during grooming
- Propose technical alternatives that achieve the same UX goal
- Communicate trade-offs in business terms, not technical jargon
