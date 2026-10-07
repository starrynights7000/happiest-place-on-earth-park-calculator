# Family Park Cost Calculator: Project Handoff

Everything from the original conversation, in one file: goal, decisions, source-of-truth data, design system, version history, ideas not yet built, and the full code of the current version.

---

## 1. Project goal

A single-file web calculator that compares the total cost of a 7-day family holiday across 5 major Disney-style resort parks.

- **Base scenario:** family of 5 flying from **Tel Aviv**: 2 adults, a 14-year-old, a 12-year-old, and one child under 9.
- **Trip shape:** 7 days total. 3 park days (3-day tickets) and 4 other nights in a regular hotel. Default split: 3 nights near the park + 4 nights elsewhere.
- **IP constraint:** no Disney logos, fonts, character art, or castle silhouettes. Park names appear only as plain text for identification.
- **Output:** per-park cost cards ranked cheapest to priciest, plus a stacked bar chart (flights / admission / hotels / food and extras).

---

## 2. The 5 parks

| # | Park | Why it's in the comparison |
|---|------|----------------------------|
| 1 | Anaheim, California (USA West) | Original park, smallest footprint, long and pricey flight from Tel Aviv |
| 2 | Orlando, Florida (USA East) | Biggest resort (4 parks), long flight |
| 3 | Paris, France | Cheapest flight from Tel Aviv. Child fare runs to age 11 |
| 4 | Tokyo, Japan | Cheapest tickets. Only resort with a third "junior" tier (12-17) |
| 5 | Hong Kong | Smallest, lowest hotel costs, no true 3-day ticket |

Shanghai was left out because of distance and visa friction. It could be a 6th park.

---

## 3. Source of truth: ticket age brackets

Each park applies its own age rule, so the same family lands in different price tiers at different parks. This is the main thing the calculator does that a simple sum would not.

| Park | Adult | Junior | Child |
|------|-------|--------|-------|
| Anaheim | 10+ | none | 3-9 |
| Orlando | 10+ | none | 3-9 |
| Paris | 12+ | none | 3-11 |
| Tokyo | 18+ | 12-17 | 4-11 |
| Hong Kong | 12-59 | none | 3-11 |

**How the family maps:**

| Traveler | Anaheim / Orlando | Paris | Tokyo | Hong Kong |
|----------|------|-------|-------|-----------|
| Adults (2) | adult | adult | adult | adult |
| Teen, 14 | adult | adult | **junior** | adult |
| Kid, 12 | adult | adult | **junior** | adult |
| Child, under 9 | child | child | child | child |

**Why the youngest child's age is not an input:** every park's child cutoff is 10 or higher. Any age from 0 to 9 gives the child fare everywhere, so an editable field would add no information. It is hardcoded as a placeholder age of 7. If you ever model a child near a cutoff (9 vs 10, or 11 vs 12), make the age editable again.

---

## 4. Source of truth: default prices (USD, regular season)

### 4a. Park admission, 3 days, per person

Based on 2026 published rates found by web search during the conversation. Treat them as approximate starting points, not live quotes.

| Park | Adult | Junior | Child | Notes |
|------|-------|--------|-------|-------|
| Anaheim | $425 | n/a | $400 | One park per day, no hopper |
| Orlando | $405 | n/a | $385 | Base ticket, no hopper |
| Paris | $308 | n/a | $285 | Converted from EUR 285 / EUR 264 (2-park multi-day ticket) |
| Tokyo | $184 | $145 | $104 | No multi-day passport, so 3 x mid-range 1-day price (JPY 9,400 / 7,400 / 5,300) |
| Hong Kong | $230 | n/a | $170 | No 3-day ticket exists. Built as 2-day + 1-day |

**Conversion rates used:** EUR 1 = $1.08, JPY 153 = $1, HKD 1 = $0.128. The ILS display toggle defaults to 3.7 ILS per USD and is editable.

### 4b. Flights, round trip, per seat, from Tel Aviv

