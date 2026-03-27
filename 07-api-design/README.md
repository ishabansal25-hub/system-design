# API Design

> "A good API is not just easy to use but also hard to misuse." — Joshua Bloch

API design is one of those skills that separates good engineers from great ones. Anyone can build an endpoint that works. Building one that's intuitive, scalable, and doesn't make your consumers want to throw their laptops out the window? That takes thought.

---

## If you're prepping for interviews

I made a condensed cheatsheet: **[Interview Cheatsheet](./INTERVIEW-CHEATSHEET.md)**

It covers the key concepts you need for a 1-2 day review. REST principles, common patterns, and the kind of questions that actually come up.

---

## What's covered

| # | Topic | What it covers |
|---|-------|----------------|
| 1 | [What is API Design?](./01-what-is-api-design.md) | The fundamentals - what makes an API "good" |
| 2 | [REST Fundamentals](./02-rest-fundamentals.md) | Constraints, resources, statelessness |
| 3 | [URL Design](./03-url-design.md) | Naming conventions, hierarchies, query params |
| 4 | [HTTP Methods Deep Dive](./04-http-methods-deep-dive.md) | GET, POST, PUT, PATCH, DELETE - when to use what |
| 5 | [Request & Response Design](./05-request-response-design.md) | Payloads, headers, content negotiation |
| 6 | [Status Codes & Error Handling](./06-status-codes-and-errors.md) | The right code for the right situation |
| 7 | [Versioning Strategies](./07-versioning-strategies.md) | URL, header, query param - tradeoffs |
| 8 | [Authentication & Authorization](./08-authentication-authorization.md) | OAuth, JWT, API keys, scopes |
| 9 | [Rate Limiting & Throttling](./09-rate-limiting-throttling.md) | Protecting your API from abuse |
| 10 | [Pagination, Filtering & Sorting](./10-pagination-filtering-sorting.md) | Handling large datasets |
| 11 | [API Documentation](./11-api-documentation.md) | OpenAPI, examples, developer experience |
| 12 | [GraphQL Essentials](./12-graphql-essentials.md) | When REST isn't enough |
| 13 | [gRPC & Protocol Buffers](./13-grpc-protocol-buffers.md) | High-performance internal APIs |
| 14 | [API Gateway Patterns](./14-api-gateway-patterns.md) | Routing, aggregation, cross-cutting concerns |
| 15 | [Real-World API Examples](./15-real-world-api-examples.md) | How Stripe, Twilio, GitHub do it |

---

## Where to start

**If you're new to API design:**
Start with [What is API Design?](./01-what-is-api-design.md) and work through REST Fundamentals. These build the foundation everything else sits on.

**If you're building something:**
Jump to [URL Design](./03-url-design.md) and [HTTP Methods](./04-http-methods-deep-dive.md) for the practical stuff. Then [Error Handling](./06-status-codes-and-errors.md) because that's where most APIs fall apart.

**If you're choosing between REST/GraphQL/gRPC:**
Read [GraphQL Essentials](./12-graphql-essentials.md) and [gRPC](./13-grpc-protocol-buffers.md) to understand the tradeoffs. Spoiler: REST is still the right choice for most public APIs.

**If you have an interview coming up:**
The [Real-World Examples](./15-real-world-api-examples.md) file shows how top companies design their APIs. Interviewers love asking "how would you design an API for X" - seeing real patterns helps.

---

## Quick reference

For most cases, this works:

| Situation | What I'd do |
|-----------|-------------|
| Public API | REST + JSON + OAuth 2.0 |
| Internal microservices | gRPC or REST |
| Mobile app backend | REST or GraphQL |
| Real-time features | WebSockets + REST |
| High-throughput internal | gRPC with Protocol Buffers |
| Third-party integrations | REST + Webhooks |

---

## My take on API design

After building and consuming hundreds of APIs, here's what I think matters most:

1. **Consistency beats cleverness.** If you use `user_id` in one endpoint, don't use `userId` in another. Pick a convention and stick to it religiously.

2. **Error messages are part of your API.** "Something went wrong" is not helpful. "The email field must be a valid email address" is. Your errors should help developers fix their code.

3. **Versioning is not optional.** You will need to make breaking changes. Plan for it from day one.

4. **Documentation is not an afterthought.** If developers can't figure out how to use your API in 5 minutes, they'll find another one.

5. **Design for the consumer, not the database.** Your API shouldn't expose your internal data model. It should expose what makes sense for the people using it.

6. **Idempotency saves lives.** Network failures happen. If I retry a request, it shouldn't create duplicate orders or charge a credit card twice.

---

## The API design checklist

Before shipping any API, ask yourself:

- [ ] Can a developer understand what this endpoint does from the URL alone?
- [ ] Are the HTTP methods used correctly?
- [ ] Do error responses tell the developer what went wrong and how to fix it?
- [ ] Is the response structure consistent with other endpoints?
- [ ] Is there a way to version this API?
- [ ] Is authentication/authorization properly implemented?
- [ ] Are there rate limits to prevent abuse?
- [ ] Is the API documented?
- [ ] Have you tested it with real consumers?

---

[Back to main notes](../README.md)