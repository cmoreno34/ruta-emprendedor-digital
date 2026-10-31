/**
 * Ruta del Emprendedor Digital · guardado en la nube por equipo.
 * Google Apps Script (web app). Lo despliega el profesor una vez; los datos quedan en SU Google Drive.
 *
 * GET  ?a=load&team=ED-03&pin=1234  → { ok, exists, version, updated, data }
 * POST {a:"save", team, pin, base, data, info}  (Content-Type text/plain) → { ok, version, updated } | { ok:false, error:"conflict"|"pin"|... }
 *
 * - Un archivo JSON por equipo en la carpeta «Ruta Emprendedor Digital · datos».
 * - Historial: se guardan las últimas 40 versiones de cada equipo (como mucho una cada 5 minutos) en la subcarpeta «historial».
 * - Panel del profesor: hoja «Ruta ED · panel del profesor» con una fila por equipo.
 * - El PIN se guarda cifrado (SHA-256 con sal). Para reiniciar el PIN de un equipo, borrad su celda «pin_hash» en el panel.
 *
 * Medidor de los prototipos (analítica casera, sin herramientas externas):
 * POST {a:"ev", team, ev:"visita"|"cta"|"registro"|"premium"|"otro", src, camp, v, email?}  → apunta una fila en la pestaña «eventos»
 * POST {a:"stats", team, pin}  → recuentos por evento, canal (utm_source) y versión A/B. Nunca devuelve emails.
 *
 * Encuesta cruzada en clase (estación 11):
 * POST {a:"enc_put", team, pin, def}  → publica la encuesta del equipo (pestaña «encuestas»)
 * POST {a:"enc_get", team}  → la encuesta, pública (no lleva datos privados)
 * POST {a:"enc_list"}  → encuestas publicadas [{team, titulo}] para la página de la clase
 * POST {a:"resp", team, answers}  → apunta una respuesta (pestaña «respuestas»)
 * POST {a:"resp_get", team, pin}  → respuestas del equipo
 */
const FOLDER_NAME = "Ruta Emprendedor Digital · datos";
const SHEET_NAME = "Ruta ED · panel del profesor";
const KEEP_HISTORY = 40;
const MAX_BYTES = 8 * 1024 * 1024;

function doGet(e) {
  const p = (e && e.parameter) || {};
  if (p.a === "ping") return out({ ok: true, service: "ruta-ed", time: new Date().toISOString() });
  if (p.a === "load") return out(load_(p.team, p.pin));
  return out({ ok: false, error: "bad_request" });
}

function doPost(e) {
  let body = {};
  try { body = JSON.parse(e.postData.contents); } catch (err) { return out({ ok: false, error: "bad_json" }); }
  if (body.a === "save") return out(save_(body));
  if (body.a === "load") return out(load_(body.team, body.pin));
  if (body.a === "ev") return out(ev_(body));
  if (body.a === "stats") return out(stats_(body.team, body.pin));
  if (body.a === "enc_put") return out(encPut_(body));
  if (body.a === "enc_get") return out(encGet_(body.team));
  if (body.a === "enc_list") return out(encList_());
  if (body.a === "resp") return out(resp_(body));
  if (body.a === "resp_get") return out(respGet_(body.team, body.pin));
  return out({ ok: false, error: "bad_request" });
}

function out(obj) {
  return ContentService.createTextOutput(JSON.stringify(obj)).setMimeType(ContentService.MimeType.JSON);
}

/* ---------------- lógica ---------------- */
function cleanTeam_(t) {
  t = String(t || "").trim().toUpperCase();
  return /^[A-Z0-9][A-Z0-9-]{1,23}$/.test(t) ? t : "";
}
function cleanPin_(p) {
  p = String(p || "").trim();
  return p.length >= 4 && p.length <= 12 ? p : "";
}
function hash_(team, pin) {
  const props = PropertiesService.getScriptProperties();
  let salt = props.getProperty("salt");
  if (!salt) { salt = Utilities.getUuid(); props.setProperty("salt", salt); }
  const raw = Utilities.computeDigest(Utilities.DigestAlgorithm.SHA_256, salt + "|" + team + "|" + pin, Utilities.Charset.UTF_8);
  return raw.map(b => ("0" + (b & 255).toString(16)).slice(-2)).join("");
}

