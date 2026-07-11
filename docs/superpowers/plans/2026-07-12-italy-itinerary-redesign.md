# Italy Itinerary Tabbed Redesign Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the existing single-view itinerary with a responsive five-tab travel guide whose 2026 dates, city stays, transport, reservations, preparation checklist, and city knowledge are internally consistent.

**Architecture:** Keep `index.html` as a static entry point and split presentation, immutable trip content, pure state helpers, and DOM behavior into focused files. Render all five panels from one structured data object; route tab, city, and filter state through URL hashes, while local storage is limited to checklist preferences.

**Tech Stack:** Semantic HTML5, CSS custom properties and responsive CSS, dependency-free ES5-compatible browser JavaScript, Node.js 23 built-in test runner, GitHub Pages-compatible relative assets.

## Global Constraints

- The itinerary is 2026-09-26 through 2026-10-06 using the actual 2026 weekdays.
- Nights are Milan 3, Florence 3, Naples 1, Rome 3; Lake Como is a Milan-based day trip.
- Main navigation has exactly five tabs: 每日行程、城市信息、交通信息、门票预订、行前准备.
- Accommodation values remain explicit `待补充` placeholders; do not read confirmation PDFs or invent hotels.
- Keep `index.html` as the only entry point and use only relative local assets.
- Do not use CDNs, online fonts, third-party UI libraries, absolute filesystem paths, or `file://`-specific behavior.
- Preserve the existing booked flight facts that belong to this trip, including MXP arrival at 08:20 on 9/26 and FCO departure at 11:15 on 10/6.
- Treat train times and unpublished 2026 attraction policies as recommendations requiring pre-departure verification, not confirmed bookings.
- The result must remain easy to migrate into a future ChatGPT Sites project, but this task does not create `.openai/hosting.json` or deploy a Site.

## File Structure

- `index.html` — semantic page shell, fallback content, tab panels, live status region, and relative asset loading.
- `assets/styles.css` — visual system, tabs, cards, timelines, accordions, tables/cards, focus states, and responsive rules.
- `assets/trip-data.js` — immutable itinerary, city, transport, ticket, flight, accommodation, and checklist content.
- `assets/app-core.js` — pure hash, filtering, search, progress, and target-resolution helpers; exports for browser and Node tests.
- `assets/app.js` — DOM rendering, event delegation, URL synchronization, local-storage fallback, and accessibility state updates.
- `tests/trip-data.test.js` — itinerary/date/night/route/accommodation integrity tests.
- `tests/app-core.test.js` — unit tests for URL state, filtering, search, and progress.
- `tests/static-site.test.js` — static contracts for semantic tabs, assets, relative paths, CSS breakpoints, focus, and forbidden dependencies.
- `package.json` — dependency-free `node --test` verification script.

---

### Task 1: Establish the canonical trip data model

**Files:**
- Create: `package.json`
- Create: `assets/trip-data.js`
- Create: `tests/trip-data.test.js`

**Interfaces:**
- Produces: `TripData` global and CommonJS export with `{ meta, flights, days, cities, transport, tickets, checklist }`.
- Produces: stable IDs `milan`, `como`, `florence`, `naples`, `rome` used by every later task.

- [ ] **Step 1: Create the test command and write failing data-integrity tests**

```json
{
  "name": "italy-itinerary-2026",
  "private": true,
  "scripts": { "test": "node --test tests/*.test.js" }
}
```

```js
// tests/trip-data.test.js
const test = require('node:test');
const assert = require('node:assert/strict');
const data = require('../assets/trip-data.js');

test('contains every travel day in chronological order', () => {
  assert.deepEqual(data.days.map((day) => day.date), [
    '2026-09-26', '2026-09-27', '2026-09-28', '2026-09-29',
    '2026-09-30', '2026-10-01', '2026-10-02', '2026-10-03',
    '2026-10-04', '2026-10-05', '2026-10-06'
  ]);
  assert.deepEqual(data.days.map((day) => day.weekday), [
    '周六', '周日', '周一', '周二', '周三', '周四',
    '周五', '周六', '周日', '周一', '周二'
  ]);
});

test('uses the approved city sequence and ten-night allocation', () => {
  assert.deepEqual(data.cities.map((city) => city.id), [
    'milan', 'como', 'florence', 'naples', 'rome'
  ]);
  assert.deepEqual(data.meta.nights, { milan: 3, florence: 3, naples: 1, rome: 3 });
  assert.equal(Object.values(data.meta.nights).reduce((sum, value) => sum + value, 0), 10);
});

test('keeps Lake Como as a day trip and Pompeii out of the active itinerary', () => {
  assert.equal(data.days.find((day) => day.date === '2026-09-28').cityId, 'como');
  assert.equal(data.cities.find((city) => city.id === 'como').stayCityId, 'milan');
  assert.equal(data.days.some((day) => JSON.stringify(day).includes('庞贝')), false);
});

test('leaves all accommodation records visibly unfilled', () => {
  for (const city of data.cities.filter((item) => item.id !== 'como')) {
    assert.deepEqual(city.accommodation, {
      hotel: '待补充', address: '待补充', checkIn: '待补充',
      checkOut: '待补充', confirmation: '待补充'
    });
  }
});

test('preserves booked flight anchors', () => {
  assert.deepEqual(data.flights.outbound.map((leg) => leg.flight), ['HU7396', 'HU7973']);
  assert.deepEqual(data.flights.return.map((leg) => leg.flight), ['HU438', 'HU7393']);
  assert.equal(data.flights.outbound[1].arrival, '2026-09-26 08:20');
  assert.equal(data.flights.return[0].departure, '2026-10-06 11:15');
  assert.equal(data.flights.baggage, '去程深圳提取行李重新托运；回程行李直挂');
});
```

- [ ] **Step 2: Run the data tests and verify the missing module failure**

Run: `npm test`

Expected: FAIL because `../assets/trip-data.js` does not exist.

- [ ] **Step 3: Implement `TripData` using the exact content matrix in Appendix A**

Translate every record in Appendix A into JavaScript object literals assigned to `days`, `cities`, `transport`, `tickets`, and `checklist`. Preserve the exact IDs, dates, tags, statuses, and copy specified there. Finish the file with this universal export so the same object works in a classic browser script and Node tests:

```js
(function (root, factory) {
  var value = factory();
  if (typeof module === 'object' && module.exports) module.exports = value;
  root.TripData = value;
})(typeof globalThis !== 'undefined' ? globalThis : this, function () {
  var pendingStay = function () {
    return { hotel: '待补充', address: '待补充', checkIn: '待补充', checkOut: '待补充', confirmation: '待补充' };
  };

  var value = {
    meta: {
      title: 'In cielo mi portò · 2026 意大利国庆之旅',
      dates: '2026.09.26—10.06',
      route: ['米兰', '科莫湖', '佛罗伦萨', '那不勒斯', '罗马'],
      nights: { milan: 3, florence: 3, naples: 1, rome: 3 }
    },
    flights: {
      outbound: [
        { flight: 'HU7396', date: '2026-09-25', route: '杭州萧山 T3 → 深圳宝安 T3', departure: '2026-09-25 17:50', arrival: '2026-09-25 20:05' },
        { flight: 'HU7973', date: '2026-09-26', route: '深圳宝安 T3 → 米兰 MXP T1', departure: '2026-09-26 01:50', arrival: '2026-09-26 08:20' }
      ],
      return: [
        { flight: 'HU438', date: '2026-10-06', route: '罗马 FCO T3 → 深圳宝安 T3', departure: '2026-10-06 11:15', arrival: '2026-10-07 05:00' },
        { flight: 'HU7393', date: '2026-10-07', route: '深圳宝安 T3 → 杭州萧山 T3', departure: '2026-10-07 08:35', arrival: '2026-10-07 10:40' }
      ],
      baggage: '去程深圳提取行李重新托运；回程行李直挂'
    },
    days: days,
    cities: cities.map(function (city) {
      if (city.id !== 'como') city.accommodation = pendingStay();
      return city;
    }),
    transport: transport,
    tickets: tickets,
    checklist: checklist
  };
  return value;
});
```