These are **my estimates, not researched quotes.** Flight prices swing too much by date and airline to pin down. Every one is editable in the UI.

| Park | $/seat |
|------|--------|
| Anaheim | 1050 |
| Orlando | 900 |
| Paris | 340 |
| Tokyo | 900 |
| Hong Kong | 750 |

Fares are flat per seat, with no child discount, since economy international fares are the same for ages 2 and up.

### 4c. Hotels, per night for the whole family (also my estimates)

| Park | Near park | Rest of trip |
|------|-----------|--------------|
| Anaheim | 280 | 180 |
| Orlando | 250 | 150 |
| Paris | 220 | 160 |
| Tokyo | 260 | 170 |
| Hong Kong | 200 | 140 |

### 4d. Food and extras

- Food: **$42 per person per day** (editable)
- Extras (transit, souvenirs): **$35 per day for the family** (editable)

### 4e. Calculation

```
flights   = flight_per_seat * 5
tickets   = sum over travelers of (tier price for that traveler's age at this park)
hotels    = hotelNear * nearNights + hotelOther * otherNights
food+misc = foodPerPerson * 5 * totalDays + miscPerDay * totalDays
total     = flights + tickets + hotels + food+misc
totalDays = nearNights + otherNights   (default 3 + 4 = 7)
```

Cards are ranked by ascending total. The cheapest gets a gold 2px border and a "#1" badge.

---

## 5. Design system ("Ticket Stub")

```
COLORS
Ink (text on light)      #0F2B25
Paper (light surface)    #F2ECD8
Teal (page background)   #123832
Teal-2 (panel surface)   #1B4B43
Gold (primary accent)    #D3A34C
Gold-dim (hover/active)  #B8863A
Line on dark             rgba(242,236,216,0.22)
Line on light            rgba(15,43,37,0.14)

Category accents (charts and legends only):
Gold  #D3A34C  flights
Berry #A3335F  tickets / admission
Sage  #5B8C7B  hotels
Slate #6E7B8A  food / extras

TYPOGRAPHY
Display / headings: Fraunces (serif), 400-600, 18-42px
Body / UI / data:   Inter (sans), 400-700, tabular-nums for numbers
Google Fonts import:
https://fonts.googleapis.com/css2?family=Fraunces:opsz,wght@9..144,400;9..144,500;9..144,600&family=Inter:wght@400;500;600;700&display=swap

STRUCTURE / TEXTURE
- Dark teal page, cream "paper" cards for ticket/receipt content,
  darker teal-2 panels for inputs and controls.
- Card radius 16px, inputs and buttons 8px.
- Dashed 1px dividers between line items (receipt feel).
- Vertical row of dots along the hero card's left edge (torn-stub perforation).
- Hairline borders; a 2px gold-dim border marks the featured (cheapest) card only.
- Flat fills only: no gradients, shadows, or glow.

VOICE
Plain sentence case, no exclamation points. Estimates are labeled as
"editable starting points", never as official prices.
```

---

## 6. Version history

**v1, the original (restored as the current file).** Fixed Tel Aviv family of 5, 5 park cards, stacked bar chart, USD/ILS toggle, all numbers editable. Simple, which is why it was kept.

**v2, the "dynamic" rebuild (overwritten, reverted by choice).** It added:

- A free-form traveler list: add and remove people, each just an age.
- A season selector (off-peak / regular / peak) in place of real dates.
- A home-region dropdown that pre-filled flight estimates.
- Hotel tier pickers with descriptions.
- A Park Hopper toggle where it exists.
- An email-capture section, later removed.

It was reverted because the extra controls buried the quick per-park estimates, which were the part that mattered. The v2 logic and data are kept below in section 7 so it can be rebuilt on demand.

**v1.1 (current).** v1 minus the "youngest child's age" input, which had no effect (see section 3).

---

## 7. v2 features: specs and data (not in the current file)

### 7a. Season multiplier

No real dates. A single selector scales flights, tickets, hotels, and food together:

