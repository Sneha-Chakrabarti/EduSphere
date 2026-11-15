/* EduSphere theme behaviour: login answer sheet and number count-up. Safe to remove. */
(function () {
  'use strict';
  var reduce = window.matchMedia && matchMedia('(prefers-reduced-motion: reduce)').matches;

  // ---- login panel ----
  var login = document.getElementById('screen-login');
  if (login && !login.querySelector('.omr-panel')) {
    var answers = [2, 0, 3, 1, 1, 2, 0, 3];
    var letters = ['A', 'B', 'C', 'D'];
    var rows = '';
    answers.forEach(function (_, i) {
      rows += '<div class="omr-row" data-r="' + i + '"><span class="n">' + (i + 1) + '</span>'
        + [0, 1, 2, 3].map(function (c) { return '<span class="omr-b" data-c="' + c + '"></span>'; }).join('')
        + '<svg class="omr-tick" viewBox="0 0 20 20" fill="none" stroke="currentColor" stroke-width="2.6" stroke-linecap="round" stroke-linejoin="round"><path d="M4 10.5l4 4 8-9"/></svg></div>';
    });
    var panel = document.createElement('aside');
    panel.className = 'omr-panel';
    panel.setAttribute('aria-hidden', 'true');
    panel.innerHTML = '<div class="omr-copy"><div class="omr-logo">Edu<b>Sphere</b></div>'
      + '<h2>Practice until the right answer is the easy one.</h2>'
      + '<p>MCQ courses for Tripura TET and STPGT, and chapter notes for CBSE classes 9 to 12. Every answer comes with its explanation.</p></div>'
      + '<div class="omr-sheet"><div class="omr-head"><span>Answer sheet</span><span id="omrCount">0 of 8 marked</span></div>'
      + '<div class="omr-row omr-letters"><span></span>' + letters.map(function (l) { return '<span>' + l + '</span>'; }).join('') + '<span></span></div>'
      + rows + '<div class="omr-foot"><span>Explained after every question</span><span class="score" id="omrScore"></span></div></div>';
    login.insertBefore(panel, login.firstChild);

    var timers = [];
    var rowEls = panel.querySelectorAll('.omr-row[data-r]');
    var count = panel.querySelector('#omrCount'), score = panel.querySelector('#omrScore');
    var clearAll = function () {
      timers.forEach(clearTimeout); timers = [];
      Array.prototype.forEach.call(rowEls, function (r) {
        r.classList.remove('ok');
        Array.prototype.forEach.call(r.querySelectorAll('.omr-b'), function (b) { b.classList.remove('on'); });
      });
      count.textContent = '0 of 8 marked'; score.textContent = '';
    };
    var mark = function (i) {
      rowEls[i].querySelectorAll('.omr-b')[answers[i]].classList.add('on');
      rowEls[i].classList.add('ok');
      count.textContent = (i + 1) + ' of 8 marked';
    };
    var cycle = function () {
      clearAll();
      if (document.hidden || !login.classList.contains('active')) { timers.push(setTimeout(cycle, 1500)); return; }
      answers.forEach(function (_, i) { timers.push(setTimeout(function () { mark(i); }, 900 + i * 520)); });
      var done = 900 + answers.length * 520;
      timers.push(setTimeout(function () { score.textContent = '8 / 8'; }, done + 150));
      timers.push(setTimeout(cycle, done + 3600));
    };
    if (reduce) { answers.forEach(function (_, i) { mark(i); }); score.textContent = '8 / 8'; }
    else cycle();
  }

  // ---- count-up for dashboard numbers ----
  function countUp(el) {
    if (reduce || el.getAttribute('data-cu')) return;
    var t = (el.textContent || '').trim();
    var m = /^(₹?)([\d,]+)$/.exec(t);
    el.setAttribute('data-cu', '1');
    if (!m) return;
    var end = parseInt(m[2].replace(/,/g, ''), 10);
    if (!(end > 0) || end > 1e7) return;
    var start = performance.now(), dur = 700;
    (function step(now) {
      var p = Math.min(1, (now - start) / dur), e = 1 - Math.pow(1 - p, 3);
      el.textContent = m[1] + Math.round(end * e).toLocaleString('en-IN');
      if (p < 1) requestAnimationFrame(step); else el.textContent = t;
    })(start);
  }
  var scanQueued = false;
  function scan() {
    scanQueued = false;
    Array.prototype.forEach.call(document.querySelectorAll('.stat-card .value, .kpi .k-val'), function (el) {
      if (el.offsetParent !== null) countUp(el);
    });
  }
  new MutationObserver(function () { if (!scanQueued) { scanQueued = true; setTimeout(scan, 80); } })
    .observe(document.body, { childList: true, subtree: true, attributes: true, attributeFilter: ['class', 'style'] });
  scan();
})();

// ---- v2 extras: landing hero shapes, cursor spotlight, 3D tilt ----
(function () {
  'use strict';
  var reduce = window.matchMedia && matchMedia('(prefers-reduced-motion: reduce)').matches;
  var hero = document.querySelector('.hero');
  if (hero && !hero.querySelector('.hero-orbs')) {
    var o = document.createElement('div');
    o.className = 'hero-orbs';
    o.setAttribute('aria-hidden', 'true');
    o.innerHTML = '<div class="hero-ring r1"></div><div class="hero-ring r2"></div>'
      + '<div class="hero-chip c1"><i>📝</i><span>TET Paper I and II<small>Notes with MCQs</small></span></div>'
      + '<div class="hero-chip c2"><i>🎓</i><span>STPGT Physics<small>4 levels, each with answers</small></span></div>'
      + '<div class="hero-chip c3"><i>🔬</i><span>CBSE 9 to 12<small>Chapter-wise notes</small></span></div>';
    hero.insertBefore(o, hero.firstChild);
  }
  var sel = '.material-card, .dash-material-card, .class-tile, .level-card';
  document.addEventListener('pointermove', function (e) {
    var t = e.target && e.target.closest ? e.target.closest(sel + ', .hub-btn') : null;
    if (!t) return;
    var r = t.getBoundingClientRect();
    t.style.setProperty('--mx', (e.clientX - r.left) + 'px');
    t.style.setProperty('--my', (e.clientY - r.top) + 'px');
    if (!reduce && t.classList.contains('hub-btn')) {
      var px = (e.clientX - r.left) / r.width - .5, py = (e.clientY - r.top) / r.height - .5;
      t.style.setProperty('--ry', (px * 8).toFixed(2) + 'deg');
      t.style.setProperty('--rx', (-py * 8).toFixed(2) + 'deg');
    }
  }, { passive: true });
  document.addEventListener('pointerout', function (e) {
    var t = e.target && e.target.closest ? e.target.closest('.hub-btn') : null;
    if (t) { t.style.setProperty('--rx', '0deg'); t.style.setProperty('--ry', '0deg'); }
  }, { passive: true });
})();