- [ ] **Step 4: Run the data tests and verify they pass**

Run: `npm test`

Expected: 5 tests pass, 0 fail.

- [ ] **Step 5: Commit the canonical model**

```bash
git add package.json assets/trip-data.js tests/trip-data.test.js
git commit -m "feat: add canonical 2026 Italy trip data"
```

---

### Task 2: Implement pure navigation, filter, search, and progress state

**Files:**
- Create: `assets/app-core.js`
- Create: `tests/app-core.test.js`

**Interfaces:**
- Consumes: city and day IDs from `TripData`.
- Produces: `TripCore.parseHash(hash)`, `TripCore.buildHash(state)`, `TripCore.filterDays(days, filter)`, `TripCore.search(data, query)`, `TripCore.checklistProgress(groups, saved)`.

- [ ] **Step 1: Write failing pure-function tests**

```js
// tests/app-core.test.js
const test = require('node:test');
const assert = require('node:assert/strict');
const core = require('../assets/app-core.js');
const data = require('../assets/trip-data.js');

test('parses valid tab and city hashes and rejects unknown targets', () => {
  assert.deepEqual(core.parseHash('#tab=cities&city=florence'), { tab: 'cities', city: 'florence' });
  assert.deepEqual(core.parseHash('#tab=tickets'), { tab: 'tickets', city: '' });
  assert.deepEqual(core.parseHash('#tab=wrong&city=wrong'), { tab: 'days', city: '' });
});

test('builds stable URL hashes', () => {
  assert.equal(core.buildHash({ tab: 'cities', city: 'rome' }), '#tab=cities&city=rome');
  assert.equal(core.buildHash({ tab: 'days', city: '' }), '#tab=days');
});

test('filters day cards by transport and reservation', () => {
  assert.deepEqual(core.filterDays(data.days, 'transport').map((day) => day.date), [
    '2026-09-26', '2026-09-28', '2026-09-29', '2026-10-02', '2026-10-03', '2026-10-06'
  ]);
  assert.deepEqual(core.filterDays(data.days, 'reservation').map((day) => day.date), [
    '2026-09-27', '2026-09-30', '2026-10-01', '2026-10-02', '2026-10-04', '2026-10-05'
  ]);
});

test('search returns typed targets across tabs', () => {
  const results = core.search(data, '大卫');
  assert.equal(results.some((item) => item.tab === 'cities' && item.targetId === 'florence'), true);
  assert.equal(results.some((item) => item.tab === 'tickets' && item.targetId === 'ticket-accademia'), true);
  assert.deepEqual(core.search(data, '不存在的内容'), []);
});

test('calculates checklist progress without trusting missing storage values', () => {
  const total = data.checklist.reduce((sum, group) => sum + group.items.length, 0);
  assert.deepEqual(core.checklistProgress(data.checklist, {}), { done: 0, total, percent: 0 });
  const first = data.checklist[0].items[0].id;
  assert.equal(core.checklistProgress(data.checklist, { [first]: true }).done, 1);
});
```

- [ ] **Step 2: Run the core tests and verify the missing module failure**

Run: `node --test tests/app-core.test.js`

Expected: FAIL because `../assets/app-core.js` does not exist.

- [ ] **Step 3: Implement the minimal universal pure-function module**

```js
(function (root, factory) {
  var api = factory();
  if (typeof module === 'object' && module.exports) module.exports = api;
  root.TripCore = api;
})(typeof globalThis !== 'undefined' ? globalThis : this, function () {
  var tabs = ['days', 'cities', 'transport', 'tickets', 'prepare'];
  var cities = ['milan', 'como', 'florence', 'naples', 'rome'];

  function parseHash(hash) {
    var params = new URLSearchParams(String(hash || '').replace(/^#/, ''));
    var tab = params.get('tab');
    var city = params.get('city') || '';
    if (tabs.indexOf(tab) < 0) return { tab: 'days', city: '' };
    if (city && cities.indexOf(city) < 0) city = '';
    return { tab: tab, city: tab === 'cities' ? city : '' };
  }

  function buildHash(state) {
    var hash = '#tab=' + (tabs.indexOf(state.tab) >= 0 ? state.tab : 'days');
    if (state.tab === 'cities' && cities.indexOf(state.city) >= 0) hash += '&city=' + state.city;
    return hash;
  }

  function filterDays(days, filter) {
    if (!filter || filter === 'all') return days.slice();
    return days.filter(function (day) { return day.tags.indexOf(filter) >= 0; });
  }

  function searchable(value) {
    return JSON.stringify(value).toLowerCase();
  }

  function search(data, query) {
    var q = String(query || '').trim().toLowerCase();
    if (!q) return [];
    var found = [];
    data.days.forEach(function (day) {
      if (searchable(day).indexOf(q) >= 0) found.push({ tab: 'days', targetId: 'day-' + day.date, label: day.date + ' · ' + day.city });
    });
    data.cities.forEach(function (city) {
      if (searchable(city).indexOf(q) >= 0) found.push({ tab: 'cities', targetId: city.id, label: city.name });
    });
    data.transport.forEach(function (item) {
      if (searchable(item).indexOf(q) >= 0) found.push({ tab: 'transport', targetId: item.id, label: item.route });
    });
    data.tickets.forEach(function (item) {
      if (searchable(item).indexOf(q) >= 0) found.push({ tab: 'tickets', targetId: item.id, label: item.name });
    });
    return found.slice(0, 12);
  }

  function checklistProgress(groups, saved) {
    var items = groups.reduce(function (all, group) { return all.concat(group.items); }, []);
    var done = items.filter(function (item) { return saved && saved[item.id] === true; }).length;
    return { done: done, total: items.length, percent: items.length ? Math.round(done * 100 / items.length) : 0 };
  }

  return { parseHash: parseHash, buildHash: buildHash, filterDays: filterDays, search: search, checklistProgress: checklistProgress };
});
```

- [ ] **Step 4: Run the core tests and the complete suite**

Run: `npm test`

Expected: 10 tests pass, 0 fail.

- [ ] **Step 5: Commit pure state behavior**

```bash
git add assets/app-core.js tests/app-core.test.js
git commit -m "feat: add itinerary navigation and search state"
```

---

### Task 3: Replace the HTML shell with accessible five-tab structure

**Files:**
- Modify: `index.html`
- Create: `tests/static-site.test.js`

**Interfaces:**
- Consumes: `assets/trip-data.js`, `assets/app-core.js`, `assets/app.js`, `assets/styles.css` by relative URL.
- Produces: `#trip-tabs`, five `[role="tabpanel"]` containers, `#global-search`, `#search-results`, and `#app-status`.

- [ ] **Step 1: Write failing static HTML contract tests**