```js
const SEASON_MULT = { offpeak: 0.82, regular: 1.0, peak: 1.35 };
```

The stored base numbers stay as "regular", so switching season never destroys edited values. These multipliers are rough judgment calls, not measured data.

### 7b. Home-region flight defaults (USD round trip per seat; my estimates)

| Region | Anaheim | Orlando | Paris | Tokyo | Hong Kong |
|--------|---------|---------|-------|-------|-----------|
| Middle East | 1050 | 900 | 340 | 900 | 750 |
| Europe | 750 | 650 | 100 | 750 | 650 |
| USA East | 400 | 250 | 500 | 950 | 1000 |
| USA West | 120 | 350 | 750 | 650 | 750 |
| Canada East | 450 | 350 | 550 | 1050 | 1100 |
| Canada West | 250 | 450 | 850 | 700 | 800 |
| Latin America | 600 | 500 | 750 | 1400 | 1300 |
| East Asia (Japan, Korea) | 700 | 950 | 900 | 100 | 350 |
| Southeast Asia | 800 | 1050 | 750 | 350 | 250 |
| Australia / NZ | 900 | 1350 | 1350 | 700 | 650 |
| Africa | 1250 | 1050 | 600 | 1050 | 900 |
| Other / custom | 0 | 0 | 0 | 0 | 0 |

A region only fills a default. The dollar field stays editable, which beats a flight-hours slider because a 6-hour flight costs different amounts from different places.

### 7c. Hotel tiers (near-park / rest-of-trip $ per night, plus what it buys)

| Park | Value | Moderate | Deluxe |
|------|-------|----------|--------|
| Anaheim | 180 / 120, 3-star off-site | 280 / 190, 4-star or resort Value | 420 / 280, on-site Deluxe |
| Orlando | 140 / 95, resort Value or 3-star | 250 / 170, resort Moderate | 450 / 300, resort Deluxe |
| Paris | 130 / 90, partner hotel with shuttle | 220 / 150, on-site standard | 380 / 250, on-site premium |
| Tokyo | 150 / 100, business hotel near the park | 260 / 175, official hotel standard | 420 / 280, official hotel premium |
| Hong Kong | 110 / 75, 3-star off-site | 200 / 135, on-site standard | 320 / 215, on-site premium |

### 7d. Park Hopper: which resorts actually have it

| Park | Hopper? | Handling |
|------|---------|----------|
| Anaheim | Yes, paid add-on | Toggle, about +$110 per person |
| Orlando | Yes, paid add-on | Toggle, about +$120 per person |
| Paris | Included | Multi-day tickets already cover both parks. Show a note |
| Tokyo | No | Standard tickets are one park per day. Show a note |
| Hong Kong | Not applicable | Only one park. Show a note |

### 7e. Other v2 ideas

- **Email delivery:** a static HTML file cannot send email. The prototype used `mailto:` plus copy-to-clipboard. Real delivery needs a form or email service (Mailchimp, ConvertKit, Google Form, Zapier/Make webhook) or a small backend.
- **Lead-gen funnel:** a reference competitor tool used an 11-step quiz and gated the results behind name and email. That needs a backend and was not built.
- **Open questions:** whether to support more families or origins publicly, and whether to add Shanghai.

---

## 8. Known limitations

- Ticket prices are approximate 2026 figures. Flights and hotels are unresearched estimates.
- No real dates, so no live dynamic pricing. Season bands would be the workaround.
- Hong Kong's 3-day cost is a combination of ticket types, not a real product.
- Tokyo's 3-day cost assumes single-day tickets, since multi-day passports were not on sale at research time.
- Fonts load from Google Fonts, so the page needs internet access to render them as designed.

---

## 9. Full code: current version (v1.1)

