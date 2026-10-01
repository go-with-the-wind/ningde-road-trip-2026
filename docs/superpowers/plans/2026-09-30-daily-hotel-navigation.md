# 每日酒店与高德导航 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 让五日路书使用者可以为 D1—D4 添加、保存、查询和导航每晚酒店，并在高德地图中定位已选酒店。

**Architecture:** 在单一静态页面内增加酒店配置常量、浏览器本地存储适配层和酒店 UI 渲染模块。高德 JS API PlaceSearch 负责候选 POI 选择；现有路线 `tripStops` 与 `routeSegments` 保持不变，新增酒店标记作为独立地图叠加层，避免用户选择的酒店改变主路线。

**Tech Stack:** HTML5 原生 `<dialog>`、CSS、原生 JavaScript、`localStorage`、高德 JS API 2.0 PlaceSearch、GitHub Pages。

## Global Constraints

- 修改源文件 `/Users/eric/Kiro/Investment/ningde-road-trip-2026.html`，并同步为发布仓库的 `index.html`。
- 不增加 npm 包、服务端、数据库、GitHub OAuth 或任何公开 token。
- D1 默认保留高德 POI `B0FFH8F056`：永泰云顶蚂蚁窝蛋居民宿，GCJ-02 `118.99915,25.76683`。
- D2、D3、D4 默认酒店为空；D4 必须可一键沿用 D3。
- 精确“高德导航”仅对拥有有效高德 POI 经纬度的酒店显示。
- 所有用户输入只保存到 `localStorage` key `ningde-road-trip-2026.hotels.v1`，页面必须说明其不会写回 GitHub。
- 现有主路线、天气、定位、实时路况、D1 酒店安全说明与既有酒店导航均不能回归。
- 用户已明确授权直接提交并推送 `main`。

---

## File Structure

- Modify: `/Users/eric/Kiro/Investment/ningde-road-trip-2026.html`
  - D1—D4 的酒店挂载点和全局编辑对话框。
  - 酒店卡、POI 候选结果与移动端样式。
  - 默认酒店数据、本地存储、POI 查询、导航 URI、地图酒店标记和事件绑定。
- Modify: `/Users/eric/Kiro/Investment/ningde-road-trip-2026-site/index.html`
  - 由源 HTML 的验证后同步结果；不得手工产生与源文件不同的内容。
- Create: `/Users/eric/Kiro/Investment/ningde-road-trip-2026-site/docs/superpowers/specs/2026-09-30-daily-hotel-navigation-design.md`
  - 已批准的功能设计。
- Create: `/Users/eric/Kiro/Investment/ningde-road-trip-2026-site/docs/superpowers/plans/2026-09-30-daily-hotel-navigation.md`
  - 本实施计划。

### Task 1: Add hotel data, local storage and rendering layer

**Files:**
- Modify: `/Users/eric/Kiro/Investment/ningde-road-trip-2026.html` (style block; D1—D4 itinerary markup; main inline script)

**Interfaces:**
- Produces `HOTEL_NIGHTS`, `loadHotels()`, `saveHotels(hotels)`, `renderHotelCards()`, `getHotelNavigationUrl(hotel)`, `focusHotel(nightId)`.
- Consumes existing `AMap`, `map`, `mapEngine`, `wgs84ToGcj02()`, `markers`, `amapInfoWindow`, `map-focus` conventions.

- [ ] **Step 1: Add stable hotel-night configuration and defaults**

```js
const HOTEL_STORAGE_KEY = 'ningde-road-trip-2026.hotels.v1';
const HOTEL_NIGHTS = Object.freeze([
  { id: 'd1', day: 'D1', date: '2026-10-01', city: '永泰', label: '10月1日 · 云顶', defaultHotel: { name: '永泰云顶蚂蚁窝蛋居民宿', address: '岭路乡长坑村187县道天池景区正对面', poiId: 'B0FFH8F056', lng: 118.99915, lat: 25.76683, tel: '0591-24769999;13774600034', note: '最终入口和停车以订单联系人为准。' } },
  { id: 'd2', day: 'D2', date: '2026-10-02', city: '屏南', label: '10月2日 · 屏南', defaultHotel: null },
  { id: 'd3', day: 'D3', date: '2026-10-03', city: '周宁', label: '10月3日 · 周宁', defaultHotel: null },
  { id: 'd4', day: 'D4', date: '2026-10-04', city: '周宁', label: '10月4日 · 周宁', defaultHotel: null }
]);
```

- [ ] **Step 2: Implement strict storage parsing and fallback**

