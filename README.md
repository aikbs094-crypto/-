/**
 * แผนตัดเดือนธันวาคม ปี 69/70 - Dashboard Web App (ฉบับรวมโค้ด + เมนูแผนที่)
 *
 * ★ หลักการทำงานหลักของระบบ:
 *   - ใช้ "รหัสแปลง" (PLOT_INTECH) เป็น Unique Key อ้างอิงข้อมูล (ไม่เชื่อเลขแถวที่ client จำไว้)
 *     ตอนบันทึกเซิร์ฟเวอร์ตรวจรหัสแปลงที่ชีตจริงก่อนเขียนทุกครั้ง + LockService กันบันทึกชนกัน
 *   - ผูกกับไฟล์ "ระบบการจัดกลุ่มตัดรายแปลง" ผ่าน MAIN_SS_ID (ชีต Data / Code)
 *     ถ้าชีต Data ใช้ IMPORTRANGE (A1) -> อ่าน A:O จากไฟล์ต้นทาง (SRC_SS_ID)
 *     ส่วน "ลำดับการตัด" (คอลัมน์ U) เขียนที่ชีต Data ของไฟล์นี้
 *   - ข้อมูลกลุ่มตัด: ใช้เฉพาะแถวชีต Code ที่ AF "แสดงข้อมูล" = 1 (แดชบอร์ดและหน้าจดแจ้งฯ ใช้ทุกแถวของชีต Data)
 *   - ความเร็ว: ข้อความ JSON + แคช gzip/base64 + พจนานุกรมข้อความซ้ำ + warmCache ทุก 10 นาที
 *   - เมนู: แดชบอร์ด / ข้อมูลกลุ่มตัด / จดแจ้งลำดับแผนตัดเดือนธันวาคม / แผนที่
 *
 * ★ [แก้ไข ข้อ 79]
 *   - หน้าจดแจ้งฯ: เมื่อกรอง "ชื่อนักเกษตร" ช่องเป้าตัดธันวาคม (ไร่/ตัน) รวมเฉพาะแปลงที่อายุอ้อย "ถึงเกณฑ์"
 *   - หน้าแผนที่ ตารางแปลงอ้อยทั้งหมด: แก้ไข "ลำดับการตัด" ได้ในช่อง (บันทึกทันทีเมื่อกด Enter / คลิกออกจากช่อง)
 *   - ตารางข้อมูลแปลง (หน้าจดแจ้งฯ): ทุกคอลัมน์มีตัวกรองที่หัวคอลัมน์ (ปุ่ม ▾) แบบ Excel
 *   - เอาเมนู "ตรวจแปลงรถตัดโรงงาน" ออก (ยังโหลดชีต GTT ไว้ไฮไลต์แถวแดง "รถตัดโรงงาน + ไม่รับตัด")
 *
 * ★ [แก้ไข ข้อ 78] ปรับความเร็วโหลด/บันทึก + ตัดโค้ดที่ไม่จำเป็น
 *   - ★ แก้บั๊กสำคัญ: โค้ดสร้างไฟล์ KML มีอักขระเกิน ("Style>';") ทำให้สคริปต์ฝั่งหน้าเว็บ Syntax Error
 *     ทั้งก้อน หน้าเว็บจึงค้างที่ "กำลังโหลดข้อมูล..." และตัวเลขเป็น "-" ทุกช่อง
 *   - บันทึก: เลิกอ่านชีตทั้งก้อนใหม่ภายใน Lock (_clearCacheAfterSave เดิม) เปลี่ยนเป็นแก้ค่าในแคชตรงๆ
 *     (_patchCacheAfterSave) ไม่แตะชีตเลย -> เร็วขึ้นมาก และผู้ใช้คนถัดไปเห็นลำดับใหม่ทันที
 *   - เปิดหน้า: doGet ฝังข้อมูลจากแคชลงในหน้าเว็บเลย (ไม่ต้องรอเรียกเซิร์ฟเวอร์อีกรอบ) ถ้าแคชไม่มีหรือใหญ่เกินจึงโหลดแบบเดิม
 *   - หน้าเว็บเริ่มโหลดข้อมูลทันที ไม่รอ window.onload (เดิมรอ Chart.js / Leaflet / JSZip โหลดครบก่อน)
 *     Leaflet และ JSZip โหลดตอนใช้งานแผนที่ / ดาวน์โหลด KMZ (และโหลดล่วงหน้าตอนว่าง)
 *   - ตัดโค้ดซ้ำ: _readCodeMap (ใช้ _buildCodeData ตัวเดียว) / dropdowns จากเซิร์ฟเวอร์ (หน้าเว็บคำนวณเอง) /
 *     duplicates ใน _buildPlotCodeRowMap / CSS บล็อกสรุปที่ประกาศซ้ำ / CSS แผนที่เต็มหน้าที่ไม่ใช้แล้ว
 *   - warmCache อ่านชีต Code ก่อนแล้วส่งต่อให้ข้อมูลแปลง (ไม่ใช้แคชชีต Code ที่เก่ากว่า)
 *   - ถ้า SRC_SS_ID เท่ากับ MAIN_SS_ID ใช้ไฟล์ที่เปิดไว้แล้ว (ไม่ openById ซ้ำ)
 *
 * ★ [ข้อ 71-77] ชีต Data คอลัมน์ V = "แปลงผิดปกติ" / หน้าจดแจ้งฯ แก้ไขได้เฉพาะ "ลำดับการตัด"
 *   แผนที่: พิกัดจากชีต Data P:R (getMapCoords) แคชแยก CK_MAP / ป๊อปอัป 3 หน้าจอ (ข้อมูล -> กรอก -> ตรวจสอบ)
 *   เส้นเชื่อมตามลำดับการตัดเชื่อมเฉพาะแปลงในกลุ่มตัดเดียวกัน (ทั้งบนแผนที่และไฟล์ KMZ)
 *   คีย์แคช CK_MAIN เปลี่ยนเลขเมื่อโครงสร้างข้อมูลเปลี่ยน (กันแคชเก่า)
 */
const SHEET_NAME = "Data";

// ─── ★ ไฟล์หลัก "ระบบการจัดกลุ่มตัดรายแปลง" (ชีต Data / Code) ─────
const MAIN_SS_ID  = '1ZsujkvnfF2fbqvhvHiVzCfTk4J_qgFS96d4SOVSxl2o';

let _mainSS = null; // เปิดไฟล์ครั้งเดียวต่อ 1 การเรียกฟังก์ชันจากฝั่ง client
function _getMainSS() {
  if (!_mainSS) _mainSS = SpreadsheetApp.openById(MAIN_SS_ID);
  return _mainSS;
}

// ─── ไฟล์ต้นทางของ IMPORTRANGE ในชีต Data (ช่อง A1) ─────
// (ตรวจ ID ให้ตรงกับสูตรในช่อง A1 ของชีต Data อีกครั้ง)
const SRC_SS_ID      = '1ZsujkvnfF2fbqvhvHiVzCfTk4J_qgFS96d4SOVSxl2o';
const SRC_SHEET_NAME = 'Data';
const SRC_COLS       = 18; // ตำแหน่งคอลัมน์ A:R (คอลัมน์ S เริ่มที่ดัชนี 18)
const SRC_READ_COLS  = 15; // อ่านเฉพาะ A:O (ข้าม lat/long P:R ที่ไม่ได้ใช้)

let _srcSheet = null;
function _getSrcSheet() {
  if (!_srcSheet) {
    const ss = SRC_SS_ID === MAIN_SS_ID ? _getMainSS() : SpreadsheetApp.openById(SRC_SS_ID);
    const s = ss.getSheetByName(SRC_SHEET_NAME);
    if (!s) throw new Error('ไม่พบชีต ' + SRC_SHEET_NAME + ' ในไฟล์ต้นทาง');
    _srcSheet = s;
  }
  return _srcSheet;
}

// ชีต Data ของไฟล์นี้ใช้ IMPORTRANGE อยู่หรือไม่ (ตรวจครั้งเดียวต่อการเรียก)
let _importModeCache = null;
function _isImportMode(sheet) {
  if (_importModeCache !== null) return _importModeCache;
  try { _importModeCache = /IMPORTRANGE/i.test(sheet.getRange('A1').getFormula()); }
  catch (e) { _importModeCache = false; }
  return _importModeCache;
}

const COL = {
  FARMER_ID:      0,
  FARMER_NAME:    1,
  ZONE:           2,
  CANE_TYPE:      3,
  AREA:           4,
  PLOT_INTECH:    5,   // ★ ใช้เป็น Unique Key หลักของทั้งระบบ (รหัสแปลง)
  PLOT_CODE:      6,
  OWNER_NAME:     7,
  PLANT_DATE:     8,
  FACTORY:        9,
  CUT_TYPE:      10,
  CUT_GROUP:     11,
  CUT_GROUP_NAME:12,
  EST_TON:       13,
  TON_PER_RAI:   14,
  CANE_AGE:      18,   // คอลัมน์ S "อายุอ้อย"
  CUT_CRITERIA:  19,   // คอลัมน์ T "เกณฑ์การตัด"
  CUT_ORDER:     20,   // คอลัมน์ U "ลำดับการตัด"
  ABNORMAL:      21,   // คอลัมน์ V "แปลงผิดปกติ"
};

// ตำแหน่งฟิลด์ในแถว allRows ที่ส่งให้หน้าเว็บ (ต้องตรงกับ F ฝั่งหน้าเว็บ)
const R_PLOT  = 5;   // รหัสแปลง
const R_ORDER = 15;  // ลำดับการตัด

// ─── ตรวจแปลงรถตัดโรงงาน: ไฟล์ "สำรวจแปลงอ้อย ปี 6970" ชีต GTT ──────────
// (ไม่มีเมนูแสดงผลแล้ว แต่ยังใช้ผลตรวจไฮไลต์แถวแดงในตารางจดแจ้งฯ)
const INSPECT_SS_ID      = '11ag5cehSeXy1jggY8raYK-atySMKNBLmvHVChaO58EI';
const INSPECT_SHEET_NAME = 'GTT';

// ★ [แก้ไข ข้อ 78] ฝังข้อมูลจากแคชลงในหน้าเว็บ (ถ้ามี) ให้แสดงผลทันทีโดยไม่ต้องรอเรียกเซิร์ฟเวอร์อีกรอบ
const PRELOAD_MAX = 3000000; // ใหญ่กว่านี้ (ตัวอักษร) ไม่ฝัง -> หน้าเว็บโหลดแบบเดิม

function doGet() {
  return HtmlService.createHtmlOutput(getHtmlPage(_buildPreload()))
    .setTitle("แผนตัดเดือนธันวาคม ปี 69/70")
    .setXFrameOptionsMode(HtmlService.XFrameOptionsMode.ALLOWALL);
}

function _buildPreload() {
  try {
    const main = _getCache(CK_MAIN);
    if (!main) return null;
    const code = _getCache(CK_CODE);
    if (main.length + (code ? code.length : 0) > PRELOAD_MAX) return null;
    return { main: main, code: code };
  } catch (e) { return null; }
}

// แปลงข้อความเป็น JS string literal ที่ฝังใน <script> ได้อย่างปลอดภัย
function _jsLit(s) {
  if (s == null) return 'null';
  return JSON.stringify(String(s))
    .replace(/</g, '\\u003c')
    .replace(/\u2028/g, '\\u2028')
    .replace(/\u2029/g, '\\u2029');
}

// ─── Cache helpers ─────────────────────────────────────────────────────────
const CACHE_TTL  = 1200;    // 20 นาที (warmCache อุ่นแคชทุก 10 นาที / บันทึก-รีเฟรช อัปเดตแคชให้อยู่แล้ว)
const CHUNK_SIZE = 90000;   // ข้อมูลในแคชเป็น base64 (1 ตัวอักษร = 1 byte) จำกัด 100KB/คีย์

const CK_MAIN    = 'z6_main';
const CK_CODE    = 'z4_code';
const CK_INSPECT = 'z2_insp';
const CK_SAVED   = 'z2_saved';  // เวลาที่บันทึกล่าสุด (กันแคชเก่าที่อ่านก่อนบันทึกมาทับ)
const CK_MAP     = 'z1_map';    // พิกัดแปลง (ไม่เปลี่ยนตอนบันทึกลำดับ จึงไม่ล้างหลังบันทึก)

// ตำแหน่งคอลัมน์ lat/long (0-based: P = 15, Q = 16) ใช้เมื่อหาจากหัวตารางไม่เจอ
const MAP_LAT_IDX = 15;
const MAP_LNG_IDX = 16;

// ตัวหาคอลัมน์จากหัวตาราง (headers ต้องเป็นตัวพิมพ์เล็กและ trim แล้ว) ใช้ร่วมกันทุกฟังก์ชัน
function _makeFindCol(headers) {
  return (...keywords) => {
    for (const kw of keywords) {
      const idx = headers.findIndex(h => h === kw.toLowerCase());
      if (idx !== -1) return idx;
    }
    const sorted = [...keywords].sort((a, b) => b.length - a.length);
    for (const kw of sorted) {
      const idx = headers.findIndex(h => h.includes(kw.toLowerCase()));
      if (idx !== -1) return idx;
    }
    return -1;
  };
}

// คอลัมน์ V "แปลงผิดปกติ": มีค่า (ที่ไม่ใช่ 0 / FALSE / ปกติ) = แปลงผิดปกติ
function _isAbnormal(v) {
  const s = String(v == null ? '' : v).trim().toLowerCase();
  return s !== '' && s !== '0' && s !== 'false' && s !== 'no' && s !== 'ไม่' && s !== 'ปกติ';
}

// อ่านข้อมูลชีต Data (รองรับกรณี A:R มาจาก IMPORTRANGE)
function _readSheet() {
  const ss    = _getMainSS();
  const sheet = ss.getSheetByName(SHEET_NAME);
  if (!sheet) return null;
  const lastRow = sheet.getLastRow();
  if (lastRow < 2) return [];
  const numCols = Math.min(COL.ABNORMAL + 1, sheet.getMaxColumns()); // อ่านถึงคอลัมน์ V

  // ไม่ใช้ IMPORTRANGE -> อ่านแบบปกติ
  if (!_isImportMode(sheet)) return sheet.getRange(2, 1, lastRow - 1, numCols).getValues();

  // ใช้ IMPORTRANGE -> A:O อ่านจากไฟล์ต้นทางโดยตรง / S:V อ่านจากไฟล์นี้
  const src     = _getSrcSheet();
  const srcLast = src.getLastRow();
  if (srcLast < 2) return [];
  const base = src.getRange(2, 1, srcLast - 1, SRC_READ_COLS).getValues();
  const pad  = SRC_COLS - SRC_READ_COLS; // เติมช่องว่างแทน P:R เพื่อให้ดัชนีคอลัมน์ S:V (18-21) ตรงเดิม

  const extraCols = numCols - SRC_COLS; // S:V
  let extra = [];
  if (extraCols > 0) {
    const rows = Math.min(srcLast - 1, sheet.getMaxRows() - 1);
    extra = sheet.getRange(2, SRC_COLS + 1, rows, extraCols).getValues();
  }
  for (let i = 0; i < base.length; i++) {
    const row = base[i];
    for (let q = 0; q < pad; q++) row.push('');
    for (let c = 0; c < extraCols; c++) row.push(extra[i] ? extra[i][c] : '');
  }
  return base;
}

// เก็บแคชแบบ gzip + base64 แบ่งเป็นชิ้น
function _putCache(key, str) {
  const cache = CacheService.getScriptCache();
  const gz    = Utilities.gzip(Utilities.newBlob(str, 'application/json'));
  const b64   = Utilities.base64Encode(gz.getBytes());
  const pairs = {};
  let n = 0;
  for (let i = 0; i < b64.length; i += CHUNK_SIZE) pairs[key + '_' + (n++)] = b64.slice(i, i + CHUNK_SIZE);
  pairs[key + '_n'] = String(n);
  cache.putAll(pairs, CACHE_TTL);
}

function _getCache(key) {
  try {
    const cache = CacheService.getScriptCache();
    const nStr  = cache.get(key + '_n');
    if (!nStr) return null;
    const n    = parseInt(nStr);
    const keys = Array.from({ length: n }, (_, i) => key + '_' + i);
    const parts = cache.getAll(keys);
    if (Object.keys(parts).length !== n) return null;
    const bytes = Utilities.base64Decode(keys.map(k => parts[k]).join(''));
    return Utilities.ungzip(Utilities.newBlob(bytes, 'application/x-gzip')).getDataAsString('UTF-8');
  } catch (e) { return null; }
}

// ดึงจากแคช ถ้าไม่มีให้ build ใหม่ แล้วส่งกลับเป็น "ข้อความ JSON"
function _cachedJson(key, forceRefresh, builder) {
  if (!forceRefresh) {
    const cached = _getCache(key);
    if (cached) return cached;
  }
  const t0 = Date.now();
  let result;
  try { result = builder(); } catch (e) { result = { error: e.message }; }
  const str = JSON.stringify(result);
  // ถ้ามีการบันทึกระหว่างที่อ่านข้อมูล ข้อมูลที่อ่านมาอาจเก่า -> ไม่เก็บลงแคช
  if ((!result || !result.error) && !_savedSince(t0)) { try { _putCache(key, str); } catch (e) {} }
  return str;
}

function _removeCacheKeys(bases) {
  try {
    const cache = CacheService.getScriptCache();
    const nVals = cache.getAll(bases.map(b => b + '_n'));
    const keys  = [];
    bases.forEach(base => {
      const nStr = nVals[base + '_n'];
      const n    = nStr ? parseInt(nStr) : 5;
      keys.push(base + '_n');
      for (let i = 0; i < n + 2; i++) keys.push(base + '_' + i);
    });
    cache.removeAll(keys);
  } catch (e) {}
}

// ★ [แก้ไข ข้อ 78] หลังบันทึก: แก้ "ลำดับการตัด" ในแคช CK_MAIN ตรงๆ โดยไม่อ่านชีตใหม่
//   (เดิมอ่านชีต Data + Code ทั้งก้อนใหม่ภายใน Lock ทำให้บันทึกช้า)
//   ชีต Code มีสูตรอิงชีต Data จึงล้างแคชไว้ให้โหลดใหม่ (หน้าเว็บโหลดใหม่เองหลังบันทึก)
function _patchCacheAfterSave(resolved) {
  try { CacheService.getScriptCache().put(CK_SAVED, String(Date.now()), 1200); } catch (e) {}
  _removeCacheKeys([CK_CODE]);
  try {
    const str = _getCache(CK_MAIN);
    if (!str) return;
    const data = JSON.parse(str);
    const rows = data && data.allRows;
    if (!rows) { _removeCacheKeys([CK_MAIN]); return; }
    let idxByCode = null;
    resolved.forEach(u => {
      if (!u.orderChanged) return;
      const code = String(u.plotCode || '').trim();
      let row = rows[u.sheetRow - 2];
      if (!row || row[R_PLOT] !== code) { // เลขแถวไม่ตรง -> ค้นด้วยรหัสแปลง
        if (!idxByCode) {
          idxByCode = {};
          for (let i = 0; i < rows.length; i++) idxByCode[rows[i][R_PLOT]] = i;
        }
        row = rows[idxByCode[code]];
      }
      if (row) row[R_ORDER] = String(u.cutOrder == null ? '' : u.cutOrder).trim();
    });
    _putCache(CK_MAIN, JSON.stringify(data));
  } catch (e) { _removeCacheKeys([CK_MAIN]); }
}

// รันเองใน Apps Script editor เมื่ออยากล้างแคชทั้งหมด (หน้าเว็บไม่ได้เรียกฟังก์ชันนี้)
function clearCache() {
  _removeCacheKeys([CK_MAIN, CK_CODE, CK_INSPECT, CK_MAP]);
}

function getInitialData(forceRefresh) {
  return _cachedJson(CK_MAIN, forceRefresh, _buildInitialData);
}

// คอลัมน์ของ allRows ที่ส่งเป็นเลขอ้างอิงพจนานุกรม (ข้อความซ้ำเยอะ)
// 1 ชื่อนักเกษตร / 2 โซน / 3 ประเภทอ้อย / 6 เลขโควต้า / 7 ชื่อชาวไร่ / 8 โรงงาน / 9 ประเภทการตัด /
// 10 กลุ่มตัด / 11 ชื่อกลุ่มตัด / 16 เกณฑ์การตัด / 18 รหัสนักเกษตร
const DICT_COLS = [1, 2, 3, 6, 7, 8, 9, 10, 11, 16, 18];

// แถวชีต Code: ใช้แคชถ้ามี ไม่มีค่อยอ่านสด (ใช้ _buildCodeData ตัวเดียว ไม่มีโค้ดอ่านซ้ำสองชุดแล้ว)
function _getCodeRows() {
  try {
    const cc = _getCache(CK_CODE);
    if (cc) { const cd = JSON.parse(cc); if (cd && cd.rows) return cd.rows; }
  } catch (e) {}
  const d = _buildCodeData();
  return (d && d.rows) || [];
}

function _buildInitialData(codeRowsIn) {
  const values = _readSheet();
  if (!values)        return { error: "ไม่พบ Sheet: " + SHEET_NAME };
  if (!values.length) return { allRows: [] };

  // ถ้าคอลัมน์รหัสแปลงว่างทั้งหมด = อ่านข้อมูลไม่ได้ -> แจ้ง error (และไม่ถูกแคช)
  let hasCode = false;
  for (let i = 0; i < values.length; i++) {
    if (String(values[i][COL.PLOT_INTECH] == null ? '' : values[i][COL.PLOT_INTECH]).trim()) { hasCode = true; break; }
  }
  if (!hasCode) return { error: 'อ่านข้อมูลแปลงไม่ได้ (คอลัมน์รหัสแปลงว่างทั้งหมด) กรุณาตรวจสิทธิ์เข้าถึงไฟล์ต้นทาง' };

  const statusMap = {}; // รหัสกลุ่มตัด -> สถานะกลุ่ม (หน้าเว็บใช้กำหนดประเภทการตัด)
  (codeRowsIn || _getCodeRows()).forEach(function(x) { if (x.code) statusMap[x.code] = x.status || ''; });

  const plotCodeSeen = {}, duplicatePlotCodesSet = {};
  const allRows = new Array(values.length);
  const len = values.length;

  for (let i = 0; i < len; i++) {
    const r      = values[i];
    const plotCode = r[COL.PLOT_INTECH] != null ? String(r[COL.PLOT_INTECH]).trim() : '';

    if (plotCode) {
      if (plotCodeSeen[plotCode]) duplicatePlotCodesSet[plotCode] = 1;
      plotCodeSeen[plotCode] = 1;
    }

    const ageRaw = r[COL.CANE_AGE];
    allRows[i] = [
      i + 2,
      r[COL.FARMER_NAME]    != null ? String(r[COL.FARMER_NAME])    : '',
      r[COL.ZONE]           != null ? String(r[COL.ZONE])           : '',
      r[COL.CANE_TYPE]      != null ? String(r[COL.CANE_TYPE])      : '',
      parseFloat(r[COL.AREA]) || 0,
      plotCode,
      r[COL.PLOT_CODE]      != null ? String(r[COL.PLOT_CODE])      : '',
      r[COL.OWNER_NAME]     != null ? String(r[COL.OWNER_NAME])     : '',
      r[COL.FACTORY]        != null ? String(r[COL.FACTORY])        : '',
      r[COL.CUT_TYPE]       != null ? String(r[COL.CUT_TYPE]).trim() : '',
      r[COL.CUT_GROUP]      != null ? String(r[COL.CUT_GROUP])      : '',
      r[COL.CUT_GROUP_NAME] != null ? String(r[COL.CUT_GROUP_NAME]) : '',
      parseFloat(r[COL.EST_TON])     || 0,
      Math.round((parseFloat(r[COL.TON_PER_RAI]) || 0) * 100) / 100, // ปัดเฉพาะค่าแสดงผล
      (ageRaw === '' || ageRaw == null || isNaN(parseFloat(ageRaw))) ? '' : Math.round(parseFloat(ageRaw)),
      (r[COL.CUT_ORDER] === '' || r[COL.CUT_ORDER] == null) ? '' : String(r[COL.CUT_ORDER]).trim(), // index 15 = R_ORDER
      r[COL.CUT_CRITERIA] != null ? String(r[COL.CUT_CRITERIA]).trim() : '',
      _isAbnormal(r[COL.ABNORMAL]) ? 1 : 0, // index 17 = คอลัมน์ V "แปลงผิดปกติ" (1 = ผิดปกติ)
      r[COL.FARMER_ID] != null ? String(r[COL.FARMER_ID]).trim() : '', // index 18 = รหัสนักเกษตร
    ];
  }

  // เข้ารหัสข้อความซ้ำเป็นเลขอ้างอิงพจนานุกรม
  const dict = [], dmap = new Map();
  for (let i = 0; i < len; i++) {
    const row = allRows[i];
    for (let k = 0; k < DICT_COLS.length; k++) {
      const c = DICT_COLS[k];
      const v = row[c];
      let id = dmap.get(v);
      if (id === undefined) { id = dict.length; dict.push(v); dmap.set(v, id); }
      row[c] = id;
    }
  }

  return {
    allRows,
    dict,
    dictCols: DICT_COLS,
    statusMap,
    duplicatePlotCodes: Object.keys(duplicatePlotCodesSet),
  };
}

// ─── ตัวช่วยเพิ่มความเร็ว ──────────────────────────────

// มีการบันทึกหลังเวลา t0 หรือไม่ (ใช้กันแคชเก่า)
function _savedSince(t0) {
  try { return Number(CacheService.getScriptCache().get(CK_SAVED) || 0) > t0; }
  catch (e) { return false; }
}

// หาเลขแถวจริงของแต่ละรหัสแปลง: ใช้เลขแถวที่ client จำไว้ (rowHint) แล้ว "ตรวจรหัสแปลงที่ชีตจริง" ก่อน
// อ่านเฉพาะช่วงแถวที่เกี่ยวข้องคอลัมน์เดียว ถ้ามีรหัสไม่ตรง (แถวถูกเลื่อน/ลบ) -> ค้นทั้งคอลัมน์
function _resolveRows(codeSheet, updates) {
  const col = COL.PLOT_INTECH + 1;
  const hints = updates.map(u => parseInt(u.rowHint, 10) || 0);
  if (hints.length && hints.every(h => h >= 2)) {
    let lo = hints[0], hi = hints[0];
    hints.forEach(h => { if (h < lo) lo = h; if (h > hi) hi = h; });
    if (hi <= codeSheet.getLastRow()) {
      const vals = codeSheet.getRange(lo, col, hi - lo + 1, 1).getValues();
      const map = {};
      let ok = true;
      for (let i = 0; i < updates.length; i++) {
        const code = String(updates[i].plotCode || '').trim();
        const v = vals[hints[i] - lo];
        if (!code || !v || String(v[0] == null ? '' : v[0]).trim() !== code) { ok = false; break; }
        map[code] = hints[i];
      }
      if (ok) return map;
    }
  }
  return _buildPlotCodeRowMap(codeSheet);
}

// อุ่นแคช (ให้ trigger เรียกทุก 10 นาที) ผู้ใช้เปิดเว็บแล้วเจอแคชเกือบทุกครั้ง
// ★ [แก้ไข ข้อ 78] อ่านชีต Code ก่อน แล้วส่งต่อให้ข้อมูลแปลง (สถานะกลุ่มตัดจึงเป็นรอบเดียวกันเสมอ)
function warmCache() {
  const t0  = Date.now();
  const out = [];
  let codeRes = null;
  try {
    codeRes = _buildCodeData();
    if (codeRes && !codeRes.error) out.push([CK_CODE, JSON.stringify(codeRes)]);
  } catch (e) {}
  const codeRows = codeRes && !codeRes.error ? codeRes.rows : null;
  [[CK_MAIN, function() { return _buildInitialData(codeRows); }], [CK_INSPECT, _buildInspectData], [CK_MAP, _buildMapCoords]]
    .forEach(function(j) {
      try { const r = j[1](); if (r && !r.error) out.push([j[0], JSON.stringify(r)]); } catch (e) {}
    });
  if (_savedSince(t0)) return; // มีการบันทึกระหว่างอ่าน ข้อมูลอาจเก่า -> ข้ามรอบนี้
  out.forEach(function(o) { try { _putCache(o[0], o[1]); } catch (e) {} });
}

// รันครั้งเดียวใน Apps Script editor เพื่อสร้าง trigger อุ่นแคชทุก 10 นาที
function installWarmTrigger() {
  removeWarmTrigger();
  ScriptApp.newTrigger('warmCache').timeBased().everyMinutes(10).create();
  warmCache();
}

function removeWarmTrigger() {
  ScriptApp.getProjectTriggers().forEach(t => {
    if (t.getHandlerFunction() === 'warmCache') ScriptApp.deleteTrigger(t);
  });
}

function getCodeData(forceRefresh) {
  return _cachedJson(CK_CODE, forceRefresh, _buildCodeData);
}

// ส่งเฉพาะฟิลด์ที่หน้าเว็บใช้จริง
function _buildCodeData() {
  try {
    const ss    = _getMainSS();
    const sheet = ss.getSheetByName('Code');
    if (!sheet) return { error: 'ไม่พบ Sheet: Code' };
    const lastRow = sheet.getLastRow();
    if (lastRow < 2) return { rows: [] };
    const numCols = sheet.getLastColumn();

    const allData = sheet.getRange(1, 1, lastRow, numCols).getValues();
    const headers = allData[0].map(h => String(h).trim().toLowerCase());
    const values  = allData.slice(1);
    const findCol = _makeFindCol(headers);

    const iZone    = findCol('เขต');
    const iCode    = findCol('code กลุ่มตัด', 'code');
    const iName    = findCol('ชื่อกลุ่มตัด');
    const iBrand   = findCol('ยี่ห้อ', 'brand', 'ยี่ห้อรถ');
    const iUnit    = findCol('คัน หรือ คน', 'คันหรือคน');
    const iTruck   = findCol('รถบรรทุก (คัน)', 'รถบรรทุก');
    const iQueue   = findCol('จำนวนคิว', 'จำนวนคัน', 'คิว');
    const iRegRai  = findCol('จดแจ้ง (ไร่/หีบ)', 'จดแจ้ง (ไร่)', 'จดแจ้ง(ไร่)');
    const iRegTon  = findCol('จดแจ้ง (ตัน/หีบ)', 'จดแจ้ง (ตัน)', 'จดแจ้ง(ตัน)');
    let iStatus    = findCol('สถานะกลุ่ม', 'สถานกลุ่ม');
    const iVerify  = findCol('ตรวจสอบ');
    // คอลัมน์ "แสดงข้อมูล" 1 = แสดง
    // (AF = วันที่พร้อมตัด, AG = หมายเหตุ ถูกเพิ่มเข้ามา -> "แสดงข้อมูล" ย้ายไปอยู่ถัดไป)
    // หาจากหัวคอลัมน์ก่อน ถ้าไม่เจอใช้ตำแหน่งสำรอง (ปรับเลขให้ตรงกับคอลัมน์จริง)
    // AF = แสดงข้อมูล (ดัชนี 31) / AG = วันที่พร้อมตัด (32) / AH = หมายเหตุ (33)
    const SHOW_FALLBACK_IDX  = 31;
    const READY_FALLBACK_IDX = 32;
    const NOTE_FALLBACK_IDX  = 33;
    let iShow = findCol('แสดงข้อมูล');
    if (iShow === -1 && numCols > SHOW_FALLBACK_IDX) iShow = SHOW_FALLBACK_IDX;
    let iReady = findCol('วันที่พร้อมตัด');
    if (iReady === -1 && numCols > READY_FALLBACK_IDX) iReady = READY_FALLBACK_IDX;
    let iNote = findCol('หมายเหตุ');
    if (iNote === -1 && numCols > NOTE_FALLBACK_IDX) iNote = NOTE_FALLBACK_IDX;
    // 4 คอลัมน์ "แผนตัดธันวาคม" (แผน / จดแจ้ง) x (ไร่ / ตัน)
    const _decCol = (isReg, unit) => headers.findIndex(h =>
      h.includes('แผนตัดธันวาคม') &&
      (isReg ? h.includes('จดแจ้ง') : !h.includes('จดแจ้ง')) &&
      h.includes(unit));
    let iDecPlan = _decCol(false, 'ไร่');
    let iDecReg  = _decCol(true,  'ไร่');
    const iDecPlanTon = _decCol(false, 'ตัน');
    const iDecRegTon  = _decCol(true,  'ตัน');
    // fallback: หัวคอลัมน์ไม่มีหน่วย
    if (iDecPlan === -1) iDecPlan = headers.findIndex((h, idx) => h.includes('แผนตัดธันวาคม') && !h.includes('จดแจ้ง') && idx !== iDecPlanTon);
    if (iDecReg  === -1) iDecReg  = headers.findIndex((h, idx) => h.includes('แผนตัดธันวาคม') &&  h.includes('จดแจ้ง') && idx !== iDecRegTon);
    if (iStatus === -1) {
      const usedIdx = new Set([iZone,iCode,iName,iBrand,iUnit,iTruck,iQueue,iRegRai,iRegTon,iVerify]);
      iStatus = headers.findIndex((h, idx) => !usedIdx.has(idx) && h.includes('สถาน'));
    }
    if (iStatus === -1) iStatus = numCols - 1;

    const _s = (r, i) => i >= 0 && r[i] !== undefined && r[i] !== '' ? String(r[i]).trim() : '';
    const _n = (r, i) => i >= 0 && r[i] !== undefined && r[i] !== '' ? (parseFloat(r[i]) || 0) : '';
    const tz = Session.getScriptTimeZone();
    const _dt = (r, i) => {
      if (i < 0 || r[i] === undefined || r[i] === '') return '';
      const v = r[i];
      if (Object.prototype.toString.call(v) === '[object Date]') {
        try { return Utilities.formatDate(v, tz, 'd/M/yyyy'); } catch (e) { return String(v); }
      }
      return String(v).trim();
    };

    const rows = [];
    for (let i = 0; i < values.length; i++) {
      const r    = values[i];
      const code = _s(r, iCode);
      if (!code) continue;

      rows.push({
        zone:   _s(r, iZone),
        code,
        name:   _s(r, iName),
        status: _s(r, iStatus),
        brand:  _s(r, iBrand),
        unit:   _s(r, iUnit),
        truck:  _n(r, iTruck),
        queue:  _n(r, iQueue),
        regRai: _n(r, iRegRai),
        regTon: _n(r, iRegTon),
        verify: _s(r, iVerify) || 'ยังไม่ยืนยัน',
        decPlanRai: _n(r, iDecPlan),    // แผนตัดธันวาคม (ไร่)
        decRegRai:  _n(r, iDecReg),     // จดแจ้งแผนตัดธันวาคม (ไร่)
        decPlanTon: _n(r, iDecPlanTon), // แผนตัดธันวาคม (ตัน)
        decRegTon:  _n(r, iDecRegTon),  // จดแจ้งแผนตัดธันวาคม (ตัน)
        readyDate:  _dt(r, iReady),     // วันที่พร้อมตัด (AG)
        note:       _s(r, iNote),       // หมายเหตุ (AH)
        show: iShow === -1 ? 1 : (String(r[iShow] == null ? '' : r[iShow]).trim() === '1' ? 1 : 0),
      });
    }

    return { rows };
  } catch (e) {
    return { error: e.message };
  }
}

