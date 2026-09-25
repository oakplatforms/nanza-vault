---
tags: [strategy, onboarding, go-to-market]
status: draft
---

## 1. Nanza in one page

**What it is.** Nanza is a **social commerce platform for collectibles**, starting with trading card games. People already buy, sell, and trade inside Discord servers and Facebook groups with nothing behind it: no record, no structure, no data. Nanza gives those same people the ability to keep doing exactly what they do, with real organization around it. Groups, listings, bids, collections, posts, and sharing, all anchored to canonical card data.

**What kind of company.** Think of Nanza as a **data and usage company**, closer to Discord than to a marketplace. The product is free to use, members pay to raise usage caps, and the businesses in the ecosystem pay to reach communities and understand them. **There is no commission on trades.** The people we serve trade with friends precisely because it's free, and a marketplace cut is the one thing they'll never accept.

**Who pays: a hybrid.** No single group is "the customer." Members, IP holders, and adjacent businesses each pay for something different.

1. **Members (freemium, Discord Nitro style).** Free to join, list, bid, chat, and trade. Usage caps on creation (listings, bids, messages, collections) and on scanning and AI features; pay roughly **$10/month** to raise caps or add capacity. Still dramatically better than commission: a few dollars a month instead of a cut of every trade. This is a caps-and-credits model, not subscription tiers that gate features.
2. **IP holders / licensors (partners).** The company that owns the game (for Flesh and Blood, Legend Story Studios). They partner with us and pay to **reach and understand their community**: sponsored content and ad generation into their own community on Nanza, canonical data distribution, and the insight layer that shows how their product actually moves. Retainer plus usage plus custom work (see below).
3. **Adjacent businesses and, later, ads and insights.** Publishers, distributors, retailers, and other trading-card and collectible companies promoting their games to communities. Over time the platform's unique insight data becomes its own revenue line.

**What we don't do.** No commission. No random third-party ads: promotion on Nanza comes from licensors and adjacent trading-card businesses speaking to their own communities, never from unrelated advertisers. No engagement algorithm built to keep people scrolling. Private groups and chats stay private.

**Why that works.** Marketplaces looked at private-community trading and concluded there was no business in it because traders won't pay a cut. We asked a different question: *who needs to see this, and what would people pay a little for?* The IP holder gains from visibility. Members will pay a few dollars for capacity and structure in a place that's theirs. Both beat a commission.


---

## 2. The market

Three concentric markets. We live in the innermost one today and the outermost one describes the category we're actually building in.

| Market | Size | Notes |
|---|---|---|
| **Trading card games (TCGs)** | **~$11B–$15B annually** | Driven heavily by character and fantasy gaming cards. Pokémon and Magic keep growing; Flesh and Blood is emerging fast. This is our entry, not our boundary. |
| **Collectibles overall** | **~$320B–$460B globally** | Trading cards, comics, toys, anime, retro gaming. The rule is simple: *if it's collectible, it fits*, because the model is re-commerce of ownable, indexable things. |
| **Social commerce (as an industry)** | **~$1.9T–$2.6T globally in 2026** (from ~$900B on pure in-app transactions up to $2.1T–$2.6T on broader social-driven GMV). Forecasts of $7.5T to nearly $18T by the early 2030s at a 28–37% CAGR. | The category Nanza is actually in. Reference point already in our material: **81% of social commerce transactions on TikTok Shop are under $30.** Nanza's volume thesis lives in low-ticket social commerce, not the high-end authenticated market. *(Verify the 81% source before citing externally.)* |


**How to use these numbers.** The TCG figure sizes the entry. The collectibles figure sizes where the IP-holder partner model extends (thousands of IP holders across cards, comics, toys, apparel). The social commerce figure frames the *behavior* we're structuring: small-ticket, trust-based trades between people who know each other.

**Scope decisions worth knowing.**
- We are deliberately *not* going after the high end. A $22,000 collectible belongs with authentication companies and brand-protective marketplaces.
- We are not "disrupting" anything. Nanza is platform-agnostic and plugs into Discord, Shopify, YouTube, and third-party marketplaces. Nothing becomes obsolete. Don't pitch it as disruption.

---

## 3. The games

We think about the TCG landscape in two tiers


### Tier A — Founder-led, social-first, and newcomers

