#!/usr/bin/env bash
# .github/scripts/gen-forms.sh — (re)build the request issue forms from THIS repo's config.json.
#
# Per-school model: a repo carries exactly ONE config.json (its own school), so only that school's
# course forms are ever generated — cross-school pollution is impossible. Wipes all request-*.yml,
# then writes one request-<course>.yml per course (dropdown = that course's assignment keys).
# register/myrepos/config.yml are course-agnostic and left untouched.
#
# THE single generator — run by both:
#   - ops/config.sh                     (local, after editing a school's config.json)
#   - .github/workflows/gen-forms.yml   (server, on config.json push)
# One implementation → local and server never drift.
set -euo pipefail

ROOT="$(git rev-parse --show-toplevel 2>/dev/null || pwd)"
CONFIG="$ROOT/config.json"
TPL="$ROOT/.github/ISSUE_TEMPLATE"

# a config-less repo (the template itself) just has no request forms
if [ ! -f "$CONFIG" ]; then
  echo "gen-forms: no config.json here — clearing request forms (template/uninitialized repo)"
  rm -f "$TPL"/request-*.yml
  exit 0
fi
jq -e . "$CONFIG" >/dev/null || { echo "gen-forms: config.json is not valid JSON" >&2; exit 1; }

rm -f "$TPL"/request-*.yml
jq -r '.courses | keys_unsorted[]' "$CONFIG" | while read -r course; do
  up="$(printf '%s' "$course" | tr '[:lower:]' '[:upper:]')"
  out="$TPL/request-$course.yml"
  {
    printf 'name: Request a %s repo\n' "$up"
    printf 'description: Get your personal repo for a %s assignment. Register first if you have not.\n' "$up"
    printf 'title: "request-%s: "\n' "$course"
    printf 'body:\n'
    printf '  - type: markdown\n    attributes:\n      value: |\n'
    printf '        ### Request your %s assignment repo\n' "$up"
    printf '        Pick the assignment. Your GitHub username is captured automatically.\n'
    printf '        Not registered yet? Open the **Register** form first.\n'
    printf '  - type: dropdown\n    id: assignment\n    attributes:\n'
    printf '      label: Assignment\n'
    printf '      description: (Auto-generated from config.json — do not hand-edit.)\n'
    printf '      options:\n'
    # sort -V = natural order so the dropdown is A51<A52<...<A59<A510<A511 (not lexical A510<A52),
    # regardless of the order assignments were added to config.json.
    jq -r --arg c "$course" '.courses[$c].assignments | keys_unsorted[]' "$CONFIG" | sort -V | sed 's/^/        - /'
    printf '    validations:\n      required: true\n'
    # Access-code box — emitted ONLY for a course that has at least one code-gated assignment
    # ("requires_code": true, i.e. a quiz/exam). One form serves the whole course, so the box also
    # shows on regular-assignment requests: request.yml IGNORES it unless the picked assignment is
    # code-gated, and the label says so — a student must never be blocked for a stray value here.
    if jq -e --arg c "$course" \
         '[.courses[$c].assignments[] | select(type=="object" and (.requires_code // false))] | length > 0' \
         "$CONFIG" >/dev/null; then
      printf '  - type: input\n    id: access_code\n    attributes:\n'
      printf '      label: Access code\n'
      printf '      description: Quizzes and exams ONLY — leave this blank for regular assignments. The code is inside the quiz on Canvas.\n'
      printf '    validations:\n      required: false\n'
    fi
  } > "$out"
  n="$(jq -r --arg c "$course" '.courses[$c].assignments | length' "$CONFIG")"
  printf 'gen-forms: generated %s  (%s assignments)\n' "request-$course.yml" "$n"
done

echo "gen-forms: done."
