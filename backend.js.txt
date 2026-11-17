// EduSphere live mode.
// When config.js holds a Supabase URL and key, this file replaces the in-browser demo data with a real
// database: accounts, device lock, payments, receipts, admin actions and private PDF viewing.
// Without a configuration the page stays in demo mode and this file does nothing.
(function () {
  'use strict';
  var CFG = window.EDU_CONFIG || {};
  if (!CFG.SUPABASE_URL || !CFG.SUPABASE_ANON_KEY || !window.supabase) {
    console.info('EduSphere: demo mode (no Supabase configuration found).');
    return;
  }
  var sb = window.supabase.createClient(CFG.SUPABASE_URL, CFG.SUPABASE_ANON_KEY, { auth: { persistSession: false } });

  // ---------- helpers ----------
  function synthEmail(mobile) { return mobile + '@students.example.com'; }
  function slug(s) { return String(s).toLowerCase().replace(/[^a-z0-9]+/g, '-').replace(/^-+|-+$/g, ''); }
  function fmtDate(d) { return new Date(d).toLocaleDateString('en-GB', { day: '2-digit', month: 'short', year: 'numeric' }); }
  function deviceFp() {
    var id = null;
    try { id = localStorage.getItem('edu_device'); } catch (e) {}
    if (!id) {
      id = (window.crypto && crypto.randomUUID) ? crypto.randomUUID() : String(Date.now()) + Math.random().toString(16).slice(2);
      try { localStorage.setItem('edu_device', id); } catch (e) {}
    }
    return id;
  }
  function deviceLabel() {
    var ua = navigator.userAgent;
    var os = /Windows/.test(ua) ? 'Windows' : /Android/.test(ua) ? 'Android' : /iPhone|iPad/.test(ua) ? 'iOS' : /Mac/.test(ua) ? 'Mac' : /Linux/.test(ua) ? 'Linux' : 'Device';
    var br = /Edg\//.test(ua) ? 'Edge' : /Chrome\//.test(ua) ? 'Chrome' : /Firefox\//.test(ua) ? 'Firefox' : /Safari\//.test(ua) ? 'Safari' : 'Browser';
    return os + ' / ' + br;
  }
  function mapProfile(p) {
    return { uid: p.id, id: p.student_id, name: p.name, father: p.father, mother: p.mother, address: p.address,
      mobile: p.mobile || '', parentMobile: p.parent_mobile || '', email: p.email || '', status: p.status,
      device: p.device_label || 'Not registered', password: '', pdfs: 0, owned: [] };
  }
  var seenKey = 'edu_seen_receipts';
  function seenSet() { try { return JSON.parse(localStorage.getItem(seenKey) || '[]'); } catch (e) { return []; } }
  function mapPay(r) {
    return { utr: r.utr, sid: r.user_id ? r.student_id : r.student_id + ' (deleted)', label: r.label, items: r.items || [],
      total: Number(r.total), date: fmtDate(r.pay_date), status: r.status, receiptNo: r.receipt_no,
      approvedOn: r.approved_on ? fmtDate(r.approved_on) : null, receiptSeen: !r.receipt_no || seenSet().indexOf(r.receipt_no) >= 0,
      paidAmount: r.paid_amount == null ? null : Number(r.paid_amount), payer: r.payer_name || '', upiApp: r.upi_app || '',
      paidAt: r.paid_at || null, createdAt: r.created_at || null,
      refundStatus: r.refund_status || null, refundedAt: r.refunded_at || null, decidedAt: r.decided_at || null };
  }
  function fmtDT(d) {
    if (!d) return '-';
    return new Date(d).toLocaleString('en-GB', { day: '2-digit', month: 'short', year: 'numeric', hour: '2-digit', minute: '2-digit', hour12: true });
  }
  function rupees(n) { return '\u20b9' + Number(n).toLocaleString('en-IN', { minimumFractionDigits: 0, maximumFractionDigits: 2 }); }
  function escH(t) { return String(t == null ? '' : t).replace(/[&<>"]/g, function (c) { return { '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;' }[c]; }); }
  function fail(msg) { showToast(msg, 'error'); }

  // ---------- start with empty data ----------
  accounts.length = 0;
  payments.length = 0;
  deviceRequests.length = 0;
  auditLog.length = 0;
  staticLogs = [];
  var loginEl = document.getElementById('loginId'), loginPw = document.getElementById('loginPw');
  if (loginEl) loginEl.value = '';
  if (loginPw) loginPw.value = '';
  Array.prototype.forEach.call(document.querySelectorAll('#screen-login div'), function (d) {
    if (/^Demo (admin )?login/.test((d.textContent || '').trim()) && d.children.length === 0) d.remove();
  });
  Array.prototype.forEach.call(document.querySelectorAll('#screen-login a'), function (a) {
    if (/Forgot password/.test(a.textContent)) a.onclick = function () { showToast('Please ask the admin to reset your password.'); };
  });
  var st = document.createElement('style');
  st.textContent = '@media print{body{display:none!important}}';
  document.head.appendChild(st);
  window.addEventListener('keydown', function (e) {
    if (currentScreen === 'viewer' && (e.ctrlKey || e.metaKey) && /^[sSpP]$/.test(e.key)) e.preventDefault();
  });

  // ---------- materials (which PDFs exist) ----------
  function clearMap(o) { Object.keys(o).forEach(function (k) { delete o[k]; }); }
  function applyMaterials(rows) {
    clearMap(SUBJ_PDF); clearMap(COMBO_PDF); clearMap(CBSE_PDF);
    var cbse = {};
    rows.forEach(function (r) {
      var p = r.path.split('/'), info = { pages: r.pages, mcq: r.mcq };
      if (p[0] === 'tet') {
        if (/^combo\d+$/.test(p[2])) COMBO_PDF[p[1] + '|' + p[2].slice(5) + '|' + p[3]] = info;
        else {
          var s = (TET[p[1]] || { subjects: [] }).subjects.filter(function (x) { return slug(x[0]) === p[2]; })[0];
          if (s) SUBJ_PDF[p[1] + '|' + s[0] + '|' + p[3]] = info;
        }
      } else if (p[0] === 'stpgt') {
        var L = STPGT.subjects[p[1]] && STPGT.subjects[p[1]].levels[+p[2]];
        if (L && r.pages) L.pages = r.pages;
      } else if (p[0] === 'cbse') {
        var k = p[1] + '|' + p[2];
        (cbse[k] = cbse[k] || []).push(+String(p[3]).replace('ch', ''));
      }
    });
    Object.keys(cbse).forEach(function (k) {
      var c = COURSES[k.split('|')[0]];
      CBSE_PDF[k] = (c && cbse[k].length >= c.n) ? 'all' : cbse[k];
    });
  }
  async function loadMaterials() {
    var r = await sb.from('materials').select('*');
    if (!r.error) applyMaterials(r.data || []);
    try { await loadLibrary(); } catch (e) {}
  }

  // ---------- data loading ----------
  async function loadMyPayments() {
    var r = await sb.from('payments').select('*').order('created_at');
    payments.length = 0;
    (r.data || []).forEach(function (x) { payments.push(mapPay(x)); });
  }
  async function loadAdminData() {
    var q = await Promise.all([
      sb.from('profiles').select('*').order('created_at'),
      sb.from('payments').select('*').order('created_at'),
      sb.from('device_requests').select('*').order('created_at', { ascending: false }),
      sb.from('audit_log').select('*').order('at', { ascending: false }).limit(300)
    ]);
    accounts.length = 0;
    (q[0].data || []).filter(function (x) { return x.role !== 'admin'; }).forEach(function (x) { accounts.push(mapProfile(x)); });
    payments.length = 0;
    (q[1].data || []).forEach(function (x) { payments.push(mapPay(x)); });
    deviceRequests.length = 0;
    (q[2].data || []).forEach(function (r) {
      deviceRequests.push({ id: r.id, sid: r.student_id, when: fmtDate(r.created_at), reason: '', otp: 'Verified', state: r.state });
    });
    auditLog.length = 0;
    (q[3].data || []).forEach(function (r) {
      auditLog.push({ t: new Date(r.at).toISOString().slice(0, 19).replace('T', ' '), user: r.actor, action: r.action, device: r.device || '', result: r.result || '' });
    });
  }
  async function adminRefresh(msg, kind) {
    await loadAdminData();
    refreshAdmin();
    if (msg) showToast(msg, kind);
  }

  // ---------- audit log ----------
  logEvent = function (user, action, device, result) {
    var row = { t: nowStamp(), user: user, action: action, device: device || 'admin_panel', result: result || 'OK' };
    auditLog.unshift(row);
    sb.from('audit_log').insert({ actor: user, action: action, device: row.device, result: row.result }).then(function () {}, function () {});
  };

  // ---------- student sign-up and login ----------
  doSignup = async function () {
    var g = function (i) { return document.getElementById(i).value.trim(); };
    var f = { id: '', name: g('suName'), father: g('suFather'), mother: g('suMother'), address: g('suAddress'), mobile: g('suMobile'),
      parentMobile: g('suParentMobile'), email: g('suEmail').toLowerCase(), pw: document.getElementById('suPw').value, pw2: document.getElementById('suPw2').value };
    var box = document.getElementById('signupError');
    var bad = function (m) { box.textContent = m; box.style.display = 'block'; };
    var err = validateNew(f);
    if (err) return bad(err);
    var chk = await sb.rpc('check_signup', { p_mobile: f.mobile, p_email: f.email || null });
    if (chk.data) return bad(chk.data);
    var res = await sb.auth.signUp({
      email: synthEmail(f.mobile), password: f.pw,
      options: { data: { name: f.name, father: f.father, mother: f.mother, address: f.address, mobile: f.mobile, parent_mobile: f.parentMobile, email: f.email } }
    });
    if (res.error || !res.data || !res.data.user) return bad((res.error && res.error.message) || 'Could not create the account.');
    var sid = null;
    if (res.data.session) {
      var p = await sb.from('profiles').select('student_id').eq('id', res.data.user.id).single();
      sid = p.data && p.data.student_id;
      await sb.auth.signOut();
    }
    box.style.display = 'none';
    ['suName', 'suFather', 'suMother', 'suAddress', 'suParentMobile', 'suEmail', 'suMobile', 'suPw', 'suPw2'].forEach(function (x) { document.getElementById(x).value = ''; });
    var n = document.getElementById('loginNotice');
    if (sid) {
      document.getElementById('loginId').value = sid;
      n.textContent = 'Account created. Your Student ID is ' + sid + '. Log in to register this device.';
    } else {
      n.textContent = 'Account created. Log in with your Student ID (ask the admin if you did not see it).';
    }
    document.getElementById('loginPw').value = '';
    n.style.display = 'block';
    showScreen('login');
    showToast(sid ? 'Account created: ' + sid : 'Account created.', 'success');
  };

  async function signInWith(identifier, pw) {
    var e = await sb.rpc('login_email', { identifier: identifier });
    if (e.error || !e.data) return null;
    var s = await sb.auth.signInWithPassword({ email: e.data, password: pw });
    if (s.error || !s.data || !s.data.user) return null;
    var p = await sb.from('profiles').select('*').eq('id', s.data.user.id).single();
    return p.data || null;
  }

  doLogin = async function () {
    var id = document.getElementById('loginId').value.trim();
    var pw = document.getElementById('loginPw').value;
    if (!id || !pw) return fail('Please enter your Student ID and password.');
    showToast('Verifying credentials and device...');
    var prof = await signInWith(id, pw);
    if (!prof) return fail('Invalid Student ID or password.');
    if (prof.role === 'admin') { await sb.auth.signOut(); return fail('Please use the admin login.'); }
    var fp = deviceFp();
    var b = await sb.rpc('bind_device', { fp: fp, label: deviceLabel() });
    if (b.data === 'BLOCKED') { await sb.auth.signOut(); return fail('This account is blocked. Please contact the admin.'); }
    if (b.data === 'DEVICE_MISMATCH') {
      await sb.rpc('request_device_change', { fp: fp });
      await sb.auth.signOut();
      showToast('This account is registered to another device. A device change request has been sent to the admin. You can log in here once it is approved.', 'warning');
      return;
    }
    var again = await sb.from('profiles').select('*').eq('id', prof.id).single();
    var acc = mapProfile(again.data || prof);
    currentUser = acc;
    await loadMyPayments();
    await loadMaterials();
    document.getElementById('dashName').textContent = acc.name;
    document.getElementById('dashId').textContent = acc.id;
    document.getElementById('dashFirst').textContent = acc.name.split(' ')[0];
    document.getElementById('dashAvatar').textContent = acc.name.split(' ').map(function (w) { return w.charAt(0); }).join('').slice(0, 2).toUpperCase();
    document.getElementById('pdfPage').setAttribute('data-watermark', acc.id + ' · ' + acc.name);
    document.getElementById('loginNotice').style.display = 'none';
    document.getElementById('menuName').textContent = acc.name.split(' ')[0];
    document.getElementById('hubName').textContent = acc.name.split(' ')[0];
    currentCat = null;
    showToast('Login successful.', 'success');
    showScreen('home');
    refreshMenu();
    logEvent(acc.id, 'LOGIN', fp.slice(0, 8), 'OK');
  };

  logout = async function () {
    try { await sb.auth.signOut(); } catch (e) {}
    currentUser = null;
    clearCart();
    payments.length = 0;
    showScreen('login');
  };

  openDeviceChange = function () {
    showToast('To change your device, log in from the new device. The request goes to the admin automatically.');
  };

  // ---------- purchases ----------
  function pathsFor(e) {
    if (e.lib) return [e.path];
    if (e.stpgt) return ['stpgt/' + e.stpgt + '/' + e.level];
    if (e.tet) {
      var T = TET[e.tet];
      if (String(e.subj).indexOf('combo') === 0) {
        var j = +String(e.subj).slice(5);
        return ['tet/' + e.tet + '/combo' + j + '/' + e.slab].concat(T.combos[j].subjects.map(function (s) { return 'tet/' + e.tet + '/' + slug(s) + '/' + e.slab; }));
      }
      return ['tet/' + e.tet + '/' + slug(e.subj) + '/' + e.slab];
    }
    if (e.courseId) {
      var c = COURSES[e.courseId];
      var chs = e.ch === 'all' ? Array.from({ length: c.n }, function (_, i) { return i + 1; }) : [e.ch];
      return chs.map(function (ch) { return 'cbse/' + e.courseId + '/' + e.level + '/ch' + ch; });
    }
    return [];
  }

  // ---------- checkout page (QR only, manual verification) ----------
  var CK = (document.getElementById('utrInput') && document.getElementById('utrInput').closest('.screen')) || document.getElementById('screen-checkout');
  function checkoutTotal() { return (checkoutList || []).reduce(function (s, i) { return s + i.price; }, 0); }
  function nowLocalInput() {
    var d = new Date(); d.setMinutes(d.getMinutes() - d.getTimezoneOffset());
    return d.toISOString().slice(0, 16);
  }
  function renderCheckout() {
    if (!CK) return;
    var total = checkoutTotal();
    var rows = (checkoutList || []).map(function (i) {
      return '<div style="display:flex;justify-content:space-between;font-size:14px;padding:6px 0"><span>' + escH(i.name) + '</span><span>' + rupees(i.price) + '</span></div>';
    }).join('');
    var inp = 'width:100%;padding:10px 12px;border:1px solid #cbd5e1;border-radius:8px;font-size:14px;box-sizing:border-box;background:#fff';
    var lab = 'display:block;font-size:13px;font-weight:600;margin:14px 0 6px';
    CK.innerHTML =
      '<div style="width:100%;max-width:520px;margin:0 auto;padding:20px 16px 40px">'
      + '<div style="display:flex;justify-content:space-between;align-items:center;margin-bottom:14px"><div style="font-size:22px;font-weight:800">Edu<span style="color:var(--accent)">Sphere</span> <span style="font-size:13px;font-weight:500;color:var(--text-muted)">/ Checkout</span></div>'
      + '<button class="btn btn-secondary btn-sm" onclick="showScreen(\'browse\')">Back</button></div>'
      + '<div style="background:#f8fafc;border:1px solid #e2e8f0;border-radius:12px;padding:16px;margin-bottom:16px"><div style="font-size:12px;font-weight:700;letter-spacing:.06em;color:#64748b;margin-bottom:8px">ORDER SUMMARY</div>' + rows
      + '<div style="display:flex;justify-content:space-between;font-weight:800;font-size:16px;border-top:1px solid #e2e8f0;margin-top:8px;padding-top:10px"><span>Total</span><span>' + rupees(total) + '</span></div></div>'
      + '<div style="background:#fff;border:1px solid #e2e8f0;border-radius:12px;padding:18px;text-align:center"><div style="font-weight:700;margin-bottom:10px">Step 1: scan the QR and pay</div>'
      + '<img src="upi-qr.jpg" alt="Payment QR" style="width:230px;max-width:80%;border-radius:8px">'
      + '<div style="font-size:14px;margin-top:10px">Scan with any UPI app and pay <strong>' + rupees(total) + '</strong></div></div>'
      + '<div style="background:#fff;border:1px solid #e2e8f0;border-radius:12px;padding:18px;margin-top:16px"><div style="font-weight:700">Step 2: enter your payment details</div>'
      + '<label style="' + lab + '">Amount paid (₹)</label><input id="ckAmount" type="number" min="1" step="0.01" inputmode="decimal" placeholder="e.g. ' + total + '" style="' + inp + '">'
      + '<label style="' + lab + '">UPI transaction reference (UTR, 12 digits)</label><input id="utrInput" inputmode="numeric" maxlength="12" placeholder="e.g. 431234567890" style="' + inp + '">'
      + '<label style="' + lab + '">Account holder name (name on the account you paid from)</label><input id="ckPayer" placeholder="e.g. Ramesh Das" style="' + inp + '">'
      + '<label style="' + lab + '">UPI app used</label><select id="ckApp" style="' + inp + '"><option value="">Select</option><option>Google Pay</option><option>PhonePe</option><option>Paytm</option><option>BHIM</option><option>Amazon Pay</option><option>Bank app</option><option>Other</option></select>'
      + '<label style="' + lab + '">Date and time of payment</label><input id="ckWhen" type="datetime-local" value="' + nowLocalInput() + '" max="' + nowLocalInput() + '" style="' + inp + '">'
      + '<button class="btn btn-primary" style="width:100%;justify-content:center;padding:12px;margin-top:18px;font-size:15px" onclick="processPayment()">I have paid: submit for verification</button></div>'
      + '<div style="background:#fffbeb;border:1px solid #fcd34d;color:#78350f;border-radius:12px;padding:14px 16px;margin-top:16px;font-size:13px;line-height:1.6"><strong>Please read</strong><br>'
      + 'Pay the exact total shown above. Content is available only after your payment is verified and approved by the admin, and only if the amount paid matches the price.<br>'
      + 'If the amount is incorrect, whether short or more than the price, the whole amount you paid will be returned to you within 3 to 5 business days at most, and no content will be unlocked. Pay the correct amount and submit again.</div>'
      + '</div>';
  }
  if (CK) {
    var ckLast = '';
    new MutationObserver(function () {
      if (!CK.classList.contains('active')) { ckLast = ''; return; }
      var key = JSON.stringify((checkoutList || []).map(function (i) { return [i.name, i.price]; }));
      if (key !== ckLast || !document.getElementById('utrInput')) { ckLast = key; renderCheckout(); }
    }).observe(CK, { attributes: true, attributeFilter: ['class'] });
  }

  processPayment = async function () {
    var utr = (document.getElementById('utrInput').value || '').trim();
    var amt = parseFloat(document.getElementById('ckAmount').value);
    var payer = (document.getElementById('ckPayer').value || '').trim();
    var app = document.getElementById('ckApp').value;
    var when = document.getElementById('ckWhen').value;
    if (!(amt > 0)) return fail('Enter the amount you paid.');
    if (!/^\d{12}$/.test(utr)) return fail('Enter the 12-digit UTR from your UPI app.');
    if (payer.length < 2) return fail('Enter the account holder name.');
    if (!app) return fail('Select the UPI app you used.');
    if (!when) return fail('Enter the date and time of payment.');
    if (!currentUser) return fail('Please log in first.');
    var items = checkoutList;
    var total = checkoutTotal();
    var entitle = items.reduce(function (a, i) { return a.concat(i.entitle); }, []);
    var seen = {}, paths = [];
    entitle.forEach(function (e) { pathsFor(e).forEach(function (p) { if (!seen[p]) { seen[p] = 1; paths.push(p + '.pdf'); } }); });
    showToast('Submitting...');
    var r = await sb.from('payments').insert({
      user_id: currentUser.uid, student_id: currentUser.id, utr: utr,
      label: items.map(function (i) { return i.name; }).join('; '), items: entitle, file_paths: paths,
      total: total, paid_amount: amt, payer_name: payer, upi_app: app, paid_at: new Date(when).toISOString()
    });
    if (r.error) return fail(r.error.code === '23505' ? 'This UTR has already been submitted.' : 'Could not submit the payment. Please try again.');
    await loadMyPayments();
    if (checkoutFromCart) { clearCart(); checkoutFromCart = false; }
    if (Math.abs(amt - total) < 0.005) showToast('Payment details submitted successfully. Access activates after admin approval.', 'success');
    else showToast('Submitted, but the amount paid does not match the price of ' + rupees(total) + '. It will be returned within 3 to 5 business days and no content will be unlocked.', 'warning');
    showScreen('dashboard');
    var link = document.querySelector('#screen-dashboard .sidebar-nav a[onclick*="purchases"]');
    if (link) link.click();
  };

  // keep the dashboard fresh and persist which receipts the student has seen
  var __rdLive = renderDashboard;
  renderDashboard = function () {
    __rdLive();
    try {
      var s = seenSet();
      payments.forEach(function (p) { if (p.receiptNo && p.receiptSeen && s.indexOf(p.receiptNo) < 0) s.push(p.receiptNo); });
      localStorage.setItem(seenKey, JSON.stringify(s));
    } catch (e) {}
  };
  setInterval(async function () {
    if (adminUser) { await adminRefresh(); }
    else if (currentUser && currentScreen === 'dashboard') { await loadMyPayments(); renderDashboard(); }
  }, 30000);

  // ---------- secure viewer (pdf.js, signed links valid 15 minutes) ----------
  var pdfDoc = null, viewMode = 'scroll', pdfTask = null, scrollObs = null, pageObs = null, pageBox = [], pdfRatio = 1.4;
  try { if (localStorage.getItem('edu_viewmode') === 'page') viewMode = 'page'; } catch (e) {}
  function pdfWidth() { return Math.min(560, (document.getElementById('pdfContent') && document.getElementById('pdfContent').clientWidth) || 540); }
  async function paintPage(n, canvas, task) {
    var page = await pdfDoc.getPage(n);
    var width = pdfWidth();
    var base = page.getViewport({ scale: 1 });
    var vp = page.getViewport({ scale: width / base.width });
    var ratio = window.devicePixelRatio || 1;
    canvas.width = vp.width * ratio; canvas.height = vp.height * ratio;
    canvas.style.width = vp.width + 'px'; canvas.style.height = vp.height + 'px';
    var ctx = canvas.getContext('2d');
    ctx.setTransform(ratio, 0, 0, ratio, 0, 0);
    var t = page.render({ canvasContext: ctx, viewport: vp });
    if (task) task.cur = t;
    await t.promise;
    ctx.save();
    ctx.globalAlpha = 0.09; ctx.fillStyle = '#2563eb'; ctx.font = 'bold 20px sans-serif';
    ctx.translate(vp.width / 2, vp.height / 2); ctx.rotate(-Math.PI / 6);
    var mark = currentUser ? currentUser.id + ' ' + currentUser.name : '';
    ctx.textAlign = 'center';
    for (var y = -vp.height; y < vp.height; y += 110) ctx.fillText(mark, 0, y);
    ctx.restore();
    return vp;
  }
  async function renderPdfPage(n) {
    if (!pdfDoc) return;
    var host = document.getElementById('pdfCanvas');
    if (!host) return;
    if (pdfTask && pdfTask.cur) { try { pdfTask.cur.cancel(); } catch (e) {} }
    var task = pdfTask = { cur: null };
    try { await paintPage(n, host, task); } catch (e) { /* cancelled by a newer page */ }
  }
  function buildScroll() {
    var box = document.getElementById('pdfScroll');
    if (!box || !pdfDoc) return;
    if (scrollObs) scrollObs.disconnect();
    if (pageObs) pageObs.disconnect();
    box.innerHTML = '';
    pageBox = [];
    var w = pdfWidth(), h = Math.round(w * pdfRatio);
    for (var i = 1; i <= pdfDoc.numPages; i++) {
      var d = document.createElement('div');
      d.setAttribute('data-p', i);
      d.style.cssText = 'margin:0 auto 12px;width:' + w + 'px;min-height:' + h + 'px;background:#f1f5f9;border-radius:4px;position:relative';
      var c = document.createElement('canvas');
      c.style.display = 'block';
      d.appendChild(c);
      box.appendChild(d);
      pageBox.push({ el: d, canvas: c, state: 0 });
    }
    scrollObs = new IntersectionObserver(function (entries) {
      entries.forEach(function (en) {
        var p = +en.target.getAttribute('data-p'), b = pageBox[p - 1];
        if (!b) return;
        if (en.isIntersecting && b.state === 0) {
          b.state = 1;
          paintPage(p, b.canvas, null).then(function (vp) { b.state = 2; b.el.style.minHeight = vp.height + 'px'; }, function () { b.state = 0; });
        } else if (!en.isIntersecting && b.state === 2) {
          b.canvas.width = 1; b.canvas.height = 1; b.state = 0;
        }
      });
    }, { rootMargin: '1500px 0px' });
    pageObs = new IntersectionObserver(function (entries) {
      entries.forEach(function (en) {
        if (en.isIntersecting) { currentPage = +en.target.getAttribute('data-p'); updatePageNum(); }
      });
    }, { threshold: 0.5 });
    pageBox.forEach(function (b) { scrollObs.observe(b.el); pageObs.observe(b.el); });
  }
  function applyMode() {
    var one = document.getElementById('pdfSingle'), many = document.getElementById('pdfScroll');
    if (!one || !many) return;
    var pm = document.getElementById('pdfModePage'), sm = document.getElementById('pdfModeScroll');
    var base = 'padding:7px 12px;border-radius:6px;font-size:13px;cursor:pointer;', on = base + 'background:#2563eb;color:#fff;border:1px solid #2563eb', off = base + 'background:#374151;color:#e5e7eb;border:1px solid #4b5563';
    if (pm) pm.style.cssText = viewMode === 'page' ? on : off;
    if (sm) sm.style.cssText = viewMode === 'scroll' ? on : off;
    if (viewMode === 'scroll') {
      one.style.display = 'none'; many.style.display = 'block';
      buildScroll();
      var b = pageBox[currentPage - 1];
      if (b) b.el.scrollIntoView({ block: 'start' });
    } else {
      many.style.display = 'none'; one.style.display = 'block';
      if (scrollObs) scrollObs.disconnect();
      if (pageObs) pageObs.disconnect();
      many.innerHTML = ''; pageBox = [];
      renderPdfPage(currentPage);
    }
  }
  window.setViewMode = function (m) {
    viewMode = m;
    try { localStorage.setItem('edu_viewmode', m); } catch (e) {}
    applyMode();
  };
  window.goToPage = function () {
    var inp = document.getElementById('pdfGoto');
    var n = parseInt(inp && inp.value, 10);
    if (!(n >= 1)) return fail('Enter a page number.');
    if (n > totalPages) { n = totalPages; showToast('This file has ' + totalPages + ' pages. Going to the last page.', 'warning'); }
    currentPage = n;
    updatePageNum();
    if (viewMode === 'scroll') { var b = pageBox[n - 1]; if (b) b.el.scrollIntoView({ block: 'start' }); }
    else renderPdfPage(n);
  };
  var _upn = updatePageNum;
  updatePageNum = function () {
    _upn();
    var inp = document.getElementById('pdfGoto');
    if (inp && document.activeElement !== inp) inp.value = currentPage;
  };
  // the page and mode controls live in the dark header bar of the viewer
  function ensureHeadBar() {
    var old = document.getElementById('pdfHeadBar');
    if (old && old.isConnected) return;
    var nav = document.querySelector('#screen-viewer [onclick*="changePage"]');
    var host = null;
    if (nav) {
      var el = nav;
      while (el.parentNode && el.parentNode !== document.body && !/Download/.test(el.parentNode.textContent || '')) el = el.parentNode;
      host = (el.parentNode && el.parentNode !== document.body) ? el.parentNode : nav.parentNode;
    }
    var bar = document.createElement('div');
    bar.id = 'pdfHeadBar';
    bar.style.cssText = 'display:flex;flex-wrap:wrap;gap:10px;align-items:center;justify-content:center;padding:10px 12px;width:100%;box-sizing:border-box;font-size:13px;color:#e2e8f0';
    var bs = 'padding:7px 12px;border:1px solid #4b5563;border-radius:6px;background:#374151;color:#e5e7eb;font-size:13px;cursor:pointer';
    bar.innerHTML = '<span><button id="pdfModePage" style="' + bs + '" onclick="setViewMode(\'page\')">Page mode</button> <button id="pdfModeScroll" style="' + bs + '" onclick="setViewMode(\'scroll\')">Scroll mode</button></span>'
      + '<span>Go to page <input id="pdfGoto" type="number" min="1" inputmode="numeric" style="width:70px;padding:6px 8px;border:1px solid #4b5563;border-radius:6px;background:#1f2937;color:#f9fafb" onkeydown="if(event.key===\'Enter\')goToPage()"> <button style="' + bs + '" onclick="goToPage()">Go</button></span>';
    if (host) host.insertAdjacentElement('afterend', bar);
    else { var c = document.getElementById('pdfContent'); if (c) c.insertBefore(bar, c.firstChild); }
  }
  async function openSecure(path, title) {
    if (!window.pdfjsLib) return fail('The PDF viewer could not load. Please check your internet connection.');
    var u = await sb.storage.from('materials').createSignedUrl(path + '.pdf', 900);
    if (u.error || !u.data) return fail('This file is not available yet, or you do not have access to it.');
    pdfjsLib.GlobalWorkerOptions.workerSrc = 'https://cdn.jsdelivr.net/npm/pdfjs-dist@3.11.174/build/pdf.worker.min.js';
    try {
      showViewer(title, '<div id="pdfSingle" style="text-align:center"><canvas id="pdfCanvas"></canvas></div><div id="pdfScroll" style="display:none"></div>');
      ensureHeadBar();
      pdfDoc = await pdfjsLib.getDocument({ url: u.data.signedUrl }).promise;
      totalPages = pdfDoc.numPages;
      currentPage = 1;
      try { var f = await pdfDoc.getPage(1), v = f.getViewport({ scale: 1 }); pdfRatio = v.height / v.width; } catch (e) {}
      updatePageNum();
      applyMode();
    } catch (e) {
      pdfDoc = null;
      fail('Could not open this file.');
    }
  }
  changePage = function (dir) {
    currentPage = Math.max(1, Math.min(totalPages, currentPage + dir));
    updatePageNum();
    if (viewMode === 'scroll') { var b = pageBox[currentPage - 1]; if (b) b.el.scrollIntoView({ block: 'start', behavior: 'smooth' }); }
    else renderPdfPage(currentPage);
  };
  openStpgt = function (sid, lvl) { openSecure('stpgt/' + sid + '/' + lvl, 'STPGT ' + STPGT.subjects[sid].name + ' (' + STPGT.subjects[sid].levels[lvl].name + ' course)'); };
  openChapter = function (id, lvl, ch) { openSecure('cbse/' + id + '/' + lvl + '/ch' + ch, classTitle(COURSES[id]) + ' (' + LEVELS[lvl] + '), Chapter ' + ch); };
  openTet = function (paper, slab, subj) { openSecure('tet/' + paper + '/' + slug(subj) + '/' + slab, 'Tripura TET ' + paper + ', ' + subj + ' (' + SLABS[slab] + ' Course)'); };
  openTetCombo = function (p, j, sl) { openSecure('tet/' + p + '/combo' + j + '/' + sl, 'Tripura TET ' + p + ', ' + TET[p].combos[j].name + ' (' + SLABS[sl] + ' Course)'); };
  openViewer = function (title) {
    var lf = LIB.byTitle[title];
    if (lf) return openSecure(lf.path, title);
    showToast('This material is not available yet.', 'warning');
  };

  // ---------- admin ----------
  doAdminLogin = async function () {
    var id = document.getElementById('adminId').value.trim();
    var pw = document.getElementById('adminPw').value;
    if (!id || !pw) return fail('Enter the admin ID and password.');
    var prof = await signInWith(id, pw);
    if (!prof || prof.role !== 'admin') {
      await sb.auth.signOut();
      return fail('Invalid admin ID or password.');
    }
    adminUser = { id: prof.student_id, name: prof.name, password: '' };
    document.getElementById('adminId').value = '';
    document.getElementById('adminPw').value = '';
    document.getElementById('adminLoginBox').style.display = 'none';
    var sp = document.querySelector('#screen-admin .topbar-actions span');
    if (sp) sp.textContent = adminUser.id + ' (' + adminUser.name + ')';
    await loadMaterials();
    await loadAdminData();
    logEvent(adminUser.id, 'ADMIN_LOGIN', 'admin_panel', 'OK');
    showScreen('admin');
    switchAdminTab('overview', document.querySelector('#screen-admin .sidebar-nav a'));
    showToast('Welcome, ' + adminUser.id + '.', 'success');
  };

  async function setPayment(utr, status, evt, msg, kind) {
    var p = payments.filter(function (x) { return x.utr === utr; })[0];
    var r = await sb.rpc('admin_set_payment', { p_utr: utr, p_status: status });
    if (r.error) return fail(/AMOUNT_MISMATCH/.test(r.error.message || '') ? 'Cannot approve: the amount paid does not match the price. Reject it and refund the student.' : 'Could not update the payment.');
    logEvent(adminId(), evt + ' · ' + (p ? p.sid : '') + ' · UTR ' + utr);
    await adminRefresh((status === 'Paid' && r.data) ? 'Approved. Access activated and receipt ' + r.data + ' issued.' : msg, kind);
  }
  approvePayment = function (utr) { return setPayment(utr, 'Paid', 'APPROVE_PAYMENT', 'Approved.', 'success'); };
  rejectPayment = function (utr) { return setPayment(utr, 'Rejected', 'REJECT_PAYMENT', 'Payment rejected. No access was granted.', 'warning'); };
  revokePayment = function (utr) { return setPayment(utr, 'Rejected', 'REVOKE_ACCESS', 'Access revoked for this payment.', 'warning'); };
  reopenPayment = function (utr) { return setPayment(utr, 'Awaiting verification', 'REOPEN_PAYMENT', 'Payment moved back to awaiting verification.', ''); };

  function acc(id) { return accounts.filter(function (a) { return a.id === id; })[0]; }
  toggleBlock = async function (id) {
    var a = acc(id); if (!a) return;
    var blocking = a.status !== 'Blocked';
    var r = await sb.from('profiles').update({ status: blocking ? 'Blocked' : 'Active' }).eq('id', a.uid);
    if (r.error) return fail('Could not update the account.');
    logEvent(adminId(), (blocking ? 'BLOCK_ACCOUNT · ' : 'UNBLOCK_ACCOUNT · ') + id);
    await adminRefresh(blocking ? id + ' blocked. Login is denied.' : id + ' unblocked.', blocking ? 'warning' : 'success');
  };
  resetDevice = async function (id) {
    var a = acc(id); if (!a) return;
    var r = await sb.from('profiles').update({ device_fp: null, device_label: null, status: 'Active' }).eq('id', a.uid);
    if (r.error) return fail('Could not reset the device.');
    await sb.from('device_requests').update({ state: 'Approved' }).eq('user_id', a.uid).eq('state', 'Open');
    logEvent(adminId(), 'RESET_DEVICE · ' + id);
    await adminRefresh('Device binding reset for ' + id + '. The next login registers a new device.', 'warning');
  };
  killSession = async function (id) {
    var a = acc(id); if (!a) return;
    var r = await sb.rpc('admin_kill_sessions', { target: a.uid });
    if (r.error) return fail('Could not end the session.');
    logEvent(adminId(), 'KILL_SESSION · ' + id);
    await adminRefresh('Sessions ended for ' + id + '.', 'warning');
  };
  resetPassword = async function (id) {
    var a = acc(id); if (!a) return;
    var tmp = 'Temp@' + (1000 + Math.floor(Math.random() * 9000));
    var r = await sb.rpc('admin_reset_password', { target: a.uid, new_pw: tmp });
    if (r.error) return fail('Could not reset the password.');
    logEvent(adminId(), 'RESET_PASSWORD · ' + id);
    await loadAdminData();
    refreshAdmin();
    openAdminModal('Password reset', '<p style="font-size:13px">Temporary password for <strong>' + esc(id) + '</strong>:</p><div style="background:#f1f5f9;padding:12px 16px;border-radius:6px;font-family:monospace;margin:12px 0">' + esc(tmp) + '</div><p style="font-size:12px;color:var(--text-muted)">Share it with the student or parent on WhatsApp.</p>', '');
  };
  deleteAccount = async function (id) {
    var a = acc(id); if (!a) return;
    var r = await sb.rpc('admin_delete_user', { target: a.uid });
    pendingDelete = null;
    if (r.error) return fail('Could not delete the account.');
    logEvent(adminId(), 'DELETE_ACCOUNT · ' + id);
    await adminRefresh('Account ' + id + ' deleted. Its phone number and email can be used again. Payment records stay in the accounting sheet.', 'warning');
  };
  newStudentModal = function () {
    openAdminModal('New student account', '<p style="font-size:13px">Students create their own accounts on the sign-up page. The Student ID is assigned automatically, for example ES26-0001.</p>', '');
  };
  approveDevReq = async function (i) {
    var r = deviceRequests[i]; if (!r) return;
    var x = await sb.rpc('admin_device_request', { p_id: r.id, p_approve: true });
    if (x.error) return fail('Could not approve the request.');
    logEvent(adminId(), 'APPROVE_DEVICE_CHANGE · ' + r.sid);
    await adminRefresh('Device change approved for ' + r.sid + '. The student can log in from the new device.', 'success');
  };
  denyDevReq = async function (i) {
    var r = deviceRequests[i]; if (!r) return;
    var x = await sb.rpc('admin_device_request', { p_id: r.id, p_approve: false });
    if (x.error) return fail('Could not deny the request.');
    logEvent(adminId(), 'DENY_DEVICE_CHANGE · ' + r.sid);
    await adminRefresh('Device change denied for ' + r.sid + '.', 'warning');
  };
  overrideDevReq = function () {};


  // ---------- admin: materials stock ----------
  async function listFolder(prefix) {
    var out = {};
    var r = await sb.storage.from('materials').list(prefix, { limit: 200 });
    (r.data || []).forEach(function (f) {
      if (f.id) out[prefix + '/' + f.name.replace(/\.pdf$/i, '')] = (f.metadata && f.metadata.size) || 0;
    });
    return out;
  }
  function fmtSize(b) { return b ? (b / 1048576).toFixed(1) + ' MB' : '-'; }
  async function renderStock() {
    var box = document.getElementById('admin-materials');
    if (!box) return;
    box.innerHTML = '<div class="page-header"><h1>Materials stock</h1><p>Loading...</p></div>';
    var rowsDb = (await sb.from('materials').select('*')).data || [];
    var meta = {}; rowsDb.forEach(function (r) { meta[r.path] = r; });
    var folders = ['stpgt/phy'];
    ['I', 'II'].forEach(function (p) {
      if (!TET[p]) return;
      TET[p].subjects.forEach(function (x) { folders.push('tet/' + p + '/' + slug(x[0])); });
      TET[p].combos.forEach(function (c, j) { folders.push('tet/' + p + '/combo' + j); });
    });
    var files = {};
    var parts = await Promise.all(folders.map(listFolder));
    parts.forEach(function (o) { Object.keys(o).forEach(function (k) { files[k] = o[k]; }); });
    var items = [];
    function add(cat, name, base, mult, stp, fees) {
      var n = stp ? STPGT.subjects.phy.levels.length : 4;
      for (var i = 0; i < n; i++) items.push({ cat: cat, name: name, level: stp ? STPGT.subjects.phy.levels[i].name : SLABS[i], path: base + '/' + i, fee: fees ? fees[i] : null, mcq: stp ? (STPGT.subjects.phy.levels[i].mcq || null) : SLAB_MCQ[i] * (mult || 1) });
    }
    add('STPGT', 'Physics', 'stpgt/phy', 1, true, STPGT.subjects.phy.levels.map(function (l) { return l.fee; }));
    ['I', 'II'].forEach(function (p) {
      if (!TET[p]) return;
      TET[p].subjects.forEach(function (x) { add('TET ' + p, x[0], 'tet/' + p + '/' + slug(x[0]), 1, false, x[1]); });
      TET[p].combos.forEach(function (c, j) { add('TET ' + p + ' combo', c.name, 'tet/' + p + '/combo' + j, c.subjects.length, false, c.fees); });
    });
    stockItems = items;
    var known = {}; items.forEach(function (it) { known[it.path] = 1; });
    var cb = {};
    rowsDb.forEach(function (r) { if (r.path.indexOf('cbse/') === 0) { var k = r.path.split('/').slice(0, 3).join('/'); cb[k] = (cb[k] || 0) + 1; } });
    var n = { ok: 0, soon: 0, attn: 0 };
    var html = '';
    items.forEach(function (it) {
      var hasFile = files.hasOwnProperty(it.path), m = meta[it.path], st, col, note = '';
      if (hasFile && m) { st = 'In stock'; col = '#15803d'; n.ok++; }
      else if (hasFile && !m) { st = 'Needs attention'; col = '#b45309'; note = 'File uploaded but no row in the materials table, so students see Coming soon.'; n.attn++; }
      else if (!hasFile && m) { st = 'Needs attention'; col = '#b45309'; note = 'Row exists but the PDF file is missing, so a student could buy it and get nothing.'; n.attn++; }
      else { st = 'Coming soon'; col = '#64748b'; n.soon++; }
      html += '<tr><td>' + escH(it.cat) + '</td><td><strong>' + escH(it.name) + '</strong></td><td>' + escH(it.level) + '</td>'
        + '<td style="color:' + col + ';font-weight:600"' + (note ? ' title="' + escH(note) + '"' : '') + '>' + st + (note ? ' <span style="font-weight:500">(!)</span>' : '') + '</td>'
        + '<td style="font-weight:700">' + (it.fee != null ? rupees(it.fee) : '-') + '</td>'
        + '<td>' + (m && m.pages ? m.pages : '-') + '</td><td>' + (m && m.mcq ? m.mcq : '-') + '</td><td>' + fmtSize(files[it.path]) + '</td>'
        + '<td style="white-space:nowrap"><button class="btn btn-secondary btn-sm" onclick="stockUpload(' + items.indexOf(it) + ')">' + (hasFile ? 'Replace' : 'Upload') + '</button>'
        + (hasFile ? ' <button class="btn btn-secondary btn-sm" onclick="stockView(' + items.indexOf(it) + ')">View</button>' : '')
        + ((hasFile || m) ? ' <button class="btn btn-secondary btn-sm" onclick="stockDelete(' + items.indexOf(it) + ')">Delete</button>' : '') + '</td></tr>';
    });
    var cbRows = Object.keys(cb).map(function (k) { return '<tr><td>CBSE</td><td><strong>' + escH(k.split('/')[1]) + '</strong></td><td>' + (LEVELS[+k.split('/')[2]] || k.split('/')[2]) + '</td><td style="color:#15803d;font-weight:600">In stock</td><td colspan="5">' + cb[k] + ' chapter file(s)</td></tr>'; }).join('');
    var extra = Object.keys(files).filter(function (k) { return !known[k]; });
    box.innerHTML = '<div class="page-header"><h1>Materials stock</h1><p>What is uploaded and what students can buy</p></div>'
      + '<div style="display:flex;gap:10px;align-items:center;margin-bottom:14px;flex-wrap:wrap">'
      + '<span style="font-size:13px"><strong style="color:#15803d">' + n.ok + '</strong> in stock &nbsp; <strong style="color:#64748b">' + n.soon + '</strong> coming soon &nbsp; <strong style="color:#b45309">' + n.attn + '</strong> need attention</span>'
      + '<button class="btn btn-secondary btn-sm" onclick="renderStock()">Refresh</button></div>'
      + '<div class="table-wrap"><table class="data-table"><thead><tr><th>Category</th><th>Item</th><th>Level</th><th>Status</th><th>Price</th><th>Pages</th><th>MCQs</th><th>Size</th><th>Action</th></tr></thead><tbody>' + html + cbRows + '</tbody></table></div>'
      + (extra.length ? '<p style="font-size:12px;color:var(--text-muted);margin-top:10px">Files not matching any listed item: ' + escH(extra.join(', ')) + '</p>' : '')
      + '<p style="font-size:12px;color:var(--text-muted);margin-top:10px">Upload adds a PDF from this device (PDF only, up to 50 MB). Replace swaps the file and keeps the price. Delete removes it and the item goes back to Coming soon. For a new exam, use the Library tab.</p>'
      + '<input type="file" id="stockPick" accept="application/pdf,.pdf" style="display:none">';
    var sp = document.getElementById('stockPick');
    if (sp) sp.onchange = stockPicked;
  }
  var stockItems = [], stockTarget = null;
  window.stockUpload = function (i) {
    stockTarget = stockItems[i];
    var sp = document.getElementById('stockPick');
    if (!stockTarget || !sp) return;
    sp.value = ''; sp.click();
  };
  async function stockPicked() {
    var sp = document.getElementById('stockPick'), file = sp && sp.files[0], it = stockTarget;
    if (!file || !it) return;
    if (!/\.pdf$/i.test(file.name) && file.type !== 'application/pdf') return fail('Choose a PDF file.');
    if (file.size > 50 * 1048576) return fail('That file is larger than 50 MB.');
    var label = it.cat + ' ' + it.name + ' (' + it.level + ')';
    var exists = (await sb.storage.from('materials').list(it.path.split('/').slice(0, -1).join('/'), { limit: 200, search: it.path.split('/').pop() })).data || [];
    exists = exists.some(function (f) { return f.name === it.path.split('/').pop() + '.pdf'; });
    if (exists && !confirm('Replace the current PDF for ' + label + '? Students who bought it will see the new file.')) return;
    showToast('Uploading ' + file.name + '...');
    var up = await sb.storage.from('materials').upload(it.path + '.pdf', file, { contentType: 'application/pdf', upsert: true });
    if (up.error) return fail('Upload failed. ' + (up.error.message || ''));
    var pages = await pdfPages(file);
    var old = (await sb.from('materials').select('*').eq('path', it.path)).data || [];
    var row = { path: it.path, title: (old[0] && old[0].title) || label, pages: pages || (old[0] && old[0].pages) || null, mcq: (old[0] && old[0].mcq) || it.mcq };
    var r = await sb.from('materials').upsert(row, { onConflict: 'path' });
    if (r.error) return fail('The file uploaded but the catalogue row could not be saved.');
    logEvent(adminId(), (exists ? 'REPLACE_FILE' : 'UPLOAD_FILE') + ' \u00b7 ' + label, 'admin_panel', 'OK');
    await loadMaterials();
    showToast((exists ? 'Replaced: ' : 'Uploaded: ') + label + '. Students can now buy it.', 'success');
    renderStock();
  }
  var adminViewOn = false;
  function adminView(path, title) {
    if (!adminUser) return;
    var saved = currentUser;
    var btn = document.querySelector('#screen-viewer .topbar-actions .btn');
    var origClick = btn ? btn.getAttribute('onclick') : null, origText = btn ? btn.textContent : '';
    function restore() {
      currentUser = saved; adminViewOn = false;
      if (btn) { btn.textContent = origText; if (origClick) btn.setAttribute('onclick', origClick); btn.onclick = null; }
    }
    currentUser = { id: adminUser.id, name: adminUser.name, status: 'Active', owned: [] };
    adminViewOn = true;
    var wm = document.getElementById('pdfPage');
    if (wm) wm.setAttribute('data-watermark', adminUser.id + ' \u00b7 ' + adminUser.name);
    if (btn) {
      btn.textContent = '\u2190 Back to Admin';
      btn.removeAttribute('onclick');
      btn.onclick = function () { restore(); showScreen('admin'); };
    }
    openSecure(path, title).then(function () { if (currentScreen !== 'viewer') restore(); }, restore);
  }
  window.stockView = function (i) { var it = stockItems[i]; if (it) adminView(it.path, it.cat + ': ' + it.name + ' (' + it.level + ')'); };
  window.stockDelete = async function (i) {
    var it = stockItems[i]; if (!it) return;
    var label = it.cat + ' ' + it.name + ' (' + it.level + ')';
    var nb = await buyersOf([it.path + '.pdf']);
    if (!confirm('Delete the PDF for ' + label + '?' + (nb ? '\n\n' + nb + ' payment(s) already include this file. Those students will lose access.' : '') + '\n\nThis cannot be undone.')) return;
    await sb.storage.from('materials').remove([it.path + '.pdf']);
    var r = await sb.from('materials').delete().eq('path', it.path);
    if (r.error) return fail('Could not remove the catalogue row.');
    logEvent(adminId(), 'DELETE_FILE \u00b7 ' + label, 'admin_panel', 'OK');
    await loadMaterials();
    showToast('Deleted: ' + label + '.', 'success');
    renderStock();
  };
  window.renderStock = renderStock;
  var _switchTab = switchAdminTab;
  switchAdminTab = function (tab, el) {
    _switchTab(tab, el);
    if (tab === 'materials') renderStock();
  };

  // ---------- admin: pending payment requests ----------
  function amountCheck(p) {
    if (p.paidAmount == null) return { ok: false, html: '<span style="color:#b45309;font-weight:600">No amount recorded</span>' };
    var d = Math.round((p.paidAmount - p.total) * 100) / 100;
    if (d === 0) return { ok: true, html: '<span style="color:#15803d;font-weight:600">Amount matches. Successful.</span>' };
    return { ok: false, html: '<span style="color:#b91c1c;font-weight:600">Mismatch: ' + (d < 0 ? 'short by ' + rupees(-d) : 'excess of ' + rupees(d)) + '</span><br><span style="font-size:11px;color:#b91c1c">Return ' + rupees(p.paidAmount) + ' in full within 3 to 5 business days</span>' };
  }
  function payRows(list, pending) {
    return list.map(function (p) {
      var a = amountCheck(p), acc = accounts.filter(function (x) { return x.id === p.sid; })[0];
      var act = '';
      if (pending) {
        act = (a.ok ? '<button class="btn btn-primary btn-sm" onclick="approvePayment(\'' + p.utr + '\')">Approve</button> ' : '<button class="btn btn-secondary btn-sm" disabled title="Amount does not match the price">Approve</button> ')
          + '<button class="btn btn-secondary btn-sm" onclick="rejectPayment(\'' + p.utr + '\')">' + (a.ok ? 'Reject' : 'Reject and refund') + '</button>';
      } else if (p.status === 'Paid') {
        act = '<button class="btn btn-secondary btn-sm" onclick="revokePayment(\'' + p.utr + '\')">Revoke</button>';
      } else if (p.status === 'Rejected') {
        act = (p.refundStatus === 'Due' ? '<span style="color:#b91c1c;font-weight:600">Refund due</span>' : p.refundStatus === 'Returned' ? '<span style="color:#15803d;font-weight:600">Refund returned</span>' : (a.ok ? '<button class="btn btn-secondary btn-sm" onclick="reopenPayment(\'' + p.utr + '\')">Reopen</button>' : ''));
      }
      return '<tr><td><strong>' + escH(p.sid) + '</strong><br><span style="font-size:11px;color:var(--text-muted)">' + escH(acc ? acc.name : '') + '</span></td>'
        + '<td style="font-size:12px;max-width:200px">' + escH(p.label) + '</td>'
        + '<td>' + rupees(p.total) + '</td><td>' + (p.paidAmount == null ? '-' : rupees(p.paidAmount)) + '</td>'
        + '<td>' + a.html + '</td>'
        + '<td style="font-family:monospace">' + escH(p.utr) + '</td><td>' + escH(p.payer) + '</td><td>' + escH(p.upiApp) + '</td>'
        + '<td style="font-size:12px">' + fmtDT(p.paidAt) + '</td><td style="font-size:12px">' + fmtDT(p.createdAt) + '</td>'
        + '<td>' + (pending ? '' : '<strong>' + escH(p.status) + '</strong><br>' + escH(p.receiptNo || '') + '<br>') + act + '</td></tr>';
    }).join('');
  }
  var PAYHEAD = '<thead><tr><th>Student</th><th>Items</th><th>Price</th><th>Paid</th><th>Check</th><th>UTR</th><th>Account holder</th><th>UPI app</th><th>Paid on</th><th>Submitted</th><th>Action</th></tr></thead>';
  function renderPayPanel() {
    var box = document.getElementById('admin-payments');
    if (!box) return;
    var rest = payments.filter(function (p) { return p.status !== 'Awaiting verification'; }).slice().reverse();
    var waiting = payments.length - rest.length;
    box.innerHTML = '<div class="page-header"><h1>Payment history</h1><p>' + (waiting ? waiting + ' payment request(s) are waiting in the Approvals tab.' : 'Approved and rejected payments.') + '</p></div>'
      + '<div class="table-wrap"><table class="data-table">' + PAYHEAD + '<tbody>' + (rest.length ? payRows(rest, false) : '<tr><td colspan="11" style="text-align:center;color:var(--text-muted);padding:18px">Nothing yet.</td></tr>') + '</tbody></table></div>';
  }

  // ---------- admin: approvals (pending payments and device changes only) ----------
  function pendingPays() { return payments.filter(function (p) { return p.status === 'Awaiting verification'; }).slice().reverse(); }
  function pendingDevs() { return deviceRequests.map(function (r, i) { return { r: r, i: i }; }).filter(function (x) { return x.r.state === 'Open'; }); }
  function mismatched() { return pendingPays().filter(function (p) { return !amountCheck(p).ok; }); }
  function refundsDue() { return payments.filter(function (p) { return p.refundStatus === 'Due'; }); }
  function addBusinessDays(d, n) {
    var x = new Date(d); var k = 0;
    while (k < n) { x.setDate(x.getDate() + 1); var w = x.getDay(); if (w !== 0 && w !== 6) k++; }
    return x;
  }
  function ensureApprovalsTab() {
    var nav = document.querySelector('#screen-admin .sidebar-nav');
    var ref = document.getElementById('admin-payments');
    if (!nav || !ref) return;
    if (!document.getElementById('admin-approvals')) {
      var d = document.createElement('div');
      d.id = 'admin-approvals'; d.className = 'admin-tab-content'; d.style.display = 'none';
      ref.parentNode.insertBefore(d, ref.parentNode.firstChild);
    }
    if (!document.getElementById('navApprovals')) {
      var a = document.createElement('a');
      a.id = 'navApprovals';
      a.onclick = function () { switchAdminTab('approvals', a); };
      nav.insertBefore(a, nav.firstChild);
    }
  }
  function field(label, val) {
    return '<div><div style="font-size:11px;color:#64748b;margin-bottom:2px">' + label + '</div><div style="font-size:13px;font-weight:600;word-break:break-word">' + (val || '-') + '</div></div>';
  }
  function payCard(p) {
    var a = amountCheck(p), acc = accounts.filter(function (x) { return x.id === p.sid; })[0] || {};
    var border = a.ok ? '#e2e8f0' : '#fca5a5';
    return '<div style="background:#fff;border:2px solid ' + border + ';border-radius:12px;padding:16px;margin-bottom:14px">'
      + '<div style="display:flex;justify-content:space-between;flex-wrap:wrap;gap:8px;margin-bottom:12px"><div><strong style="font-size:15px">' + escH(p.sid) + '</strong> &nbsp;' + escH(acc.name || '') + '</div><div>' + a.html + '</div></div>'
      + '<div style="font-size:13px;margin-bottom:12px">' + escH(p.label) + '</div>'
      + '<div style="display:grid;grid-template-columns:repeat(auto-fit,minmax(150px,1fr));gap:12px;margin-bottom:14px">'
      + field('Price', rupees(p.total)) + field('Amount paid', p.paidAmount == null ? '-' : rupees(p.paidAmount))
      + field('UTR', escH(p.utr)) + field('Account holder', escH(p.payer)) + field('UPI app', escH(p.upiApp))
      + field('Payment date and time', fmtDT(p.paidAt)) + field('Submitted on', fmtDT(p.createdAt))
      + field('Student phone', escH(acc.mobile || '')) + field('Parent phone', escH(acc.parentMobile || '')) + '</div>'
      + '<div style="display:flex;gap:8px;flex-wrap:wrap">'
      + (a.ok ? '<button class="btn btn-primary btn-sm" onclick="approvePayment(\'' + p.utr + '\')">Approve</button> <button class="btn btn-secondary btn-sm" onclick="rejectPayment(\'' + p.utr + '\')">Reject</button>'
              : '<button class="btn btn-secondary btn-sm" disabled>Approve (blocked: amount differs)</button> <button class="btn btn-primary btn-sm" style="background:#dc2626;border-color:#dc2626" onclick="rejectPayment(\'' + p.utr + '\')">Reject and set refund reminder</button>')
      + '</div></div>';
  }
  var markRefunded = async function (utr) {
    var r = await sb.rpc('admin_mark_refunded', { p_utr: utr });
    if (r.error) return fail('Could not update the refund.');
    logEvent(adminId(), 'REFUND_RETURNED \u00b7 UTR ' + utr);
    await adminRefresh('Refund marked as returned.', 'success');
  };
  window.markRefunded = markRefunded;
  function renderApprovals() {
    ensureApprovalsTab();
    var box = document.getElementById('admin-approvals'), nav = document.getElementById('navApprovals');
    if (!box) return;
    var pp = pendingPays(), dv = pendingDevs(), bad = mismatched(), rf = refundsDue();
    var total = pp.length + dv.length + rf.length;
    if (nav) nav.innerHTML = '\u2705&nbsp; Approvals ' + (total ? '<span style="background:' + ((bad.length || rf.length) ? '#dc2626' : '#2563eb') + ';color:#fff;border-radius:10px;padding:1px 8px;font-size:11px;margin-left:6px">' + total + '</span>' : '');
    var refundBox = '';
    if (rf.length) {
      refundBox = '<div style="background:#fef2f2;border:2px solid #dc2626;border-radius:12px;padding:16px;margin-bottom:20px"><div style="font-weight:800;color:#991b1b;margin-bottom:8px">\u26a0 Refunds to return (' + rf.length + ')</div>'
        + '<div style="font-size:12px;color:#7f1d1d;margin-bottom:10px">Each reminder stays here until you tick Returned.</div>'
        + rf.map(function (p) {
          var acc = accounts.filter(function (x) { return x.id === p.sid; })[0] || {};
          var by = p.decidedAt ? addBusinessDays(p.decidedAt, 5) : null;
          return '<div style="background:#fff;border:1px solid #fca5a5;border-radius:10px;padding:12px 14px;margin-bottom:10px;display:flex;justify-content:space-between;gap:12px;flex-wrap:wrap;align-items:center">'
            + '<div style="font-size:13px;line-height:1.7"><strong>Return ' + rupees(p.paidAmount) + '</strong> to ' + escH(p.payer) + ' (' + escH(p.sid) + ' ' + escH(acc.name || '') + ') via ' + escH(p.upiApp) + '<br>'
            + 'UTR ' + escH(p.utr) + ' \u00b7 paid ' + fmtDT(p.paidAt) + ' \u00b7 price was ' + rupees(p.total) + '<br>'
            + 'Phone: ' + escH(acc.mobile || '-') + ' / parent ' + escH(acc.parentMobile || '-') + (by ? '<br><strong>Return by ' + by.toLocaleDateString('en-GB', { day: '2-digit', month: 'short', year: 'numeric' }) + '</strong> (3 to 5 business days)' : '') + '</div>'
            + '<button class="btn btn-primary btn-sm" onclick="markRefunded(\'' + p.utr + '\')">\u2611 Returned</button></div>';
        }).join('') + '</div>';
    }
    var alert = '';
    if (bad.length) {
      alert = '<div style="background:#fef2f2;border:1px solid #fca5a5;color:#7f1d1d;border-radius:10px;padding:14px 16px;margin-bottom:18px;font-size:13px;line-height:1.7"><strong>\u26a0 Incorrect amount paid (' + bad.length + ')</strong><br>'
        + bad.map(function (p) {
          var d = Math.round((p.paidAmount - p.total) * 100) / 100;
          return escH(p.sid) + ' entered ' + (p.paidAmount == null ? 'no amount' : rupees(p.paidAmount)) + ' for a price of ' + rupees(p.total) + (p.paidAmount == null ? '' : (d < 0 ? ' (short by ' + rupees(-d) + ')' : ' (excess of ' + rupees(d) + ')')) + '. UTR ' + escH(p.utr) + '. Reject it to set a refund reminder.';
        }).join('<br>') + '</div>';
    } else if (pp.length) {
      alert = '<div style="background:#f0fdf4;border:1px solid #86efac;color:#166534;border-radius:10px;padding:12px 16px;margin-bottom:18px;font-size:13px">All pending payment amounts match their prices. Check each UTR in your UPI app, then approve.</div>';
    }
    var devRows = dv.map(function (x) {
      var acc = accounts.filter(function (a) { return a.id === x.r.sid; })[0];
      return '<tr><td><strong>' + escH(x.r.sid) + '</strong></td><td>' + escH(acc ? acc.name : '') + '</td><td>' + escH(x.r.when) + '</td><td><button class="btn btn-primary btn-sm" onclick="approveDevReq(' + x.i + ')">Approve</button> <button class="btn btn-secondary btn-sm" onclick="denyDevReq(' + x.i + ')">Deny</button></td></tr>';
    }).join('');
    box.innerHTML = '<div class="page-header"><h1>Approvals</h1><p>' + pp.length + ' payment request(s), ' + dv.length + ' device change request(s) and ' + rf.length + ' refund(s) to return. Students get access only after you approve.</p></div>'
      + refundBox + alert
      + '<div class="section-title" style="margin:8px 0 10px;font-weight:700">Pending payment requests for me to approve (' + pp.length + ')</div>'
      + (pp.length ? pp.map(payCard).join('') : '<div style="color:var(--text-muted);padding:10px 0 20px;font-size:13px">No pending payments.</div>')
      + '<div class="section-title" style="margin:18px 0 10px;font-weight:700">Device change requests (' + dv.length + ')</div>'
      + '<div class="table-wrap"><table class="data-table"><thead><tr><th>Student ID</th><th>Name</th><th>Requested</th><th>Action</th></tr></thead><tbody>' + (dv.length ? devRows : '<tr><td colspan="4" style="text-align:center;color:var(--text-muted);padding:18px">No device change requests.</td></tr>') + '</tbody></table></div>';
  }
  var _rejectBase = rejectPayment;
  rejectPayment = async function (utr) {
    var p = payments.filter(function (x) { return x.utr === utr; })[0];
    var wrong = p && p.paidAmount != null && Math.abs(p.paidAmount - p.total) >= 0.005;
    await _rejectBase(utr);
    if (wrong) showToast('Rejected. Return ' + rupees(p.paidAmount) + ' to ' + p.payer + ' within 3 to 5 business days. The reminder stays in Approvals until you tick Returned.', 'warning');
  };

  var _refreshAdmin = refreshAdmin;
  refreshAdmin = function () { _refreshAdmin(); renderPayPanel(); renderApprovals(); };
  var _switchTab2 = switchAdminTab;
  switchAdminTab = function (tab, el) { _switchTab2(tab, el); if (tab === 'payments') renderPayPanel(); if (tab === 'approvals') renderApprovals(); };
  var _doAdminLogin = doAdminLogin;
  doAdminLogin = async function () {
    await _doAdminLogin();
    if (!adminUser) return;
    renderApprovals();
    var n = pendingPays().length + pendingDevs().length, bad = mismatched().length, rf = refundsDue().length;
    switchAdminTab('approvals', document.getElementById('navApprovals'));
    if (rf) showToast(rf + ' refund(s) still to return. See Approvals.', 'warning');
    else if (bad) showToast(bad + ' payment(s) have an incorrect amount. See Approvals.', 'warning');
    else if (n) showToast(n + ' item(s) waiting for your approval.', 'success');
  };

  // ---------- library: exam folders made by the admin ----------
  var LIB = { exams: [], folders: [], files: [], byTitle: {}, ready: false };
  var PAL = [['#2b3fd6', '#6a3df0', 'rgba(43,63,214,.8)'], ['#0a8f86', '#2b8fd6', 'rgba(10,143,134,.8)'], ['#e4572e', '#f2a20c', 'rgba(228,87,46,.8)'],
    ['#121a3a', '#3a4aa8', 'rgba(18,26,58,.8)'], ['#b3237b', '#e4572e', 'rgba(179,35,123,.8)'], ['#0b7a4b', '#7cc418', 'rgba(11,122,75,.8)']];
  function libExam(id) { return LIB.exams.filter(function (e) { return e.id === id; })[0]; }
  function libFile(id) { return LIB.files.filter(function (f) { return f.id === id; })[0]; }
  function libTitle(f) { var e = libExam(f.exam_id); return (e ? e.name : 'Library') + ': ' + f.title; }
  async function loadLibrary() {
    var q = await Promise.all([
      sb.from('exams').select('*').order('sort').order('id'),
      sb.from('exam_folders').select('*').order('name'),
      sb.from('exam_files').select('*').order('title')
    ]);
    if (q[0].error || q[1].error || q[2].error) { LIB.ready = false; return; }
    LIB.ready = true;
    LIB.exams = q[0].data || []; LIB.folders = q[1].data || []; LIB.files = q[2].data || [];
    LIB.byTitle = {};
    LIB.files.forEach(function (f) { LIB.byTitle[libTitle(f)] = f; });
    paintHub();
  }
  function paintHub() {
    var grid = document.querySelector('#screen-home .hub-grid');
    if (!grid) return;
    Array.prototype.forEach.call(grid.querySelectorAll('.hub-lib'), function (n) { n.remove(); });
    LIB.exams.forEach(function (e) {
      var c = PAL[(e.color || 0) % PAL.length];
      var n = LIB.files.filter(function (f) { return f.exam_id === e.id; }).length;
      var b = document.createElement('button');
      b.type = 'button'; b.className = 'hub-btn hub-lib';
      b.style.cssText = '--g:linear-gradient(145deg,' + c[0] + ',' + c[1] + ');--gs:' + c[2];
      b.innerHTML = '<span class="hub-icon">' + escH(e.icon || '🎓') + '</span><span class="hub-title">' + escH(e.name) + '</span><span class="hub-sub">'
        + escH(e.description || (n ? n + ' study file' + (n > 1 ? 's' : '') : 'Material coming soon')) + '</span>';
      b.onclick = function () { openLibExam(e.id); };
      grid.appendChild(b);
    });
  }

  // student side: browse an exam folder
  function openLibExam(id) {
    currentCat = 'lib'; lastCatalogue = 'browse';
    browse = { cat: 'lib:' + id, folder: null, cls: null, courseId: null, level: 1 };
    renderLib();
    showScreen('browse');
    window.scrollTo(0, 0);
  }
  function libItem(f) {
    var t = libTitle(f);
    return { name: t, price: Number(f.price), entitle: [{ title: t, lib: f.id, path: f.path }] };
  }
  window.libOpenFolder = function (id) { browse.folder = id; renderLib(); window.scrollTo(0, 0); };
  window.libAdd = function (id) { var f = libFile(id); if (f) { pushCart(libItem(f)); renderLib(); } };
  window.libBuy = function (id) { var f = libFile(id); if (!f) return; checkoutFromCart = false; window.renderCheckout([libItem(f)]); };
  window.libOpen = function (id) { var f = libFile(id); if (f) openSecure(f.path, libTitle(f)); };
  function renderLib() {
    var el = document.getElementById('browseBody');
    var id = +String(browse.cat).slice(4), e = libExam(id);
    if (!el) return;
    if (!e) { showScreen('home'); return; }
    var folders = LIB.folders.filter(function (x) { return x.exam_id === id; });
    var files = LIB.files.filter(function (x) { return x.exam_id === id; });
    var fol = browse.folder ? folders.filter(function (x) { return x.id === browse.folder; })[0] : null;
    if (browse.folder && !fol) browse.folder = null;
    var html = '<div style="display:flex;align-items:center;gap:12px;margin-bottom:20px;flex-wrap:wrap"><button class="btn btn-secondary btn-sm" onclick="browseBack()">← Back</button><strong style="font-size:15px">' + escH(e.name) + (fol ? ' / ' + escH(fol.name) : '') + '</strong></div>';
    html += '<h2 style="font-size:24px;font-weight:700;margin-bottom:6px">' + escH(fol ? fol.name : e.name) + '</h2><p style="color:var(--text-muted);margin-bottom:20px">' + escH(e.description || ('Study material for ' + e.name + '.')) + '</p>';
    if (!fol && folders.length) {
      html += '<div class="class-grid">' + folders.map(function (x) {
        var n = files.filter(function (f) { return f.folder_id === x.id; }).length;
        return '<button class="class-tile" onclick="libOpenFolder(' + x.id + ')"><span class="ct-title">' + escH(x.name) + '</span><span class="ct-sub">' + n + (n === 1 ? ' file' : ' files') + '</span></button>';
      }).join('') + '</div>';
    }
    var list = fol ? files.filter(function (f) { return f.folder_id === fol.id; }) : files.filter(function (f) { return !f.folder_id; });
    if (list.length) {
      if (!fol && folders.length) html += '<div class="section-title" style="margin-top:24px">Other files</div>';
      html += '<div class="chapter-list"' + (!fol && folders.length ? '' : ' style="margin-top:34px"') + '>' + list.map(function (f) {
        var own = entOwned({ title: libTitle(f) });
        return '<div class="chapter-row"><div style="flex:1;font-weight:600">' + escH(f.title) + (f.pages ? '<div style="font-size:11px;font-weight:400;color:var(--text-muted)">' + f.pages + ' pages</div>' : '') + '</div>'
          + '<div style="font-weight:700;margin-right:6px">' + rupees(f.price) + '</div>'
          + (own ? '<span class="badge badge-green">Purchased</span><button class="btn btn-secondary btn-sm" onclick="libOpen(' + f.id + ')">Open</button>'
            : '<button class="btn btn-secondary btn-sm" onclick="libAdd(' + f.id + ')">Add to Cart</button><button class="btn btn-primary btn-sm" onclick="libBuy(' + f.id + ')">Buy Now</button>') + '</div>';
      }).join('') + '</div>';
    } else if (fol || !folders.length) {
      html += '<div class="chapter-list" style="margin-top:34px"><div class="chapter-row" style="color:var(--text-muted)">No files here yet. New material is added soon.</div></div>';
    }
    el.innerHTML = html;
  }
  var _renderBrowse = window.renderBrowse;
  window.renderBrowse = function () {
    if (browse && String(browse.cat).indexOf('lib:') === 0) return renderLib();
    return _renderBrowse.apply(this, arguments);
  };
  var _browseBack = window.browseBack;
  window.browseBack = function () {
    if (browse && String(browse.cat).indexOf('lib:') === 0) {
      if (browse.folder) { browse.folder = null; renderLib(); } else showScreen('home');
      return;
    }
    return _browseBack.apply(this, arguments);
  };
  // the page's own checkout starter expects elements that the live checkout replaces; this one only sets the order
  window.renderCheckout = function (items) { checkoutList = items; showScreen('checkout'); };

  // ---------- admin: Library tab ----------
  var AL = { view: 'home', examId: null, color: 0, picked: [] };
  function ensureLibraryTab() {
    var nav = document.querySelector('#screen-admin .sidebar-nav');
    var ref = document.getElementById('admin-payments');
    if (!nav || !ref) return;
    if (!document.getElementById('admin-library')) {
      var d = document.createElement('div');
      d.id = 'admin-library'; d.className = 'admin-tab-content'; d.style.display = 'none';
      ref.parentNode.insertBefore(d, ref.parentNode.firstChild);
      wireLibrary(d);
    }
    if (!document.getElementById('navLibrary')) {
      var a = document.createElement('a');
      a.id = 'navLibrary'; a.innerHTML = '📁&nbsp; Library';
      a.onclick = function () { AL.view = 'home'; switchAdminTab('library', a); };
      var ap = document.getElementById('navApprovals');
      nav.insertBefore(a, ap ? ap.nextSibling : nav.firstChild);
    }
  }
  function uid() { return (window.crypto && crypto.randomUUID) ? crypto.randomUUID().replace(/-/g, '').slice(0, 12) : Date.now().toString(36) + Math.random().toString(36).slice(2, 8); }
  async function pdfPages(file) {
    try {
      if (!window.pdfjsLib) return null;
      pdfjsLib.GlobalWorkerOptions.workerSrc = 'https://cdn.jsdelivr.net/npm/pdfjs-dist@3.11.174/build/pdf.worker.min.js';
      var d = await pdfjsLib.getDocument({ data: await file.arrayBuffer() }).promise;
      var n = d.numPages; if (d.destroy) d.destroy();
      return n;
    } catch (e) { return null; }
  }
  async function buyersOf(paths) {
    if (!paths.length) return 0;
    var r = await sb.from('payments').select('id', { count: 'exact', head: true }).eq('status', 'Paid').overlaps('file_paths', paths);
    return r.count || 0;
  }
  var IS = 'width:100%;padding:8px 10px;border:1px solid #cbd5e1;border-radius:8px;font-size:13px;box-sizing:border-box;background:#fff';
  function swatches() {
    return PAL.map(function (c, i) {
      return '<button type="button" data-act="color" data-i="' + i + '" title="Colour ' + (i + 1) + '" style="width:30px;height:30px;border-radius:50%;border:3px solid ' + (AL.color === i ? '#121a3a' : 'transparent') + ';background:linear-gradient(135deg,' + c[0] + ',' + c[1] + ');cursor:pointer"></button>';
    }).join(' ');
  }
  function homeView() {
    var h = '<div class="page-header"><h1>Library</h1><p>Each folder is one exam. Students see every folder as a card on their menu.</p></div>';
    if (!LIB.ready) return h + '<div class="card card-p" style="max-width:560px"><strong>Library is not set up yet.</strong><p style="font-size:13px;margin-top:6px;color:var(--text-muted)">Open the Supabase SQL Editor, run the file update_library.sql once, then click Library again.</p></div>';
    h += '<div style="margin-bottom:16px"><button class="btn btn-primary" data-act="new-toggle">+ New exam folder</button></div>'
      + '<div id="alNew" class="card card-p" style="display:none;max-width:620px;margin-bottom:20px">'
      + '<div style="font-weight:700;margin-bottom:10px">New exam folder</div>'
      + '<div style="display:grid;grid-template-columns:90px 1fr;gap:10px"><div><label style="font-size:12px;font-weight:600">Icon</label><input id="alIcon" maxlength="4" value="🎓" style="' + IS + ';font-size:20px;text-align:center"></div>'
      + '<div><label style="font-size:12px;font-weight:600">Exam name</label><input id="alName" placeholder="e.g. WBCS Prelims" maxlength="60" style="' + IS + '"></div></div>'
      + '<label style="font-size:12px;font-weight:600;display:block;margin-top:10px">Short description (shown on the card)</label><input id="alDesc" maxlength="120" placeholder="e.g. West Bengal Civil Service preliminary exam" style="' + IS + '">'
      + '<div style="margin-top:12px"><span style="font-size:12px;font-weight:600">Card colour</span><div id="alSw" style="margin-top:6px">' + swatches() + '</div></div>'
      + '<div style="margin-top:14px"><button class="btn btn-primary" data-act="new-save">Create folder</button></div></div>';
    var cards = '<div class="card card-p" style="display:flex;flex-direction:column;gap:6px"><div style="font-size:26px">📝</div><strong>TET I and TET II</strong><span class="badge badge-gray" style="width:fit-content">Built-in</span><div style="font-size:12px;color:var(--text-muted)">Subjects and course levels</div><div><button class="btn btn-secondary btn-sm" data-act="builtin">Manage files</button></div></div>'
      + '<div class="card card-p" style="display:flex;flex-direction:column;gap:6px"><div style="font-size:26px">🎓</div><strong>STPGT</strong><span class="badge badge-gray" style="width:fit-content">Built-in</span><div style="font-size:12px;color:var(--text-muted)">Subfolder: Physics (five course levels)</div><div><button class="btn btn-secondary btn-sm" data-act="builtin">Manage files</button></div></div>';
    cards += LIB.exams.map(function (e) {
      var c = PAL[(e.color || 0) % PAL.length], n = LIB.files.filter(function (f) { return f.exam_id === e.id; }).length;
      return '<div class="card card-p" style="display:flex;flex-direction:column;gap:6px;border-top:5px solid ' + c[0] + '"><div style="font-size:26px">' + escH(e.icon || '🎓') + '</div><strong>' + escH(e.name) + '</strong>'
        + '<div style="font-size:12px;color:var(--text-muted)">' + n + (n === 1 ? ' file' : ' files') + (e.description ? ' &middot; ' + escH(e.description) : '') + '</div>'
        + '<div style="display:flex;gap:8px;margin-top:4px"><button class="btn btn-primary btn-sm" data-act="open" data-id="' + e.id + '">Open</button><button class="btn btn-secondary btn-sm" data-act="del-exam" data-id="' + e.id + '">Delete</button></div></div>';
    }).join('');
    return h + '<div style="display:grid;grid-template-columns:repeat(auto-fill,minmax(240px,1fr));gap:16px">' + cards + '</div>'
      + (LIB.exams.length ? '' : '<p style="font-size:13px;color:var(--text-muted);margin-top:16px">No custom exam folders yet. Click New exam folder to add one.</p>');
  }
  function pickedHtml() {
    if (!AL.picked.length) return '';
    return '<div style="margin-top:12px">' + AL.picked.map(function (p, i) {
      return '<div style="display:grid;grid-template-columns:1fr 130px;gap:10px;margin-bottom:8px;align-items:end"><div><label style="font-size:11px;color:var(--text-muted)">' + escH(p.file.name) + ' (' + (p.file.size / 1048576).toFixed(1) + ' MB)</label><input data-pi="' + i + '" data-pf="title" value="' + escH(p.title) + '" placeholder="Title students see" style="' + IS + '"></div>'
        + '<div><label style="font-size:11px;color:var(--text-muted)">Price (₹)</label><input data-pi="' + i + '" data-pf="price" type="number" min="1" step="1" inputmode="decimal" value="' + escH(p.price) + '" placeholder="e.g. 500" style="' + IS + '"></div></div>';
    }).join('') + '<button class="btn btn-primary" data-act="upload">Upload ' + AL.picked.length + (AL.picked.length === 1 ? ' file' : ' files') + '</button> <span id="alStatus" style="font-size:12px;color:var(--text-muted)"></span></div>';
  }
  function examView() {
    var e = libExam(AL.examId);
    var folders = LIB.folders.filter(function (x) { return x.exam_id === e.id; });
    var files = LIB.files.filter(function (x) { return x.exam_id === e.id; });
    AL.color = e.color || 0;
    var fopts = function (sel) { return '<option value="">No folder</option>' + folders.map(function (x) { return '<option value="' + x.id + '"' + (x.id === sel ? ' selected' : '') + '>' + escH(x.name) + '</option>'; }).join(''); };
    var h = '<div style="margin-bottom:10px"><button class="btn btn-secondary btn-sm" data-act="home">← All exam folders</button></div>'
      + '<div class="page-header"><h1>' + escH(e.icon || '') + ' ' + escH(e.name) + '</h1><p>' + files.length + (files.length === 1 ? ' file' : ' files') + ' in this exam folder</p></div>';
    h += '<div class="card card-p" style="margin-bottom:18px"><div style="font-weight:700;margin-bottom:10px">Card details</div>'
      + '<div style="display:grid;grid-template-columns:90px 1fr 1.4fr;gap:10px"><input id="alIcon" maxlength="4" value="' + escH(e.icon || '') + '" style="' + IS + ';font-size:20px;text-align:center"><input id="alName" maxlength="60" value="' + escH(e.name) + '" style="' + IS + '"><input id="alDesc" maxlength="120" value="' + escH(e.description || '') + '" placeholder="Short description" style="' + IS + '"></div>'
      + '<div style="margin-top:10px" id="alSw">' + swatches() + '</div><div style="margin-top:12px"><button class="btn btn-secondary btn-sm" data-act="save-exam">Save card</button></div></div>';
    h += '<div class="card card-p" style="margin-bottom:18px"><div style="font-weight:700;margin-bottom:4px">Sub-folders (optional)</div><div style="font-size:12px;color:var(--text-muted);margin-bottom:10px">Use these to split an exam, for example Paper 1 and Paper 2.</div>'
      + '<div style="display:flex;gap:8px;flex-wrap:wrap;margin-bottom:10px">' + (folders.length ? folders.map(function (x) {
        var n = files.filter(function (f) { return f.folder_id === x.id; }).length;
        return '<span style="display:inline-flex;align-items:center;gap:8px;background:#f1f5f9;border:1px solid #e2e8f0;border-radius:999px;padding:5px 6px 5px 12px;font-size:13px;font-weight:600">📂 ' + escH(x.name) + ' <span style="font-weight:500;color:#64748b">' + n + '</span><button class="btn btn-secondary btn-sm" style="padding:2px 8px" data-act="del-folder" data-id="' + x.id + '" title="Delete folder (files stay in the exam)">×</button></span>';
      }).join('') : '<span style="font-size:13px;color:var(--text-muted)">None yet</span>') + '</div>'
      + '<div style="display:flex;gap:8px;max-width:420px"><input id="alFolder" maxlength="60" placeholder="New sub-folder name" style="' + IS + '"><button class="btn btn-secondary btn-sm" data-act="add-folder">Add</button></div></div>';
    h += '<div class="card card-p" style="margin-bottom:18px"><div style="font-weight:700;margin-bottom:10px">Upload PDFs from this device</div>'
      + '<div style="display:flex;gap:10px;flex-wrap:wrap;align-items:center"><select id="alUpFolder" style="' + IS + ';max-width:220px">' + fopts(null) + '</select>'
      + '<input id="alFiles" type="file" accept="application/pdf,.pdf" multiple style="' + IS + ';max-width:360px"></div>'
      + '<div style="font-size:11px;color:var(--text-muted);margin-top:6px">PDF files only, up to 50 MB each. Choose the files, then set a title and a price for each.</div>'
      + '<div id="alPicked">' + pickedHtml() + '</div></div>';
    h += '<div class="table-wrap"><table class="data-table"><thead><tr><th>Title</th><th>Folder</th><th>Price (₹)</th><th>Pages</th><th>Added</th><th></th></tr></thead><tbody>'
      + (files.length ? files.map(function (f) {
        return '<tr><td><input data-fid="' + f.id + '" data-ff="title" value="' + escH(f.title) + '" style="' + IS + '"></td>'
          + '<td><select data-fid="' + f.id + '" data-ff="folder" style="' + IS + '">' + fopts(f.folder_id) + '</select></td>'
          + '<td><input data-fid="' + f.id + '" data-ff="price" type="number" min="1" step="1" value="' + f.price + '" style="' + IS + ';width:100px"></td>'
          + '<td>' + (f.pages || '-') + '</td><td style="font-size:12px">' + fmtDate(f.created_at) + '</td>'
          + '<td style="white-space:nowrap"><button class="btn btn-secondary btn-sm" data-act="view-file" data-id="' + f.id + '">View</button> <button class="btn btn-secondary btn-sm" data-act="del-file" data-id="' + f.id + '">Delete</button></td></tr>';
      }).join('') : '<tr><td colspan="6" style="color:var(--text-muted)">No files yet. Upload your first PDF above.</td></tr>')
      + '</tbody></table></div>';
    return h;
  }
  async function renderAdminLib() {
    var box = document.getElementById('admin-library');
    if (!box) return;
    await loadLibrary();
    if (AL.view === 'exam' && libExam(AL.examId)) box.innerHTML = examView();
    else { AL.view = 'home'; box.innerHTML = homeView(); }
  }
  function val(id) { var n = document.getElementById(id); return n ? n.value.trim() : ''; }
  function dupMsg(r, what) { return r.error && r.error.code === '23505' ? 'A ' + what + ' with that name already exists.' : 'Could not save. Please try again.'; }
  function wireLibrary(box) {
    box.addEventListener('click', async function (ev) {
      var t = ev.target.closest('[data-act]');
      if (!t) return;
      var act = t.getAttribute('data-act'), id = +t.getAttribute('data-id');
      if (act === 'new-toggle') { var n = document.getElementById('alNew'); if (n) n.style.display = n.style.display === 'none' ? 'block' : 'none'; }
      else if (act === 'color') { AL.color = +t.getAttribute('data-i'); document.getElementById('alSw').innerHTML = swatches(); }
      else if (act === 'home') { AL.view = 'home'; renderAdminLib(); }
      else if (act === 'open') { AL.view = 'exam'; AL.examId = id; AL.picked = []; renderAdminLib(); }
      else if (act === 'builtin') { var a = Array.prototype.filter.call(document.querySelectorAll('#screen-admin .sidebar-nav a'), function (x) { return /materials/.test(x.getAttribute('onclick') || ''); })[0]; switchAdminTab('materials', a); }
      else if (act === 'new-save') {
        var name = val('alName');
        if (name.length < 2) return fail('Enter the exam name.');
        var r = await sb.from('exams').insert({ name: name, icon: val('alIcon') || '🎓', color: AL.color, description: val('alDesc') }).select().single();
        if (r.error) return fail(dupMsg(r, 'folder'));
        logEvent(adminId(), 'LIBRARY_NEW_EXAM · ' + name, 'admin_panel', 'OK');
        showToast('Exam folder created. Students now see its card.', 'success');
        AL.view = 'exam'; AL.examId = r.data.id; AL.picked = []; renderAdminLib();
      }
      else if (act === 'save-exam') {
        var nm = val('alName');
        if (nm.length < 2) return fail('Enter the exam name.');
        var u = await sb.from('exams').update({ name: nm, icon: val('alIcon') || '🎓', color: AL.color, description: val('alDesc') }).eq('id', AL.examId);
        if (u.error) return fail(dupMsg(u, 'folder'));
        showToast('Card saved.', 'success'); renderAdminLib();
      }
      else if (act === 'add-folder') {
        var fn = val('alFolder');
        if (fn.length < 1) return fail('Enter a sub-folder name.');
        var fr = await sb.from('exam_folders').insert({ exam_id: AL.examId, name: fn });
        if (fr.error) return fail(dupMsg(fr, 'sub-folder'));
        renderAdminLib();
      }
      else if (act === 'del-folder') {
        if (!confirm('Delete this sub-folder? Its files stay in the exam and move out of the folder.')) return;
        var dr = await sb.from('exam_folders').delete().eq('id', id);
        if (dr.error) return fail('Could not delete the sub-folder.');
        renderAdminLib();
      }
      else if (act === 'view-file') { var vf = libFile(id); if (vf) adminView(vf.path, libTitle(vf)); }
      else if (act === 'del-file') {
        var f = libFile(id); if (!f) return;
        var nb = await buyersOf([f.path + '.pdf']);
        if (!confirm('Delete "' + f.title + '"?' + (nb ? '\n\n' + nb + ' payment(s) already include this file. Those students will lose access.' : '') + '\n\nThis cannot be undone.')) return;
        await sb.storage.from('materials').remove([f.path + '.pdf']);
        var xr = await sb.from('exam_files').delete().eq('id', id);
        if (xr.error) return fail('Could not delete the file.');
        logEvent(adminId(), 'LIBRARY_DELETE_FILE · ' + f.title, 'admin_panel', 'OK');
        showToast('File deleted.', 'success'); renderAdminLib();
      }
      else if (act === 'del-exam') {
        var e = libExam(id); if (!e) return;
        var fl = LIB.files.filter(function (x) { return x.exam_id === id; });
        var nb2 = await buyersOf(fl.map(function (x) { return x.path + '.pdf'; }));
        if (!confirm('Delete the exam folder "' + e.name + '" and its ' + fl.length + ' file(s)?' + (nb2 ? '\n\n' + nb2 + ' payment(s) already include these files. Those students will lose access.' : '') + '\n\nThis cannot be undone.')) return;
        if (fl.length) await sb.storage.from('materials').remove(fl.map(function (x) { return x.path + '.pdf'; }));
        var er = await sb.from('exams').delete().eq('id', id);
        if (er.error) return fail('Could not delete the exam folder.');
        logEvent(adminId(), 'LIBRARY_DELETE_EXAM · ' + e.name, 'admin_panel', 'OK');
        showToast('Exam folder deleted.', 'success'); AL.view = 'home'; renderAdminLib();
      }
      else if (act === 'upload') {
        var rows = AL.picked, st = document.getElementById('alStatus');
        var folderSel = document.getElementById('alUpFolder'), folderId = folderSel && folderSel.value ? +folderSel.value : null;
        for (var i = 0; i < rows.length; i++) {
          var p = rows[i], price = parseFloat(p.price);
          if (!p.title.trim()) return fail('Enter a title for ' + p.file.name + '.');
          if (!(price > 0)) return fail('Enter a price above 0 for ' + p.file.name + '.');
          if (p.file.size > 50 * 1048576) return fail(p.file.name + ' is larger than 50 MB.');
        }
        t.disabled = true;
        var okN = 0;
        for (var j = 0; j < rows.length; j++) {
          var q = rows[j];
          if (st) st.textContent = 'Uploading ' + (j + 1) + ' of ' + rows.length + ': ' + q.file.name + ' ...';
          var path = 'library/' + AL.examId + '/' + uid();
          var up = await sb.storage.from('materials').upload(path + '.pdf', q.file, { contentType: 'application/pdf', upsert: false });
          if (up.error) { fail('Upload failed for ' + q.file.name + '. ' + (up.error.message || '')); continue; }
          var pages = await pdfPages(q.file);
          var ins = await sb.from('exam_files').insert({ exam_id: AL.examId, folder_id: folderId, title: q.title.trim(), path: path, price: parseFloat(q.price), pages: pages });
          if (ins.error) { await sb.storage.from('materials').remove([path + '.pdf']); fail(dupMsg(ins, 'file') + ' (' + q.title + ')'); continue; }
          logEvent(adminId(), 'LIBRARY_UPLOAD · ' + q.title.trim(), 'admin_panel', 'OK');
          okN++;
        }
        AL.picked = [];
        if (okN) showToast(okN + ' file(s) uploaded. Students can buy them now.', 'success');
        renderAdminLib();
      }
    });
    box.addEventListener('change', async function (ev) {
      var t = ev.target;
      if (t.id === 'alFiles') {
        AL.picked = Array.prototype.map.call(t.files, function (f) {
          return { file: f, title: f.name.replace(/\.pdf$/i, '').replace(/[_]+/g, ' ').trim(), price: '' };
        }).filter(function (p) { return /\.pdf$/i.test(p.file.name) || p.file.type === 'application/pdf'; });
        document.getElementById('alPicked').innerHTML = pickedHtml();
      } else if (t.getAttribute('data-fid')) {
        var f = libFile(+t.getAttribute('data-fid')), field = t.getAttribute('data-ff'), patch = {};
        if (!f) return;
        if (field === 'title') { if (!t.value.trim()) { t.value = f.title; return fail('Title cannot be empty.'); } patch.title = t.value.trim(); }
        else if (field === 'price') { var pv = parseFloat(t.value); if (!(pv > 0)) { t.value = f.price; return fail('Price must be above 0.'); } patch.price = pv; }
        else if (field === 'folder') patch.folder_id = t.value ? +t.value : null;
        var r = await sb.from('exam_files').update(patch).eq('id', f.id);
        if (r.error) { fail(dupMsg(r, 'file')); renderAdminLib(); return; }
        showToast('Saved.', 'success');
        await loadLibrary();
      }
    });
    box.addEventListener('input', function (ev) {
      var t = ev.target, i = t.getAttribute('data-pi');
      if (i != null && AL.picked[+i]) AL.picked[+i][t.getAttribute('data-pf')] = t.value;
    });
  }
  var _switchTab3 = switchAdminTab;
  switchAdminTab = function (tab, el) { ensureLibraryTab(); _switchTab3(tab, el); if (tab === 'library') renderAdminLib(); };
  var _refreshAdmin2 = refreshAdmin;
  refreshAdmin = function () { _refreshAdmin2(); ensureLibraryTab(); };
  var _doAdminLogin2 = doAdminLogin;
  doAdminLogin = async function () { await _doAdminLogin2(); if (adminUser) ensureLibraryTab(); };

  // ---------- start ----------
  sb.auth.signOut().then(function () {}, function () {});
  loadMaterials().then(function () { if (typeof refreshMenu === 'function') refreshMenu(); });
  console.info('EduSphere: live mode (Supabase).');
})();