Small studios, often one to five people, usually Kickstarter or Discord-native. They are the IP holders most likely to want a *full-loop* integration because they have no other way to get it, and a deep integration with a small partner can be worth more to us than a shallow one with a big one.

Gino scored this tier in the **TCG Partner Prioritization** report (25 Aug 2026). Scores below are from that report; ask Gino for the full PDF.

| Game | Score | Decision | What it is |
|---|---|---|---|
| **Elestrals** | 94 | Priority 1, flagship pilot | Elemental creature TCG. 7,530-backer Kickstarter (~$1.47M); "Elestrals Beyond" digital sandbox launched 2026. The founder is also a content creator with a large digital footprint. Already has first-party tools, so a generic database pitch will fail; the pitch is a branded social layer around reveals, decks, events, trades. |
| **Alpha Clash** | 89 | Priority 2, organized-play growth pilot | Original-IP superhero TCG. Seventh core set (Ancient Alphas) Feb 2026; globally expanding organized play, online leagues, worlds in 2026. Nanza complements tournament admin rather than replacing it. |
| **NeuroScape** | 81 | Priority 3, launch-window partnership | Cyberpunk TCG. Asmodee U.S. distribution announced April 2026; 255-card Genesis set with serialized chase cards; unlimited release Aug 28, 2026; ~$570K Kickstarter. Launch window makes onboarding urgent and measurable. |
| **Echoes of Astra** | 81 | Priority 4, fulfillment-to-retail bridge | Parhelion Studio, ~$821K from 1,965 backers; retail launch (Imperion) coming, explicit LGS ambitions. Confirm fulfillment health before heavy outreach. |
| **Cryptid Camp** | 79 | Promote to strategic pilot | Traditional-art cryptid TCG. 2025 Kickstarter ~$81K / 224 backers; Discord, subreddit, digital play, 2026 expansion cadence. Small team, strong social collecting behavior, good founder access. |
| **SolForge Fusion** (Stone Blade) | 78 | Strategic, tightly scoped | Founder-led hybrid physical/digital game with worldwide events. Narrower need gap because they already have digital play and event infra. Only invest if they name a problem their stack doesn't solve. |
| **Neverrift** | 75 | Qualification sprint | Grid-based tactical TCG, Gate to the Games. Live storefront, rules, Discord. Audience scale and operator authority unproven. |
| **Here Be Monsters** | 74 | Post-fulfillment pilot | Pirate TCG / expandable hybrid. 2026 Kickstarter ~$119K / 632 backers, Team Covenant wholesale. Wait for fulfillment. |
| **Shard Bugs** | 69 | Promote to qualification target | Origins Awards finalist; official card DB, deck builder, store locator, organized play; Aug 28, 2026 release. Audience scale is the open question. |
| **Hubworld: Aidalon** (Earthborne Games) | 68 | Community test, not collector pilot | Values-led studio with huge crowdfunding support (also makes Earthborne Rangers). Explicitly *non-collectible*, so the angle is groups, play, events, not singles. |
| **The Caster Chronicles** | 66 | Revival qualification target | Speedrobo Games holds the English license (2025); 2026 Kickstarter ~$45K / 193 backers. Small, license-dependent. |
| **Divine Cross** | 63 | Market-entry research only | Japan-led multi-IP game; strong product cadence but licensing and localization are structurally hard. |
| **Planarist TCG** | 58 | Low-cost founder test | Rules, Tabletop Simulator, weekly events, store portal. Scale unproven. |
| **Quantum TCG** | 38 | Cautious outreach / compliance review | Limited Genesis print run, prize-card mechanics, crypto framing. Trust and compliance risk. |
| **Castlebound TCG** | 31 | Do not prioritize | Live storefront and archive; no evidenced community, team, or collector market. Skylar's note: looks AI-generated. |
| **Akora** | 18 | Hold, validate operator status | Latest official news is early 2024 and public reports say operations closed. Diligence note only; no pitch. |
| **MetaZoo** | — | Not scored | Cryptid TCG, 2021 boom. May be defunct; verify before any outreach. |
| **The Journey TCG** | — | Not scored | Fantasy TCG where you build a deck of heroes and race to complete your journey across new terrains before your opponent. Small indie; sells direct via Shopify. Little public evidence of scale. |
| **Primal TCG** | — | Not scored | Miami-based Dattebayo Labs (Claudio Atilano). Anime-styled fantasy TCG (pirates, dragons, plague doctors); first base set April 2024, now on Set 4 (New Genesis); Kickstarter-funded. Founder-led and social-native. |
| **Plunderlust** | — | Not scored | Creature Curation (same studio as Sqwürmish). Sci-fi *expandable* card game in the "Dying Universe"; Kickstarter funded in 66 minutes (October); solo and competitive play; carried by hobby retailers. Expandable, not collectible, so the angle is play groups. |
| **Sorcery: Contested Realm** | — | Not scored | Erik's Curiosa. Painted-art fantasy TCG, Kickstarter-born, premium positioning. Dedicated collector community. |
| **Gudnak** | — | Not scored | Chaotic Great Games. Tactical *expandable* (non-collectible) card game with square cards and four factions, launched via Kickstarter in late 2023 (940 backers, ~$102K, a record for square-card expandable games) with follow-on expansion campaigns and Team Covenant carrying it. Expandable rather than collectible, so the Nanza angle is groups and play, not singles. |
| **Warlord: Saga of the Storm** | 78 | Revival community | Kingswood Games (official AEG licensee). Legacy IP revival. Relaunch Kickstarter raised ~$598K from 1,743 backers, still posting updates July 2026. Thousands of historical cards, existing collectors. Scored 78 in Gino's model as a "revival community" play; AEG license dependency is the main constraint. |

