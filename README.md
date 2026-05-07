# Insurance Comparison & Broker Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open-source, AI-native platform that lets independent insurance agents and brokers quote, compare, and bind policies across multiple carriers from a single interface.

Insurance agencies today are locked into expensive, proprietary software ecosystems just to do the basic work of comparing carrier quotes. Enterprise platforms like Applied Epic and Vertafore command USD 50k-500k+ annual contracts, while even mid-market tools start at USD 300/month. This project delivers an affordable, open-source alternative that combines multi-carrier comparative quoting with modern agency management -- powered by AI to eliminate the manual data re-keying, carrier appetite guesswork, and coverage gap blind spots that plague the current tooling landscape.

---

## Why Insurance Comparison & Broker Platform?

- **Prohibitive pricing locks out small agencies.** Enterprise AMS platforms cost six figures annually, and even mid-tier raters start at USD 300/month. Solo agents and small agencies are priced out of efficient comparative quoting, despite being a significant share of the market.
- **Proprietary lock-in across the stack.** Applied Systems owns both the dominant AMS (Applied Epic) and the IVANS carrier connectivity network, plus EZLynx and Tarmika. This vertical integration creates switching costs and limits interoperability for agencies not in the Applied ecosystem.
- **Dated user experiences slow down agents.** Vertafore AMS360 and Applied Epic are widely criticised for ageing UIs, steep learning curves, and long onboarding cycles -- problems that compound for agencies with staff turnover.
- **Commercial lines quoting remains fragmented.** Personal lines raters cover 300+ carriers, but commercial lines tools like Tarmika support only 30+ carriers and cap out at small commercial risks. Mid-market commercial placement still requires manual broker legwork.
- **No open-source alternative exists.** Every analysed solution in this space is proprietary commercial software. There is no open-source AMS, comparative rater, or quote-to-bind platform available today.

---

## Key Features

### Multi-Carrier Comparative Quoting

- Single-entry quoting across personal and small commercial lines with real-time carrier API integrations
- Side-by-side quote comparison with coverage-level detail and standardised terminology
- Bindable quote responses enabling full quote-to-bind digital workflow without leaving the platform
- NAICS code mapping and risk classification for commercial lines routing

### Agency Management

- Client, policy, and document lifecycle management with role-based access for producers, CSRs, and principals
- Commission tracking and basic premium accounting
- Renewal and expiration management with automated alerts and re-marketing workflows
- ACORD-compliant data model for risk, policy, and carrier interchange

### Consumer-Facing Distribution

- White-label self-service quoting portal for end consumers
- Embedded insurance APIs for non-insurance platforms to offer coverage at point of need
- Brandable quote pages with agency customisation
- Progressive question flows that reduce data entry to essential fields

### AI-Powered Automation

- Document extraction from applications, loss runs, and certificates with automated ACORD form population
- Carrier appetite pre-screening matching risk characteristics against appetite rules before submission
- Coverage gap detection using NLP analysis of policy wordings against industry benchmarks
- Conversational quoting interface allowing agents or consumers to describe risks in natural language

### Workflow and Integration

- API-first architecture enabling headless deployment and custom UIs
- Real-time carrier connectivity via direct API integrations
- No-code workflow builder for custom quoting flows
- Webhook support for event-driven integrations with CRMs and external systems

---

## AI-Native Advantage

Current insurance platforms bolt AI onto legacy architectures as an afterthought. This project treats AI as foundational infrastructure. Machine learning models pre-screen risks against carrier appetite rules before submission, reducing declination rates and speeding placement for complex commercial accounts. NLP-driven document extraction eliminates the manual re-keying between client documents and ACORD forms that consumes hours of agent time daily. Predictive models identify accounts likely to shop at renewal and trigger proactive retention outreach, while coverage gap analysis flags exclusions and under-insurance that human review routinely misses.

---

## Tech Stack & Deployment

The platform targets self-hosted and cloud deployment modes, with a modular architecture allowing agencies to adopt quoting, AMS, and distribution components independently or as a combined suite. The data model is built on ACORD standards for carrier interchange compatibility. Carrier integrations use direct RESTful APIs where available, aligned with the SEMCI (Single Entry Multiple Company Interface) approach to standardised comparative rating data exchange. The platform must comply with PCI DSS for premium payment processing and GDPR/CCPA/state insurance privacy laws for policyholder data handling.

---

## Market Context

The global insurance brokerage market is projected to reach USD 10.16 trillion in revenue in 2026, with the insurtech segment valued at USD 23.54 billion and forecast to reach USD 132.71 billion by 2034 at a ~24% CAGR (Business Research Insights, 2026). The insurance software market specifically sits at USD 15.03 billion in 2026, forecast to reach USD 20.41 billion by 2031 (Mordor Intelligence, 2025). Primary buyers are independent agents and brokers seeking comparative quoting efficiency, MGAs building digital distribution channels, and insurtechs building embedded insurance products. Traditional agents still hold 41% of insurtech distribution but face growing disintermediation pressure from direct digital channels.

---

## Project Status

> This project is in the **research and specification phase**.
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