Save as `park-cost-calculator.html` and open in any browser. No build step.

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Family park cost comparison</title>
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Fraunces:opsz,wght@9..144,400;9..144,500;9..144,600&family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">
<style>
  :root{
    --ink:#0F2B25;
    --paper:#F2ECD8;
    --paper-dim:#E7DFC5;
    --teal:#123832;
    --teal-2:#1B4B43;
    --gold:#D3A34C;
    --gold-dim:#B8863A;
    --berry:#A3335F;
    --line: rgba(15,43,37,0.14);
    --line-light: rgba(242,236,216,0.22);
  }
  *{box-sizing:border-box;}
  body{
    margin:0;
    background:var(--teal);
    color:var(--paper);
    font-family:'Inter',sans-serif;
    padding:32px 20px 64px;
  }
  .wrap{max-width:1180px;margin:0 auto;}

  header.hero{
    position:relative;
    background:var(--paper);
    color:var(--ink);
    border-radius:18px;
    padding:34px 34px 28px;
    margin-bottom:22px;
    overflow:hidden;
    border:1px solid var(--line);
  }
  header.hero::before{
    content:"";
    position:absolute; top:0; bottom:0; left:0;
    width:14px;
    background-image: radial-gradient(circle, var(--teal) 3px, transparent 3.5px);
    background-size: 14px 18px;
    background-position: center;
    opacity:0.5;
  }
  .eyebrow{
    font-size:13px;
    letter-spacing:0.02em;
    color:var(--gold-dim);
    font-weight:600;
    margin:0 0 6px 20px;
  }
  h1{
    font-family:'Fraunces',serif;
    font-weight:500;
    font-size:clamp(28px,4vw,42px);
    line-height:1.08;
    margin:0 0 14px 20px;
    max-width:640px;
  }
  .hero-meta{
    display:flex;
    flex-wrap:wrap;
    gap:10px 28px;
    margin:0 0 4px 20px;
    font-size:14px;
    color:var(--ink);
    opacity:0.82;
  }
  .hero-meta b{font-weight:600;}

  .panel{
    background:var(--teal-2);
    border:1px solid var(--line-light);
    border-radius:16px;
    padding:22px 26px;
    margin-bottom:22px;
  }
  .panel h2{
    font-family:'Fraunces',serif;
    font-weight:500;
    font-size:18px;
    margin:0 0 4px;
  }
  .panel p.sub{
    margin:0 0 18px;
    font-size:13px;
    color:rgba(242,236,216,0.6);
    max-width:640px;
  }
  .assume-grid{
    display:grid;
    grid-template-columns:repeat(auto-fit,minmax(160px,1fr));
    gap:16px 20px;
  }
  .field label{
    display:block;
    font-size:12px;
    color:rgba(242,236,216,0.62);
    margin-bottom:6px;
  }
  .field .row{display:flex; align-items:center; gap:6px;}
  input[type=number]{
    width:100%;
    background:var(--teal);
    border:1px solid var(--line-light);
    color:var(--paper);
    border-radius:8px;
    padding:8px 10px;
    font-family:'Inter',sans-serif;
    font-size:14px;
    font-variant-numeric: tabular-nums;
  }
  input[type=number]:focus{outline:2px solid var(--gold); outline-offset:1px;}
  .unit{font-size:12px;color:rgba(242,236,216,0.5); white-space:nowrap;}
  .toggle-row{display:flex; gap:8px; margin-top:2px;}
  .toggle-btn{
    flex:1;
    background:var(--teal);
    border:1px solid var(--line-light);
    color:var(--paper);
    border-radius:8px;
    padding:8px 10px;
    font-size:13px;
    font-family:'Inter',sans-serif;
    cursor:pointer;
  }
  .toggle-btn.active{
    background:var(--gold);
    border-color:var(--gold);
    color:var(--ink);
    font-weight:600;
  }

  .cards{
    display:grid;
    grid-template-columns:repeat(auto-fit,minmax(210px,1fr));
    gap:16px;
    margin-bottom:24px;
  }
  .card{
    background:var(--paper);
    color:var(--ink);
    border-radius:16px;
    position:relative;
    padding:20px 18px 18px;
    display:flex;
    flex-direction:column;
    border:1px solid rgba(15,43,37,0.12);
  }
  .card.best{
    border:2px solid var(--gold-dim);
  }
  .rank{
    position:absolute;
    top:14px; right:14px;
    font-family:'Fraunces',serif;
    font-size:12px;
    background:var(--teal);
    color:var(--paper);
    border-radius:20px;
    padding:3px 10px;
    font-weight:500;
  }
  .card.best .rank{background:var(--gold-dim); color:var(--ink);}
  .card-title{
    font-family:'Fraunces',serif;
    font-size:19px;
    font-weight:500;
    margin:0 0 2px;
    padding-right:60px;
  }
  .card-sub{
    font-size:12px;
    color:rgba(15,43,37,0.55);
    margin:0 0 14px;
  }
  .line-item{
    font-size:12.5px;
    display:flex;
    justify-content:space-between;
    align-items:baseline;
    gap:8px;
    padding:5px 0;
    border-top:1px dashed rgba(15,43,37,0.16);
  }
  .line-item:first-of-type{border-top:none;}
  .line-item .lbl{color:rgba(15,43,37,0.68);}
  .line-item .val{font-variant-numeric:tabular-nums; white-space:nowrap; font-weight:500;}
  .subrow{
    display:flex;
    gap:6px;
    align-items:center;
    margin:4px 0 2px;
  }
  .subrow input{
    width:74px;
    background:transparent;
    border:1px solid rgba(15,43,37,0.24);
    color:var(--ink);
    border-radius:6px;
    padding:4px 6px;
    font-size:12px;
    font-family:'Inter',sans-serif;
    font-variant-numeric: tabular-nums;
  }
  .subrow .tag{font-size:11.5px; color:rgba(15,43,37,0.6); flex:1;}
  .tier-block{margin:10px 0 6px;}
  .tier-block .tier-title{font-size:12px; font-weight:600; margin-bottom:5px; color:rgba(15,43,37,0.72);}
  .total-row{
    margin-top:14px;
    padding-top:12px;
    border-top:1.5px solid var(--ink);
    display:flex;
    justify-content:space-between;
    align-items:baseline;
  }
  .total-row .lbl{font-size:12.5px; text-transform:none; color:rgba(15,43,37,0.65);}
  .total-row .val{
    font-family:'Fraunces',serif;
    font-size:23px;
    font-weight:500;
  }
  .per-person{font-size:11px; color:rgba(15,43,37,0.5); margin-top:2px; text-align:right;}

  .chart-panel .bar-row{
    display:grid;
    grid-template-columns:130px 1fr 84px;
    align-items:center;
    gap:12px;
    margin:12px 0;
    font-size:13px;
  }
  .bar-track{
    height:20px;
    border-radius:5px;
    overflow:hidden;
    display:flex;
    background:rgba(242,236,216,0.08);
  }
  .bar-seg{height:100%;}
  .legend{
    display:flex;
    gap:18px;
    flex-wrap:wrap;
    margin-top:16px;
    font-size:12px;
    color:rgba(242,236,216,0.65);
  }
  .legend span{display:inline-flex; align-items:center; gap:6px;}
  .swatch{width:10px; height:10px; border-radius:2px; display:inline-block;}

  footer.note{
    font-size:12px;
    color:rgba(242,236,216,0.45);
    max-width:820px;
    line-height:1.6;
    margin-top:8px;
  }
