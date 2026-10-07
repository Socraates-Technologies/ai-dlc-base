# Writing Rule — ASD-STE100

Write all prose in Simplified Technical English, as ASD-STE100 Issue 8 (2021) defines it.

ASD-STE100 is a controlled language. The aerospace industry wrote it so that a technician cannot misread a
maintenance instruction. It has two parts:

- **Part 1, the writing rules.** 53 rules in nine sections: words, noun phrases, verbs, sentences, procedures,
  descriptive writing, safety instructions, punctuation and word counts, and writing practices.
- **Part 2, the dictionary.** About 900 approved words, each with one approved meaning, and the unapproved words
  with the approved word to use in their place.

The specification is free. Get it from <https://www.asd-ste100.org/>. Do not copy the dictionary into a repo. It
is copyrighted, and a copy goes stale.

---

## Where the rule applies

The rule applies to every sentence the project writes for a person to read:

- documentation: the master rule file, the framework's files, unit files, bolt files, retros, ADRs and edge cases
- commit messages and pull request descriptions
- text a user of the product reads: labels, messages, notifications and emails
- text an engineer reads from a tool: a script's `--help`, its error lines, and the message part of a log line
- code comments
- what the AI says to the engineer in a session

The rule does not apply to:

- code: identifiers, keys, enum values, SQL. The domain glossary's storage names govern those
- quoted words. The engineer's words stay as they said them. A quoted log line, error or standard stays as it is.
  A user's own text is theirs
- text that already exists. Do not rewrite a file to make it comply. When you edit a paragraph for another
  reason, rewrite that paragraph

---

## The rules that matter most

The specification has the detail. These are the rules that project prose breaks most often.

**Words (STE section 1).**

- Use a word only in its approved meaning. One word has one meaning. STE's own example: `follow` means _come
  after_, so write "obey the procedure", not "follow the procedure".
- Use the domain glossary's terms as Technical Names. STE permits a technical name outside the dictionary when
  the domain defines it. `guidelines/domain-glossary.md` is the project's list, and its banned synonyms are not
  permitted.
- A word from a tool's own vocabulary is a Technical Name or a Technical Verb when the tool's documentation
  defines it: `commit`, `rebase`, `deploy`, `migrate`, `render`. Use the tool's spelling.
- Do not use slang, idiom or metaphor. "The gate is red" is permitted: it is this framework's technical term.

**Noun phrases (section 2).**

- A noun cluster has three nouns at most. "Customer order status notification email" is five. Write "the email
  that tells a customer the status of an order".
- Do not drop articles. Write "the server", not "server".

**Verbs (section 3).**

- Use the active voice. Say who does the thing: "the server sends the email", not "the email is sent".
- Use the simple present, the simple past or the future. Do not use an `-ing` form as a verb.
- Use the imperative for an instruction.

**Sentences (section 4) and word counts (section 8).**

- One topic per sentence.
- A sentence in a procedure has 20 words or fewer. A sentence in a description has 25 words or fewer.
- A paragraph has six sentences or fewer, and its first sentence states its topic.
- Do not join two sentences with a dash or a semicolon. Start a new sentence. (A framework rule, in the spirit
  of section 8.)
- Use a vertical list when a sentence would hold more than three items.

**Procedures (section 5).**

- One instruction per sentence.
- Write the condition before the instruction: "If the gate is red, stop."
- Write a prohibition as "Do not …".

**Descriptive writing (section 6).** Start a new paragraph for a new topic.

**Safety instructions (section 7).** A warning or a caution starts with a command, then gives the reason. "Do not
run the migration against production. It drops the table."

---

## How the rule meets the framework's habits

- **A rule's reason stays.** This framework records the incident behind each rule, with its date. STE changes the
  sentences, not the practice. Write the incident as short sentences, in a paragraph of six or fewer. Put a second
  incident in its own paragraph.
- **The glossary wins on nouns.** When STE's dictionary and the domain glossary disagree on a term, the glossary
  wins, because a synonym in a column name is a migration.
- **The reader may be a child, a patient or a person in a hurry.** STE's short sentences and one-meaning words
  serve them. Keep the tone warm within the rules. STE forbids slang, not kindness.

---

## Checks

- `skills/review-checklist.md` Section 9 asks, before any output, whether the new prose obeys this rule.
- There is no mechanical gate. A word counter cannot tell a procedure from a description, and the dictionary is
  not in the repo. A prose rule needs a reader, and a hook needs none. If a retro finds this rule broken twice,
  the retro proposes a gate. The first candidate is a sentence-length check over the lines a commit adds.
