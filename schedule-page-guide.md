# 스케줄 페이지 복제 가이드

서랍길드 관리 페이지(`index.html`)에 만든 "🗓️ 스케줄" 탭 — 진행 중인 게임 이벤트를 카드 목록으로 보여주는 기능 — 을 다른 길드 사이트에도 그대로 붙여넣기 위한 코드 모음입니다.

이 사이트는 프레임워크 없이 `index.html` 한 파일에 `<style>` / HTML / `<script>`가 다 들어있는 구조입니다. 대상 사이트도 비슷한 구조라면 아래 순서대로 붙여넣으면 됩니다.

---

## 1. 필요한 CSS 변수 (색상 토큰)

`:root`에 아래 색상들이 이미 있는지 확인하고, 없으면 추가하세요. (다크 테마 배경 기준 색상입니다. 대상 사이트가 라이트 테마라면 톤을 맞게 조정 필요)

```css
:root{
  --bg:#0d0f14;--s1:#141720;--s2:#1c2030;
  --border:rgba(255,255,255,0.07);--border2:rgba(255,255,255,0.13);
  --text:#dde1ed;--t2:#8490aa;--t3:#4a5268;
  --blue:#5b8af5;--blue2:#3d6be0;
  --green:#3ecf8e;--red:#f06565;--gold:#f5c842;--purple:#9b7ff5;--teal:#2dd4bf;--orange:#f5924a;
}
```

## 2. 공통 클래스 (없으면 추가)

```css
.page{display:none;padding:24px;max-width:1200px;margin:0 auto}
.page.on{display:block}

.sec-title{font-size:13px;color:var(--t2);margin-bottom:14px;display:flex;align-items:center;gap:7px}
.sec-title::before{content:'';width:3px;height:13px;background:var(--blue);border-radius:2px;display:block}
```

## 3. 스케줄 카드 CSS

```css
/* 스케줄 */
.sch-event-card{background:var(--s1);border:1px solid var(--border);border-left:4px solid var(--blue);border-radius:10px;padding:13px 16px;margin-bottom:10px;display:flex;gap:12px;align-items:flex-start}
.sch-event-icon{font-size:19px;line-height:1;margin-top:1px}
.sch-event-title{font-size:14px;font-weight:600;color:var(--text)}
.sch-event-date{font-size:11.5px;color:var(--t3);margin-top:2px}
.sch-event-desc{font-size:12.5px;color:var(--t2);margin-top:5px;line-height:1.6}
```

## 4. 네비게이션 탭 + 페이지 컨테이너 (HTML)

기존 nav 탭 목록(데스크톱용 `.nav-tabs-wrap`, 모바일 드롭다운용 `.nav-dropdown`) 두 군데에 동일하게 버튼을 추가하고, 본문에 빈 `<div>`를 하나 둡니다.

```html
<!-- 데스크톱 nav 탭 목록 안에 -->
<button class="nav-tab" onclick="goPage('schedule')">🗓️ 스케줄</button>

<!-- 모바일 드롭다운 목록 안에 (동일한 버튼) -->
<button class="nav-tab" onclick="goPage('schedule')">🗓️ 스케줄</button>

<!-- 페이지 컨테이너들 옆에 -->
<div id="page-schedule" class="page"></div>
```

## 5. 페이지 전환 로직 연결 (JS)

사이트에 이미 `goPage(name)` / `PAGE_LABELS` / `_showPage(name)` 같은 탭 전환 함수가 있다면 아래처럼 `schedule`을 끼워 넣습니다. (비밀번호 없이 누구나 볼 수 있게 하려면 인증 예외 목록에도 추가)

```js
// 비밀번호 없이 볼 수 있는 페이지 목록에 'schedule' 추가
function goPage(name){
  if(name !== 'dashboard' && name !== 'info' && name !== 'schedule' && !_authenticated){
    // ...비밀번호 모달 로직...
  }
  _showPage(name);
}

const PAGE_LABELS = {/* 기존 항목들 */, schedule:'스케줄'};

function _showPage(name){
  // ...기존 로직...
  if(name==='schedule') renderSchedulePage();
}
```

사이트 구조가 다르다면 이 부분만 해당 사이트의 라우팅 방식에 맞게 손보면 됩니다. 핵심은 `#page-schedule` 요소가 화면에 보일 때 `renderSchedulePage()`를 호출해주는 것뿐입니다.

## 6. 이벤트 데이터 + 렌더 함수 (JS)

이 부분이 실제 기능의 핵심입니다. `SCHEDULE_EVENTS` 배열만 길드/게임에 맞게 수정하면 나머지는 그대로 재사용 가능합니다.

```js
// ══════════════════════════════════════════════
//  SCHEDULE
// ══════════════════════════════════════════════
const SCH_TYPE_COLOR = {
  coupon:'--gold', hero:'--purple', buff:'--red', attendance:'--teal',
  dropup:'--green', event:'--blue', pvp:'--orange',
};
const SCHEDULE_EVENTS = [
  // id: 고유 번호
  // type: coupon | hero | buff | attendance | dropup | event | pvp (SCH_TYPE_COLOR의 키와 맞춰야 함)
  // icon: 카드에 표시할 이모지
  // title: 이벤트 이름
  // start: 시작일 'YYYY-MM-DD'
  // end: 종료일 'YYYY-MM-DD' 또는 종료일 미정이면 null (목록에 "진행중"으로 표시됨)
  // desc: 상세 설명
  {id:1, type:'coupon', icon:'🎟️', title:'예시 쿠폰 이벤트', start:'2026-07-23', end:'2026-09-02', desc:'설명을 적어주세요.'},
];

function _schFmtDate(dateStr){
  const [yy,mm,dd] = dateStr.split('-');
  return parseInt(mm)+'월 '+parseInt(dd)+'일';
}

function renderSchedulePage(){
  const sortedEvents = [...SCHEDULE_EVENTS].sort((a,b)=>a.start<b.start?-1:1);
  const listHtml = sortedEvents.map(e=>{
    const dateRange = e.start===e.end ? _schFmtDate(e.start) : (_schFmtDate(e.start)+' ~ '+(e.end?_schFmtDate(e.end):'진행중'));
    return `<div class="sch-event-card" style="border-left-color:var(${SCH_TYPE_COLOR[e.type]})">
      <div class="sch-event-icon">${e.icon}</div>
      <div>
        <div class="sch-event-title">${e.title}</div>
        <div class="sch-event-date">${dateRange}</div>
        <div class="sch-event-desc">${e.desc}</div>
      </div>
    </div>`;
  }).join('');

  document.getElementById('page-schedule').innerHTML = `
    <div class="sec-title" style="margin-bottom:18px">이벤트 스케줄</div>
    ${listHtml}
  `;
}
```

## 7. 새 길드 사이트에 붙일 때 체크리스트

- [ ] `:root`에 색상 변수(`--gold`, `--purple`, `--red`, `--teal`, `--green`, `--blue`, `--orange` 등) 있는지 확인
- [ ] `.page`, `.sec-title` 클래스 있는지 확인 (없으면 2번 섹션 CSS 추가)
- [ ] 3번 섹션 CSS를 `<style>`에 추가
- [ ] nav 탭 버튼 2곳 + `#page-schedule` div 추가
- [ ] 탭 전환 함수에 `schedule` 케이스 연결
- [ ] `SCHEDULE_EVENTS` 배열을 새 길드의 실제 이벤트로 교체
- [ ] 페이지 열어서 카드가 시작일순으로 잘 나오는지, 색상 구분이 잘 되는지 확인