</style>
</head>
<body>
<div class="wrap">

  <header class="hero">
    <p class="eyebrow">Family trip budgeting</p>
    <h1>Which park makes the most sense this year?</h1>
    <div class="hero-meta">
      <span><b>2 adults</b> + <b>3 kids</b> (ages 14, 12, and one under 9)</span>
      <span>Flying from <b>Tel Aviv</b></span>
      <span><b>7-day</b> trip &middot; <b>3</b> park days &middot; <b>4</b> other nights</span>
    </div>
  </header>

  <div class="panel">
    <h2>Trip assumptions</h2>
    <p class="sub">Every number below — and every number on the cards — is editable. Fill in real quotes as you collect them; everything recalculates instantly.</p>
    <div class="assume-grid">
      <div class="field">
        <label for="foodPerPerson">Food, per person / day</label>
        <div class="row"><input type="number" id="foodPerPerson" value="42" min="0" step="1"><span class="unit">$</span></div>
      </div>
      <div class="field">
        <label for="miscPerDay">Extras (transit, souvenirs) / day</label>
        <div class="row"><input type="number" id="miscPerDay" value="35" min="0" step="1"><span class="unit">$/family</span></div>
      </div>
      <div class="field">
        <label for="nearNights">Nights near the park</label>
        <input type="number" id="nearNights" value="3" min="0" max="10" step="1">
      </div>
      <div class="field">
        <label for="otherNights">Nights, rest of the trip</label>
        <input type="number" id="otherNights" value="4" min="0" max="10" step="1">
      </div>
      <div class="field">
        <label>Show totals in</label>
        <div class="toggle-row">
          <button class="toggle-btn active" data-cur="USD">USD</button>
          <button class="toggle-btn" data-cur="ILS">ILS</button>
        </div>
      </div>
      <div class="field" id="fxField" style="display:none;">
        <label for="fxRate">USD → ILS rate</label>
        <input type="number" id="fxRate" value="3.7" min="1" step="0.01">
      </div>
    </div>
  </div>

  <div id="cards" class="cards"></div>

  <div class="panel chart-panel">
    <h2>Total cost, side by side</h2>
    <p class="sub">Flights &middot; park admission &middot; hotels &middot; food and extras, stacked per park.</p>
    <div id="chart"></div>
    <div class="legend">
      <span><i class="swatch" style="background:#D3A34C"></i>Flights</span>
      <span><i class="swatch" style="background:#A3335F"></i>Park admission</span>
      <span><i class="swatch" style="background:#5B8C7B"></i>Hotels</span>
      <span><i class="swatch" style="background:#6E7B8A"></i>Food and extras</span>
    </div>
  </div>

  <footer class="note">
    Figures are editable starting points based on published 2026 rates, not live quotes — actual prices move with season and date. Park admission is calculated per traveler using each resort's own age rule for adult / junior / child pricing, so the same family can land in a different bracket at different parks. No trademarked names, characters, or fonts belonging to any park operator are used in this tool; park names appear only as plain text for identification.
  </footer>