function load_(team, pin) {
  team = cleanTeam_(team); pin = cleanPin_(pin);
  if (!team) return { ok: false, error: "team" };
  if (!pin) return { ok: false, error: "pin_format" };
  const row = findRow_(team);
  if (!row) return { ok: true, exists: false, version: 0 };
  if (row.pinHash && row.pinHash !== hash_(team, pin)) return { ok: false, error: "pin" };
  const file = fileFor_(team, false);
  if (!file) return { ok: true, exists: false, version: 0 };
  let data = null;
  try { data = JSON.parse(file.getBlob().getDataAsString("UTF-8")); } catch (err) { return { ok: false, error: "corrupt" }; }
  return { ok: true, exists: true, version: row.version, updated: row.updated, data: data };
}

function save_(b) {
  const team = cleanTeam_(b.team), pin = cleanPin_(b.pin);
  if (!team) return { ok: false, error: "team" };
  if (!pin) return { ok: false, error: "pin_format" };
  if (!b.data || typeof b.data !== "object") return { ok: false, error: "no_data" };
  const json = JSON.stringify(b.data);
  if (json.length > MAX_BYTES) return { ok: false, error: "too_large" };
  const lock = LockService.getScriptLock();
  if (!lock.tryLock(15000)) return { ok: false, error: "busy" };
  try {
    const h = hash_(team, pin);
    let row = findRow_(team);
    if (row && row.pinHash && row.pinHash !== h) return { ok: false, error: "pin" };
    const current = row ? row.version : 0;
    const base = Number(b.base || 0);
    if (current && base !== current) return { ok: false, error: "conflict", version: current, updated: row.updated };
    const file = fileFor_(team, true);
    // Historial como mucho cada 5 minutos por equipo: copiar el archivo en cada guardado bloqueaba a toda la clase.
    const cache = CacheService.getScriptCache();
    if (current && !cache.get("arch_" + team)) { archive_(team, file, current); cache.put("arch_" + team, "1", 300); }
    file.setContent(json);
    const now = new Date();
    const info = b.info || {};
    writeRow_(team, {
      version: current + 1, updated: now, pinHash: h,
      proyecto: String(info.proyecto || "").slice(0, 120), miembros: String(info.miembros || "").slice(0, 200),
      progreso: Number(info.progreso || 0), estacion: String(info.estacion || "").slice(0, 60)
    }, row);
    return { ok: true, version: current + 1, updated: now.toISOString() };
  } finally {
    lock.releaseLock();
  }
}

/* ---------------- almacenamiento ---------------- */
function folder_() {
  const props = PropertiesService.getScriptProperties();
  const id = props.getProperty("folderId");
  if (id) { try { return DriveApp.getFolderById(id); } catch (err) {} }
  const f = DriveApp.createFolder(FOLDER_NAME);
  props.setProperty("folderId", f.getId());
  return f;
}
function historyFolder_() {
  const props = PropertiesService.getScriptProperties();
  const id = props.getProperty("historyId");
  if (id) { try { return DriveApp.getFolderById(id); } catch (err) {} }
  const f = folder_().createFolder("historial");
  props.setProperty("historyId", f.getId());
  return f;
}
function fileFor_(team, create) {
  const it = folder_().getFilesByName(team + ".json");
  if (it.hasNext()) return it.next();
  return create ? folder_().createFile(team + ".json", "{}", MimeType.PLAIN_TEXT) : null;
}
function archive_(team, file, version) {
  const hf = historyFolder_();
  const stamp = Utilities.formatDate(new Date(), Session.getScriptTimeZone(), "yyyyMMdd-HHmmss");
  hf.createFile(team + "_v" + version + "_" + stamp + ".json", file.getBlob().getDataAsString("UTF-8"), MimeType.PLAIN_TEXT);
  const old = [];
  const it = hf.searchFiles('title contains "' + team + '_v"');
  while (it.hasNext()) old.push(it.next());
  old.sort((a, b) => b.getDateCreated() - a.getDateCreated());
  old.slice(KEEP_HISTORY).forEach(f => f.setTrashed(true));
}

