# Breezy Take-Home — Click Here Labs Code Test

## Table of Contents

- [Part 1: Bug Fix: FAQ](#part-1-bug-fix-faq)
  - [The problem](#the-problem)
  - [Root cause](#root-cause)
  - [How I found it](#how-i-found-it)
  - [The fix](#the-fix)
  - [Result](#result)
- [Part 2: Smart Plan Finder](#part-2-smart-plan-finder)
  - [What I built and why](#what-i-built-and-why)
  - [Process](#process)
  - [Changes I made integrating it into my actual file](#changes-i-made-integrating-it-into-my-actual-file)
  - [How it works technically](#how-it-works-technically)
  - [Setup instructions](#setup-instructions)
  - [Link To The AI Transcript](#link-to-the-ai-transcript)
  - [What I'd improve with more time](#what-id-improve-with-more-time)
  - [Tools I used] (#Tools-I-used)

## Part 1: Bug Fix: FAQ

# The problem

The FAQ accordion did not behave like it was supposed to. Clicking a question opened it, but clicking it again did nothing, and clicking a different question left the first one open too. Every clicked item just stacked up open.

# Root cause

The `toggleFaq` function only ever added the `open` class to the clicked button and its answer. It never removed it from anything, so nothing could close.

```js
function toggleFaq(btn) {
  const answer = btn.nextElementSibling;
  btn.classList.add('open');
  answer.classList.add('open');
}
```

# How I found it

1. I clicked a FAQ question and saw that it opened correctly.
2. I clicked it again, but it stayed open.
3. I used Inspect(DevTools) to check the FAQ element while clicking it.
4. I saw that the open class was being added but never removed.
5. I checked the toggleFaq function and found that it only used classList.add('open').

# The fix

Before changing anything, I check whether the clicked item was already open. Then I close every open item. If the clicked item was not open before, I open it.

```js
function toggleFaq(btn) {
  const answer = btn.nextElementSibling;
  const wasOpen = btn.classList.contains('open');

  document.querySelectorAll('.faq-q.open').forEach(q => {
    q.classList.remove('open');
    q.nextElementSibling.classList.remove('open');
  });

  if (!wasOpen) {
    btn.classList.add('open');
    answer.classList.add('open');
  }
}
```

# Result

Only one FAQ item can be open at a time. Clicking an open item closes it. Clicking a different item closes the current one and opens the new one. It now works as expected.



## Part 2: Smart Plan Finder

I named this the Smart Plan Finder rather than an 'AI quiz' on purpose, since the recommendation comes from a transparent weighted-scoring model, not a live AI call, and I didn't want the name to imply something the implementation doesn't do.

# What I built and why

I added a "Smart Plan Finder," a 5-question quiz that recommends one of Breezy's
three existing pricing plans based on the user's answers, then scrolls to and
highlights the matching card in the real pricing section.

The prompt listed "an AI-integrated smart quiz that varies answers based on
previous input" as one of the open-ended options. I picked it because it's the
one option on the list that's actually about a decision being made from
evolving input, which is close to what I work on in my own research (adaptive
scheduling that picks an action based on changing state). I wanted to build
something in that spirit rather than bolt on a feature I had no real opinion
about.

Before writing any code, I worked through one design question with an AI
assistant: should this quiz call an external LLM API to generate its
recommendation? I decided no. This file is static, self-contained, and has no
backend, so any API key embedded in it would ship in plain text to anyone who
opens dev tools or views source. That's a real vulnerability, not just a
style preference, and I didn't think it was worth the risk for the sake of
saying "it calls an LLM." Instead, I built the "AI-integrated" feel with a
transparent, inspectable weighted-scoring model: every answer adds a fixed
number of points to one or more plans, and the recommendation is just
whichever plan has the most points at the end. A production version of this
feature would be a good candidate for moving that scoring, or a real LLM
call, behind a server-side endpoint where a key can be kept secret.

# Process

My initial brainstorm and pseudocode sketch (including the chatbot-vs-quiz decision and a rough wireframe of the question/progress/result flow) are attached as Process Work.pdf

**1. Deciding on the idea.** I talked through the API question above first,
since it changed what "AI-integrated" could actually mean here. Once I'd
settled on a scoring model instead of a live API call, I asked for a range of
question *angles* (usage frequency, team size, budget sensitivity, feature
needs, growth trajectory, etc.) along with what each one actually measures,
so I could pick five that made sense for Breezy's three tiers and write the
actual copy myself rather than use generic SaaS phrasing.

**2. Pseudocode.** Before touching real code, I wrote out the core scoring
loop as pseudocode to make sure I understood the state machine before I
looked at any implementation:

```
# Smart Plan Finder - core scoring idea

# 3 plans: casual, power, enterprise
scores = {casual: 0, power: 0, enterprise: 0}
questionIndex = 0
TOTAL_QUESTIONS = 5

# each question's answers are just pre-set point values per plan
# e.g. picking "answer A" on Q1 might add +2 casual, +0 power, +0 enterprise

onAnswerClick(qIndex, pointsForThisAnswer):
    add pointsForThisAnswer to scores
    questionIndex = qIndex + 1

    # figure out who's winning so far
    leader = plan with highest score in scores
    total = sum of all scores
    confidence = scores[leader] / total   # as a %, only meaningful once total > 0

    IF questionIndex < TOTAL_QUESTIONS:
        show next question
        update "leaning toward {leader} ({confidence}%)" hint text
    ELSE:
        showResult(leader, confidence)

showResult(leader, confidence):
    display leader's plan name + a canned reason string per plan
    maybe show a small bar per plan = scores[plan] / total, so user can see
    the breakdown instead of just trusting a black-box verdict
    add a button that scrolls down and highlights the matching pricing card

restartQuiz():
    reset scores back to 0
    reset questionIndex to 0
    hide result, show question 0 again

# note to self: don't overthink this into real bandit math, there's no
# session history to learn from in a single quiz run, adding fake "learning"
# would just be decoration
```

I deliberately stopped myself from adding real adaptive/bandit-style logic
here. There's no session history within a single quiz run to actually learn
from, so implementing something like UCB or epsilon-greedy would have been
decoration dressed up as substance rather than a real use of the technique.

**3. Reviewing the pseudocode before implementing.** I sent that pseudocode
back for review rather than treating it as final, and two real gaps came out
of that:
- `confidence = leader's score / total score` conflates "how much this plan
  is favored" with "how many total points have been handed out." A cleaner
  signal is the *margin* over the runner-up divided by the total possible
  points, mapped to a small number of plain-language bands ("Strong match,"
  "Good match," "Close call") instead of a fake-precise percentage a 5-question
  integer tally doesn't actually support.
- The budget question I wanted to treat as a "modifier" doesn't need special
  branching in the scoring engine at all. Giving it a wider point spread
  (±3 instead of ±1/±2), including a small negative value against the plan
  it argues against, achieves the same effect through magnitude rather than
  through separate logic paths.

**4. Implementation.** With the scoring approach settled, I had the actual
HTML/CSS/JS written to match Breezy's existing design tokens (`--sky-*`,
`--slate-*`, `--radius`, `--shadow-*`), with a progress bar, one question
visible at a time with a fade transition, a collapsed "see how we scored
this" breakdown so the recommendation isn't visually undercut by three
equal-weight bars, and a CTA that scrolls to and highlights the matching
pricing card via `data-plan` attributes.

# Changes I made integrating it into my actual file

The generated section was written as a generic drop-in with placeholder plan
names ("Casual," "Power," "Enterprise") and placeholder question copy. To fit
it into the real site, I:

- Renamed the plans in `PLAN_META` to match Breezy's actual tier names
  (Casual Breather, Power Inhaler, Enterprise Lung) and rewrote each plan's
  `reason` string to reference things those tiers actually offer.
- Rewrote all five question titles and answer labels in Breezy's voice
  (nostril optimization, the Monthly Air Report™, SSO as "Single Sniff-On"),
  keeping the underlying `points` objects and scoring logic completely
  unchanged, since the logic doesn't care about wording.
- Added `data-plan="casual"`, `data-plan="power"`, and `data-plan="enterprise"`
  to my three real `.price-card` elements so the "View this plan" button and
  scroll-highlight behavior actually resolve to a card, instead of silently
  finding nothing.
- Merged the quiz's `<style>` block into my existing stylesheet and folded
  its `<script>` into my existing script tag rather than
  keeping it as a separate embedded block, so the file stays a single
  self-contained HTML document as the test requires.
- The generated code was a standalone section with no heading or framing, meant to be dropped in anywhere. I wrapped it in my own outer section following the same layout pattern as every other section on the page (.container, .section-label, .section-title, .section-sub), and wrote the heading and subtitle copy myself, so it reads as a native part of the site rather than a pasted-in widget.
- While testing, I found a bug in the  the generated code: the .spf-breakdown element had display: flex set unconditionally in CSS, so score breakdown was visible immediately instead of being hidden until toggled, caused by a CSS rule that overrode the element's hidden attribute. I fixed it by adding an explicit [hidden] rule.
-I folded the generated script into my site's existing script tag rather than keeping it as a separate block, so the page stays one script section instead of two. I also replaced the generated code's generic numbered section comments with my own comment explaining the actual reasoning behind the no-external-API decision, since that context is more useful to a future reader than a table of contents for a drop-in snippet.
- Verified end to end that answering all five questions correctly reaches a
  result, that the confidence band changes appropriately for a close-call
  case versus a clear one, and that clicking "View this plan" actually
  scrolls to and highlights the right card for all three possible outcomes.

# How it works technically

- `QUESTIONS` is an array of 5 questions, each with 3 answers. Each answer
  carries a `points` object like `{ casual: 2, power: 0, enterprise: 0 }`.
- Clicking an answer adds its points to a running `scores` object, advances
  to the next question with a short fade transition, and re-renders the
  progress bar.
- After the last question, `getRanked()` sorts the three plans by score.
  `getConfidenceLabel()` computes the margin between the top plan and the
  runner-up as a fraction of total points earned, then maps that margin to
  one of three plain-language bands rather than displaying a raw percentage.
- The result view shows the winning plan's name and a one-line reason, the
  confidence band, and a collapsed "see how we scored this" breakdown that
  renders a small bar per plan when expanded, with the leading plan visually
  distinct from the other two.
- "View this plan" reads the winning plan's id off the button's dataset,
  looks up the matching `[data-plan="..."]` element among the real pricing
  cards, scrolls to it, and applies a brief highlight class that removes
  itself after 2.2 seconds.
- "Retake the quiz" resets all state and re-shows the first question.

# Setup instructions

None. It's plain HTML/CSS/JS embedded in the existing single file, open it in a browser and it works with no build step, server, or dependencies. I deployed the code at [https://breezy_intern_test.github.io/](https://maisharchy.github.io/breezy_intern_test.github.io/) and the code is at [https://github.com/maisharchy/breezy_intern_test.github.io](https://github.com/maisharchy/breezy_intern_test.github.io) this github repository.

# Link To The AI Transcript

https://claude.ai/share/72d40231-5d09-4380-b901-7c888059003f 

# What I'd improve with more time

- Move the scoring (or a real LLM-based recommendation) to a small
  server-side endpoint, so an actual API key could be used safely and the
  recommendation logic wouldn't be visible in page source.
- Add basic analytics on which questions produce the most "Close call"
  results, which would be a useful signal for whether the pricing tiers
  themselves need clearer differentiation, not just better quiz questions.
- Persist quiz answers in memory across a session (not localStorage, since
  this is a marketing site and I'd want to think through privacy
  implications first) so a user who navigates away and back doesn't lose
  their progress.
- Expand test coverage beyond manual click-through testing, ideally with a
  small script that runs all `3^5` possible answer combinations and checks
  that no combination produces an undefined or crashing result.
# Tools I used
I used google, and online resources to read any programming syntax, how a praticular functions behave etc. to understand better how I can implement or fix bugs. I used google inspect(Devtool) to see the problem and how the code behaves. I used GoodNotes to write my initial though process, I used claude for implemnting my idea based on my pseudocode and VS Code to write code myself. I deployed the code at [https://breezy_intern_test.github.io/](https://maisharchy.github.io/breezy_intern_test.github.io/) and the code is at [https://github.com/maisharchy/breezy_intern_test.github.io](https://github.com/maisharchy/breezy_intern_test.github.io) this github repository.

Overall, I completed all the tasks in around 4 hours to comply by the recommended timeline.