```js
// tests/static-site.test.js
const test = require('node:test');
const assert = require('node:assert/strict');
const fs = require('node:fs');
const html = fs.readFileSync('index.html', 'utf8');

test('declares exactly five accessible primary tabs and panels', () => {
  assert.equal((html.match(/role="tab"/g) || []).length, 5);
  assert.equal((html.match(/role="tabpanel"/g) || []).length, 5);
  for (const id of ['days', 'cities', 'transport', 'tickets', 'prepare']) {
    assert.match(html, new RegExp('id="tab-' + id + '"'));
    assert.match(html, new RegExp('id="panel-' + id + '"'));
  }
});

test('loads only relative local style and script assets', () => {
  for (const asset of ['assets/styles.css', 'assets/trip-data.js', 'assets/app-core.js', 'assets/app.js']) {
    assert.match(html, new RegExp(asset.replace('.', '\\.') ));
  }
  assert.doesNotMatch(html, /https?:\/\/|file:\/\/|\/Users\//);
});

test('provides search, status, and no-script fallback', () => {
  assert.match(html, /id="global-search"/);
  assert.match(html, /id="search-results"/);
  assert.match(html, /id="app-status"[^>]*aria-live="polite"/);
  assert.match(html, /<noscript>[\s\S]*2026-09-26[\s\S]*2026-10-06[\s\S]*<\/noscript>/);
});
```

- [ ] **Step 2: Run the static test and verify failure against the old page**

Run: `node --test tests/static-site.test.js`

Expected: FAIL because the current page does not define five ARIA tabs and five panels.

- [ ] **Step 3: Replace `index.html` with the semantic shell**

The page must include this exact structural contract:

```html
<header class="hero">
  <p class="eyebrow">ITALIA · 2026</p>
  <h1>把十一天，走成一条清晰的线</h1>
  <p>米兰进，罗马出 · 2026.09.26—10.06 · 4 城 10 晚 + 科莫湖一日</p>
  <div id="route-summary" aria-label="旅行路线"></div>
</header>
<main>
  <section class="toolbar" aria-label="攻略导航与搜索">
    <div id="trip-tabs" class="tabs" role="tablist" aria-label="攻略栏目">
      <button id="tab-days" role="tab" aria-controls="panel-days" aria-selected="true" data-tab="days">每日行程</button>
      <button id="tab-cities" role="tab" aria-controls="panel-cities" aria-selected="false" data-tab="cities" tabindex="-1">城市信息</button>
      <button id="tab-transport" role="tab" aria-controls="panel-transport" aria-selected="false" data-tab="transport" tabindex="-1">交通信息</button>
      <button id="tab-tickets" role="tab" aria-controls="panel-tickets" aria-selected="false" data-tab="tickets" tabindex="-1">门票预订</button>
      <button id="tab-prepare" role="tab" aria-controls="panel-prepare" aria-selected="false" data-tab="prepare" tabindex="-1">行前准备</button>
    </div>
    <label class="search"><span>搜索攻略</span><input id="global-search" type="search" autocomplete="off" placeholder="日期、城市、景点或车站"></label>
    <div id="search-results" hidden></div>
  </section>
  <section id="panel-days" role="tabpanel" aria-labelledby="tab-days"><div id="day-filters"></div><div id="days-view"></div></section>
  <section id="panel-cities" role="tabpanel" aria-labelledby="tab-cities" hidden><div id="cities-view"></div></section>
  <section id="panel-transport" role="tabpanel" aria-labelledby="tab-transport" hidden><div id="transport-view"></div></section>
  <section id="panel-tickets" role="tabpanel" aria-labelledby="tab-tickets" hidden><div id="tickets-view"></div></section>
  <section id="panel-prepare" role="tabpanel" aria-labelledby="tab-prepare" hidden><div id="prepare-view"></div></section>
  <p id="app-status" class="sr-only" aria-live="polite"></p>
</main>
<noscript><section class="noscript"><h2>每日行程</h2><p>2026-09-26 米兰抵达；2026-09-27 米兰；2026-09-28 科莫湖；2026-09-29 至 2026-10-01 佛罗伦萨；2026-10-02 那不勒斯；2026-10-03 至 2026-10-05 罗马；2026-10-06 罗马返程。</p></section></noscript>
```

Load `styles.css` in `<head>` and the three scripts, in data/core/app order, immediately before `</body>` using non-module `<script src="…"></script>` tags.

- [ ] **Step 4: Run static and existing tests**

Run: `npm test`

Expected: 13 tests pass, 0 fail.

- [ ] **Step 5: Commit the accessible shell**

```bash
git add index.html tests/static-site.test.js
git commit -m "feat: add five-tab itinerary shell"
```

---

### Task 4: Render all panels and wire cross-tab navigation

**Files:**
- Create: `assets/app.js`
- Modify: `tests/static-site.test.js`

**Interfaces:**
- Consumes: `window.TripData`, `window.TripCore`, and all DOM IDs from Task 3.
- Produces: day cards `#day-YYYY-MM-DD`, city cards `#city-{id}`, transport cards, ticket cards, checklist groups, and delegated actions using `data-action`.

- [ ] **Step 1: Add failing static behavior-contract tests**

```js
const app = fs.readFileSync('assets/app.js', 'utf8');

test('app renders every view and uses delegated actions', () => {
  for (const fn of ['renderRoute', 'renderDays', 'renderCities', 'renderTransport', 'renderTickets', 'renderChecklist']) {
    assert.match(app, new RegExp('function ' + fn + '\\('));
  }
  for (const action of ['open-city', 'set-filter', 'toggle-check']) {
    assert.match(app, new RegExp("data-action=['\"]" + action));
  }
});

test('city navigation updates tab, hash, expansion, focus, and status', () => {
  assert.match(app, /core\.buildHash\(\{ tab: 'cities', city: cityId \}\)/);
  assert.match(app, /details\.open = true/);
  assert.match(app, /scrollIntoView/);
  assert.match(app, /app-status/);
});
```

- [ ] **Step 2: Run the static test and verify the missing app failure**

Run: `node --test tests/static-site.test.js`

Expected: FAIL because `assets/app.js` does not exist.

- [ ] **Step 3: Implement safe rendering helpers and all five renderers**

Use DOM APIs or an `escapeHtml` helper for all data-derived strings:

```js
(function () {
  'use strict';
  var data = window.TripData;
  var core = window.TripCore;
  var activeFilter = 'all';
  var savedChecks = {};

  function escapeHtml(value) {
    return String(value).replace(/[&<>'"]/g, function (char) {
      return { '&': '&amp;', '<': '&lt;', '>': '&gt;', "'": '&#39;', '"': '&quot;' }[char];
    });
  }

  function renderRoute() {
    document.getElementById('route-summary').innerHTML = data.meta.route.map(function (name, index) {
      return '<span class="route-stop">' + escapeHtml(name) + '</span>' + (index < data.meta.route.length - 1 ? '<span aria-hidden="true">→</span>' : '');
    }).join('');
  }

  function renderDays() {
    document.getElementById('day-filters').innerHTML = [
      ['all', '全部'], ['city', '城市游览'], ['transport', '跨城交通'], ['reservation', '预约项目']
    ].map(function (item) {
      return '<button data-action="set-filter" data-filter="' + item[0] + '" aria-pressed="' + String(activeFilter === item[0]) + '">' + item[1] + '</button>';
    }).join('');
    document.getElementById('days-view').innerHTML = core.filterDays(data.days, activeFilter).map(function (day) {
      var events = day.events.map(function (event) {
        return '<li><time>' + escapeHtml(event.time) + '</time><div><strong>' + escapeHtml(event.title) + '</strong><p>' + escapeHtml(event.detail) + '</p></div></li>';
      }).join('');
      return '<article class="day-card" id="day-' + day.date + '"><header><p>' + escapeHtml(day.date + ' · ' + day.weekday) + '</p><h2>' + escapeHtml(day.theme) + '</h2><button data-action="open-city" data-city="' + day.cityId + '">' + escapeHtml(day.city) + '城市信息</button></header><ol class="timeline">' + events + '</ol><div class="day-notes"><p><strong>用餐：</strong>' + escapeHtml(day.meal) + '</p><p><strong>住宿：</strong>' + escapeHtml(day.stay) + '</p><p><strong>注意：</strong>' + escapeHtml(day.alerts) + '</p></div></article>';
    }).join('');
  }

  function renderCities() {
    document.getElementById('cities-view').innerHTML = data.cities.map(function (city) {
      var sections = city.sections.map(function (section, index) {
        return '<details' + (index === 0 ? ' open' : '') + '><summary>' + escapeHtml(section.title) + '<span>' + escapeHtml(section.readTime) + '</span></summary>' + section.paragraphs.map(function (paragraph) { return '<p>' + escapeHtml(paragraph) + '</p>'; }).join('') + (section.bullets.length ? '<ul>' + section.bullets.map(function (bullet) { return '<li>' + escapeHtml(bullet) + '</li>'; }).join('') + '</ul>' : '') + '</details>';
      }).join('');
      var stay = city.id === 'como' ? '<p>当晚返回米兰，住宿见米兰。</p>' : '<dl class="stay-grid"><div><dt>酒店</dt><dd>' + escapeHtml(city.accommodation.hotel) + '</dd></div><div><dt>地址</dt><dd>' + escapeHtml(city.accommodation.address) + '</dd></div><div><dt>入住</dt><dd>' + escapeHtml(city.accommodation.checkIn) + '</dd></div><div><dt>退房</dt><dd>' + escapeHtml(city.accommodation.checkOut) + '</dd></div><div><dt>预订号</dt><dd>' + escapeHtml(city.accommodation.confirmation) + '</dd></div></dl>';
      return '<article class="city-card" id="city-' + city.id + '" data-city-name="' + escapeHtml(city.name) + '"><header><p>' + escapeHtml(city.dates + ' · ' + city.nightsLabel) + '</p><h2 tabindex="-1">' + escapeHtml(city.name) + '</h2><p>' + escapeHtml(city.summary) + '</p><ul class="chips">' + city.highlights.map(function (item) { return '<li>' + escapeHtml(item) + '</li>'; }).join('') + '</ul></header>' + sections + '<details><summary>住宿信息</summary>' + stay + '</details></article>';
    }).join('');
  }

  function renderTransport() {
    var flights = '<section class="flight-summary"><h2>已订航班</h2><p>' + escapeHtml(data.flights.baggage) + '</p>' + data.flights.outbound.concat(data.flights.return).map(function (leg) {
      return '<article><strong>' + escapeHtml(leg.flight) + '</strong><span>' + escapeHtml(leg.route) + '</span><time>' + escapeHtml(leg.departure + ' → ' + leg.arrival) + '</time></article>';
    }).join('') + '</section>';
    document.getElementById('transport-view').innerHTML = flights + data.transport.map(function (item) {
      return '<article class="transport-card" id="' + item.id + '"><p class="status">' + escapeHtml(item.status) + '</p><h2>' + escapeHtml(item.route) + '</h2><dl><div><dt>日期</dt><dd>' + escapeHtml(item.date) + '</dd></div><div><dt>方式</dt><dd>' + escapeHtml(item.mode) + '</dd></div><div><dt>车站</dt><dd>' + escapeHtml(item.from + ' → ' + item.to) + '</dd></div><div><dt>建议窗口</dt><dd>' + escapeHtml(item.window) + '</dd></div><div><dt>预计耗时</dt><dd>' + escapeHtml(item.duration) + '</dd></div></dl><p>' + escapeHtml(item.notes) + '</p></article>';
    }).join('');
  }

  function renderTickets() {
    document.getElementById('tickets-view').innerHTML = data.tickets.map(function (item) {
      return '<article class="ticket-card" id="' + item.id + '"><p class="status">' + escapeHtml(item.priority + ' · ' + item.status) + '</p><h2>' + escapeHtml(item.name) + '</h2><p>' + escapeHtml(item.date + ' · ' + item.suggestedTime) + '</p><p><strong>预约：</strong>' + escapeHtml(item.reservation) + '</p><p>' + escapeHtml(item.caveat) + '</p><a href="' + escapeHtml(item.officialUrl) + '" target="_blank" rel="noopener noreferrer">官方入口</a></article>';
    }).join('');
  }

  function renderChecklist() {
    var progress = core.checklistProgress(data.checklist, savedChecks);
    document.getElementById('prepare-view').innerHTML = '<header class="progress"><h2>行前完成度</h2><p>' + progress.done + ' / ' + progress.total + ' · ' + progress.percent + '%</p><div role="progressbar" aria-valuemin="0" aria-valuemax="100" aria-valuenow="' + progress.percent + '"><span style="width:' + progress.percent + '%"></span></div></header>' + data.checklist.map(function (group) {
      return '<section class="check-group"><h2>' + escapeHtml(group.title) + '</h2>' + group.items.map(function (item) {
        return '<label><input type="checkbox" data-action="toggle-check" data-check-id="' + item.id + '"' + (savedChecks[item.id] ? ' checked' : '') + '><span>' + escapeHtml(item.label) + '</span></label>';
      }).join('') + '</section>';
    }).join('');
  }

  function renderAll() {
    renderRoute(); renderDays(); renderCities(); renderTransport(); renderTickets(); renderChecklist();
  }
  renderAll();
})();
```

Use native `<details>` for city sections and ordinary `<a target="_blank" rel="noopener noreferrer">` only for official external ticket pages.

- [ ] **Step 4: Implement one event delegation path and URL state application**

Add these concrete behaviors inside the same IIFE:

```js
function activateTab(tabId, focusTab) {
  document.querySelectorAll('[role="tab"]').forEach(function (tab) {
    var active = tab.dataset.tab === tabId;
    tab.setAttribute('aria-selected', String(active));
    tab.tabIndex = active ? 0 : -1;
    document.getElementById(tab.getAttribute('aria-controls')).hidden = !active;
    if (active && focusTab) tab.focus();
  });
}

function announce(message) {
  document.getElementById('app-status').textContent = message;
}

function reducedMotion() {
  return window.matchMedia && window.matchMedia('(prefers-reduced-motion: reduce)').matches;
}

function openCity(cityId) {
  activateTab('cities', false);
  history.pushState(null, '', core.buildHash({ tab: 'cities', city: cityId }));
  var card = document.getElementById('city-' + cityId);
  if (!card) return announce('城市信息未找到，已打开城市信息首页');
  var details = card.querySelector('details');
  if (details) details.open = true;
  card.classList.remove('is-target');
  void card.offsetWidth;
  card.classList.add('is-target');
  card.scrollIntoView({ behavior: reducedMotion() ? 'auto' : 'smooth', block: 'start' });
  card.querySelector('h2').setAttribute('tabindex', '-1');
  card.querySelector('h2').focus({ preventScroll: true });
  announce('已打开' + card.dataset.cityName + '城市信息');
}

function openCityFromHistory(cityId) {
  var card = document.getElementById('city-' + cityId);
  if (!card) return announce('城市信息未找到，已打开城市信息首页');
  var details = card.querySelector('details');
  if (details) details.open = true;
  card.scrollIntoView({ behavior: 'auto', block: 'start' });
}

function applyHash() {
  var state = core.parseHash(location.hash);
  activateTab(state.tab, false);
  if (state.city) openCityFromHistory(state.city);
}

document.addEventListener('click', function (event) {
  var control = event.target.closest('[data-action], [data-tab]');
  if (!control) return;
  if (control.dataset.tab) {
    activateTab(control.dataset.tab, false);
    history.pushState(null, '', core.buildHash({ tab: control.dataset.tab, city: '' }));
    return;
  }
  if (control.dataset.action === 'open-city') openCity(control.dataset.city);
  if (control.dataset.action === 'set-filter') {
    activeFilter = control.dataset.filter;
    renderDays();
  }
  if (control.dataset.action === 'toggle-check') {
    savedChecks[control.dataset.checkId] = control.checked;
    renderChecklist();
  }
});

document.getElementById('trip-tabs').addEventListener('keydown', function (event) {
  var keys = ['ArrowLeft', 'ArrowRight', 'Home', 'End'];
  if (keys.indexOf(event.key) < 0) return;
  event.preventDefault();
  var tabs = Array.from(this.querySelectorAll('[role="tab"]'));
  var index = tabs.indexOf(document.activeElement);
  if (event.key === 'Home') index = 0;
  if (event.key === 'End') index = tabs.length - 1;
  if (event.key === 'ArrowLeft') index = (index - 1 + tabs.length) % tabs.length;
  if (event.key === 'ArrowRight') index = (index + 1) % tabs.length;
  tabs[index].click();
  tabs[index].focus();
});

window.addEventListener('hashchange', applyHash);
applyHash();
```

