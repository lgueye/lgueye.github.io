---
layout: post
title: Platform developer: episode 0
---

The main business of my previous company was to provide a `security system` for `elderly people`.

Imagine your aunt which is quite old and finds herself alone in her house because her husband was taken from her.
With time, she might show early signs of loss in motricity or even dementia.
But before she becomes a real danger for herself she wants to stay home, where she feels safe.

This describes a standard use case (not limited to) for the security system my previous company has been building.

The system is split in 2 distinct parts:

- the hardware: sensors (dry contact, infra red, etc) installed at the beneficiary living place communicating via RF (radio frequency) with a central panel. Sensors basically witness the activity of the elderly (door opened, door closed, bed on bed off, movement in living-room/bathroom, help pressed, etc) and send it to the main panel which forwards it to the platform via HTTP. In case of emergency, for clients willing to subscribe to a call center, the call center will call back the panel to confirm the situation. 

- the software: receives events from various hardware living places (installations). Events are stored, normalized, analyzed over time. The analysis yields activity reports and graphs but more importantly notifications to call centers or any configured recipient in case of emergency situations like fall of bed, wandering at night, help call, etc.

I could go through numerous use cases and clarifications about the business because it is so motivating but I'll stick to the technical part, specifically the software platform which handles the events.

Ultimately, the platform needs to detect `emergency situations`. That requirement leaves very few rooms for approximation. After all we're dealing with people safety.
 
The worst case scenario for an emergency system is to miss an emergency situation which can happen for 2 reasons:

- wrong analysis of the events (false negative): not being able to detect that your aunt has fallen at night, broke her leg and lied there for some time because she could not call anybody for help. This situation is very difficult to detect for obvious reasons (usually seniors do not wear their panic button at night) yet not so unlikely to happen. Falls into the `correctness` of the system.

- availability of the system: if any of the components of the whole solution is unavailable, the system will miss to notify a potential emergency situation. Falls into the `availability` of the system.

By now, you should have a pretty good picture of the 2 critical constraints of the platform.

Addressing the correctness is the responsibility of the software: we simply can't save on automated tests. The software design was driven upfront by tests scenarios.

One of the trickiest part was to be able to control the time which had been made so much easier with the java 8 Time API.

The other non trivial part was related to the asynchronous nature of the system: a lot of communications between backends involve producer backends sending messages to queues then consumers backends reacting on such notifications. We needed to assert some expected state of the system `within X ms`. 

Addressing the availability of the system was the responsibility of the platform. But I would nuance this claim because availability implicitly forces your software to support `distribution` (cross data-center distribution or single data-center distribution) and everybody knows that distributed comes with its own sets of challenges, the most difficult one being handling state distribution properly (ie transactions atomicity).
 
Once you go distributed (because you need to tackle availability) you can't go half way. Your entire stack needs to be distributed. This is not entirely true: the business-critical components need to be distributed. Others could technically be unavailable without any client noticing the difference. That's why we conducted an analysis to assert each component business criticity which helped us focusing our efforts. Find below the summary:
 
| Component/Feature| Description                                                                                                                                              | Business critical  | Platform  provided |
| -------------    | ---------------------------------------------------------------------------------------------------------------------------------------------------------|:------------------:|:------------------:| 
| load balancer    | this is the main entry point of the platform. Accepts all http requests an routes them to the relevant components. Is also bound to the main http domain | yes                | yes                |
| API gateway      | proxy to backends, takes care of edge services like authentication, correlation, rate limiting                                                           | yes                | yes                |
| discovery        | allows one services to locate each others without going through the gateway and without knowing physical address (host:port) before hand                 | yes                | yes                |
| frontend         | static web assets (html, js, css)                                                                                                                        | yes                | yes                |
| metrics/alerting | foundational component which collects nodes technical or business metrics in a central DB. Help diagnose the system then act on them (alerting)          | no                 | yes                |
| logs             | foundational component which helps investigate issues                                                                                                    | no                 | yes                |
| tracing          | foundational component which helps investigate issues by providing correlation                                                                           | no                 | yes                |
| broker           | communication hub between services/backends                                                                                                              | yes                | yes                |
| cache            | offloads the store                                                                                                                                       | yes                | yes                |
| store            | persists all system events                                                                                                                               | yes                | yes                |
| scheduler        | schedules processings/computations in the future                                                                                                         | yes                | yes                |
| ias (identity and authrorizations service) | checks identities, provides access, revokes them                                                                                                         | yes                | yes                |
| backend/service  | business services such as ingestion (one per hardware provider), normalization, care engine, maintenance engine, notification engine, i18n, etc          | yes                | no                 |

All `platform provided` components form a cohesive set of technical services used to build business solutions. They should be implemented with ease of use in mind. Their main consumers are the Product/Business components.

The `business critical` will require the component to be `distributed`. Others can be but are not forced to.

This post series will focus on the platform provided components, why they're used in the first place and which challenges we could face when trying to implement them.

You don't provide a dozen of platform components in your company without industrialization practices. 

You need a great deal of focus, discipline as well as a dose of empathy for the consumers. In the [next episode](/2019-10-27-platform-developer-episode-1) I'll share the steps that we followed to develop our most critical components. Stay tuned !

