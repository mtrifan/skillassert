# skillassert.com

A quiz site where learners test themselves on a topic at a chosen tier and review what they missed. Content is
generated ahead of time and curated by an admin before learners see it.

## Language

### Subjects

**Category**:
A node in the subject tree shown on the homepage, such as IT, Frontend or History. Categories nest.
_Avoid_: domain, section, subcategory (a subcategory is just a child category)

**Topic**:
A leaf category that questions belong to, such as Angular or SQL.
_Avoid_: subject, field, area

**Tier**:
The difficulty band of a question and of a quiz: Novice, Intermediate or Senior.
_Avoid_: level, difficulty, seniority

### Content

**Question**:
A prompt with answer options, belonging to exactly one topic, one tier and one kind, with an explanation.
_Avoid_: item, card

**Option**:
One selectable answer of a question, marked correct or not.
_Avoid_: answer, choice, alternative

**Kind**:
Whether a question has exactly one correct option (single-choice) or several (multi-select).
_Avoid_: type, mode, format (format is a quiz-level choice)

**Explanation**:
The text on a question that justifies its correct options, shown after a quiz for questions the learner missed.
_Avoid_: rationale, solution

**Draft** / **Approved** / **Rejected**:
The three states of a question: generated and awaiting review, accepted into the question bank, or declined.
_Avoid_: pending, published, live, candidate

**Question bank**:
All approved questions, the pool a quiz is drawn from.
_Avoid_: quiz candidates, pool, catalog

**Generation batch**:
One admin-triggered run that asks an LLM for questions for a topic, tier and kind. Its output enters the review
queue as drafts.
_Avoid_: job, import, run

**Review queue**:
The admin's list of questions awaiting a decision: new drafts and flagged questions.
_Avoid_: moderation, inbox

### Playing

**Quiz**:
A set of questions served to a learner in one sitting: today sampled from the question bank for one topic, tier and
format; later possibly a curated fixed set.
_Avoid_: test, exam, set

**Quiz settings**:
The learner's choices at quiz start: topic, tier, format, source, and whether the quiz is timed.
_Avoid_: options, configuration, mode

**Quiz source**:
Where a quiz draws its questions from: the question bank, the learner's missed questions, or their bookmarks.
_Avoid_: mode, pool, filter

**Format**:
Which kinds of questions a quiz draws: single-choice only, multi-select only, or mixed.
_Avoid_: kind (kind belongs to one question), type

**Attempt**:
One learner's run through a quiz: their answers, timing and result.
_Avoid_: session, submission, try

**Trial quiz**:
The one attempt a learner may take before signing in.
_Avoid_: guest quiz, demo, free quiz

### People

**Learner**:
A person taking quizzes, signed in or not.
_Avoid_: user (ambiguous with admin), player, candidate, student

**Admin**:
The person who reviews, edits and approves questions and triggers generation.
_Avoid_: moderator, editor, owner

### Learner data

**Bookmark**:
A learner's mark on a question to find it again later.
_Avoid_: save, favorite, star

**Flag**:
A learner's report that a question looks wrong; it puts the question in front of the admin.
_Avoid_: report, dispute, complaint

**Note**:
A free-form text a learner keeps for themselves, optionally about a question or topic.
_Avoid_: memo, annotation, comment

### Reserved for later

**Author**:
Where a question or quiz originates: the system or a learner.
_Avoid_: creator, owner, source

**Visibility**:
Whether a question or quiz is free or premium.
_Avoid_: access level, plan, tier (tier means difficulty)

**Premium**:
A future paid plan that unlocks premium-visibility content.
_Avoid_: pro, paid tier, subscription