```js
function isHotel(value) {
  return value && typeof value.name === 'string' && value.name.trim() &&
    (value.lng === null || (Number.isFinite(value.lng) && Number.isFinite(value.lat)));
}
function loadHotels() {
  try {
    const stored = JSON.parse(window.localStorage.getItem(HOTEL_STORAGE_KEY));
    if (stored?.version === 1 && stored.nights) return stored.nights;
  } catch (error) {
    console.warn('Stored hotels unavailable:', error);
  }
  return Object.fromEntries(HOTEL_NIGHTS.map(night => [night.id, night.defaultHotel]));
}
function saveHotels(nights) {
  try {
    window.localStorage.setItem(HOTEL_STORAGE_KEY, JSON.stringify({ version: 1, nights }));
    return true;
  } catch (error) {
    console.warn('Hotel storage unavailable:', error);
    return false;
  }
}
```

- [ ] **Step 3: Add one hotel mount point to each D1—D4 day card**

Add `<section class="hotel-slot" data-hotel-night="d1"></section>` after each D1—D4 schedule or note box. Do not add one to D5 because it has no overnight stay.

- [ ] **Step 4: Render hotel cards and exact navigation URLs**

```js
function getHotelNavigationUrl(hotel) {
  if (!Number.isFinite(hotel.lng) || !Number.isFinite(hotel.lat)) return null;
  return `https://uri.amap.com/navigation?from=&to=${hotel.lng},${hotel.lat},${encodeURIComponent(hotel.name)}&mode=car&policy=1&src=ningde-road-trip-2026&coordinate=gaode&callnative=0`;
}
function renderHotelCards() {
  HOTEL_NIGHTS.forEach(night => {
    const slot = document.querySelector(`[data-hotel-night="${night.id}"]`);
    const hotel = hotels[night.id];
    slot.innerHTML = hotel ? renderSavedHotel(night, hotel) : renderEmptyHotel(night);
  });
}
```

- [ ] **Step 5: Verify storage and markup locally**

Run a Python assertion script that confirms four `data-hotel-night` slots, `HOTEL_STORAGE_KEY`, D1 default POI ID, D4 entry and no D5 hotel slot. Extract inline JavaScript and run `node --check`.

- [ ] **Step 6: Commit**

```bash
BLOB=$(git hash-object -w --path=index.html "/Users/eric/Kiro/Investment/ningde-road-trip-2026.html")
git update-index --cacheinfo "100644,$BLOB,index.html"
git checkout-index -f -- index.html
git add index.html
git commit -m "feat: add local daily hotel cards"
```

### Task 2: Add hotel edit dialog and AMap POI selection

**Files:**
- Modify: `/Users/eric/Kiro/Investment/ningde-road-trip-2026.html` (dialog markup, hotel card styles, inline script)

**Interfaces:**
- Consumes `HOTEL_NIGHTS`, `hotels`, `saveHotels()`, `renderHotelCards()` from Task 1.
- Produces `openHotelDialog(nightId)`, `searchHotelPoi()`, `selectHotelPoi(poi)`, `closeHotelDialog()`.

- [ ] **Step 1: Add dialog fields and accessible status region**

```html
<dialog id="hotel-dialog" aria-labelledby="hotel-dialog-title">
  <form method="dialog" id="hotel-form">
    <h2 id="hotel-dialog-title">添加今晚酒店</h2>
    <label>酒店名称 <input id="hotel-name" required autocomplete="organization"></label>
    <label>所在城市 <input id="hotel-city" required></label>
    <label>备注（可选） <textarea id="hotel-note"></textarea></label>
    <p id="hotel-search-status" aria-live="polite"></p>
    <div id="hotel-poi-results" aria-live="polite"></div>
    <button type="button" id="hotel-search">查询高德 POI</button>
    <button type="submit" id="hotel-save" disabled>保存酒店</button>
    <button type="button" id="hotel-cancel">取消</button>
  </form>
