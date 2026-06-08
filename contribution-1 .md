# Contribution #1: consider giving more context for safe schema change based on usage

**Contribution Number:** 1
**Student:** Jenisha Subedi
**Issue:** https://github.com/graphql-hive/console/issues/6892
**Status:** Phase I In Progress

---

## Why I Chose This Issue

I chose this issue because it sits at the intersection of API design and frontend UI, which matches my skills in both backend and frontend development. The problem is clearly scoped — surfacing client inclusion context in the schema check view — making it a great first contribution to a real-world GraphQL tooling platform.

I'm also excited to learn more about how GraphQL schema validation and client inclusion lists work in practice. Contributing to graphql-hive will give me hands-on experience with a production-grade open source tool used by real engineering teams, which is exactly the kind of work I want on my resume.

---

## Understanding the Issue

### Problem Description

When a schema change is marked as "safe" because of the client inclusion list, the schema version/schema check view does not explain why it is considered safe. Developers are left without context about what made the change safe.

### Expected Behavior

The schema version/schema check view should display information about which clients are in the inclusion list that caused the change to be classified as safe, giving developers full visibility into the safety determination.

### Current Behavior

The view shows the change is safe but provides no explanation or context about the client inclusion list that determined it.

### Affected Components

- Schema version/schema check view (dashboard frontend)
- Client inclusion list API logic

---

## Reproduction Process

### Environment Setup

[To be filled in Phase II]

### Steps to Reproduce

[To be filled in Phase II]

### Reproduction Evidence

[To be filled in Phase II]

---

## Solution Approach

[To be filled in Phase II]

---

## Testing Strategy

[To be filled in Phase II]

---

## Implementation Notes

[To be filled in Phase II onwards]

---

## Pull Request

[To be filled in Phase III]

---

## Learnings & Reflections

[To be filled upon completion]

---

## Resources Used

- https://github.com/graphql-hive/console/issues/6892
- https://github.com/graphql-hive/console/blob/main/docs/CONTRIBUTING.md
