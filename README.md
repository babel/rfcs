# Babel RFCs

Many changes, including bug fixes and documentation improvements, can be
implemented and reviewed via the normal GitHub pull request workflow.

Some changes are "substantial", and we ask that these be put
through a design process and produce a consensus among the Babel team.

The "RFC" (request for comments) process is intended to provide a
consistent and controlled path for new features to enter the project.

## When you need to follow this process

> Like other projects, Babel is still actively developing this process.

You need to follow this process if you intend to make "substantial" changes to any part of Babel.
Some examples that would benefit from an RFC are:

- A new feature that creates new API surface area, and would require a feature flag if introduced.
- The removal of features that already shipped as part of the release channel.
- The introduction of new idiomatic usage or conventions, even if they do not include code changes to Babel itself.

The RFC process is a great opportunity to get more eyeballs on your proposal before it becomes released. Quite often, even proposals that seem "obvious" can be significantly improved once a wider group of interested people have a chance to weigh in.

Some changes do not require an RFC:

- Rephrasing, reorganizing or refactoring
- Addition or removal of warnings
- Additions that strictly improve objective, numerical quality criteria (speedup, better browser support)

## What the process is

In short, to get a major feature added, one usually first gets the RFC merged into the RFC repo as a markdown file. At that point the RFC is 'active' and may be implemented with the goal of eventual inclusion into Babel.

1. [Fork](https://github.com/babel/rfcs/fork) the RFC repo.
1. Copy `0000-template.md` to `text/0000-my-feature.md` (where 'my-feature' is descriptive. Don't assign an RFC number yet).
1. Fill in the RFC. Put care into the details: RFCs that do not present convincing motivation, demonstrate understanding of the impact of the design, or are disingenuous about the drawbacks or alternatives tend to be poorly-received.
1. Submit a pull request. As a pull request the RFC will receive design feedback from the larger community, and the author should be prepared to revise it in response.
1. Build consensus and integrate feedback. RFCs that have broad support are much more likely to make progress than those that don't receive any comments.
1. Eventually, the team will decide whether the RFC is a candidate for inclusion.
1. RFCs that are candidates for inclusion will enter a "final comment period" lasting 3 calendar days. The beginning of this period will be signaled with a comment and tag on the RFCs pull request.
1. An RFC can be modified based upon feedback from the team and community. Significant modifications may trigger a new final comment period.
1. An RFC may be rejected by the team after public discussion has settled and comments have been made summarizing the rationale for rejection. A member of the team should then close the RFCs associated pull request.
1. An RFC may be accepted at the close of its final comment period. A team member will merge the RFCs associated pull request, at which point the RFC will become 'active'.

## The RFC Lifecycle

Once an RFC is merged into this repo, then the authors may implement it and submit a pull request to the appropriate Babel repo without opening an issue. Note that the implementation still needs to be reviewed separate from the RFC, so you should expect more feedback and iteration. 

If the RFC authors choose not to implement the RFC, then the RFC may be implemented by anyone. There is no guarantee that RFCs not implemented by their author will be implemented by the Babel team.

Changes to the design during implementation should be reflected by updating the related RFC. The goal is to have RFCs to look back on to understand the motivation and design of shipped Babel features.

## Implementing an RFC

The author of an RFC is not obligated to implement it, but the RFC author (like any other developer) is welcome to post an
implementation for review after the RFC has been accepted.

If you are interested in working on the implementation for an 'active' RFC, but cannot determine if someone else is already working on it, feel free to ask (e.g. by leaving a comment on the associated issue).

## Reviewing RFCs

We try to make sure that any RFC that we accept is accepted at a team meeting. Every accepted feature should have a core team champion, who will represent the feature and its progress.

**Thanks to the [ESLint](https://github.com/eslint/rfcs) and [React](https://github.com/reactjs/rfcs) RFC process for inspiration.**