</div>

<script>
const PARKS = [
  {
    id:'anaheim', name:'Anaheim, California', place:'USA — West Coast',
    ageRule: a => a>=10 ? 'adult' : 'child',
    tiers: ['adult','child'],
    tierLabels: {adult:'Ages 10+', child:'Ages 3–9'},
    flight: 1050,
    ticket: {adult:425, child:400},
    hotelNear: 280, hotelOther: 180
  },
  {
    id:'orlando', name:'Orlando, Florida', place:'USA — East Coast',
    ageRule: a => a>=10 ? 'adult' : 'child',
    tiers: ['adult','child'],
    tierLabels: {adult:'Ages 10+', child:'Ages 3–9'},
    flight: 900,
    ticket: {adult:405, child:385},
    hotelNear: 250, hotelOther: 150
  },
  {
    id:'paris', name:'Paris', place:'France',
    ageRule: a => a>=12 ? 'adult' : 'child',
    tiers: ['adult','child'],
    tierLabels: {adult:'Ages 12+', child:'Ages 3–11'},
    flight: 340,
    ticket: {adult:308, child:285},
    hotelNear: 220, hotelOther: 160
  },
  {
    id:'tokyo', name:'Tokyo', place:'Japan',
    ageRule: a => a>=18 ? 'adult' : (a>=12 ? 'junior' : 'child'),
    tiers: ['adult','junior','child'],
    tierLabels: {adult:'Ages 18+', junior:'Ages 12–17', child:'Ages 4–11'},
    flight: 900,
    ticket: {adult:184, junior:145, child:104},
    hotelNear: 260, hotelOther: 170
  },
  {
    id:'hongkong', name:'Hong Kong', place:'China — SAR',
    ageRule: a => a>=12 ? 'adult' : 'child',
    tiers: ['adult','child'],
    tierLabels: {adult:'Ages 12–59', child:'Ages 3–11'},
    flight: 750,
    ticket: {adult:230, child:170},
    hotelNear: 200, hotelOther: 140
  }
];