</dialog>
```

- [ ] **Step 2: Implement PlaceSearch with a user-selected result**

```js
function searchHotelPoi() {
  if (!window.AMap) return setHotelStatus('高德地图尚未加载，暂不能查询 POI。');
  const keyword = hotelNameInput.value.trim();
  const city = hotelCityInput.value.trim();
  if (!keyword || !city) return setHotelStatus('请先填写酒店名称和城市。');
  AMap.plugin('AMap.PlaceSearch', () => {
    const service = new AMap.PlaceSearch({ city, citylimit: false, pageSize: 6, extensions: 'all' });
    service.search(keyword, (status, result) => {
      const pois = status === 'complete' ? result?.poiList?.pois || [] : [];
      renderPoiResults(pois);
    });
  });
}
```

Each candidate button stores `id`, `name`, `address`, `location.lng`, `location.lat` and `tel` in a pending selection. Save remains disabled until selection is present. Do not automatically select the first result.

- [ ] **Step 3: Support D4 “沿用 D3 酒店” and clear actions**

The empty D4 card displays “沿用 D3 酒店” only when D3 contains a valid hotel. It deep-copies D3 to D4, changes no D3 data, saves, rerenders and updates map markers. Every saved card has “清除本晚酒店” with a confirmation dialog.

- [ ] **Step 4: Verify POI selection behavior in a real browser**

Use the published-domain AMap key in a local Chrome session or the GitHub Pages preview after deployment. Verify: querying a known D1 hotel yields a candidate; save remains disabled before selection; selected POI produces name, address and navigation button; no-result state is readable; dialog closes with Escape.

- [ ] **Step 5: Commit**

```bash
BLOB=$(git hash-object -w --path=index.html "/Users/eric/Kiro/Investment/ningde-road-trip-2026.html")
git update-index --cacheinfo "100644,$BLOB,index.html"
git checkout-index -f -- index.html
git add index.html
git commit -m "feat: add AMap hotel POI editor"
```

### Task 3: Add independent hotel map markers and integration validation

**Files:**
- Modify: `/Users/eric/Kiro/Investment/ningde-road-trip-2026.html`

**Interfaces:**
- Consumes `hotels`, `renderHotelCards()`, `mapEngine`, `map`, existing fallback map behavior.
- Produces `renderHotelMapMarkers()`, `clearHotelMapMarkers()`, `focusHotel(nightId)`.

- [ ] **Step 1: Add separate hotel marker state**

```js
const hotelMarkers = new Map();
function clearHotelMapMarkers() {
  hotelMarkers.forEach(marker => marker.setMap?.(null));
  hotelMarkers.clear();
}
```

Keep `hotelMarkers` out of `amapOverlays`; hotel changes must not recalculate, recolor or change the bounds of the four main route segments.

- [ ] **Step 2: Render markers only for valid selected hotels**

For AMap, use stored GCJ-02 `lng`/`lat` directly; do not pass them through `wgs84ToGcj02`. Render a marker with D1—D4 badge and title. On click, show hotel name, address, optional `高德登记电话` and note. For the Leaflet/vector fallback, render a lightweight HTML marker at the WGS84 conversion of the selected POI or leave the card navigation functional if conversion is unavailable.

- [ ] **Step 3: Wire card actions**

- `data-hotel-action="focus"` calls `focusHotel(nightId)` and scrolls to map.
- `data-hotel-action="edit"` opens the dialog.
- `data-hotel-action="clear"` confirms then clears current night.
- `data-hotel-action="copy-d3"` copies D3 into D4.

Use one delegated click listener on the itinerary container, so rerendering hotel card HTML does not lose behavior.

- [ ] **Step 4: Run regression validation**

1. Extract all inline JavaScript and run `node --check`.
2. Assert source HTML has four hotel slots, one dialog, no GitHub token/OAuth code, and D1 POI ID.
3. Use headless Chrome on the deployed page to verify AMap loads, four main route markers still render, four real routes still render, five weather cards load, and a saved D1 hotel card exposes an AMap navigation URI.
4. Test local storage persistence by setting D2 in Chrome, reloading, then verifying the saved D2 card remains.
5. Test D4 copy behavior and D3 non-mutation.

- [ ] **Step 5: Synchronize source and Pages file**

```bash
BLOB=$(git hash-object -w --path=index.html "/Users/eric/Kiro/Investment/ningde-road-trip-2026.html")
git update-index --cacheinfo "100644,$BLOB,index.html"
git checkout-index -f -- index.html
git diff --cached --check
cmp -s index.html ../ningde-road-trip-2026.html
```

- [ ] **Step 6: Commit and publish**

```bash
git add index.html docs/superpowers/specs/2026-09-30-daily-hotel-navigation-design.md docs/superpowers/plans/2026-09-30-daily-hotel-navigation.md
git commit -m "feat: add daily hotel navigation"
git push origin main
```

Poll `repos/go-with-the-wind/ningde-road-trip-2026/pages/builds/latest` until the committed revision is `built`, then verify the live page.

## Plan Self-Review

- **Spec coverage:** Task 1 provides D1—D4 cards and safe local persistence; Task 2 provides the user-facing add/edit/POI flow and D4 copy; Task 3 provides map integration, navigation, fallback behavior, validation and Pages deployment.
- **Placeholder scan:** No TODO/TBD placeholders; all required fields, IDs, storage key, POI ID and validation steps are explicit.
- **Type consistency:** Every task uses the same `HOTEL_NIGHTS`, `hotels`, `nightId`, `lng`/`lat` and `HOTEL_STORAGE_KEY` names. High-precision POI data is explicitly GCJ-02 where passed to AMap.

## Execution Handoff

User explicitly requested direct implementation and publication. Execute Task 1 through Task 3 inline in this session, preserving the validation and deployment gates above.