/* ---------------- panel del profesor ---------------- */
const COLS = ["equipo", "proyecto", "miembros", "version", "actualizado", "progreso_%", "ultima_estacion", "pin_hash"];
function sheet_() {
  const props = PropertiesService.getScriptProperties();
  const id = props.getProperty("sheetId");
  let ss = null;
  if (id) { try { ss = SpreadsheetApp.openById(id); } catch (err) {} }
  if (!ss) {
    ss = SpreadsheetApp.create(SHEET_NAME);
    DriveApp.getFileById(ss.getId()).moveTo(folder_());
    props.setProperty("sheetId", ss.getId());
    const sh = ss.getSheets()[0];
    sh.setName("equipos");
    sh.getRange(1, 1, 1, COLS.length).setValues([COLS]).setFontWeight("bold");
    sh.setFrozenRows(1);
  }
  return ss.getSheetByName("equipos");
}
function findRow_(team) {
  const sh = sheet_();
  const n = sh.getLastRow();
  if (n < 2) return null;
  const vals = sh.getRange(2, 1, n - 1, COLS.length).getValues();
  for (let i = 0; i < vals.length; i++) {
    if (vals[i][0] === team) return { index: i + 2, version: Number(vals[i][3]) || 0, updated: vals[i][4] ? new Date(vals[i][4]).toISOString() : null, pinHash: String(vals[i][7] || "") };
  }
  return null;
}
function writeRow_(team, r, existing) {
  const sh = sheet_();
  const values = [[team, r.proyecto, r.miembros, r.version, r.updated, r.progreso, r.estacion, r.pinHash]];
  if (existing) sh.getRange(existing.index, 1, 1, COLS.length).setValues(values);
  else sh.appendRow(values[0]);
}

/* ---------------- medidor de los prototipos ---------------- */
const EV_COLS = ["fecha", "equipo", "evento", "version_ab", "utm_source", "utm_campaign", "email"];
const EVENTS = ["visita", "cta", "registro", "premium", "otro"];
function evSheet_() {
  const ss = sheet_().getParent();
  let sh = ss.getSheetByName("eventos");
  if (!sh) {
    sh = ss.insertSheet("eventos");
    sh.getRange(1, 1, 1, EV_COLS.length).setValues([EV_COLS]).setFontWeight("bold");
    sh.setFrozenRows(1);
  }
  return sh;
}
const clip_ = (x, n) => String(x == null ? "" : x).replace(/[\r\n\t]/g, " ").trim().slice(0, n);
function ev_(b) {
  const team = cleanTeam_(b.team);
  if (!team || !findRow_(team)) return { ok: false, error: "team" };
  const ev = EVENTS.indexOf(String(b.ev)) >= 0 ? String(b.ev) : "otro";
  let email = clip_(b.email, 120);
  if (email && !/^[^@\s]+@[^@\s]+\.[^@\s]+$/.test(email)) email = "";
  evSheet_().appendRow([new Date(), team, ev, clip_(b.v, 10).toLowerCase() || "a", clip_(b.src, 40).toLowerCase() || "directo", clip_(b.camp, 60), ev === "registro" ? email : ""]);
  return { ok: true };
}
function stats_(team, pin) {
  team = cleanTeam_(team); pin = cleanPin_(pin);
  if (!team) return { ok: false, error: "team" };
  const row = findRow_(team);
  if (!row) return { ok: false, error: "team" };
  if (row.pinHash && row.pinHash !== hash_(team, pin)) return { ok: false, error: "pin" };
  const sh = evSheet_(), n = sh.getLastRow();
  const res = { ok: true, total: {}, porCanal: {}, porVersion: {}, primero: null, ultimo: null };
  if (n < 2) return res;
  sh.getRange(2, 1, n - 1, 6).getValues().forEach(r => {
    if (r[1] !== team) return;
    const ev = r[2], v = r[3] || "a", src = r[4] || "directo";
    res.total[ev] = (res.total[ev] || 0) + 1;
    (res.porCanal[src] = res.porCanal[src] || {})[ev] = (res.porCanal[src][ev] || 0) + 1;
    (res.porVersion[v] = res.porVersion[v] || {})[ev] = (res.porVersion[v][ev] || 0) + 1;
    const t = new Date(r[0]).toISOString();
    if (!res.primero || t < res.primero) res.primero = t;
    if (!res.ultimo || t > res.ultimo) res.ultimo = t;
  });
  return res;
}