const cardsEl = document.getElementById('cards');
const chartEl = document.getElementById('chart');
let currency = 'USD';

function familyAges(){
  // The youngest child (under 10) is always the "child" fare at every park in
  // this list, since the lowest adult/junior cutoff anywhere is age 10 — so a
  // fixed placeholder age works exactly as well as an editable one here.
  return [30,30,14,12,7];
}

function buildCard(park){
  const div = document.createElement('div');
  div.className = 'card';
  div.dataset.id = park.id;

  const tierRows = park.tiers.map(t => `
    <div class="subrow" data-tier="${t}">
      <span class="tag" id="${park.id}-${t}-tag"></span>
      <input type="number" id="${park.id}-ticket-${t}" value="${park.ticket[t]}" min="0" step="1">
    </div>
  `).join('');

  div.innerHTML = `
    <div class="rank" data-rank>#1</div>
    <p class="card-title">${park.name}</p>
    <p class="card-sub">${park.place}</p>

    <div class="line-item">
      <span class="lbl">Flights, round trip × 5</span>
      <span class="val"><input type="number" id="${park.id}-flight" value="${park.flight}" min="0" step="10" style="width:64px;background:transparent;border:1px solid rgba(15,43,37,0.24);border-radius:6px;padding:3px 5px;font-family:Inter,sans-serif;font-size:12px;color:var(--ink);"> $/seat</span>
    </div>
    <div class="line-item"><span class="lbl">→ Flights subtotal</span><span class="val" id="${park.id}-flightSub">–</span></div>

    <div class="tier-block">
      <div class="tier-title">Park admission, 3 days — per person by age</div>
      ${tierRows}
    </div>
    <div class="line-item"><span class="lbl">→ Admission subtotal</span><span class="val" id="${park.id}-ticketSub">–</span></div>

    <div class="subrow">
      <span class="tag">Hotel near park, $/night</span>
      <input type="number" id="${park.id}-hotelNear" value="${park.hotelNear}" min="0" step="5">
    </div>
    <div class="subrow">
      <span class="tag">Hotel, rest of trip $/night</span>
      <input type="number" id="${park.id}-hotelOther" value="${park.hotelOther}" min="0" step="5">
    </div>
    <div class="line-item"><span class="lbl">→ Hotels subtotal</span><span class="val" id="${park.id}-hotelSub">–</span></div>

    <div class="line-item"><span class="lbl">Food and extras (shared)</span><span class="val" id="${park.id}-foodSub">–</span></div>

    <div class="total-row">
      <span class="lbl">Trip total</span>
      <span class="val" id="${park.id}-total">–</span>
    </div>
    <div class="per-person" id="${park.id}-perPerson"></div>
  `;
  cardsEl.appendChild(div);
}

PARKS.forEach(buildCard);

function fmt(n){
  const rate = currency === 'ILS' ? (parseFloat(document.getElementById('fxRate').value)||3.7) : 1;
  const symbol = currency === 'ILS' ? '₪' : '$';
  const val = Math.round(n * rate);
  return symbol + val.toLocaleString();
}