Keep all of these functions and listeners inside the IIFE, after the render functions and before its closing `})();`.

- [ ] **Step 5: Run the complete suite**

Run: `npm test`

Expected: all tests pass with no failed assertions.

- [ ] **Step 6: Commit rendering and navigation**

```bash
git add assets/app.js tests/static-site.test.js
git commit -m "feat: render itinerary tabs and city deep links"
```

---

### Task 5: Complete search and resilient checklist persistence

**Files:**
- Modify: `assets/app.js`
- Modify: `tests/static-site.test.js`

**Interfaces:**
- Consumes: `TripCore.search`, `TripCore.checklistProgress`, data target IDs, and `localStorage` when available.
- Produces: keyboard-accessible result list and storage key `italy-2026-checklist-v1`.

- [ ] **Step 1: Add failing resilience and search contract tests**

```js
test('search supports empty state, clear action, and typed navigation', () => {
  assert.match(app, /TripCore\.search\(data, query\)/);
  assert.match(app, /没有找到相关内容/);
  assert.match(app, /data-action="clear-search"/);
  assert.match(app, /result\.tab/);
  assert.match(app, /result\.targetId/);
  assert.match(app, /data-action="open-search-result"/);
});

test('checklist storage failures degrade without breaking interaction', () => {
  assert.match(app, /italy-2026-checklist-v1/);
  assert.match(app, /function loadChecks\(\)[\s\S]*try[\s\S]*catch/);
  assert.match(app, /function saveChecks\(checks\)[\s\S]*try[\s\S]*catch/);
  assert.match(app, /TripCore\.checklistProgress/);
});
```

- [ ] **Step 2: Run tests and verify they fail for missing behaviors**

Run: `node --test tests/static-site.test.js`

Expected: FAIL on search empty-state and/or resilient storage assertions.

- [ ] **Step 3: Implement search and storage fallbacks**

```js
var STORAGE_KEY = 'italy-2026-checklist-v1';

function loadChecks() {
  try { return JSON.parse(localStorage.getItem(STORAGE_KEY) || '{}'); }
  catch (error) { return {}; }
}

savedChecks = loadChecks();

function saveChecks(checks) {
  try { localStorage.setItem(STORAGE_KEY, JSON.stringify(checks)); return true; }
  catch (error) { announce('清单已更新，但当前浏览器无法永久保存'); return false; }
}

function updateSearch(query) {
  var results = core.search(data, query);
  var box = document.getElementById('search-results');
  if (!query.trim()) { box.hidden = true; box.innerHTML = ''; return; }
  box.hidden = false;
  box.innerHTML = results.length
    ? results.map(function (result) {
        return '<button data-action="open-search-result" data-tab="' + escapeHtml(result.tab) + '" data-target="' + escapeHtml(result.targetId) + '">' + escapeHtml(result.label) + '</button>';
      }).join('')
    : '<p>没有找到相关内容</p><button data-action="clear-search">清除搜索</button>';
}

document.getElementById('global-search').addEventListener('input', function (event) {
  updateSearch(event.target.value);
});

document.getElementById('global-search').addEventListener('keydown', function (event) {
  if (event.key === 'Escape') {
    document.getElementById('search-results').hidden = true;
    event.target.focus();
  }
});
```

Replace its original `toggle-check` branch with the first branch below, then append the two search branches:

```js
if (control.dataset.action === 'toggle-check') {
  savedChecks[control.dataset.checkId] = control.checked;
  saveChecks(savedChecks);
  renderChecklist();
}
if (control.dataset.action === 'clear-search') {
  document.getElementById('global-search').value = '';
  updateSearch('');
  document.getElementById('global-search').focus();
}
if (control.dataset.action === 'open-search-result') {
  var tabId = control.dataset.tab;
  var targetId = control.dataset.target;
  activateTab(tabId, false);
  history.pushState(null, '', core.buildHash({ tab: tabId, city: tabId === 'cities' ? targetId : '' }));
  var target = document.getElementById(tabId === 'cities' ? 'city-' + targetId : targetId);
  if (target) {
    var parentDetails = target.closest('details');
    if (parentDetails) parentDetails.open = true;
    target.scrollIntoView({ behavior: reducedMotion() ? 'auto' : 'smooth', block: 'start' });
    var heading = target.querySelector('h2, h3') || target;
    heading.setAttribute('tabindex', '-1');
    heading.focus({ preventScroll: true });
    announce('已定位到' + control.textContent);
  }
  document.getElementById('search-results').hidden = true;
}
```

- [ ] **Step 4: Run the full suite**

Run: `npm test`

Expected: all tests pass.

- [ ] **Step 5: Commit search and checklist behavior**

```bash
git add assets/app.js tests/static-site.test.js
git commit -m "feat: add guide search and resilient checklist"
```

---

### Task 6: Apply the responsive travel-journal visual system

**Files:**
- Create: `assets/styles.css`
- Modify: `tests/static-site.test.js`

**Interfaces:**
- Consumes: semantic classes and state attributes produced by `index.html` and `app.js`.
- Produces: usable desktop, tablet, and 360px mobile layouts with semantic status colors and reduced motion.

- [ ] **Step 1: Add failing CSS contract tests**

```js
const css = fs.readFileSync('assets/styles.css', 'utf8');

test('CSS defines the approved palette and visible focus treatment', () => {
  for (const token of ['--ink', '--paper', '--forest', '--terracotta', '--gold', '--danger']) {
    assert.match(css, new RegExp(token + ':'));
  }
  assert.match(css, /:focus-visible/);
  assert.match(css, /outline:/);
});

test('CSS supports mobile cards, sticky tabs, and reduced motion', () => {
  assert.match(css, /position:\s*sticky/);
  assert.match(css, /@media\s*\(max-width:\s*720px\)/);
  assert.match(css, /overflow-x:\s*auto/);
  assert.match(css, /@media\s*\(prefers-reduced-motion:\s*reduce\)/);
});

test('CSS prevents page-level overflow and keeps controls touchable', () => {
  assert.match(css, /max-width:\s*100%/);
  assert.match(css, /min-height:\s*44px/);
  assert.match(css, /overflow-wrap:\s*anywhere/);
});
```

