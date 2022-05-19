# RFCs for XState and Stately tools

This [RFCs](https://en.wikipedia.org/wiki/Request_for_Comments) repository is a place to discuss changes to XState and Stately tools. A Request for Comments (RFC) is a design proposal formatted as a pull request (PR) proposal, including its design’s motivation, technical background, implementation, alternatives and drawbacks.

A considerable part of the value of an RFC is defining the problem clearly, collecting use cases, showing how others have solved a problem, etc. Coming up with a design is very iterative and only one part of the process. An RFC can provide tremendous value without the team accepting the design as described.

Minor changes may not need to go through the RFC process and can rely on issues and pull requests. Major changes should always go through the RFC process, where ‘major’ implies significant changes either to public interfaces or internal implementation details, particularly those that could be controversial or involve breaking changes.

Anyone can submit RFCs. The most common reasons the team rejects an RFC is that it has significant design gaps or flaws, does not work cohesively with all the other features, or does not fall into our view of the scope of XState or the Stately tools. 

However, getting merged is not the only success criteria for an RFC. Even when the design does not match the direction we’d like to take, we find RFC discussions very valuable for research and inspiration. Whenever we start work on a related area, we check the RFCs in that area and review the use cases and concerns the community members have posted. When you send an RFC, your primary goal should not necessarily be to get it merged into XState or the Stately tools as is, but to generate a valuable discussion with the community members. If your proposal later becomes accepted, that’s great. But even if it doesn’t, your work isn’t for nothing! The resulting discussion often informs the next proposal in the same problem space, whether the proposal comes from the community or the team.

Our RFC process is inspired by the process adopted by [Svelte](https://github.com/sveltejs/rfcs), [React](https://github.com/reactjs/rfcs), [Ember](https://github.com/emberjs/rfcs) and others. The process itself is subject to change, or may even be abandoned, as we gain experience.


## The process

![Statechart representing the RFC process detailed below.](https://user-images.githubusercontent.com/266663/169092679-3f64947d-31e6-4e04-bdba-baf408dfd832.png) [View this statechart in the Stately editor](https://stately.ai/registry/editor/share/eee673df-9f4d-4965-beb8-1a649676521a)

1. Fork the RFC repo at http://github.com/statelyai/rfcs
2. Copy `0000-template.md` to `text/0000-my-feature.md`, where ‘my-feature’ is descriptive. Don’t assign an RFC number yet.
3. Fill in the RFC. Put care into the details: **RFCs that do not present convincing motivation, demonstrate understanding of the impact of the design or are disingenuous about the drawbacks or alternatives tend to be poorly received.**
4. Submit a pull request. As a pull request, the RFC will receive design feedback from the larger community, and the author should be prepared to revise their pull request in response.
5. Build consensus and integrate feedback. RFCs with broad support are much more likely to make progress than those who don’t receive any comments.
6. Eventually, the team will decide whether the RFC is a candidate for inclusion in XState or the Stately tools. These candidate RFCs will enter a “final comment period.” The team will signal the beginning of this period with a comment and tag on the RFC’s pull request.
7. An RFC can be modified based on feedback from the team and community. Significant modifications may trigger a new final comment period.
The team may reject an RFC after public discussion has settled and comments have been made summarizing the rationale for rejection. A team member should then close the RFC’s associated pull request.
8. An RFC may be accepted at the close of its final comment period. A team member will merge the RFCs associated pull request, at which point the RFC will become “active.”


## The RFC life-cycle

Once an RFC becomes active, authors may implement it and submit the feature as a pull request to the XState repo or relevant Stately repo. Becoming “active” is not a guarantee and, in particular, still does not mean the team will ultimately merge the feature; it does mean that the core team has agreed to it in principle and are amenable to merging it.

The fact that an RFC has been accepted and is “active” does not imply a priority assigned to its implementation. Neither does it mean that anybody is currently working on it.

The team or community can modify active RFCs in follow-up PRs. We strive to write each RFC in a manner that will reflect the final design of the feature. Still, the nature of the process means that we cannot expect every merged RFC to completely reflect the end result at the time of the next major release. Therefore, we try to keep each RFC document somewhat in sync with the feature as planned, tracking such changes via follow-up pull requests to the document.


## Implementing an RFC

The author of an RFC is not obligated to implement it. Of course, the RFC author (like any other developer) is welcome to post an implementation for review after the team has accepted the RFC.

If you are interested in working on the implementation for an “active” RFC, but cannot determine if someone else is already working on it, feel free to ask by leaving a comment on the associated issue or in [the Stately Discord](https://discord.gg/xstate).


## Reviewing RFCs

We tend to do our thinking informally in the open when time allows. There are many community members relative to our small team who have many responsibilities. You can help ensure the team reviews your RFC in a timely manner by taking the time to consider the various details discussed in the template. It doesn’t scale to push the thinking onto a small number of core contributors. If reviewers raise an issue, don’t dismiss it as irrelevant; instead, provide additional examples or data and develop ways you could adapt the design in response. Sometimes answering a single question can be very time-consuming (such as setting up a benchmark), but discussions tend to stall out if concerns don’t get thoroughly addressed.
<<<<<<< HEAD


## Commenting on RFCs

Good feedback is an essential part of the RFC process. The RFC author has put a lot of time and effort into their design and should expect constructive feedback that helps improve their RFC. We absolutely welcome positive feedback, and if you don’t have a comment that contributes to the RFC’s improvement, we encourage you to show your support using a positive emoji (this also ensures the author doesn’t get overwhelmed by notifications!)

Helpful feedback includes:
- Your real-life use cases that are related to the RFC. Would the RFC solve your use case? Is your use case close to the RFC but would remain unsolved by the proposed design?
- Alternative designs or solutions that solve the same problems described in the RFC. If you have an alternative that may improve upon the RFC’s design, explain your reasoning while respecting the author’s chosen approach.
=======
>>>>>>> a0a5724478259a3ab1197c8a1700bcbb4dfcc0b1