function recalc(){
  const ages = familyAges();
  const foodPerPerson = parseFloat(document.getElementById('foodPerPerson').value) || 0;
  const miscPerDay = parseFloat(document.getElementById('miscPerDay').value) || 0;
  const nearNights = parseFloat(document.getElementById('nearNights').value) || 0;
  const otherNights = parseFloat(document.getElementById('otherNights').value) || 0;
  const totalDays = nearNights + otherNights;

  const results = PARKS.map(park => {
    const flight = parseFloat(document.getElementById(`${park.id}-flight`).value) || 0;
    const flightSub = flight * 5;

    const counts = {};
    park.tiers.forEach(t => counts[t] = 0);
    ages.forEach(a => { counts[park.ageRule(a)]++; });

    let ticketSub = 0;
    park.tiers.forEach(t => {
      const price = parseFloat(document.getElementById(`${park.id}-ticket-${t}`).value) || 0;
      const tagEl = document.getElementById(`${park.id}-${t}-tag`);
      if(tagEl) tagEl.textContent = `${park.tierLabels[t]} × ${counts[t]}`;
      ticketSub += price * counts[t];
    });

    const hotelNear = parseFloat(document.getElementById(`${park.id}-hotelNear`).value) || 0;
    const hotelOther = parseFloat(document.getElementById(`${park.id}-hotelOther`).value) || 0;
    const hotelSub = hotelNear*nearNights + hotelOther*otherNights;

    const foodSub = foodPerPerson * 5 * totalDays + miscPerDay * totalDays;

    const total = flightSub + ticketSub + hotelSub + foodSub;

    document.getElementById(`${park.id}-flightSub`).textContent = fmt(flightSub);
    document.getElementById(`${park.id}-ticketSub`).textContent = fmt(ticketSub);
    document.getElementById(`${park.id}-hotelSub`).textContent = fmt(hotelSub);
    document.getElementById(`${park.id}-foodSub`).textContent = fmt(foodSub);
    document.getElementById(`${park.id}-total`).textContent = fmt(total);
    document.getElementById(`${park.id}-perPerson`).textContent = `${fmt(total/5)} per person`;

    return {park, flightSub, ticketSub, hotelSub, foodSub, total};
  });

  const sorted = [...results].sort((a,b) => a.total - b.total);
  sorted.forEach((r, i) => {
    const card = cardsEl.querySelector(`[data-id="${r.park.id}"]`);
    card.classList.toggle('best', i===0);
    card.querySelector('[data-rank]').textContent = '#' + (i+1);
  });

  renderChart(sorted);
}

function renderChart(sorted){
  const maxTotal = Math.max(...sorted.map(r=>r.total), 1);
  chartEl.innerHTML = sorted.map(r => {
    const w = n => (n/maxTotal*100).toFixed(2)+'%';
    return `
      <div class="bar-row">
        <span>${r.park.name}</span>
        <div class="bar-track">
          <div class="bar-seg" style="width:${w(r.flightSub)};background:#D3A34C"></div>
          <div class="bar-seg" style="width:${w(r.ticketSub)};background:#A3335F"></div>
          <div class="bar-seg" style="width:${w(r.hotelSub)};background:#5B8C7B"></div>
          <div class="bar-seg" style="width:${w(r.foodSub)};background:#6E7B8A"></div>
        </div>
        <span style="text-align:right;font-variant-numeric:tabular-nums;">${fmt(r.total)}</span>
      </div>
    `;
  }).join('');
}

document.querySelectorAll('.toggle-btn').forEach(btn => {
  btn.addEventListener('click', () => {
    document.querySelectorAll('.toggle-btn').forEach(b=>b.classList.remove('active'));
    btn.classList.add('active');
    currency = btn.dataset.cur;
    document.getElementById('fxField').style.display = currency==='ILS' ? 'block' : 'none';
    recalc();
  });
});

document.addEventListener('input', (e) => {
  if(e.target.tagName === 'INPUT'){ recalc(); }
});

recalc();
</script>
</body>
</html>
```

---

## 10. Resuming in a new account

Paste this into a new chat along with the file:

> I'm continuing a project: a family park cost calculator comparing 5 Disney-style resort parks (Anaheim, Orlando, Paris, Tokyo, Hong Kong) for a family of 5 from Tel Aviv. The attached handoff file has the source-of-truth data, design system, version history, and current code. Start from the current code in section 9. Don't re-add the removed youngest-child age input or the email section. Use section 7 if I ask to bring back the dynamic features.