- [ ] **Step 2: Run tests and verify failure because the stylesheet is absent**

Run: `node --test tests/static-site.test.js`

Expected: FAIL because `assets/styles.css` does not exist.

- [ ] **Step 3: Implement the palette and layout primitives**

Start with these exact tokens and global constraints:

```css
:root {
  --ink: #18312b;
  --muted: #63716d;
  --paper: #f7f2e8;
  --surface: #fffdf8;
  --forest: #1f5a4a;
  --forest-soft: #dcebe4;
  --terracotta: #bd6548;
  --terracotta-soft: #f4e1d8;
  --gold: #b38a3e;
  --danger: #a43f3f;
  --line: #ded7c9;
  --shadow: 0 16px 44px rgba(41, 50, 45, .09);
  --radius: 18px;
}
*, *::before, *::after { box-sizing: border-box; }
html { scroll-behavior: smooth; background: var(--paper); }
body { margin: 0; color: var(--ink); background: var(--paper); font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", "PingFang SC", "Hiragino Sans GB", sans-serif; overflow-wrap: anywhere; }
img, svg, table { max-width: 100%; }
button, input, summary, a { min-height: 44px; }
:focus-visible { outline: 3px solid var(--terracotta); outline-offset: 3px; }
.sr-only { position: absolute; width: 1px; height: 1px; padding: 0; margin: -1px; overflow: hidden; clip: rect(0, 0, 0, 0); white-space: nowrap; border: 0; }
```

Implement a 1180px maximum content width, spacious hero, sticky `.toolbar`, horizontally scrollable `.tabs`, two-column day cards on wide screens, timeline rows, city-section `details`, transport/ticket grids, progress bar, status chips, `.is-target` highlight, empty search result, and print rules. Use the palette consistently; danger and reservation states must include labels or icons in markup rather than color alone.

- [ ] **Step 4: Add the mobile and motion rules**

```css
@media (max-width: 720px) {
  .hero, main { padding-inline: 16px; }
  .tabs { overflow-x: auto; scrollbar-width: thin; scroll-snap-type: x proximity; }
  .tabs [role="tab"] { flex: 0 0 auto; scroll-snap-align: start; }
  .day-card, .city-card, .transport-card, .ticket-card { border-radius: 14px; }
  .day-layout, .city-overview, .transport-grid, .ticket-grid { grid-template-columns: 1fr; }
  table, thead, tbody, tr, th, td { display: block; width: 100%; }
  thead { position: absolute; width: 1px; height: 1px; overflow: hidden; }
}

@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after { scroll-behavior: auto !important; transition-duration: .01ms !important; animation-duration: .01ms !important; animation-iteration-count: 1 !important; }
}
```

- [ ] **Step 5: Run the full suite and inspect the generated page at desktop and mobile widths**

Run: `npm test`

Expected: all tests pass.

Then open the page through the available local preview surface. Verify 1440×900 and 390×844 layouts, tab overflow, 44px targets, no page-level horizontal overflow, focus visibility, accordion readability, and route/card hierarchy. If the in-app Browser still blocks local-file access, use the repository's permitted static preview route or report the visual QA limitation; do not bypass browser security policy.

- [ ] **Step 6: Commit the visual system**

```bash
git add assets/styles.css tests/static-site.test.js
git commit -m "style: add responsive Italy travel journal UI"
```

---

### Task 7: Verify official links, facts, accessibility, and release readiness

**Files:**
- Modify: `assets/trip-data.js`
- Modify: `tests/trip-data.test.js`
- Modify: `tests/static-site.test.js`

**Interfaces:**
- Consumes: complete site from Tasks 1–6 and current official attraction/transport sources.
- Produces: reviewed `officialUrl`, `caveat`, and `status` values plus a passing release checklist.

- [ ] **Step 1: Add failing official-link and release-contract tests**

```js
test('every reservable attraction has an HTTPS official URL and caveat', () => {
  for (const ticket of data.tickets.filter((item) => item.reservation !== '无需预约')) {
    assert.match(ticket.officialUrl, /^https:\/\//);
    assert.ok(ticket.caveat.length >= 8);
    assert.ok(['优先锁定', '计划预订', '出发前复核'].includes(ticket.status));
  }
});

test('unconfirmed transport never looks like a purchased ticket', () => {
  for (const item of data.transport) {
    if (!item.booked) assert.match(item.status, /建议|现场|复核/);
  }
});
```

```js
test('site contains no forbidden external runtime dependency', () => {
  const files = ['index.html', 'assets/styles.css', 'assets/trip-data.js', 'assets/app-core.js', 'assets/app.js'];
  const source = files.map((file) => fs.readFileSync(file, 'utf8')).join('\n');
  assert.doesNotMatch(source, /cdn\.|unpkg|jsdelivr|fonts\.googleapis|file:\/\/|\/Users\//i);
});
```

- [ ] **Step 2: Run the full suite and verify new tests expose missing or weak records**

Run: `npm test`

Expected: FAIL for any ticket without an official HTTPS URL/caveat or transport record without an uncertainty status.

- [ ] **Step 3: Check each unstable fact against first-party sources and update the data**

Verify only current, primary sources for:

- Cenacolo Vinciano / official ticket partner for 《最后的晚餐》;
- Duomo Milano for rooftop access;
- Navigazione Laghi for Lake Como ferry schedules and seasonal caveat;
- Uffizi, Galleria dell’Accademia, and Opera del Duomo Firenze;
- Museo Cappella Sansevero;
- Parco archeologico del Colosseo, including first-Sunday policy for 2026-10-04;
- Musei Vaticani and Basilica di San Pietro;
- Trenitalia/Italo for route conventions and Leonardo Express.

Do not add a precise 2026 timetable or price unless the official source currently publishes it for the travel date. Otherwise use `出发前复核`, a duration range, and a recommended departure window.

- [ ] **Step 4: Run automated verification**

Run: `npm test`

Expected: all tests pass, 0 fail.

Run: `git diff --check`

Expected: exit 0 with no whitespace errors.

Run: `git status --short`

Expected: only the files intentionally changed in this task are listed; preserve unrelated `.DS_Store` files without staging them.

- [ ] **Step 5: Perform functional and accessibility smoke checks**

Verify in the local preview:

1. Activate all five tabs by pointer and keyboard.
2. From 9/26, 9/28, 9/29, 10/2, and 10/3 day cards, open Milan, Como, Florence, Naples, and Rome city details.
3. Load `#tab=cities&city=florence`, `#tab=tickets`, and an invalid `#tab=wrong&city=wrong` directly.
4. Search `大卫`, `FCO`, and a nonexistent phrase; open a result and clear the empty state.
5. Toggle a checklist item, reload, and verify persistence; repeat with storage disabled if the preview supports it.
6. Confirm that every accommodation field reads `待补充`.
7. Check 1440×900 and 390×844 with no page-level horizontal overflow.
8. Inspect console output and confirm zero uncaught errors.

- [ ] **Step 6: Commit the verified content**

```bash
git add assets/trip-data.js tests/trip-data.test.js tests/static-site.test.js
git commit -m "chore: verify itinerary facts and release contracts"
```

---

## Appendix A: Exact Content Matrix

### Days

Each day object has `{ date, weekday, cityId, city, theme, tags, events, meal, stay, alerts }`. Each event has `{ time, title, detail, kind }`.