// ─── ★ Map "รหัสแปลง -> เลขแถวปัจจุบัน" อ่านสดจากชีตทุกครั้งก่อนเขียน ──────
// รับชีตที่เก็บรหัสแปลงจริง (ถ้าชีต Data ใช้ IMPORTRANGE คือชีตต้นทาง)
function _buildPlotCodeRowMap(codeSheet) {
  const map = {};
  const lastRow = codeSheet.getLastRow();
  if (lastRow < 2) return map;
  const codes = codeSheet.getRange(2, COL.PLOT_INTECH + 1, lastRow - 1, 1).getValues();
  for (let i = 0; i < codes.length; i++) {
    const code = codes[i][0] != null ? String(codes[i][0]).trim() : '';
    if (code) map[code] = i + 2;
  }
  return map;
}

// เขียนหลายแถวให้ใช้จำนวนครั้งเรียก API น้อยที่สุด
//   rowMap = { เลขแถว: [ค่าแต่ละคอลัมน์] }
//   - ช่วงแถวที่แก้มีไม่เกิน MERGE_GROUPS_MIN ก้อน -> เขียนทีละก้อน
//   - กระจายหลายก้อนแต่อยู่ในช่วงไม่เกิน MERGE_SPAN_MAX แถว -> อ่านช่วงนั้นครั้งเดียว แทนค่าเฉพาะแถวที่แก้
//     แล้วเขียนกลับครั้งเดียว (แถวที่ไม่ได้แก้คงค่าเดิม และถ้าเป็นสูตรจะเขียนสูตรเดิมกลับ ไม่กลายเป็นค่า)
const MERGE_SPAN_MAX = 60000;
const MERGE_GROUPS_MIN = 8;
function _writeRowsSmart(sheet, startCol, width, rowMap) {
  const rows = Object.keys(rowMap).map(Number).sort((a, b) => a - b);
  if (!rows.length) return;
  const groups = [[rows[0]]];
  for (let i = 1; i < rows.length; i++) {
    if (rows[i] === rows[i - 1] + 1) groups[groups.length - 1].push(rows[i]);
    else groups.push([rows[i]]);
  }
  const first = rows[0], last = rows[rows.length - 1], span = last - first + 1;
  if (groups.length > MERGE_GROUPS_MIN && span <= MERGE_SPAN_MAX) {
    const rng  = sheet.getRange(first, startCol, span, width);
    const vals = rng.getValues();
    const fmls = rng.getFormulas();
    const out  = new Array(span);
    for (let i = 0; i < span; i++) {
      const upd = rowMap[first + i];
      out[i] = upd ? upd : vals[i].map((v, j) => fmls[i][j] ? fmls[i][j] : v);
    }
    rng.setValues(out);
    return;
  }
  groups.forEach(g => {
    sheet.getRange(g[0], startCol, g.length, width).setValues(g.map(r => rowMap[r]));
  });
}

// "ถึงเกณฑ์" หรือไม่ (ระวัง "ไม่ถึงเกณฑ์" มีคำว่า "ถึงเกณฑ์" อยู่ข้างในด้วย)
function _isCriteriaMet(v) {
  const s = String(v == null ? '' : v).trim();
  return s.indexOf('ถึงเกณฑ์') !== -1 && s.indexOf('ไม่') === -1;
}

// แปลงค่าลำดับการตัดก่อนเขียนลงชีต: ว่าง -> '' / ตัวเลข -> number / อื่นๆ -> ข้อความเดิม
function _normOrder(v) {
  const s = String(v == null ? '' : v).trim();
  if (!s) return '';
  const n = Number(s);
  return isNaN(n) ? s : n;
}

// ─── บันทึก "ลำดับการตัด" (คอลัมน์ U) (กันข้อมูลสลับแถว + กันบันทึกชนกัน) ────────────
function saveCutTypes(updates) {
  const lock = LockService.getScriptLock();
  try {
    lock.waitLock(15000);
  } catch (e) {
    return { ok: false, msg: 'ขณะนี้มีผู้ใช้อื่นกำลังบันทึกข้อมูลอยู่ กรุณารอสักครู่แล้วลองใหม่อีกครั้ง' };
  }

  try {
    const sheet = _getMainSS().getSheetByName(SHEET_NAME);
    if (!sheet) return { ok: false, msg: "ไม่พบ Sheet: " + SHEET_NAME };

    // ใช้เลขแถวที่ client ส่งมาแล้วตรวจรหัสแปลงที่ชีตจริง (ไม่ตรงค่อยค้นทั้งคอลัมน์)
    const codeSheet  = _isImportMode(sheet) ? _getSrcSheet() : sheet;
    const plotRowMap = _resolveRows(codeSheet, updates);

    const notFound = [];
    const resolved = [];
    updates.forEach(u => {
      const code = String(u.plotCode || '').trim();
      const row  = code ? plotRowMap[code] : undefined;
      if (!code || !row) {
        notFound.push(code || '(ไม่มีรหัสแปลง)');
        return;
      }
      resolved.push(Object.assign({}, u, { sheetRow: row }));
    });

    if (notFound.length) {
      return {
        ok: false,
        msg: 'ไม่พบรหัสแปลงต่อไปนี้ในข้อมูลปัจจุบัน (อาจถูกแก้ไข/ลบระหว่างทาง) กรุณากด "รีเฟรชข้อมูล" แล้วกรอกใหม่อีกครั้ง: ' + notFound.join(', ')
      };
    }

    const orderUpdates = resolved.filter(u => u.orderChanged);

    // ตรวจรูปแบบตัวเลข / เกณฑ์การตัด (อ่านเกณฑ์เฉพาะช่วงแถวที่แก้ และเฉพาะเมื่อมีการกรอกเลข)
    const filled = orderUpdates.filter(u => String(u.cutOrder == null ? '' : u.cutOrder).trim() !== '');
    if (filled.length) {
      let critLo = filled[0].sheetRow, critHi = critLo;
      filled.forEach(u => { if (u.sheetRow < critLo) critLo = u.sheetRow; if (u.sheetRow > critHi) critHi = u.sheetRow; });
      const critVals = sheet.getRange(critLo, COL.CUT_CRITERIA + 1, critHi - critLo + 1, 1).getValues();
      const notNum = [], notMet = [];
      filled.forEach(u => {
        if (!/^[0-9]+$/.test(String(u.cutOrder).trim())) { notNum.push(u.plotCode); return; }
        const crit = critVals[u.sheetRow - critLo] ? critVals[u.sheetRow - critLo][0] : '';
        if (!_isCriteriaMet(crit)) notMet.push(u.plotCode);
      });
      if (notNum.length) return { ok: false, msg: 'ลำดับการตัดต้องเป็นตัวเลขเท่านั้น: ' + notNum.join(', ') };
      if (notMet.length) return { ok: false, msg: 'แปลงต่อไปนี้ "ไม่ถึงเกณฑ์" จึงกรอกลำดับการตัดไม่ได้: ' + notMet.join(', ') };
    }

    // "ลำดับการตัด" (คอลัมน์ U) เขียนที่ชีต Data ของไฟล์นี้
    const orderMap = {};
    orderUpdates.forEach(u => { orderMap[u.sheetRow] = [_normOrder(u.cutOrder)]; });
    _writeRowsSmart(sheet, COL.CUT_ORDER + 1, 1, orderMap);

    _patchCacheAfterSave(orderUpdates); // แก้เฉพาะแคชข้อมูลแปลง ไม่อ่านชีตใหม่ / ไม่ล้างแคช GTT และพิกัดแผนที่
    return { ok: true, saved: resolved.length };
  } catch (e) {
    return { ok: false, msg: e.message };
  } finally {
    lock.releaseLock();
  }
}

// ─── ★ ผลตรวจแปลงรถตัดโรงงาน: ดึงข้อมูลจากไฟล์ "สำรวจแปลงอ้อย ปี 6970" ชีต GTT ──
function getInspectData(forceRefresh) {
  return _cachedJson(CK_INSPECT, forceRefresh, _buildInspectData);
}

function _buildInspectData() {
  try {
    const ss    = SpreadsheetApp.openById(INSPECT_SS_ID);
    const sheet = ss.getSheetByName(INSPECT_SHEET_NAME);
    if (!sheet) return { error: 'ไม่พบ Sheet: ' + INSPECT_SHEET_NAME + ' ในไฟล์สำรวจแปลงอ้อย ปี 6970' };
    const lastRow = sheet.getLastRow();
    if (lastRow < 2) return { rows: [] };
    const numCols = sheet.getLastColumn();

    const allData = sheet.getRange(1, 1, lastRow, numCols).getValues();
    const headers = allData[0].map(h => String(h).trim().toLowerCase());
    const values  = allData.slice(1);
    const findCol = _makeFindCol(headers);

    const iFarmerId     = findCol('รหัสนักเกษตร');
    const iFarmerName   = findCol('ชื่อนักเกษตร');
    const iZone         = findCol('โซน', 'เขต');
    const iCaneType     = findCol('ชื่อประเภทอ้อย', 'ชื่อพันธุ์อ้อย', 'พันธุ์อ้อย');
    const iArea         = findCol('พื้นที่');
    const iPlotNo       = findCol('เลขแปลง');
    const iQuotaNo      = findCol('เลขโควต้า');
    const iOwner        = findCol('ชื่อชาวไร่');
    const iCutGroup     = findCol('กลุ่มตัด');
    const iCutGroupName = findCol('ชื่อกลุ่มตัด');
    const iCheckDate    = findCol('วันที่ตรวจ/แก้ไข', 'วันที่ตรวจ', 'วันที่แก้ไข', 'วันที่ตรวจสอบ');
    const iResult       = findCol('ผลการตรวจ');
    const iReason       = findCol('สาเหตุไม่รับตัด', 'สาเหตุ');

    const _s = (r, i) => i >= 0 && r[i] !== undefined && r[i] !== '' ? String(r[i]).trim() : '';
    const _n = (r, i) => i >= 0 && r[i] !== undefined && r[i] !== '' ? (parseFloat(r[i]) || 0) : '';
    const tz = Session.getScriptTimeZone();
    const _d = (r, i) => {
      if (i < 0 || r[i] === undefined || r[i] === '') return '';
      const v = r[i];
      if (Object.prototype.toString.call(v) === '[object Date]') {
        try { return Utilities.formatDate(v, tz, 'dd/MM/yyyy'); }
        catch (e) { return String(v); }
      }
      return String(v).trim();
    };

    const rows = [];
    for (let i = 0; i < values.length; i++) {
      const r = values[i];
      const farmerId   = _s(r, iFarmerId);
      const farmerName = _s(r, iFarmerName);
      const plotNo     = _s(r, iPlotNo);
      if (!farmerId && !farmerName && !plotNo) continue;

      rows.push({
        farmerId,
        farmerName,
        zone:         _s(r, iZone),
        caneType:     _s(r, iCaneType),
        area:         _n(r, iArea),
        plotNo,
        quotaNo:      _s(r, iQuotaNo),
        owner:        _s(r, iOwner),
        cutGroup:     _s(r, iCutGroup),
        cutGroupName: _s(r, iCutGroupName),
        checkDate:    _d(r, iCheckDate),
        result:       _s(r, iResult),
        reason:       _s(r, iReason),
      });
    }

    return { rows };
  } catch (e) {
    return { error: e.message };
  }
}

// ─── ★ พิกัดแปลงสำหรับหน้าแผนที่ (อ่านจากคอลัมน์ P:R ของชีต Data ต้นทาง) ──────
function getMapCoords(forceRefresh) {
  return _cachedJson(CK_MAP, forceRefresh, _buildMapCoords);
}

function _buildMapCoords() {
  try {
    const ss    = _getMainSS();
    const sheet = ss.getSheetByName(SHEET_NAME);
    if (!sheet) return { error: 'ไม่พบ Sheet: ' + SHEET_NAME };
    const src     = _isImportMode(sheet) ? _getSrcSheet() : sheet;
    const lastRow = src.getLastRow();
    if (lastRow < 2) return { plots: [] };
    if (src.getLastColumn() < 17) return { error: 'ชีต Data ไม่มีคอลัมน์พิกัด (P:R)' };
    const n = lastRow - 1;

    // หาคอลัมน์ lat / long จากหัวตาราง (P:R) ถ้าไม่เจอใช้ค่าคงที่ด้านบน
    const head = src.getRange(1, 16, 1, 3).getValues()[0].map(h => String(h).trim().toLowerCase());
    let iLat = MAP_LAT_IDX - 15, iLng = MAP_LNG_IDX - 15;
    const a = head.findIndex(h => h.indexOf('lat') !== -1 || h.indexOf('ละติจูด') !== -1);
    const b = head.findIndex(h => h.indexOf('long') !== -1 || h.indexOf('lng') !== -1 || h.indexOf('ลองจิจูด') !== -1);
    if (a !== -1) iLat = a;
    if (b !== -1) iLng = b;

    const codes  = src.getRange(2, COL.PLOT_INTECH + 1, n, 1).getValues();
    const coords = src.getRange(2, 16, n, 3).getValues();
    const plots = [];
    for (let i = 0; i < n; i++) {
      const code = codes[i][0] != null ? String(codes[i][0]).trim() : '';
      if (!code) continue;
      let lat = parseFloat(coords[i][iLat]), lng = parseFloat(coords[i][iLng]);
      if (!isFinite(lat) || !isFinite(lng)) continue;
      if (lat > 40 && lng < 40) { const t = lat; lat = lng; lng = t; } // กรอกสลับช่อง
      if (lat < 4 || lat > 22 || lng < 94 || lng > 108) continue;      // นอกประเทศไทย = ข้อมูลผิด
      plots.push([code, Math.round(lat * 1e6) / 1e6, Math.round(lng * 1e6) / 1e6]);
    }
    return { plots };
  } catch (e) {
    return { error: e.message };
  }
}

