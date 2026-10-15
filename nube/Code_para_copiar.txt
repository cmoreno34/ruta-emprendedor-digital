/**
 * Ruta del Emprendedor Digital · guardado en la nube por equipo.
 * Google Apps Script (web app). Lo despliega el profesor una vez; los datos quedan en SU Google Drive.
 *
 * GET  ?a=load&team=ED-03&pin=1234  → { ok, exists, version, updated, data }
 * POST {a:"save", team, pin, base, data, info}  (Content-Type text/plain) → { ok, version, updated } | { ok:false, error:"conflict"|"pin"|... }
 *
 * - Un archivo JSON por equipo en la carpeta «Ruta Emprendedor Digital · datos».
 * - Historial: se guardan las últimas 40 versiones de cada equipo en la subcarpeta «historial».
 * - Panel del profesor: hoja «Ruta ED · panel del profesor» con una fila por equipo.
 * - El PIN se guarda cifrado (SHA-256 con sal). Para reiniciar el PIN de un equipo, borrad su celda «pin_hash» en el panel.
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
    if (current) archive_(team, file, current);
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

/** Ejecutad esta función una vez desde el editor para autorizar los permisos y crear carpeta y panel. */
function instalar() {
  folder_(); historyFolder_(); sheet_();
  Logger.log("Carpeta: " + folder_().getUrl());
  Logger.log("Panel: " + SpreadsheetApp.openById(PropertiesService.getScriptProperties().getProperty("sheetId")).getUrl());
}