A fuller landscape list (with links and notes on every game we've looked at, including the majors) is in Appendix A.

### Tier B — The big boys, multi national, global licensing

Global IP holders and publishers. Honest position from our own strategy doc: **most of these probably don't need us** — they have the resources to build this internally. The one path where they become plausible is *trying the model without building or owning it*: Nanza as an arm's-length service.

Listed by **licensor**, with the game(s) they own or publish. Where the IP owner and the publisher differ, both are named.

| Licensor | Game(s) | What it is |
|---|---|---|
| **Legend Story Studios** (NZ) | Flesh and Blood | Hero-based competitive TCG launched 2019 with a strong organized-play culture and fast growth. Mid-sized, global distribution, no in-house path to a full-loop solution. **Our live game and first target partner** (Section 4). |
| **Riot Games** (IP) / **UVS Games** (publisher) | Riftbound | The League of Legends TCG (2025). Massive IP on a brand-new card ecosystem; Riot's first physical TCG. UVS Games also runs UniVersus, a long-running licensed fighting TCG (My Hero Academia, Godzilla, Critical Role). |
| **The Pokémon Company** | Pokémon TCG | The largest TCG in the world, ~12% of the global TCG market on its own. Has the resources to build anything internally; plausible only on the arm's-length service path. |
| **Hasbro / Wizards of the Coast** | Magic: The Gathering | The original TCG (1993) and the deepest competitive and collector ecosystem in the category. Same internal-build caveat as Pokémon. |
| **Konami** | Yu-Gi-Oh! | One of the three global anime TCG pillars; decades of product, organized play, and a huge secondary market. |
| **Disney** (IP) / **Ravensburger** (publisher) | Disney Lorcana | Launched 2023 and became the biggest new TCG of the decade; Disney IP across films and characters. |
| **Bandai** | One Piece Card Game · Digimon Card Game · Dragon Ball Super Card Game · Union Arena · Gundam Card Game · Naruto Card Game · Battle Spirits | The busiest TCG publisher in the world right now. One Piece is where Tahir first saw the problem Nanza solves; Gundam (2025) and Union Arena are the newest lines. |
| **Bushiroad** | Cardfight!! Vanguard · Weiss Schwarz · Shadowverse: Evolve · hololive OFFICIAL CARD GAME · Godzilla Card Game (with Toho) | Japan's other major TCG house, strong in anime and VTuber licensing. Not on the original list; added because they are a clear fit for the multi-IP licensor pitch. |
| **Square Enix** | Final Fantasy TCG | Long-running licensed TCG with organized play in the West. Not on the original list; added for the same reason. |
| **Asmodee** | Star Wars: Unlimited (Fantasy Flight) · Marvel Champions (LCG) · Arkham Horror (LCG) · distributes NeuroScape in the U.S. | The largest tabletop publisher/distributor group. Star Wars: Unlimited (2024) is a top-five TCG; Marvel Champions and Arkham are living card games, not collectible, so the angle there is play groups. |
| **Marvel** (IP) | licensed to Upper Deck (Legendary, Vs. System 2PCG), Asmodee (Marvel Champions), and digital (Marvel Snap) | Marvel licenses rather than publishes card games. The pitch goes to the licensee for product and to Marvel for community insight across licensees. |
| **Upper Deck** | Marvel Legendary · Vs. System 2PCG · Marvel entertainment trading cards | Sports-card giant with a games arm; Marvel is their main card-game IP. Organized play program exists for both games. |
| **WeirdCo** | Cyberpunk TCG (CD Projekt Red IP) | Physical-only TCG built on Cyberpunk 2077 and Edgerunners. **Most successful crowdfunding campaign in gaming history:** 50,773 backers, $28.4M; 3,000+ retail stores signed up. Organized play on Melee.gg. New company, huge launch cohort. |
| **Skybound** | Overpower TCG | Skybound (The Walking Dead, Invincible) is a legit studio that works with real partners; Overpower is their new TCG. |
| **Pudgy Penguins** (IP) / **Orange Cap Games** (publisher) | Vibes | Pudgy Penguins is a web-3-native IP that crossed into mainstream toys; Vibes is its TCG. |
| **Archon Studio** | World of Tanks: The Card Game (with Wargaming) · Heroes of Might & Magic: The Card Game | Polish publisher with licensed card games on big video-game IP; World of Tanks got a 151-card new wave in 2026, HoMM funded on Gamefound. |
| **Ghost Galaxy** | KeyForge · Echo of Omens · Lord of the Rings card and board games | Founded by Fantasy Flight founder Christian Petersen; bought KeyForge from Asmodee. KeyForge's procedurally generated unique decks make it an unusual data story. |


---

## 4. Where Nanza is today

**Live product.**
- **Mobile app** (iOS/Android) is the flagship: groups, listings, bids, bulk lots, collections, posts, tags, chat, sharing.
- **Web app** (nanza.app) is currently a **public distribution surface**: every listing, bid, collection, product, lot, group, profile, post, and tag has a shareable link that lands on a web page and routes app-less mobile visitors to the store. Plus the marketing site.
- **Closed-feed model.** No public feed of user posts. Members see posts only from mutually accepted connections and from vetted **affiliates**. Signed-out visitors see affiliate content only. This is a trust-and-safety decision, not a limitation.
- **0% commission.** Sellers pay only the card-processing fee (3% + 50¢). Stripe handles payments and payouts. Member caps/credits (the freemium layer) are designed but not yet built; a membership-tier prototype was tried and reverted in June 2026 in favor of caps.
- **Nanza AI**, also known as **Cortex**, is the insight engine that makes an IP holder's canonical data smarter over time.

**Flesh and Blood is live today.** We chose one IP and went deep rather than launching as a generic category marketplace. Collectors and players love the specific IP they care about. The product, the data, and the community work are all Flesh and Blood today.

**Legend Story Studios (LSS).** The relationship is healthy and they are giving us access to their community as we build. **They are not yet a signed partner.** Converting them is milestone #1 for the company. Everything in our strategy is theory until money moves: a signed IP-holder partner, and members paying to raise their caps.

**The near-term agenda, in Skylar's words:** the next six to twelve months and roughly five signed IP-holder partners. Four or five partners with real deals across at least two collectible categories, plus a growing base of paying members, proves repeatability and sustains the team outright.

**We are not currently venture-backed.** Investor relations is one of the three workstreams for this engagement. See Section 6.3.

---

## 5. Go-to-market: the sequence

The model is the opposite of viral. You don't download Nanza cold; you join because the people in your community are already using it. So growth runs through the people who *anchor* communities, in this order:

### Step 1 — Affiliates
Content creators, YouTubers, streamers, and organizations that work with us directly. Affiliates are onboarded and vetted by our team (the status can't be self-selected). They bring their existing audience, make the platform livelier with content, and their posts are the only member-created content signed-out visitors see. **This is our cheapest credible acquisition channel** and the one being built right now.

### Step 2 — Retailers
Local game stores and online shops. Stores follow proof, not pitches: when one store sees a business account genuinely making money in this ecosystem, that's the pull. Retailers also anchor in-person play and events, which is where a lot of trading originates.

### Step 3 — Big group moderators
The people who run the large Discord servers and Facebook trading groups. They *are* the community's trust layer today. Bringing a moderator in brings their group's trading structure with them, without asking anyone to leave Discord or Facebook. Nanza sits alongside those tools; it doesn't replace them.

### Step 4 — Standardize, then scale across games
Once the affiliate → retailer → moderator motion is a repeatable playbook for Flesh and Blood, we run the same playbook per game. Every licensor keeps their own community, rules, and canonical data, but all of it sits under one shared schema, so each new licensor makes the standard stronger for the ones already on it. This is where Tier A and Tier B IP holders become partners rather than prospects.

**Strongest channel overall:** connectivity into the tools people already use (Discord, Facebook). We'll use everything (LinkedIn, game stores, local events), but meeting people inside their existing tools beats asking them to move.

**Growth tactics we refuse:** anything that fills the network with people who don't belong in it. No paid-install blitzes, no engagement dark patterns, no commission, no random ads. Trust is the product.

### 5.1 The current six-week plan

Two tracks running in parallel, nothing else:

1. **Flesh and Blood go-to-market.** The whole product and community effort is on FAB: affiliates, retailers, and moderators in the FAB community, and the LSS relationship. This is the proof case. Nothing about other games gets built until this motion is repeatable.
2. **Outreach to the target list.** Skylar and Gino work their respective lists (Tier B and Tier A in Section 3) so that when the FAB playbook is standardized, the next licensors are already warm. Outreach is conversation and qualification, not product commitments.

The 90-day engagement overlaps this window. In practice that means the first six weeks of your work should sharpen the FAB motion and the outreach, not open new fronts.

### 5.2 LinkedIn outreach (Skylar's motion for Tier B)

LinkedIn is the channel for publishers and IP holders because that's where their partnership, licensing, brand, and community leads actually are. The motion as it stands:

- **Who we contact:** partnership / licensing / brand / community leads at the companies in Tier B, not generic inbound addresses.
- **The message:** short, specific to their game, framed as a *service* (canonical data distribution, community insight, community tooling) rather than a platform migration or a partnership ask. Lead with the black-box problem: trading in Discord and Facebook groups that they can't see.
- **The ask:** a conversation, not a deal. First-meeting goal is to learn how they think about community data and re-commerce, and whether anyone internally owns it.
- **What we bring to the call:** the FAB story (what's live, what the community does on Nanza), the partner economics in one slide, and the honest posture (they could build it; the data stays theirs; the door stays open).

**Where it stands (Sept 2026).** Skylar has already connected with a pool of roughly **1,200 LinkedIn contacts** directly associated with trading cards, collectibles, or toys: licensing and partnership leads at gaming companies and TCG publishers, and founders ranging from small studios to multi-billion-dollar businesses. That pool is the asset; the plan is to keep growing it and work it. The Tier B list in Section 3 is the named priority set inside that pool.

**Where the marketing hire can help:** sharpen Skylar's profile and the content that runs alongside the outreach, turn the pool into a segmented list with a follow-up cadence, and draft message variants per segment (publisher partnership lead vs. founder vs. licensing) so the 1,200 become conversations rather than connections.

---

## 6. Where you come in — the 90-day engagement

We're looking for a **90-day strategy** across three workstreams. Below is what we think each one needs. Treat it as a starting point to push back on, not a spec.

### 6.1 Marketing strategy

**Questions we need answered.**
- How do we present Nanza to each of the three audiences (community members, affiliates/retailers/moderators, IP-holder partners) without diluting the one-line story?
- What does an affiliate program look like as a *program* (recruitment, onboarding, incentives that aren't cash-per-install, content cadence)?
- What's the retailer story: what does a local game store get, concretely, from having a Nanza presence?
- How do we tell the LSS / Flesh and Blood story publicly in a way that recruits the next IP-holder partner (the "visible community success" that sells the next deal)?
- Which channels and content formats fit a *tighter, not broader* community thesis?

**Constraints.** No doomscroll mechanics, no commission, no third-party ads, no "disruption" framing, and everything we say to a partner has to be something we can deliver. Overpromising damages the only asset that compounds.

### 6.2 Business development

**Questions we need answered.**
- A repeatable IP-holder partner pipeline: stages, qualification criteria (strong community, need to grow, limited internal tech capability, cares about community over extraction), and what "signed" looks like.
- The **LSS agreement structure**: scope, economics (retainer + usage + custom), data terms, and the **data-exit terms** (what a departing partner takes vs. what Nanza keeps). We want this written down *before* LSS signs so we can put it on the table unprompted.
- Sequencing across Tier A / Tier B: who are the best next two or three after LSS, and why?
- Retailer and moderator partnership templates that Gino and Skylar can run without reinventing each time.
- Which Tier B names are worth real time versus a light-touch LinkedIn presence.

### 6.3 Investor relationships

**What we want from this workstream.**
- Map the right investors for a company like this (collectibles, community platforms, freemium and data businesses like Discord, small-team companies) and open relationships early.
- Build the narrative and materials: the story in Section 1 and Q134 of the questionnaire, the partner economics and the member freemium model, the FAB proof case, and the honest posture on moat and scale.
- Define the evidence bar we want to show investors (LSS signed, IP-holder partners in pipeline, members converting to paid caps, community metrics) and sequence the conversations around it.
- Recommend the raise: amount, timing, and what it funds (partner onboarding capacity, community specialists, account management) versus what it should not (growth theater, headcount for its own sake).
- Keep the raise from pulling the company away from the small, community-first shape it's built to be.

### 6.4 Suggested 30 / 60 / 90 shape

| Window | Marketing | BD | Investor relations |
|---|---|---|---|
| **Days 1–30** | Audit current story, site, and app messaging; interview Skylar, Tahir, Gino; define the three-audience positioning. **All applied to FAB first.** | Document the LSS deal structure and data-exit terms; sharpen Skylar's LinkedIn motion and Gino's flagship-lane pitches with real objections. | Map the investor landscape; agree on the evidence bar and target raise shape. |
| **Days 31–60** | FAB affiliate program design; FAB retailer one-pager; the FAB community success story. | Support live outreach (Tier B on LinkedIn, Tier A founder-to-founder); by week six, a written FAB go-to-market playbook the team can repeat per game. | Open relationships with well-fit investors; draft the narrative and materials. |
| **Days 61–90** | Launch affiliate program v1; content cadence; measurable sharing volume (our retention signal). | LSS agreement in motion; at least one Tier A or Tier B IP holder in active negotiation; retailer/moderator templates handed to the team. | Written recommendation on raise amount and timing; active investor conversations underway. |

**How we'll know it's working.** Not vanity metrics. Sharing volume in both directions (content in from Discord/YouTube, content out to Discord), groups formed, trades completed, and above all: IP-holder partnerships moving toward signed and members choosing to pay for more capacity.

---

## 7. Materials

**Available now.**
- This brief.
- [[nanza-founder-skylar|Founder Strategy Questionnaire]] — 136 questions on origin, customer, product vision, moat, licensor strategy, business model, growth, risk, and philosophy. The best single source for Nanza's positioning in Skylar's own words. **One caveat:** it describes the licensor as the *only* paying customer. The model has since moved to the hybrid in Section 1 (member freemium + IP holders + adjacent businesses); read Section 06 of the questionnaire with that in mind.
- **TCG Partner Prioritization** (Gino, 25 Aug 2026) — the scored ranking of founder-led games, outreach lanes, four-week partner offer, diligence questions, and source register. Scores are in Section 3; ask Gino for the full PDF.
- The TCG landscape list (Appendix A).
- The live mobile app and nanza.app (marketing site + share pages).
- Landing page logo strip and copy (Flesh and Blood, Magic, Pokémon, Riftbound).

**Missing / to supply**
- Export of the ~1,200-contact LinkedIn pool, segmented, with notes on who's been messaged (Section 5.2).
- Current traction numbers for Flesh and Blood on Nanza (members, groups, listings, shares).
- Status notes on the LSS relationship and any other IP-holder conversations from the summer.
- Any existing deck, one-pager, or affiliate/retailer collateral.
- Brand assets (logo, fonts, screenshots).
- Contact list / CRM for outreach already in flight.

---

## 8. Vocabulary

- **TCG** — trading card game: a collectible card game sold in randomized packs with a secondary market for singles (Magic, Pokémon, Flesh and Blood). Distinct from an *expandable* or *living* card game, where every card is available in fixed sets and there is little secondary market.
- **Licensor / IP holder** — the company that owns the IP (e.g. Legend Story Studios for Flesh and Blood). A paying **partner**: pays to reach and understand their community. Not "the customer"; members and adjacent businesses pay too.
- **Community member** — anyone using Nanza. Free to use; can pay a few dollars a month to raise usage caps. Never pays commission.
- **Caps / credits** — the freemium model: lifetime and usage caps on creation, scanning, and AI features, raised by paying. Not feature-gated subscription tiers.
- **Affiliate** — a business, creator, or organization vetted by our team that works with us directly; their content is visible to everyone, including signed-out visitors.
- **Group** — a private space where trading actually happens; the heart of the product.
- **Listing / Bid / Bulk (Lot) / Collection / Post / Tag** — the content types members create and share.
- **Reference code / share link** — every content item has a short code and a public web page.
- **Canonical data** — the IP holder's source-of-truth card data, distributed through Nanza rather than scraped.
- **Nanza AI / Cortex** — the insight engine on top of canonical data. Cortex is the engine's name, used alongside Nanza AI.
- **Full-loop integration** — an IP-holder partner whose data, community, and insight all run through Nanza, as opposed to one that just indexes their products.
- **LSS** — Legend Story Studios.
- **FAB** — Flesh and Blood.

---

## Appendix A — TCG landscape reference

Every game we've looked at, grouped by who publishes it. Notes are Skylar's and Gino's. This is a reference, not a target list; the target list is Section 3.

### Major publishers and licensed IP

| Game | Publisher / IP | Link | Notes |
|---|---|---|---|
| Magic: The Gathering | Wizards of the Coast (Hasbro) | https://magic.wizards.com/ | |
| Pokémon TCG | The Pokémon Company | https://www.pokemon.com/us/pokemon-tcg | |
| Yu-Gi-Oh! | Konami | https://www.yugioh-card.com/en/ | |
| Disney Lorcana | Ravensburger / Disney | https://www.disneylorcana.com/ | |
| Star Wars: Unlimited | Fantasy Flight (Asmodee) | https://starwarsunlimited.com/ | |
| Riftbound (League of Legends) | Riot IP, published by UVS Games | https://riftbound.leagueoflegends.com/ | On our landing page logo strip. |
| One Piece Card Game | Bandai | https://en.onepiece-cardgame.com/ | Where Tahir first saw the problem. |
| Digimon Card Game | Bandai | https://world.digimoncard.com/ | |
| Dragon Ball Super Card Game | Bandai | https://www.dbs-cardgame.com/fw/en/ | |
| Union Arena | Bandai | https://www.unionarena-tcg.com/en/ | |
| Gundam Card Game | Bandai | https://www.gundam-gcg.com/en/ | |
| Naruto Card Game | Bandai | https://www.naruto-cardgame.com/en/ | Distinct from Kayou Naruto collectibles. |
| Battle Spirits | Bandai | https://www.battlespirits.com/ | JP site is bs-tcg.com. |
| Weiss Schwarz | Bushiroad | https://en.ws-tcg.com/ | |
| Cardfight!! Vanguard | Bushiroad | https://en.cf-vanguard.com/ | |
| hololive OFFICIAL CARD GAME | Bushiroad | https://en.hololive-official-cardgame.com/ | |
| Godzilla Card Game | Bushiroad + Toho | https://en.godzilla-cardgame.com/ | |
| Shadowverse: Evolve | Bushiroad | https://en.shadowverse-evolve.com/ | |
| Final Fantasy TCG | Square Enix | https://fftcg.square-enix-games.com/ | |
| Duel Masters | Takara Tomy | https://dm.takaratomy.co.jp/ | JP only; English line discontinued 2006. |
| WIXOSS | Takara Tomy | https://www.takaratomy.co.jp/products/wixoss/ | |
| Detective Conan Card Game | Takara Tomy | https://www.takaratomy.co.jp/english/products/brand/conan.html | JP only, no English release. |
| Palworld TCG | | https://en.palworld-official-cardgame.com/ | |
| Marvel Champions | Fantasy Flight (Asmodee) | https://store.asmodee.com/products/marvel-champions-the-card-game | LCG, not collectible. |
| Arkham Horror | Fantasy Flight (Asmodee) | http://arkhamhorror.com/ | LCG, not collectible. |
| Overpower TCG | Skybound | https://overpowercardgame.com/ | Skybound is legit and works with real studios; Overpower is their new TCG. |
| Vibes | Orange Cap Games, Pudgy Penguins IP | https://www.vibes.game/ | |
| Azuki TCG | Azuki | https://tcg.azuki.com/ | Has an incredible tech stack already. |
| Hearthstone | Blizzard | https://hearthstone.blizzard.com/ | Digital only, no physical TCG. |
| Chaotic | | https://www.chaoticgame.com/about | |

### Established independents

| Game | Publisher | Link | Notes |
|---|---|---|---|
| Flesh and Blood | Legend Story Studios | (see Section 3) | Our live game. |
| Sorcery: Contested Realm | Erik's Curiosa | https://sorcerytcg.com/ | |
| Grand Archive | Weebs of the Shore | https://www.gatcg.com/ | |
| Force of Will | | https://www.fowtcg.com/ | |
| SolForge Fusion | Stone Blade | https://solforgefusion.com/ | Scored 78 (Section 3). |
| Vampire: The Eternal Struggle | Black Chantry | https://www.blackchantry.com/ | |
| Netrunner | Null Signal Games | https://nullsignal.games/ | Community run. |
| Gudnak | | https://gudnak.com/ | |
| Warlord: Saga of the Storm | Kingswood Games (AEG licensee) | https://www.warlordsots.com/ | Scored 78. |

### Founder-led, crowdfunded, and new

| Game | Publisher | Link | Notes |
|---|---|---|---|
| Elestrals | | https://www.shopelestrals.com/ | Founder is also a content creator, large digital footprint. Scored 94. |
| Alpha Clash | | https://alphaclashtcg.com/ | Scored 89. |
| NeuroScape TCG | Asmodee distribution | https://www.neuroscapetcg.com/ | Scored 81. |
| Cyberpunk TCG | | https://cyberpunktcg.com/ | |
| Echoes of Astra | Parhelion Studio | https://echoesofastra.com/ | Scored 81. |
| Cryptid Camp | | https://cryptidcamp.com/ | Scored 79. |
| Neverrift | Gate to the Games | https://neverrift.com/ | Scored 75. |
| Here Be Monsters | | https://www.herebemonstersthegame.com/ | Scored 74. |
| Shard Bugs | | https://www.shardbugstcg.com/ | Scored 69. |
| Hubworld: Aidalon | Earthborne Games | https://gamefound.com/en/projects/earthborne-games/hubworld-aidalon | Huge crowdfunding support. Scored 68. |
| Earthborne Rangers | Earthborne Games | https://earthbornegames.com/ | Same studio. |
| The Caster Chronicles | Speedrobo Games (English license) | https://www.kickstarter.com/projects/mafiaworks/the-caster-chronicles-card-game | Scored 66. |
| Divine Cross | | https://divinecross.jp/ | Scored 63. |
| Planarist TCG | | https://www.planarist.com/ | Scored 58. |
| LIFE TCG | | https://www.lifetcg.com/ | Scored 50. |
| Beast Hunt | | https://shop.beast-hunt.com/ | Scored 44. |
| Quantum TCG | | https://quantumtcg.com/ | Scored 38. |
| Castlebound TCG | | https://castleboundtcg.com/ | Looks like AI. Scored 31. |
| Akora TCG | | https://akoraofficial.com/ | Hold; may be closed. Scored 18. |
| MetaZoo TCG | | https://www.metazootcg.com/ | May be defunct. |
| The Journey TCG | | https://www.journeytcg.com/ | |
| Hellbreak TCG | | https://hellbreakgame.com/ | |
| Wondrlnd | | https://wondrlnd.com/ | Digital friendly. |
| Rise TCG | | https://www.risetcgofficial.com/ | |
| Sqwürmish | Creature Curation | https://creaturecuration.com/sqwurmish/ | |
| OSICA | | https://www.osicatcg.com/ | |