// ─── HTML ──────────────────────────────────────────────────────────────────
function getHtmlPage(pre) {
  return `<!DOCTYPE html>
<html lang="th">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>ติดตามประเภทการตัด ปี 69/70</title>
<script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.0/dist/chart.umd.min.js" defer><\/script>
<script src="https://cdn.jsdelivr.net/npm/chartjs-plugin-datalabels@2.2.0/dist/chartjs-plugin-datalabels.min.js" defer><\/script>
<style>
* { box-sizing: border-box; margin: 0; padding: 0; }
html, body { height: 100%; }
body { font-family: 'Sarabun','Segoe UI',sans-serif; background: #f0f4f8; color: #1a202c; }
.app-shell { display: flex; min-height: 100vh; }
.sidebar {
  width: 250px; flex-shrink: 0; background: #ffffff;
  box-shadow: 2px 0 10px rgba(0,0,0,.06); display: flex; flex-direction: column;
  position: sticky; top: 0; height: 100vh; z-index: 100;
}
.sidebar-brand { display: flex; align-items: center; gap: 10px; padding: 20px 18px; border-bottom: 1px solid #e2e8f0; }
.sidebar-brand-icon { width: 38px; height: 38px; border-radius: 10px; flex-shrink: 0; background: linear-gradient(135deg,#1b4332,#52b788); display: flex; align-items: center; justify-content: center; font-size: 1.2rem; }
.sidebar-brand-text { line-height: 1.25; min-width: 0; }
.sidebar-brand-text strong { display: block; font-size: .95rem; color: #1a202c; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.sidebar-brand-text span { display: block; font-size: .72rem; color: #718096; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.sidebar-section-title { font-size: .72rem; font-weight: 700; color: #a0aec0; letter-spacing: .03em; padding: 16px 18px 6px; }
.sidebar-menu { display: flex; flex-direction: column; gap: 2px; padding: 0 10px; }
.menu-item { display: flex; align-items: center; gap: 10px; width: 100%; border: none; background: transparent; cursor: pointer; text-align: left; font-family: inherit; font-size: .9rem; font-weight: 600; color: #4a5568; padding: 10px 12px; border-radius: 8px; transition: background .15s, color .15s; }
.menu-item .menu-icon { font-size: 1.05rem; flex-shrink: 0; }
.menu-item:hover { background: #f0f4f8; }
.menu-item.active { background: #e6f4ea; color: #1b4332; }
.menu-item.refresh-btn { color: #2b6cb0; margin-top: 8px; border-top: 1px solid #e2e8f0; padding-top: 14px; }
.menu-item.refresh-btn:hover { background: #ebf8ff; color: #2c5282; }
.main-content { flex: 1; min-width: 0; height: 100vh; overflow-y: auto; }
@media(max-width:768px){
  .app-shell { flex-direction: column; }
  .sidebar { width: 100%; height: auto; position: sticky; top: 0; flex-direction: row; align-items: center; flex-wrap: wrap; }
  .sidebar-brand { border-bottom: none; padding: 10px 14px; }
  .sidebar-section-title { display: none; }
  .sidebar-menu { flex-direction: row; padding: 0 10px 10px; flex-wrap: wrap; }
  .main-content { height: auto; overflow-y: visible; }
}
header { background: linear-gradient(135deg,#1b4332 0%,#2d6a4f 100%); color: white; text-align: center; padding: 18px 24px 12px; transition: background .2s; }
header h1 { font-size: 1.5rem; font-weight: 700; }
header small { font-size: .9rem; opacity: .9; }
header.header-cutgroup { background: linear-gradient(135deg,#3aa6d6 0%,#89CFF0 100%); }
header.header-table    { background: linear-gradient(135deg,#cc7000 0%,#FF8C00 100%); }
header.header-map      { background: linear-gradient(135deg,#0f8b78 0%,#26bfa9 100%); }
.container { max-width: 1400px; margin: 0 auto; padding: 20px 16px; }
.cards { display: grid; grid-template-columns: repeat(auto-fit, minmax(260px, 1fr)); gap: 16px; margin-bottom: 24px; }
.card { background: white; border-radius: 14px; padding: 20px 24px; box-shadow: 0 2px 8px rgba(0,0,0,.08); display: flex; flex-direction: column; gap: 10px; position: relative; overflow: hidden; min-height: 140px; }
.card::before { content:''; position:absolute; top:0; left:0; right:0; height:4px; background: linear-gradient(90deg,#2d6a4f,#52b788); }
.card.blue::before { background: linear-gradient(90deg,#2b6cb0,#63b3ed); }
.card.orange::before { background: linear-gradient(90deg,#c2410c,#fb923c); }
.card.purple::before { background: linear-gradient(90deg,#6b21a8,#c084fc); }
.card-top { display:flex; align-items:center; gap:10px; }
.card-icon { width:38px; height:38px; border-radius:10px; display:flex; align-items:center; justify-content:center; font-size:1.2rem; flex-shrink:0; background:#e6f4ea; }
.card.blue .card-icon { background:#ebf8ff; }
.card.purple .card-icon { background:#f3e8ff; }
.card.orange .card-icon { background:#fff7ed; }
.card-label { font-size:clamp(0.8rem,1.6vw,1.28rem); font-weight:700; color:#2d3748; }
.card-divider { height:1px; background:#f0f4f8; }
.donut-card-body { display: flex; align-items: center; gap: 18px; flex-wrap: wrap; }
.donut-wrap { position: relative; width: 110px; height: 110px; flex-shrink: 0; }
.donut-wrap canvas { width: 110px !important; height: 110px !important; }
.donut-center { position: absolute; top: 0; left: 0; width: 100%; height: 100%; display: flex; flex-direction: column; align-items: center; justify-content: center; pointer-events: none; }
.donut-center-val { font-size: 1.15rem; font-weight: 800; color: #1a202c; line-height: 1.1; }
.donut-center-lbl { font-size: .62rem; font-weight: 600; color: #718096; margin-top: 1px; }
.donut-legend { display: flex; flex-direction: column; gap: 6px; flex: 1; min-width: 130px; }
.dl-item { display: flex; align-items: center; gap: 6px; font-size: .76rem; color: #4a5568; }
.dl-item .dl-txt { flex: 1; }
.dl-item strong { color: #1a202c; font-weight: 700; font-size: .79rem; }
.dl-dot { width: 10px; height: 10px; border-radius: 50%; flex-shrink: 0; }
.dl-total { font-size: .72rem; color: #1a202c; font-weight: 700; margin-top: 2px; padding-top: 5px; border-top: 1px dashed #e2e8f0; }
.donut-legend.dl-big .dl-item { font-size: .96rem; }
.donut-legend.dl-big .dl-item strong { font-size: 1rem; }
.donut-legend.dl-big .dl-total { font-size: .92rem; }
.donut-legend.dl-big .dl-total strong { font-size: .96rem; }
.combined-chart-box {
  background: #e8f5e9;
  border-radius: 14px;
  padding: 18px 16px 16px;
  box-shadow: 0 2px 8px rgba(45,106,79,.12);
  margin-bottom: 20px;
  border: 1.5px solid #b7dfc0;
}
.combined-chart-box-title {
  font-size: 1rem;
  font-weight: 700;
  color: #1b4332;
  margin-bottom: 14px;
  padding-bottom: 10px;
  border-bottom: 2px solid #b7dfc0;
}
.combined-chart-inner {
  display: grid;
  grid-template-columns: 300px 1fr;
  gap: 14px;
}
.chart-inner-panel {
  background: #f0faf2;
  border-radius: 10px;
  padding: 16px;
  box-shadow: 0 1px 4px rgba(45,106,79,.08);
}
@media(max-width:768px){ .combined-chart-inner { grid-template-columns: 1fr; } }
.combined-chart-box.blue { background: #eaf6fd; border-color: #bfe3f5; box-shadow: 0 2px 8px rgba(27,110,154,.12); }
.combined-chart-box.blue .combined-chart-box-title { color: #1b6e9a; border-bottom-color: #bfe3f5; }
.combined-chart-box.blue .chart-inner-panel { background: #f4fbfe; box-shadow: 0 1px 4px rgba(27,110,154,.08); }
.combined-chart-box.amber { background: #fff8e6; border-color: #f3e0a8; box-shadow: 0 2px 8px rgba(154,91,0,.12); }
.combined-chart-box.amber .combined-chart-box-title { color: #9a5b00; border-bottom-color: #f3e0a8; }
.combined-chart-box.amber .chart-inner-panel { background: #fffdf5; box-shadow: 0 1px 4px rgba(154,91,0,.08); }
.combined-chart-box.teal { background: #e6faf6; border-color: #a8e6d9; box-shadow: 0 2px 8px rgba(0,121,107,.12); }
.combined-chart-box.teal .combined-chart-box-title { color: #00695c; border-bottom-color: #a8e6d9; }
.combined-chart-box.teal .chart-inner-panel { background: #f3fefb; box-shadow: 0 1px 4px rgba(0,121,107,.08); }
.combined-chart-box.orange { background: #fff3e6; border-color: #fcd9b0; box-shadow: 0 2px 8px rgba(194,65,12,.12); }
.combined-chart-box.orange .combined-chart-box-title { color: #c2410c; border-bottom-color: #fcd9b0; }
.combined-chart-box.orange .chart-inner-panel { background: #fff9f2; box-shadow: 0 1px 4px rgba(194,65,12,.08); }
/* กล่อง "จัดลำดับตัด" (สีม่วงอ่อน) + บล็อกสรุป */
.combined-chart-box.violet { background: #f5f0ff; border-color: #ddd0fa; box-shadow: 0 2px 8px rgba(109,40,217,.12); }
.combined-chart-box.violet .combined-chart-box-title { color: #5b21b6; border-bottom-color: #ddd0fa; }
.combined-chart-box.violet .chart-inner-panel { background: #faf7ff; box-shadow: 0 1px 4px rgba(109,40,217,.08); }
.ord-c1::before { background: #7c3aed; }
.ord-c2::before { background: #f97316; }
.ord-c3::before { background: #16a34a; }
.ord-c4::before { background: #f59e0b; }
.ord-c3 .cg-scard-val { color: #15803d; }
.ord-c4 .cg-scard-val { color: #b45309; }
.pie-body { display: flex; align-items: center; gap: 18px; flex-wrap: wrap; }
.pie-canvas-wrap { flex: 1 1 220px; min-width: 200px; max-width: 280px; }
.pie-legend { display: flex; flex-direction: column; gap: 10px; flex: 1 1 200px; min-width: 180px; }
.pie-legend-item { display: flex; align-items: center; gap: 8px; font-size: .92rem; color: #2d3748; font-weight: 600; }
.pie-legend-dot { width: 13px; height: 13px; border-radius: 4px; flex-shrink: 0; }
.filters { background: white; border-radius: 10px; padding: 14px 16px; box-shadow: 0 1px 4px rgba(0,0,0,.08); display: flex; gap: 12px; flex-wrap: wrap; align-items: flex-end; margin-bottom: 16px; }
.filter-group { display: flex; flex-direction: column; gap: 4px; }
.filter-group label { font-size: .8rem; color: #718096; font-weight: 600; }
.filter-group select { border: 1px solid #e2e8f0; border-radius: 6px; padding: 7px 10px; font-size: .9rem; background: #f7fafc; cursor: pointer; min-width: 170px; }
.filter-group select:focus { outline: none; border-color: #2d6a4f; }
.btn-reset { background: #718096; color: white; border: none; border-radius: 6px; padding: 8px 18px; cursor: pointer; font-size: .9rem; align-self: flex-end; font-weight: 600; }
.btn-reset:hover { background: #4a5568; }
.chart-mode-pills { display: flex; gap: 6px; }
.chart-mode-pill { padding: 7px 16px; border-radius: 6px; border: 1px solid #e2e8f0; background: #f7fafc; font-size: .87rem; font-weight: 700; color: #4a5568; cursor: pointer; font-family: inherit; transition: all .15s; white-space: nowrap; }
.chart-mode-pill:hover { background: #edf2f7; }
.chart-mode-pill.active { background: #1b4332; color: #fff; border-color: #1b4332; }
.table-box { background: white; border-radius: 10px; box-shadow: 0 1px 4px rgba(0,0,0,.08); overflow: hidden; contain: layout style paint; }
.table-header { display: flex; align-items: center; justify-content: space-between; padding: 14px 16px 10px; border-bottom: 2px solid #e2e8f0; flex-wrap: wrap; gap: 8px; }
.table-header h3 { font-size: 1rem; font-weight: 700; color: #2d3748; }
.btn-save { background: #2d6a4f; color: white; border: none; border-radius: 6px; padding: 8px 20px; cursor: pointer; font-size: .9rem; font-weight: 700; display: flex; align-items: center; gap: 6px; }
.btn-save:hover:not(:disabled) { background: #1b4332; }
.btn-save:disabled { background: #a0aec0; cursor: not-allowed; }
.btn-save .dot { width: 8px; height: 8px; border-radius: 50%; background: #f6ad55; display: none; }
.btn-save.dirty .dot { display: inline-block; }
#toast { position: fixed; bottom: 24px; right: 24px; z-index: 9999; padding: 12px 20px; border-radius: 8px; font-size: .9rem; font-weight: 600; color: white; opacity: 0; transition: opacity .3s; pointer-events: none; }
#toast.show    { opacity: 1; }
#toast.success { background: #2d6a4f; }
#toast.error   { background: #e53e3e; }
.table-wrapper { overflow-x: auto; max-height: 460px; overflow-y: auto; }
table { width: 100%; border-collapse: collapse; font-size: .85rem; }
thead th { background: #1b4332; color: white; padding: 11px 12px; text-align: left; position: sticky; top: 0; z-index: 2; white-space: nowrap; transition: background .2s; }
#viewTable thead th { background: #FF8C00; }
tbody tr:nth-child(even) { background: #f7fafc; }
tbody tr:hover { background: #e6fffa; }
/* สีแถวตาราง "จดแจ้งประเภทการตัดอ้อย": ถึงเกณฑ์ = ขาว / ไม่ถึงเกณฑ์ = เทา / บันทึกลำดับการตัดแล้ว = เขียว */
tbody tr.row-age-old, tbody tr.row-age-old:nth-child(even)     { background: #ffffff !important; }
tbody tr.row-age-old:hover                                     { background: #f7fafc !important; }
tbody tr.row-age-young, tbody tr.row-age-young:nth-child(even) { background: #e2e8f0 !important; }
tbody tr.row-age-young:hover                                   { background: #d5dce6 !important; }
tbody tr.row-ordered, tbody tr.row-ordered:nth-child(even)     { background: #c6f6d5 !important; }
tbody tr.row-ordered:hover                                     { background: #b2f0c5 !important; }
.order-input { border: 1px solid #e2e8f0; border-radius: 6px; padding: 5px 8px; font-size: .85rem; background: #fff; width: 80px; text-align: center; font-family: inherit; }
.order-input:focus { outline: none; border-color: #2d6a4f; }
.order-input.changed { border-color: #f6ad55; background: #fffbeb; font-weight: 600; }
.order-input:disabled { background: #d5dce6; color: #718096; border-color: #cbd5e0; cursor: not-allowed; }
/* ตัวกรองรายคอลัมน์ (เหมือน Excel) ที่หัวตารางข้อมูลแปลง */
.colf-btn { background:transparent; border:none; color:#fff; cursor:pointer; font-size:.7rem; padding:2px 5px; margin-left:4px; border-radius:4px; font-family:inherit; line-height:1; }
.colf-btn:hover { background:rgba(255,255,255,.25); }
.colf-btn.active { background:#fff; color:#c2410c; }
#colfPanel { position:fixed; z-index:3000; background:#fff; border:1px solid #e2e8f0; border-radius:8px; box-shadow:0 6px 20px rgba(0,0,0,.2); width:260px; color:#1a202c; font-size:.85rem; }
#colfPanel .colf-head { display:flex; justify-content:space-between; gap:16px; padding:10px 12px; font-size:.78rem; font-weight:600; border-bottom:1px solid #edf2f7; }
#colfPanel .colf-head a { color:#0f8b78; cursor:pointer; user-select:none; }
#colfPanel .colf-head a:hover { text-decoration:underline; }
#colfPanel .colf-search { padding:8px 12px; border-bottom:1px solid #edf2f7; }
#colfPanel .colf-search input { width:100%; border:1.5px solid #cbd5e0; border-radius:6px; padding:6px 10px; font-size:.82rem; font-family:inherit; }
#colfPanel .colf-list { max-height:260px; overflow-y:auto; padding:4px 0; }
#colfPanel .colf-item { display:flex; align-items:center; gap:10px; padding:6px 12px; cursor:pointer; }
#colfPanel .colf-item:hover { background:#f0faf6; }
#colfPanel .colf-item input { width:auto; margin:0; }
#colfPanel .colf-item span { overflow:hidden; text-overflow:ellipsis; white-space:nowrap; }
#colfPanel .colf-empty, #colfPanel .colf-note { padding:8px 12px; font-size:.78rem; color:#a0aec0; }
/* แถวที่ประเภทการตัด = "รถตัดโรงงาน" แต่ผลตรวจ = "ไม่รับตัด" ไฮไลต์แดงเต็มทั้งแถว */
tbody tr.row-rejected,
tbody tr.row-rejected:hover,
tbody tr.row-rejected:nth-child(even) { background: #e53e3e !important; }
tbody tr.row-rejected td,
tbody tr.row-rejected td * {
  background-color: #e53e3e !important;
  color: #ffffff !important;
  border-color: #ffffff66 !important;
}
tbody tr.row-rejected td input[type="text"] { background-color: #c0392b !important; }
tbody tr.row-rejected td input[type="text"]:focus { outline: 2px solid #ffffff; }
tbody td { padding: 8px 12px; border-bottom: 1px solid #e2e8f0; vertical-align: middle; }
.std-ton-cell { text-align: right; color: #1a202c; font-weight: 600; font-size: .85rem; }
.skel { height: 14px; border-radius: 4px; background: linear-gradient(90deg,#e2e8f0 25%,#f0f4f8 50%,#e2e8f0 75%); background-size: 200% 100%; animation: shimmer 1.2s infinite; }
@keyframes shimmer { 0%{background-position:200% 0} 100%{background-position:-200% 0} }
.cg-summary-row { display: grid; gap: 14px; margin-bottom: 14px; }
.cg-summary-row.cols-3 { grid-template-columns: repeat(3, 1fr); }
.cg-summary-row.cols-4 { grid-template-columns: repeat(4, 1fr); }
.cg-summary-row.cols-5 { grid-template-columns: repeat(5, auto); overflow-x: auto; }
@media(max-width:768px){
  .cg-summary-row.cols-3, .cg-summary-row.cols-4 { grid-template-columns: repeat(2, 1fr); }
}
@media(max-width:480px){
  .cg-summary-row.cols-3, .cg-summary-row.cols-4 { grid-template-columns: 1fr; }
}
.cg-scard { background:white; border-radius:12px; padding:16px 20px; box-shadow:0 2px 8px rgba(0,0,0,.07); position:relative; overflow:hidden; }
.cg-scard::before { content:''; position:absolute; top:0; left:0; right:0; height:3px; background:linear-gradient(90deg,#1B6E9A,#89CFF0); }
.cg-scard-val { font-size:1.5rem; font-weight:700; color:#1a202c; line-height:1.2; }
.cg-scard-lbl { font-size:.78rem; color:#718096; font-weight:600; margin-top:4px; }
/* ★ [แก้ไข ข้อ 78] บล็อกสรุปหน้าจดแจ้งฯ (เป้า / จัดลำดับตัด / แปลงผิดปกติ) รวมเป็นชุดเดียว (เดิมประกาศซ้ำ 2 ชุดแล้วทับกัน)
   เป้า (ไร่) = ตัวอักษรน้ำเงิน / เป้า (ตัน) = ตัวอักษรส้ม : พื้นขาว
   จัดลำดับตัด (ไร่/ตัน) = ตัวอักษรดำ : พื้นแดงอ่อน (ยังไม่ถึง 100%) / เขียวอ่อน (ถึง 100%)
   แปลงผิดปกติ = พื้นส้มอ่อน ตัวอักษรแดง / บล็อกขยายตามข้อความ (nowrap) */
.cg-scard-decrai, .cg-scard-decreg, .cg-scard-decton, .cg-scard-decregton, .cg-scard-abn {
  border: 1.5px solid; border-radius: 14px; box-shadow: 0 2px 8px rgba(0,0,0,.08); padding: 18px 22px; min-width: 0; overflow: hidden;
}
.cg-scard-decrai::before, .cg-scard-decreg::before, .cg-scard-decton::before, .cg-scard-decregton::before, .cg-scard-abn::before { height: 4px; }
.cg-scard-decrai .cg-scard-val, .cg-scard-decrai .cg-scard-lbl,
.cg-scard-decreg .cg-scard-val, .cg-scard-decreg .cg-scard-lbl,
.cg-scard-decton .cg-scard-val, .cg-scard-decton .cg-scard-lbl,
.cg-scard-decregton .cg-scard-val, .cg-scard-decregton .cg-scard-lbl,
.cg-scard-abn .cg-scard-val, .cg-scard-abn .cg-scard-lbl { font-family: Tahoma, 'Segoe UI', sans-serif; }
.cg-scard-decrai .cg-scard-lbl, .cg-scard-decreg .cg-scard-lbl, .cg-scard-decton .cg-scard-lbl, .cg-scard-decregton .cg-scard-lbl, .cg-scard-abn .cg-scard-lbl {
  font-size: 1.5rem; font-weight: 700; line-height: 1.2; white-space: nowrap;
}
@media(max-width:1100px){
  .cg-scard-decrai .cg-scard-lbl, .cg-scard-decreg .cg-scard-lbl, .cg-scard-decton .cg-scard-lbl, .cg-scard-decregton .cg-scard-lbl, .cg-scard-abn .cg-scard-lbl { font-size: 1.1rem; }
}
.cg-summary-row.cols-5 .cg-scard { min-width: max-content; overflow: visible; }
.cg-summary-row.cols-5 .cg-scard .cg-scard-lbl { font-size: clamp(.9rem, 1.2vw, 1.5rem); overflow: visible; }
.cg-summary-row.cols-5 .cg-scard .cg-scard-val { font-size: clamp(1.1rem, 1.5vw, 1.5rem); white-space: nowrap; }
.cg-scard-decrai .cg-scard-val, .cg-scard-decrai .cg-scard-lbl { color: #1d4ed8; }
.cg-scard-decton .cg-scard-val, .cg-scard-decton .cg-scard-lbl { color: #ea580c; }
.cg-scard-decreg .cg-scard-val, .cg-scard-decreg .cg-scard-lbl,
.cg-scard-decregton .cg-scard-val, .cg-scard-decregton .cg-scard-lbl { color: #000000; }
.cg-scard-abn .cg-scard-val, .cg-scard-abn .cg-scard-lbl { color: #dc2626; }
.cg-scard-decrai { background:#ffffff; border-color:#bfdbfe; }
.cg-scard-decrai::before { background: linear-gradient(90deg,#1e3a8a,#60a5fa); }
.cg-scard-decton { background:#ffffff; border-color:#fed7aa; }
.cg-scard-decton::before { background: linear-gradient(90deg,#c2410c,#fb923c); }
.cg-scard-decreg, .cg-scard-decregton { background:#f1f5f9; border-color:#cbd5e1; }
.cg-scard-decreg::before, .cg-scard-decregton::before { background: linear-gradient(90deg,#64748b,#cbd5e1); }
.cg-scard-decreg.dec-low, .cg-scard-decregton.dec-low { background:#fef2f2; border-color:#fca5a5; }
.cg-scard-decreg.dec-low::before, .cg-scard-decregton.dec-low::before { background: linear-gradient(90deg,#dc2626,#fca5a5); }
.cg-scard-decreg.dec-ok, .cg-scard-decregton.dec-ok { background:#f0fdf4; border-color:#86efac; }
.cg-scard-decreg.dec-ok::before, .cg-scard-decregton.dec-ok::before { background: linear-gradient(90deg,#15803d,#4ade80); }
.cg-scard-abn { background:#fff7ed; border-color:#fdba74; }
.cg-scard-abn::before { background: linear-gradient(90deg,#dc2626,#fb923c); }
.cg-scard-pct { font-size:1rem; font-weight:700; margin-left:6px; opacity:.85; }
/* บล็อกรถตัด / แรงงาน / รถบรรทุก (แดชบอร์ด) */
.db-fleet-card { display:flex; align-items:center; justify-content:space-between; gap:12px; border-radius:12px; padding:18px 22px; box-shadow:0 2px 8px rgba(0,0,0,.07); border:1px solid transparent; font-size:1.5rem; font-weight:700; line-height:1.2; }
.db-fleet-lbl, .db-fleet-val { font-size:inherit; font-weight:700; white-space:nowrap; }
.db-fleet-val { font-variant-numeric: tabular-nums; }
.db-fleet-harvester { background:#f3e8ff; border-color:#e9d5ff; color:#6b21a8; }
.db-fleet-labor     { background:#ccfbf1; border-color:#99f6e4; color:#115e59; }
.db-fleet-truck     { background:#dbeafe; border-color:#bfdbfe; color:#1e40af; }
@media(max-width:1100px){ .db-fleet-card { font-size:1.15rem; } }
/* การ์ดสรุปหน้าข้อมูลกลุ่มตัด 3 หมวด: เหลือง / ฟ้า / ม่วง */
.cg-cards { grid-template-columns: repeat(3, 1fr); margin-bottom: 20px; }
@media(max-width:1100px){ .cg-cards { grid-template-columns: 1fr; } }
.cg-card .donut-legend { gap: 8px; }
.cg-card .dl-total { display:flex; flex-direction:column; gap:4px; }
.cg-foot-row { display:flex; justify-content:space-between; align-items:center; gap:10px; }
.cg-foot-row strong { white-space:nowrap; font-variant-numeric: tabular-nums; }
/* ตัวเลขชิดซ้ายต่อจากข้อความ เว้น ~2 วรรค */
.cg-card .dl-item .dl-txt { flex: 0 0 auto; }
.cg-card .dl-item strong { margin-left: .6em; white-space: nowrap; font-variant-numeric: tabular-nums; }
.cg-card .cg-foot-row { justify-content: flex-start; gap: .6em; }
.cg-card .cg-num-green { color: #15803d; }
/* % ต่อท้ายตัวเลข */
.cg-inline-pct { font-weight:700; font-size:.85em; margin-left:4px; opacity:.8; }
/* ชื่อรายการดำหนา */
.cg-card-boldlbl .dl-item .dl-txt { color:#1a202c; font-weight:700; }
.cg-card-1 { background:#fefce8; border:1.5px solid #fde68a; }
.cg-card-2 { background:#eff6ff; border:1.5px solid #bfdbfe; }
.cg-card-3 { background:#faf5ff; border:1.5px solid #e9d5ff; }
.cg-card-1::before { background: linear-gradient(90deg,#ca8a04,#fde047); }
.cg-card-2::before { background: linear-gradient(90deg,#1e3a8a,#60a5fa); }
.cg-card-3::before { background: linear-gradient(90deg,#6b21a8,#c084fc); }
.cg-card-1 .card-icon { background:#fef3c7; }
.cg-card-2 .card-icon { background:#dbeafe; }
.cg-card-3 .card-icon { background:#f3e8ff; }
.cg-card-1 .card-divider, .cg-card-2 .card-divider, .cg-card-3 .card-divider { background: rgba(0,0,0,.06); }
.cg-filter { background:white; border-radius:10px; padding:14px 16px; box-shadow:0 1px 4px rgba(0,0,0,.08); display:flex; gap:12px; flex-wrap:wrap; align-items:flex-end; margin-bottom:16px; }
.cg-table-box { background:white; border-radius:10px; box-shadow:0 1px 4px rgba(0,0,0,.08); overflow:hidden; contain: layout style paint; }
.cg-table-wrapper { overflow-x:auto; max-height:520px; overflow-y:auto; }
.cg-table { width:100%; border-collapse:separate; border-spacing:0; font-size:.85rem; }
.cg-table thead th { background:#1B6E9A; color:white; padding:11px 12px; text-align:center; position:sticky; top:0; z-index:2; white-space:nowrap; }
.cg-table tbody tr:nth-child(even) { background:#f7fafc; }
.cg-table tbody tr:hover { background:#eaf7fd; }
.cg-table tbody td { padding:8px 12px; border-bottom:1px solid #e2e8f0; vertical-align:middle; text-align:center; }
.cg-table tbody td.num { font-variant-numeric:tabular-nums; }
/* ล็อคคอลัมน์ซ้าย 3 คอลัมน์แรก (เขต, Code กลุ่มตัด, ชื่อกลุ่มตัด) */
.cg-table th:nth-child(-n+3), .cg-table td:nth-child(-n+3) { position: sticky; z-index: 3; }
.cg-table thead th:nth-child(-n+3) { z-index: 4; }
.cg-table th:nth-child(1), .cg-table td:nth-child(1) { left: 0px;   width: 55px;  min-width: 55px; }
.cg-table th:nth-child(2), .cg-table td:nth-child(2) { left: 55px;  width: 110px; min-width: 110px; }
.cg-table th:nth-child(3), .cg-table td:nth-child(3) { left: 165px; width: 240px; min-width: 240px; max-width: 240px; }
/* ชื่อกลุ่มตัดยาวเกินให้ตัดเป็น ... ไม่ล้นทับคอลัมน์ยี่ห้อ */
.cg-table td:nth-child(3) { overflow: hidden; text-overflow: ellipsis; white-space: nowrap; box-shadow: 2px 0 0 #e2e8f0; }
.cg-table thead th:nth-child(3) { box-shadow: 2px 0 0 rgba(255,255,255,.35); }
.cg-table td:nth-child(4) { white-space: nowrap; }
.cg-table tbody td:nth-child(-n+3) { background-color: #fff; }
.cg-table tbody tr:nth-child(even) td:nth-child(-n+3) { background-color: #f7fafc; }
.cg-table tbody tr:hover td:nth-child(-n+3) { background-color: #eaf7fd; }
/* จดแจ้งแผนตัดธันวาคม (ไร่) ไม่เป็น 0 = เขียวทั้งแถว (รวมคอลัมน์ที่ล็อคซ้าย) */
.cg-table tbody tr.row-decreg,
.cg-table tbody tr.row-decreg:hover,
.cg-table tbody tr.row-decreg:nth-child(even) { background: #c6f6d5 !important; }
.cg-table tbody tr.row-decreg td,
.cg-table tbody tr.row-decreg td:nth-child(-n+3),
.cg-table tbody tr.row-decreg:nth-child(even) td:nth-child(-n+3),
.cg-table tbody tr.row-decreg:hover td:nth-child(-n+3) { background-color: #c6f6d5 !important; }
.cg-table tbody tr.row-decreg:hover td { background-color: #b2f0c5 !important; }
.cg-badge-zone { color:#1a202c; font-weight:700; }
/* จดแจ้งแผนตัดธันวาคม เทียบกับแผนตัดธันวาคม (ใช้ทั้งไร่และตัน) */
.cg-dec-under { color:#e53e3e; font-weight:700; }
.cg-dec-ok    { color:#2d6a4f; font-weight:700; }
.cg-status-pills { display: flex; gap: 8px; flex-wrap: wrap; align-items: center; }
.cg-status-pill { display: inline-flex; align-items: center; gap: 5px; padding: 5px 14px; border-radius: 20px; border: 2px solid #e2e8f0; font-size: .82rem; font-weight: 700; cursor: pointer; background: #f1f5f9; color: #64748b; transition: all .15s; font-family: inherit; }
.cg-status-pill:hover { filter: brightness(.95); }
.cg-status-pill.pill-all.active     { background: #1b4332; color: #fff; border-color: #1b4332; }
.cg-status-pill.pill-factory.active { background: #f3e8ff; color: #6b21a8; border-color: #a855f7; }
.cg-status-pill.pill-cluster.active { background: #e6f4ea; color: #1b4332; border-color: #2d6a4f; }
.cg-status-pill.pill-contract.active{ background: #fff7ed; color: #9a3412; border-color: #f97316; }
.cg-status-pill.pill-labor.active   { background: #fef9c3; color: #854d0e; border-color: #eab308; }
.cg-status-pill.pill-landzone1.active { background: #fce7f3; color: #9d174d; border-color: #ec4899; }
.cg-status-pill.pill-landzone8.active { background: #dbeafe; color: #1e3a8a; border-color: #2563eb; }
.cg-pill-dot { width: 8px; height: 8px; border-radius: 50%; flex-shrink: 0; }
.save-confirm-overlay { position: fixed; top:0; left:0; right:0; bottom:0; background: rgba(0,0,0,.55); display:flex; align-items:center; justify-content:center; z-index: 10000; padding: 16px; }
.save-confirm-box { background:#fff; border-radius:14px; padding: 20px 22px; width: 100%; max-width: 900px; max-height: 82vh; display:flex; flex-direction:column; box-shadow: 0 12px 40px rgba(0,0,0,.3); }
.save-confirm-title { font-size: 1.05rem; font-weight:700; color:#1b4332; margin-bottom: 4px; display:flex; align-items:center; gap:8px; }
.save-confirm-sub { font-size:.85rem; color:#718096; margin-bottom: 14px; }
.save-confirm-scroll { overflow-y:auto; border:1px solid #e2e8f0; border-radius:10px; margin-bottom: 16px; }
.save-confirm-table { width:100%; border-collapse:collapse; font-size:.85rem; }
.save-confirm-table thead th { background:#2d6a4f; color:#fff; padding:9px 12px; text-align:left; position:sticky; top:0; white-space:nowrap; }
.save-confirm-table tbody td { padding:7px 12px; border-bottom:1px solid #edf2f7; }
.save-confirm-table tbody tr:nth-child(even) { background:#f7fafc; }
.save-confirm-actions { display:flex; justify-content:flex-end; gap:10px; }
.btn-save-cancel { background:#e2e8f0; color:#4a5568; border:none; border-radius:8px; padding:10px 22px; font-weight:700; font-size:.9rem; cursor:pointer; font-family:inherit; }
.btn-save-cancel:hover { background:#cbd5e0; }
.btn-save-confirm { background:#2d6a4f; color:#fff; border:none; border-radius:8px; padding:10px 22px; font-weight:700; font-size:.9rem; cursor:pointer; font-family:inherit; }
.btn-save-confirm:hover { background:#1b4332; }
@media(max-width:768px){
  .cards      { grid-template-columns: 1fr; }
  .filters    { flex-direction: column; align-items: stretch; }
  .cg-filter  { flex-direction: column; align-items: stretch; }
}
/* ตัวกรองรายคอลัมน์ของตารางข้อมูลกลุ่มตัด (เหมือน Excel + กรองสี) */
.cgf-btn { background:transparent; border:none; color:#fff; cursor:pointer; font-size:.7rem; padding:2px 5px; margin-left:4px; border-radius:4px; font-family:inherit; line-height:1; }
.cgf-btn:hover { background:rgba(255,255,255,.25); }
.cgf-btn.active { background:#fff; color:#c2410c; }
#cgfPanel { position:fixed; z-index:3000; background:#fff; border:1px solid #e2e8f0; border-radius:8px; box-shadow:0 6px 20px rgba(0,0,0,.2); width:270px; color:#1a202c; font-size:.85rem; text-align:left; }
#cgfPanel .cgf-head { display:flex; justify-content:space-between; gap:16px; padding:10px 12px; font-size:.78rem; font-weight:600; border-bottom:1px solid #edf2f7; }
#cgfPanel .cgf-head a { color:#0f8b78; cursor:pointer; user-select:none; }
#cgfPanel .cgf-head a:hover { text-decoration:underline; }
#cgfPanel .cgf-search { padding:8px 12px; border-bottom:1px solid #edf2f7; }
#cgfPanel .cgf-search input { width:100%; border:1.5px solid #cbd5e0; border-radius:6px; padding:6px 10px; font-size:.82rem; font-family:inherit; }
#cgfPanel .cgf-list { max-height:320px; overflow-y:auto; padding:4px 0; }
#cgfPanel .cgf-sec { padding:6px 12px 2px; font-size:.74rem; font-weight:700; color:#718096; border-top:1px solid #edf2f7; }
#cgfPanel .cgf-sec:first-child { border-top:none; }
#cgfPanel .cgf-item { display:flex; align-items:center; gap:10px; padding:6px 12px; cursor:pointer; }
#cgfPanel .cgf-item:hover { background:#f0faf6; }
#cgfPanel .cgf-item input { width:auto; margin:0; }
#cgfPanel .cgf-item span.t { overflow:hidden; text-overflow:ellipsis; white-space:nowrap; }
#cgfPanel .cgf-item small { margin-left:auto; color:#a0aec0; }
#cgfPanel .cgf-sw { display:inline-flex; align-items:center; justify-content:center; width:20px; height:20px; border:1px solid #cbd5e0; border-radius:4px; font-weight:800; font-size:.8rem; flex-shrink:0; }
#cgfPanel .cgf-empty, #cgfPanel .cgf-note { padding:8px 12px; font-size:.78rem; color:#a0aec0; }

/* ===== แผนที่ ===== */
.map-panel { background:#fff; border-radius:14px; box-shadow:0 2px 8px rgba(0,0,0,.07); padding:18px 20px; margin-bottom:16px; }
.map-panel-head { display:flex; justify-content:space-between; align-items:baseline; gap:10px; flex-wrap:wrap; margin-bottom:14px; }
.map-panel-head h2 { font-size:1.3rem; font-weight:800; color:#1a202c; }
.map-panel-head span { font-size:.78rem; color:#718096; font-weight:600; }
.map-filters { display:grid; grid-template-columns:repeat(6,1fr); gap:14px; background:#fff; border-radius:12px; box-shadow:0 1px 6px rgba(0,0,0,.08); padding:16px; margin-bottom:16px; }
.map-filters .filter-group input { width:100%; min-width:0; border:1px solid #e2e8f0; border-radius:8px; padding:8px 10px; font-size:.88rem; background:#fff; font-family:inherit; }
.map-filters .filter-group label { font-size:.74rem; }
.map-filter-btns { grid-column:1 / -1; display:flex; gap:10px; flex-wrap:wrap; }
.map-btn { border:none; border-radius:20px; padding:8px 18px; font-weight:700; font-size:.85rem; cursor:pointer; color:#fff; font-family:inherit; }
.map-btn.blue  { background:#3b82f6; } .map-btn.blue:hover  { background:#2563eb; }
.map-btn.green { background:#2f9e6e; border-radius:8px; } .map-btn.green:hover { background:#237a54; }
.map-stats { display:grid; grid-template-columns:repeat(4,1fr); gap:14px; margin-bottom:16px; }
.map-stat { background:#fff; border-radius:14px; padding:16px 20px; box-shadow:0 1px 6px rgba(0,0,0,.08); }
.map-stat-top { display:flex; align-items:center; gap:10px; font-size:.8rem; color:#718096; font-weight:600; margin-bottom:10px; }
.map-stat-ic { width:30px; height:30px; border-radius:8px; display:flex; align-items:center; justify-content:center; font-size:.95rem; }
.map-stat-val { font-size:1.75rem; font-weight:800; color:#1a202c; line-height:1.1; font-variant-numeric:tabular-nums; }
.map-stat-val small { font-size:.8rem; font-weight:600; color:#718096; margin-left:4px; }
.map-main { display:grid; grid-template-columns:minmax(0,1fr) max-content; gap:16px; align-items:start; }
.map-card, .map-side { background:#fff; border-radius:14px; box-shadow:0 2px 8px rgba(0,0,0,.07); padding:16px 18px; }
.map-side { max-width:760px; }
.map-side { max-width:760px; }
.map-card h3 { font-size:1rem; font-weight:800; color:#1a202c; margin-bottom:10px; }
.map-card-bar { display:flex; justify-content:space-between; align-items:center; margin-bottom:10px; gap:8px; }
.map-base-pills { display:flex; gap:8px; }
.map-base-pill { border:1px solid #e2e8f0; background:#fff; border-radius:7px; padding:6px 14px; font-size:.8rem; font-weight:700; color:#4a5568; cursor:pointer; font-family:inherit; }
.map-base-pill.active { background:#2f9e6e; border-color:#2f9e6e; color:#fff; }
.map-btn-expand { border:1px solid #e2e8f0; background:#fff; border-radius:7px; padding:6px 14px; font-size:.8rem; font-weight:700; color:#4a5568; cursor:pointer; font-family:inherit; }
#mapCanvas { width:100%; height:480px; border-radius:10px; background:#d9e2d0; }
.map-card.map-expanded { position:fixed; top:0; left:0; right:0; bottom:0; z-index:900; border-radius:0; overflow:auto; }
.map-card.map-expanded #mapCanvas { height:calc(100vh - 150px); }
.map-legend { display:flex; gap:16px; flex-wrap:wrap; align-items:center; margin-top:10px; font-size:.76rem; color:#4a5568; font-weight:600; }
.map-legend i { display:inline-block; width:10px; height:10px; border-radius:50%; margin-right:5px; vertical-align:middle; }
.map-legend .lg-ord { display:inline-flex; width:16px; height:16px; border-radius:50%; background:#f39c32; color:#fff; font-size:.62rem; font-weight:800; align-items:center; justify-content:center; margin-right:5px; }
.map-legend .lg-line { display:inline-block; width:22px; height:3px; background:#e53e3e; margin-right:5px; vertical-align:middle; }
.map-note { margin-top:6px; font-size:.76rem; color:#718096; }
.map-ord-pin span { display:flex; width:26px; height:26px; border-radius:50%; background:#f28c28; color:#fff; font-weight:800; font-size:.8rem; align-items:center; justify-content:center; border:2px solid #fff; box-shadow:0 1px 5px rgba(0,0,0,.5); }
.map-ord-pin.ord-tor span  { background:#f28c28; }
.map-ord-pin.ord-ton span  { background:#5bb89a; }
.map-ord-pin.ord-plai span { background:#1e2d55; }
.map-tabs { display:flex; gap:18px; border-bottom:1px solid #e2e8f0; margin-bottom:8px; }
.map-tab { background:none; border:none; border-bottom:2px solid transparent; padding:6px 2px 8px; font-size:.9rem; font-weight:700; color:#4a5568; cursor:pointer; font-family:inherit; }
.map-tab.active { color:#1a202c; border-bottom-color:#2f9e6e; }
#mapSideInfo { font-size:.76rem; color:#718096; margin:0 0 6px; }
#mapSideScroll { max-height:560px; overflow-y:auto; overflow-x:auto; }
.map-mem { display:flex; justify-content:space-between; align-items:center; gap:10px; padding:11px 6px; border-bottom:1px solid #edf2f7; cursor:pointer; }
.map-mem:hover { background:#f7fafc; }
.map-mem-name { font-size:.9rem; font-weight:700; color:#1a202c; }
.map-mem-sub { font-size:.72rem; color:#718096; margin-top:3px; }
.map-mem-r { text-align:right; flex-shrink:0; }
.map-mem-ton { font-size:.9rem; font-weight:800; color:#1a202c; margin-bottom:4px; }
.map-mem-badge { display:inline-block; background:#e6f7ee; color:#1b7a4b; border:1px solid #b7e4c7; border-radius:12px; padding:2px 12px; font-size:.72rem; font-weight:700; }
.map-tbl { width:100%; border-collapse:collapse; font-size:.8rem; }
.map-tbl thead th { background:#f3f6f9; color:#4a5568; font-size:.74rem; text-align:center; padding:9px 5px; position:sticky; top:0; z-index:2; white-space:nowrap; }
.map-tbl tbody td { padding:9px 5px; border-bottom:1px solid #edf2f7; text-align:left; background:transparent; white-space:nowrap; }
.map-tbl tbody tr { cursor:pointer; background:#fff; }
.map-tbl tbody tr:nth-child(even) { background:#fff; }
.map-tbl tbody tr:hover { background:#f0faf6; }
.map-tbl td.c { text-align:center; font-weight:700; } .map-tbl td.r { text-align:right; }
.map-ord-input { width:44px; border:1px solid #e2e8f0; border-radius:6px; padding:4px 4px; font-size:.8rem; font-weight:700; text-align:center; font-family:inherit; background:#fff; }
.map-ord-input:focus { outline:none; border-color:#2f9e6e; box-shadow:0 0 0 2px rgba(47,158,110,.2); }
.map-ord-input:disabled { background:#e2e8f0; color:#718096; cursor:not-allowed; }
.mp { min-width:250px; font-size:.8rem; }
.mp-title { font-size:.98rem; font-weight:800; color:#1a202c; margin-bottom:8px; }
.mp-row { display:flex; justify-content:space-between; gap:12px; padding:2px 0; color:#718096; }
.mp-row b { color:#1a202c; font-weight:700; text-align:right; }
.mp-hint { font-size:.7rem; color:#a0aec0; margin:2px 0 6px; }
.mp-edit { border:1px solid #bfd4fb; background:#eaf2ff; color:#2563eb; border-radius:14px; padding:2px 10px; font-size:.72rem; font-weight:700; cursor:pointer; font-family:inherit; }
.mp-edit:disabled { border-color:#e2e8f0; background:#f1f5f9; color:#a0aec0; cursor:not-allowed; }
.mp-box { margin:10px 0 8px; padding:10px 12px; border:1px solid #e2e8f0; border-radius:8px; background:#f8fafc; }
.mp-box-lbl { font-size:.74rem; color:#718096; font-weight:600; margin-bottom:6px; }
.mp-box-lbl.c { text-align:center; }
.mp-input { width:100%; border:1.5px solid #cbd5e0; border-radius:8px; padding:8px 10px; font-size:.9rem; font-family:inherit; background:#fff; }
.mp-input:focus { outline:none; border-color:#3b82f6; box-shadow:0 0 0 2px rgba(59,130,246,.2); }
.mp-warn { margin-top:8px; background:#fff4e0; color:#b45309; border-radius:6px; padding:6px 10px; font-size:.74rem; font-weight:700; }
.mp-note { margin-top:8px; font-size:.72rem; color:#718096; line-height:1.4; }
.mp-note.c { text-align:center; }
.mp-change { text-align:center; font-size:1.5rem; font-weight:800; color:#1a202c; margin:6px 0 4px; }
.mp-actions { display:flex; justify-content:flex-end; gap:8px; margin-top:10px; }
.mp-btn { border:1px solid #e2e8f0; background:#fff; color:#4a5568; border-radius:7px; padding:6px 16px; font-size:.8rem; font-weight:700; cursor:pointer; font-family:inherit; }
.mp-btn.primary { background:#2f9e6e; border-color:#2f9e6e; color:#fff; }
.mp-btn:disabled { opacity:.6; cursor:not-allowed; }
/* ตัวกรองแบบเลือกได้หลายรายการ */
.ms { position:relative; }
.ms-btn { width:100%; display:flex; justify-content:space-between; align-items:center; gap:6px; border:1px solid #e2e8f0; border-radius:8px; padding:8px 10px; font-size:.88rem; background:#fff; font-family:inherit; cursor:pointer; text-align:left; color:#1a202c; }
.ms-btn .ms-txt { overflow:hidden; text-overflow:ellipsis; white-space:nowrap; }
.ms-btn .ms-arrow { font-size:.7rem; color:#718096; flex-shrink:0; }
.ms-panel { position:absolute; top:100%; left:0; margin-top:4px; min-width:100%; width:max-content; max-width:320px; background:#fff; border:1px solid #e2e8f0; border-radius:8px; box-shadow:0 6px 20px rgba(0,0,0,.15); z-index:2000; }
.ms-head { display:flex; justify-content:space-between; gap:16px; padding:10px 12px; font-size:.78rem; font-weight:600; border-bottom:1px solid #edf2f7; }
.ms-head a { color:#0f8b78; cursor:pointer; user-select:none; }
.ms-head a:hover { text-decoration:underline; }
.ms-list { max-height:240px; overflow-y:auto; padding:4px 0; }
.ms-item { display:flex; align-items:center; gap:10px; padding:8px 12px; font-size:.85rem; cursor:pointer; color:#1a202c; }
.ms-item:hover { background:#f0faf6; }
.map-filters .filter-group .ms-item input { width:auto; min-width:0; padding:0; border:none; margin:0; cursor:pointer; }
.ms-empty { padding:10px 12px; font-size:.8rem; color:#a0aec0; }
.ms-search { padding:8px 12px; border-bottom:1px solid #edf2f7; }
.map-filters .filter-group .ms-search input { width:100%; border:1.5px solid #cbd5e0; border-radius:6px; padding:6px 10px; font-size:.82rem; font-family:inherit; background:#fff; }
.map-filters .filter-group .ms-search input:focus { outline:none; border-color:#0f8b78; }
@media(max-width:1100px){ .map-filters { grid-template-columns:repeat(3,1fr); } .map-stats { grid-template-columns:repeat(2,1fr); } .map-main { grid-template-columns:1fr; } }
@media(max-width:560px){ .map-filters { grid-template-columns:1fr 1fr; } #mapCanvas { height:380px; } }
</style>
</head>
<body>
<div class="app-shell">
  <aside class="sidebar">
    <div class="sidebar-brand">
      <div class="sidebar-brand-icon">🌿</div>
      <div class="sidebar-brand-text">
        <strong>แผนตัดเดือนธันวาคม</strong>
        <span>ปี 69/70</span>
      </div>
    </div>
    <div class="sidebar-section-title">เมนูหลัก</div>
    <nav class="sidebar-menu">
      <button class="menu-item active" data-view="dashboard" onclick="switchView('dashboard')">
        <span class="menu-icon">📊</span><span>แดชบอร์ด</span>
      </button>
      <button class="menu-item" data-view="cutgroup" onclick="switchView('cutgroup')">
        <span class="menu-icon">🚜</span><span>ข้อมูลกลุ่มตัด</span>
      </button>
      <button class="menu-item" data-view="table" onclick="switchView('table')">
        <span class="menu-icon">📋</span><span>จดแจ้งลำดับแผนตัดเดือนธันวาคม</span>
      </button>
      <button class="menu-item refresh-btn" onclick="refreshData()">
        <span class="menu-icon">🔄</span><span>รีเฟรชข้อมูล</span>
      </button>
      <button class="menu-item" data-view="map" onclick="switchView('map')">
        <span class="menu-icon">🗺️</span><span>แผนที่</span>
      </button>
    </nav>
  </aside>

  <div class="main-content">
    <header id="pageHeader">
      <h1 id="pageTitle">🌿 จัดคิวอ้อยเข้าหีบ ปี 69/70</h1>
      <small id="updateTime">กำลังโหลดข้อมูล...</small>
    </header>
    <div id="toast"></div>
    <div id="colfPanel" style="display:none"></div>
    <div id="cgfPanel" style="display:none"></div>
    <div class="container">

      <!-- ==================== DASHBOARD ==================== -->
      <div id="viewDashboard">
        <div class="filters" style="margin-bottom:16px;">
          <div class="filter-group"><label>เขต / โซน</label>
            <select id="dbFilterZone" onchange="applyDbFilters()"><option value="all">ทั้งหมด (เขต)</option></select></div>
          <div class="filter-group"><label>ชื่อนักเกษตร</label>
            <select id="dbFilterFarmer" onchange="applyDbFilters()"><option value="all">ทั้งหมด (นักเกษตร)</option></select></div>
          <div class="filter-group"><label>ประเภทการตัด</label>
            <select id="dbFilterCutType" onchange="applyDbFilters()"><option value="all">ทั้งหมด (ประเภทการตัด)</option></select></div>
          <div class="filter-group"><label>ค้นหา ชื่อกลุ่มตัด</label>
            <input type="text" id="dbFilterCutGroup" placeholder="🔍 พิมพ์เพื่อค้นหา..." oninput="applyDbFilters()"
              style="border:1px solid #e2e8f0;border-radius:6px;padding:7px 10px;font-size:.9rem;background:#f7fafc;font-family:inherit;min-width:200px;"></div>
          <div class="filter-group"><label>แสดงกราฟ</label>
            <select id="dbChartMode" onchange="setDbChartMode(this.value)">
              <option value="ton">ตัน STD</option>
              <option value="korat">ครบุรี</option>
              <option value="sikhio">สีคิ้ว</option>
              <option value="plot">แปลง</option>
              <option value="rai">ไร่</option>
              <option value="order">จัดลำดับตัด(แปลง</option>
              <option value="group">จัดลำดับตัด(กลุ่ม)</option>
            </select>
          </div>
          <button class="btn-reset" onclick="resetDbFilters()">🔄 รีเซ็ต</button>
        </div>

        <div class="cards">
          <div class="card">
           <div class="card-top"><div class="card-icon">🗺️</div><div class="card-label">พื้นที่ KAT (แปลง)</div></div>
            <div class="card-divider"></div>
            <div class="donut-card-body">
              <div class="donut-wrap"><canvas id="donutPlotChart" width="110" height="110"></canvas>
                <div class="donut-center"><div class="donut-center-val" id="donutPlotPct">-</div><div class="donut-center-lbl">จัดแล้ว</div></div></div>
              <div class="donut-legend dl-big">
                <div class="dl-item"><span class="dl-dot" style="background:#1b4332"></span><span class="dl-txt">จัดแล้ว</span><strong id="donutPlotAssigned">-</strong></div>
                <div class="dl-item"><span class="dl-dot" style="background:#b7e4c7"></span><span class="dl-txt" style="color:#e53e3e;font-weight:700">ยังไม่จัด</span><strong id="donutPlotUnassigned" style="color:#e53e3e">-</strong></div>
                <div class="dl-total">ทั้งหมด <strong id="donutPlotTotal">-</strong></div>
              </div>
            </div>
          </div>
          <div class="card orange">
            <div class="card-top"><div class="card-icon">✅</div><div class="card-label">พื้นที่ KAT (ไร่)</div></div>
            <div class="card-divider"></div>
            <div class="donut-card-body">
              <div class="donut-wrap"><canvas id="donutAreaChart" width="110" height="110"></canvas>
                <div class="donut-center"><div class="donut-center-val" id="donutAreaPct">-</div><div class="donut-center-lbl">จัดแล้ว</div></div></div>
              <div class="donut-legend dl-big">
                <div class="dl-item"><span class="dl-dot" style="background:#c2410c"></span><span class="dl-txt">จัดแล้ว</span><strong id="donutAreaAssigned">-</strong></div>
                <div class="dl-item"><span class="dl-dot" style="background:#fed7aa"></span><span class="dl-txt" style="color:#e53e3e;font-weight:700">ยังไม่จัด</span><strong id="donutAreaUnassigned" style="color:#e53e3e">-</strong></div>
                <div class="dl-total">ทั้งหมด <strong id="donutAreaTotal">-</strong></div>
              </div>
            </div>
          </div>
          <div class="card blue">
            <div class="card-top"><div class="card-icon">👨‍🌾</div><div class="card-label">ชาวไร่ (ราย)</div></div>
            <div class="card-divider"></div>
            <div class="donut-card-body">
              <div class="donut-wrap"><canvas id="donutOwnerChart" width="110" height="110"></canvas>
                <div class="donut-center"><div class="donut-center-val" id="donutOwnerPct">-</div><div class="donut-center-lbl">จัดครบแล้ว</div></div></div>
              <div class="donut-legend dl-big">
                <div class="dl-item"><span class="dl-dot" style="background:#1e3a8a"></span><span class="dl-txt">จัดครบแล้ว</span><strong id="donutOwnerFull">-</strong></div>
                <div class="dl-item"><span class="dl-dot" style="background:#60a5fa"></span><span class="dl-txt" style="color:#b45309;font-weight:700">จัดบางส่วน</span><strong id="donutOwnerPartial" style="color:#b45309">-</strong></div>
                <div class="dl-item"><span class="dl-dot" style="background:#bfdbfe"></span><span class="dl-txt" style="color:#e53e3e;font-weight:700">ยังไม่จัด</span><strong id="donutOwnerNone" style="color:#e53e3e">-</strong></div>
                <div class="dl-total">ทั้งหมด <strong id="donutOwnerTotal">-</strong></div>
              </div>
            </div>
          </div>
          <div class="card purple">
            <div class="card-top"><div class="card-icon">⚖️</div><div class="card-label">ปริมาณอ้อย (ตัน STD)</div></div>
            <div class="card-divider"></div>
            <div class="donut-card-body">
              <div class="donut-wrap"><canvas id="donutCutTypeChart" width="110" height="110"></canvas>
                <div class="donut-center"><div class="donut-center-val" id="donutCutTypePct">-</div><div class="donut-center-lbl">จัดแล้ว</div></div></div>
              <div class="donut-legend dl-big">
                <div class="dl-item"><span class="dl-dot" style="background:#6b21a8"></span><span class="dl-txt">จัดแล้ว</span><strong id="donutCutTypeAssigned">-</strong></div>
                <div class="dl-item"><span class="dl-dot" style="background:#e9d5ff"></span><span class="dl-txt" style="color:#e53e3e;font-weight:700">ยังไม่จัด</span><strong id="donutCutTypeUnassigned" style="color:#e53e3e">-</strong></div>
                <div class="dl-total">ทั้งหมด <strong id="donutCutTypeTotal">-</strong></div>
              </div>
            </div>
          </div>
        </div>

        <!-- บล็อกรถตัดอ้อย / แรงงาน / รถบรรทุก (ผลรวมจากชีต Code เฉพาะกลุ่มที่จดแจ้ง (ตัน) > 0) -->
        <div class="cg-summary-row cols-3" style="margin-bottom:20px">
          <div class="db-fleet-card db-fleet-harvester"><span class="db-fleet-lbl">🚜 รถตัดอ้อย (คัน)</span><span class="db-fleet-val" id="dbTotalHarvester">-</span></div>
          <div class="db-fleet-card db-fleet-labor"><span class="db-fleet-lbl">👷 แรงงาน (คน)</span><span class="db-fleet-val" id="dbTotalLabor">-</span></div>
          <div class="db-fleet-card db-fleet-truck"><span class="db-fleet-lbl">🚚 รถบรรทุก (คัน)</span><span class="db-fleet-val" id="dbTotalTruck">-</span></div>
        </div>

        <div class="combined-chart-box blue" id="chartBoxTon">
          <div class="combined-chart-box-title"> 📊 ประเภทการตัด (ตัน STD)</div>
          <div class="combined-chart-inner">
            <div class="chart-inner-panel">
              <div class="pie-body">
                <div class="pie-canvas-wrap"><canvas id="pieTonChart"></canvas></div>
                <div class="pie-legend" id="pieTonLegend"></div>
              </div>
            </div>
            <div class="chart-inner-panel">
              <canvas id="barTonChart"></canvas>
            </div>
          </div>
        </div>

        <div class="combined-chart-box" id="chartBoxPlot" style="display:none">
          <div class="combined-chart-box-title">📊 ประเภทการตัด (แปลง)</div>
          <div class="combined-chart-inner">
            <div class="chart-inner-panel">
              <div class="pie-body">
                <div class="pie-canvas-wrap"><canvas id="pieChart"></canvas></div>
                <div class="pie-legend" id="pieLegend"></div>
              </div>
            </div>
            <div class="chart-inner-panel">
              <canvas id="barChart"></canvas>
            </div>
          </div>
        </div>

        <div class="combined-chart-box orange" id="chartBoxRai" style="display:none">
          <div class="combined-chart-box-title">📊 ประเภทการตัด (ไร่)</div>
          <div class="combined-chart-inner">
            <div class="chart-inner-panel">
              <div class="pie-body">
                <div class="pie-canvas-wrap"><canvas id="pieRaiChart"></canvas></div>
                <div class="pie-legend" id="pieRaiLegend"></div>
              </div>
            </div>
            <div class="chart-inner-panel">
              <canvas id="barRaiChart"></canvas>
            </div>
          </div>
        </div>

        <!-- จัดลำดับตัด (แปลงที่ถึงเกณฑ์ คอลัมน์ T) -->
        <div class="combined-chart-box violet" id="chartBoxOrder" style="display:none">
          <div class="combined-chart-box-title">📊 จัดลำดับตัด (แปลงที่ถึงเกณฑ์)</div>
          <div class="cg-summary-row cols-4">
            <div class="cg-scard ord-c1"><div class="cg-scard-val" id="ordMetPlots">-</div><div class="cg-scard-lbl">แปลงที่ถึงเกณฑ์ (แปลง)</div></div>
            <div class="cg-scard ord-c2"><div class="cg-scard-val" id="ordMetArea">-</div><div class="cg-scard-lbl">พื้นที่ถึงเกณฑ์ (ไร่)</div></div>
            <div class="cg-scard ord-c3"><div class="cg-scard-val" id="ordDonePlots">-</div><div class="cg-scard-lbl">จัดลำดับแล้ว (แปลง)</div></div>
            <div class="cg-scard ord-c4"><div class="cg-scard-val" id="ordTodoPlots">-</div><div class="cg-scard-lbl">ยังไม่จัดลำดับ (แปลง)</div></div>
          </div>
          <div class="combined-chart-inner">
            <div class="chart-inner-panel">
              <div class="pie-body">
                <div class="pie-canvas-wrap"><canvas id="pieOrderChart"></canvas></div>
                <div class="pie-legend" id="pieOrderLegend"></div>
              </div>
            </div>
            <div class="chart-inner-panel">
              <canvas id="barOrderChart"></canvas>
            </div>
          </div>
        </div>

        <!-- แผนตัดเดือนธันวาคม (กลุ่มตัด) -->
        <div class="combined-chart-box violet" id="chartBoxGroup" style="display:none">
          <div class="combined-chart-box-title">📊 แผนตัดเดือนธันวาคม (กลุ่มตัด)</div>
          <div class="cg-summary-row cols-3">
            <div class="cg-scard ord-c1"><div class="cg-scard-val" id="grpTotal">-</div><div class="cg-scard-lbl">ทั้งหมด (กลุ่ม)</div></div>
            <div class="cg-scard ord-c3"><div class="cg-scard-val" id="grpDone">-</div><div class="cg-scard-lbl">จัดแล้ว (กลุ่ม)</div></div>
            <div class="cg-scard ord-c2"><div class="cg-scard-val" id="grpTodo" style="color:#e53e3e">-</div><div class="cg-scard-lbl">ยังไม่จัด (กลุ่ม)</div></div>
          </div>
          <div class="combined-chart-inner">
            <div class="chart-inner-panel">
              <div class="pie-body">
                <div class="pie-canvas-wrap"><canvas id="pieGroupChart"></canvas></div>
                <div class="pie-legend" id="pieGroupLegend"></div>
              </div>
            </div>
            <div class="chart-inner-panel">
              <canvas id="barGroupChart"></canvas>
            </div>
          </div>
        </div>

        <div class="combined-chart-box amber" id="chartBoxKorat" style="display:none">
          <div class="combined-chart-box-title">🏭 ตัน STD (โรงงานครบุรี)</div>
          <div class="combined-chart-inner">
            <div class="chart-inner-panel">
              <div class="pie-body">
                <div class="pie-canvas-wrap"><canvas id="pieTonKoratChart"></canvas></div>
                <div class="pie-legend" id="pieTonKoratLegend"></div>
              </div>
            </div>
            <div class="chart-inner-panel">
              <canvas id="barTonKoratChart"></canvas>
            </div>
          </div>
        </div>

        <div class="combined-chart-box teal" id="chartBoxSikhio" style="display:none">
          <div class="combined-chart-box-title">🏭 ตัน STD (โรงงานสีคิ้ว)</div>
          <div class="combined-chart-inner">
            <div class="chart-inner-panel">
              <div class="pie-body">
                <div class="pie-canvas-wrap"><canvas id="pieTonSikhioChart"></canvas></div>
                <div class="pie-legend" id="pieTonSikhioLegend"></div>
              </div>
            </div>
            <div class="chart-inner-panel">
              <canvas id="barTonSikhioChart"></canvas>
            </div>
          </div>
        </div>
      </div>
      <!-- ==================== /DASHBOARD ==================== -->

      <!-- ==================== TABLE ==================== -->
      <div id="viewTable" style="display:none">
        <div class="filters">
          <div class="filter-group"><label>เขต / โซน</label>
            <select id="filterZone" onchange="applyFilters()"><option value="all">ทั้งหมด (เขต)</option></select></div>
          <div class="filter-group"><label>ชื่อนักเกษตร</label>
            <select id="filterFarmer" onchange="applyFilters()"><option value="all">ทั้งหมด (นักเกษตร)</option></select></div>
          <div class="filter-group"><label>ค้นหาชื่อชาวไร่</label>
            <input type="text" id="filterOwnerSearch" placeholder="🔍 พิมพ์ชื่อชาวไร่..." oninput="onFilterSearchInput()"
              style="border:1px solid #e2e8f0;border-radius:6px;padding:7px 10px;font-size:.9rem;background:#f7fafc;font-family:inherit;min-width:180px;"></div>
          <div class="filter-group"><label>ค้นหารหัสแปลง</label>
            <input type="text" id="filterPlotCodeSearch" placeholder="🔍 รหัสแปลง..." oninput="onFilterSearchInput()"
              style="border:1px solid #e2e8f0;border-radius:6px;padding:7px 10px;font-size:.9rem;background:#f7fafc;font-family:inherit;min-width:180px;"></div>
          <div class="filter-group"><label>ค้นหาชื่อกลุ่มตัด</label>
            <input type="text" id="filterCutGroupSearch" placeholder="🔍 ชื่อกลุ่มตัด..." oninput="onFilterSearchInput()"
              style="border:1px solid #e2e8f0;border-radius:6px;padding:7px 10px;font-size:.9rem;background:#f7fafc;font-family:inherit;min-width:200px;"></div>
          <div class="filter-group"><label>ประเภทการตัด</label>
            <select id="filterCutType" onchange="applyFilters()"><option value="all">ทั้งหมด (ประเภทการตัด)</option></select></div>
          <!-- กรองตาม "อายุอ้อย" อิงคอลัมน์ T "เกณฑ์การตัด" -->
          <div class="filter-group"><label>อายุอ้อย</label>
            <select id="filterCriteria" onchange="applyFilters()">
              <option value="all">ทั้งหมด</option>
              <option value="met">ถึงเกณฑ์</option>
              <option value="notmet">ไม่ถึงเกณฑ์</option>
            </select></div>
          <!-- กรองตาม "ลำดับการตัด" (คอลัมน์ U) -->
          <div class="filter-group"><label>ลำดับการตัด</label>
            <select id="filterOrdered" onchange="applyFilters()">
              <option value="all">ทั้งหมด</option>
              <option value="ordered">จัดลำดับแล้ว</option>
              <option value="notordered">ยังไม่จัดลำดับ</option>
            </select></div>
          <button class="btn-reset" onclick="resetFilters()">🔄 รีเซ็ต</button>
        </div>
        <!-- บล็อกความก้าวหน้าจัดลำดับ + แปลงผิดปกติ (Data!V) -->
        <div class="cg-summary-row cols-5">
          <div class="cg-scard cg-scard-decrai"><div class="cg-scard-val" id="tblDecRai">-</div><div class="cg-scard-lbl">เป้าตัดธันวาคม (ไร่)</div></div>
          <div class="cg-scard cg-scard-decreg"><div class="cg-scard-val" id="tblDecReg">-</div><div class="cg-scard-lbl">จัดลำดับตัด (ไร่)</div></div>
          <div class="cg-scard cg-scard-decton"><div class="cg-scard-val" id="tblDecTon">-</div><div class="cg-scard-lbl">เป้าตัดธันวาคม (ตัน)</div></div>
          <div class="cg-scard cg-scard-decregton"><div class="cg-scard-val" id="tblDecRegTon">-</div><div class="cg-scard-lbl">จัดลำดับตัด (ตัน)</div></div>
          <div class="cg-scard cg-scard-abn"><div class="cg-scard-val" id="tblAbnormal">-</div><div class="cg-scard-lbl">แปลงผิดปกติ (แปลง)</div></div>
        </div>
        <div class="table-box">
          <div class="table-header">
            <h3>📋 ข้อมูลแปลง</h3>
            <div style="display:flex;align-items:center;gap:12px;">
              <span id="tableCount" style="font-size:.85rem;color:#718096;"></span>
              <button class="btn-save" id="btnSave" onclick="saveChanges()" disabled>
                <span class="dot"></span>💾 บันทึกข้อมูล
              </button>
            </div>
          </div>
          <div class="table-wrapper">
            <table>
              <thead><tr id="tblHeadRow"></tr></thead>
              <tbody id="tableBody"><tr><td colspan="14">⏳ กำลังโหลดข้อมูล...</td></tr></tbody>
            </table>
          </div>
        </div>
      </div>
      <!-- ==================== /TABLE ==================== -->

      <!-- ==================== CUT GROUP ==================== -->
      <div id="viewCutgroup" style="display:none">
        <!-- 3 หมวดสรุป รูปแบบเดียวกับการ์ด "ชาวไร่ (ราย)" (ไม่มีกราฟวงกลม) -->
        <div class="cards cg-cards">
          <!-- หมวด 1: ปริมาณอ้อยและจำนวนคิว -->
          <div class="card cg-card cg-card-1 cg-card-boldlbl">
            <div class="card-top"><div class="card-icon">🗺️</div><div class="card-label">ปริมาณอ้อยและจำนวนคิว</div></div>
            <div class="card-divider"></div>
            <div class="donut-legend dl-big">
              <div class="dl-item"><span class="dl-dot" style="background:#e69b7a"></span><span class="dl-txt">จดแจ้ง (ไร่/หีบ)</span><strong id="cgTotalRegRai">-</strong></div>
              <div class="dl-item"><span class="dl-dot" style="background:#db2777"></span><span class="dl-txt">จดแจ้ง (ตัน/หีบ)</span><strong id="cgTotalRegTon">-</strong></div>
              <div class="dl-item"><span class="dl-dot" style="background:#c2185b"></span><span class="dl-txt">คิวรวม (คิว)</span><strong id="cgTotalQueue">-</strong></div>
            </div>
          </div>
          <!-- หมวด 2: ความคืบหน้าของกลุ่ม -->
          <div class="card cg-card cg-card-2 cg-card-boldlbl">
            <div class="card-top"><div class="card-icon">📈</div><div class="card-label">ความคืบหน้าของกลุ่ม</div></div>
            <div class="card-divider"></div>
            <div class="donut-legend dl-big">
              <div class="dl-item"><span class="dl-dot" style="background:#1e3a8a"></span><span class="dl-txt">จัดแล้ว (กลุ่ม)</span><strong id="cgTotalAssigned">-</strong></div>
              <div class="dl-item"><span class="dl-dot" style="background:#fca5a5"></span><span class="dl-txt">ยังไม่จัด (กลุ่ม)</span><strong id="cgTotalNotAssigned" style="color:#e53e3e">-</strong></div>
              <div class="dl-total">
                <div class="cg-foot-row"><span>ทั้งหมด (กลุ่ม)</span><strong id="cgTotalConfirmed">-</strong></div>
              </div>
            </div>
          </div>
          <!-- หมวด 3: ความก้าวหน้าจัดลำดับ -->
          <div class="card cg-card cg-card-3 cg-card-boldlbl">
            <div class="card-top"><div class="card-icon">🗓️</div><div class="card-label">ความก้าวหน้าจัดลำดับ</div></div>
            <div class="card-divider"></div>
            <div class="donut-legend dl-big">
              <div class="dl-item"><span class="dl-dot" style="background:#f97316"></span><span class="dl-txt">เป้าตัดธันวาคม (ไร่)</span><strong id="cgTotalDecRai">-</strong></div>
              <div class="dl-item"><span class="dl-dot" style="background:#c2410c"></span><span class="dl-txt">จัดลำดับตัด (ไร่)</span><strong id="cgTotalDecReg" class="cg-num-green">-</strong></div>
              <div class="dl-item"><span class="dl-dot" style="background:#4ade80"></span><span class="dl-txt">เป้าตัดธันวาคม (ตัน)</span><strong id="cgTotalDecTon">-</strong></div>
              <div class="dl-item"><span class="dl-dot" style="background:#15803d"></span><span class="dl-txt">จัดลำดับตัด (ตัน)</span><strong id="cgTotalDecRegTon" class="cg-num-green">-</strong></div>
            </div>
          </div>
        </div>
        <div class="cg-filter">
          <div class="filter-group"><label>เขต</label>
            <select id="cgFilterZone" onchange="cgApplyFilter()"><option value="all">ทั้งหมด (เขต)</option></select></div>
          <div class="filter-group"><label>ค้นหา ชื่อกลุ่มตัด</label>
            <input type="text" id="cgSearch" placeholder="🔍 พิมพ์เพื่อค้นหา..." oninput="cgApplyFilter()"
              style="border:1px solid #e2e8f0;border-radius:6px;padding:7px 10px;font-size:.9rem;background:#f7fafc;font-family:inherit;min-width:220px;"></div>
          <div class="filter-group"><label>ประเภทการตัด</label>
            <div class="cg-status-pills" id="cgStatusPills">
              <button type="button" class="cg-status-pill pill-all active" data-status="all" onclick="cgSetStatusFilter('all')">ทั้งหมด</button>
              <button type="button" class="cg-status-pill pill-factory" data-status="factory" onclick="cgSetStatusFilter('factory')"><span class="cg-pill-dot" style="background:#a855f7"></span>รถตัดโรงงาน</button>
              <button type="button" class="cg-status-pill pill-cluster" data-status="cluster" onclick="cgSetStatusFilter('cluster')"><span class="cg-pill-dot" style="background:#89CFF0"></span>รถตัด Cluster</button>
              <button type="button" class="cg-status-pill pill-contract" data-status="contract" onclick="cgSetStatusFilter('contract')"><span class="cg-pill-dot" style="background:#f97316"></span>รถตัดผู้รับเหมา</button>
              <button type="button" class="cg-status-pill pill-labor" data-status="labor" onclick="cgSetStatusFilter('labor')"><span class="cg-pill-dot" style="background:#eab308"></span>แรงงาน</button>
              <button type="button" class="cg-status-pill pill-landzone1" data-status="landzone1" onclick="cgSetStatusFilter('landzone1')"><span class="cg-pill-dot" style="background:#ec4899"></span>ลานขนถ่ายเขต 1</button>
              <button type="button" class="cg-status-pill pill-landzone8" data-status="landzone8" onclick="cgSetStatusFilter('landzone8')"><span class="cg-pill-dot" style="background:#2563eb"></span>ลานขนถ่ายเขต 8</button>
            </div>
          </div>
          <button class="btn-reset" onclick="cgResetFilter()">🔄 รีเซ็ต</button>
        </div>
        <div class="cg-table-box">
          <div class="table-header">
            <h3>🚜 ข้อมูลกลุ่มตัด</h3>
            <span id="cgTableCount" style="font-size:.85rem;color:#718096;"></span>
          </div>
          <div class="cg-table-wrapper">
            <table class="cg-table">
              <thead><tr id="cgHeadRow"></tr></thead>
              <tbody id="cgTableBody"><tr><td colspan="10" style="text-align:center;padding:20px">⏳ กำลังโหลดข้อมูล...</td></tr></tbody>
            </table>
          </div>
        </div>
      </div>
      <!-- ==================== /CUT GROUP ==================== -->

      <!-- ==================== MAP ==================== -->
      <div id="viewMap" style="display:none">
        <div class="map-panel">
          <div class="map-panel-head"><h2>สถานะกลุ่มตัดอ้อย</h2><span id="mapGroupCount">รวม - กลุ่มตัดในแผนนี้</span></div>
          <div class="map-filters" style="padding:0;box-shadow:none;margin-bottom:0">
            <div class="filter-group"><label>เขต</label>
              <div class="ms" id="mapFZone" data-all="ทุกเขต"><button type="button" class="ms-btn" onclick="msToggle('mapFZone')"><span class="ms-txt">ทุกเขต</span><span class="ms-arrow">▾</span></button><div class="ms-panel" style="display:none"><div class="ms-head"><a onclick="msAll('mapFZone')">เลือกทั้งหมด</a><a onclick="msClear('mapFZone')">ล้างการเลือก</a></div><div class="ms-list"></div></div></div></div>
            <div class="filter-group"><label>ประเภทการตัด</label>
              <div class="ms" id="mapFCutType" data-all="ทุกประเภท"><button type="button" class="ms-btn" onclick="msToggle('mapFCutType')"><span class="ms-txt">ทุกประเภท</span><span class="ms-arrow">▾</span></button><div class="ms-panel" style="display:none"><div class="ms-head"><a onclick="msAll('mapFCutType')">เลือกทั้งหมด</a><a onclick="msClear('mapFCutType')">ล้างการเลือก</a></div><div class="ms-list"></div></div></div></div>
            <div class="filter-group"><label>กลุ่มตัด</label>
              <div class="ms" id="mapFGroup" data-all="ทุกกลุ่มตัด"><button type="button" class="ms-btn" onclick="msToggle('mapFGroup')"><span class="ms-txt">ทุกกลุ่มตัด</span><span class="ms-arrow">▾</span></button><div class="ms-panel" style="display:none"><div class="ms-head"><a onclick="msAll('mapFGroup')">เลือกทั้งหมด</a><a onclick="msClear('mapFGroup')">ล้างการเลือก</a></div><div class="ms-search"><input type="text" placeholder="พิมพ์ค้นหากลุ่มตัด..." oninput="msSearch('mapFGroup', this.value)"></div><div class="ms-list"></div></div></div></div>
            <div class="filter-group"><label>ชื่อกลุ่มตัด</label>
              <div class="ms" id="mapFGroupName" data-all="ทุกชื่อกลุ่มตัด"><button type="button" class="ms-btn" onclick="msToggle('mapFGroupName')"><span class="ms-txt">ทุกชื่อกลุ่มตัด</span><span class="ms-arrow">▾</span></button><div class="ms-panel" style="display:none"><div class="ms-head"><a onclick="msAll('mapFGroupName')">เลือกทั้งหมด</a><a onclick="msClear('mapFGroupName')">ล้างการเลือก</a></div><div class="ms-search"><input type="text" placeholder="พิมพ์ค้นหาชื่อกลุ่มตัด..." oninput="msSearch('mapFGroupName', this.value)"></div><div class="ms-list"></div></div></div></div>
            <div class="filter-group"><label>รหัสแปลง</label><input type="text" id="mapFPlot" placeholder="ค้นหารหัสแปลง..." oninput="mapOnSearch()"></div>
            <div class="filter-group"><label>เกณฑ์การตัด</label>
              <div class="ms" id="mapFCrit" data-all="ทุกเกณฑ์"><button type="button" class="ms-btn" onclick="msToggle('mapFCrit')"><span class="ms-txt">ทุกเกณฑ์</span><span class="ms-arrow">▾</span></button><div class="ms-panel" style="display:none"><div class="ms-head"><a onclick="msAll('mapFCrit')">เลือกทั้งหมด</a><a onclick="msClear('mapFCrit')">ล้างการเลือก</a></div><div class="ms-list"></div></div></div></div>
            <div class="map-filter-btns">
              <button class="map-btn blue" onclick="mapResetFilters()">↺ รีเซ็ต</button>
              <button class="map-btn green" onclick="mapDownloadKmz()">⬇ ดาวน์โหลด KMZ</button>
            </div>
          </div>
        </div>

        <div class="map-stats">
          <div class="map-stat"><div class="map-stat-top"><span class="map-stat-ic" style="background:#e0ecff">▤</span>จำนวนแปลง</div><div class="map-stat-val" id="mapStPlots">-</div></div>
          <div class="map-stat"><div class="map-stat-top"><span class="map-stat-ic" style="background:#dcf5e6">◈</span>พื้นที่รวม</div><div class="map-stat-val"><span id="mapStArea">-</span><small>ไร่</small></div></div>
          <div class="map-stat"><div class="map-stat-top"><span class="map-stat-ic" style="background:#fff0d1">▣</span>ต้นประเมินรวม</div><div class="map-stat-val"><span id="mapStTon">-</span><small>ตัน</small></div></div>
          <div class="map-stat"><div class="map-stat-top"><span class="map-stat-ic" style="background:#ece4ff">◎</span>ชาวไร่ในกลุ่ม</div><div class="map-stat-val" id="mapStOwners">-</div></div>
        </div>

        <div class="map-main">
          <div class="map-card" id="mapCard">
            <h3>📍 แผนที่กลุ่มตัด</h3>
            <div class="map-card-bar">
              <div class="map-base-pills">
                <button type="button" class="map-base-pill active" data-base="sat" onclick="mapSetBase('sat')">ภาพถ่ายดาวเทียม</button>
                <button type="button" class="map-base-pill" data-base="street" onclick="mapSetBase('street')">แผนที่</button>
              </div>
              <button type="button" class="map-btn-expand" id="mapExpandBtn" onclick="mapToggleExpand()">⛶ ขยาย</button>
            </div>
            <div id="mapCanvas"></div>
            <div class="map-legend">
              <span><i style="background:#f28c28"></i>อ้อยตอ</span>
              <span><i style="background:#5bb89a"></i>อ้อยต้นฝน</span>
              <span><i style="background:#1e2d55"></i>อ้อยปลายฝน</span>
              <span><span class="lg-ord">1</span>หมุดมีเลข = ลำดับการตัด (สีตามประเภทอ้อย)</span>
              <span><span class="lg-line"></span>เส้นเชื่อมตามลำดับการตัด (เฉพาะในกลุ่มตัดเดียวกัน)</span>
            </div>
            <div class="map-note" id="mapNote">⏳ กำลังโหลด...</div>
          </div>
          <div class="map-side">
            <div class="map-tabs">
              <button type="button" class="map-tab active" data-tab="members" onclick="mapSetTab('members')">สมาชิก</button>
              <button type="button" class="map-tab" data-tab="plots" onclick="mapSetTab('plots')">แปลงอ้อยทั้งหมด</button>
            </div>
            <div id="mapSideInfo" style="display:none"></div>
            <div id="mapSideScroll"><div id="mapSideBody"></div></div>
          </div>
        </div>
      </div>
      <!-- ==================== /MAP ==================== -->

    </div><!-- /container -->

    <!-- ==================== SAVE CONFIRM MODAL ==================== -->
    <div id="saveConfirmOverlay" class="save-confirm-overlay" style="display:none">
      <div class="save-confirm-box">
        <div class="save-confirm-title">💾 ยืนยันการบันทึกข้อมูล</div>
        <div class="save-confirm-sub" id="saveConfirmSub">กรุณาตรวจสอบรายการก่อนบันทึก</div>
        <div class="save-confirm-scroll">
          <table class="save-confirm-table">
            <thead><tr>
              <th>รหัสแปลง</th><th>ชื่อชาวไร่</th><th>โรงงาน</th><th>กลุ่มตัด</th><th>ชื่อกลุ่มตัด</th><th>ลำดับการตัด</th>
            </tr></thead>
            <tbody id="saveConfirmBody"></tbody>
          </table>
        </div>
        <div class="save-confirm-actions">
          <button type="button" class="btn-save-cancel" onclick="cancelSaveConfirm()">ยกเลิก</button>
          <button type="button" class="btn-save-confirm" onclick="confirmSaveConfirm()">✅ ยืนยันบันทึก</button>
        </div>
      </div>
    </div>
    <!-- ==================== /SAVE CONFIRM MODAL ==================== -->
  </div><!-- /main-content -->
</div><!-- /app-shell -->

<script>
// ★ [แก้ไข ข้อ 78] ข้อมูลที่เซิร์ฟเวอร์ฝังมาให้ (จากแคช) ใช้ครั้งเดียวตอนเปิดหน้า / รีเฟรชจะอ่านจากเซิร์ฟเวอร์ตามปกติ
let PRE = { main: ${_jsLit(pre && pre.main)}, code: ${_jsLit(pre && pre.code)} };

// ===== STATE =====
let ALL_ROWS = [], CURRENT_ROWS = [], DB_ROWS = [];
let CG_ALL_ROWS = [];
let INSP_ALL_ROWS = [];
let INSP_RESULT_MAP = {};
let cgStatusFilter = 'all';
let dbChartMode = 'ton';
let _dbRenderedModes = {};
// หน้าจดแจ้งฯ แก้ไขได้เฉพาะ "ลำดับการตัด" (โรงงาน / กลุ่มตัด / ชื่อกลุ่มตัด เป็นข้อความอ่านอย่างเดียว)
let pendingOrders = {};
let STATUS_MAP = {};
let plotCodeMap = {};

// ทะเบียน Chart.js (key = id ของ canvas) แทนตัวแปรกราฟแยกรายตัว
const CHARTS = {};
function mkChart(canvasId, cfg) {
  if (CHARTS[canvasId]) CHARTS[canvasId].destroy();
  CHARTS[canvasId] = new Chart(document.getElementById(canvasId).getContext('2d'), cfg);
  return CHARTS[canvasId];
}

// ตัวช่วยกลาง
const $ = id => document.getElementById(id);
const setText = (id, v) => { const el = $(id); if (el) el.textContent = v; };
const fmt = n => Math.round(n).toLocaleString('th-TH');
const thSort = (a, b) => a.localeCompare(b, 'th');
function pctStr(a, b) {
  return b > 0 ? (a / b * 100).toLocaleString('th-TH', { minimumFractionDigits: 1, maximumFractionDigits: 1 }) + '%' : '-';
}
// ตัวเลข + (เปอร์เซ็นต์ เทียบกับ b) ต่อท้าย
function setValPct(id, valTxt, a, b, cls) {
  const el = $(id);
  if (el) el.innerHTML = valTxt + '<span class="' + cls + '">(' + pctStr(a, b) + ')</span>';
}

// ABNORMAL:17 = คอลัมน์ V "แปลงผิดปกติ" (1 = ผิดปกติ)
const F = { ROW:0,FARMER:1,ZONE:2,CANETYPE:3,AREA:4,PLOTINTECH:5,PLOTCODE:6,OWNER:7,FACTORY:8,CUTTYPE:9,CUTGROUP:10,CUTGROUPNAME:11,EST_TON:12,TON_PER_RAI:13,CANE_AGE:14,CUT_ORDER:15,CUT_CRITERIA:16,ABNORMAL:17,FARMERID:18 };
const ZONE_ORDER = ['1','2','3','4','5','6','7','8','10','11','12','21','9','16','C1','C2','C3','C4'];
function zoneCompare(a, b) {
  const ia = ZONE_ORDER.indexOf(a), ib = ZONE_ORDER.indexOf(b);
  if (ia === -1 && ib === -1) return String(a).localeCompare(String(b), 'th');
  if (ia === -1) return 1;
  if (ib === -1) return -1;
  return ia - ib;
}
const CUT_COLORS = { 'รถตัดโรงงาน':'#9b59b6','รถตัดผู้รับเหมา':'#27ae60','รถตัดชาวไร่':'#2980b9','แรงงาน':'#e67e22','รถตัด Cluster':'#89CFF0','ลานขนถ่ายเขต 1':'#ec4899','ลานขนถ่ายเขต 8':'#2563eb','ยังไม่จัดคิว':'#95a5a6' };

// จำนวนคอลัมน์ตาราง "จดแจ้งประเภทการตัดอ้อย"
const TABLE_COLS = 14;

function plotKey(r) {
  const code = (r[F.PLOTINTECH] || '').trim();
  return code || ('row_' + r[F.ROW]);
}

let _inspectLoaded = false;
// บังคับอ่านชีตใหม่ (หลังกดรีเฟรช)
let _forceRefresh = false;
let _forceInsp    = false;
// เซิร์ฟเวอร์ส่งข้อมูลกลับเป็นข้อความ JSON (เร็วกว่าส่ง object ก้อนใหญ่) -> แปลงกลับที่นี่
function _pj(d) {
  if (typeof d !== 'string') return d || {};
  try { return JSON.parse(d); } catch (e) { return { error: 'อ่านข้อมูลไม่สำเร็จ: ' + e.message }; }
}

// ★ [แก้ไข ข้อ 78] โหลดไลบรารีภายนอกตอนใช้งาน (Leaflet / JSZip) แทนการโหลดทุกครั้งที่เปิดหน้า
const LEAFLET_CSS = 'https://cdn.jsdelivr.net/npm/leaflet@1.9.4/dist/leaflet.css';
const LEAFLET_JS  = 'https://cdn.jsdelivr.net/npm/leaflet@1.9.4/dist/leaflet.js';
const JSZIP_JS    = 'https://cdn.jsdelivr.net/npm/jszip@3.10.1/dist/jszip.min.js';
const _libs = {};
function _loadScript(src, cb) {
  const l = _libs[src] || (_libs[src] = { done: false, cbs: [] });
  if (l.done) { cb(true); return; }
  l.cbs.push(cb);
  if (l.cbs.length > 1) return;
  const s = document.createElement('script');
  s.src = src;
  s.onload  = function() { l.done = true; l.cbs.splice(0).forEach(function(f) { f(true); }); };
  s.onerror = function() { const cbs = l.cbs.splice(0); delete _libs[src]; s.remove(); cbs.forEach(function(f) { f(false); }); };
  document.head.appendChild(s);
}
function _ensureLeafletCss() {
  if ($('leafletCss')) return;
  const k = document.createElement('link');
  k.id = 'leafletCss'; k.rel = 'stylesheet'; k.href = LEAFLET_CSS;
  document.head.appendChild(k);
}
// โหลดล่วงหน้าตอนว่าง เพื่อให้เปิดแผนที่ครั้งแรกไม่ต้องรอดาวน์โหลด
function _prefetchMapLibs() {
  _ensureLeafletCss();
  _loadScript(LEAFLET_JS, function() {});
  _loadScript(JSZIP_JS, function() {});
}

// ===== CHART MODE TOGGLE =====
const DB_CHART_BOXES = { ton:'chartBoxTon', korat:'chartBoxKorat', sikhio:'chartBoxSikhio', plot:'chartBoxPlot', rai:'chartBoxRai', order:'chartBoxOrder', group:'chartBoxGroup' };
function setDbChartMode(mode) {
  dbChartMode = mode;
  const sel = $('dbChartMode');
  if (sel && sel.value !== mode) sel.value = mode;
  Object.keys(DB_CHART_BOXES).forEach(m => { $(DB_CHART_BOXES[m]).style.display = m === mode ? '' : 'none'; });
  if (!_dbRenderedModes[mode]) _renderDbChartMode(mode, DB_ROWS);
}

// ===== VIEW SWITCHER =====
let currentView = 'dashboard';
const VIEW_IDS = { dashboard:'viewDashboard', table:'viewTable', cutgroup:'viewCutgroup', map:'viewMap' };
const VIEW_TITLES = { dashboard:'🌿 จัดคิวอ้อยเข้าหีบ ปี 69/70', table:'📋 จดแจ้งลำดับแผนตัดเดือนธันวาคม', cutgroup:'🚜 ข้อมูลกลุ่มตัด', map:'🗺️ แผนที่ ปี 69/70' };
function switchView(view) {
  colfClose();
  cgfClose();
  if (currentView === 'table' && view !== 'table') $('tableBody').innerHTML = '';
  Object.keys(VIEW_IDS).forEach(v => { $(VIEW_IDS[v]).style.display = v === view ? '' : 'none'; });
  document.querySelectorAll('.menu-item').forEach(el => el.classList.toggle('active', el.dataset.view === view));
  $('pageTitle').textContent = VIEW_TITLES[view] || VIEW_TITLES.dashboard;
  const header = $('pageHeader');
  header.classList.toggle('header-cutgroup', view === 'cutgroup');
  header.classList.toggle('header-table',    view === 'table');
  header.classList.toggle('header-map',      view === 'map');

  if (view === 'table') {
    tableUpdateFilterOptions(); // ★ [แก้ไข ข้อ 78] เติมตัวเลือกตัวกรองตอนเปิดหน้านี้ (ไม่ทำตอนโหลดข้อมูลแล้ว)
    renderTable(CURRENT_ROWS);
    renderCodeSummaries();
    // ยังโหลดข้อมูล GTT ไว้ เพื่อไฮไลต์แถวแดง "รถตัดโรงงาน + ไม่รับตัด"
    if (!_inspectLoaded) {
      _inspectLoaded = true;
      loadInspectData(function() {
        if (currentView === 'table') _refreshAllRowClasses(); // แค่อัปเดตสีแถวที่วาดอยู่ ไม่วาดตารางใหม่
      });
    }
  }
  if (view === 'dashboard') applyDbFilters();
  if (view === 'cutgroup') {
    if (!CG_ALL_ROWS.length) {
      // ข้อมูลกำลังโหลด (เริ่มโหลดตั้งแต่เปิดหน้า) หรือโหลดไม่สำเร็จ -> แสดงสถานะรอ แล้ววาดเมื่อโหลดเสร็จ
      $('cgTableBody').innerHTML =
        '<tr><td colspan="' + CG_COLS + '" style="text-align:center;padding:20px">⏳ กำลังโหลดข้อมูล...</td></tr>';
      if (!_codeForTableLoading) fetchCodeDataForTable(false);
    } else { cgApplyFilter(); }
  }
  if (view === 'map') mapOpen();

  currentView = view;
}

// ===== SKELETON =====
function showSkeleton() {
  if (currentView !== 'table') return; // ตารางถูกซ่อนอยู่ ไม่ต้องวาด
  const skRow = '<tr>' + Array(TABLE_COLS).fill('<td><div class="skel"></div></td>').join('') + '</tr>';
  $('tableBody').innerHTML = Array(8).fill(skRow).join('');
}

// ===== REFRESH DATA =====
function refreshData() {
  showToast('⏳ กำลังรีเฟรชข้อมูลจาก Google Sheet...', 'success');
  $('updateTime').textContent = 'กำลังโหลดข้อมูลใหม่...';

  ALL_ROWS = []; CURRENT_ROWS = []; DB_ROWS = [];
  CG_ALL_ROWS = [];
  INSP_ALL_ROWS = []; INSP_RESULT_MAP = {};
  pendingOrders = {};
  _inspectLoaded = false;
  MAP_COORDS = {}; _mapCoordsLoaded = false; _mapCoordsLoading = false; _forceMapCoords = true; _mapFitted = false; MAP_ROWS = [];

  // สั่งให้ทุกฟังก์ชันอ่านชีตใหม่ (forceRefresh) ในรอบเดียว
  _forceRefresh = true;
  _forceInsp    = true;
  init();
}

// ===== INIT =====
// ★ [แก้ไข ข้อ 78] ถ้าเซิร์ฟเวอร์ฝังข้อมูลมาให้ (PRE) ใช้เลยโดยไม่เรียกเซิร์ฟเวอร์ / รีเฟรชข้อมูลจะอ่านสดทุกครั้ง
function init() {
  showSkeleton();
  const force = _forceRefresh;
  _forceRefresh = false;
  const pre = force ? { main: null, code: null } : PRE;
  PRE = { main: null, code: null };

  // ชีต Code ก่อน (ถ้ามาพร้อมกัน บล็อกสรุปจะได้ไม่วาดเป็น 0 ชั่วขณะ)
  if (pre.code) _applyCodeData(_pj(pre.code));
  else fetchCodeDataForTable(force);

  // ชีต GTT ใช้ไฮไลต์แถวแดงในตารางจดแจ้งฯ เท่านั้น -> ไม่โหลดตอนเปิดแดชบอร์ด
  if (!_inspectLoaded && currentView === 'table') {
    _inspectLoaded = true;
    loadInspectData(function() {
      if (currentView === 'table') _refreshAllRowClasses();
    }, force);
  }

  if (pre.main) { _applyInitialData(_pj(pre.main), false); return; }

  google.script.run
    .withSuccessHandler(function(raw) { _applyInitialData(_pj(raw), force); })
    .withFailureHandler(function(err) {
      showToast('❌ ' + err.message, 'error');
      $('updateTime').textContent = 'โหลดข้อมูลไม่สำเร็จ';
      if (currentView === 'table') $('tableBody').innerHTML =
        '<tr><td colspan="' + TABLE_COLS + '" style="color:red">❌ ' + err.message + '</td></tr>';
    })
    .getInitialData(force);
}

function _applyInitialData(data, showDone) {
  if (data.error) { showToast(data.error, 'error'); $('updateTime').textContent = 'โหลดข้อมูลไม่สำเร็จ'; return; }
  // ถอดรหัสพจนานุกรม (เลขอ้างอิง -> ข้อความ)
  if (data.dict && data.dictCols && data.allRows) {
    const S = data.dict, C = data.dictCols, R = data.allRows, nc = C.length;
    for (let i = 0; i < R.length; i++) {
      const r = R[i];
      for (let k = 0; k < nc; k++) r[C[k]] = S[r[C[k]]];
    }
  }
  $('updateTime').textContent =
    'ข้อมูล ณ วันที่ ' + new Date().toLocaleDateString('th-TH', { day:'2-digit', month:'long', year:'numeric' });

  STATUS_MAP   = data.statusMap || {};
  ALL_ROWS     = data.allRows   || [];
  DB_ROWS      = ALL_ROWS;      // แดชบอร์ดใช้ทุกแถว
  CURRENT_ROWS = ALL_ROWS;      // ตารางจดแจ้งฯ แสดงทุกแถว

  plotCodeMap = {};
  ALL_ROWS.forEach((r, i) => { plotCodeMap[plotKey(r)] = i; });

  dbUpdateFilterOptions(); // ตัวเลือกตัวกรองคำนวณจากข้อมูลแปลงที่โหลดมา (เซิร์ฟเวอร์ไม่ต้องส่ง dropdowns แล้ว)
  renderDashboard(ALL_ROWS);

  if (currentView === 'table') { tableUpdateFilterOptions(); renderTable(CURRENT_ROWS); }
  renderCodeSummaries(); // ถ้าชีต Code โหลดเสร็จก่อนแล้ว -> วาดบล็อกสรุป
  if (currentView === 'map') mapOpen(); // เปิดหน้าแผนที่ค้างไว้ระหว่างรีเฟรช -> วาดใหม่เมื่อข้อมูลมาถึง

  if (showDone) showToast('✅ โหลดข้อมูลใหม่สำเร็จ', 'success');
  setTimeout(_prefetchMapLibs, 2500);

  if (data.duplicatePlotCodes && data.duplicatePlotCodes.length) {
    setTimeout(function() {
      showToast('⚠️ พบรหัสแปลงซ้ำ ' + data.duplicatePlotCodes.length + ' รหัสในชีต อาจกรอกข้อมูลผิดแถวได้ กรุณาแจ้งผู้ดูแลระบบ', 'error');
    }, 3800);
  }
}

// ===== ชีต Code (ข้อมูลกลุ่มตัด + บล็อกสรุปในหน้าแดชบอร์ด / จดแจ้งฯ) =====
const CG_COLS = 12; // จำนวนคอลัมน์ตารางกลุ่มตัด
let _codeForTableLoading = false;

function _codeLoadFailed(msg) {
  showToast('❌ ' + msg, 'error');
  if (currentView === 'cutgroup' && !CG_ALL_ROWS.length) {
    $('cgTableBody').innerHTML =
      '<tr><td colspan="' + CG_COLS + '" style="color:red;text-align:center;padding:20px">❌ ' + msg + '</td></tr>';
  }
  renderCodeSummaries();
}

function _applyCodeData(data) {
  if (data.error) { _codeLoadFailed(data.error); return; }
  CG_ALL_ROWS = data.rows || [];
  renderCodeSummaries();
  if (currentView === 'cutgroup') cgApplyFilter();
  // กราฟ "จัดลำดับตัด(กลุ่ม)" ในแดชบอร์ดใช้ข้อมูลชีต Code -> วาดใหม่เมื่อข้อมูลมาถึง
  _dbRenderedModes.group = false;
  if (currentView === 'dashboard' && dbChartMode === 'group' && typeof Chart !== 'undefined' && typeof ChartDataLabels !== 'undefined') {
    _renderDbChartMode('group', DB_ROWS);
  }
}

function fetchCodeDataForTable(forceRefresh) {
  if (_codeForTableLoading) return;
  _codeForTableLoading = true;
  ['tblDecRai','tblDecReg','tblDecTon','tblDecRegTon','dbTotalHarvester','dbTotalLabor','dbTotalTruck'].forEach(id => {
    const el = $(id); if (el && el.textContent === '-') el.textContent = '⏳';
  });
  google.script.run
    .withSuccessHandler(function(raw) {
      _codeForTableLoading = false;
      _applyCodeData(_pj(raw));
    })
    .withFailureHandler(function(err) {
      _codeForTableLoading = false;
      _codeLoadFailed(err.message);
    })
    .getCodeData(!!forceRefresh);
}

function renderCodeSummaries() {
  renderTableDecSummary();
  renderTableAbnormal();
  renderDbFleetSummary();
}

// แดชบอร์ด: รถตัดอ้อย / แรงงาน / รถบรรทุก จากชีต Code (คอลัมน์ "คัน หรือ คน" / "รถบรรทุก (คัน)")
// นับเฉพาะกลุ่มที่ "จดแจ้ง (ตัน)" > 0
function renderDbFleetSummary() {
  if (!CG_ALL_ROWS.length && _codeForTableLoading) return;
  const zone = $('dbFilterZone') ? $('dbFilterZone').value : 'all';
  let harvester = 0, labor = 0, truck = 0;
  for (let i = 0; i < CG_ALL_ROWS.length; i++) {
    const r = CG_ALL_ROWS[i];
    if (zone !== 'all' && String(r.zone) !== zone) continue;
    if (!((Number(r.regTon) || 0) > 0)) continue;
    const bucket = cgStatusBucket(r.status);
    const st = String(r.status || '').trim();
    const unit = parseFloat(r.unit) || 0;
    if (bucket === 'labor' || st === 'ลานขนถ่ายเขต 1' || st === 'ลานขนถ่ายเขต 8') labor += unit;
    if (bucket === 'factory' || bucket === 'cluster' || bucket === 'contract') harvester += unit;
    truck += Number(r.truck) || 0;
  }
  setText('dbTotalHarvester', fmt(harvester));
  setText('dbTotalLabor',     fmt(labor));
  setText('dbTotalTruck',     fmt(truck));
}

// มีตัวกรองอื่นนอกจาก "เขต / โซน" หรือไม่ (ไม่นับ "อายุอ้อย" และ "ลำดับการตัด")
function _tableHasNonZoneFilter() {
  const f = tableGetFilterValues();
  return f.farmer !== 'all' || !!f.ownerSearch || !!f.plotSearch || !!f.cgSearch || f.ctType !== 'all' || _colfActive();
}

// บล็อกเป้าตัดธันวาคม / จัดลำดับตัด (หน้าจดแจ้งฯ)
// ใช้ตัวเลขชุดเดียวกับการ์ดหน้า "ข้อมูลกลุ่มตัด" (ผลรวมจากชีต Code เฉพาะกลุ่มที่ show = 1)
function renderTableDecSummary() {
  if (!CG_ALL_ROWS.length && _codeForTableLoading) return;
  const zone = $('filterZone') ? $('filterZone').value : 'all';
  const fv   = tableGetFilterValues();
  const f2   = Object.assign({}, fv, { criteria: 'all', ordered: 'all' });
  // มีตัวกรองอื่น -> ใช้เฉพาะกลุ่มตัดที่มีแปลงอยู่ในผลการกรอง (ไม่นับอายุอ้อย / ลำดับการตัด)
  let codeSet = null;
  if (_tableHasNonZoneFilter()) {
    codeSet = {};
    for (let i = 0; i < ALL_ROWS.length; i++) {
      const p = ALL_ROWS[i];
      if (!tableRowMatchesFilters(p, f2, null)) continue;
      const c = String(p[F.CUTGROUP] || '').trim();
      if (c) codeSet[c] = 1;
    }
  }
  let plRai = 0, plTon = 0, rgRai = 0, rgTon = 0;
  for (let i = 0; i < CG_ALL_ROWS.length; i++) {
    const r = CG_ALL_ROWS[i];
    if (!r.show) continue; // เฉพาะกลุ่มที่ Code!AF "แสดงข้อมูล" = 1
    if (codeSet) { if (!codeSet[String(r.code || '').trim()]) continue; }
    else if (zone !== 'all' && String(r.zone) !== zone) continue;
    plRai += Number(r.decPlanRai) || 0;
    plTon += Number(r.decPlanTon) || 0;
    rgRai += Number(r.decRegRai)  || 0;
    rgTon += Number(r.decRegTon)  || 0;
  }

  // เมื่อกรอง "ชื่อนักเกษตร": เป้า (ไร่/ตัน) = รวมเฉพาะแปลงที่อายุอ้อย "ถึงเกณฑ์"
  if (fv.farmer !== 'all' || COLF.farmer) {
    plRai = 0; plTon = 0;
    for (let i = 0; i < ALL_ROWS.length; i++) {
      const p = ALL_ROWS[i];
      if (!tableRowMatchesFilters(p, f2, null)) continue;
      if (!_isCriteriaMet(p[F.CUT_CRITERIA])) continue;
      plRai += p[F.AREA]    || 0;
      plTon += p[F.EST_TON] || 0;
    }
  }

  setText('tblDecRai', fmt(plRai));
  setText('tblDecTon', fmt(plTon));
  setValPct('tblDecReg',    fmt(rgRai), rgRai, plRai, 'cg-scard-pct');
  setValPct('tblDecRegTon', fmt(rgTon), rgTon, plTon, 'cg-scard-pct');
  // จัดลำดับตัด: ถึง 100% = พื้นเขียว / ยังไม่ถึง = พื้นแดง
  [['tblDecReg', rgRai, plRai], ['tblDecRegTon', rgTon, plTon]].forEach(function(x) {
    const card = $(x[0]) ? $(x[0]).closest('.cg-scard') : null;
    if (!card) return;
    const ok = x[2] > 0 && x[1] >= x[2];
    card.classList.toggle('dec-ok', ok);
    card.classList.toggle('dec-low', !ok);
  });
}

// บล็อก "แปลงผิดปกติ (แปลง)" = จำนวนแปลงที่ Data!V เป็นแปลงผิดปกติ
// นับตามตัวกรองของหน้าจดแจ้งฯ (ไม่นับตัวกรอง "อายุอ้อย" / "ลำดับการตัด")
function renderTableAbnormal() {
  if (!ALL_ROWS.length) return;
  const f = Object.assign({}, tableGetFilterValues(), { criteria: 'all', ordered: 'all' });
  let n = 0;
  for (let i = 0; i < ALL_ROWS.length; i++) {
    const p = ALL_ROWS[i];
    if (p[F.ABNORMAL] && tableRowMatchesFilters(p, f, null)) n++;
  }
  setText('tblAbnormal', fmt(n));
}

function cgStatusBucket(status) {
  if (!status) return 'other';
  const s = String(status).toLowerCase();
  if (s.includes('โรงงาน') || s.includes('factory')) return 'factory';
  if (s.includes('cluster') || s.includes('คลัสเตอร์')) return 'cluster';
  if (s.includes('ผู้รับเหมา') || s.includes('contract') || s.includes('เหมา')) return 'contract';
  if (s.includes('แรงงาน') || s.includes('labor') || s.includes('labour')) return 'labor';
  return 'other';
}

function cgRowBucket(r) {
  const name = String(r.name || '').trim();
  if (name === 'ลานขนถ่ายเขต 1') return 'landzone1';
  if (name === 'ลานขนถ่ายเขต 8') return 'landzone8';
  return cgStatusBucket(r.status);
}

function cgPassesNonZoneFilters(r, q) {
  if (!r.show) return false; // เฉพาะกลุ่มที่ Code!AF "แสดงข้อมูล" = 1
  if (q && !r.code.toLowerCase().includes(q) && !r.name.toLowerCase().includes(q)) return false;
  if (cgStatusFilter !== 'all' && cgRowBucket(r) !== cgStatusFilter) return false;
  return true;
}

function cgUpdateZoneOptions() {
  const q = ($('cgSearch').value || '').trim().toLowerCase();
  const zoneSet = {};
  CG_ALL_ROWS.forEach(r => {
    if (r.zone && cgPassesNonZoneFilters(r, q)) zoneSet[String(r.zone)] = 1;
  });
  const zoneEl = $('cgFilterZone');
  const prevValue = zoneEl.value;
  zoneEl.innerHTML = '<option value="all">ทั้งหมด (เขต)</option>';
  Object.keys(zoneSet).sort(zoneCompare).forEach(z => {
    const o = document.createElement('option'); o.value = z; o.textContent = 'เขต ' + z; zoneEl.appendChild(o);
  });
  zoneEl.value = zoneSet[prevValue] ? prevValue : 'all';
}

function cgSetStatusFilter(key) {
  cgStatusFilter = key;
  document.querySelectorAll('#cgStatusPills .cg-status-pill').forEach(btn => btn.classList.toggle('active', btn.dataset.status === key));
  cgApplyFilter();
}

function cgApplyFilter() {
  cgUpdateZoneOptions();
  const zone = $('cgFilterZone').value;
  const q    = ($('cgSearch').value || '').trim().toLowerCase();
  cgRenderTable(CG_ALL_ROWS.filter(r => {
    if (zone !== 'all' && String(r.zone) !== zone) return false;
    if (!cgPassesNonZoneFilters(r, q)) return false;
    return _cgfMatch(r, null);
  }));
}

function cgResetFilter() {
  $('cgSearch').value = '';
  cgStatusFilter = 'all';
  document.querySelectorAll('#cgStatusPills .cg-status-pill').forEach(btn => btn.classList.toggle('active', btn.dataset.status === 'all'));
  $('cgFilterZone').value = 'all';
  Object.keys(CGF).forEach(k => { delete CGF[k]; });
  _cgfMark(); cgfClose();
  cgApplyFilter(); // ผ่าน cgApplyFilter เพื่อไม่ให้หลุดตัวกรอง "แสดงข้อมูล"
}

// ===== ตัวกรองรายคอลัมน์ตารางข้อมูลกลุ่มตัด (เหมือน Excel + กรองตามสีพื้นหลัง / สีตัวอักษร) =====
// CGF[คีย์คอลัมน์] = { sel:{ค่า:true}, bg:{สี:true}, fg:{สี:true} } มีเฉพาะคอลัมน์ที่กำลังกรอง
const CG_BG_GREEN = '#c6f6d5';
const CGF_COLOR_NAMES = { '#c6f6d5':'เขียวอ่อน', '#e53e3e':'แดง', '#2d6a4f':'เขียวเข้ม', '#1a202c':'ดำ (ปกติ)', '':'ไม่มีสี' };
const _cgNumTxt = v => (v !== '' && v != null) ? fmt(Number(v)) : '';
// สีพื้นหลังของแถว (ตรงกับ class row-decreg)
const _cgBg = r => (Number(r.decRegRai) || 0) !== 0 ? CG_BG_GREEN : '';
// สีตัวอักษรของช่องจดแจ้ง (ตรงกับ cg-dec-under / cg-dec-ok)
function _cgFgDec(plan, reg) {
  const c = _cgDecCls(plan, reg);
  return c === 'cg-dec-under' ? '#e53e3e' : (c === 'cg-dec-ok' ? '#2d6a4f' : '#1a202c');
}
const _cgFgNormal = () => '#1a202c';

const CGF_DEFS = [
  { k:'zone',       label:'เขต',                          th:'', get:r => String(r.zone || ''), fg:_cgFgNormal },
  { k:'code',       label:'Code กลุ่มตัด',                th:'', get:r => r.code || '',          fg:_cgFgNormal },
  { k:'name',       label:'ชื่อกลุ่มตัด',                 th:'', get:r => r.name || '',          fg:_cgFgNormal },
  { k:'brand',      label:'ยี่ห้อ',                       th:' style="min-width:110px"', get:r => r.brand || '', fg:_cgFgNormal },
  { k:'regTon',     label:'จดแจ้ง (ตัน/หีบ)',             th:' class="num" style="background:#db2777"', num:true, get:r => _cgNumTxt(r.regTon),     fg:_cgFgNormal },
  { k:'regRai',     label:'จดแจ้ง (ไร่/หีบ)',             th:' class="num" style="background:#db2777"', num:true, get:r => _cgNumTxt(r.regRai),     fg:_cgFgNormal },
  { k:'decPlanRai', label:'แผนตัดธันวาคม (ไร่)',          th:' class="num" style="background:#f97316"', num:true, get:r => _cgNumTxt(r.decPlanRai), fg:_cgFgNormal },
  { k:'decRegRai',  label:'จดแจ้งแผนตัดธันวาคม (ไร่)',    th:' class="num" style="background:#f97316"', num:true, get:r => _cgNumTxt(r.decRegRai),  fg:r => _cgFgDec(r.decPlanRai, r.decRegRai) },
  { k:'decPlanTon', label:'แผนตัดธันวาคม (ตัน)',          th:' class="num" style="background:#2d6a4f"', num:true, get:r => _cgNumTxt(r.decPlanTon), fg:_cgFgNormal },
  { k:'decRegTon',  label:'จดแจ้งแผนตัดธันวาคม (ตัน)',    th:' class="num" style="background:#2d6a4f"', num:true, get:r => _cgNumTxt(r.decRegTon),  fg:r => _cgFgDec(r.decPlanTon, r.decRegTon) },
  { k:'readyDate',  label:'วันที่พร้อมตัด', th:' style="background:#475569;min-width:110px"', date:true, get:r => r.readyDate || '', fg:_cgFgNormal },
  { k:'note',       label:'หมายเหตุ',       th:' style="background:#475569;min-width:200px"',            get:r => r.note || '',      fg:_cgFgNormal }
];
const CGF_BY = {}; CGF_DEFS.forEach(d => { CGF_BY[d.k] = d; });
const CGF = {};
const CGF_SHOW_MAX = 300;
const CGF_BLANK = '(ว่าง)';
let _cgfOpen = null, _cgfQ = '', _cgfVals = [], _cgfBgs = [], _cgfFgs = [];

const _hasKeys = o => { for (const k in o) return true; return false; };

// แถวนี้ผ่านตัวกรองรายคอลัมน์หรือไม่ (exKey = ข้ามคอลัมน์นั้น ใช้ตอนหารายการค่าของคอลัมน์เอง)
function _cgfMatch(r, exKey) {
  for (const k in CGF) {
    if (k === exKey) continue;
    const st = CGF[k], d = CGF_BY[k];
    if (_hasKeys(st.sel) && !st.sel[d.get(r)]) return false;
    if (_hasKeys(st.bg)  && !st.bg[_cgBg(r)])  return false;
    if (_hasKeys(st.fg)  && !st.fg[d.fg(r)])   return false;
  }
  return true;
}

function buildCgHead() {
  $('cgHeadRow').innerHTML = CGF_DEFS.map(d =>
    '<th' + d.th + '>' + d.label +
    '<button type="button" class="cgf-btn" data-k="' + d.k + '" title="กรอง ' + d.label +
    '" onclick="cgfOpen(this.dataset.k, this)">▾</button></th>').join('');
}

function _cgfMark() {
  document.querySelectorAll('#cgHeadRow .cgf-btn').forEach(b => b.classList.toggle('active', !!CGF[b.dataset.k]));
}

// แถวฐานของคอลัมน์ k = ผ่านตัวกรองเขต/ค้นหา/ประเภท + ตัวกรองคอลัมน์อื่น (ไม่นับตัวกรองของคอลัมน์ k เอง)
function _cgfBaseRows(exKey) {
  const zone = $('cgFilterZone').value;
  const q = ($('cgSearch').value || '').trim().toLowerCase();
  return CG_ALL_ROWS.filter(r =>
    (zone === 'all' || String(r.zone) === zone) &&
    cgPassesNonZoneFilters(r, q) && _cgRowVisible(r) && _cgfMatch(r, exKey));
}

function _cgfCompute(k) {
  const d = CGF_BY[k], vc = {}, bc = {}, fc = {};
  const rows = _cgfBaseRows(k);
  for (let i = 0; i < rows.length; i++) {
    const r = rows[i];
    const v = d.get(r), b = _cgBg(r), f = d.fg(r);
    vc[v] = (vc[v] || 0) + 1; bc[b] = (bc[b] || 0) + 1; fc[f] = (fc[f] || 0) + 1;
  }
  const num = x => parseFloat(String(x).replace(/,/g, ''));
  const dkey = x => { const p = String(x).split('/'); return p.length === 3 ? (+p[2]) * 10000 + (+p[1]) * 100 + (+p[0]) : 0; };
  _cgfVals = Object.keys(vc).map(v => ({ c: v, n: vc[v] })).sort(function(a, b) {
    if (a.c === '') return 1;
    if (b.c === '') return -1;
    if (d.date) return dkey(a.c) - dkey(b.c);
    return d.num ? num(a.c) - num(b.c) : thSort(a.c, b.c);
  });
  _cgfBgs = Object.keys(bc).map(c => ({ c: c, n: bc[c] }));
  _cgfFgs = Object.keys(fc).map(c => ({ c: c, n: fc[c] }));
}

function cgfOpen(k, btn) {
  const p = $('cgfPanel');
  if (_cgfOpen === k && p.style.display !== 'none') { cgfClose(); return; }
  _cgfOpen = k; _cgfQ = '';
  const rc = btn.getBoundingClientRect();
  p.style.top  = (rc.bottom + 4) + 'px';
  p.style.left = Math.max(8, Math.min(rc.left, window.innerWidth - 286)) + 'px';
  p.innerHTML =
    '<div class="cgf-head"><a onclick="cgfAll()">เลือกทั้งหมด</a><a onclick="cgfClear()">ล้างการเลือก</a></div>' +
    '<div class="cgf-search"><input type="text" id="cgfSearch" placeholder="ค้นหา..." oninput="cgfSearch(this.value)"></div>' +
    '<div class="cgf-list" id="cgfList"></div>';
  p.style.display = '';
  _cgfCompute(k);
  cgfRender();
  setTimeout(function() { const i = $('cgfSearch'); if (i) i.focus(); }, 30);
}

function cgfClose() {
  const p = $('cgfPanel');
  if (p) p.style.display = 'none';
  _cgfOpen = null;
}

function _cgfColorSection(type, arr, st, title) {
  if (arr.length < 2) return ''; // มีสีเดียว ไม่ต้องแสดงตัวกรองสี
  let h = '<div class="cgf-sec">' + title + '</div>';
  arr.forEach(function(it, i) {
    const name = CGF_COLOR_NAMES[it.c] || it.c;
    const sw = type === 'bg'
      ? '<span class="cgf-sw" style="background:' + (it.c || '#ffffff') + '"></span>'
      : '<span class="cgf-sw" style="color:' + it.c + '">A</span>';
    h += '<label class="cgf-item"><input type="checkbox" data-t="' + type + '" data-i="' + i + '"' + (st[type][it.c] ? ' checked' : '') +
      ' onchange="cgfColor(this)">' + sw + '<span class="t">' + _escHtml(name) + '</span><small>' + fmt(it.n) + '</small></label>';
  });
  return h;
}

function cgfRender() {
  const k = _cgfOpen; if (!k) return;
  const st = CGF[k] || { sel: {}, bg: {}, fg: {} }, q = _cgfQ.toLowerCase(), list = $('cgfList');
  let h = '';
  if (!q) h += _cgfColorSection('bg', _cgfBgs, st, 'กรองตามสีพื้นหลัง') + _cgfColorSection('fg', _cgfFgs, st, 'กรองตามสีตัวอักษร');
  let vals = '', shown = 0, total = 0;
  for (let i = 0; i < _cgfVals.length; i++) {
    const it = _cgfVals[i], lbl = it.c === '' ? CGF_BLANK : it.c;
    if (q && lbl.toLowerCase().indexOf(q) === -1) continue;
    total++;
    if (shown >= CGF_SHOW_MAX) continue;
    shown++;
    vals += '<label class="cgf-item"><input type="checkbox" data-i="' + i + '"' + (st.sel[it.c] ? ' checked' : '') +
      ' onchange="cgfChange(this)"><span class="t">' + _escHtml(lbl) + '</span><small>' + fmt(it.n) + '</small></label>';
  }
  if (h && vals) h += '<div class="cgf-sec">กรองตามค่า</div>';
  h += vals;
  if (!shown && !h) h = '<div class="cgf-empty">ไม่พบรายการ</div>';
  else if (total > shown) h += '<div class="cgf-note">แสดง ' + fmt(shown) + ' จาก ' + fmt(total) + ' รายการ พิมพ์ค้นหาเพื่อเจาะจง</div>';
  const top = list.scrollTop;
  list.innerHTML = h;
  list.scrollTop = top;
}

function _cgfEnsure(k) { return CGF[k] || (CGF[k] = { sel: {}, bg: {}, fg: {} }); }

function _cgfSync(k) {
  const st = CGF[k];
  if (st && !_hasKeys(st.sel) && !_hasKeys(st.bg) && !_hasKeys(st.fg)) delete CGF[k];
  _cgfMark();
  cgApplyFilter();
}

function cgfChange(inp) {
  const k = _cgfOpen, it = _cgfVals[parseInt(inp.dataset.i, 10)];
  if (!k || !it) return;
  const st = _cgfEnsure(k);
  if (inp.checked) st.sel[it.c] = true; else delete st.sel[it.c];
  _cgfSync(k);
}

function cgfColor(inp) {
  const k = _cgfOpen; if (!k) return;
  const arr = inp.dataset.t === 'bg' ? _cgfBgs : _cgfFgs;
  const it = arr[parseInt(inp.dataset.i, 10)];
  if (!it) return;
  const st = _cgfEnsure(k);
  if (inp.checked) st[inp.dataset.t][it.c] = true; else delete st[inp.dataset.t][it.c];
  _cgfSync(k);
}

function cgfSearch(v) { _cgfQ = String(v || '').trim(); cgfRender(); }

// มีคำค้นอยู่ = เลือกเฉพาะรายการที่ค้นเจอ (เพิ่มจากที่เลือกไว้แล้ว)
function cgfAll() {
  const k = _cgfOpen; if (!k) return;
  const q = _cgfQ.toLowerCase(), st = _cgfEnsure(k);
  for (let i = 0; i < _cgfVals.length; i++) {
    const c = _cgfVals[i].c, lbl = c === '' ? CGF_BLANK : c;
    if (!q || lbl.toLowerCase().indexOf(q) !== -1) st.sel[c] = true;
  }
  cgfRender();
  _cgfSync(k);
}

function cgfClear() {
  const k = _cgfOpen; if (!k) return;
  delete CGF[k];
  cgfRender();
  _cgfSync(k);
}

// คลิกนอกกล่อง = ปิด
document.addEventListener('mousedown', function(e) {
  if (e.target.closest && (e.target.closest('#cgfPanel') || e.target.closest('.cgf-btn'))) return;
  cgfClose();
});

// กลุ่มที่แสดงในตารางข้อมูลกลุ่มตัด: ไม่แสดงกลุ่มที่ จดแจ้ง (ตัน/หีบ) และ จดแจ้ง (ไร่/หีบ) เป็น 0 ทั้งคู่
function _cgRowVisible(r) {
  return (Number(r.regTon) || 0) !== 0 || (Number(r.regRai) || 0) !== 0;
}

// การ์ดสรุป 3 หมวด (คำนวณจากทุกกลุ่มตามตัวกรอง)
function cgRenderSummary(rows) {
  const _has = v => v !== '' && v != null;
  let totRegRai = 0, totRegTon = 0, totQueue = 0;
  let totConfirmed = 0, totAssigned = 0;
  let totDecPlanRai = 0, totDecRegRai = 0, totDecPlanTon = 0, totDecRegTon = 0;
  for (let i = 0; i < rows.length; i++) {
    const r = rows[i];
    if (r.regRai !== '') totRegRai += r.regRai || 0;
    if (r.regTon !== '') totRegTon += r.regTon || 0;
    if (r.queue  !== '') totQueue  += r.queue  || 0;
    if (_has(r.decPlanRai)) totDecPlanRai += r.decPlanRai || 0;
    if (_has(r.decRegRai))  totDecRegRai  += r.decRegRai  || 0;
    if (_has(r.decPlanTon)) totDecPlanTon += r.decPlanTon || 0;
    if (_has(r.decRegTon))  totDecRegTon  += r.decRegTon  || 0;
    // ทั้งหมด (กลุ่ม) = กลุ่มที่ "ยืนยันแล้ว"
    // จัดแล้ว (กลุ่ม) = กลุ่มที่ยืนยันแล้ว และมีการบันทึก "จดแจ้งแผนตัดธันวาคม (ไร่)" (> 0) นับกลุ่มละ 1
    if (r.verify === 'ยืนยันแล้ว') {
      totConfirmed++;
      if ((Number(r.decRegRai) || 0) > 0) totAssigned++;
    }
  }
  const totNotAssigned = Math.max(0, totConfirmed - totAssigned);
  const cls = 'cg-inline-pct';
  // หมวด 1
  setText('cgTotalRegRai', fmt(totRegRai));
  setText('cgTotalRegTon', fmt(totRegTon));
  setText('cgTotalQueue',  fmt(totQueue));
  // หมวด 2
  setText('cgTotalConfirmed', totConfirmed.toLocaleString('th-TH'));
  setValPct('cgTotalAssigned',    totAssigned.toLocaleString('th-TH'),    totAssigned,    totConfirmed, cls);
  setValPct('cgTotalNotAssigned', totNotAssigned.toLocaleString('th-TH'), totNotAssigned, totConfirmed, cls);
  // หมวด 3
  setText('cgTotalDecRai', fmt(totDecPlanRai));
  setText('cgTotalDecTon', fmt(totDecPlanTon));
  setValPct('cgTotalDecReg',    fmt(totDecRegRai), totDecRegRai, totDecPlanRai, cls);
  setValPct('cgTotalDecRegTon', fmt(totDecRegTon), totDecRegTon, totDecPlanTon, cls);
}

// จดแจ้ง < แผน = แดง / จดแจ้ง >= แผน = เขียว (ช่องว่างนับเป็น 0, ว่างทั้งคู่ = ไม่ใส่สี)
function _cgDecCls(plan, reg) {
  const has = (reg !== '' && reg != null) || (plan !== '' && plan != null);
  if (!has) return '';
  return (Number(reg) || 0) < (Number(plan) || 0) ? 'cg-dec-under' : 'cg-dec-ok';
}

// ตารางกลุ่มตัด 10 คอลัมน์
function cgRenderTable(allRows) {
  // ถ้าการ์ดสรุปมีปัญหา ไม่ให้กระทบการวาดตาราง
  try { cgRenderSummary(allRows); } catch (e) { console.error('cgRenderSummary', e); }
  const rows = allRows.filter(_cgRowVisible);
  $('cgTableCount').textContent = rows.length.toLocaleString('th-TH') + ' กลุ่ม';
  const tbody = $('cgTableBody');
  if (!rows.length) {
    tbody.innerHTML = '<tr><td colspan="' + CG_COLS + '" style="text-align:center;padding:20px">🔍 ไม่พบข้อมูลที่ตรงกับเงื่อนไข</td></tr>';
    return;
  }
  const fmtN = n => (n !== '' && n != null) ? fmt(Number(n)) : '-';
  const parts = new Array(rows.length);
  for (let i = 0; i < rows.length; i++) {
    const r = rows[i];
    const decRaiCls = _cgDecCls(r.decPlanRai, r.decRegRai);
    const decTonCls = _cgDecCls(r.decPlanTon, r.decRegTon);
    // จดแจ้งแผนตัดธันวาคม (ไร่) ไม่เป็น 0 -> เขียวทั้งแถว
    const rowCls = (Number(r.decRegRai) || 0) !== 0 ? ' class="row-decreg"' : '';
    parts[i] =
      '<tr'+rowCls+'>' +
      '<td><span class="cg-badge-zone">'+(r.zone||'-')+'</span></td>' +
      '<td style="color:#1a202c">'+(r.code||'-')+'</td>' +
      '<td title="'+_escAttr(r.name)+'">'+(r.name||'-')+'</td>' +
      '<td>'+(r.brand||'-')+'</td>' +
      '<td class="num">'+fmtN(r.regTon)+'</td>' +
      '<td class="num">'+fmtN(r.regRai)+'</td>' +
      '<td class="num">'+fmtN(r.decPlanRai)+'</td>' +
      '<td class="num '+decRaiCls+'">'+fmtN(r.decRegRai)+'</td>' +
      '<td class="num">'+fmtN(r.decPlanTon)+'</td>' +
      '<td class="num '+decTonCls+'">'+fmtN(r.decRegTon)+'</td>' +
      '<td>'+(r.readyDate||'-')+'</td>' +
      '<td style="text-align:left;max-width:260px;white-space:nowrap;overflow:hidden;text-overflow:ellipsis" title="'+_escAttr(r.note)+'">'+_escHtml(r.note||'-')+'</td>' +
      '</tr>';
  }
  tbody.innerHTML = parts.join('');
}

// ===== ผลตรวจแปลงรถตัดโรงงาน (GTT) : ใช้ไฮไลต์แถวแดงในตารางจดแจ้งฯ (ไม่มีหน้าจอแสดงผลแล้ว) =====
function loadInspectData(onLoaded, forceRefresh) {
  const done = function() { if (typeof onLoaded === 'function') onLoaded(); };
  google.script.run
    .withSuccessHandler(function(raw) {
      const data = _pj(raw);
      if (!data.error) { INSP_ALL_ROWS = data.rows || []; buildInspResultMap(); }
      done();
    })
    .withFailureHandler(done)
    .getInspectData(!!(forceRefresh || _forceInsp));
  _forceInsp = false;
}

function buildInspResultMap() {
  INSP_RESULT_MAP = {};
  for (let i = 0; i < INSP_ALL_ROWS.length; i++) {
    const r = INSP_ALL_ROWS[i];
    const key = String(r.plotNo || '').trim();
    if (!key) continue;
    INSP_RESULT_MAP[key] = r.result || '';
  }
}

// ===== DASHBOARD FILTERS =====
function dbGetFilterValues() {
  return {
    zone:    $('dbFilterZone').value,
    farmer:  $('dbFilterFarmer').value,
    cutType: $('dbFilterCutType').value,
    cgSearch:($('dbFilterCutGroup').value || '').trim().toLowerCase()
  };
}

function dbRowMatchesFilters(r, f, excludeKey) {
  if (excludeKey !== 'zone'   && f.zone   !== 'all' && r[F.ZONE]   !== f.zone)   return false;
  if (excludeKey !== 'farmer' && f.farmer !== 'all' && r[F.FARMER] !== f.farmer) return false;
  if (excludeKey !== 'cutType' && f.cutType !== 'all') {
    const ct = (r[F.CUTTYPE] || '').trim() || 'ยังไม่จัดคิว';
    if (ct !== f.cutType) return false;
  }
  if (f.cgSearch) {
    const code = (r[F.CUTGROUP]     || '').toLowerCase();
    const name = (r[F.CUTGROUPNAME] || '').toLowerCase();
    if (!code.includes(f.cgSearch) && !name.includes(f.cgSearch)) return false;
  }
  return true;
}

function dbUpdateFilterOptions() {
  const f = dbGetFilterValues();

  const zoneSet = {};
  ALL_ROWS.forEach(r => { if (r[F.ZONE] && dbRowMatchesFilters(r, f, 'zone')) zoneSet[r[F.ZONE]] = 1; });
  populateDrop('dbFilterZone', Object.keys(zoneSet).sort(zoneCompare), 'ทั้งหมด (เขต)');

  const farmerSet = {};
  ALL_ROWS.forEach(r => { if (r[F.FARMER] && dbRowMatchesFilters(r, f, 'farmer')) farmerSet[r[F.FARMER]] = 1; });
  populateDrop('dbFilterFarmer', Object.keys(farmerSet).sort(thSort), 'ทั้งหมด (นักเกษตร)');

  const ctSet = {};
  ALL_ROWS.forEach(r => {
    if (!dbRowMatchesFilters(r, f, 'cutType')) return;
    ctSet[(r[F.CUTTYPE] || '').trim() || 'ยังไม่จัดคิว'] = 1;
  });
  populateDrop('dbFilterCutType', Object.keys(ctSet).sort(thSort), 'ทั้งหมด (ประเภทการตัด)');
}

function applyDbFilters() {
  dbUpdateFilterOptions();
  const f = dbGetFilterValues();
  DB_ROWS = ALL_ROWS.filter(r => dbRowMatchesFilters(r, f, null));
  renderDashboard(DB_ROWS);
  renderDbFleetSummary();
}

function resetDbFilters() {
  $('dbFilterZone').value = 'all';
  $('dbFilterFarmer').value = 'all';
  $('dbFilterCutType').value = 'all';
  $('dbFilterCutGroup').value = '';
  applyDbFilters();
}

// ===== ตัวกรองรายคอลัมน์ (เหมือน Excel) =====
// COLF[คีย์คอลัมน์] = { sel: {ค่า:true} } มีเฉพาะคอลัมน์ที่กำลังกรอง / ติ๊กค่าไหน = แสดงเฉพาะค่านั้น / ไม่ติ๊กเลย = ไม่กรอง
const _fmt1 = n => Number(n).toLocaleString('th-TH', { minimumFractionDigits: 1, maximumFractionDigits: 1 });
const COLF_DEFS = [
  { k:'farmer',       label:'ชื่อนักเกษตร',   th:' style="min-width:160px;white-space:nowrap"', get: r => r[F.FARMER] || '' },
  { k:'zone',         label:'โซน',            th:'', get: r => r[F.ZONE] || '' },
  { k:'caneType',     label:'ชื่อประเภทอ้อย', th:'', get: r => r[F.CANETYPE] || '' },
  { k:'area',         label:'พื้นที่ (ไร่)',  th:' style="text-align:right"', num:true, get: r => (r[F.AREA] || 0) > 0 ? _fmt1(r[F.AREA]) : '' },
  { k:'plot',         label:'รหัสแปลง',       th:'', get: r => r[F.PLOTINTECH] || '' },
  { k:'owner',        label:'ชื่อชาวไร่',     th:' style="min-width:170px;white-space:nowrap"', get: r => r[F.OWNER] || '' },
  { k:'ton',          label:'ตัน STD',        th:' style="text-align:right"', num:true, get: r => (r[F.EST_TON] || 0) > 0 ? fmt(r[F.EST_TON]) : '' },
  { k:'tpr',          label:'ตัน/ไร่',        th:' style="text-align:right"', num:true, get: r => (r[F.TON_PER_RAI] || 0) > 0 ? _fmt1(r[F.TON_PER_RAI]) : '' },
  { k:'cutType',      label:'ประเภทการตัด',   th:'', get: r => _rowCutType(r).trim() },
  { k:'factory',      label:'โรงงาน',         th:'', get: r => r[F.FACTORY] || '' },
  { k:'cutGroup',     label:'กลุ่มตัด',       th:'', get: r => r[F.CUTGROUP] || '' },
  { k:'cutGroupName', label:'ชื่อกลุ่มตัด',   th:'', get: r => r[F.CUTGROUPNAME] || '' },
  { k:'age',          label:'อายุอ้อย (วัน)', th:' style="text-align:right"', num:true, get: r => (r[F.CANE_AGE] !== '' && r[F.CANE_AGE] != null) ? fmt(r[F.CANE_AGE]) : '' },
  { k:'order',        label:'ลำดับการตัด',    th:' style="text-align:center"', num:true, get: r => _hasCutOrder(r) ? String(r[F.CUT_ORDER]).trim() : '' }
];
const COLF_BY = {}; COLF_DEFS.forEach(d => { COLF_BY[d.k] = d; });
const COLF = {};
const COLF_SHOW_MAX = 300;   // แสดงรายการในกล่องสูงสุดเท่านี้ (รหัสแปลงมีหลายหมื่นค่า) พิมพ์ค้นหาเพื่อเจาะจง
const COLF_BLANK = '(ว่าง)';
let _colfOpen = null, _colfQ = '', _colfVals = [];

function _colfActive() { for (const k in COLF) return true; return false; }

// แถวนี้ผ่านตัวกรองรายคอลัมน์หรือไม่ (exKey = 'c:คีย์' ข้ามคอลัมน์นั้น ใช้ตอนหารายการค่าของคอลัมน์เอง)
function _colfMatch(r, exKey) {
  for (const k in COLF) {
    if (exKey === 'c:' + k) continue;
    if (!COLF[k].sel[COLF_BY[k].get(r)]) return false;
  }
  return true;
}

function buildTableHead() {
  $('tblHeadRow').innerHTML = COLF_DEFS.map(d =>
    '<th' + d.th + '>' + d.label + '<button type="button" class="colf-btn" data-k="' + d.k + '" title="กรอง ' + d.label +
    '" onclick="colfOpen(this.dataset.k, this)">▾</button></th>').join('');
}

function _colfMarkButtons() {
  document.querySelectorAll('#tblHeadRow .colf-btn').forEach(b => b.classList.toggle('active', !!COLF[b.dataset.k]));
}

function _colfCompute(k) {
  const d = COLF_BY[k], f = tableGetFilterValues(), cnt = {};
  for (let i = 0; i < ALL_ROWS.length; i++) {
    const r = ALL_ROWS[i];
    if (!tableRowMatchesFilters(r, f, 'c:' + k)) continue;
    const v = d.get(r);
    cnt[v] = (cnt[v] || 0) + 1;
  }
  const num = x => parseFloat(String(x).replace(/,/g, ''));
  _colfVals = Object.keys(cnt).map(v => ({ v: v, n: cnt[v] })).sort(function(a, b) {
    if (a.v === '') return 1;
    if (b.v === '') return -1;
    return d.num ? num(a.v) - num(b.v) : thSort(a.v, b.v);
  });
}

function colfOpen(k, btn) {
  const p = $('colfPanel');
  if (_colfOpen === k && p.style.display !== 'none') { colfClose(); return; }
  _colfOpen = k; _colfQ = '';
  const rc = btn.getBoundingClientRect();
  p.style.top  = (rc.bottom + 4) + 'px';
  p.style.left = Math.max(8, Math.min(rc.left, window.innerWidth - 276)) + 'px';
  p.innerHTML =
    '<div class="colf-head"><a onclick="colfAll()">เลือกทั้งหมด</a><a onclick="colfClear()">ล้างการเลือก</a></div>' +
    '<div class="colf-search"><input type="text" id="colfSearch" placeholder="ค้นหา..." oninput="colfSearch(this.value)"></div>' +
    '<div class="colf-list" id="colfList"></div>';
  p.style.display = '';
  _colfCompute(k);
  colfRenderList();
  setTimeout(function() { const i = $('colfSearch'); if (i) i.focus(); }, 30);
}

function colfClose() {
  const p = $('colfPanel');
  if (p) p.style.display = 'none';
  _colfOpen = null;
}

function colfRenderList() {
  const k = _colfOpen; if (!k) return;
  const st = COLF[k], q = _colfQ.toLowerCase(), list = $('colfList');
  let h = '', shown = 0, total = 0;
  for (let i = 0; i < _colfVals.length; i++) {
    const it = _colfVals[i], lbl = it.v === '' ? COLF_BLANK : it.v;
    if (q && lbl.toLowerCase().indexOf(q) === -1) continue;
    total++;
    if (shown >= COLF_SHOW_MAX) continue;
    shown++;
    h += '<label class="colf-item"><input type="checkbox" data-i="' + i + '"' + (st && st.sel[it.v] ? ' checked' : '') +
      ' onchange="colfChange(this)"><span>' + _escHtml(lbl) + '</span><small style="margin-left:auto;color:#a0aec0">' + fmt(it.n) + '</small></label>';
  }
  if (!shown) h = '<div class="colf-empty">ไม่พบรายการ</div>';
  else if (total > shown) h += '<div class="colf-note">แสดง ' + fmt(shown) + ' จาก ' + fmt(total) + ' รายการ พิมพ์ค้นหาเพื่อเจาะจง</div>';
  const top = list.scrollTop;
  list.innerHTML = h;
  list.scrollTop = top;
}

function _colfSync(k) {
  if (COLF[k] && !Object.keys(COLF[k].sel).length) delete COLF[k];
  _colfMarkButtons();
  applyFilters();
}

function colfChange(inp) {
  const k = _colfOpen, it = _colfVals[parseInt(inp.dataset.i, 10)];
  if (!k || !it) return;
  const st = COLF[k] || (COLF[k] = { sel: {} });
  if (inp.checked) st.sel[it.v] = true; else delete st.sel[it.v];
  _colfSync(k);
}

function colfSearch(v) { _colfQ = String(v || '').trim(); colfRenderList(); }

// มีคำค้นอยู่ = เลือกเฉพาะรายการที่ค้นเจอ (เพิ่มจากที่เลือกไว้แล้ว)
function colfAll() {
  const k = _colfOpen; if (!k) return;
  const q = _colfQ.toLowerCase();
  const st = COLF[k] || (COLF[k] = { sel: {} });
  for (let i = 0; i < _colfVals.length; i++) {
    const v = _colfVals[i].v, lbl = v === '' ? COLF_BLANK : v;
    if (!q || lbl.toLowerCase().indexOf(q) !== -1) st.sel[v] = true;
  }
  colfRenderList();
  _colfSync(k);
}

function colfClear() {
  const k = _colfOpen; if (!k) return;
  delete COLF[k];
  colfRenderList();
  _colfSync(k);
}

// คลิกนอกกล่อง = ปิด
document.addEventListener('mousedown', function(e) {
  if (e.target.closest && (e.target.closest('#colfPanel') || e.target.closest('.colf-btn'))) return;
  colfClose();
});

// ===== FILTERS (TABLE) =====
let _filterDebounceTimer = null;
function onFilterSearchInput() {
  const tableCountEl = $('tableCount');
  if (tableCountEl) tableCountEl.textContent = '⏳ กำลังค้นหาข้อมูล...';
  clearTimeout(_filterDebounceTimer);
  _filterDebounceTimer = setTimeout(applyFilters, 300);
}

function tableGetFilterValues() {
  return {
    zone:        $('filterZone').value,
    farmer:      $('filterFarmer').value,
    ownerSearch: ($('filterOwnerSearch').value || '').trim().toLowerCase(),
    plotSearch:  ($('filterPlotCodeSearch').value || '').trim().toLowerCase(),
    cgSearch:    ($('filterCutGroupSearch').value || '').trim().toLowerCase(),
    ctType:      $('filterCutType').value,
    criteria:    $('filterCriteria').value,
    ordered:     $('filterOrdered').value,
  };
}

// ประเภทการตัดของแปลง: ถ้ามีกลุ่มตัดใช้สถานะของกลุ่ม ไม่มีก็ใช้ประเภทการตัดในแถว
function _rowCutType(r) {
  const grp = (r[F.CUTGROUP] || '').trim();
  return grp ? (STATUS_MAP[grp] || '') : (r[F.CUTTYPE] || '');
}

function tableRowMatchesFilters(r, f, excludeKey) {
  if (!_colfMatch(r, excludeKey)) return false;
  if (excludeKey !== 'zone'   && f.zone   !== 'all' && r[F.ZONE]   !== f.zone)   return false;
  if (excludeKey !== 'farmer' && f.farmer !== 'all' && r[F.FARMER] !== f.farmer) return false;
  if (f.ownerSearch && !(r[F.OWNER] || '').toLowerCase().includes(f.ownerSearch)) return false;
  if (f.plotSearch && !(r[F.PLOTINTECH] || '').toLowerCase().includes(f.plotSearch)) return false;

  // กรองอายุอ้อย: ถึงเกณฑ์ / ไม่ถึงเกณฑ์ (ค่าว่างนับเป็นไม่ถึงเกณฑ์)
  if (excludeKey !== 'criteria' && f.criteria !== 'all') {
    const met = _isCriteriaMet(r[F.CUT_CRITERIA]);
    if (f.criteria === 'met' ? !met : met) return false;
  }

  // กรองลำดับการตัด: จัดลำดับแล้ว / ยังไม่จัดลำดับ
  if (excludeKey !== 'ordered' && f.ordered !== 'all') {
    const has = _hasCutOrder(r);
    if (f.ordered === 'ordered' ? !has : has) return false;
  }

  if (f.cgSearch) {
    const grpCode = (r[F.CUTGROUP]     || '').trim().toLowerCase();
    const grpName = (r[F.CUTGROUPNAME] || '').trim().toLowerCase();
    if (!grpCode.includes(f.cgSearch) && !grpName.includes(f.cgSearch)) return false;
  }

  if (excludeKey !== 'ctType' && f.ctType !== 'all') {
    const ctValNorm = _rowCutType(r).trim() || 'ยังไม่จัดคิว';
    if (ctValNorm !== f.ctType) return false;
  }
  return true;
}

// ซ่อนตัวเลือกที่ไม่มีข้อมูลเหลือตามตัวกรองอื่น
function _syncFilterOptions(selId, counts) {
  const el = $(selId);
  Array.from(el.options).forEach(opt => {
    if (opt.value === 'all') return;
    const has = counts[opt.value] > 0;
    opt.disabled = !has;
    opt.hidden   = !has;
  });
  if (el.value !== 'all' && !counts[el.value]) el.value = 'all';
}

function tableUpdateFilterOptions() {
  const f = tableGetFilterValues();

  const zoneSet = {};
  ALL_ROWS.forEach(r => { if (r[F.ZONE] && tableRowMatchesFilters(r, f, 'zone')) zoneSet[r[F.ZONE]] = 1; });
  populateDrop('filterZone', Object.keys(zoneSet).sort(zoneCompare), 'ทั้งหมด (เขต)');

  const farmerSet = {};
  ALL_ROWS.forEach(r => { if (r[F.FARMER] && tableRowMatchesFilters(r, f, 'farmer')) farmerSet[r[F.FARMER]] = 1; });
  populateDrop('filterFarmer', Object.keys(farmerSet).sort(thSort), 'ทั้งหมด (นักเกษตร)');

  const ctSet = {};
  ALL_ROWS.forEach(r => {
    if (!tableRowMatchesFilters(r, f, 'ctType')) return;
    ctSet[_rowCutType(r).trim() || 'ยังไม่จัดคิว'] = 1;
  });
  populateDrop('filterCutType', Object.keys(ctSet).sort(thSort), 'ทั้งหมด (ประเภทการตัด)');

  const critCounts = { met: 0, notmet: 0 };
  ALL_ROWS.forEach(r => {
    if (!tableRowMatchesFilters(r, f, 'criteria')) return;
    critCounts[_isCriteriaMet(r[F.CUT_CRITERIA]) ? 'met' : 'notmet']++;
  });
  _syncFilterOptions('filterCriteria', critCounts);

  const ordCounts = { ordered: 0, notordered: 0 };
  ALL_ROWS.forEach(r => {
    if (!tableRowMatchesFilters(r, f, 'ordered')) return;
    ordCounts[_hasCutOrder(r) ? 'ordered' : 'notordered']++;
  });
  _syncFilterOptions('filterOrdered', ordCounts);
}

function applyFilters() {
  tableUpdateFilterOptions();
  const f = tableGetFilterValues();
  CURRENT_ROWS = ALL_ROWS.filter(r => tableRowMatchesFilters(r, f, null));
  renderTable(CURRENT_ROWS);
  renderCodeSummaries(); // บล็อกตามเขต / ตัวกรองที่เลือก
}

function resetFilters() {
  $('filterZone').value = 'all';
  $('filterFarmer').value = 'all';
  $('filterOwnerSearch').value = '';
  $('filterPlotCodeSearch').value = '';
  $('filterCutGroupSearch').value = '';
  $('filterCutType').value = 'all';
  $('filterCriteria').value = 'all';
  $('filterOrdered').value  = 'all';
  Object.keys(COLF).forEach(k => { delete COLF[k]; });
  _colfMarkButtons(); colfClose();
  applyFilters();
}

// ===== RENDER (แดชบอร์ด) =====
// ตั้งค่ากราฟแต่ละโหมดไว้ที่เดียว แล้วใช้ renderCutPie / renderCutBar ร่วมกัน
const CUT_METRICS = {
  plot: { valueFn: r => 1,                  unit: 'แปลง', yTitle: 'จำนวนแปลง', step: 200 },
  ton:  { valueFn: r => r[F.EST_TON] || 0,  unit: 'ตัน',  yTitle: 'ตัน STD' },
  rai:  { valueFn: r => r[F.AREA] || 0,     unit: 'ไร่',  yTitle: 'ไร่' }
};
const DB_CHART_MODES = {
  ton:    { metric:'ton',  pie:'pieTonChart',       bar:'barTonChart',       legend:'pieTonLegend' },
  plot:   { metric:'plot', pie:'pieChart',          bar:'barChart',          legend:'pieLegend' },
  rai:    { metric:'rai',  pie:'pieRaiChart',       bar:'barRaiChart',       legend:'pieRaiLegend' },
  korat:  { metric:'ton',  pie:'pieTonKoratChart',  bar:'barTonKoratChart',  legend:'pieTonKoratLegend',  factory:'โรงงานครบุรี' },
  sikhio: { metric:'ton',  pie:'pieTonSikhioChart', bar:'barTonSikhioChart', legend:'pieTonSikhioLegend', factory:'โรงงานสีคิ้ว' }
};
const PIE_LEGEND_ORDER = ['รถตัดโรงงาน','รถตัดผู้รับเหมา','รถตัดชาวไร่','แรงงาน','ลานขนถ่ายเขต 1','ลานขนถ่ายเขต 8','รถตัด Cluster','ยังไม่จัดคิว'];
const YARD_GROUPS = ['ลานขนถ่ายเขต 1', 'ลานขนถ่ายเขต 8'];

function _renderDbChartMode(mode, rows) {
  if (mode === 'order') {
    renderOrderMode(rows);
  } else if (mode === 'group') {
    renderGroupMode(rows);
  } else {
    const cfg = DB_CHART_MODES[mode];
    const data = cfg.factory ? rows.filter(r => (r[F.FACTORY] || '').trim() === cfg.factory) : rows;
    renderCutPie(data, cfg);
    renderCutBar(data, cfg);
  }
  _dbRenderedModes[mode] = true;
}

function renderDashboard(rows) {
  // ★ [แก้ไข ข้อ 78] Chart.js โหลดแบบ defer และหน้าเว็บเริ่มโหลดข้อมูลทันที -> ถ้าไลบรารียังมาไม่ถึงให้รอแล้ววาดใหม่
  if (typeof Chart === 'undefined' || typeof ChartDataLabels === 'undefined') {
    setTimeout(function() { renderDashboard(rows); }, 80);
    return;
  }
  renderDonuts(rows); renderCutTypeDonut(rows);
  _dbRenderedModes = {};
  _renderDbChartMode(dbChartMode, rows);
}

function drawDonut(canvasId, data, colors) {
  return mkChart(canvasId, {
    type: 'doughnut',
    data: { datasets: [{ data, backgroundColor: colors, borderWidth: 0, hoverOffset: 4 }] },
    options: { responsive: false, cutout: '72%', animation: { duration: 400 },
      plugins: { legend: { display: false }, tooltip: { callbacks: { label: ctx => ' ' + ctx.raw.toLocaleString('th-TH') } } } }
  });
}

function renderDonuts(rows) {
  let totalArea = 0, assignedArea = 0, assignedPlots = 0;
  for (let i = 0; i < rows.length; i++) {
    const r = rows[i];
    totalArea += r[F.AREA];
    if ((r[F.CUTTYPE] || '').trim()) {
      assignedArea += r[F.AREA];
      assignedPlots++;
    }
  }
  const unassignedArea  = totalArea - assignedArea;
  const unassignedPlots = rows.length - assignedPlots;

  const plotPct = rows.length > 0 ? Math.round(assignedPlots / rows.length * 100) : 0;
  const areaPct = totalArea  > 0  ? Math.round(assignedArea  / totalArea   * 100) : 0;

  setText('donutPlotPct',        plotPct + '%');
  setText('donutPlotAssigned',   fmt(assignedPlots));
  setText('donutPlotUnassigned', fmt(unassignedPlots));
  setText('donutPlotTotal',      fmt(rows.length));
  drawDonut('donutPlotChart', [assignedPlots, unassignedPlots], ['#1b4332','#b7e4c7']);

  setText('donutAreaPct',        areaPct + '%');
  setText('donutAreaAssigned',   fmt(assignedArea));
  setText('donutAreaUnassigned', fmt(unassignedArea));
  setText('donutAreaTotal',      fmt(totalArea));
  drawDonut('donutAreaChart', [assignedArea, unassignedArea], ['#c2410c','#fed7aa']);

  const ownerMap = {};
  for (let i = 0; i < rows.length; i++) {
    const r     = rows[i];
    const quota = r[F.PLOTCODE];
    if (!quota) continue;
    if (!ownerMap[quota]) ownerMap[quota] = { total: 0, assigned: 0 };
    ownerMap[quota].total++;
    if ((r[F.CUTTYPE] || '').trim()) ownerMap[quota].assigned++;
  }
  let ownerFull = 0, ownerPartial = 0, ownerNone = 0;
  for (const quota in ownerMap) {
    const o = ownerMap[quota];
    if (o.assigned === o.total)  ownerFull++;
    else if (o.assigned > 0)     ownerPartial++;
    else                         ownerNone++;
  }
  const totalOwners = ownerFull + ownerPartial + ownerNone;
  const ownerPct = totalOwners > 0 ? Math.round(ownerFull / totalOwners * 100) : 0;

  setText('donutOwnerPct',     ownerPct + '%');
  setText('donutOwnerFull',    fmt(ownerFull));
  setText('donutOwnerPartial', fmt(ownerPartial));
  setText('donutOwnerNone',    fmt(ownerNone));
  setText('donutOwnerTotal',   fmt(totalOwners));
  drawDonut('donutOwnerChart', [ownerFull, ownerPartial, ownerNone], ['#1e3a8a', '#60a5fa', '#bfdbfe']);
}

function renderCutTypeDonut(rows) {
  let totalTon = 0, assignedTon = 0;
  for (let i = 0; i < rows.length; i++) {
    const ton = rows[i][F.EST_TON] || 0;
    totalTon += ton;
    if ((rows[i][F.CUTTYPE] || '').trim()) assignedTon += ton;
  }
  const unassignedTon = totalTon - assignedTon;
  const pct = totalTon > 0 ? Math.round(assignedTon / totalTon * 100) : 0;

  setText('donutCutTypePct',        pct + '%');
  setText('donutCutTypeAssigned',   fmt(assignedTon));
  setText('donutCutTypeUnassigned', fmt(unassignedTon));
  setText('donutCutTypeTotal',      fmt(totalTon));
  drawDonut('donutCutTypeChart', [assignedTon, unassignedTon], ['#6b21a8','#e9d5ff']);
}

function isDashboardFiltered() {
  return $('dbFilterZone').value !== 'all' || $('dbFilterFarmer').value !== 'all';
}

function _orderedCutKeys(ctSet) {
  const ordered = PIE_LEGEND_ORDER.filter(k => ctSet[k]);
  Object.keys(ctSet).forEach(k => { if (!ordered.includes(k)) ordered.push(k); });
  return ordered;
}

// กราฟวงกลม (แปลง / ตัน STD / ไร่ / รายโรงงาน) พร้อมคำอธิบายสีด้านข้าง
function renderCutPie(rows, cfg) {
  const m = CUT_METRICS[cfg.metric];
  const sums = {};
  let total = 0;
  for (let i = 0; i < rows.length; i++) {
    const r  = rows[i];
    const gn = (r[F.CUTGROUPNAME] || '').trim();
    const k  = YARD_GROUPS.includes(gn) ? gn : (r[F.CUTTYPE] || 'ยังไม่จัดคิว');
    const v  = m.valueFn(r);
    sums[k] = (sums[k] || 0) + v;
    total += v;
  }
  const labels = _orderedCutKeys(sums);
  const values = labels.map(k => Math.round(sums[k] * 100) / 100);
  const colors = labels.map(l => CUT_COLORS[l] || '#9e9e9e');
  mkChart(cfg.pie, {
    type: 'pie',
    data: { labels, datasets: [{ data: values, backgroundColor: colors, borderWidth: 0 }] },
    options: {
      responsive: true, animation: { duration: 400 },
      plugins: {
        legend: { display: false },
        datalabels: { color: '#fff', font: { weight: 'bold', size: 13 }, anchor: 'center', align: 'center',
          formatter: v => total > 0 ? Math.round(v / total * 100) + '%' : '0%' },
        tooltip: { titleFont: { size: 15 }, bodyFont: { size: 14 },
          callbacks: { label: ctx => {
            const p = total > 0 ? ((ctx.raw / total) * 100).toFixed(1) : 0;
            return ' ' + ctx.label + ': ' + ctx.raw.toLocaleString('th-TH') + ' ' + m.unit + ' (' + p + '%)';
          }}}
      }
    }, plugins: [ChartDataLabels]
  });

  const legendEl = $(cfg.legend);
  if (cfg.factory && !labels.length) {
    legendEl.innerHTML = '<div style="color:#a0aec0;font-size:.85rem">ไม่มีข้อมูล</div>';
    return;
  }
  const items = labels.map((l, i) =>
    '<div class="pie-legend-item"><span class="pie-legend-dot" style="background:'+colors[i]+'"></span><span class="dl-txt">'+l+'</span><strong>'+fmt(values[i])+' '+m.unit+'</strong></div>'
  );
  if (cfg.factory) {
    items.push('<div class="dl-total" style="border-top:1px dashed #cbd5e0;padding-top:6px;margin-top:4px">รวมทั้งหมด <strong>'+fmt(total)+' '+m.unit+'</strong></div>');
  }
  legendEl.innerHTML = items.join('');
}

// กราฟแท่งซ้อน: รายเขต/โซน (ไม่กรอง) หรือรายนักเกษตร (เมื่อกรองเขต/นักเกษตร)
function renderCutBar(rows, cfg) {
  const m = CUT_METRICS[cfg.metric];
  const byFarmer = isDashboardFiltered();
  const keySet = {}, ctSet = {}, map = {};
  for (let i = 0; i < rows.length; i++) {
    const r = rows[i];
    const k = byFarmer ? (r[F.FARMER] || '-') : (r[F.ZONE] || '?');
    const c = r[F.CUTTYPE] || 'ยังไม่จัดคิว';
    keySet[k] = 1; ctSet[c] = 1;
    if (!map[k]) map[k] = {};
    map[k][c] = (map[k][c] || 0) + m.valueFn(r);
  }
  const keys = Object.keys(keySet).sort(byFarmer ? thSort : zoneCompare);
  const cuts = (byFarmer || cfg.factory) ? _orderedCutKeys(ctSet) : Object.keys(ctSet).sort(thSort);
  const datasets = cuts.map(ct => ({
    label: ct,
    data: keys.map(k => Math.round((map[k] && map[k][ct] || 0) * 10) / 10),
    backgroundColor: CUT_COLORS[ct] || '#9e9e9e',
    borderWidth: 0, barThickness: byFarmer ? 18 : 28, maxBarThickness: byFarmer ? 22 : 28
  }));
  const labelFont = { weight: 'bold', size: byFarmer ? 10 : 11 };
  const datalabels = byFarmer
    ? { color: '#1a202c', font: labelFont, anchor: 'end', align: 'top', offset: 2,
        formatter: v => v > 0 ? fmt(v) : null }
    : { color: '#1a202c', font: labelFont, anchor: 'end', align: 'top', offset: 3,
        formatter: (value, ctx) => {
          const ds = ctx.chart.data.datasets;
          const lastNZ = ds.reduce((last, d, i) => (d.data[ctx.dataIndex] || 0) > 0 ? i : last, -1);
          if (ctx.datasetIndex !== lastNZ) return null;
          const sum = ds.reduce((s, d) => s + (d.data[ctx.dataIndex] || 0), 0);
          return sum > 0 ? fmt(sum) : null;
        } };
  mkChart(cfg.bar, {
    type: 'bar', data: { labels: keys, datasets },
    options: {
      responsive: true, animation: { duration: 300 }, layout: { padding: { top: byFarmer ? 26 : 30 } },
      plugins: {
        legend: { position: 'top', align: 'end', labels: { font: { size: 13 }, boxWidth: 15, padding: 10 } },
        tooltip: { mode: 'index', intersect: false, titleFont: { size: 14 }, bodyFont: { size: 13 },
          callbacks: { label: ctx => ' ' + ctx.dataset.label + ': ' + ctx.raw.toLocaleString('th-TH') + ' ' + m.unit } },
        datalabels
      },
      scales: {
        x: { stacked: !byFarmer,
             ticks: byFarmer ? { font: { size: 11 }, autoSkip: false, maxRotation: 60, minRotation: 0 } : { font: { size: 12 } },
             title: { display: true, text: byFarmer ? 'ชื่อนักเกษตร' : 'เขต / โซน', font: { size: 13 } } },
        y: { stacked: !byFarmer, beginAtZero: true,
             ticks: (!byFarmer && m.step) ? { stepSize: m.step, font: { size: 12 } } : { font: { size: 12 } },
             title: { display: true, text: m.yTitle, font: { size: 13 } } }
      }
    }, plugins: [ChartDataLabels]
  });
}

// โหมด "จัดลำดับตัด": แปลงที่ถึงเกณฑ์ (คอลัมน์ T) แยกจัดลำดับแล้ว / ยังไม่จัดลำดับ (คอลัมน์ U)
function renderOrderMode(rows) {
  const met = [];
  const done = { n: 0, area: 0, ton: 0 }, todo = { n: 0, area: 0, ton: 0 };
  for (let i = 0; i < rows.length; i++) {
    const r = rows[i];
    if (!_isCriteriaMet(r[F.CUT_CRITERIA])) continue;
    met.push(r);
    const g = _hasCutOrder(r) ? done : todo;
    g.n++; g.area += r[F.AREA] || 0; g.ton += r[F.EST_TON] || 0;
  }
  const nMet = met.length, areaMet = done.area + todo.area, tonMet = done.ton + todo.ton;

  // บล็อกสรุป
  setText('ordMetPlots', fmt(nMet));
  setText('ordMetArea',  fmt(areaMet));
  setValPct('ordDonePlots', fmt(done.n), done.n, nMet, 'cg-scard-pct');
  setValPct('ordTodoPlots', fmt(todo.n), todo.n, nMet, 'cg-scard-pct');

  // กราฟวงกลม
  const labels = ['จัดลำดับแล้ว', 'ยังไม่จัดลำดับ'];
  const colors = ['#27ae60', '#a0aec0'];
  mkChart('pieOrderChart', {
    type: 'pie',
    data: { labels, datasets: [{ data: [done.n, todo.n], backgroundColor: colors, borderWidth: 0 }] },
    options: {
      responsive: true, animation: { duration: 400 },
      plugins: {
        legend: { display: false },
        datalabels: { color: '#fff', font: { weight: 'bold', size: 13 }, anchor: 'center', align: 'center',
          formatter: v => (nMet > 0 && v > 0) ? Math.round(v / nMet * 100) + '%' : null },
        tooltip: { titleFont: { size: 15 }, bodyFont: { size: 14 },
          callbacks: { label: ctx => ' ' + ctx.label + ': ' + ctx.raw.toLocaleString('th-TH') + ' แปลง (' + pctStr(ctx.raw, nMet) + ')' } }
      }
    }, plugins: [ChartDataLabels]
  });

  const groups = [done, todo];
  const items = labels.map((l, i) =>
    '<div class="pie-legend-item"><span class="pie-legend-dot" style="background:' + colors[i] + '"></span><span class="dl-txt">' + l + '</span><strong>' + fmt(groups[i].n) + ' แปลง</strong></div>' +
    '<div style="margin:-4px 0 2px 21px;font-size:.8rem;color:#4a5568;font-weight:600">' + fmt(groups[i].area) + ' ไร่ · ' + fmt(groups[i].ton) + ' ตัน STD</div>'
  );
  items.push('<div class="dl-total" style="border-top:1px dashed #cbd5e0;padding-top:6px;margin-top:4px">รวมถึงเกณฑ์ <strong>' + fmt(nMet) + ' แปลง</strong>' +
    '<div style="font-size:.8rem;color:#4a5568;font-weight:600;margin-top:2px">' + fmt(areaMet) + ' ไร่ · ' + fmt(tonMet) + ' ตัน STD</div></div>');
  $('pieOrderLegend').innerHTML = items.join('');

  _renderOrderBar(met);
}
// ===== โหมด "จัดลำดับตัด(กลุ่ม)" : แผนตัดเดือนธันวาคม (กลุ่มตัด) =====
// กลุ่มที่นับ = Code!AF แสดงข้อมูล = 1 และ ยืนยันแล้ว / จัดแล้ว = จดแจ้งแผนตัดธันวาคม (ไร่) > 0
function dbGroupRows(rows) {
  const f = dbGetFilterValues();
  let codeSet = null;
  // มีตัวกรองนักเกษตร / ประเภทการตัด / ค้นหากลุ่มตัด -> ใช้เฉพาะกลุ่มที่มีแปลงอยู่ในผลการกรอง
  if (f.farmer !== 'all' || f.cutType !== 'all' || f.cgSearch) {
    codeSet = {};
    for (let i = 0; i < rows.length; i++) {
      const c = String(rows[i][F.CUTGROUP] || '').trim();
      if (c) codeSet[c] = 1;
    }
  }
  return CG_ALL_ROWS.filter(r => {
    if (!r.show || r.verify !== 'ยืนยันแล้ว') return false;
    if (codeSet) return !!codeSet[String(r.code || '').trim()];
    return f.zone === 'all' || String(r.zone) === f.zone;
  });
}

function renderGroupMode(rows) {
  if (!CG_ALL_ROWS.length) { // ชีต Code ยังโหลดไม่เสร็จ -> _applyCodeData จะวาดให้เมื่อมาถึง
    ['grpTotal','grpDone','grpTodo'].forEach(id => setText(id, '⏳'));
    return;
  }
  const gs = dbGroupRows(rows);
  let done = 0, todo = 0;
  const byZone = {};
  for (let i = 0; i < gs.length; i++) {
    const r = gs[i], z = r.zone || '?';
    const m = byZone[z] || (byZone[z] = { done: 0, todo: 0 });
    if ((Number(r.decRegRai) || 0) > 0) { done++; m.done++; } else { todo++; m.todo++; }
  }
  const n = done + todo;

  setText('grpTotal', fmt(n));
  setValPct('grpDone', fmt(done), done, n, 'cg-scard-pct');
  setValPct('grpTodo', fmt(todo), todo, n, 'cg-scard-pct');

  const labels = ['จัดแล้ว (กลุ่ม)', 'ยังไม่จัด (กลุ่ม)'];
  const colors = ['#1e3a8a', '#a0aec0'];
  mkChart('pieGroupChart', {
    type: 'pie',
    data: { labels, datasets: [{ data: [done, todo], backgroundColor: colors, borderWidth: 0 }] },
    options: {
      responsive: true, animation: { duration: 400 },
      plugins: {
        legend: { display: false },
        datalabels: { color: '#fff', font: { weight: 'bold', size: 13 }, anchor: 'center', align: 'center',
          formatter: v => (n > 0 && v > 0) ? Math.round(v / n * 100) + '%' : null },
        tooltip: { titleFont: { size: 15 }, bodyFont: { size: 14 },
          callbacks: { label: ctx => ' ' + ctx.label + ': ' + ctx.raw.toLocaleString('th-TH') + ' กลุ่ม (' + pctStr(ctx.raw, n) + ')' } }
      }
    }, plugins: [ChartDataLabels]
  });

  const vals = [done, todo];
  const items = labels.map((l, i) =>
    '<div class="pie-legend-item"><span class="pie-legend-dot" style="background:' + colors[i] + '"></span><span class="dl-txt">' + l + '</span><strong>' + fmt(vals[i]) + ' กลุ่ม</strong></div>');
  items.push('<div class="dl-total" style="border-top:1px dashed #cbd5e0;padding-top:6px;margin-top:4px">ทั้งหมด <strong>' + fmt(n) + ' กลุ่ม</strong></div>');
  $('pieGroupLegend').innerHTML = items.join('');

  const keys = Object.keys(byZone).sort(zoneCompare);
  const datasets = [
    { label: 'จัดแล้ว (กลุ่ม)',   data: keys.map(k => byZone[k].done), backgroundColor: '#1e3a8a', borderWidth: 0, barThickness: 28, maxBarThickness: 28 },
    { label: 'ยังไม่จัด (กลุ่ม)', data: keys.map(k => byZone[k].todo), backgroundColor: '#a0aec0', borderWidth: 0, barThickness: 28, maxBarThickness: 28 }
  ];
  mkChart('barGroupChart', {
    type: 'bar', data: { labels: keys, datasets },
    options: {
      responsive: true, animation: { duration: 300 }, layout: { padding: { top: 30 } },
      plugins: {
        legend: { position: 'top', align: 'end', labels: { font: { size: 13 }, boxWidth: 15, padding: 10 } },
        tooltip: { mode: 'index', intersect: false, titleFont: { size: 14 }, bodyFont: { size: 13 },
          callbacks: { label: ctx => ' ' + ctx.dataset.label + ': ' + ctx.raw.toLocaleString('th-TH') + ' กลุ่ม' } },
        datalabels: { color: '#1a202c', font: { weight: 'bold', size: 11 }, anchor: 'end', align: 'top', offset: 3,
          formatter: (value, ctx) => {
            const ds = ctx.chart.data.datasets;
            const lastNZ = ds.reduce((last, d, i) => (d.data[ctx.dataIndex] || 0) > 0 ? i : last, -1);
            if (ctx.datasetIndex !== lastNZ) return null;
            const sum = ds.reduce((s, d) => s + (d.data[ctx.dataIndex] || 0), 0);
            return sum.toLocaleString('th-TH');
          } }
      },
      scales: {
        x: { stacked: true, ticks: { font: { size: 12 }, autoSkip: false },
             title: { display: true, text: 'เขต / โซน', font: { size: 13 } } },
        y: { stacked: true, beginAtZero: true, ticks: { font: { size: 12 } },
             title: { display: true, text: 'จำนวนกลุ่มตัด', font: { size: 13 } } }
      }
    }, plugins: [ChartDataLabels]
  });
}

// ===== TABLE =====
// วาดทีละ CHUNK แถว แล้วโหลดเพิ่มเมื่อเลื่อนใกล้ท้ายตาราง
function _renderOrderBar(met) {
  const byFarmer = isDashboardFiltered();
  const keySet = {}, map = {};
  for (let i = 0; i < met.length; i++) {
    const r = met[i];
    const k = byFarmer ? (r[F.FARMER] || '-') : (r[F.ZONE] || '?');
    keySet[k] = 1;
    if (!map[k]) map[k] = { done: 0, todo: 0 };
    map[k][_hasCutOrder(r) ? 'done' : 'todo']++;
  }
  const keys = Object.keys(keySet).sort(byFarmer ? thSort : zoneCompare);
  const thick = byFarmer ? 18 : 28;
  const datasets = [
    { label: 'จัดลำดับแล้ว',   data: keys.map(k => map[k].done), backgroundColor: '#27ae60', borderWidth: 0, barThickness: thick, maxBarThickness: thick },
    { label: 'ยังไม่จัดลำดับ', data: keys.map(k => map[k].todo), backgroundColor: '#a0aec0', borderWidth: 0, barThickness: thick, maxBarThickness: thick }
  ];
  mkChart('barOrderChart', {
    type: 'bar', data: { labels: keys, datasets },
    options: {
      responsive: true, animation: { duration: 300 }, layout: { padding: { top: 30 } },
      plugins: {
        legend: { position: 'top', align: 'end', labels: { font: { size: 13 }, boxWidth: 15, padding: 10 } },
        tooltip: { mode: 'index', intersect: false, titleFont: { size: 14 }, bodyFont: { size: 13 },
          callbacks: { label: ctx => ' ' + ctx.dataset.label + ': ' + ctx.raw.toLocaleString('th-TH') + ' แปลง' } },
        datalabels: { color: '#1a202c', font: { weight: 'bold', size: 11 }, anchor: 'end', align: 'top', offset: 3,
          formatter: (value, ctx) => {
            const ds = ctx.chart.data.datasets;
            const lastNZ = ds.reduce((last, d, i) => (d.data[ctx.dataIndex] || 0) > 0 ? i : last, -1);
            if (ctx.datasetIndex !== lastNZ) return null;
            const sum = ds.reduce((s, d) => s + (d.data[ctx.dataIndex] || 0), 0);
            return sum.toLocaleString('th-TH');
          } }
      },
      scales: {
        x: { stacked: true, ticks: { font: { size: byFarmer ? 11 : 12 }, autoSkip: false, maxRotation: byFarmer ? 60 : 0, minRotation: 0 },
             title: { display: true, text: byFarmer ? 'ชื่อนักเกษตร' : 'เขต / โซน', font: { size: 13 } } },
        y: { stacked: true, beginAtZero: true, ticks: { font: { size: 12 } }, title: { display: true, text: 'จำนวนแปลง (ถึงเกณฑ์)', font: { size: 13 } } }
      }
    }, plugins: [ChartDataLabels]
  });
}

// ===== TABLE =====
// วาดทีละ CHUNK แถว แล้วโหลดเพิ่มเมื่อเลื่อนใกล้ท้ายตาราง
// ตัวกรอง / ค้นหา / บล็อกสรุป ยังทำงานกับข้อมูลทั้งหมดเหมือนเดิม มีผลเฉพาะจำนวนแถวที่วาดบนหน้าจอ
const CHUNK = 200;
let _tblRows = [], _tblOffset = 0;

function _updateTableCount() {
  const total = _tblRows.length;
  let txt = total.toLocaleString('th-TH') + ' รายการ';
  if (_tblOffset < total) txt += ' (แสดง ' + _tblOffset.toLocaleString('th-TH') + ' แถวแรก เลื่อนลงเพื่อดูเพิ่ม)';
  const el = $('tableCount');
  if (el) el.textContent = txt;
}

function renderTable(rows) {
  _tblRows = rows;
  const tbody = $('tableBody');
  const wrap = tbody.closest('.table-wrapper');
  if (wrap) wrap.scrollTop = 0;

  if (!rows.length) {
    _tblOffset = 0;
    _updateTableCount();
    tbody.innerHTML = '<tr><td colspan="' + TABLE_COLS + '" style="text-align:center;padding:20px">🔍 ไม่พบข้อมูลที่ตรงกับเงื่อนไข</td></tr>';
    return;
  }
  _tblOffset = Math.min(CHUNK, rows.length);
  tbody.innerHTML = buildRows(rows, 0, _tblOffset);
  _updateTableCount();
}

function _appendTableRows() {
  if (_tblOffset >= _tblRows.length) return;
  const end = Math.min(_tblOffset + CHUNK, _tblRows.length);
  $('tableBody').insertAdjacentHTML('beforeend', buildRows(_tblRows, _tblOffset, end));
  _tblOffset = end;
  _updateTableCount();
}

function _onTableScroll(e) {
  const w = e.currentTarget;
  if (w.scrollTop + w.clientHeight >= w.scrollHeight - 500) _appendTableRows();
}

// อัปเดตสีแถว (เช่น หลังโหลดผลตรวจ GTT เสร็จ) เฉพาะแถวที่วาดอยู่ ไม่ต้องวาดตารางใหม่
function _refreshAllRowClasses() {
  document.querySelectorAll('#tableBody tr').forEach(tr => {
    const el = tr.querySelector('.order-input');
    if (el) _refreshRowClass(el.dataset.code, tr);
  });
}

function _escAttr(v) {
  return String(v == null ? '' : v).replace(/&/g,'&amp;').replace(/"/g,'&quot;');
}

function _ctBadgeHtml(val) {
  if (!val) return '<span style="color:#a0aec0">-</span>';
  return '<span style="color:#1a202c;font-weight:700">'+val+'</span>';
}

function _isRejectedCut(ctVal, inspResult) {
  return ctVal === 'รถตัดโรงงาน' && !!inspResult && String(inspResult).indexOf('ไม่รับตัด') !== -1;
}

// "ถึงเกณฑ์" หรือไม่ (ระวัง "ไม่ถึงเกณฑ์" มีคำว่า "ถึงเกณฑ์" อยู่ข้างในด้วย)
function _isCriteriaMet(v) {
  const s = String(v == null ? '' : v).trim();
  return s.indexOf('ถึงเกณฑ์') !== -1 && s.indexOf('ไม่') === -1;
}

// แปลงนี้บันทึก "ลำดับการตัด" แล้วหรือยัง (ตรงกับแถบเขียวในตาราง)
function _hasCutOrder(r) {
  const v = r[F.CUT_ORDER];
  return v !== '' && v != null && String(v).trim() !== '';
}

// สีแถว: แดง (รถตัดโรงงาน+ไม่รับตัด) > เขียว (บันทึกลำดับการตัดแล้ว) > ขาว (ถึงเกณฑ์) > เทา (ไม่ถึงเกณฑ์)
function _rowClass(ctVal, inspResult, r) {
  if (_isRejectedCut(ctVal, inspResult)) return 'row-rejected';
  if (_hasCutOrder(r)) return 'row-ordered';
  return _isCriteriaMet(r[F.CUT_CRITERIA]) ? 'row-age-old' : 'row-age-young';
}

function _refreshRowClass(pKey, tr) {
  const r = ALL_ROWS[plotCodeMap[pKey]];
  if (!r || !tr) return;
  const inspResult = INSP_RESULT_MAP[(r[F.PLOTINTECH] || '').trim()];
  tr.className = _rowClass(_rowCutType(r), inspResult, r);
}

function buildRows(rows, start, end) {
  const parts = new Array(end - start);
  for (let i = start; i < end; i++) {
    const r      = rows[i];
    const pKey   = plotKey(r);
    const pKeyAttr = _escAttr(pKey);
    const areaDisplay = r[F.AREA] > 0
      ? r[F.AREA].toLocaleString('th-TH', {minimumFractionDigits:1, maximumFractionDigits:1})
      : '-';
    const stdTon = r[F.EST_TON] || 0;
    const tonDisplay = stdTon > 0 ? fmt(stdTon) : '-';
    const tprVal = r[F.TON_PER_RAI] || 0;
    const tprDisplay = tprVal > 0
      ? tprVal.toLocaleString('th-TH', {minimumFractionDigits:1, maximumFractionDigits:1})
      : '-';
    const tonCell = '<td class="std-ton-cell">'+tonDisplay+'</td>';
    const tprCell = '<td style="text-align:right;color:#1a202c;font-weight:600">'+tprDisplay+'</td>';
    const ctVal   = _rowCutType(r);
    const ctCell  = '<td>'+_ctBadgeHtml(ctVal)+'</td>';
    // โรงงาน / กลุ่มตัด / ชื่อกลุ่มตัด เป็นข้อความธรรมดา (อ่านอย่างเดียว)
    const facCell = '<td>'+(r[F.FACTORY]      || '-')+'</td>';
    const grpCell = '<td>'+(r[F.CUTGROUP]     || '-')+'</td>';
    const gnCell  = '<td style="white-space:nowrap">'+(r[F.CUTGROUPNAME] || '-')+'</td>';
    // ไม่แสดงคอลัมน์ "ผลการตรวจ (KBA)" แต่ยังใช้ผลตรวจกำหนดสีแถวแดง
    const inspResult = INSP_RESULT_MAP[(r[F.PLOTINTECH] || '').trim()];
    const ageVal  = r[F.CANE_AGE];
    const ageCell = '<td style="text-align:right;font-weight:600">'+((ageVal !== '' && ageVal != null) ? fmt(ageVal) : '-')+'</td>';
    const curOrd  = pendingOrders[pKey] !== undefined ? pendingOrders[pKey] : (r[F.CUT_ORDER] || '');
    const chgOrd  = pendingOrders[pKey] !== undefined;
    const canOrder  = _isCriteriaMet(r[F.CUT_CRITERIA]);
    const orderCell = '<td style="text-align:center"><input type="text" inputmode="numeric" pattern="[0-9]*" class="order-input'+(chgOrd?' changed':'')+'" data-code="'+pKeyAttr+'" value="'+_escAttr(curOrd)+'" placeholder="-"'+(canOrder ? '' : ' disabled title="ไม่ถึงเกณฑ์ กรอกลำดับการตัดไม่ได้"')+' oninput="_sanitizeOrder(this)" onchange="onOrderInputChange(this)"></td>';
    const rowCls = _rowClass(ctVal, inspResult, r);
    parts[i - start] =
      '<tr class="'+rowCls+'">' +
      '<td style="white-space:nowrap;min-width:160px">'+(r[F.FARMER]  ||'-')+'</td>' +
      '<td>'+(r[F.ZONE]    ||'-')+'</td>' +
      '<td>'+(r[F.CANETYPE]||'-')+'</td>' +
      '<td style="text-align:right">'+areaDisplay+'</td>' +
      '<td><strong>'+(r[F.PLOTINTECH]||'-')+'</strong></td>' +
      '<td style="white-space:nowrap;min-width:170px">'+(r[F.OWNER]   ||'-')+'</td>' +
      tonCell + tprCell +
      ctCell +
      facCell + grpCell + gnCell +
      ageCell + orderCell +
      '</tr>';
  }
  return parts.join('');
}

function _sanitizeOrder(input) {
  const clean = input.value.replace(/[^0-9]/g, '');
  if (clean !== input.value) input.value = clean;
}

function onOrderInputChange(input) {
  const pKey = input.dataset.code;
  const row  = ALL_ROWS[plotCodeMap[pKey]] || {};
  const orig = String(row[F.CUT_ORDER] == null ? '' : row[F.CUT_ORDER]).trim();
  if (!_isCriteriaMet(row[F.CUT_CRITERIA])) {
    input.value = orig; input.disabled = true; delete pendingOrders[pKey]; updateSaveBtn();
    showToast('⚠️ แปลงนี้ไม่ถึงเกณฑ์ กรอกลำดับการตัดไม่ได้', 'error');
    return;
  }
  const val  = input.value.trim();
  if (val && !/^[0-9]+$/.test(val)) {
    showToast('⚠️ ลำดับการตัดต้องเป็นตัวเลขเท่านั้น', 'error');
    input.value = pendingOrders[pKey] !== undefined ? pendingOrders[pKey] : orig;
    return;
  }
  if (val === orig) { delete pendingOrders[pKey]; input.classList.remove('changed'); }
  else              { pendingOrders[pKey] = val;  input.classList.add('changed'); }
  updateSaveBtn();
}

function updateSaveBtn() {
  const has = Object.keys(pendingOrders).length > 0;
  const btn = $('btnSave');
  btn.disabled = !has;
  btn.classList.toggle('dirty', has);
}

// ===== SAVE =====
let _pendingSaveUpdates = null;

// บันทึกเฉพาะ "ลำดับการตัด" (คอลัมน์ U) / ฟิลด์อื่นส่งไปเพื่อแสดงในหน้ายืนยันเท่านั้น
function saveChanges() {
  const keys = Object.keys(pendingOrders);
  if (!keys.length) return;

  const updates = keys.map(k => {
    const row = ALL_ROWS[plotCodeMap[k]] || {};
    return {
      plotCode:     k,
      rowHint:      row[F.ROW] || 0, // เลขแถวตอนโหลด (เซิร์ฟเวอร์ตรวจรหัสแปลงซ้ำก่อนใช้)
      ownerName:    row[F.OWNER] || '',
      factory:      row[F.FACTORY] || '',
      cutGroup:     row[F.CUTGROUP] || '',
      cutGroupName: row[F.CUTGROUPNAME] || '',
      orderChanged: true,
      cutOrder:     pendingOrders[k]
    };
  });

  showSaveConfirm(updates);
}

function showSaveConfirm(updates) {
  _pendingSaveUpdates = updates;
  $('saveConfirmSub').textContent =
    'พบ ' + updates.length.toLocaleString('th-TH') + ' แถวที่มีการเปลี่ยนแปลง กรุณาตรวจสอบก่อนบันทึก';
  $('saveConfirmBody').innerHTML = updates.map(u =>
    '<tr>' +
    '<td><strong>'+(u.plotCode||'-')+'</strong></td>' +
    '<td style="white-space:nowrap">'+(u.ownerName||'-')+'</td>' +
    '<td>'+(u.factory||'-')+'</td>' +
    '<td>'+(u.cutGroup||'-')+'</td>' +
    '<td style="white-space:nowrap">'+(u.cutGroupName||'-')+'</td>' +
    '<td style="text-align:center">'+(u.cutOrder||'-')+'</td>' +
    '</tr>'
  ).join('');
  $('saveConfirmOverlay').style.display = 'flex';
}

function cancelSaveConfirm() {
  $('saveConfirmOverlay').style.display = 'none';
  _pendingSaveUpdates = null;
}

function confirmSaveConfirm() {
  $('saveConfirmOverlay').style.display = 'none';
  const updates = _pendingSaveUpdates;
  _pendingSaveUpdates = null;
  if (updates && updates.length) doSaveChanges(updates);
}

function doSaveChanges(updates) {
  const btn = $('btnSave');
  btn.disabled = true; btn.textContent = '⏳ กำลังบันทึก...';

  google.script.run
    .withSuccessHandler(function(res) {
      btn.innerHTML = '<span class="dot"></span>💾 บันทึกการเปลี่ยนแปลง';
      btn.disabled = true; btn.classList.remove('dirty');
      if (res.ok) {
        updates.forEach(u => {
          const idx = plotCodeMap[u.plotCode];
          if (idx !== undefined) ALL_ROWS[idx][F.CUT_ORDER] = String(u.cutOrder == null ? '' : u.cutOrder).trim();
        });
        pendingOrders = {};
        showToast('✅ บันทึกสำเร็จ '+res.saved+' รายการ','success');
        // ชีต Code (ยอดจัดลำดับ) คำนวณจากชีต Data ที่ซิงค์ผ่าน IMPORTRANGE ต้องรอสักพัก -> โหลดใหม่รอบเดียวหลัง 8 วินาที
        setTimeout(function() { fetchCodeDataForTable(true); }, 8000);

        updates.forEach(u => {
          const oi = document.querySelector('.order-input[data-code="'+CSS.escape(u.plotCode)+'"]');
          if (oi) { oi.classList.remove('changed'); _refreshRowClass(u.plotCode, oi.closest('tr')); }
        });

        // ถ้าเลือกตัวกรองลำดับการตัดอยู่ ให้กรองใหม่ (แถวอาจย้ายกลุ่มหลังบันทึก)
        if ($('filterOrdered').value !== 'all') applyFilters();
      } else {
        showToast('❌ บันทึกไม่สำเร็จ: '+res.msg,'error');
        btn.disabled=false; btn.classList.add('dirty');
        if (res.msg && res.msg.indexOf('รีเฟรชข้อมูล') !== -1) {
          setTimeout(refreshData, 1200);
        }
      }
    })
    .withFailureHandler(function(err) {
      btn.innerHTML = '<span class="dot"></span>💾 บันทึกการเปลี่ยนแปลง';
      btn.disabled=false; btn.classList.add('dirty');
      showToast('❌ เกิดข้อผิดพลาด: ' + err.message, 'error');
    })
    .saveCutTypes(updates);
}

// ===== MAP (แผนที่กลุ่มตัด) =====
let MAP_COORDS = {};                       // รหัสแปลง -> [lat, lng]
let _mapCoordsLoaded = false, _mapCoordsLoading = false, _forceMapCoords = false, _mapFitted = false;
let LMAP = null, LMAP_BASE = {}, LMAP_LAYER = null, LMAP_LINE = null, LMAP_CANVAS = null;
let MAP_ROWS = [];
let mapTab = 'members', mapBase = 'sat';
const MAP_MAX_PINS = 9000;
const MAP_AGE_COLORS = { tor:'#f28c28', ton:'#5bb89a', plai:'#1e2d55' };
const MAP_SIDE_CHUNK = 80;
let _mapSideItems = [], _mapSideOff = 0, _mapSearchTimer = null;

function _escHtml(v) {
  return String(v == null ? '' : v).replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;').replace(/"/g,'&quot;');
}

// จัดหมวดสีหมุดจาก "ชื่อประเภทอ้อย"
function mapCaneCat(name) {
  const n = String(name || '');
  if (n.indexOf('ปลายฝน') !== -1) return 'plai';
  if (n.indexOf('ต้นฝน')  !== -1) return 'ton';
  return 'tor';
}

function mapInit() {
  if (LMAP) return;
  LMAP_CANVAS = L.canvas({ padding: 0.5, tolerance: 6 });
  LMAP = L.map('mapCanvas', { center: [14.6, 101.7], zoom: 9, renderer: LMAP_CANVAS });
  LMAP_BASE.sat = L.tileLayer('https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}',
    { maxZoom: 19, attribution: 'Esri, Maxar, Earthstar Geographics' });
  LMAP_BASE.street = L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png',
    { maxZoom: 19, attribution: '&copy; OpenStreetMap' });
  LMAP_BASE.sat.addTo(LMAP);
  LMAP.on('popupopen',  function(e) { _mapPop = e.popup; _mapEdit = null; });
  LMAP.on('popupclose', function(e) { if (_mapPop === e.popup) { _mapPop = null; _mapEdit = null; } });
  $('mapSideScroll').addEventListener('scroll', function(e) {
    const w = e.currentTarget;
    if (w.scrollTop + w.clientHeight >= w.scrollHeight - 300) _mapSideAppendMore();
  }, { passive: true });
}

function mapOpen() {
  // ★ [แก้ไข ข้อ 78] โหลด Leaflet ตอนเปิดแผนที่ครั้งแรก (ถ้ายังไม่มา) แล้วเรียกตัวเองซ้ำ
  if (typeof L === 'undefined') {
    setText('mapNote', '⏳ กำลังโหลดไลบรารีแผนที่...');
    _ensureLeafletCss();
    _loadScript(LEAFLET_JS, function(ok) {
      if (!ok) { showToast('❌ โหลดไลบรารีแผนที่ไม่สำเร็จ กรุณารีเฟรชหน้าเว็บ', 'error'); setText('mapNote', '❌ โหลดไลบรารีแผนที่ไม่สำเร็จ'); return; }
      if (currentView === 'map') mapOpen();
    });
    return;
  }
  mapInit();
  setTimeout(function() { LMAP.invalidateSize(); }, 60);
  if (!ALL_ROWS.length) { setText('mapNote', '⏳ กำลังโหลดข้อมูลแปลง...'); return; } // init() จะเรียก mapOpen() อีกครั้งเมื่อโหลดเสร็จ
  if (!_mapCoordsLoaded) { if (!_mapCoordsLoading) mapLoadCoords(); return; }
  mapApply(!_mapFitted);
}

function mapLoadCoords() {
  _mapCoordsLoading = true;
  setText('mapNote', '⏳ กำลังโหลดพิกัดแปลง...');
  const force = _forceMapCoords; _forceMapCoords = false;
  google.script.run
    .withSuccessHandler(function(raw) {
      _mapCoordsLoading = false;
      const data = _pj(raw);
      if (data.error) { showToast('❌ ' + data.error, 'error'); setText('mapNote', '❌ ' + data.error); return; }
      MAP_COORDS = {};
      (data.plots || []).forEach(function(p) { MAP_COORDS[p[0]] = [p[1], p[2]]; });
      _mapCoordsLoaded = true;
      if (currentView === 'map' && ALL_ROWS.length) mapApply(true);
    })
    .withFailureHandler(function(err) {
      _mapCoordsLoading = false;
      showToast('❌ ' + err.message, 'error');
      setText('mapNote', '❌ ' + err.message);
    })
    .getMapCoords(!!force);
}

// ----- ตัวกรอง -----
// ตัวกรองแบบติ๊กเลือกได้หลายรายการ: MS[id] = { opts:[{v,l}], sel:{ค่า:true} } (ไม่เลือกเลย = แสดงทั้งหมด)
const MS = {};

function msSetOptions(id, opts) {
  const st = MS[id] || (MS[id] = { opts: [], sel: {} });
  st.opts = opts;
  const keep = {};
  opts.forEach(function(o) { if (st.sel[o.v]) keep[o.v] = true; }); // ตัดค่าที่เลือกไว้แต่ไม่มีในตัวเลือกแล้วออก
  st.sel = keep;
  msRender(id);
}

function msGet(id) {
  const st = MS[id];
  return st ? Object.keys(st.sel) : [];
}

function msRender(id) {
  const st = MS[id], root = $(id);
  if (!st || !root) return;
  const list = root.querySelector('.ms-list'), top = list.scrollTop;
  const q = (st.q || '').toLowerCase();
  let h = '', shown = 0;
  st.opts.forEach(function(o, i) {
    if (q && String(o.l).toLowerCase().indexOf(q) === -1) return; // ซ่อนรายการที่ไม่ตรงคำค้น
    shown++;
    h += '<label class="ms-item"><input type="checkbox" data-id="' + id + '" data-i="' + i + '"' + (st.sel[o.v] ? ' checked' : '') +
      ' onchange="msChange(this)"><span>' + _escHtml(o.l) + '</span></label>';
  });
  if (!shown) h = '<div class="ms-empty">' + (st.opts.length ? 'ไม่พบรายการที่ค้นหา' : 'ไม่มีตัวเลือก') + '</div>';
  list.innerHTML = h;
  list.scrollTop = top;
  const n = Object.keys(st.sel).length;
  let txt = root.dataset.all;
  if (n === 1) { const o = st.opts.filter(function(x) { return st.sel[x.v]; })[0]; txt = o ? o.l : txt; }
  else if (n > 1) txt = 'เลือก ' + n + ' รายการ';
  root.querySelector('.ms-txt').textContent = txt;
}

function msChange(inp) {
  const id = inp.dataset.id, st = MS[id];
  if (!st) return;
  const o = st.opts[parseInt(inp.dataset.i, 10)];
  if (!o) return;
  if (inp.checked) st.sel[o.v] = true; else delete st.sel[o.v];
  msRender(id);
  mapApply(true);
}

function msAll(id) {
  const st = MS[id];
  if (!st) return;
  const q = (st.q || '').toLowerCase();
  // มีคำค้นอยู่ = เลือกเฉพาะรายการที่ค้นเจอ (เพิ่มจากที่เลือกไว้แล้ว)
  if (!q) st.sel = {};
  st.opts.forEach(function(o) {
    if (!q || String(o.l).toLowerCase().indexOf(q) !== -1) st.sel[o.v] = true;
  });
  msRender(id);
  mapApply(true);
}

// พิมพ์ค้นหาในรายการ (ไม่กรองแผนที่ แค่ซ่อน/แสดงตัวเลือก)
function msSearch(id, val) {
  const st = MS[id];
  if (!st) return;
  st.q = String(val || '').trim();
  msRender(id);
}

function msClear(id) {
  const st = MS[id];
  if (!st) return;
  st.sel = {};
  msRender(id);
  mapApply(true);
}

function msCloseAll() {
  document.querySelectorAll('.ms-panel').forEach(function(p) { p.style.display = 'none'; });
}

function msToggle(id) {
  const root = $(id), p = root.querySelector('.ms-panel');
  const open = p.style.display === 'none';
  msCloseAll();
  if (open) {
    p.style.display = '';
    const inp = root.querySelector('.ms-search input');
    if (inp) {
      inp.value = '';
      if (MS[id]) { MS[id].q = ''; msRender(id); }
      setTimeout(function() { inp.focus(); }, 30);
    }
  }
}

// คลิกนอกกล่องตัวกรอง = ปิดรายการ
document.addEventListener('mousedown', function(e) {
  if (e.target.closest && e.target.closest('.ms')) return;
  msCloseAll();
});

function mapGetFilters() {
  return {
    zone: msGet('mapFZone'), ct: msGet('mapFCutType'), grp: msGet('mapFGroup'),
    gname: msGet('mapFGroupName'), plot: ($('mapFPlot').value || '').trim().toLowerCase(), crit: msGet('mapFCrit')
  };
}

function mapRowMatches(r, f, ex) {
  if (ex !== 'zone' && f.zone.length && f.zone.indexOf(r[F.ZONE]) === -1) return false;
  if (ex !== 'ct' && f.ct.length && f.ct.indexOf((_rowCutType(r).trim()) || 'ยังไม่จัดคิว') === -1) return false;
  if (ex !== 'grp' && f.grp.length && f.grp.indexOf((r[F.CUTGROUP] || '').trim()) === -1) return false;
  if (ex !== 'gname' && f.gname.length && f.gname.indexOf((r[F.CUTGROUPNAME] || '').trim()) === -1) return false;
  if (f.plot && (r[F.PLOTINTECH] || '').toLowerCase().indexOf(f.plot) === -1) return false;
  if (ex !== 'crit' && f.crit.length) {
    const key = _isCriteriaMet(r[F.CUT_CRITERIA]) ? 'met' : 'notmet';
    if (f.crit.indexOf(key) === -1) return false;
  }
  return true;
}

function mapUpdateOptions() {
  const f = mapGetFilters();
  const z = {}, ct = {}, g = {}, gn = {};
  for (let i = 0; i < ALL_ROWS.length; i++) {
    const r = ALL_ROWS[i];
    if (r[F.ZONE] && mapRowMatches(r, f, 'zone')) z[r[F.ZONE]] = 1;
    if (mapRowMatches(r, f, 'ct')) ct[(_rowCutType(r).trim()) || 'ยังไม่จัดคิว'] = 1;
    const gc = (r[F.CUTGROUP] || '').trim();
    if (gc && mapRowMatches(r, f, 'grp')) g[gc] = 1;
    const gnm = (r[F.CUTGROUPNAME] || '').trim();
    if (gnm && mapRowMatches(r, f, 'gname')) gn[gnm] = 1;
  }
  const toOpts = function(o, sorter) { return Object.keys(o).sort(sorter).map(function(v) { return { v: v, l: v }; }); };
  msSetOptions('mapFZone',      toOpts(z, zoneCompare));
  msSetOptions('mapFCutType',   toOpts(ct, thSort));
  msSetOptions('mapFGroup',     toOpts(g, thSort));
  msSetOptions('mapFGroupName', toOpts(gn, thSort));
  msSetOptions('mapFCrit',      [{ v: 'met', l: 'ถึงเกณฑ์' }, { v: 'notmet', l: 'ไม่ถึงเกณฑ์' }]);
}

function mapOnSearch() {
  clearTimeout(_mapSearchTimer);
  _mapSearchTimer = setTimeout(function() { mapApply(true); }, 300);
}

function mapResetFilters() {
  ['mapFZone','mapFCutType','mapFGroup','mapFGroupName','mapFCrit'].forEach(function(id) { if (MS[id]) { MS[id].sel = {}; MS[id].q = ''; } });
  $('mapFPlot').value = '';
  msCloseAll();
  mapApply(true);
}

function mapApply(fit) {
  if (!LMAP || !ALL_ROWS.length) return;
  mapUpdateOptions();
  const f = mapGetFilters();
  MAP_ROWS = ALL_ROWS.filter(function(r) { return mapRowMatches(r, f, null); });
  mapRenderStats();
  mapRenderMarkers(fit);
  mapRenderSide();
}

function mapRenderStats() {
  let area = 0, ton = 0; const owners = {}, groups = {};
  for (let i = 0; i < MAP_ROWS.length; i++) {
    const r = MAP_ROWS[i];
    area += r[F.AREA] || 0; ton += r[F.EST_TON] || 0;
    if (r[F.PLOTCODE]) owners[r[F.PLOTCODE]] = 1;
    const g = (r[F.CUTGROUP] || '').trim(); if (g) groups[g] = 1;
  }
  setText('mapStPlots',  fmt(MAP_ROWS.length));
  setText('mapStArea',   fmt(area));
  setText('mapStTon',    fmt(ton));
  setText('mapStOwners', fmt(Object.keys(owners).length));
  setText('mapGroupCount', 'รวม ' + fmt(Object.keys(groups).length) + ' กลุ่มตัดในแผนนี้');
}

// ----- หมุดบนแผนที่ -----
function mapRenderMarkers(fit) {
  if (LMAP_LAYER) LMAP_LAYER.clearLayers(); else LMAP_LAYER = L.layerGroup().addTo(LMAP);
  if (LMAP_LINE) { LMAP.removeLayer(LMAP_LINE); LMAP_LINE = null; }

  const ordered = [], normal = [];
  let noCoord = 0, orderedAll = 0;
  for (let i = 0; i < MAP_ROWS.length; i++) {
    const r = MAP_ROWS[i];
    const has = _hasCutOrder(r);
    if (has) orderedAll++;
    if (!MAP_COORDS[(r[F.PLOTINTECH] || '').trim()]) { noCoord++; continue; }
    (has ? ordered : normal).push(r);
  }
  ordered.sort(function(a, b) { return (parseInt(a[F.CUT_ORDER], 10) || 0) - (parseInt(b[F.CUT_ORDER], 10) || 0); });

  // เกิน 9,000 หมุด: เก็บแปลงที่มีลำดับไว้ก่อน แล้วสุ่มกระจายเท่าๆ กันจากที่เหลือ
  const room = Math.max(0, MAP_MAX_PINS - ordered.length);
  let pins = normal;
  if (normal.length > room) {
    pins = []; const step = normal.length / room;
    for (let k = 0; k < room; k++) pins.push(normal[Math.floor(k * step)]);
  }

  const pts = [];
  pins.forEach(function(r) {
    const c = MAP_COORDS[(r[F.PLOTINTECH] || '').trim()];
    const col = MAP_AGE_COLORS[mapCaneCat(r[F.CANETYPE])];
    const m = L.circleMarker(c, { renderer: LMAP_CANVAS, radius: 5, color: '#ffffff', weight: 1, fillColor: col, fillOpacity: 0.92 });
    m.bindPopup(function() { return mapPopupHtml(r); }, { maxWidth: 340 });
    m.addTo(LMAP_LAYER);
    pts.push(c);
  });

  const byGrp = {}; // ordered เรียงตามลำดับแล้ว -> จุดในแต่ละกลุ่มจึงเรียงถูกอัตโนมัติ
  ordered.forEach(function(r) {
    const c = MAP_COORDS[(r[F.PLOTINTECH] || '').trim()];
    const n = String(r[F.CUT_ORDER]).trim();
    const m = L.marker(c, { icon: L.divIcon({ className: 'map-ord-pin ord-' + mapCaneCat(r[F.CANETYPE]), html: '<span>' + _escHtml(n) + '</span>', iconSize: [26, 26], iconAnchor: [13, 13] }), zIndexOffset: 1000 });
    m.bindPopup(function() { return mapPopupHtml(r); }, { maxWidth: 340 });
    m.addTo(LMAP_LAYER);
    pts.push(c);
    const g = (r[F.CUTGROUP] || '').trim();
    if (g) (byGrp[g] = byGrp[g] || []).push(c);
  });
  const lines = Object.keys(byGrp).filter(function(g) { return byGrp[g].length > 1; }).map(function(g) {
    return L.polyline(byGrp[g], { color: '#e53e3e', weight: 3, opacity: 0.9 });
  });
  if (lines.length) LMAP_LINE = L.layerGroup(lines).addTo(LMAP);

  let note = 'แสดง ' + fmt(pins.length + ordered.length) + ' หมุด จาก ' + fmt(MAP_ROWS.length) + ' แปลง · มีลำดับการตัด ' + fmt(orderedAll) + ' แปลง';
  if (noCoord) note += ' · ไม่มีพิกัด ' + fmt(noCoord) + ' แปลง';
  setText('mapNote', note);

  if (fit && pts.length) { LMAP.fitBounds(L.latLngBounds(pts), { padding: [20, 20] }); _mapFitted = true; }
}

let _mapPop = null, _mapEdit = null;

function mapRowHtml(l, v) {
  return '<div class="mp-row"><span>' + l + '</span><b>' + _escHtml(v === '' || v == null ? '-' : v) + '</b></div>';
}

// ลำดับการตัดของแปลงอื่นในกลุ่มเดียวกัน (ใช้หาเลขถัดไป / เช็คเลขซ้ำ)
function mapGroupStat(grp, selfCode) {
  let n = 0, max = 0; const used = {};
  for (let i = 0; i < ALL_ROWS.length; i++) {
    const x = ALL_ROWS[i];
    if ((x[F.CUTGROUP] || '').trim() !== grp) continue;
    const xc = (x[F.PLOTINTECH] || '').trim();
    if (xc === selfCode || !_hasCutOrder(x)) continue;
    const v = parseInt(x[F.CUT_ORDER], 10);
    if (isNaN(v)) continue;
    n++; if (v > max) max = v;
    (used[v] = used[v] || []).push(xc);
  }
  return { n: n, max: max, used: used };
}

function mapPopSet(html) {
  // เลื่อนไปทำหลัง click จบ: ถ้าเปลี่ยนเนื้อหาทันที ปุ่มที่ถูกกดจะหลุดจาก DOM
  // แล้ว Leaflet เข้าใจว่าเป็นการคลิกบนแผนที่ และปิดป๊อปอัปทิ้ง
  setTimeout(function() {
    if (!_mapPop || !LMAP) return;
    _mapPop.setContent('<div class="mp" id="mapPopBox">' + html + '</div>');
    // เนื้อหาเปลี่ยนความสูง -> เลื่อนแผนที่ให้ป๊อปอัปอยู่ในกรอบเสมอ
    setTimeout(function() {
      if (!_mapPop || !LMAP) return;
      const el = _mapPop.getElement();
      if (!el) return;
      const mapR = LMAP.getContainer().getBoundingClientRect();
      const popR = el.getBoundingClientRect();
      const dy = popR.top - mapR.top - 12;
      if (dy < 0) LMAP.panBy([0, dy], { animate: true });
    }, 30);
  }, 0);
}

// ---------- มุมมอง 1: ข้อมูลแปลง ----------
function mapPopMainInner(r, code) {
  const met  = _isCriteriaMet(r[F.CUT_CRITERIA]);
  const ord  = _hasCutOrder(r) ? String(r[F.CUT_ORDER]).trim() : '-';
  const age  = (r[F.CANE_AGE] !== '' && r[F.CANE_AGE] != null) ? fmt(r[F.CANE_AGE]) + ' วัน' : '-';
  const area = (r[F.AREA] || 0).toLocaleString('th-TH', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) + ' ไร่';
  const btn = met
    ? '<button type="button" class="mp-edit" data-code="' + _escAttr(code) + '" onclick="mapEditOrder(this.dataset.code)">✎ แก้ไขลำดับ</button>'
    : '<button type="button" class="mp-edit" disabled>✎ แก้ไขลำดับ</button>';
  return '<div class="mp-row"><span>ลำดับการตัด:</span><span><b>' + _escHtml(ord) + '</b> ' + btn + '</span></div>' +
    (met ? '' : '<div class="mp-hint">คีย์ลำดับได้เฉพาะแปลงที่เกณฑ์การตัดเป็น “ถึงเกณฑ์”</div>') +
    mapRowHtml('เลขแปลง Intech:',  code) +
    mapRowHtml('เลขโควต้า:',       r[F.PLOTCODE]) +
    mapRowHtml('ชื่อชาวไร่:',      r[F.OWNER]) +
    mapRowHtml('ชื่อประเภทอ้อย:',  r[F.CANETYPE]) +
    mapRowHtml('พื้นที่:',          area) +
    mapRowHtml('รหัสนักเกษตร:',    r[F.FARMERID]) +
    mapRowHtml('ชื่อนักเกษตร:',    r[F.FARMER]) +
    mapRowHtml('โซน:',              r[F.ZONE]) +
    mapRowHtml('กลุ่มตัด:',        r[F.CUTGROUP]) +
    mapRowHtml('ชื่อกลุ่มตัด:',    r[F.CUTGROUPNAME]) +
    mapRowHtml('อายุอ้อย:',        age) +
    mapRowHtml('เกณฑ์การตัด:',     r[F.CUT_CRITERIA]);
}

function mapPopupHtml(r) {
  try {
    return '<div class="mp" id="mapPopBox">' +
      mapPopMainInner(r, (r[F.PLOTINTECH] || '').trim()) + '</div>';
  } catch (e) {
    console.error('mapPopupHtml', e, r);
    return '<div class="mp"><div class="mp-title">แสดงข้อมูลแปลงไม่ได้</div>' +
      '<div class="mp-warn">' + _escHtml(e.message) + '</div></div>';
  }
}

function mapPopBack(code) {
  _mapEdit = null;
  const r = ALL_ROWS[plotCodeMap[code]];
  if (r) mapPopSet(mapPopMainInner(r, code));
}

function _mapPopHead(r, code) {
  return '<div class="mp-title">' + _escHtml(r[F.OWNER] || '-') + '</div>' +
    mapRowHtml('เลขแปลงอินเทค:', code) + mapRowHtml('กลุ่มตัด:', r[F.CUTGROUP]);
}

// ---------- มุมมอง 2: กรอกลำดับการตัดใหม่ ----------
function mapEditOrder(code, preset) {
  const r = ALL_ROWS[plotCodeMap[code]];
  if (!r) return;
  if (!_isCriteriaMet(r[F.CUT_CRITERIA])) { showToast('⚠️ แปลงนี้ไม่ถึงเกณฑ์ กรอกลำดับการตัดไม่ได้', 'error'); return; }
  const grp = (r[F.CUTGROUP] || '').trim();
  if (!grp) { showToast('⚠️ แปลงนี้ยังไม่มีกลุ่มตัด จึงกำหนดลำดับการตัดไม่ได้', 'error'); return; }
  const st  = mapGroupStat(grp, code);
  const cur = _hasCutOrder(r) ? String(r[F.CUT_ORDER]).trim() : '-';
  const c = _escAttr(code);
  mapPopSet(
    mapRowHtml('เลขแปลง:', code) +
    mapRowHtml('กลุ่มตัด:', grp) +
    mapRowHtml('ลำดับการตัดปัจจุบัน:', cur) +
    '<div class="mp-box"><div class="mp-box-lbl">ลำดับการตัดใหม่</div>' +
    '<input type="number" min="1" step="1" inputmode="numeric" id="mapPopOrdInput" class="mp-input" data-code="' + c + '" ' +
      'placeholder="เว้นว่าง = ลำดับที่ ' + (st.max + 1) + '" value="' + _escAttr(preset || '') + '" onkeydown="mapOrdKey(event, this.dataset.code)"></div>' +
    '<div class="mp-actions">' +
    '<button type="button" class="mp-btn" data-code="' + c + '" onclick="mapPopBack(this.dataset.code)">ยกเลิก</button>' +
    '<button type="button" class="mp-btn primary" data-code="' + c + '" onclick="mapOrderReview(this.dataset.code)">บันทึก</button></div>');
  setTimeout(function() { const inp = $('mapPopOrdInput'); if (inp) inp.focus(); }, 50);
}

function mapOrdKey(ev, code) {
  ev.stopPropagation(); // กัน Leaflet เอาปุ่มลูกศร / เลข 6 ไปเลื่อน-ซูมแผนที่
  if (ev.key === 'Enter') mapOrderReview(code);
}

// ---------- มุมมอง 3: ตรวจสอบข้อมูลที่กรอก ----------
function mapOrderReview(code) {
  const r = ALL_ROWS[plotCodeMap[code]];
  if (!r) return;
  const raw = (($('mapPopOrdInput') || {}).value || '').trim();
  if (raw && (!/^[0-9]+$/.test(raw) || Number(raw) < 1)) { showToast('⚠️ ลำดับการตัดต้องเป็นตัวเลขตั้งแต่ 1 ขึ้นไป', 'error'); return; }
  const grp = (r[F.CUTGROUP] || '').trim();
  const st  = mapGroupStat(grp, code);
  const fin = raw ? Number(raw) : st.max + 1;
  const cur = _hasCutOrder(r) ? String(r[F.CUT_ORDER]).trim() : '-';
  if (cur === String(fin)) { showToast('ลำดับการตัดไม่มีการเปลี่ยนแปลง', 'success'); mapPopBack(code); return; }
  _mapEdit = { code: code, raw: raw, final: fin };
  const c = _escAttr(code);
  let h = _mapPopHead(r, code) +
    '<div class="mp-box"><div class="mp-box-lbl c">ตรวจสอบข้อมูลที่กรอก</div>' +
    '<div class="mp-change">' + _escHtml(cur) + ' → ' + fin + '</div>';
  if (!raw) h += '<div class="mp-note c">ไม่ได้กรอกเลข: ระบบจะใส่เป็นลำดับต่อท้ายกลุ่มตัดนี้ และรันลำดับให้ต่อเนื่องอัตโนมัติ</div>';
  if (raw && st.used[fin]) h += '<div class="mp-warn">⚠️ ลำดับที่ ' + fin + ' ในกลุ่มนี้ถูกใช้แล้ว ' + st.used[fin].length + ' แปลง (เช่น ' + _escHtml(st.used[fin][0]) + ')</div>';
  h += '</div><div class="mp-actions">' +
    '<button type="button" class="mp-btn" data-code="' + c + '" data-raw="' + _escAttr(raw) + '" onclick="mapEditOrder(this.dataset.code, this.dataset.raw)">แก้ไข</button>' +
    '<button type="button" class="mp-btn primary" data-code="' + c + '" onclick="mapSaveOrder(this)">ตกลง</button></div>';
  mapPopSet(h);
}

// ---------- บันทึกจริง (ใช้ saveCutTypes เดิม ฝั่งเซิร์ฟเวอร์ตรวจเกณฑ์ / รหัสแปลง / Lock เหมือนเดิม) ----------
function mapSaveOrder(btn) {
  const code = btn.dataset.code, r = ALL_ROWS[plotCodeMap[code]];
  if (!r || !_mapEdit || _mapEdit.code !== code) return;
  const val = String(_mapEdit.final);
  btn.disabled = true; btn.textContent = '⏳...';
  google.script.run
    .withSuccessHandler(function(res) {
      if (res.ok) {
        r[F.CUT_ORDER] = val;
        delete pendingOrders[code]; updateSaveBtn();
        _mapEdit = null;
        showToast('✅ บันทึกลำดับการตัดสำเร็จ (ลำดับที่ ' + val + ')', 'success');
        LMAP.closePopup();
        mapApply(false);
        setTimeout(function() { fetchCodeDataForTable(true); }, 8000);
      } else {
        btn.disabled = false; btn.textContent = 'ตกลง';
        showToast('❌ บันทึกไม่สำเร็จ: ' + res.msg, 'error');
      }
    })
    .withFailureHandler(function(err) {
      btn.disabled = false; btn.textContent = 'ตกลง';
      showToast('❌ เกิดข้อผิดพลาด: ' + err.message, 'error');
    })
    .saveCutTypes([{ plotCode: code, rowHint: r[F.ROW] || 0, orderChanged: true, cutOrder: val }]);
}

// ----- แผงด้านขวา (สมาชิก / แปลงอ้อยทั้งหมด) -----
function mapSetTab(t) {
  mapTab = t;
  document.querySelectorAll('.map-tab').forEach(function(b) { b.classList.toggle('active', b.dataset.tab === t); });
  mapRenderSide();
}

function mapRenderSide() {
  const box = $('mapSideBody'), info = $('mapSideInfo');
  $('mapSideScroll').scrollTop = 0;
  if (mapTab === 'members') {
    const m = {};
    MAP_ROWS.forEach(function(r) {
      const q = r[F.PLOTCODE] || '-';
      let o = m[q];
      if (!o) o = m[q] = { q: q, name: r[F.OWNER] || '-', n: 0, area: 0, ton: 0 };
      o.n++; o.area += r[F.AREA] || 0; o.ton += r[F.EST_TON] || 0;
    });
    _mapSideItems = Object.keys(m).map(function(k) { return m[k]; }).sort(function(a, b) { return b.ton - a.ton; });
    info.style.display = 'none';
    box.innerHTML = '<div id="mapSideAppend"></div>';
  } else {
    const ord = [], rest = []; let area = 0;
    MAP_ROWS.forEach(function(r) { area += r[F.AREA] || 0; (_hasCutOrder(r) ? ord : rest).push(r); });
    ord.sort(function(a, b) { return (parseInt(a[F.CUT_ORDER], 10) || 0) - (parseInt(b[F.CUT_ORDER], 10) || 0); });
    _mapSideItems = ord.concat(rest);
    info.style.display = '';
    info.textContent = fmt(MAP_ROWS.length) + ' แปลง · พื้นที่รวม ' +
      area.toLocaleString('th-TH', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) + ' ไร่ · มีลำดับการตัด ' + fmt(ord.length) + ' แปลง';
    box.innerHTML = '<table class="map-tbl"><thead><tr><th>ลำดับการตัด</th><th>เลขแปลงอินเทค</th><th>ชื่อชาวไร่</th><th>ชื่อประเภทอ้อย</th><th>อายุอ้อย (วัน)</th><th>พื้นที่ (ไร่)</th></tr></thead><tbody id="mapSideAppend"></tbody></table>';
  }
  _mapSideOff = 0;
  if (!_mapSideItems.length) { box.innerHTML = '<div style="text-align:center;padding:20px;color:#a0aec0">🔍 ไม่พบข้อมูลที่ตรงกับเงื่อนไข</div>'; return; }
  _mapSideAppendMore();
}

function _mapSideAppendMore() {
  const el = $('mapSideAppend');
  if (!el || _mapSideOff >= _mapSideItems.length) return;
  const end = Math.min(_mapSideOff + MAP_SIDE_CHUNK, _mapSideItems.length);
  let h = '';
  for (let i = _mapSideOff; i < end; i++) {
    const it = _mapSideItems[i];
    if (mapTab === 'members') {
      h += '<div class="map-mem" data-q="' + _escAttr(it.q) + '" onclick="mapFocusMember(this.dataset.q)">' +
        '<div><div class="map-mem-name">' + _escHtml(it.name) + '</div>' +
        '<div class="map-mem-sub">โควตา ' + _escHtml(it.q) + ' · ' + fmt(it.n) + ' แปลง · ' + fmt(it.area) + ' ไร่</div></div>' +
        '<div class="map-mem-r"><div class="map-mem-ton">' + fmt(it.ton) + ' ตัน</div><span class="map-mem-badge">แปลงอ้อย</span></div></div>';
    } else {
      const r = it;
      const pc     = (r[F.PLOTINTECH] || '').trim();
      const canOrd = _isCriteriaMet(r[F.CUT_CRITERIA]);
      const ordVal = _hasCutOrder(r) ? String(r[F.CUT_ORDER]).trim() : '';
      h += '<tr data-code="' + _escAttr(pc) + '" onclick="mapFocusPlot(this.dataset.code)">' +
        '<td class="c" onclick="event.stopPropagation()"><input type="text" inputmode="numeric" class="map-ord-input" data-code="' + _escAttr(pc) +
          '" data-orig="' + _escAttr(ordVal) + '" value="' + _escAttr(ordVal) + '" placeholder="-"' +
          (canOrd ? '' : ' disabled title="ไม่ถึงเกณฑ์ กรอกลำดับการตัดไม่ได้"') +
          ' oninput="_sanitizeOrder(this)" onchange="mapSideSaveOrder(this)" onkeydown="mapSideOrdKey(event, this)"></td>' +
        '<td>' + _escHtml(r[F.PLOTINTECH] || '-') + '</td>' +
        '<td>' + _escHtml(r[F.OWNER] || '-') + '</td>' +
        '<td>' + _escHtml(r[F.CANETYPE] || '-') + '</td>' +
        '<td class="r">' + ((r[F.CANE_AGE] !== '' && r[F.CANE_AGE] != null) ? fmt(r[F.CANE_AGE]) : '-') + '</td>' +
        '<td class="r">' + (r[F.AREA] || 0).toLocaleString('th-TH', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) + '</td></tr>';
    }
  }
  el.insertAdjacentHTML('beforeend', h);
  _mapSideOff = end;
}

// กด Enter ในช่องลำดับการตัด = ออกจากช่อง (ทำให้ onchange ทำงานและบันทึก)
function mapSideOrdKey(ev, inp) {
  ev.stopPropagation();
  if (ev.key === 'Enter') inp.blur();
}

// บันทึกลำดับการตัดจากตารางด้านขวาของแผนที่ (ใช้ saveCutTypes เดิม เซิร์ฟเวอร์ตรวจเกณฑ์ / รหัสแปลง / Lock เหมือนเดิม)
function mapSideSaveOrder(inp) {
  const code = inp.dataset.code, r = ALL_ROWS[plotCodeMap[code]];
  if (!r) return;
  const orig = inp.dataset.orig || '';
  const val  = inp.value.trim();
  if (val === orig) return;
  if (!_isCriteriaMet(r[F.CUT_CRITERIA])) {
    inp.value = orig; inp.disabled = true;
    showToast('⚠️ แปลงนี้ไม่ถึงเกณฑ์ กรอกลำดับการตัดไม่ได้', 'error');
    return;
  }
  if (val && !/^[0-9]+$/.test(val)) {
    inp.value = orig;
    showToast('⚠️ ลำดับการตัดต้องเป็นตัวเลขเท่านั้น', 'error');
    return;
  }
  inp.disabled = true;
  google.script.run
    .withSuccessHandler(function(res) {
      inp.disabled = false;
      if (!res.ok) {
        inp.value = orig;
        showToast('❌ บันทึกไม่สำเร็จ: ' + res.msg, 'error');
        return;
      }
      r[F.CUT_ORDER] = val;
      inp.dataset.orig = val;
      delete pendingOrders[code]; updateSaveBtn();

      // เตือนถ้าเลขซ้ำกับแปลงอื่นในกลุ่มตัดเดียวกัน (ยังบันทึกให้แล้ว)
      let dup = 0;
      const grp = (r[F.CUTGROUP] || '').trim();
      if (val && grp) { const u = mapGroupStat(grp, code).used[Number(val)]; dup = u ? u.length : 0; }
      showToast(dup ? '✅ บันทึกแล้ว แต่ลำดับที่ ' + val + ' ซ้ำกับอีก ' + dup + ' แปลงในกลุ่มเดียวกัน'
                    : (val ? '✅ บันทึกลำดับการตัดสำเร็จ (ลำดับที่ ' + val + ')' : '✅ ล้างลำดับการตัดสำเร็จ'),
                dup ? 'error' : 'success');

      // วาดหมุด / เส้นเชื่อมใหม่ โดยไม่วาดตารางด้านขวาใหม่ (ตำแหน่งเลื่อนจะได้ไม่เด้งกลับขึ้นบน)
      mapRenderMarkers(false);
      const info = $('mapSideInfo');
      if (info && info.style.display !== 'none') {
        let n = 0;
        for (let i = 0; i < MAP_ROWS.length; i++) if (_hasCutOrder(MAP_ROWS[i])) n++;
        info.textContent = info.textContent.replace(/มีลำดับการตัด .* แปลง$/, 'มีลำดับการตัด ' + fmt(n) + ' แปลง');
      }
      setTimeout(function() { fetchCodeDataForTable(true); }, 8000);
    })
    .withFailureHandler(function(err) {
      inp.disabled = false; inp.value = orig;
      showToast('❌ เกิดข้อผิดพลาด: ' + err.message, 'error');
    })
    .saveCutTypes([{ plotCode: code, rowHint: r[F.ROW] || 0, orderChanged: true, cutOrder: val }]);
}

function mapFocusMember(q) {
  const pts = [];
  MAP_ROWS.forEach(function(r) {
    if ((r[F.PLOTCODE] || '-') !== q) return;
    const c = MAP_COORDS[(r[F.PLOTINTECH] || '').trim()];
    if (c) pts.push(c);
  });
  if (!pts.length) { showToast('⚠️ ไม่พบพิกัดของสมาชิกรายนี้', 'error'); return; }
  LMAP.fitBounds(L.latLngBounds(pts), { padding: [40, 40], maxZoom: 15 });
}

function mapFocusPlot(code) {
  const r = ALL_ROWS[plotCodeMap[code]], c = MAP_COORDS[code];
  if (!r) return;
  if (!c) { showToast('⚠️ แปลงนี้ไม่มีพิกัด', 'error'); return; }
  LMAP.setView(c, 16);
  L.popup({ maxWidth: 340 }).setLatLng(c).setContent(mapPopupHtml(r)).openOn(LMAP);
}

// ----- ปุ่มต่างๆ -----
function mapSetBase(b) {
  mapBase = b;
  document.querySelectorAll('.map-base-pill').forEach(function(x) { x.classList.toggle('active', x.dataset.base === b); });
  LMAP.removeLayer(b === 'sat' ? LMAP_BASE.street : LMAP_BASE.sat);
  LMAP_BASE[b].addTo(LMAP);
}

function mapToggleExpand() {
  const on = $('mapCard').classList.toggle('map-expanded');
  $('mapExpandBtn').textContent = on ? '✕ ย่อ' : '⛶ ขยาย';
  setTimeout(function() { LMAP.invalidateSize(); }, 120);
}

// ดาวน์โหลด KMZ: ทุกแปลงที่ผ่านตัวกรอง (ไม่จำกัด 9,000 หมุด) + เส้นเชื่อมตามลำดับการตัด (แยกตามกลุ่มตัด)
function _xml(v) { return String(v == null ? '' : v).replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;'); }

function mapBuildKml() {
  const styles = { tor:'ff288cf2', ton:'ff9ab85b', plai:'ff552d1e', ordtor:'ff288cf2', ordton:'ff9ab85b', ordplai:'ff552d1e' }; // รูปแบบสี KML = aabbggrr
  let k = '<?xml version="1.0" encoding="UTF-8"?><kml xmlns="http://www.opengis.net/kml/2.2"><Document><name>แผนที่กลุ่มตัด ปี 69/70</name>';
  Object.keys(styles).forEach(function(s) {
    const isOrd = s.indexOf('ord') === 0;
    // ★ [แก้ไข ข้อ 78] ลบอักขระเกิน "Style>';" ท้ายบรรทัดนี้ (เดิมทำให้สคริปต์หน้าเว็บ Syntax Error ทั้งก้อน)
    k += '<Style id="s_' + s + '"><IconStyle><color>' + styles[s] + '</color><scale>' + (isOrd ? '1.0' : '0.7') +
      '</scale><Icon><href>http://maps.google.com/mapfiles/kml/shapes/placemark_circle.png</href></Icon></IconStyle><LabelStyle><scale>' + (isOrd ? '0.9' : '0') + '</scale></LabelStyle></Style>';
  });
  k += '<Style id="s_line"><LineStyle><color>ff0000ff</color><width>3</width></LineStyle></Style>';
  const ordered = [];
  MAP_ROWS.forEach(function(r) {
    const code = (r[F.PLOTINTECH] || '').trim(), c = MAP_COORDS[code];
    if (!c) return;
    const has = _hasCutOrder(r), ordTxt = has ? String(r[F.CUT_ORDER]).trim() : '';
    if (has) ordered.push(r);
    const desc = 'ชาวไร่: ' + _xml(r[F.OWNER]) + '<br>นักเกษตร: ' + _xml(r[F.FARMER]) + '<br>โซน: ' + _xml(r[F.ZONE]) +
      '<br>ประเภทอ้อย: ' + _xml(r[F.CANETYPE]) + '<br>พื้นที่: ' + (r[F.AREA] || 0) + ' ไร่<br>ตัน STD: ' + fmt(r[F.EST_TON] || 0) +
      '<br>ประเภทการตัด: ' + _xml(_rowCutType(r)) + '<br>เกณฑ์การตัด: ' + _xml(r[F.CUT_CRITERIA]) +
      '<br>กลุ่มตัด: ' + _xml(r[F.CUTGROUP]) + ' ' + _xml(r[F.CUTGROUPNAME]) + '<br>ลำดับการตัด: ' + _xml(ordTxt || '-');
    k += '<Placemark><name>' + _xml(has ? ordTxt + ' - ' + code : code) + '</name><styleUrl>#s_' + (has ? 'ord' : '') + mapCaneCat(r[F.CANETYPE]) +
      '</styleUrl><description><![CDATA[' + desc + ']]></description><Point><coordinates>' + c[1] + ',' + c[0] + ',0</coordinates></Point></Placemark>';
  });
  ordered.sort(function(a, b) { return (parseInt(a[F.CUT_ORDER], 10) || 0) - (parseInt(b[F.CUT_ORDER], 10) || 0); });
  const kg = {};
  ordered.forEach(function(r) { const g = (r[F.CUTGROUP] || '').trim(); if (g) (kg[g] = kg[g] || []).push(r); });
  Object.keys(kg).forEach(function(g) {
    if (kg[g].length < 2) return;
    k += '<Placemark><name>เส้นเชื่อมตามลำดับการตัด ' + _xml(g) + '</name><styleUrl>#s_line</styleUrl><LineString><tessellate>1</tessellate><coordinates>' +
      kg[g].map(function(r) { const c = MAP_COORDS[(r[F.PLOTINTECH] || '').trim()]; return c[1] + ',' + c[0] + ',0'; }).join(' ') +
      '</coordinates></LineString></Placemark>';
  });
  return k + '</Document></kml>';
}

function _mapSaveBlob(blob, name) {
  const a = document.createElement('a');
  a.href = URL.createObjectURL(blob); a.download = name;
  document.body.appendChild(a); a.click();
  setTimeout(function() { URL.revokeObjectURL(a.href); a.remove(); }, 1500);
}

function mapDownloadKmz() {
  if (!MAP_ROWS.length || !_mapCoordsLoaded) { showToast('⚠️ ยังไม่มีข้อมูลสำหรับดาวน์โหลด', 'error'); return; }
  showToast('⏳ กำลังสร้างไฟล์ KMZ...', 'success');
  setTimeout(function() {
    const kml = mapBuildKml();
    const saveKml = function() {
      _mapSaveBlob(new Blob([kml], { type: 'application/vnd.google-earth.kml+xml' }), 'cutting_group_map_6970.kml');
      showToast('✅ ดาวน์โหลดเป็นไฟล์ KML (ไม่พบตัวบีบอัด KMZ)', 'success');
    };
    // ★ [แก้ไข ข้อ 78] JSZip โหลดตอนใช้งาน (ถ้ายังไม่มา)
    _loadScript(JSZIP_JS, function(ok) {
      if (!ok || typeof JSZip === 'undefined') { saveKml(); return; }
      const zip = new JSZip(); zip.file('doc.kml', kml);
      zip.generateAsync({ type: 'blob', compression: 'DEFLATE' }).then(function(blob) {
        _mapSaveBlob(blob, 'cutting_group_map_6970.kmz');
        showToast('✅ ดาวน์โหลด KMZ สำเร็จ', 'success');
      });
    });
  }, 50);
}

// ===== UTILS =====
function populateDrop(id, vals, allLabel) {
  const sel = $(id);
  const prevValue = sel.value;
  const frag = document.createDocumentFragment();
  const def  = document.createElement('option'); def.value='all'; def.textContent=allLabel; frag.appendChild(def);
  (vals||[]).forEach(v => { const o=document.createElement('option'); o.value=v; o.textContent=v; frag.appendChild(o); });
  sel.innerHTML=''; sel.appendChild(frag);
  sel.value = (vals||[]).includes(prevValue) ? prevValue : 'all';
}

let _toastTimer = null;
function showToast(msg, type) {
  const t = $('toast');
  t.textContent = msg; t.className = 'show ' + type;
  if (_toastTimer) clearTimeout(_toastTimer);
  _toastTimer = setTimeout(() => { t.className=''; }, 3500);
}

// ★ [แก้ไข ข้อ 78] เริ่มโหลดข้อมูลทันทีที่สคริปต์ทำงาน (เดิมรอ window.onload ซึ่งต้องรอไลบรารีภายนอกโหลดครบก่อน)
(function() {
  buildTableHead();
  buildCgHead();
  const tw = document.querySelector('#viewTable .table-wrapper');
  if (tw) {
    // โหลดแถวตารางเพิ่มเมื่อเลื่อนใกล้ท้าย
    tw.addEventListener('scroll', _onTableScroll, { passive: true });
    tw.addEventListener('scroll', colfClose, { passive: true }); // เลื่อนตารางแล้วปิดกล่องกรอง (กล่องลอยอยู่ ไม่เลื่อนตาม)
  }
  const cw = document.querySelector('#viewCutgroup .cg-table-wrapper');
  if (cw) cw.addEventListener('scroll', cgfClose, { passive: true });
  init();
})();
<\/script>
</body>
</html>
`;
}