1. `2026-09-26`, 周六, `milan`, 米兰, “抵达与第一眼米兰”, tags `transport`, `city`; events: `08:20` MXP T1 抵达、入境和取行李；`10:30–12:00` Malpensa Express 前往市区并寄存行李；`14:00–17:30` 大教堂外观、埃马努埃莱二世拱廊、布雷拉漫步；`18:00` 提前晚餐并休息。Meal: 番红花烩饭或米兰炸小牛排。Stay: 米兰，待补充。Alert: 抵达日不安排必须预约项目。
2. `2026-09-27`, 周日, `milan`, 米兰, “达芬奇与城市地标”, tags `city`, `reservation`; events: `08:45` 提前抵达圣母感恩教堂；`09:00` 《最后的晚餐》建议预约窗口；`11:00–13:00` 大教堂天台及内部，宗教活动可能影响内部参观；`14:30–17:30` 斯福尔扎城堡和森皮奥内公园；`18:30` Navigli aperitivo。Stay: 米兰，待补充。Alert: 《最后的晚餐》以抢到的实际票面时间为准，全日围绕它调整。
3. `2026-09-28`, 周一, `como`, 科莫湖, “湖光与贝拉吉奥”, tags `transport`, `city`; events: `07:45` 前往 Milano Centrale；`08:30–09:30` 建议窗口乘区域火车至 Como S. Giovanni；`09:45–11:00` 科莫湖滨和老城；`11:00–12:30` 渡船前往贝拉吉奥，具体班次出发前复核；`12:30–16:00` 贝拉吉奥午餐、石阶小巷和 Villa Melzi 可选；`16:00–19:00` 渡船及火车返回米兰。Meal: 湖鱼烩饭。Stay: 米兰，待补充。Alert: 船班受季节和天气影响；恶劣天气改为米兰布雷拉美术馆、咖啡馆和购物。
4. `2026-09-29`, 周二, `florence`, 佛罗伦萨, “抵达文艺复兴之城”, tags `transport`, `city`; events: `08:30–10:00` 建议窗口从 Milano Centrale 乘高铁至 Firenze SMN；`12:00` 寄存行李和午餐；`13:30–17:00` 百花大教堂外观、洗礼堂、领主广场、老桥；`17:30–19:30` 米开朗基罗广场日落。Meal: 奥尔特拉诺小馆。Stay: 佛罗伦萨，待补充。Alert: 高铁为建议窗口，非已购票。
5. `2026-09-30`, 周三, `florence`, 佛罗伦萨, “乌菲兹与美第奇”, tags `city`, `reservation`; events: `08:30` 乌菲兹预约入场，约 3 小时；`12:00` 奥尔特拉诺午餐；`13:30–16:30` 皮蒂宫或波波里花园二选一；`17:00` 阿诺河与老桥黄昏。Meal: 牛肚包或托斯卡纳家常菜。Stay: 佛罗伦萨，待补充。Alert: 下午项目二选一，不同时承诺。
6. `2026-10-01`, 周四, `florence`, 佛罗伦萨, “大卫、穹顶与大师长眠处”, tags `city`, `reservation`; events: `08:30` 学院美术馆《大卫》；`10:30–13:00` 百花大教堂穹顶预约登顶；`14:30–16:00` 圣十字大殿；`16:30` 圣米尼亚托教堂仅在体力允许时增加。Meal: 佛罗伦萨牛排，两人分食并提前预约餐厅。Stay: 佛罗伦萨，待补充。Alert: 穹顶 463 级台阶且无电梯，穿防滑舒适鞋。
7. `2026-10-02`, 周五, `naples`, 那不勒斯, “火山下的老城与海湾”, tags `transport`, `city`, `reservation`; events: `08:30–09:00` 建议窗口从 Firenze SMN 乘高铁；`11:30–12:00` 抵达 Napoli Centrale、寄存行李；`12:30` Da Michele 或 Sorbillo 披萨；`14:00–17:00` Spaccanapoli、圣诞工坊街、圣塞维诺礼拜堂预约；`17:30–19:30` 蛋堡外观和海湾日落。Meal: 披萨午餐、海鲜或蛤蜊意面晚餐。Stay: 那不勒斯，待补充。Alert: 一晚行程不安排庞贝；老城和车站注意扒手与摩托车。
8. `2026-10-03`, 周六, `rome`, 罗马, “抵达与巴洛克漫游”, tags `transport`, `city`; events: `08:30–09:30` 建议窗口从 Napoli Centrale 乘高铁；`10:30–12:00` 抵达 Roma Termini、寄存行李；`13:30–18:00` 万神殿、纳沃纳广场、鲜花广场、威尼斯广场；`20:30` 特雷维喷泉夜景。Meal: Trastevere 或历史中心外缘的 Carbonara。Stay: 罗马，待补充。Alert: 万神殿开放和礼拜安排出发前复核。
9. `2026-10-04`, 周日, `rome`, 罗马, “古罗马遗址日”, tags `city`, `reservation`; events: `08:00` 提前抵达斗兽场区域；`08:30–11:30` 斗兽场；`12:00` Monti 或 Testaccio 午餐；`13:30–17:30` 古罗马广场和帕拉丁山。Meal: Cacio e Pepe。Stay: 罗马，待补充。Alert: 2026-10-04 为首个周日，免费政策、预约方式和人流须以官方当期公告为准。
10. `2026-10-05`, 周一, `rome`, 罗马/梵蒂冈, “梵蒂冈艺术与信仰”, tags `city`, `reservation`; events: `08:00` 梵蒂冈博物馆预约窗口；`08:00–12:30` 地图廊、拉斐尔画室、西斯廷礼拜堂；`12:45` Prati 午餐；`14:00–17:00` 圣彼得广场和圣彼得大教堂，穹顶仅在体力和排队允许时增加。Meal: Prati 最后一晚晚餐。Stay: 罗马，待补充。Alert: 教堂遮肩遮膝，安检排队；西斯廷礼拜堂禁止拍照。
11. `2026-10-06`, 周二, `rome`, 罗马返程, “前往 FCO”, tags `transport`; events: `06:45` 退房；`07:15–07:30` 从酒店前往 Termini 或直接叫车；`08:00–08:45` Leonardo Express 或固定价出租车前往 FCO；`11:15` HU438 从 FCO T3 起飞。Meal: 提前准备简便早餐。Stay: 无。Alert: 国际航班至少提前 3 小时到机场，并按实际酒店位置选择交通。

### Cities

Each city has `{ id, name, dates, nightsLabel, stayCityId, summary, highlights, sections }`; each section has `{ id, title, readTime, paragraphs, bullets }`.

