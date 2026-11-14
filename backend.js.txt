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
      approvedOn: r.approved_on ? fmtDate(r.approved_on) : null, receiptSeen: !r.receipt_no || seenSet().indexOf(r.receipt_no) >= 0 };
  }
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

  processPayment = async function () {
    var utr = document.getElementById('utrInput').value.trim();
    if (!/^\d{12}$/.test(utr)) return fail('Enter the 12-digit UTR from your UPI app.');
    if (!currentUser) return fail('Please log in first.');
    var items = checkoutList;
    var entitle = items.reduce(function (a, i) { return a.concat(i.entitle); }, []);
    var seen = {}, paths = [];
    entitle.forEach(function (e) { pathsFor(e).forEach(function (p) { if (!seen[p]) { seen[p] = 1; paths.push(p + '.pdf'); } }); });
    showToast('Submitting UTR ' + utr + '...');
    var r = await sb.from('payments').insert({
      user_id: currentUser.uid, student_id: currentUser.id, utr: utr,
      label: items.map(function (i) { return i.name; }).join('; '), items: entitle, file_paths: paths,
      total: items.reduce(function (s, i) { return s + i.price; }, 0)
    });
    if (r.error) return fail(r.error.code === '23505' ? 'This UTR has already been submitted.' : 'Could not submit the payment. Please try again.');
    await loadMyPayments();
    if (checkoutFromCart) { clearCart(); checkoutFromCart = false; }
    showToast('Submitted. Status: awaiting verification. Access activates after admin approval.', 'success');
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
  var pdfDoc = null;
  async function renderPdfPage(n) {
    if (!pdfDoc) return;
    var page = await pdfDoc.getPage(n);
    var host = document.getElementById('pdfCanvas');
    if (!host) return;
    var width = Math.min(560, document.getElementById('pdfContent').clientWidth || 540);
    var base = page.getViewport({ scale: 1 });
    var vp = page.getViewport({ scale: width / base.width });
    var ratio = window.devicePixelRatio || 1;
    host.width = vp.width * ratio; host.height = vp.height * ratio;
    host.style.width = vp.width + 'px'; host.style.height = vp.height + 'px';
    var ctx = host.getContext('2d');
    ctx.setTransform(ratio, 0, 0, ratio, 0, 0);
    await page.render({ canvasContext: ctx, viewport: vp }).promise;
    ctx.save();
    ctx.globalAlpha = 0.09; ctx.fillStyle = '#2563eb'; ctx.font = 'bold 20px sans-serif';
    ctx.translate(vp.width / 2, vp.height / 2); ctx.rotate(-Math.PI / 6);
    var mark = currentUser ? currentUser.id + ' ' + currentUser.name : '';
    ctx.textAlign = 'center';
    for (var y = -vp.height; y < vp.height; y += 110) ctx.fillText(mark, 0, y);
    ctx.restore();
  }
  async function openSecure(path, title) {
    if (!window.pdfjsLib) return fail('The PDF viewer could not load. Please check your internet connection.');
    var u = await sb.storage.from('materials').createSignedUrl(path + '.pdf', 900);
    if (u.error || !u.data) return fail('This file is not available yet, or you do not have access to it.');
    pdfjsLib.GlobalWorkerOptions.workerSrc = 'https://cdn.jsdelivr.net/npm/pdfjs-dist@3.11.174/build/pdf.worker.min.js';
    try {
      showViewer(title, '<div style="text-align:center"><canvas id="pdfCanvas"></canvas></div>');
      pdfDoc = await pdfjsLib.getDocument({ url: u.data.signedUrl }).promise;
      totalPages = pdfDoc.numPages;
      currentPage = 1;
      updatePageNum();
      await renderPdfPage(1);
    } catch (e) {
      pdfDoc = null;
      fail('Could not open this file.');
    }
  }
  changePage = function (dir) {
    currentPage = Math.max(1, Math.min(totalPages, currentPage + dir));
    updatePageNum();
    renderPdfPage(currentPage);
  };
  openStpgt = function (sid, lvl) { openSecure('stpgt/' + sid + '/' + lvl, 'STPGT ' + STPGT.subjects[sid].name + ' (' + STPGT.subjects[sid].levels[lvl].name + ' course)'); };
  openChapter = function (id, lvl, ch) { openSecure('cbse/' + id + '/' + lvl + '/ch' + ch, classTitle(COURSES[id]) + ' (' + LEVELS[lvl] + '), Chapter ' + ch); };
  openTet = function (paper, slab, subj) { openSecure('tet/' + paper + '/' + slug(subj) + '/' + slab, 'Tripura TET ' + paper + ', ' + subj + ' (' + SLABS[slab] + ' Course)'); };
  openTetCombo = function (p, j, sl) { openSecure('tet/' + p + '/combo' + j + '/' + sl, 'Tripura TET ' + p + ', ' + TET[p].combos[j].name + ' (' + SLABS[sl] + ' Course)'); };
  openViewer = function () { showToast('This material is not available yet.', 'warning'); };

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
    if (r.error) return fail('Could not update the payment.');
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

  // ---------- start ----------
  sb.auth.signOut().then(function () {}, function () {});
  loadMaterials().then(function () { if (typeof refreshMenu === 'function') refreshMenu(); });
  console.info('EduSphere: live mode (Supabase).');
})();