/* ---------------- encuesta cruzada en clase ---------------- */
const MAX_DEF = 30000, MAX_RESP = 6000;
function tab_(name, cols) {
  const ss = sheet_().getParent();
  let sh = ss.getSheetByName(name);
  if (!sh) {
    sh = ss.insertSheet(name);
    sh.getRange(1, 1, 1, cols.length).setValues([cols]).setFontWeight("bold");
    sh.setFrozenRows(1);
  }
  return sh;
}
const encSheet_ = () => tab_("encuestas", ["equipo", "titulo", "actualizado", "definicion_json"]);
const respSheet_ = () => tab_("respuestas", ["fecha", "equipo", "respuesta_json"]);
function teamOk_(team, pin) {
  team = cleanTeam_(team); pin = cleanPin_(pin);
  if (!team) return { error: "team" };
  if (!pin) return { error: "pin_format" };
  const row = findRow_(team);
  if (!row) return { error: "team" };
  if (row.pinHash && row.pinHash !== hash_(team, pin)) return { error: "pin" };
  return { team: team };
}
function encRow_(team) {
  const sh = encSheet_(), n = sh.getLastRow();
  if (n < 2) return null;
  const vals = sh.getRange(2, 1, n - 1, 4).getValues();
  for (let i = 0; i < vals.length; i++) if (vals[i][0] === team) return { index: i + 2, titulo: vals[i][1], def: vals[i][3] };
  return null;
}
function encPut_(b) {
  const t = teamOk_(b.team, b.pin);
  if (t.error) return { ok: false, error: t.error };
  if (!b.def || typeof b.def !== "object" || !Array.isArray(b.def.preguntas)) return { ok: false, error: "no_def" };
  const json = JSON.stringify(b.def);
  if (json.length > MAX_DEF) return { ok: false, error: "too_large" };
  const lock = LockService.getScriptLock();
  if (!lock.tryLock(15000)) return { ok: false, error: "busy" };
  try {
    const row = encRow_(t.team), vals = [[t.team, clip_(b.def.titulo, 120), new Date(), json]];
    if (row) encSheet_().getRange(row.index, 1, 1, 4).setValues(vals); else encSheet_().appendRow(vals[0]);
    return { ok: true };
  } finally { lock.releaseLock(); }
}
function encGet_(team) {
  team = cleanTeam_(team);
  if (!team) return { ok: false, error: "team" };
  const row = encRow_(team);
  if (!row) return { ok: false, error: "no_survey" };
  try { return { ok: true, def: JSON.parse(row.def) }; } catch (err) { return { ok: false, error: "corrupt" }; }
}
function encList_() {
  const sh = encSheet_(), n = sh.getLastRow();
  if (n < 2) return { ok: true, list: [] };
  return { ok: true, list: sh.getRange(2, 1, n - 1, 2).getValues().filter(r => r[0]).map(r => ({ team: r[0], titulo: r[1] })) };
}
function resp_(b) {
  const team = cleanTeam_(b.team);
  if (!team || !encRow_(team)) return { ok: false, error: "no_survey" };
  if (!b.answers || typeof b.answers !== "object") return { ok: false, error: "no_data" };
  const json = JSON.stringify(b.answers);
  if (json.length > MAX_RESP) return { ok: false, error: "too_large" };
  respSheet_().appendRow([new Date(), team, json]);
  return { ok: true };
}
function respGet_(team, pin) {
  const t = teamOk_(team, pin);
  if (t.error) return { ok: false, error: t.error };
  const sh = respSheet_(), n = sh.getLastRow(), rows = [];
  if (n >= 2) sh.getRange(2, 1, n - 1, 3).getValues().forEach(r => {
    if (r[1] !== t.team) return;
    let a = {};
    try { a = JSON.parse(r[2]); } catch (err) { return; }
    a._fecha = new Date(r[0]).toISOString();
    rows.push(a);
  });
  return { ok: true, rows: rows };
}

/** Ejecutad esta función una vez desde el editor para autorizar los permisos y crear carpeta y panel. */
function instalar() {
  folder_(); historyFolder_(); sheet_(); evSheet_(); encSheet_(); respSheet_();
  Logger.log("Carpeta: " + folder_().getUrl());
  Logger.log("Panel: " + SpreadsheetApp.openById(PropertiesService.getScriptProperties().getProperty("sheetId")).getUrl());
}