- Milan: summary “把帝国、达芬奇与现代设计穿在同一件衣服上的城市。” Highlights: 米兰大教堂、《最后的晚餐》、斯福尔扎城堡、Navigli. Sections: `story` describes Mediolanum and the 313 Edict of Milan, the Visconti/Sforza era, Leonardo’s 17 Milan years, and modern fashion/economy; `sights` explains the Duomo’s long construction and rooftop, the mural’s original refectory setting and fragile timed visit, the castle’s Rondanini Pietà, and the Galleria; `food` covers risotto alla Milanese, cotoletta, standing coffee, aperitivo; `practical` covers metro plus walking, Centrale/metro pickpockets, fake bracelets, 112; `stay` renders the five pending fields.
- Como: summary “从米兰出发，用火车、船和步行串起阿尔卑斯山脚的湖湾。” Highlights: 科莫湖滨、渡船、贝拉吉奥、Villa Melzi. Sections: `story` explains the glacial Y-shaped lake and historic villa culture; `sights` explains Como waterfront, ferry panorama, Bellagio lanes, Villa Melzi as optional; `food` covers lake-fish risotto and avoiding time-consuming formal lunch; `practical` covers regional train, seasonal ferry, weather fallback, return buffer; `stay` states “当晚返回米兰，住宿见米兰”。
- Florence: summary “在一个步行圈里，看见美第奇如何把银行财富变成文艺复兴。” Highlights: 乌菲兹、《大卫》、百花大教堂、老桥. Sections: `story` describes guild republic wealth, Medici patronage, and Oltrarno crafts; `sights` explains Botticelli/Leonardo/Raphael in Uffizi, David and Prisoners in Accademia, Brunelleschi’s double-shell dome and 463 steps, Santa Croce tombs, Ponte Vecchio and Vasari Corridor; `food` covers bistecca served rare and shared, lampredotto, flat-pan gelato; `practical` covers walkability, stone streets, church dress, SMN pickpockets; `stay` renders pending fields.
- Naples: summary “希腊街道、巴洛克密度、火山阴影与披萨共同构成的南意大利现场。” Highlights: Spaccanapoli、《蒙纱的基督》、披萨、蛋堡海湾. Sections: `story` describes Greek Neapolis street grid, Roman bay, successive kingdoms, and living UNESCO old town; `sights` explains the ancient street line, Sansevero marble veil, Christmas workshop street, Egg Castle legend and bay view; `food` covers Margherita/Marinara, Da Michele/Sorbillo as alternatives, sfogliatella, seafood; `practical` covers walking/taxi/Metro Line 1, station and old-town theft, scooters and keeping bags forward; `next-time` names Pompeii and MANN explicitly as future-trip options, not active itinerary; `stay` renders pending fields.
- Rome: summary “古罗马的权力、梵蒂冈的信仰与巴洛克的城市剧场在三天内逐层展开。” Highlights: 斗兽场、古罗马广场、梵蒂冈、万神殿. Sections: `story` explains republic/empire public space, papal Rome, Renaissance and Baroque urban theatre; `sights` explains Colosseum/Forum/Palatine as one archaeological day, Vatican Museums’ Maps/Raphael/Sistine route, St Peter’s Pietà and baldachin, Pantheon oculus and Raphael tomb, Navona/Four Rivers and Trevi; `food` covers Carbonara, Cacio e Pepe, Amatriciana, Gricia, standing coffee, avoiding immediate monument frontage; `practical` covers metro A/B plus walking, cobblestones, church dress, Termini/Colosseum pickpockets, 112; `stay` renders pending fields.

### Transport

Each record has `{ id, route, date, mode, from, to, window, duration, booked, status, notes }`.

- `transport-mxp-milan`: 9/26, Malpensa Express, MXP T1 → Milano Centrale or Cadorna, after baggage collection, about 50–55 minutes, not booked, `现场购票/出发前复核`, choose terminus by final hotel.
- `transport-milan-como`: 9/28, regional train, Milano Centrale → Como S. Giovanni, recommended 08:30 window, about 40–60 minutes, not booked, `建议班次，非已购票`.
- `transport-como-ferry`: 9/28, Navigazione Laghi ferry, Como → Bellagio → Como, late morning outbound and around 16:00 return target, duration varies by service, not booked, `季节船班，出发前复核`.
- `transport-milan-florence`: 9/29, Frecciarossa/Italo, Milano Centrale → Firenze SMN, 08:30–10:00 departure window, about 1h45–2h, not booked, `建议班次，非已购票`.
- `transport-florence-naples`: 10/2, Frecciarossa/Italo, Firenze SMN → Napoli Centrale, 08:30–09:00 departure window, about 2h45–3h, not booked, `建议班次，非已购票`.
- `transport-naples-rome`: 10/3, Frecciarossa/Italo, Napoli Centrale → Roma Termini, 08:30–09:30 departure window, about 1h10, not booked, `建议班次，非已购票`.
- `transport-rome-fco`: 10/6, Leonardo Express or official fixed-fare taxi, hotel/Termini → FCO T3, leave around 07:15–07:30, about 32 minutes by express or 40–60 by road, not booked, `按酒店位置决定，出发前复核`.

### Tickets

Each record has `{ id, name, cityId, date, suggestedTime, reservation, priority, status, officialUrl, caveat }`.

- `ticket-last-supper`: 《最后的晚餐》, Milan, 9/27, around 09:00, advance timed ticket, highest, `优先锁定`, `https://cenacolovinciano.vivaticket.it/`, actual ticket time controls the entire day.
- `ticket-duomo-milan`: 米兰大教堂天台, Milan, 9/27, late morning, advance recommended, high, `计划预订`, `https://ticket.duomomilano.it/en/`, Sunday worship may affect cathedral interior.
- `ticket-como-ferry`: 科莫湖渡船, Como, 9/28, late morning, schedule-dependent, medium, `出发前复核`, `https://www.navigazionelaghi.it/en/`, seasonal/weather changes.
- `ticket-uffizi`: 乌菲兹美术馆, Florence, 9/30, 08:30, timed reservation, highest, `计划预订`, `https://www.uffizi.it/en/tickets`, allow about three hours.
- `ticket-accademia`: 学院美术馆《大卫》, Florence, 10/1, 08:30, timed reservation, highest, `计划预订`, `https://www.galleriaaccademiafirenze.it/en/visit/`, use official channel only.
- `ticket-duomo-florence`: 百花大教堂穹顶, Florence, 10/1, 10:30–11:00, timed reservation, highest, `计划预订`, `https://tickets.duomo.firenze.it/en/store#/en/buy`, 463 steps/no elevator.
- `ticket-sansevero`: 圣塞维诺礼拜堂, Naples, 10/2, around 15:00, timed reservation, highest, `计划预订`, `https://www.museosansevero.it/en/plan-your-visit/`, small venue and strict entry time.
- `ticket-colosseum`: 斗兽场联票, Rome, 10/4, early morning, policy-dependent, highest, `出发前复核`, `https://ticketing.colosseo.it/en/`, first-Sunday rules and crowds require current notice.
- `ticket-vatican`: 梵蒂冈博物馆, Rome, 10/5, 08:00, timed reservation, highest, `计划预订`, `https://tickets.museivaticani.va/home`, Sistine Chapel no photography.

### Checklist

Groups and item IDs:

- `documents`: `passport` 护照有效期与复印件、`visa` 申根签证、`insurance` 旅行保险、`flight-copies` 往返机票确认、`hotel-copies` 酒店确认单待预订后补齐。
- `bookings`: `book-last-supper`, `book-uffizi`, `book-accademia`, `book-florence-dome`, `book-sansevero`, `book-colosseum`, `book-vatican`, `book-trains` 三段跨城高铁。
- `money-connectivity`: `cards` 双卡与少量欧元现金、`esim` eSIM/漫游、`offline-maps` 离线地图、`apps` Trenitalia/Italo/Google Maps、`power` 欧标转换插头与充电宝。
- `packing`: `shoes` 防滑舒适鞋、`layers` 薄外套、`rain` 折叠伞、`church-cover` 教堂遮肩遮膝衣物、`day-bag` 防盗随身包。
- `final-review`: `ferry-check` 科莫船班、`opening-check` 景点开放与罢工公告、`train-check` 实际车次站台、`flight-check` 航班与航站楼、`hotel-fill` 补齐页面住宿信息。
